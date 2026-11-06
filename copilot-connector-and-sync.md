# GitHub Copilot connector and sync jobs

## Introduction

The **GitHub Copilot connector** lets Agent Prism read a customer's Copilot organisation straight from GitHub's API. With an org login and an org-scoped token saved, it pulls four kinds of data:

| Data | GitHub source | Re-read at most every |
| --- | --- | --- |
| **Seats**: who has a Copilot seat, plan, last activity | `GET /orgs/{org}/copilot/billing/seats` | 15 minutes |
| **Usage per person**: AI credits, lines added, prompts, acceptances over the last 28 days | `GET /orgs/{org}/copilot/metrics/reports/users-28-day/latest` (NDJSON report files) | 6 hours |
| **Usage per model**: AI credits by model for one month, for the org and the top 5 users | `GET /organizations/{org}/settings/billing/ai_credit/usage` | 6 hours |
| **Budgets per person**: each user's AI-credit cap and how much of it is used | `GET /organizations/{org}/settings/billing/budgets?scope=user&user=` (one call per seat) | 15 minutes |

There's **no schedule**. Each sync runs when someone opens the screen that needs it, and only if the stored copy is older than the limit above. If GitHub fails, the last stored copy is shown with a warning.

This data feeds:

- the Copilot panel: seats, idle seats and AI credits;
- **Return on Copilot** (see [copilot-roi](copilot-roi.md));
- **budget breaches** (see [copilot-budget-breach](copilot-budget-breach.md));
- **model waste findings** (this document);
- the team report's Copilot column (see [savings and ROI](savings-and-roi.md)).

Copilot activity sent by webhook to `/api/ingest/copilot` is a separate path, covered in [passive ingest](passive-ingest.md).

Example: Acme saves its org `acme-eng` and a token. An admin opens the Copilot panel, and Agent Prism reads 120 seats from GitHub; 14 have had no activity for 45 days, so they're idle. The 28-day report shows 18,400 AI credits. The model view for this month shows that "Claude Opus 4.7", chosen by hand, used 62% of all credits. That's a **medium** finding: "make Auto the default".

This is how the connector works today.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/copilot/github-api.js` | `fetchCopilotSeats`, `fetchCopilotUserCredits`, `fetchCopilotModelUsage`, `fetchCopilotUserBudget`. 15-second timeout, API version header `2026-03-10`, error text scrubbed |
| `src/jobs/copilot-seat-sync.js` | `syncCopilotSeatsIfStale` (15 min), idle seats (no activity for 45 days, or never used) |
| `src/jobs/copilot-usage-sync.js` | `syncCopilotUsageIfStale` (6 h) |
| `src/jobs/copilot-model-usage-sync.js` | `syncCopilotModelUsageIfStale` (6 h, org plus top 5 users), `resolvePeriod` |
| `src/jobs/copilot-budget-sync.js` | `syncCopilotBudgetsIfStale` (15 min, 4 calls at a time), `isBudgetBreached`, `summarizeBudgets` |
| `src/copilot/model-usage.js` | `parseModelName`, `parseModelUsageText` (for pasted tables), `analyzeModelUsage` (findings) |
| `src/copilot/copilot-roi.js` | Return on Copilot (own document) |
| `server.js` | `POST /api/connectors` (org and token), `findCopilotApiConnector`, `findCopilotConnector`, `/api/copilot/seats`, `/usage`, `/budgets`, `/models`, `/models/import`, `buildCopilotModelReport` |
| Storage | `copilot_seats`, `copilot_usage`, `copilot_model_usage`, `copilot_budget_status`, each keyed by workspace and connector |

## What it does

- Shows how many Copilot seats a company pays for, and which ones nobody uses.
- Shows who uses AI credits, and how much, without anyone exporting reports by hand.
- Shows which models the credits went to, and points out wasteful patterns such as hand-picked premium models or one person using most of the pool.
- Makes GitHub's own per-user budget breaches visible on the dashboard.
- Keeps working through a GitHub outage, using the last good copy.

## Where the data comes from

**The connector.** `POST /api/connectors` with `provider: "github-copilot"`, `org` (the GitHub org login) and `apiKey` (an org-scoped token). The token is encrypted with the workspace's key; the org login isn't, because it isn't secret. The token needs:

| For | Scope or role |
| --- | --- |
| Seats | `read:org` or `manage_billing:copilot` |
| 28-day usage report | `read:org`, plus the org policies "Copilot metrics API" and "Copilot usage metrics" switched on |
| Model usage, budgets | Org administrator or billing manager (`admin:org`) |

**Which connector is used.** Saving the Copilot card adds a **new** connector each time. The **newest** one that has both org and token is used for seats, usage and budgets. Model usage can use any Copilot connector, because it can also be filled by hand.

**Prices.** AI credit cost is `credits × price_per_credit_usd` from `config/pricing.json`. That's `null`, not $0, until a price is set. Model rows from GitHub's billing API use GitHub's own `grossAmount`.

## How it works

### 1. Common pattern for every sync

1. Check the connector has an org and token. If not: `not_configured` (model usage returns stored rows instead).
2. Read the stored rows for this connector. If the newest `syncedAt` is within the limit, return them.
3. Otherwise call GitHub, **replace** all stored rows for this connector (delete, then insert), and return the new rows.
4. If GitHub fails and stored rows exist, return them with `syncError`. If nothing is stored, return the error (`502 copilot_sync_failed`).

### 2. Seats

All pages are read, 100 at a time, up to 500 pages (50,000 seats). For each seat: login, assigning team, plan type, last activity time and editor, last authentication, pending cancellation, seat created. A seat is an **idle candidate** when it has never been used or its last activity is more than 45 days old.

### 3. Usage per person (28 days)

The report endpoint returns download links. Each file is fetched **without** our token, because the links are signed, and parsed one JSON line at a time; a line that can't be read is skipped and counted. Kept fields: `user_login`, `ai_credits_used`, `loc_added_sum`, `user_initiated_interaction_count`, `code_acceptance_activity_count`, plus the report's start and end days. Rows stored before lines-added was read are fetched again rather than shown as zero.

If GitHub says "No report available" and nothing is stored, the panel explains how to switch on the Copilot metrics policies, and that the first report can take about a day.

### 4. Usage per model (one month)

`?month=YYYY-MM` (default: this month, UTC). It reads the org-level breakdown, then a breakdown for each of the **5 users with the most credits** (from the 28-day report). GitHub's several rows per model (one per product or SKU) are folded into one. If one user's call fails, the org-level rows are kept.

**Manual import** (`POST /api/copilot/models/import`, `settings.manage`) is for tokens that can't read billing. You send either `text` (the "Model used / AI credits used" table pasted from GitHub, or CSV/TSV; header and blank lines are skipped) or `items` (`[{ model, credits, login? }]`). It accepts at most 500 rows; duplicates are summed. It replaces that month's rows and is audited.

**Findings** (`analyzeModelUsage`). Patterns under 100 credits are ignored. "Auto: X" means Copilot chose the model; a bare name means a person picked it.

| Finding | When | Severity |
| --- | --- | --- |
| `dominant_manual_model` | One hand-picked model uses more than 50% of all credits | high above 80%, otherwise medium |
| `manual_over_auto` | Hand-picked credits are more than 5 × Auto credits (only when Auto rows exist) | medium |
| `outdated_model_version` | An older version of a model line uses more credits than the newer versions in the same org | low |
| `user_concentration` | With 3 or more users, one uses more than 30% of credits | high above 60%, otherwise medium |
| `user_pinned_model` | More than 50% of a user's own credits go to one hand-picked model | low |

Each finding gives credits, cost where it's known, and an action. On purpose, findings **don't** estimate a saving, because GitHub doesn't publish per-request credit multipliers.

### 5. Budgets per person

There's one call per seat login (from the seat sync), 4 at a time. The user's **effective** budget is used, and only if it's an AI-credit budget (`budget_product_sku = ai_credits`), so an Actions or Codespaces budget is never shown as a Copilot cap. A seat is **breached** when consumed ≥ cap, which is the same point at which GitHub blocks the user. The summary gives the cap, how many seats have a cap, how many are breached, and the breached seats, worst first.

### 6. Who sees what

Totals are shown to everyone in the workspace. **Per-person** detail (seat holders, each person's usage and cost, who breached a cap, per-user model use and findings that name a person) is shown only with `reports.view` (admin). Everyone else gets `seatsHidden`, `breachedHidden` or `perPersonHidden`.

## API

| Method and path | Auth | Returns |
| --- | --- | --- |
| `POST /api/connectors` `{ provider: "github-copilot", org, apiKey }` | `connectors.manage` | The new connector (secrets hidden) |
| `GET /api/copilot/seats` | Tenant (per-person: admin) | `org`, `totalSeats`, `idleCandidateCount`, `synced`, `syncError`, `seats` |
| `GET /api/copilot/usage` | Tenant (per-person: admin) | `roi` (Return on Copilot), `reportAvailable`, `reportMessage`, `syncError`, report days, `totalAiCreditsUsed`, `totalCostUsd`, `pricingConfigured`, `records` |
| `GET /api/copilot/budgets` | Tenant (per-person: admin) | `capUsd`, `cappedSeatCount`, `breachedCount`, `breached`, `totalSeats`, `synced`, `syncError`, `userFetchFailures` |
| `GET /api/copilot/models?month=` | Tenant (per-person: admin) | `period`, `source` (`github_api` or `manual`), the model table, `userModels`, `findings`, `pricingConfigured`, `syncError`, `userFetchFailures` |
| `POST /api/copilot/models/import` | `settings.manage` | `{ imported, skippedLines }` |

## Dashboard / UI

- **Agent Sources → GitHub Copilot**: org login and token.
- **Copilot panel**: seats and idle seats, AI credits and cost (or "price not set"), budget breaches, model breakdown with findings, an import box, and warnings when the last sync failed.
- **Overview → Return on Copilot** (see [copilot-roi](copilot-roi.md)).

## Changes by file

Where the Copilot sync code is:

| File | What it does for Copilot |
| --- | --- |
| `src/copilot/github-api.js` | GitHub API calls |
| `src/jobs/copilot-*-sync.js` | Four lazy syncs |
| `src/copilot/model-usage.js` | Model parsing, paste import, findings |
| `src/copilot/copilot-roi.js` | Return on Copilot |
| `src/cost/copilot.js`, `config/pricing.json` | Credit and seat prices |
| `server.js` | Connector save, connector choice, Copilot routes, model report |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | The four Copilot tables |
| `public/app.js` | Connector form and Copilot panels |

## Tests

What's tested today:

- `test/copilot-connector.test.js`: the most recently saved Copilot connector is used.
- `test/copilot-roi.test.js`: report fields read, missing values count as 0; seat cost by plan; ROI cases; a failed seat sync shows stored seats with the reason, and fails with nothing stored; usage stored before activity was read is fetched again.
- `test/copilot-model-usage.test.js`: model name parsing; version comparison; pasted tables and CSV; each finding, including healthy and edge cases; the billing call folds SKUs; HTTP errors surface; month checking; the sync reads the org and top users, keeps org rows when one user fails, returns manual rows without a token, and fails with nothing stored.
- `test/copilot-budget.test.js`: effective AI-credit budget read; no budget means no cap; other budgets ignored; errors surface; breach rule; summary; sync once then served fresh; one seat failing keeps the others; failure with nothing stored; not configured.

Not tested yet, and worth adding:

- Seat paging past 100 and the 500-page stop.
- Two requests arriving together for a stale sync (nothing stops them both calling GitHub).
- Budget sync for a large org against GitHub's rate limit (see limitation 2).

## Known limitations

Known problems with the Copilot sync:

1. **Nothing syncs unless someone opens the screen.** Budget breaches, new idle seats and usage only update when a person loads the Copilot panel. Nobody is alerted when a developer hits a cap or a seat goes idle.
2. **Budgets for a large org can hit GitHub's rate limit.** One call per seat every 15 minutes means a 2,000-seat org makes 2,000 calls on each refresh. GitHub's usual limit is 5,000 an hour, shared with the other syncs.
3. **A partly failed budget sync forgets the failed seats.** Stored rows are fully replaced by the seats that succeeded, so a seat whose call failed loses its last known breach, and the copy counts as fresh for another 15 minutes.
4. **Saving the connector again keeps the old one.** Each save adds a connector, and old ones, with their tokens and stored rows, are never removed. Changing the token means the new connector starts with nothing stored.
