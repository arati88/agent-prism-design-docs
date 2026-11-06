# Agent sessions, usage and session timeout

## Introduction

An **agent session** is one working period of an agent or assistant, for example one Claude Code session or one Copilot day. It's tracked separately from runs. A client:

1. **Opens** a session (`POST /api/sessions`).
2. **Adds usage** to it as it goes (`POST /api/usage`), as tokens, seat hours or a dollar amount. Agent Prism prices it and adds it to the session's cost.
3. **Updates its status** (`PATCH /api/sessions/:id`): running, idle, completed or error.

A session that sends nothing for **30 minutes** (`session_timeout_minutes` in `config/pricing.json`) is marked **timed out**.

Sessions feed `/api/metrics/cost` (cost by platform), `/api/metrics/roi` and `/api/metrics/summary` (session ROI: completed sessions × hours saved × hourly rate, see [savings and ROI](savings-and-roi.md)), and `/api/sessions/active` (how many are running now).

Sessions are **not** the same as the dashboard's **Live Sessions** view. That comes from the fleet collector (`/api/fleet/*`), which has its own document.

Example: a script opens session `sess_42` for `claude`. It reports usage twice: 20,000 input and 2,000 output tokens on `claude-haiku-4-5` ($0.03), then the same again, for a session cost of **$0.06**. It then sets the status to `completed`. The session ROI counts it as 1 completed session = 2 hours × $85 = $170 of value against $0.06 of cost.

This is how sessions work in the current code.

## What we have today

| Piece | What it does |
| --- | --- |
| `server.js` | `POST /api/sessions`, `PATCH /api/sessions/:id`, `GET /api/sessions/active`, `POST /api/usage`, `GET /api/metrics/cost`, `/roi`, `/summary`, `getWindowStart`, `computeCostByPlatform` |
| `src/jobs/session-timeout.js` | `runSessionTimeout()`: marks idle sessions timed out |
| `src/validation.js` | `createSession`, `updateSession`, `ingestUsage` schemas |
| `src/cost/*` | Prices the usage (see [cost and pricing](cost-and-pricing.md)) |
| `src/roi.js` | Session ROI |
| Storage | `agent_sessions` (migration `020`): workspace, id, platform, status, start, last seen, end, cost. The file store keeps a `sessions` list |
| `public/dashboard.html` | The older `/dashboard` page, which reads `/api/metrics/summary` |

## What it does

- Gives scripts and integrations without a gateway or ingest path a simple way to report cost as it happens.
- Shows cost by platform for today, the last 7 days and the last 30 days.
- Counts how many sessions are active right now.
- Closes forgotten sessions automatically, so "active" stays accurate.

## Where the data comes from

| Request | Fields |
| --- | --- |
| `POST /api/sessions` | `platform` (required: `claude`, `copilot` or `generic`), `session_id` (optional, up to 128 characters; made up if missing), `start_time` (optional) |
| `PATCH /api/sessions/:id` | `status` (required: `running`, `idle`, `completed` or `error`) |
| `POST /api/usage` | `session_id` (required), `platform` (`claude`, `copilot` or `generic`), `input_tokens`, `output_tokens`, `seat_hours`, `cost_usd` (all 0 or more), `model` |
| Timeout | `session_timeout_minutes` (30) in `config/pricing.json` |

## How it works

### 1. Lifecycle

```text
POST /api/sessions ─► running ◄──► idle
                         │  │
     PATCH completed/error  └─ no usage or update for 30 min ─► timed_out
                         ▼
               completed / error  (end_time set)
```

- **Open.** The session is created as `running`, with last seen set to the start time. If that `session_id` already exists in the workspace, the existing session is returned unchanged, so a retried open is safe.
- **Update.** Sets the status and last seen to now. `completed` and `error` also set the end time. A session id from another workspace is "not found".
- **Usage.** Adds the priced amount to the session's cost and sets last seen to now. Its status isn't changed.

### 2. Pricing usage (`POST /api/usage`)

| `platform` | Cost |
| --- | --- |
| `claude` | `computeClaudeCost` with input and output tokens and the model (no cache tokens) |
| `copilot` | `computeCopilotCost`: seat hours × hourly seat rate |
| `generic` or missing | `computeGenericCost`: the `cost_usd` sent |

The response is `{ session_id, costDelta, sessionCostUsd }`. `server.js` has an `openai` branch too, but the validation schema doesn't allow `openai`, so that branch is never reached (see limitation 1).

### 3. Timeout

`runSessionTimeout()` runs **at the start of** each `/api/sessions/active`, `/api/metrics/cost`, `/api/metrics/roi` and `/api/metrics/summary` request. There's no schedule. It sets **every** `running` or `idle` session, in **all** workspaces, whose last seen is older than 30 minutes to `timed_out`.

### 4. Windows and totals

| `window` | From |
| --- | --- |
| `day` | Midnight UTC today |
| `week` | 7 days ago |
| `30d` (default, also any other value) | 30 days ago |

A session is in a window if it **started or was last seen** inside it. Its **whole** cost counts. Cost is added up by platform.

### 5. Retention

Finished sessions (anything but running or idle) are deleted once their last seen is older than the plan's retention period. GDPR erasure deletes all sessions (see [audit, GDPR and SIEM](audit-gdpr-and-siem.md)).

## API

| Method and path | Auth | Returns |
| --- | --- | --- |
| `POST /api/sessions` | Tenant (not read-only) | `201 { session }` |
| `PATCH /api/sessions/:id` | Tenant (not read-only) | `{ session }`, or `404` |
| `GET /api/sessions/active` | Tenant | `{ total, byPlatform }` |
| `POST /api/usage` | Tenant (not read-only) | `{ session_id, costDelta, sessionCostUsd }`, or `404` |
| `GET /api/metrics/cost?window=` | Tenant | `{ window, windowStart, totalCostUsd, byPlatform }` |
| `GET /api/metrics/roi?window=` | Tenant | Session ROI (see [savings and ROI](savings-and-roi.md)) |
| `GET /api/metrics/summary` | Tenant | Active count; cost and ROI for today and 30 days; `meta.pricingStale`, `pricingLastVerified`, `platformStatus` |

Session object: `{ id, tenantId, platform, status, startTime, lastSeen, endTime, costUsd }`.

## Dashboard / UI

The main dashboard doesn't show agent sessions. The older `/dashboard` page (`public/dashboard.html`) polls `/api/metrics/summary` for active agents, cost and ROI by day and month. It's also in the Postman collection (`postman/agent-prism.postman_collection.json`).

## Changes by file

Session code lives in:

| File | What it does for sessions |
| --- | --- |
| `server.js` | Routes, windows, cost by platform |
| `src/jobs/session-timeout.js` | Timeout |
| `src/validation.js` | Request schemas |
| `src/cost/*`, `config/pricing.json` | Prices, timeout minutes |
| `src/roi.js` | Session ROI |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | `createSession`, `updateSession`, `listSessions`, `getActiveSessionCounts`, `applySessionTimeout` |
| `db/migrations/020_agent_sessions_prompt_analyses.sql` | `agent_sessions` table |
| `public/dashboard.html` | Older summary page |

## Tests

What's tested today:

- `test/sessions.test.js`: open creates a running session; active counts include running and idle; completing a session; unknown id returns null; usage adds up; timeout marks stale sessions; another workspace can't touch a session by id; opening the same id twice returns the existing one.
- `test/roi.test.js`: session ROI (see [savings and ROI](savings-and-roi.md)).

Not tested yet, and worth adding:

- `POST /api/usage` pricing for each platform, including that `openai` is rejected.
- Window filtering in `/api/metrics/cost`.
- Timeout across workspaces, and usage reported after a timeout.

## Known limitations

Session issues I found:

1. **OpenAI usage can't be reported.** The schemas only allow `claude`, `copilot` and `generic`, so `platform: "openai"` gets `400`, and the OpenAI pricing in `/api/usage` is never used. OpenAI users have to send `generic` with their own `cost_usd`.
2. **A session has no agent, team or model.** It only records a platform, so session cost and ROI can't be split by agent or team, and sessions aren't linked to runs.
3. **The timeout runs only when someone reads the metrics.** If nobody calls those four routes, sessions stay "running" forever. When someone does, the sweep updates **every workspace's** sessions, which is extra work on a busy installation.
4. **A timed-out session can still collect cost, and come back.** Usage after a timeout is added and updates last seen, but the status stays `timed_out`, so it's not counted as active. A `PATCH` can set it back to `running`. Timed-out and errored sessions never count as completed, so they add cost to the ROI but no value.
