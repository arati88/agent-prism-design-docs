# Copilot per-seat budget breach on the dashboard

## Introduction

Agent Prism gives teams one place to see what they spend on AI tools and where that money goes. GitHub Copilot is one of those tools. With a GitHub org connected, the Copilot panel on the Admin tab already shows the org's seats, how many AI credits people have used, and what that cost per model.

Copilot's pricing has two parts. Each person given Copilot takes up a paid seat, and on top of that every request to Copilot uses AI credits, with heavier models using more. To stop costs running away, GitHub lets an org owner cap how much each person can spend on AI credits in a month. Once someone reaches the cap, GitHub blocks Copilot for them.

This document describes how we'll read those caps from GitHub and show, on the Agent Prism dashboard, which seats have hit theirs.

## Seats in Copilot

A seat is one Copilot licence assigned to one person in a GitHub org. Copilot Business costs $19 per seat per month. The org pays for every assigned seat whether or not the person uses Copilot.

How seats get assigned: an org owner decides who gets Copilot under the org's Copilot settings. They can give it to all members, to people they pick one by one, or to everyone in chosen teams. A newly assigned seat is billed from that day, prorated for the rest of the month, and can take a little while to become active. When an owner removes someone, the seat stays usable until the end of the billing cycle and then goes away.

Each Business seat also comes with 1,900 AI credits a month ($19 worth at $0.01 a credit). These credits are pooled across the org rather than kept per person, so heavy users can draw on what light users don't use. Once the pool runs out, extra usage is charged at $0.01 a credit unless the org has turned paid usage off.

What Agent Prism does with seats today: when the Copilot panel is opened, it reads the org's seat list from GitHub (login, plan, last activity and so on), at most once every 15 minutes, and flags idle seats, meaning seats that have never been used or haven't been used in 45 days. Those are seats the org is paying for and could take back. Agent Prism only reads seats; it never assigns or removes them.

This change builds on that seat list. For each seat we already know about, we'll also find out its spending cap and how much of it has been used.

## How GitHub budgets work

The cap is a user-scoped budget, set under Billing and licensing → Budgets and alerts, with type "AI credits budget" and scope "Users". A few things about it that matter for us:

- It counts every AI credit the user uses, including the ones that come with the seat. An org-level budget only counts paid usage on top of that, so it's not what we want here.
- "Stop usage" is always on for user budgets. Hitting the cap always blocks the user.
- Anything used before the budget was created isn't counted.
- It resets at the start of the next billing cycle, or the user gets access back if the cap is raised.

## What we'll build

- A call to GitHub's budgets API for each Copilot seat
- A small table to keep the result per seat
- A new endpoint, `GET /api/copilot/budgets`
- A "Budget cap" section in the Copilot panel that lists seats over the cap

We're keeping it simple on purpose:

- The cap always comes from GitHub. Agent Prism doesn't set or store its own.
- We expect one cap for everyone. If seats end up with different caps, we show the highest one.
- No warnings at 80% or 90%. A seat is either under the cap or over it.
- Dashboard only, no email or Slack alerts.

## Where the data comes from

```text
GET /organizations/{org}/settings/billing/budgets?scope=user&user={login}
```

We checked the response against a real org. The parts we use look like this:

```json
{
  "budgets": [
    {
      "id": "1769d79f-…",
      "budget_type": "BundlePricing",
      "budget_product_sku": "ai_credits",
      "budget_scope": "user",
      "budget_amount": 400,
      "prevent_further_usage": true,
      "consumed_amount": 412.5,
      "user": "alice"
    }
  ],
  "effective_budget": { "id": "1769d79f-…", "budget_amount": 400, "consumed_amount": 412.5 }
}
```

`effective_budget` already has what we need: the cap that applies to the user and how much of it they've used. GitHub works out both, so our numbers line up with the point where GitHub blocks someone.

Budgets can also exist for Actions, Codespaces and so on, so we only accept one whose `budget_product_sku` is `ai_credits`.

The token has to belong to an org owner or billing manager and needs the `admin:org` scope, on top of the `read:org` and `manage_billing:copilot` the connector already asks for.

## How it works

For each seat, we read the cap and amount used from GitHub. A seat is breached when it has used all of its cap (`consumed >= cap`).

Results are refreshed at most every 15 minutes, checking 4 seats at a time. If GitHub fails, we show the last stored result, and this section loads separately so it never hides the rest of the Copilot panel.

## Changes by file

| File | What changes |
| --- | --- |
| `src/copilot/github-api.js` | New `fetchCopilotUserBudget(org, token, login)`. Returns the cap and consumed amount, or `null` if the user has no AI-credit budget |
| `src/jobs/copilot-budget-sync.js` (new) | `syncCopilotBudgetsIfStale` does the 15-minute check, fetches seats a few at a time, stores the result and handles failures. `isBudgetBreached` and `summarizeBudgets` hold the breach rule and build what the dashboard needs |
| `db/migrations/025_copilot_budget_status.sql` (new) | New table `copilot_budget_status` |
| `src/stores/postgres-store.js`, `src/stores/file-store.js`, `src/saas-store.js` | Save and read functions for the new table, and the table added to GDPR erasure |
| `server.js` | New `GET /api/copilot/budgets` endpoint, and `admin:org` added to the connector setup text |
| `public/app.js` | New Budget cap section, and `admin:org` added to the token hint |
| `test/copilot-budget.test.js` (new) | Unit tests |
| `test/features/copilot-budget-breach.feature`, `test/features/steps/copilot_budget_steps.py` (new) | BDD scenarios and their step definitions |
| `scripts/copilot-budget-cli.js` (new) | Small helper the BDD steps use to run the sync job with GitHub's replies faked |

### The new table

`copilot_budget_status` has one row per seat:

| Column | Type | Notes |
| --- | --- | --- |
| `tenant_id` | `TEXT` | References `tenants(id)`, deleted with the tenant |
| `connector_id` | `TEXT` | References `connectors(id)`, deleted with the connector |
| `login` | `TEXT` | GitHub login |
| `budget_usd` | `NUMERIC(12,4)` | Empty if the user has no AI-credit budget |
| `consumed_usd` | `NUMERIC(12,4)` | Defaults to 0 |
| `synced_at` | `TIMESTAMPTZ` | When we fetched it |

The primary key is `(tenant_id, connector_id, login)` and every sync replaces the connector's rows. Row-level security is set up the same way as `copilot_seats`. We don't store a "breached" flag; it's worked out from the two amounts when we read them, so it can't go stale. The table also goes into the Postgres startup schema and the file store, like the other Copilot tables.

### The new endpoint

`GET /api/copilot/budgets` uses the same tenant auth as the other Copilot endpoints and returns something like:

```json
{
  "org": "acme",
  "capUsd": 400,
  "cappedSeatCount": 25,
  "breachedCount": 1,
  "breached": [
    { "login": "alice", "budgetUsd": 400, "consumedUsd": 412.5, "percentUsed": 103 }
  ],
  "totalSeats": 25,
  "synced": true,
  "syncError": null,
  "userFetchFailures": 0
}
```

`capUsd` is null when nobody has a cap. `breached` only has the seats over the cap, worst first. If there's no connector with an org and token it returns 404 `not_configured`, and if GitHub fails with nothing stored it returns 502 `copilot_sync_failed`, matching the other Copilot endpoints.

### On the dashboard

The new section sits above "Spend by model" in the Copilot panel. It has two tiles, "Cap per seat" and "Seats over cap" (for example "1 of 25", in red when it's more than zero), and a table of the seats over the cap with how much they've used, their cap and the percentage. Under the table there's a short note that GitHub has blocked Copilot for those seats until the budget resets or the cap is raised.

If nobody is over the cap it just says so. If no budget is set in GitHub at all, it tells the admin where to set one. The footer shows where the data came from, any sync error, and a reminder that usage from before a budget was created isn't counted.

## Steps

1. Add `fetchCopilotUserBudget` to `src/copilot/github-api.js`, reusing `githubFetch`.
2. Add migration 025, the save and read functions for both stores, and the GDPR erasure entry.
3. Add the sync job in `src/jobs/copilot-budget-sync.js`.
4. Add `GET /api/copilot/budgets` next to the other Copilot endpoints in `server.js`. It gets the seat list from the existing seat sync, then the budgets.
5. Add the Budget cap section to the Copilot panel, loaded as its own request.
6. Update the connector setup text and token hint to mention `admin:org`.
7. Write the unit tests and BDD scenarios, then check it end to end against a real org.

## Testing

We test this at three levels: unit tests for the code, BDD scenarios for the behaviour, and a manual check against a real GitHub org. GitHub is mocked in the first two, so they run offline.

### Unit tests

`test/copilot-budget.test.js`, run with `npm test`. The GitHub sample response is a real one captured from a test org.

| # | Case | Expected |
| --- | --- | --- |
| 1 | AI-credit budget, cap $1, $1 used | Cap 1, consumed 1, correct API path called |
| 2 | No effective budget | No cap |
| 3 | Only an Actions budget | Ignored, no cap |
| 4 | GitHub returns 404 | Error with status 404 |
| 5 | Over, at, under the cap, and no cap | Over and at are breached; under and no cap aren't |
| 6 | Mix of seats over and under the cap | Only breached seats listed, worst first |
| 7 | No seat has a cap | Cap is empty, 0 breached |
| 8 | Sync twice within 15 minutes | Second sync doesn't call GitHub |
| 9 | One seat's call fails | Others kept, failure counted |
| 10 | Every call fails and nothing is stored | Error |
| 11 | Connector has no org or token | `not_configured` |

### BDD scenarios

`test/features/copilot-budget-breach.feature`, run with `npm run test:bdd`. These describe the feature the way an admin would, and run the real sync code through `scripts/copilot-budget-cli.js` with GitHub's replies faked. Each scenario gets its own temporary data folder.

| # | Scenario | Expected |
| --- | --- | --- |
| 1 | One seat over its $400 cap, one under | Cap $400, 1 seat over, the over-cap seat listed at 103%, the other not listed |
| 2 | A seat exactly at its cap | Counted as breached, listed at 100% |
| 3 | No budget set in GitHub | No cap, 0 seats over |
| 4 | Only an Actions budget | Not treated as a Copilot cap |
| 5 | Several seats over the cap | Listed worst first |
| 6 | Last sync 5 minutes ago | Stored result reused, GitHub not called |
| 7 | Cap raised in GitHub, last sync 16 minutes ago | GitHub called again, new cap shown, 0 seats over |
| 8 | One seat's call fails | Other seats still reported, 1 seat couldn't be checked |
| 9 | GitHub down, earlier result stored | Stored result shown with a sync error |
| 10 | GitHub down, nothing stored | Sync fails |

### Manual check against a real org

On an org with Copilot Business and a token that has `admin:org`:

| # | Check | Expected |
| --- | --- | --- |
| 1 | No user budget set | Section says no per-seat budget is set |
| 2 | Set a $1 user budget and use Copilot until GitHub blocks it | Cap $1.00, 1 seat over cap, user listed at 100% |
| 3 | Raise the budget above what's been used | After the next sync, 0 seats over cap |
| 4 | Delete the budget | Back to "no per-seat budget" |
| 5 | Remove `admin:org` from the token | Budget section shows an error, the rest of the panel still loads |
| 6 | `npm test` and `npm run test:bdd` | No new failures |

## Rollout

Migration 025 runs on startup when `RUN_MIGRATIONS_ON_STARTUP=true`, or with `npm run migrate`. Existing Copilot connectors will need `admin:org` added to their token. Until then, the Budget cap section shows an error and nothing else in the Copilot panel changes. Existing endpoints and data aren't affected.

## Known limitations

- Like the other Copilot endpoints, this uses the most recently saved Copilot connector that has an org and token. (It used to use the oldest one; that was fixed in the "most recently saved Copilot connector" change.) Saving the card again adds a new connector, so the new one starts with no stored budget results.
- The list of seats over the cap names people, so since the security fixes of 22 October 2026 it's shown only to admins (`reports.view`). Other users see the totals only.
- The numbers are only as current as GitHub's own budget figures, plus up to 15 minutes of caching on our side.
- Seats under the cap aren't shown with their percentage. Easy to add later if people want it.
