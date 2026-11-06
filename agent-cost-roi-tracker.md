# Agent cost and ROI tracker (sessions → usage → ROI, and the `/dashboard` page)

## Introduction

The **agent cost and ROI tracker** was Agent Prism's first live data feature. An integration:

1. registers each agent **session**;
2. reports **usage** as it goes (tokens, Copilot seat hours, or a cost), which Agent Prism prices from `config/pricing.json`;
3. marks the session **completed**.

Agent Prism then shows, on a separate **`/dashboard`** page:

- **Active agents** by platform;
- **cost** for today and the last 30 days, by platform;
- **FTE hours saved**: completed sessions × hours a person would need;
- **net savings** and the **ROI multiplier**;
- which platforms are **connected**;
- a warning when prices haven't been checked for 90 days.

The page refreshes every 5 seconds.

This document brings together:

- the original design, `openspec/changes/agent-cost-roi-tracker/`, which this replaces in our format;
- the setup guide, [docs/agent-tracker-setup.md](../agent-tracker-setup.md);
- what the code does today.

The parts are documented in more detail elsewhere: [sessions and timeout](sessions-and-timeout.md), [cost and pricing](cost-and-pricing.md), [savings and ROI](savings-and-roi.md) (session ROI) and [passive ingest](passive-ingest.md) (webhooks).

Example: a CI job opens session `ci-4412` (`claude`) and reports two usage events of 1,500 input and 400 output tokens (Sonnet rates, $0.0105 each). It then sets `completed`. `/dashboard` shows 0 active agents, $0.021 today, 2 FTE hours saved, $170 of FTE cost against $0.021 of agent cost, and an ROI of about 8,095×.

This is how the tracker works in the current code.

## What we have today

| Piece | What it does |
| --- | --- |
| Session and usage routes | `POST /api/sessions`, `PATCH /api/sessions/:id`, `GET /api/sessions/active`, `POST /api/usage` (see [sessions](sessions-and-timeout.md)) |
| Metrics routes | `GET /api/metrics/cost`, `/roi`, `/summary` |
| `src/cost/*`, `src/pricing.js`, `config/pricing.json` | Pricing, the stale-price flag, timeout minutes, FTE assumptions |
| `src/roi.js` | `computeRoi` |
| `src/jobs/session-timeout.js` | Marks idle sessions timed out (30 minutes) |
| `public/dashboard.html` | The `/dashboard` page: polls `/api/metrics/summary` every 5 seconds |
| `server.js` → `requireBasicAuth` | HTTP Basic Auth on `/dashboard` only, using `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD` |
| `getPlatformStatus()` | "Connected" platforms, from server environment keys |
| Webhooks | `/api/ingest/claude` (`anthropic-signature` or `x-webhook-signature`), `/api/ingest/copilot` (`x-hub-signature-256`), `/api/ingest/generic` |

## What it does

- Tracks agent cost **while** sessions run, for platforms that don't go through the gateway.
- Turns agent work into a simple FTE comparison that a manager can read.
- Shows the assumptions next to the ROI, so the number can be questioned.

## Where the data comes from

| Data | Source |
| --- | --- |
| Sessions and their cost | `POST /api/sessions`, `POST /api/usage`, `PATCH /api/sessions/:id` |
| Prices, FTE assumptions, timeout | `config/pricing.json` (`platforms`, `fte`, `session_timeout_minutes`, `last_verified`) |
| "Connected" platforms | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GITHUB_TOKEN` in the server's environment |
| API key for the page | `localStorage["acp_api_key"]` in the browser |

## How it works

1. **Set up** (from the setup guide):
   - edit `config/pricing.json`;
   - set `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD` and the webhook secrets;
   - bootstrap a workspace once with `POST /api/bootstrap` (platform admin; a second call returns `409 already_bootstrapped`).
2. **Report**: open a session, send usage events, mark it completed (see [sessions](sessions-and-timeout.md)).
3. **Read**: `/api/metrics/summary` runs the session timeout, then returns active counts, cost for today and 30 days by platform, session ROI for both, the stale-price flag and platform status.
4. **Show**: `/dashboard` asks for Basic Auth, then reads the API key from `localStorage` and polls the summary every 5 seconds. When there's no key, it asks for one in a browser pop-up and saves it in `localStorage`.

## API

See [sessions and timeout](sessions-and-timeout.md#api) for every route. `GET /api/metrics/summary` returns:

```json
{ "activeAgents": { "total": 0, "byPlatform": {} },
  "cost": { "day": { "totalUsd": 0.021, "byPlatform": { "claude": 0.021 } }, "month": { … } },
  "roi":  { "day": { "completedSessions": 1, "fteHoursSaved": 2, "fteCostEquivalentUsd": 170, "agentCostUsd": 0.021, "netSavingsUsd": 169.98, "roiMultiplier": 8095.24, "assumptions": { … } }, "month": { … } },
  "meta": { "pricingStale": false, "pricingLastVerified": "2026-08-09", "platformStatus": { "claude": true, "openai": false, "copilot": false, "generic": true } } }
```

## Dashboard / UI

The **`/dashboard`** page has four cards (Active agents, Agent cost, ROI with its assumptions and negative values in red, Platform status) and a stale-price banner. Nothing in the main dashboard (`/`) links to it; it's only reached by typing `/dashboard`. None of these numbers appear in the main dashboard itself.

## Changes by file

Files involved:

| File | What it does |
| --- | --- |
| `server.js` | Routes, `requireBasicAuth`, `getPlatformStatus`, `/dashboard` serving with a CSP nonce |
| `src/roi.js`, `src/cost/*`, `src/pricing.js`, `src/jobs/session-timeout.js` | Calculations |
| `src/stores/*` | Session storage |
| `public/dashboard.html` | The page |
| `docs/agent-tracker-setup.md` | Setup guide |
| `openspec/changes/agent-cost-roi-tracker/*` | Original design, replaced by this document |

## Tests

`test/sessions.test.js`, `test/cost.test.js`, `test/roi.test.js` and `test/pricing.test.js` (see the linked documents). Nothing tests `/dashboard` itself or Basic Auth.

## How this differs from the original design

| Original design | Today |
| --- | --- |
| Next.js API routes, SQLite / Turso, Drizzle | Raw `node:http` in `server.js`; file store or Postgres |
| `config/pricing.yaml` | `config/pricing.json`, reloaded when it changes |
| Single operator, no multi-tenancy | Workspaces and API keys; the page still uses one key from the browser |
| Basic Auth on `/dashboard` **and `/api/`** | Basic Auth on `/dashboard` only. APIs use workspace credentials |
| Real usage from Claude and Copilot APIs | Usage is **reported** by the integration. Copilot data from GitHub is a separate feature ([Copilot connector](copilot-connector-and-sync.md)) |

## Known limitations

Problems with the tracker as it stands:

1. **The `/dashboard` page doesn't work for normal users.** It ignores the dashboard sign-in and reads an API key from the browser's `localStorage` (`acp_api_key`). Sign-in no longer puts one there, and older ones were revoked. After the Basic Auth prompt the page asks for an API key in a browser pop-up, so a user has to create a key in Admin → API keys and paste it. That keeps a long-lived key in the browser's storage, and it isn't linked from the main dashboard.
2. **The ROI is the most optimistic one in the product.** Each completed session counts as 2 hours of a person's work at $85, so a cheap automated session shows ROI in the thousands (see [savings and ROI](savings-and-roi.md)).
3. **Two dashboards that don't agree.** The main dashboard is built from runs; this page is built from sessions. The same agent can show different costs on each, and nothing explains the difference.
