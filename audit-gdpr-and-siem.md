# Audit log, GDPR export and erasure, and SIEM delivery

## Introduction

- **Audit log.** A per-workspace record of who did what, when and from where: sign-ins, user and role changes, API keys, connectors, certification and promotions, ingest, settings, erasure and so on. Each entry is chained to the one before it with a SHA-256 hash, so a deleted or edited entry can be spotted. Admins can view it and download it as CSV.
- **GDPR.** An owner or admin can **export** the workspace's data as JSON. An owner can **erase** the workspace's telemetry: runs, prompt captures, sessions, fleet data, usage, Copilot data and so on.
- **SIEM delivery.** Each audit entry is also posted, as it happens, to the customer's SIEM (Splunk, Sentinel, Datadog and so on), optionally signed with HMAC-SHA256.
- **Privacy defaults.** IP addresses in the audit log are shortened by default, and data that could be personal is only collected when switched on (see `docs/PRIVACY.md`).

Example: Bob (admin) approves the promotion of `invoice-bot`. The audit log gets `Agent Promotion Approved`, with actor `bob@acme.com`, resource `invoice-bot`, details `{ promotionId, requestedBy }` and IP `203.0.113.0` (last part zeroed). Its `prevHash` is the previous entry's hash. Within a second, Acme's Splunk receives `{ type: "agent_prism.audit_log", deliveredAt, event: {…} }` with an `X-Agent-Prism-Signature: sha256=…` header.

This document describes how these work **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/saas-store.js` → `logAuditEvent` | The single entry point for every audit write. Shortens the IP, then calls the backend |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | `logAuditEvent` (hash chain, store, start SIEM delivery), `listAuditLogs`, `resetTenantRuns`, `erasePersonalTelemetry`, `applyDataRetention` |
| `src/siem-webhook.js` | `dispatchSiemWebhook`: signs and posts one event, with a timeout, checking the address is safe |
| `src/security/outbound-url.js` | `assertSafeOutboundUrl`: blocks private and internal addresses (also re-checked at send time) |
| `src/privacy.js` | `auditIp`, the prompt capture and classifier switches, fleet snapshot trimming, `stripPromptText` |
| `server.js` | `/api/audit`, `/api/audit/export`, `/api/gdpr/export`, `DELETE /api/gdpr/data`, `/api/reset`, the SIEM connector in the catalog, and the 42 places that write audit events |
| Storage | `audit_logs` (id, workspace, timestamp, actor, action, resource, details, IP, hash, previous hash). A `siem_webhook` connector holds the URL and secret, encrypted with the workspace's key |
| `public/app.js` | Audit log table, CSV download, connector setup |

## What it does

- Gives security and compliance teams a trail of every important action, in Agent Prism and in their own SIEM.
- Makes tampering visible: each entry's hash includes the previous entry's hash.
- Lets a customer take all their data out (portability) and remove their telemetry (erasure).
- Keeps less personal data by default: shortened IPs, no prompt text, no fleet command lines.

## Where the data comes from

| Field | Source |
| --- | --- |
| `actor` | Usually the signed-in user's email, or the API key prefix. Some events use fixed names such as `Admin`, `SSO`, `System (Auto-Revoke)` or the workspace name |
| `action` | A fixed name at each call site, for example `Dashboard Login`, `User Role Changed`, `Tenant API Key Created`, `Agent Certified`, `Telemetry Ingested`, `GDPR_DATA_ERASURE` |
| `resource`, `details` | Set at each call site, for example the agent name and the changes made |
| `ip` | The client IP (see `src/middleware/client-ip.js`), changed by `AUDIT_LOG_IP_MODE` |
| SIEM URL and secret | The workspace's `siem_webhook` connector (`url`, and `apiKey` as the signing secret) |

`AUDIT_LOG_IP_MODE`:

| Value | Stored |
| --- | --- |
| `truncated` (default) | IPv4 with the last part zeroed (`203.0.113.0`). IPv6 keeps its first 3 groups |
| `full` | The exact address |
| `none` | Nothing |

## How it works

### 1. Writing an audit entry

1. `saas-store.logAuditEvent` shortens the IP.
2. The backend reads the workspace's latest hash. The first entry uses 64 zeros.
3. `hash = SHA-256(prevHash : id : workspace : actor : action : resource : timestamp)`.
4. The entry is saved with `hash` and `prevHash`.
5. SIEM delivery starts **without waiting** (section 4).

On Postgres, entries are kept until the workspace is deleted. On the file store, only the **newest 500 entries across the whole installation** are kept.

### 2. Reading and exporting

- `GET /api/audit` (`audit.view`, admin) returns the latest **100** entries, newest first. The file store returns all of its entries for the workspace.
- `GET /api/audit/export` (`audit.view`) returns the same entries as CSV, with columns timestamp, actor, action, resource, ip, details, hash and prevHash. Cells that start with `=`, `+`, `-`, `@`, a tab or a carriage return are neutralised, so a spreadsheet can't run them.

### 3. GDPR

**Export** (`GET /api/gdpr/export?limit=&offset=`, `data.export`, admin) returns JSON:

| Part | Contents |
| --- | --- |
| `tenant` | Workspace details |
| `users` | id, email, name, role, createdAt |
| `connectors` | Each connector, with every config value replaced by `[REDACTED]` |
| `runs` | One page (`limit` 1–5,000, default 1,000) of the workspace's latest 10,000 runs, with `runsTotal` |
| `auditLogs` | The latest 100 entries |
| `promptCaptures` | One page, using the same `limit` and `offset`, with `promptCapturesTotal` |

**Erasure** (`DELETE /api/gdpr/data`, `data.erase`, owner) runs in order: delete runs, savings baselines and spend blocks; delete prompt captures; delete per-person telemetry (sessions, prompt analyses, fleet snapshots, daily usage by developer and by model, team assignments, Copilot seats, usage, model usage and budget status, failed ingests). Then it writes a `GDPR_DATA_ERASURE` audit entry and returns `{ status: "erased", erasedAt }`.

**Reset** (`POST /api/reset`, owner) deletes the workspace's runs, baselines and spend blocks only.

**Retention** (daily): runs and prompt captures older than the plan's limit are deleted, as are finished sessions (Free 30 days, Starter 90, Pro 365, Enterprise no limit; `DATA_RETENTION_DAYS` can set a lower cap for the installation). Audit entries aren't affected (see [storage backends](storage-backends-and-migrations.md)).

### 4. SIEM delivery

For every audit entry:

1. Look up the workspace's `siem_webhook` connector. If there isn't one, stop.
2. Decrypt its URL and secret with the workspace's key. If the key is sealed or missing, skip without a word.
3. Build the body `{ "type": "agent_prism.audit_log", "deliveredAt": "…", "event": { …the audit entry… } }`.
4. With a secret: add `X-Agent-Prism-Signature: sha256=<HMAC-SHA256 of the body>`.
5. Check again that the URL resolves to a public address (protection against DNS rebinding). POST with a **5-second** timeout (`SIEM_WEBHOOK_TIMEOUT_MS`), and refuse redirects.
6. On failure, write a line to the server's error output. No retry.

The SIEM can check the signature by computing HMAC-SHA256 over the raw body with the shared secret and comparing it with the header.

## API

| Method and path | Auth | What it does |
| --- | --- | --- |
| `GET /api/audit` | `audit.view` (admin) | Latest 100 entries |
| `GET /api/audit/export` | `audit.view` | The same, as CSV |
| `GET /api/gdpr/export?limit=&offset=` | `data.export` (admin) | Workspace data as JSON, paged |
| `DELETE /api/gdpr/data` | `data.erase` (owner) | Erase telemetry |
| `POST /api/reset` | `data.erase` (owner) | Delete runs only |
| `POST /api/connectors` with `provider: "siem_webhook"` | `connectors.manage` (admin) | Set the SIEM URL and secret (see the connectors document) |

## Dashboard / UI

- **Admin → Audit log.** A table of the latest entries (time, actor, action, resource, IP) and **Export CSV**.
- **Connectors → SIEM webhook.** "Stream every audit log event (certifications, promotion approvals, SCIM provisioning, connector changes) to your SIEM in real time." It has a destination URL and an optional signing secret.
- GDPR export and erasure are API-only (no buttons). The dashboard's reset action calls `POST /api/reset`.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does |
| --- | --- |
| `src/saas-store.js` | IP shortening at the single audit entry point |
| `src/privacy.js` | `auditIp` and the other privacy switches |
| `src/siem-webhook.js` | SIEM delivery |
| `src/security/outbound-url.js` | Safe outbound address check |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | Hash chain, storage, listing, erasure, retention |
| `server.js` | Routes and every audit call site |
| `public/app.js` | Audit table, CSV, SIEM connector form |
| `docs/PRIVACY.md` | The list of personal data kept and the switches for it |

## Tests

What's tested today:

- `test/privacy.test.js`: prompt capture and the classifier are off by default; audit IPs are shortened by default; fleet snapshots lose prompt snippets, paths and command lines; failed-ingest copies keep metrics but drop prompt text; `scrubPiiDeep` reaches nested content; the browser and server only contact expected hosts; no tracking SDKs; personal data sent to ingest and fleet never reaches disk; the audit log stores a shortened IP; the dashboard CSP blocks inline and third-party scripts; **GDPR erasure removes runs and every per-person telemetry record**.
- `test/user-management.test.js`: audit CSV export neutralises spreadsheet formulas.

Not tested yet, and worth adding:

- The hash chain: each entry's `prevHash` matches the entry before it, including when two entries are written at once (see limitation 1).
- SIEM delivery: the signature, the timeout, a private address refused, a redirect refused.
- GDPR export: paging past 10,000 runs.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **The hash chain proves less than it suggests.** `details` and `ip` aren't part of the hash, so they can be edited without breaking it. Nothing ever checks the chain: there's no verify route or tool. Two entries written at the same moment can both use the same previous hash, which forks the chain, because the latest hash is read without a lock.
2. **The audit log can only be read 100 entries at a time.** `/api/audit`, the CSV export and the GDPR export all return the latest 100 on Postgres, with no paging and no date filter. Older entries exist but can't be retrieved through the product.
3. **On the file store, one busy workspace can push out everyone's history.** Only the newest 500 entries **across all workspaces** are kept.
4. **SIEM delivery can lose events.** Delivery is attempted once, with no retry and no queue. A failure only writes a line to the server log, so the customer can't see that events are missing. If the workspace's key is sealed, delivery stops without a word. The signature has no timestamp, so a captured delivery can be replayed. Only audit events are sent, not cost alerts or leaks.
5. **Erasure covers the whole workspace's telemetry, not one person.** There's no way to export or erase a single data subject. It also leaves behind **users** (names and emails), the **audit log** (actor emails, shortened IPs, resources), connectors, certifications and promotions, ML weights, savings settings and dashboard sessions. Whether that's acceptable depends on the customer's legal basis; it isn't documented in the product.
6. **Export and erasure don't match.** Erasure deletes sessions, fleet snapshots, daily usage, team assignments and Copilot data, but the export includes none of them. The export only reaches the latest 10,000 runs, so older runs can't be exported even though they exist. Prompt captures use the same page settings as runs.
7. **Some changes aren't audited.** `POST /api/reset` deletes all runs with no audit entry. Audit entries are only written where the code calls `logAuditEvent`, and there's no list of which actions are covered.
8. **The actor isn't always the person.** Erasure records the API key prefix or `"tenant"`, Prompt Advisor apply records the workspace name, and platform-admin actions record `"Admin"` with no indication of who used the admin secret.
9. **Audit entries are never deleted on Postgres.** Retention doesn't touch them, so they grow forever, and there's no setting for how long to keep them.
10. **A failed audit write can fail a request that already succeeded.** Most routes wait for `logAuditEvent` after making the change. If the audit write throws, the caller gets an error although the change was made.
