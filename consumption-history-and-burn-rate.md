# Spend (consumption) history, calendar range and burn rate

## Introduction

Two dashboard views show what developers' AI coding tools are using. The data comes from the fleet collector (`collector.js` on each machine) or the local Live Sessions scan:

- **Spend History** (the `cost-history` tab, called "AI Consumption History" in the original plan). It shows daily consumption value by developer, team and model, with a trend chart, a developer leaderboard and plan-utilisation advice. The range can be **Today**, **7 Days**, **30 Days**, **All Time**, or **any calendar range** chosen with two date fields. It refreshes every 30 seconds.
- **Live Sessions.** Each running session's tokens, context and **burn rate**, an estimated **$ per hour** pace.

Both values are **API list-price estimates**, not what the customer is billed. Claude Code or Codex on a subscription isn't billed per token. Because people kept reading burn rate as "what my current prompt costs", both views carry always-visible notes, not just hover text.

This document replaces two openspec items in our format: the change `openspec/changes/consumption-history-calendar-and-burn-rate-explainer/`, and the Live Sessions metric glossary (`openspec/changes/archive/2026-08-24-live-sessions-business-context/`, spec `openspec/specs/live-sessions-metric-glossary/spec.md`). It also describes the views themselves, which had no design document. The fleet collector and fleet ingest are a separate topic.

Example: a developer's Claude Code session started at 09:00. By 10:30 it has used tokens worth $4.20 at list price, so its burn rate is $4.20 ÷ 1.5 h = $2.80/hr. Hovering shows "$4.20 in 1h 30m — burning $2.80/hr, on pace for $7.00 if this continues another hour (API-list-price estimate — not billed cost on a subscription plan)". In Spend History, a manager picks **3 March – 17 March**, and the view calls `/api/usage/daily?from=2026-03-03&to=2026-03-17`.

Both views are described as they currently behave.

## What we have today

| Piece | Where | What it does |
| --- | --- | --- |
| Range state | `public/app.js`: `costHistoryRange` (`today`, `7d`, `30d`, `all`, `custom`), `costHistoryCustomRange` | The selected range |
| `costHistoryDateRange(preset)` | `public/app.js` | Turns the range into `{ from, to }` UTC dates |
| `renderCostHistoryView`, `loadCostHistory`, `startCostHistoryPolling` | `public/app.js` | Header with buttons and date fields, data load, 30-second refresh (paused while the plan form is open) |
| `aggregateCostHistory`, `renderCostHistoryContent` | `public/app.js` | Daily trend, developer leaderboard, by team, by model, plan utilisation (see [plan utilisation](plan-utilization-recommendation.md)) |
| `GET /api/usage/daily` | `server.js` | Rows from `daily_usage_rollup`, by model from `daily_usage_by_model`, plus total, today and by-team sums |
| Burn rate | `server.js` (`/api/fleet/sessions` and the local scan) | `costUsd ÷ (lastActivity − startTime)` per session |
| `burnNarrative(s)`, `renderMetricGlossary(mode)` | `public/app.js` | Hover sentence; always-visible glossary under the stats |
| Notes | `.ls-subtitle` (Spend History), `.ls-metric-note` (Live Sessions) | Visible "estimate, not billed cost" text |

## What it does

- Lets a manager report AI consumption for any date range, by developer and team.
- Shows how fast each running session is spending, and where it's heading if it carries on.
- Makes clear, without hovering, that these are list-price estimates, not invoices.

## Where the data comes from

| Data | Source |
| --- | --- |
| Daily rows (developer, date, team, input, output and cache tokens, cost, sessions) | `daily_usage_rollup`, filled by `/api/fleet/ingest` from the **increase** in each session's totals, priced with `computeClaudeCost`, dated by UTC day |
| By model | `daily_usage_by_model`, the same source |
| Team | Developer → team assignments (`POST /api/tenant/team-assignments`) |
| Session cost and burn rate | Each session's lifetime token totals, priced with Claude rates; start and last-activity times from the collector |

## How it works

### 1. Ranges

| Range | `from` – `to` (UTC) |
| --- | --- |
| Today | today – today |
| 7 Days | today − 6 – today |
| 30 Days | today − 29 – today |
| All Time | no filter |
| Custom | both date fields, applied only when **both** are set and end ≥ start; otherwise nothing changes |

Choosing a preset or a custom range re-draws the view, highlights the active control and reloads. The 30-second refresh always reloads the current range, including a custom one.

### 2. `GET /api/usage/daily?from=&to=`

Both dates are optional and must be `YYYY-MM-DD`. It returns:

- `rows`: per developer and day;
- `byModel`: `[{ model, costUsd, tokens }]`;
- `summary`: `{ totalCostUsd, todayCostUsd, byTeam }`.

### 3. Burn rate

```text
elapsed   = lastActivity − startTime, at least 1 minute
burn rate = session cost so far ÷ elapsed hours
```

If there are no start and end times (an older collector), burn rate is `null` in Fleet Mode and the hover text says "restart collector.js to see burn rate". The single-machine scan returns `0` instead of `null` in that case. For a running session, the hover text projects **cost + one hour at that rate**. For a finished one, it gives the average. The colour tone is warn at $5/hr or more. The Live Sessions total burn adds up the rates of active sessions.

### 4. Always-visible notes

- **Spend History header:** "Estimated API-list-price consumption value… not billed per-token… won't match your provider's actual bill."
- **Live Sessions glossary:**
  - Burn Rate: "an estimated $/hr spend pace — not the cost of your current or last prompt, and not a billed amount on subscription plans";
  - Active sessions, Tokens and Context have their own lines;
  - single-machine view only: "Idle → Cost History".

## API

| Method and path | Auth | Returns |
| --- | --- | --- |
| `GET /api/usage/daily?from=&to=` | Tenant | `{ rows, byModel, summary }` |
| `GET /api/fleet/sessions` | Tenant | Machines and sessions, each with `costUsd` and `burnRatePerHour` |

## Dashboard / UI

- **Spend History:**
  - header: auto-refresh checkbox, Today, 7 Days, 30 Days, All Time, from and to date fields, Refresh;
  - content: total and today, trend chart with hover, developer leaderboard, by team, by model, plan utilisation.
- **Live Sessions:** sessions table with "Burn Rate (est.)", hover sentences, and the glossary under the stats bar.

## Changes by file

Code for both views:

| File | What it does |
| --- | --- |
| `public/app.js` | Range state and controls, loading, aggregation, burn-rate text, glossary |
| `public/styles.css` | `.ch-calendar-range`, `.ch-date-input`, `.ls-metric-note`, `.ls-metric-glossary` |
| `server.js` | `/api/usage/daily`, burn rate in `/api/fleet/sessions` and the local scan, fleet rollups |
| `src/stores/*` | `getDailyUsageRollup`, `getUsageByModel` |
| `openspec/changes/consumption-history-calendar-and-burn-rate-explainer/*` | Original change, replaced by this document |

## Tests

None automated for the range controls, burn rate or notes. The openspec tasks record manual checks:

- the notes show without hovering;
- custom single-day and multi-day ranges call the matching `from`/`to`;
- end-before-start is refused;
- a custom range survives the 30-second refresh.

## Known limitations

Things that can mislead people reading these views:

1. **Burn rate is a lifetime average, not recent pace.** It divides **all** the session's cost by the time from its first to its last activity. A session resumed after a long pause shows a very low rate, and a short burst shows a very high one. The original plan described it as "a rate estimate based on recent activity", which isn't what the formula does. The "on pace for…" projection assumes the same rate for another full hour.
2. **Every session is priced at Claude rates.** Fleet rollups and burn rate use `computeClaudeCost` whatever the tool, so Codex or other non-Claude sessions get Claude prices, or the default Sonnet price for unknown models.
3. **Per-developer spend is visible to everyone in the workspace.** `/api/usage/daily` returns rows per developer to any signed-in user, including viewers. The Copilot views, by contrast, hide per-person detail from non-admins (see [Copilot connector](copilot-connector-and-sync.md)).
