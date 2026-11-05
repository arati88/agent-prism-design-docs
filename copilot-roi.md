# Return on Copilot: how much work each Copilot seat gets done for what it costs

## Introduction

Agent Prism shows what GitHub Copilot costs: seats, AI credits and spend per model. It doesn't show whether that money is worth it. Many teams spend most of their AI budget on Copilot, so the question they ask is: **"For every $1 we pay for Copilot, how much work do we get back?"**

Copilot data doesn't arrive as agent runs. It comes from GitHub's API as seats, usage and billing, so the answer has to be built from that data.

Example: a developer's Copilot seat costs about $17 for 28 days. In that time Copilot added 900 lines of code to their editor and answered 60 prompts. At 50 lines an hour plus 2 minutes per prompt, that's about 20 hours of work, worth $1,700 at $85 an hour. So that seat turned $1 into about $100 of work.

## What we have today

With a GitHub org and token on the Copilot connector, Agent Prism already syncs:

| Data | Table | What it gives us |
| --- | --- | --- |
| Seats | `copilot_seats` | Who has Copilot, plan (Business or Enterprise), last activity |
| Per-user 28-day report | `copilot_usage` | AI credits each person used |
| Spend by model | `copilot_model_usage` | AI credits and $ per model per month |
| Budget caps | `copilot_budget_status` | Each seat's cap and how much is used |

So the **cost** side is covered. The **work** side is missing: we keep nothing about what Copilot actually did for each person.

GitHub already sends that data to us. The per-user 28-day report we download (`GET /orgs/{org}/copilot/metrics/reports/users-28-day/latest`) has activity fields per person. We only read `ai_credits_used` today. When that sync was built, the other fields weren't documented, so we didn't parse them. GitHub now documents them.

## What we'll build

- Read three more fields from the report we already download: lines added, prompts and acceptances
- Keep them per seat, next to the AI credits we already store
- Work out **spent**, **work done** and **$1 → $X** for each seat
- A **"Return on Copilot"** panel at the top of the **Overview** tab, the first thing people see. Everyone sees the totals; only admins see the per-developer table

We're keeping it simple :

- **No new GitHub call and no new token permission.** It's the same report, needing the same `read:org` scope.
- **Per developer (seat)**, plus one overall figure for the org.
- **GitHub's 28-day window**, the same as the report, not a calendar month. The panel shows the exact dates.
- Hours per line and per prompt are **assumptions** in `config/pricing.json`. They're shown next to the result and hot-reload.
- Copilot gets its own section, inside the Copilot panel where the rest of its data lives.

## Where the data comes from

The fields we'll read from each user's row, as named in GitHub's [Copilot usage metrics reference](https://docs.github.com/en/copilot/reference/copilot-usage-metrics/copilot-usage-metrics):

| Field | What GitHub says it is | Use |
| --- | --- | --- |
| `user_login` | GitHub username | Match to the seat |
| `ai_credits_used` | AI credits used in the period | Spend (already read) |
| `loc_added_sum` | Lines of code actually added to the editor: accepted completions, applied code blocks, agent and edit mode | **Work done** |
| `user_initiated_interaction_count` | Explicit prompts sent to Copilot. Opening the chat panel, switching modes and shortcuts don't count | **Work done** |
| `code_acceptance_activity_count` | Suggestions or code blocks accepted | Shown only, not valued |

What GitHub's notes mean for us:

- `loc_added_sum` counts code that **landed in the editor**, from completions and from agent edits. It does **not** say whether the code was kept, committed or merged.
- The per-user report has **no pull request counts**. It only has yes/no flags such as `used_copilot_cloud_agent`. So Copilot coding-agent PRs can't be valued per person yet.
- Reports exist from 2025-10-10 onward, and only when the organization's **Copilot metrics API** and **Copilot usage metrics** policies are on (see [When GitHub has no usage report](#when-github-has-no-usage-report)). Both already apply to the usage sync we have.

## How it's worked out

For each seat, over the report's 28 days:

```text
spent      = seat price for 28 days + cost of the AI credits it used
work hours = lines added ÷ lines per hour  +  prompts × minutes per prompt ÷ 60
work done  = work hours × hourly rate ($85, fte.loaded_hourly_rate_usd in pricing.json)
result     = work done ÷ spent          ("$1 → $X")
```

- **Seat price** comes from the seat's plan: Business $19 a month, Enterprise $39 a month, times 28 ÷ 30.44.
- **AI credit cost** = credits × `price_per_credit_usd`, which already exists in `pricing.json`. While that's not set (it's `null` today), only the seat price counts, and the panel says so.
- **A seat with no activity** spends its seat price and does $0 of work, so it shows **$1 → $0 ⚠**. That fits with the idle-seat flag the Copilot panel already has.
- **The overall figure** adds up every seat's spent and work done.

### When GitHub has no usage report

GitHub only builds the report when the organization's Copilot policies **Copilot metrics API** and **Copilot usage metrics** are on (for an organization in an enterprise, "Enabled everywhere"). The first report can take about a day to appear. Until then GitHub answers `404 No report available for this organization`.

We don't treat that as a failure:

- If an earlier report is stored, we keep showing it.
- If not, the section still appears with each seat and what it costs. Work done and the return show as "—" with "Waiting for GitHub's usage report", plus GitHub's reason and how to fix it.
- Work is **unknown**, not $0, so no seat is flagged ⚠ as wasted.
- The top of the Copilot panel shows the same reason, instead of the old "Seats & 28-day usage need an org + token on the Copilot connector", which was misleading when the org and token were fine.

`GET /api/copilot/usage` returns `200` with `reportAvailable: false` and `reportMessage`, and `roi.reportAvailable` is `false`.

### When GitHub can't be reached

If GitHub fails for any other reason (an expired token, an outage), seats and usage already stored are shown instead of an error. `GET /api/copilot/seats` and `GET /api/copilot/usage` return them with `syncError`, and the panel adds one line: "last GitHub sync failed (…), showing stored data". Only when nothing is stored yet is the failure shown as an error.

Only GitHub's literal "No report available" 404 counts as "no report yet". A plain 404 means a wrong org name or a token that can't see the org. That's shown as an error with a hint to check the GitHub Copilot card, not as advice to turn on policies. Return on Copilot is hidden when the seat list couldn't be loaded at all, so "$0, 0 seats" never appears as if it were data.

### Which Copilot connector is used

Saving the GitHub Copilot card adds a new connector rather than editing the existing one, and there's no way to delete one from the dashboard. The panel therefore uses the **most recently saved** Copilot connector that has an org and a token. It used to take the oldest, so a corrected org or token was saved but never used (found on the hosted instance). `test/copilot-connector.test.js` covers this, and that a member gets the totals but not the per-developer list.

### Starting assumptions

| Setting | Starting value | Meaning |
| --- | --- | --- |
| `lines_per_hour` | 50 | 50 lines of code Copilot added ≈ 1 hour of a developer's time |
| `minutes_per_prompt` | 2 | Each prompt saves about 2 minutes: looking something up, explaining code |
| `seat_price_monthly_usd` | Business 19, Enterprise 39 | List prices. Change if the org pays something else |

These are conventions, not measurements, so they should be agreed with Sandeep. They're deliberately on the cautious side because `loc_added_sum` counts code that may later be deleted, and prompts that led to code are counted twice (once as a prompt, once in its lines). In `pricing.json`, under the existing `copilot` block:

```json
"copilot": {
  "hourly_seat_rate": 0.054,
  "price_per_credit_usd": null,
  "seat_price_monthly_usd": { "business": 19, "enterprise": 39 },
  "value": { "lines_per_hour": 50, "minutes_per_prompt": 2 }
}
```

### A worked example

| Seat | Plan | Spent (28 days) | Lines added | Prompts | Work hours | Work done | Result |
| --- | --- | --- | --- | --- | --- | --- | --- |
| bob | Business | $17.48 | 0 | 0 | 0 | $0 | **$1 → $0** ⚠ |
| alice | Business | $17.48 | 400 | 30 | 8 + 1 = 9 | $765 | **$1 → $44** |
| carol | Enterprise | $35.87 | 2,000 | 90 | 40 + 3 = 43 | $3,655 | **$1 → $102** |

Overall: $70.83 spent → $4,420 of work, so **$1 → $62**. (AI credits not priced in this example.)

## What the API returns

The Copilot panel already calls `GET /api/copilot/usage`. Its response gains a `roi` object, so there's no new endpoint:

```json
"roi": {
  "reportStartDay": "2026-09-14",
  "reportEndDay": "2026-10-11",
  "days": 28,
  "total": { "seats": 3, "activeSeats": 2, "spentUsd": 70.83, "hours": 52, "workUsd": 4420, "valuePerDollar": 62.4 },
  "seats": [
    { "login": "bob", "plan": "business", "spentUsd": 17.48, "linesAdded": 0, "prompts": 0, "acceptances": 0, "hours": 0, "workUsd": 0, "valuePerDollar": 0, "active": false }
  ],
  "creditsPriced": false,
  "reportAvailable": true,
  "reportMessage": null,
  "assumptions": { "linesPerHour": 50, "minutesPerPrompt": 2, "hourlyRateUsd": 85, "seatPriceMonthlyUsd": { "business": 19, "enterprise": 39 } }
}
```

Seats are sorted with the lowest return first. `roi` is `null` only when no report has been synced and GitHub hasn't said why. When GitHub has no report, `reportAvailable` is `false` and work fields are `null` (see above). The response itself also gains `reportAvailable`, `reportMessage` and `syncError`.

## The dashboard section

A **"Return on Copilot"** panel at the top of the **Overview** tab, full width. It only appears when the workspace has a GitHub Copilot connector. The GitHub Copilot panel on the Admin tab keeps seats, budget cap and spend by model, with a link to it.

**Who sees what:** everyone sees the four tiles and the footnote. The per-developer table (who used Copilot, and what their seat cost) is for admins only, people with the `reports.view` permission. This is enforced on the server: for anyone else, `GET /api/copilot/usage` returns `roi.seats` and `records` empty, with `roi.seatsHidden: true`, and the panel says the breakdown is visible to admins.

As an admin:

```text
Return on Copilot
[ Spent (seats) $70.83 ]  [ Work done $4,420 ≈ 52 h ]  [ Return $1 → $62.40 ]  [ Seats used 2 of 3 ]

Developer   Spent    Work done                                    Result
bob         $17.48   No Copilot activity                          $1 → $0.00 ⚠
alice       $17.48   400 lines · 30 prompts ≈ 9 h ($765.00)       $1 → $43.77
carol       $35.87   2,000 lines · 90 prompts ≈ 43 h ($3,655.00)  $1 → $101.88

2026-09-14 → 2026-10-11 (GitHub's latest 28 days). Lowest return first; ⚠ = the
seat's work was worth less than it cost. Based on 50 lines of code per hour and
2 min per prompt, at $85.00/hour. Lines are code Copilot added to the editor, kept
or not. AI credits aren't priced yet (set price_per_credit_usd), so Spent is seat
cost only. Rates are estimates, set in config/pricing.json.
```

"Seats used" turns orange when some seats are unused, and a seat's result turns red with ⚠ when it returns less than $1 of work for each $1 it costs. Without a usage report, the tiles show "—" for work and return, and each seat says "Waiting for GitHub's usage report".

## Changes by file

| File | What changes |
| --- | --- |
| `src/copilot/github-api.js` | `fetchCopilotUserCredits` also returns `locAdded`, `prompts` and `acceptances` from the same rows |
| `db/migrations/026_copilot_usage_activity.sql` (new) | Adds `loc_added`, `prompts` and `acceptances` (integer, nullable, no default) to `copilot_usage`. Null means "synced before activity was read" |
| `src/stores/postgres-store.js` | Save and read the three new columns, plus the same columns in the startup schema patch. The file store saves them as they are |
| `src/jobs/copilot-usage-sync.js` | Usage stored before this change has no activity, so it's fetched again instead of being served as fresh |
| `src/jobs/copilot-seat-sync.js` | When GitHub fails and seats are stored, returns them with `syncError` instead of throwing |
| `src/cost/copilot.js` | New `computeCopilotSeatCost(plan, days)`, so the seat price goes through `src/cost/` like every other rate |
| `src/copilot/copilot-roi.js` (new) | `buildCopilotRoi({ seats, usage, reportStartDay, reportEndDay, seatCost, creditCost, assumptions, reportMissing })`, plus `workHours`, `reportDays` and `formatHours`. Pure, no I/O: `server.js` passes in the cost calculators |
| `config/pricing.json` | `seat_price_monthly_usd` and `value` added to `platforms.copilot` |
| `server.js` | `GET /api/copilot/usage` adds `roi`, `reportAvailable`, `reportMessage` and `syncError`, and falls back to stored usage when GitHub fails; per-developer data only for `reports.view`. `GET /api/copilot/seats` adds `syncError` |
| `public/app.js` | "Return on Copilot" panel on Overview (`loadOverviewCopilotRoi`), totals only for non-admins, the "no report" message, the "showing stored data" note, and GitHub's real error instead of "need an org + token" |
| `public/styles.css` | The Overview panel spans the full width |
| `test/copilot-roi.test.js` (new) | Unit tests |
| `test/features/copilot-roi.feature`, `test/features/steps/copilot_roi_steps.py` (new) | BDD scenarios and their steps |

## Tests

In `test/copilot-roi.test.js`:

- GitHub's report rows are read into lines added, prompts and acceptances; missing or non-numeric values count as 0.
- Business and Enterprise seats use their own prices, prorated to 28 days; unlisted plans are priced as Business.
- The worked example above: hours, work done, $1 → $X per seat and overall, lowest return first.
- A seat with no activity shows $0 of work and still costs its seat price.
- A seat with prompts but no lines is valued on prompts alone.
- AI credit cost is added only when `price_per_credit_usd` is set; otherwise `creditsPriced` is false.
- A report row with no matching seat (the person lost their seat mid-period) still counts, using the Business price.
- Nothing synced yet → `roi` is `null`; a seat that costs nothing has no ratio, never Infinity.
- GitHub has no report → seats and their cost still show, work and the return are unknown (`null`, not $0), and the reason is returned.
- `workHours`, `formatHours` and `reportDays` (strings or Postgres `Date` objects).
- Seat sync: GitHub failing returns stored seats with the reason; with nothing stored, it's an error.
- Usage sync: rows stored before activity was read are fetched again, then served fresh.

The BDD scenarios in `test/features/copilot-roi.feature` cover the same rules in plain language.

## Not in this version

- **Copilot coding-agent pull requests.** The per-user report has no PR counts. GitHub's repo-level or org-level reports might.
- **Whether code was kept.** GitHub counts lines added to the editor, not lines that survived to a merge.
- **ROI for other agents** (Claude, OpenAI and custom agents reporting runs), and one org-wide "$1 → $X" across agents and Copilot.
