# Passive ingest: agents reporting their own runs to Agent Prism

## Introduction

Passive ingest is the second of the two ways run data gets into Agent Prism. The first is the [live gateway proxy](gateway-proxy.md).

With the gateway, Agent Prism sits in the middle of every model call. With passive ingest it doesn't: the agent calls its model provider directly, as usual, and **afterwards** sends Agent Prism a short report of the run. The report says which agent ran, which model it used, the tokens, the cost, how long it took, and whether it succeeded.

Passive ingest works for any agent on any provider, including ones the gateway doesn't support (Copilot, LangChain, custom agents, CI jobs). It needs no change to how the agent talks to its model. The trade-off is that Agent Prism has to **trust what the agent reports**.

Example: a nightly CI agent finishes reviewing a pull request and posts this:

```json
POST /api/ingest
{ "source": "generic",
  "payload": { "agent_name": "PR Reviewer", "model": "gpt-4.1", "prompt_tokens": 12000,
               "completion_tokens": 900, "cost_usd": 0.031, "status": "success",
               "environment": "staging", "tool_manifest": ["read_file", "post_comment"] } }
```

The run appears on the dashboard within seconds, counted in cost, scores, leaks and Run value like any other run.

Everything below is how passive ingest behaves in the current code.

## What we have today

Five routes. Four of them write **runs**; the fifth writes **developer session snapshots** from the fleet collector.

| Route | Who sends it | What it expects | Signature check | Writes |
| --- | --- | --- | --- | --- |
| `POST /api/ingest` | Any agent, the SDK (`src/sdk/index.js`) | `{ source, payload }`, where `source` is `claude`, `copilot` or `generic` | No | A run |
| `POST /api/ingest/claude` | Claude agents, Claude webhooks | Claude-shaped fields (`runId`, `agent`, `inputTokens`, …) | Optional (`CLAUDE_WEBHOOK_SECRET`) | A run |
| `POST /api/ingest/copilot` | Copilot webhooks, custom Copilot agents | Copilot-shaped fields (`session_id`, `agent_name`, `prompt_tokens`, …) | Optional (`COPILOT_WEBHOOK_SECRET`) | A run |
| `POST /api/ingest/generic` | Anything else (LiteLLM logs, custom agents) | Any run fields, camelCase or snake_case, optionally wrapped in `payload` | No | A run |
| `POST /api/fleet/ingest` | The fleet collector daemon on developer machines, every ~30 seconds | A machine snapshot: developer, sessions with cumulative token totals | No | A machine snapshot + daily usage rollups (not runs) |

All five need an Agent Prism key (`acp_…`), so every report lands in the caller's own workspace.

## What it does

For the four **run** routes, in this order:

1. **Checks who is calling** (`requireTenant`).
2. **Checks the signature**, on the Claude and Copilot routes only, when a webhook secret is set.
3. **Turns the report into a standard run** with the matching *normalizer* (see below). Every field is cleaned and given a safe default.
4. **Skips duplicates** (`/api/ingest` only).
5. **Checks the plan**: is the workspace under its monthly run cap and agent cap?
6. **Checks production certification.** A run marked `environment: "production"` from an agent that isn't certified is **refused**.
7. **Saves the run** (`upsertTenantRuns`).
8. **Afterwards:** checks whether the run should revoke the agent's certification, and on some routes writes an audit entry, sends a budget alert and classifies the task.

The fleet route works differently. It's described in [Fleet ingest](#fleet-ingest).

## Where the data comes from

Everything comes **from the agent's own report**. Agent Prism doesn't call the provider to check it.

### Normalizers: one standard run from three report shapes

Each provider names things differently. A normalizer in `src/connectors.js` maps each shape onto the same run fields:

| Run field | `normalizeGenericRun` accepts | `normalizeClaudeRun` reads | `normalizeCopilotRun` reads | Default if missing |
| --- | --- | --- | --- | --- |
| `id` | `id` | `runId` | `session_id` | random `run_…` |
| `agentName` | `agentName`, `agent_name` | `agent` | `agent_name` | "Unknown Agent" / "Claude Agent" / "GitHub Copilot Agent" |
| `provider` | `provider` | always "Anthropic" | always "GitHub" | "Unknown Provider" |
| `model` | `model` | `model` | `model_name` | "unknown" |
| `taskType` | `taskType`, `task_type` | `jobType` | `intent` | "general" |
| `status` | `status` | `status` | `outcome` | "success" |
| `startTime` / `endTime` | `startTime`, `started_at` / `endTime`, `completed_at`, `finished_at` | `startedAt` / `finishedAt` | `started_at` / `completed_at` | now |
| `latencyMs` | `latencyMs`, `latency_ms`, `duration_ms` | `elapsedMs` | `duration_ms` | 0 |
| `tokensIn` / `tokensOut` | `tokensIn`, `prompt_tokens` / `tokensOut`, `completion_tokens` | `inputTokens` / `outputTokens` | `prompt_tokens` / `completion_tokens` | 0 |
| `costUsd` | `costUsd`, `cost_usd`, `estimated_cost_usd` | `costUsd` | `estimated_cost_usd` | 0 |
| `budgetUsd` | `budgetUsd`, `budget_usd` | `budgetUsd` | `budget_usd` | 1 |
| `retryCount` | `retryCount`, `retry_count` | `retries` | `retry_count` | 0 |
| `toolCalls` | `toolCalls`, `tool_invocations` | `toolCalls` | `tool_invocations` | 0 |
| `policyViolations` | `policyViolations`, `policy_alerts` | `guardrailHits` | `policy_alerts` | 0 |
| `userSatisfaction` | `userSatisfaction`, `user_score` | `feedbackScore` | `user_score` | 4 |
| `autonomyLevel` | `autonomyLevel`, `autonomy_level` | `autonomyLevel` | `autonomy_level` | 3 |
| `environment` | `environment` | `environment` | `environment` | **"production"** |
| `workflow` / `team` | `workflow` / `team` | `flow` / `team` | `workflow` / `team` | "default" / "platform" |
| `tags` / `breadcrumbs` / `notes` | `tags`, `labels` / `breadcrumbs`, `trace` / `notes`, `summary` | `tags` / `breadcrumbs` / `notes` | `labels` / `trace` / `summary` | empty |
| `toolManifest`, `humanApprovals` | `toolManifest`, `tool_manifest` / `humanApprovals`, `human_approvals` | same | same | empty |

Every normalizer also:

- **Cleans every field** so a bad value can't break storage: numbers are clamped to 0 or more and to what the database column holds, so `"abc"` tokens become 0. Text is cut to a maximum length, dates outside 2000–2100 fall back to now, and tag lists keep at most 50 items.
- **Reads the prompt token breakdown** if sent: user, system, context, tool-result and memory tokens.
- **Classifies the tool list** (`classifyManifest` in `src/certification/danger-classifier.js`): each tool gets a danger level, and the run gets a `dangerScore` and an `agentTier`. Certification uses these.

## How it works

### Signature check (Claude and Copilot routes)

If `CLAUDE_WEBHOOK_SECRET` or `COPILOT_WEBHOOK_SECRET` is set, the route checks an HMAC-SHA256 signature of the **raw** body before parsing it (`verifyHmacSignature` in `src/ingest/verify.js`, using a timing-safe comparison).

| Route | Signature header | Wrong or missing signature |
| --- | --- | --- |
| `/api/ingest/claude` | `anthropic-signature` or `x-webhook-signature` | `401 invalid_signature` |
| `/api/ingest/copilot` | `x-hub-signature-256` (GitHub's) or `x-webhook-signature` | `401 invalid_signature` |

The value can be plain hex or `sha256=<hex>`. If the secret isn't set, no signature is checked.

### Duplicates (`/api/ingest` only)

Webhook senders retry when they don't get an answer in time, so the same run can arrive twice.

- The key is the `Idempotency-Key` header, or the run's `id` if there's no header.
- `checkAndSetIdempotencyKey` records the key per workspace. If it's been seen before, the answer is `200 { "status": "duplicate", "runId": … }` and nothing is saved again.
- Keys are deleted after 24 hours (a job runs every hour).

On the other three run routes, sending the same `id` twice **overwrites** the earlier run instead of adding a second one, because runs are saved by id.

### Production certification gate

`enforceProductionCertGate` runs on all four run routes:

- Not production → allowed.
- Production, and the agent has a valid production certificate → allowed.
- Production and not certified → `403 agent_not_certified`, with the reasons the certification failed and how to certify.
- Production and the certificate has expired → `403 cert_expired`.

`environment` defaults to `"production"` when it isn't sent, so **an agent that leaves it out is treated as production** and must be certified.

### After the run is saved

| Step | `/api/ingest` | `/claude` | `/copilot` | `/generic` |
| --- | --- | --- | --- | --- |
| **Auto-revoke** (`checkAutoRevoke`): a production run using a high-danger or unclassified tool that wasn't there when the agent was certified revokes the certificate and writes an audit entry | ✅ | ✅ | ✅ | ✅ |
| **Audit log** entry "Telemetry Ingested" | ✅ | ❌ | ❌ | ❌ |
| **Budget alert** to Slack (`SLACK_WEBHOOK_URL`) when the run cost more than its budget | ✅ | ❌ | ❌ | ❌ |
| **AI task classification** in the background (`classifyTaskWithLLM`, falls back to the regex classifier), which updates the run's task type | ❌ | ✅ if `messages` sent | ❌ | ✅ if `prompt` sent (see limitation 9) |

### Retry queue for failed saves (`/api/ingest` only)

If saving the run fails (for example the database is briefly unavailable), the report isn't lost:

1. The body is put in the **failed-ingest queue** (`saveFailedIngest`), with prompt text and personal data removed first.
2. The agent gets `202 { "status": "queued" }`, so it doesn't retry by itself.
3. Every 5 minutes, up to 20 queued reports are tried again.
4. A report that keeps failing waits longer each time (1 min, 5 min, 25 min, then 2 hours at most) and is marked **failed** after 5 attempts.

### Fleet ingest

`/api/fleet/ingest` takes a **snapshot** of one developer machine from the fleet collector: its Claude Code sessions, each with *cumulative* token totals.

1. The snapshot is cleaned (`normalizeFleetSnapshot`): at most 500 sessions, 200 processes and 200 ports, numbers clamped. Prompt snippets, full paths and command lines are then removed (`sanitizeFleetSnapshot` in `src/privacy.js`).
2. Each session is compared with the **previous** snapshot from the same machine. Only the **increase** in tokens counts. Otherwise every 30-second push would count the whole session again.
3. A session seen for the first time adds nothing. Its totals may cover days before tracking started, so they become the starting point and only later growth is counted.
4. Cost of the increase = `computeClaudeCost` with the session's model and `config/pricing.json`. This is the one ingest path where Agent Prism calculates the cost itself.
5. The increase is added to the developer's **daily usage** for today (`upsertDailyUsage`) and to a per-model breakdown (`upsertDailyUsageByModel`). This feeds the Live Sessions view and AI Consumption History.
6. The snapshot replaces the machine's previous one.

The fleet route has no plan check, no certification gate and no duplicate check, and it writes no runs.

## API

### Requests

`/api/ingest` wraps the report:

```json
{ "source": "claude", "payload": { "runId": "r-123", "agent": "Docs Writer", "model": "claude-sonnet-4-5",
                                   "inputTokens": 5000, "outputTokens": 800, "costUsd": 0.027,
                                   "environment": "staging" } }
```

`source` must be `claude`, `copilot` or `generic` (or left out, meaning generic). The other routes take the report directly. `/api/ingest/generic` also accepts it wrapped in `payload`.

### Responses

| Status | When | Body |
| --- | --- | --- |
| `201` | Saved | `/api/ingest`: `{ status: "ingested", tenant, runId, normalizedRun }`. Others: `{ status: "ingested", source }` |
| `200` | Duplicate (`/api/ingest`) | `{ status: "duplicate", runId }` |
| `200` | Fleet snapshot saved | `{ ok: true, machineId, tenantId }` |
| `202` | Couldn't save now, queued for retry (`/api/ingest`) | `{ status: "queued", message }` |
| `400` | Invalid JSON, or `source` not allowed | Validation error |
| `401` | Bad key or bad signature | `invalid_signature` / auth error |
| `402` | Plan's run or agent cap reached | `{ error, message, usage, upgrade }` |
| `403` | Production run from an uncertified agent, or expired certificate | `{ error: "agent_not_certified" \| "cert_expired", … }` |
| `413` / `415` | Body too large / not JSON | |

### Settings

| Setting | Default | What it does |
| --- | --- | --- |
| `CLAUDE_WEBHOOK_SECRET` | not set (no check) | Signature secret for `/api/ingest/claude` |
| `COPILOT_WEBHOOK_SECRET` | not set (no check) | Signature secret for `/api/ingest/copilot` |
| `SLACK_WEBHOOK_URL` | not set | Where budget alerts from `/api/ingest` go |
| `LLM_CLASSIFIER_ENABLED` and an AI provider | off | AI task classification; otherwise the regex classifier is used |

## Dashboard / UI

Passive ingest has no page of its own. Ingested runs show up everywhere runs do: Overview, Runs, cost leaks, the control score, Run value, the savings report and agent certification. Fleet snapshots feed **Live Sessions** and **AI Consumption History**.

On the Admin tab, the **Connectors** catalog shows each webhook connector's ingest URL (for example `/api/ingest/copilot` on the GitHub Copilot card).

## Changes by file

Where the ingest code lives:

| File | What it does for ingest |
| --- | --- |
| `server.js` | The five route handlers, `normalizePayload`, `readRawBody`, `enforceProductionCertGate`, `checkAutoRevoke`, `dispatchBudgetAlert`, the duplicate-key cleanup and retry-queue timers |
| `src/connectors.js` | `normalizeGenericRun`, `normalizeClaudeRun`, `normalizeCopilotRun`, `normalizeFleetSnapshot`, field cleaning |
| `src/ingest/verify.js` | `verifyHmacSignature` |
| `src/validation.js` | `SCHEMAS.ingest` (the allowed `source` values) |
| `src/certification/danger-classifier.js` | `classifyManifest`: tool danger levels on every run |
| `src/privacy.js` | `sanitizeFleetSnapshot`, prompt-text removal before queueing |
| `src/ml/llm-classifier.js` | `classifyTaskWithLLM` |
| `src/cost/claude.js` | Cost of fleet session increases |
| `src/saas-store.js` + `src/stores/*` | `upsertTenantRuns`, `checkAndSetIdempotencyKey`, `pruneIdempotencyKeys`, `saveFailedIngest`, `listPendingFailedIngests`, `markFailedIngestAttempt`, `upsertFleetSnapshot`, `upsertDailyUsage`, `upsertDailyUsageByModel` |
| `src/sdk/index.js` | The Node.js SDK agents use to send runs |

## Tests

What's tested today:

- `test/connectors.test.js`: the three normalizers, field mapping, cleaning bad values and defaults.
- `test/idempotency.test.js`: the duplicate-key store. First key is new, the same key again is a duplicate, keys are per workspace, two at once only let one through, keys expire.
- `test/ingest-resilience.test.js`: the retry queue store. Queueing, when an item is due, success, the backoff and giving up after 5 attempts.
- `test/certification.test.js`: tool danger levels (`classifyTool`, `classifyManifest`), which every normalizer uses.
- `test/fleet-snapshots.test.js`, `test/daily-usage-rollup.test.js`: fleet snapshots, counting only increases, the daily rollup.
- `test/security.test.js`: the `source` value check (`SCHEMAS.ingest`).
- BDD: `features/08_webhook_ingest_invalid_json.feature`, `09_ingest_idempotency_key.feature`, `10_failed_ingest_retry_queue.feature`.

Not tested yet, and worth adding:

- The signature check on `/api/ingest/claude` and `/api/ingest/copilot`: right, wrong and missing signature, with and without a secret.
- The production certification gate and auto-revoke, through the routes.
- A failed save on `/api/ingest` returning `202` and being saved on retry, end to end.
- An ingest run blocked by the plan or certification, then sent again after the fix (see limitation 2).
- Each route's "after save" steps (audit, budget alert, classification) to pin down which route does what.

## Known limitations

Ingest issues I found:

1. **Cost is whatever the agent reports.** Unlike the gateway and fleet ingest, the run routes don't calculate cost from tokens and `config/pricing.json`. An agent that sends tokens but no cost is recorded at **$0**, and a wrong cost is stored as sent.
2. **A refused run can be wrongly marked as a duplicate.** On `/api/ingest` the duplicate key is recorded **before** the plan and certification checks. If a run is refused (402 or 403) and sent again after the agent is certified or the plan upgraded, within 24 hours, it gets `duplicate` back and is **never saved**.
3. **The run routes don't behave the same.** Only `/api/ingest` has duplicate keys, the retry queue, the audit entry and budget alerts. On the other three, a failed save returns an error and the report is lost, unless the sender retries.
4. **Missing `environment` means production.** An agent that doesn't send it is blocked until certified. That's the safe choice, but it often surprises new users.
5. **Signatures are optional and shared.** The Claude and Copilot secrets are one value per server, not per workspace, and with no secret set anyone with a valid key can post. `/api/ingest` and `/api/ingest/generic` have no signature check at all; they rely on the Agent Prism key alone.
6. **Fleet ingest has no plan check**, so it doesn't count toward the plan's run cap and isn't gated by plan.
7. **The first snapshot of a session is never counted.** That's on purpose, to avoid counting old activity as today's, but tokens used between the session starting and the collector's first push are lost.
8. **Fleet daily usage uses the server's UTC date**, so a developer's late-evening work in some time zones counts toward the next day.
9. **`/api/ingest/generic` ignores `messages` when classifying.** The code means "use `messages`, or else `prompt`", but because of operator precedence it always builds one message from `prompt`. A report with `messages` and no `prompt` is classified from empty text, so its task type isn't improved.
