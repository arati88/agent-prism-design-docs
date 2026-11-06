# Agent certification and the production gate

## Introduction

Agent Prism can stop an agent from running in production until a person has reviewed it. This is **certification**.

There are three parts:

- **Tool danger classification.** Every tool an agent reports is given a danger level from 0 (read only) to 4 (critical), based on the tool's name.
- **Certification checks.** A fixed set of checks runs over the agent's past runs: how dangerous its tools are, whether a human approved the risky actions, policy violations, and how many successful staging runs it has. The result is a plain pass or fail with a reason for each check.
- **The production gate.** When a run is reported with `environment: "production"`, Agent Prism rejects it with `403` unless the agent has a valid production certificate.

Going to production takes **two people**. One person requests the promotion, and a different owner or admin approves it.

Example: the agent `invoice-bot` reports the tools `search_invoices` (level 0), `create_record` (level 1) and `send_email` (level 2). Its tier is 2, its highest level. A tier 2 agent needs 20 successful staging runs, a danger score of 50 or less, a human approval on every run that used a level 2 tool, and no policy violations. When it passes, Alice requests promotion and Bob, an admin, approves it. From then on, `invoice-bot` production runs are accepted for 30 days.

What follows is the certification flow as it's coded today, not a proposal.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/certification/danger-classifier.js` | `classifyTool`, `classifyManifest`: give each tool a danger category and level, then work out the run's danger score and tier. `inferAgentType`: labels the agent (observer, data-processor, ops-agent and so on) |
| `src/certification/certifier.js` | `evaluateAgent`: runs the checks and returns pass or fail |
| `src/connectors.js` | `enrichCertification`: classifies the tool manifest on every passive ingest run |
| `server.js` | `enforceProductionCertGate`, `checkAutoRevoke`, and the `/api/agents/*` and `/api/promotions/*` routes |
| Storage | Postgres: `agent_definitions`, `agent_tools`, `agent_certifications`, `certification_checks`, `agent_promotions` (migrations `007`, `008`). File store: `certifications` and `promotions` lists, with agent definitions worked out from runs each time they're read |
| `public/app.js` | The **Agent Certification** panel and the **Pending Promotions** queue |

## What it does

- Lists every agent that has reported a tool manifest, with its tier, its tools and the risk of each one.
- Checks an agent against rules that get stricter as its tier rises, and explains every failure.
- Blocks production runs from agents that aren't certified, or whose certificate has expired.
- Requires a second, different owner or admin to approve each move to production.
- **Revokes** a production certificate automatically when the agent starts using a new high-risk tool in production.
- Writes every certify, promotion request, approval, rejection and revocation to the audit log.

## Where the data comes from

Everything is taken from the agent's **runs**, sent through passive ingest (see [passive ingest](passive-ingest.md)). These run fields matter:

| Field | Used for |
| --- | --- |
| `toolManifest` (or `tool_manifest`) | The tools the agent used. A list of names, or objects with a `name` |
| `humanApprovals` (or `human_approvals`) | The human approvals in that run. Each one can name the tool it approved in `toolCalled` |
| `environment` | `staging` or `production`. Production runs pass through the gate. Staging runs count towards the minimum |
| `status` | `success` or `completed` counts as a successful run |
| `policyViolations` | Must be 0 across every run |
| `autonomyLevel` | Compared with the tier's recommended maximum |

**The danger category and level are always worked out from the tool's name by the server.** If the caller sends its own `dangerCategory`, `dangerLevel` or `requiresHitl`, they're ignored. Otherwise an agent could certify itself by labelling a destructive tool as "read".

## How it works

### 1. Classifying tools

Before matching, the tool name is split into words: `dropTable` becomes `drop Table`, and `exec_shell` becomes `exec shell`. The name is then checked against the rules from **most dangerous down**, and the first match wins.

| Level | Category | Example names | Human approval needed |
| --- | --- | --- | --- |
| 4 Critical | `privilege-esc` | `assume_role`, `sudo`, `escalate` | Yes |
| 4 Critical | `infra-mutate` | `terraform_destroy`, `delete_cluster`, `provision_cloud` | Yes |
| 3 High | `destructive` | `drop`, `truncate`, `rm -rf`, `purge`, `bulk_delete`, `delete_all` | Yes |
| 3 High | `process-exec` | `exec`, `spawn`, `run_command`, `eval`, `subprocess` | Yes |
| 3 High | `secret-access` | `get_secret`, `vault_get`, `kms_decrypt`, `get_api_key` | Yes |
| 3 High | `bulk-mutation` | `update_all`, `batch_delete`, `bulk_update` | Yes |
| 2 Medium | `destructive` | `delete`, `remove`, `destroy`, `kill` | Yes |
| 2 Medium | `external-call` | `send_email`, `post_slack`, `send_webhook`, `send_sms` | No |
| 1 Low | `internal-write` | `insert`, `update`, `create`, `write_file`, `upsert`, `add` | No |
| 0 Read | `read` | `get`, `list`, `search`, `query`, `fetch`, `read_file` | No |
| 2 | `unclassified` | Anything that matches none of the above | No |

For each run:

```text
danger score = sum of tool weights, capped at 100
               (weights: level 0 → 0, 1 → 5, 2 → 15, 3 → 35, 4 → 75)
run tier     = the highest level of any tool in the run
```

These values are saved on the run (`toolManifest`, `dangerScore`, `agentTier`).

### 2. The agent's tier

An agent's tier is the **highest tier of any run it has ever had**. On Postgres, `agent_definitions` and `agent_tools` are updated on each run that has a manifest. The tier only ever goes up. The file store works out the same thing from the runs each time.

### 3. The checks

`evaluateAgent(agentName, runs, environment)` looks at **all** of the agent's runs. The rules depend on the tier:

| Tier | Min successful staging runs | Max danger score | Max autonomy (warning only) | Approval needed on tools of level | Production |
| --- | --- | --- | --- | --- | --- |
| 0 | 5 | 5 | 5 | — | Allowed |
| 1 | 10 | 20 | 4 | — | Allowed |
| 2 | 20 | 50 | 3 | 2 | Allowed |
| 3 | 30 | 75 | 2 | 2, 3 | Allowed |
| 4 | — | 0 | 1 | 2, 3, 4 | **Never** |

| Check | Fails when | Blocks? |
| --- | --- | --- |
| `tool_manifest_present` | No run ever sent a tool manifest | No, warning only |
| `no_unclassified_tools` | Any tool is `unclassified` | Yes |
| `tier_4_prod_blocked` | Production check for a tier 4 agent | Yes |
| `danger_score_ceiling` | The highest danger score of any run is above the tier's maximum | Yes |
| `hitl_coverage` | A tool at a level that needs approval was used in a run without a human approval for it | Yes |
| `policy_violations_zero` | The total of `policyViolations` over all runs is above 0 | Yes |
| `min_staging_runs` (production only) | Fewer successful staging runs than the tier's minimum | Yes |
| `autonomy_level_appropriate` | The highest `autonomyLevel` is above the tier's maximum | No, warning only |

If no blocking check fails, the result is `certified`. Otherwise it's `uncertified`, with `failureReasons` listing each failed check. An agent with no runs at all is always `uncertified`.

A human approval counts for a tool when its `toolCalled` is that tool's name, or when it has no `toolCalled` at all.

### 4. Certifying, promoting and approving

```text
Staging:     POST /api/agents/:name/certify  { environment: "staging" }
               → checks run → staging certificate saved (certified or uncertified)

Production:  POST /api/agents/:name/promote
               → production checks run
                  fail → production certificate saved as uncertified,
                         promotion saved as rejected, 422 returned
                  pass → promotion opened as "pending", 202 returned
                         (nothing is certified yet; asking again returns the same pending one)

             POST /api/promotions/:id/approve   (a different owner or admin, signed in)
               → production certificate saved as certified, valid for 30 days

             POST /api/promotions/:id/reject    { reason }
               → promotion closed, agent stays uncertified
```

Approval rules: the approver must be **signed in to the dashboard** (an API key isn't enough), must be an **owner or admin**, and must **not be the person who requested** the promotion. Approving or rejecting a promotion that's already closed returns `409`.

Who can do what (`src/permissions.js`):

| Action | Permission | Lowest role |
| --- | --- | --- |
| Request promotion | `agents.operate` | member |
| Certify, revoke, approve, reject | `agents.govern` | admin |

### 5. The production gate

`enforceProductionCertGate` runs on `/api/ingest`, `/api/ingest/claude`, `/api/ingest/copilot` and `/api/ingest/generic`, after the plan check and before the run is saved:

1. If the run's environment isn't `production`, it's let through.
2. If there's no production certificate, or it isn't `certified`, it's rejected with `403 agent_not_certified`, listing any failures.
3. If the certificate's `expiresAt` has passed, it's rejected with `403 cert_expired`.
4. Otherwise it's let through.

### 6. Automatic revocation

After a production run is saved, `checkAutoRevoke` compares its tools with the risky tools recorded when the agent was certified. If the run has a tool that is **level 3 or higher, or unclassified**, and wasn't in that list, the production certificate is set to `revoked` and an `Agent Cert Auto-Revoked` audit event is written. The next production run is then blocked.

### 7. Manual revocation

`POST /api/agents/:name/revoke { environment, reason }` sets the certificate to `revoked`. Certifying again replaces it.

## API

| Method and path | Who | What it does |
| --- | --- | --- |
| `GET /api/agents` | Tenant | All agents with tier, type, tools and staging and production status |
| `GET /api/agents/certifications` | Tenant | Every certificate in the workspace |
| `GET /api/agents/:name` | Tenant | One agent and its tools. `404` if it has never sent a manifest |
| `GET /api/agents/:name/cert?env=production` | Tenant | The current certificate, with its checks. `uncertified` if there isn't one |
| `POST /api/agents/:name/certify` | admin | Runs the checks for `environment` (default `staging`) and saves the result |
| `POST /api/agents/:name/promote` | member | Runs the production checks and opens a pending promotion if they pass |
| `GET /api/promotions?status=pending` | Tenant | Promotions, newest first |
| `POST /api/promotions/:id/approve` | Signed-in owner or admin, not the requester | Certifies the agent for production |
| `POST /api/promotions/:id/reject` | Signed-in owner or admin | Closes the promotion, with an optional `reason` |
| `POST /api/agents/:name/revoke` | admin | Revokes the certificate for `environment` (default `production`) |

Gate rejection example:

```json
{ "error": "agent_not_certified",
  "message": "Agent \"invoice-bot\" is not certified for production. POST /api/agents/invoice-bot/certify then /promote to certify.",
  "certStatus": "uncertified", "failures": [] }
```

## Dashboard / UI

The **Agent Certification** panel shows:

- A "How production enforcement works" guide in four steps: certify in staging, promote, the gate blocks uncertified runs, auto-revoke.
- A card for each agent. It shows the tier badge, agent type, up to 5 tool pills, a risk score bar, human review coverage, and the staging and production status (Certified, Not Certified, Pending Approval or Revoked).
- One next step on each card: **Review & Certify for Staging** → **Request Production Promotion** → **Live in production**, with **Revoke Production Cert**.
- A review screen before certifying. It lists every tool with its risk and what the person is approving: the tier, whether human approval is needed, 30-day validity and auto-revoke. A tier 4 agent can't be certified for production.

The **Pending Promotions** queue lists open requests. Owners and admins see **Approve** and **Reject**. The person who made the request sees "a different reviewer must approve it".

## Changes by file

Where the certification code lives:

| File | What it does for certification |
| --- | --- |
| `src/certification/danger-classifier.js` | Danger rules, weights, agent type |
| `src/certification/certifier.js` | Tier thresholds and checks |
| `src/connectors.js` | Classifies the tool manifest on ingest |
| `src/permissions.js` | `agents.operate` and `agents.govern` |
| `server.js` | Gate, auto-revoke, agent and promotion routes, audit events |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | Agent definitions, certificates, check log, promotions |
| `db/migrations/007_agent_certification_phase1.sql`, `008_agent_certification_phase2.sql` | Tables and run columns |
| `public/app.js` | Certification panel, review screens, Pending Promotions |

## Tests

What's tested today:

- `test/certification.test.js`: a danger level or category sent by the caller is ignored; camelCase names (`dropTable`, `truncateAuditLogs`, `killContainer`, `deleteAllRecords`) are classified correctly; snake_case and kebab-case still work; unknown names fall to `unclassified`; plain string names work; a disguised destructive tool still gives a high danger score.
- `test/agent-definitions.test.js` (file store): runs without a manifest are ignored; tier and tool danger are the highest seen; results are kept per workspace; staging and production status are shown.

Not tested yet, and worth adding:

- `evaluateAgent`: each check, each tier's thresholds, and pass or fail overall.
- The gate: uncertified, revoked and expired certificates are blocked; staging runs pass.
- The promotion flow: the requester can't approve; API keys can't approve; approving twice returns `409`; a failed evaluation is rejected straight away.
- Auto-revoke: a new level 3 or unclassified tool revokes; a tool that was already certified doesn't.

## Known limitations

Gaps I found while going through the certification code:

1. **The gate only covers the four `/api/ingest*` routes.** Live gateway calls (`/v1/messages`, `/v1/responses`, `/v1/chat/completions`, `/v1/azure/*`) and `/api/fleet/ingest` are never blocked. The dashboard's step 3 says that routing calls through the gateway with `ANTHROPIC_BASE_URL` "enforces automatically", but it doesn't.
2. **The two-person rule can be skipped.** `POST /api/agents/:name/certify` with `environment: "production"` saves a certified production certificate straight away, with no pending promotion and no second reviewer. Anyone with `agents.govern` can do this, alone. That includes a full API key created by an admin, and older keys with no role, which act as owner (see [multi-tenancy](multi-tenancy-and-api-keys.md)).
3. **An expired certificate still shows as "Certified".** Its status stays `certified` and only the gate looks at `expiresAt`. There's no warning before the 30 days run out, so production runs start failing with no notice.
4. **A failed re-check takes a live agent out of production.** Running `/promote` or a production `/certify` for an agent that's already certified, and failing, overwrites the certificate with `uncertified`. Its production runs are blocked straight away.
