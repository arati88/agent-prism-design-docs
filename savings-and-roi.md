# Savings report and ROI

## Introduction

Agent Prism answers "is this worth it?" in four places:

- **Savings report** (Reports → Savings report). This is for the CFO. It shows money **Saved** (measured against each agent's own first 14 days), money **Avoided** (gateway requests a budget stopped), money **Still to capture** (waste found but not fixed) and **Value delivered** (hours of work × hourly rate). Only Saved and Avoided are added to the total. Once the monthly cost of Agent Prism is entered, it also shows the return as a multiple ("8.2× what you paid"). It downloads as PDF or CSV.
- **Team report** (Reports → Team report). What each team spent on AI (their agents, their developers' coding assistants and Copilot), and the value of the work their agents completed.
- **ROI summary** (`/api/roi-summary`). The Overview's "recoverable per month" figure: projected monthly spend, how much of it is leaking, and how much Token Coach says can be recovered.
- **Session ROI** (`/api/metrics/roi`, `/api/metrics/summary`). Completed sessions × hours a person would have needed × hourly rate, compared with what the agents cost.

Return on Copilot has its own design document ([copilot-roi](copilot-roi.md)). The user guide for the savings report is [docs/SAVINGS-REPORT.md](../SAVINGS-REPORT.md).

Example: in its first 14 days, `invoice-bot` spent $60 on 40 runs, 30 of them successful, so its baseline is **$2.00 per task**. In October it spent $90 on 120 runs, 100 of them successful: $0.90 per task.

```text
Saved   = ($2.00 − $0.90) × 100 tasks          = $110
Avoided = 12 blocked minutes × $0.75 avg/run   = $9
Total   = $119;  Agent Prism cost $50 → 2.4×
Value delivered (not in total) = 100 tasks × 2 h × $85 = $17,000
```

These numbers are described as the current code calculates them.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/savings-report.js` | `planBaselines`, `buildSavingsReport`, `savingsReportCsv`. `BASELINE_DAYS = 14`, `BASELINE_MIN_SUCCESSES = 5` |
| `src/reports/team-report.js` | `buildTeamReport`, `teamReportCsv` |
| `src/store.js` | `buildRoiSummary` |
| `src/roi.js` | `computeRoi` (session ROI) |
| `src/ml/value-report.js` | Run value report, which supplies "Still to capture" (see [machine learning](machine-learning.md)) |
| `config/pricing.json` → `fte` | `avg_human_hours_per_task` (2) and `loaded_hourly_rate_usd` (85), the defaults for value delivered |
| `server.js` | Routes, `captureAgentBaselines`, `recordSpendBlock` calls in the gateway, the hourly baseline job |
| Storage | `agent_baselines`, `spend_blocks`, `savings_settings` (Postgres), with matching file-store functions |
| `public/app.js` | Savings report and team report pages, the Overview recoverable figure, settings for the monthly cost |

## What it does

- Gives finance one page with money split by how certain it is: measured, avoided, found, and productivity.
- Compares each agent with **itself**, so a cheap agent and an expensive one are both measured fairly.
- Counts losses: agents that got more expensive reduce the total.
- Shows the return on Agent Prism once its cost is entered.
- Splits AI spend and value by team, for chargeback.

## Where the data comes from

| Number | Source |
| --- | --- |
| Runs, successful runs and cost for each agent and day | `agent_runs` in the chosen period (UTC days). A success is `success` or `completed` |
| Baseline | Each agent's runs in the 14 days from its **first run**, recorded once in `agent_baselines` |
| Blocked requests | `spend_blocks`: one row for each workspace, provider, API key and minute in which the gateway refused a request because the monthly budget was reached, with a count of requests |
| Still to capture | Run value report: low-value spend in the last 30 days (up to 5,000 runs) |
| Hours per task, hourly rate | `fte` in `config/pricing.json`, or `hoursPerTask` and `hourlyRate` in the request |
| Agent Prism monthly cost | `savings_settings`, entered by an admin |
| Team spend | Agent runs by their `team`; coding-assistant usage by developer → team assignment; Copilot AI credits × the contracted credit price, by GitHub login → team |
| Sessions | `sessions` (see the sessions document) |

## How it works

### 1. Recording baselines

`captureAgentBaselines` runs **every hour on Postgres** and every time the savings report is opened. For each agent that doesn't have a baseline yet:

| Situation | Result |
| --- | --- |
| Less than 14 days since its first run | Pending: "collecting", with the date it will be ready and the days left |
| 14 days passed, fewer than 5 successful tasks in them | Pending: "insufficient" |
| 14 days passed, 5 or more successful tasks | Saved: runs, successes, cost, and **cost per success = all cost in the window ÷ successful tasks** |

A baseline is never changed after it's saved (`on conflict do nothing`).

### 2. Saved

For each agent with a baseline, only days **after** the baseline window, within the report period, count:

```text
cost per task now = spend ÷ successful tasks
saved             = (baseline cost per task − cost per task now) × successful tasks
change %          = (now − baseline) ÷ baseline
```

If the agent had runs but no successful task, all of its spend counts as a loss (`saved = −spend`). Saved is the sum over agents, including negative ones. The report also shows how many agents improved and how many got worse.

### 3. Avoided

```text
avoided = number of blocked minutes × the period's average cost per run (all agents)
```

A blocked request is recorded when the gateway refuses it because the month's spend has reached `monthlyBudgetUsd` on the connector, or `GATEWAY_MONTHLY_BUDGET_USD`. That happens on `/v1/messages`, `/v1/responses` and `/v1/azure/*`. Requests are grouped by API key and minute, so a retry loop counts once a minute. The report shows the minutes counted, the total requests refused, the value used for each, and the minutes for each provider.

### 4. Still to capture

This is the Run value report's low-value spend over the last 30 days, the basis ("standard rules" or "your team's 👍/👎 labels"), and the top 3 agents with their reason and fix. It's **not** added to the total.

### 5. Value delivered

```text
value delivered = successful tasks × hours per task × hourly rate
```

It's **not** added to the total. The report calls it productivity, not cash.

### 6. Headline and return

```text
total               = saved + avoided
Agent Prism cost    = monthly cost × days in the period ÷ 30.44
return (multiple)   = total ÷ Agent Prism cost   (only when a monthly cost is set)
```

The period defaults to the first of this month up to today.

### 7. Team report

For each team: agent runs, successful and failed runs, success rate, people, agent spend, coding-assistant spend, Copilot credits and spend, total spend, and then:

```text
hours saved = successful agent runs × hours per task
value       = hours saved × hourly rate
net value   = value − total spend
ROI         = value ÷ total spend
cost per successful run = agent spend ÷ successful runs
```

Coding-assistant and Copilot usage add to spend but **not** to value. Copilot uses GitHub's rolling 28 days, not the report dates. Copilot spend is `null` until a credit price is set. Anything without a team goes to "Unassigned".

### 8. ROI summary (`/api/roi-summary`)

Built from the latest 10,000 runs:

```text
projected monthly spend       = Token Coach's projected monthly cost
projected monthly leak        = cost of all cost-leak runs × (projected monthly ÷ total cost)
projected monthly recoverable = sum of Token Coach suggestions' monthly savings
recoverable %                 = recoverable ÷ projected spend, at most 100%
```

It also returns the leak count, the top 5 leaks and the top 5 recommendations, with a note that the savings overlap. The Overview shows "recoverable per month" from this. If the call fails, it shows **$14 × the number of leaks** instead.

### 9. Session ROI (`/api/metrics/roi`, `/api/metrics/summary`)

```text
hours saved      = completed sessions × avg_human_hours_per_task
FTE equivalent   = hours saved × loaded_hourly_rate_usd
net savings      = FTE equivalent − agent cost (sessions' cost by platform)
ROI multiplier   = FTE equivalent ÷ agent cost  (null when cost is 0)
```

`/api/metrics/roi?window=30d` gives one window. `/api/metrics/summary` gives today and the last 30 days. Both first mark idle sessions as timed out.

## API

| Method and path | Auth | What it does |
| --- | --- | --- |
| `GET /api/reports/savings?from=&to=&hoursPerTask=&hourlyRate=&format=csv` | `reports.view` (admin) | Savings report as JSON or CSV |
| `GET /api/tenant/savings-settings` | Tenant | `{ agentPrismCostMonthlyUsd }` |
| `POST /api/tenant/savings-settings` `{ agentPrismCostMonthlyUsd }` | `settings.manage` (admin) | Sets the monthly cost (0 to 10,000,000, or `null`). Audited |
| `GET /api/reports/teams?from=&to=&hoursPerTask=&hourlyRate=&format=csv` | `reports.view` (admin) | Team report as JSON or CSV |
| `GET /api/roi-summary` | Tenant | ROI summary |
| `GET /api/metrics/roi?window=` | Tenant | Session ROI for one window |
| `GET /api/metrics/summary` | Tenant | Active agents, cost and session ROI for today and the last 30 days, and the pricing-stale flag |

Assumptions must be numbers from 0 to 1,000 hours and from 0 to $10,000 an hour. `from` must be on or before `to`. CSV cells that start with `=`, `+`, `-`, `@`, a tab or a carriage return get a leading `'`, so a spreadsheet can't run them as formulas.

## Dashboard / UI

- **Savings report.** Cards for Saved, Avoided, Total, Still to capture and Value delivered. A table for each agent (baseline, now, tasks, spend, saved), agents still collecting with their ready date, avoided by provider, all assumptions printed, the return once the monthly cost is set, and **Save as PDF** and **CSV**.
- **Team report.** One row per team with spend by source, hours saved, value, net value and ROI, and a CSV download.
- **Overview.** "Recoverable per month" from the ROI summary.

## Changes by file

Savings and ROI code:

| File | What it does for savings and ROI |
| --- | --- |
| `src/savings-report.js` | Baselines, savings report, CSV |
| `src/reports/team-report.js` | Team report, CSV |
| `src/store.js` | `buildRoiSummary` |
| `src/roi.js` | Session ROI |
| `src/ml/value-report.js` | Still to capture |
| `config/pricing.json` | Hours per task, hourly rate |
| `server.js` | Routes, baseline capture, spend-block recording, hourly job |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | Baseline stats, daily stats, spend blocks, settings, team stats |
| `public/app.js` | Report pages and the Overview figure |

## Tests

What's tested today:

- `test/savings-report.test.js`: a baseline is recorded after 14 days with 5 or more successes, and otherwise explained; Saved is measured per task against the baseline, Avoided is counted once per key per minute, and Still to capture stays out of the total; runs inside the baseline window don't count, and no successes counts as a loss; CSV formulas in agent names are neutralised; end to end, a baseline is captured, savings measured and budget blocks counted.
- `test/team-report.test.js`: spend by source, value and ROI per team; an unknown Copilot rate stays unknown, never $0; CSV with a total row and formula protection; the route with custom assumptions, CSV download and input checks.
- `test/roi.test.js`: positive and negative net savings; no multiplier when cost is 0; only completed sessions count; assumptions are always returned.
- `test/roi-helpers.test.js`: hours and dollar formatting, with unknown values shown as n/a.

Not tested yet, and worth adding:

- `buildRoiSummary`, apart from the empty case: projected leak and recoverable figures.
- The return multiple and the prorated Agent Prism cost.
- Baseline capture on the file store and after data retention.

## Known limitations

Reasons to treat these numbers with care:

1. **"Saved" credits Agent Prism with any drop in cost.** Comparing an agent with its first 14 days doesn't separate what Agent Prism did from anything else: a provider price cut, a cheaper model, an edit to `config/pricing.json`, or the agent being given easier tasks. An agent tested cheaply during onboarding, then used for real work, shows a loss. A baseline can't be reset, so there's no fix for a bad one.
2. **Value delivered and ROI assume every success is 2 hours of a person's work.** A 3-second classification run and a long research run both count as 2 hours at $85. The savings report keeps this out of the total, but the team report's value, net value and ROI, and the session ROI's net savings and multiplier, are built entirely on it. The defaults are the same for every client unless someone changes `pricing.json` or the query.
3. **The ROI summary inherits Token Coach's inflated projection.** Projected monthly spend is total cost × 30 whatever the date range (see [dashboard and analytics](dashboard-and-analytics.md)), and the leak and recoverable figures are scaled the same way. Recoverable is capped at 100%, but it's still described as an "upper bound".
4. **Three different ROI formulas.** The savings report counts runs, session ROI counts completed sessions, and Return on Copilot uses lines and prompts. The same work can be counted differently, and the screens don't say which formula they use.
