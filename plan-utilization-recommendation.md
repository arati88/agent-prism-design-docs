# Plan utilisation: License Value Multiple and seat recommendation

## Introduction

Spend History has a **Plan Utilization** panel that compares a team's **AI consumption value** with what it **pays** for its AI coding-tool plan, for example Claude Team at $30 a seat. It shows three figures:

- **Allocated licence cost** for the selected range: seats × price per seat per month × days ÷ 30.
- **AI consumption value**: token activity in the same range, priced at API list prices (see [consumption history](consumption-history-and-burn-rate.md)).
- **License Value Multiple** = consumption value ÷ allocated licence cost, with a band: Poor below 1×, Low 1–3×, Healthy 3–7×, High 7–15×, Very High above 15×.

Then a **recommendation**. The workspace sets a **target multiple** (default 3×). If the multiple is more than 20% below target, seats look **underused**. More than 20% above, seats look **near or over capacity**. Otherwise it's **on target**. In the first two cases, the panel suggests the seat count that would hit the target, and **Adjust seats → N** opens the Edit Plan form with that number filled in. Nothing is saved until someone clicks Save. The banner can be dismissed until the next refresh.

The plan figures (plan name, seats, price per seat, target) are entered by hand. This is the customer's **own** AI-tool subscription, not Agent Prism's edition (see [licensing and plans](licensing-and-plans.md)).

This document replaces the openspec change `openspec/changes/archive/2026-08-24-plan-utilization-recommendation/` and its spec in our format.

Example: 10 seats at $30 a month, a 30-day range, $450 of consumption value. Allocated cost is $300, the multiple is **1.5×** ("Low"), and the target is 3×. 1.5 is below 2.4 (80% of target), so the panel says "underused". The suggested seats are round(450 ÷ (3 × 30 × 1)) = **5**, shown as "Consider cutting to 5 seats to hit target".

This document describes the panel **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | Where | What it does |
| --- | --- | --- |
| `licenseValueBand(multiple)` | `public/app.js` | Band and colour |
| `planRecommendation({ usageValueUsd, pricePerSeatUsd, daysInRange, seats, targetMultiple, multiple })` | `public/app.js` | State and suggested seats |
| `renderUtilizationPanel(...)` | `public/app.js` | Panel, sentence, banner, Edit Plan form |
| `initPlanConfigForm()` | `public/app.js` | Form open/close, Adjust prefill, Dismiss, save (pauses the 30-second refresh while open) |
| `GET` / `POST /api/tenant/plan-config` | `server.js` | Read (any workspace user) / save (`settings.manage`) |
| `SCHEMAS.setPlanConfig` | `src/validation.js` | `planName` up to 64 characters, `seats` 1–100,000, `pricePerSeatUsd` 0–100,000, `targetMultiple` 0.1–1,000 |
| Storage | `tenant_plan_config` (migrations 015, 016 `target_multiple NUMERIC(4,2) DEFAULT 3.0`) | One row per workspace |

## What it does

- Turns "are we getting our money's worth from these seats?" into a number and a suggested action.
- Lets a manager set what "good" means for them (the target).
- Never changes anything by itself.

## Where the data comes from

| Input | Source |
| --- | --- |
| Consumption value | `summary.totalCostUsd` from `/api/usage/daily` for the selected range |
| Days | `daily.length`: the number of days **with usage rows** in the range |
| Active developers | Developers in the rows |
| Plan name, seats, price, target | `/api/tenant/plan-config`. Defaults when nothing is saved, or the call fails: Pro, 1 seat, $20, 3× |

## How it works

```text
allocated   = seats × pricePerSeatUsd × days ÷ 30
multiple    = consumption value ÷ allocated                 (none if allocated is 0)
low / high  = target × 0.8 / target × 1.2
state       = underused if multiple < low; over-capacity if > high; else on-target
suggested   = max(1, round(consumption value ÷ (target × pricePerSeatUsd × days ÷ 30)))
action      = shown only if the state isn't on-target and suggested ≠ current seats
```

| State | Banner |
| --- | --- |
| Underused | "Usage is below your N× target — seats look underused. Consider cutting to S seats to hit target." |
| Over capacity | "Usage is above your N× target — seats look near or over capacity. Consider adding to S seats…" |
| On target | "Usage is within range of your N× target. No seat-count action suggested." |

Below the multiple: "This is a utilization/economic efficiency indicator and should not be interpreted as business ROI."

## API

| Method and path | Auth | Body / returns |
| --- | --- | --- |
| `GET /api/tenant/plan-config` | Tenant | `{ planName, seats, pricePerSeatUsd, targetMultiple }` |
| `POST /api/tenant/plan-config` | `settings.manage` (admin) | Same fields; `422` when invalid; returns `{ ok, … }` |

## Dashboard / UI

**Spend History → Plan Utilization:** three stat tiles, a one-line summary ("N developers generated $X of AI consumption value against $Y…"), the recommendation banner with **Adjust seats → N** and **Dismiss**, and **Edit Plan** (plan name, seats, price per seat, target multiple).

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does |
| --- | --- |
| `public/app.js` | Band, recommendation, panel, form |
| `public/styles.css` | `.ch-util-*`, `.ch-rec-*` |
| `server.js` | Plan-config routes |
| `src/validation.js` | `setPlanConfig` schema |
| `src/stores/*` | `getPlanConfig`, `setPlanConfig` |
| `db/migrations/015_*`, `016_plan_config_target_multiple.sql` | Table and target column |
| `openspec/changes/archive/2026-08-24-plan-utilization-recommendation/*`, `openspec/specs/plan-utilization-recommendation/spec.md` | Original change and spec, replaced by this document |

## Tests

Nothing automated. `planRecommendation` is a pure function inside `public/app.js`, but it isn't exported or tested. The validation schema is covered by the general validation tests.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **"Days" means days with usage, not days in the range.** Licence cost is prorated by `daily.length`. A 30-day range where usage happened on 10 days is charged as 10 days of licence, so the multiple looks three times better than it is. "All Time" is prorated over however many days happen to have data.
2. **The suggestion ignores how many people use the tool.** It solves the ratio for seats without looking at active developers. Ten active developers can get "cut to 3 seats". The seat count is never compared with real seat assignments, such as Copilot seats or SCIM users.
3. **"Over capacity" isn't really about capacity.** A high multiple means each seat produces a lot of list-price value, which suggests the subscription is good value. The banner says to **add** seats, which only lowers the ratio back to target. The recommendation steers towards a target ratio, not towards what the team needs.
4. **The value side is a list-price estimate,** priced at Claude rates for every tool (see [consumption history](consumption-history-and-burn-rate.md)). Comparing it with a real subscription price mixes an estimate with an invoice.
5. **One plan per workspace.** A workspace with several tools or plans (for example Claude Team and Copilot Business) can only enter one seat count and price.
6. **Fixed 20% band.** The ±20% tolerance is in code, and the target and band are judgment calls, not derived from data.
7. **Changes aren't audited, and anyone can read them.** Saving plan config writes no audit entry, and every workspace user can read it.
8. **Silent defaults.** If the plan config can't be loaded, the panel quietly uses Pro / 1 seat / $20, which produces a confident-looking but meaningless multiple.
