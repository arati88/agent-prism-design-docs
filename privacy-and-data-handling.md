# Privacy and data handling

## Introduction

Agent Prism's privacy design rests on three rules:

1. **Keep as little as possible by default.** No prompt or response text is stored, IP addresses in the audit log are shortened, fleet collector snapshots lose prompt snippets, full paths and command lines, and failed-ingest copies lose prompt fields.
2. **Make everything extra opt-in, in one place.** `src/privacy.js` holds every switch that collects more: prompt capture, the LLM task classifier, full IPs, fleet details. Each one is off unless an operator turns it on.
3. **Check the promises with tests.** `test/privacy.test.js` checks the defaults, scans the source for unexpected outbound hosts, and runs an end-to-end "canary" test. That test sends an email, a phone number, prompt text, a home path and a secret, then searches every byte written to disk.

The customer-facing statement is [docs/PRIVACY.md](../PRIVACY.md): what's stored, what's never stored, outbound calls, opt-in features, how to verify, and answers for security questionnaires. This document is the design view: how those promises are enforced, and **where PRIVACY.md and the code no longer agree**.

Example: an agent sends `POST /api/ingest` with `prompt: "Email jane@acme.com about invoice 4411"` and `notes: "ran ok"`. The prompt isn't stored; only metrics are. If the save fails and the run goes to the retry queue, `prompt` and `messages` are removed and the rest is scrubbed first. With `LLM_CLASSIFIER_ENABLED=true`, up to 800 characters are sent to the organisation's AI provider as `Email [EMAIL] about invoice 4411` to label the task. `notes` is stored as sent.

It describes the controls as they exist in the code today.

## What we have today

| Control | Where | What it does |
| --- | --- | --- |
| Prompt capture switch | `isPromptCaptureEnabled()` (`PROMPT_CAPTURE_ENABLED=true`) **and** a plan with `promptCapture` (Pro and above) | Gateway prompts and responses are stored only when both are on. Sampling with `PROMPT_CAPTURE_SAMPLING_RATE`; scrubbed unless `PROMPT_CAPTURE_SCRUB_PII=false` |
| Classifier switch | `isLlmClassifierEnabled()` (`LLM_CLASSIFIER_ENABLED=true` and a provider configured), read at startup | Sends scrubbed prompt text to the AI provider (see [machine learning](machine-learning.md)) |
| Audit IP | `auditIp()` in `saas-store.logAuditEvent`, the single entry point | `truncated` (default), `full` or `none` |
| Fleet trimming | `sanitizeFleetSnapshot()` | Removes `summary`, shortens `cwd` to the folder name, removes `cmd`, unless `FLEET_STORE_SESSION_DETAILS=true` |
| Failed-ingest trimming | `stripPromptText()` then `scrubPiiDeep()` | Removes `prompt`, `messages`, `system` and `completion` (also inside `payload`), then scrubs the rest |
| Personal data scrub | `scrubPiiDeep()` / `scrubPii()` in `src/model-classifier.js` | Pattern rules: email, card, API key, bearer token, payment handle, IBAN, US SSN and phone, UK NI number and phone, Canada SIN, Australia TFN, Singapore NRIC, India Aadhaar, PAN and phone, international phone. Structural keys (`type`, `role`, `model`, `id`…) are left alone |
| Secrets at rest | `src/crypto.js` | Connector secrets encrypted with a key for each workspace (see [per-workspace encryption](per-tenant-encryption-byok.md)) |
| Browser isolation | `src/middleware/security-headers.js` | CSP `default-src 'self'`, `script-src 'self'` (no inline scripts), `connect-src 'self'`, `frame-ancestors 'self'`; self-hosted fonts; HSTS in production |
| Outbound address check | `src/security/outbound-url.js` | Workspace-supplied URLs (SIEM, Azure endpoint): HTTPS only, no embedded credentials, no loopback, link-local or cloud metadata addresses, private ranges only when allowed, checked again before each call, no redirects |
| Retention, export, erasure | `server.js`, stores | See [audit, GDPR and SIEM](audit-gdpr-and-siem.md) and [licensing](licensing-and-plans.md) |
| Proof | `test/privacy.test.js` | Defaults, IP shortening, trimming, scrubbing, no third-party origins in pages, allowed outbound hosts, no tracking dependencies, end-to-end canary, CSP, complete erasure |

## What it does

- Lets a customer's security team approve Agent Prism with the defaults, without reading the code.
- Puts every "collect more" decision in operator configuration, not in the code.
- Gives customers a way to check the claims themselves (`node --test test/privacy.test.js`).

## Where the data comes from

Everything the product collects, and where it's handled:

| Data | Entry point | Privacy handling |
| --- | --- | --- |
| Run metrics | `/api/ingest*`, gateway | Only normalised fields kept; `prompt`/`messages` not stored |
| Run free text: `notes`, `tags`, `breadcrumbs` | Ingest | **Stored as sent**, not scrubbed |
| Gateway prompts and responses | `/v1/*` | Forwarded; stored only with prompt capture |
| Fleet snapshots | `/api/fleet/ingest` | Trimmed; developer id, hostname, folder name and git branch kept |
| Users, sessions | Sign-in, SCIM, SSO | Email, name, role, password hash; tokens hashed |
| Audit log | 42 call sites | Actor, action, resource, details, shortened IP |
| Copilot data | GitHub API | Logins, activity, credits (see [Copilot connector](copilot-connector-and-sync.md)) |
| Data sent to the AI provider | AI Advisor, Prompt Advisor, classifier | See section 2 |

## How it works

### 1. What reaches disk

The end-to-end canary test starts a real server on an empty data folder. It posts ingest and fleet payloads containing an email, a phone number, prompt text, a home-folder path and a command-line secret, then reads every file the server wrote and fails if any of them appears. That's the main guarantee behind "never stored".

### 2. What leaves the installation

| Destination | When | What |
| --- | --- | --- |
| Anthropic, OpenAI, the workspace's Azure endpoint | Gateway calls | The agent's own request |
| The installation's AI provider | AI Advisor (each dashboard load), Prompt Advisor (dashboard tab), classifier (opt-in) | Telemetry summary; prompt text (**not** scrubbed for the Prompt Advisor); scrubbed prompt text (classifier) |
| `api.github.com` | Copilot connector with a token | The token and org; reads seats and usage |
| The workspace's SIEM | SIEM connector | Audit entries |
| `CB_ALERT_WEBHOOK_URL`, `OIDC_ISSUER` | Operator configured | Circuit alerts, sign-in |

## API

There's no privacy API of its own. The related routes are `GET /api/gdpr/export`, `DELETE /api/gdpr/data` and `POST /api/reset` (see [audit, GDPR and SIEM](audit-gdpr-and-siem.md)), and the routes for prompt captures and analyses.

## Dashboard / UI

There's no privacy screen. Prompt capture and the other switches are environment settings. The plan page shows whether prompt capture is included in the plan.

## Changes by file

Privacy-related code:

| File | What it does for privacy |
| --- | --- |
| `src/privacy.js` | All opt-in switches and trimming helpers |
| `src/model-classifier.js` | Personal data scrub rules |
| `src/saas-store.js` | Audit IP shortening |
| `src/middleware/security-headers.js` | CSP and HSTS |
| `src/security/outbound-url.js` | Outbound address checks |
| `server.js` | Capture checks, failed-ingest trimming, fleet trimming, GDPR routes |
| `docs/PRIVACY.md` | Customer statement |
| `test/privacy.test.js` | Proof |

## Tests

`test/privacy.test.js` is described in [docs/PRIVACY.md → How to verify](../PRIVACY.md#how-to-verify-this-yourself).

Not covered yet, and worth adding:

- The Prompt Advisor's outbound text, which isn't scrubbed today (see mismatch 1 below).
- `bin/cli.js` and `src/sdk/` in the outbound-host scan (see mismatch 4).
- Free-text run fields (`notes`, `breadcrumbs`) going to the AI provider.

## Where PRIVACY.md and the code don't match

These need to be fixed in the code or in the statement before PRIVACY.md is shown to customers.

1. **"Prompt Advisor: a prompt the user chose to submit."** In fact the dashboard's Prompt Advisor tab sends text from a page of recent runs **automatically** (breadcrumbs, notes or captured prompts), and `/api/advisor` sends it **without scrubbing**. The statement that "the optional classifier scrubs PII" is true, but the Prompt Advisor doesn't (see [AI Advisor and Prompt Advisor](ai-advisor-and-prompt-advisor.md)).
2. **"AI Advisor: aggregate metrics only… no prompt text."** It also sends the workspace's id and name, workflow names and details of the last 12 runs, and it runs on **every dashboard load**, not only when "someone opens the AI Advisor panel".
3. **"LLM_CLASSIFIER_ENABLED (+ ANTHROPIC_API_KEY) sends prompt text to Anthropic."** That's out of date. The classifier uses whichever `AI_PROVIDER` is configured, and an Anthropic key alone doesn't turn it on.
4. **"Does the product send data to the vendor? No."** `agent-prism login` saves the vendor's hosted demo, `https://agent-prism.onrender.com`, as the endpoint unless `AGENT_PRISM_URL` is set. SDK users who don't set it send their runs to the vendor-hosted server (see [SDK and CLI](node-sdk-and-cli.md)). The outbound-host test doesn't catch this, because it skips `src/sdk/` and doesn't scan `bin/`. The test also lists `agent-prism.onrender.com` as an allowed host, with the comment "never requested", which isn't true for the CLI.
5. **"Row-level security enforces tenant isolation in the database too."** Row-level security is only **forced** on some tables, and many queries use the pool directly (see [storage backends](storage-backends-and-migrations.md)). Scoping in the code is the real protection.
6. **"Audit trail… hash-chained."** True, but `details` and `ip` aren't part of the hash, and nothing ever checks the chain (see [audit, GDPR and SIEM](audit-gdpr-and-siem.md)).
7. **Export.** PRIVACY.md says the export covers "runs". It covers only the latest 10,000, and leaves out sessions, fleet snapshots, usage rollups, team assignments and Copilot data, which erasure does delete.
8. **"CSP… protects the stored API key from injected content."** Scripts can't be injected, but one place in the Prompt Advisor shows run text as **HTML** without escaping, so injected markup still appears (see [AI Advisor and Prompt Advisor](ai-advisor-and-prompt-advisor.md)).

## Known limitations

Other privacy gaps:

1. **Free-text run fields are a gap.** `notes`, `tags` and `breadcrumbs` are stored unscrubbed, appear in the activity feed, and are sent to the AI provider by the Prompt Advisor. Agents often put prompt excerpts or customer details there.
2. **Pattern scrubbing is best effort.** It can't recognise names, addresses or free-text identifiers. It also has false positives: for example, `[A-Z]{2}\d{2}…` (IBAN) and 10-digit rules can replace ordinary codes and numbers.
3. **The demo sign-in exposes the oldest workspace.** When the demo login is set, its password is public and the sign-in screen uses it automatically (see [auth](auth-sso-and-scim.md)).
4. **Erasure is for the whole workspace, not one person.** There's no way to find or erase one data subject's data across runs, fleet data, usage and the audit log.
