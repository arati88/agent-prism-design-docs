# Cost calculation and pricing: how Agent Prism turns tokens into dollars

## Introduction

Almost every number on the dashboard is built from cost: spend per agent, budgets, cost leaks, the control score, ROI, the savings report, Return on Copilot. This document explains where that cost comes from.

There are two parts:

- **One price list**, `config/pricing.json`. It holds every rate: price per token for each Claude and OpenAI model, Copilot seat and credit prices, and the human-work rates used for ROI.
- **Four small calculators** in `src/cost/`, one per provider. Each reads its rates from the price list. No rate is written anywhere else in the code.

The price list **reloads by itself**: edit the file and the next calculation uses the new rate, with no restart and no redeploy.

Example: a gateway call to `claude-sonnet-4-5` uses 10,000 input tokens and 1,000 output tokens. The model name contains `sonnet`, so it gets the Sonnet rates ($3 and $15 per million tokens):

```text
10,000 × $0.000003 + 1,000 × $0.000015 = $0.030 + $0.015 = $0.045
```

This document describes how cost works **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | What it does |
| --- | --- |
| `config/pricing.json` | The price list. The only place a rate is set |
| `src/pricing.js` | Loads the price list, checks required fields, reloads it when the file changes, says when it's out of date |
| `src/cost/claude.js` | `computeClaudeCost`: input, output, cache-write and cache-read tokens |
| `src/cost/openai.js` | `computeOpenAiCost`: input and output tokens. Also used for Azure OpenAI |
| `src/cost/copilot.js` | `computeCopilotCost` (seat hours), `computeCopilotCreditCost` (AI credits), `computeCopilotSeatCost` (seat price for N days) |
| `src/cost/generic.js` | `computeGenericCost`: passes through a cost the caller already worked out |

## What it does

- Gives every part of Agent Prism **one way** to price usage, so the same call always costs the same everywhere.
- Picks the **right rate for the model**, so an Opus call costs more than a Haiku call.
- Lets an admin **change prices without a restart**, for example when a vendor changes its pricing.
- Never crashes on bad input: missing or invalid numbers give $0, never `NaN` or an error.
- For Copilot credits, says **"unknown"** (`null`) rather than $0 when no price has been set, so Copilot never looks free by mistake.

## Where the data comes from

### The price list: `config/pricing.json`

| Section | Fields | Used for |
| --- | --- | --- |
| `platforms.claude` | `input_price_per_token`, `output_price_per_token`, `cache_write_price_per_token`, `cache_read_price_per_token` | Fallback Claude rates |
| `platforms.claude.models` | Same four rates per model family: `opus`, `sonnet-5`, `sonnet`, `haiku` | Model-specific Claude rates |
| `platforms.openai` | `input_price_per_token`, `output_price_per_token` | Fallback OpenAI and Azure rates |
| `platforms.openai.models` | Same two rates per family: `gpt-4o-mini`, `gpt-4o`, `gpt-4-turbo`, `o1` | Model-specific OpenAI rates |
| `platforms.copilot` | `hourly_seat_rate` (0.054), `price_per_credit_usd` (null), `seat_price_monthly_usd` (Business 19, Enterprise 39), `value` (lines per hour, minutes per prompt) | Copilot cost and Return on Copilot |
| `fte` | `avg_human_hours_per_task` (2), `loaded_hourly_rate_usd` (85) | ROI: what the same work would cost a person |
| `session_timeout_minutes` | 30 | When a session with no events is marked timed out |
| `last_verified` | A date, e.g. `2026-08-09` | When someone last checked the rates against the vendors |

Current per-million-token rates, for reference:

| Model family | Input | Output | Cache write | Cache read |
| --- | --- | --- | --- | --- |
| Claude `opus` | $5 | $25 | $6.25 | $0.50 |
| Claude `sonnet-5` | $2 | $10 | $2.50 | $0.20 |
| Claude `sonnet` | $3 | $15 | $3.75 | $0.30 |
| Claude `haiku` | $1 | $5 | $1.25 | $0.10 |
| Claude, no match | $3 | $15 | $3.75 | $0.30 |
| OpenAI `gpt-4o-mini` | $0.15 | $0.60 | | |
| OpenAI `gpt-4o` | $2.50 | $10 | | |
| OpenAI `gpt-4-turbo` | $10 | $30 | | |
| OpenAI `o1` | $15 | $60 | | |
| OpenAI, no match | $2.50 | $10 | | |

### Tokens and model name

These come from whoever calls the calculator:

| Caller | Tokens from | Calculator |
| --- | --- | --- |
| Gateway `/v1/messages` | Anthropic's response `usage`, including cache tokens | Claude |
| Gateway `/v1/responses`, `/v1/chat/completions`, `/v1/azure/…` | OpenAI's or Azure's response `usage` | OpenAI (Azure uses the deployment name as the model) |
| Fleet ingest and the local Live Sessions scan | The **increase** in a session's token totals since the last snapshot | Claude |
| Live Sessions view (`/api/fleet/sessions`, local sessions) | A session's full totals, for "cost so far" and burn rate | Claude |
| `POST /api/usage` | `input_tokens`, `output_tokens`, `seat_hours` or `cost_usd` in the request, chosen by `platform` | Claude, Copilot (seat hours) or generic. OpenAI is rejected by validation |
| Copilot panel, Return on Copilot | GitHub's AI credits and seat plans | Copilot credit and seat cost |
| Passive ingest runs (`/api/ingest*`) | **None. The agent's reported `costUsd` is stored as sent** | — |

## How it works

### Picking the rate for a model

`resolveClaudeRates` and `resolveOpenAiRates`:

1. Lowercase the model name.
2. Go through the `models` entries **in the order they appear in the file**. The first family name found *inside* the model name wins. For example, `claude-opus-4-6` contains `opus`.
3. If nothing matches, or no model was given, use the platform's fallback rates.

Order matters. A more specific key must come first: `gpt-4o-mini` before `gpt-4o`, and `sonnet-5` before `sonnet`. Otherwise `gpt-4o-mini` would get the `gpt-4o` price.

### The formulas

```text
Claude   cost = input × input rate + output × output rate
              + cache-write tokens × cache-write rate + cache-read tokens × cache-read rate

OpenAI   cost = input × input rate + output × output rate

Copilot  seat-hour cost = seat hours × hourly_seat_rate
         credit cost    = AI credits × price_per_credit_usd   (null if no price is set)
         seat cost      = monthly seat price for the plan × days ÷ 30.44
                          (plans not in the list are priced as Business)

Generic  cost = the cost_usd sent by the caller
```

If a needed rate is missing, the calculator returns $0. If the result isn't a valid number, it also returns $0. Credit cost returns `null` instead.

### Reloading the price list

- At start-up, `src/pricing.js` reads the file and **stops the server** if it's missing or a required field is absent: Claude and OpenAI input/output rates, `copilot.hourly_seat_rate`, both `fte` fields and `session_timeout_minutes`.
- Before **every** calculation, `refreshPricingIfStale()` checks the file's modified time, which is a single cheap file check. If the file changed, it's read again and the rates are replaced in place.
- If the new file is broken, for example half-saved or with a field missing, the change is **ignored**: the last good rates stay in use, an error is logged, and it tries again on the next calculation.

### "Rates may be out of date"

`isPricingStale()` returns true when `last_verified` is missing or more than **90 days** old. `GET /api/metrics/summary` returns it as `meta.pricingStale` with `meta.pricingLastVerified`.

## API

There's no pricing endpoint of its own. Cost shows up inside other responses:

| Endpoint | What it includes |
| --- | --- |
| `GET /api/dashboard` | `pricing.claude`, `pricing.openai`, `pricing.copilot`: the current rates, so the dashboard can show them |
| `GET /api/metrics/summary` | `meta.pricingStale`, `meta.pricingLastVerified` |
| `POST /api/usage` | `costDelta` (cost of this usage) and `sessionCostUsd` (the session's new total) |
| `GET /api/metrics/cost?window=` | Total and per-platform cost of sessions in the window |
| Runs, dashboard, reports | `costUsd` on every run, already calculated when the run was saved |

`POST /api/usage` request:

```json
{ "session_id": "sess_123", "platform": "claude", "model": "claude-haiku-4-5",
  "input_tokens": 20000, "output_tokens": 2000 }
```

`platform` can be `claude`, `copilot` (uses `seat_hours`) or `generic` (uses `cost_usd`), and defaults to `generic`. `server.js` also has an `openai` branch, but the request schema rejects `openai` with `400`, so it's never reached (see [sessions](sessions-and-timeout.md)).

## Dashboard / UI

There's no pricing screen. Changing a rate means editing `config/pricing.json` on the server, and the change shows on the next refresh.

Costs appear across the dashboard: Overview spend, Runs, cost leaks, Live Sessions burn rate, AI Consumption History, the Copilot panel, Return on Copilot and the savings report.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does for cost |
| --- | --- |
| `config/pricing.json` | Every rate |
| `src/pricing.js` | `pricing`, `refreshPricingIfStale`, `isPricingStale`, required-field check |
| `src/cost/claude.js`, `openai.js`, `copilot.js`, `generic.js` | The calculators |
| `server.js` | Calls the calculators: gateway routes, fleet ingest, Live Sessions, `/api/usage`, Copilot routes. Sends rates in `/api/dashboard`, the stale flag in `/api/metrics/summary` |
| `src/roi.js` | Reads the `fte` rates |
| `src/jobs/session-timeout.js` | Reads `session_timeout_minutes` |
| `src/copilot/copilot-roi.js` | Uses the Copilot seat and credit calculators (passed in by `server.js`) |

## Tests

What's tested today:

- `test/cost.test.js`: Claude and OpenAI input + output; zero tokens; Opus priced above and Haiku below the Sonnet fallback; unknown model uses the fallback; cache tokens are billed; `gpt-4o-mini` is matched before `gpt-4o`; **an edited rate takes effect without a restart**; Copilot seat hours; generic pass-through and missing cost.
- `test/pricing.test.js`: the price list loads with its required fields; a recent `last_verified` isn't stale.
- `test/copilot-roi.test.js`: Copilot seat cost per plan and credit cost only when a price is set.

Not tested yet, and worth adding:

- A broken edit to `pricing.json` keeps the last good rates.
- `last_verified` older than 90 days counts as stale.
- `computeCopilotCreditCost` returns `null`, not 0, with no price set.
- Bad inputs (text, negative numbers) never give `NaN`.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **The Sonnet 5 introductory price has expired but is still in the file.** Its own note says the `sonnet-5` entry ($2/$10) should be deleted after **2026-08-31**, so that Sonnet 5 falls back to the normal $3/$15. It's still there, so Sonnet 5 usage is currently **under-priced by about a third**.
2. **The "out of date" warning is hardly ever seen.** `meta.pricingStale` is returned by `/api/metrics/summary`. Only the older `/dashboard` page shows it, as a banner, and that page doesn't load data for normal signed-in users (see [agent cost and ROI tracker](agent-cost-roi-tracker.md)). The main dashboard doesn't show it at all. `last_verified` is 2026-08-09, so it turns stale on about 2026-11-07, and almost nobody will see it.
3. **Newer models fall back to default prices.** Only four OpenAI families are listed. Models such as `gpt-4.1`, `gpt-5` or `o3` get the default $2.50/$10, which may be well off. Older Claude models are priced at current rates too: for example `opus` matches older, more expensive Opus versions at today's $5/$25.
4. **Name matching can pick the wrong entry.** The first family name found anywhere in the model name wins, so a model whose name contains an existing key gets that key's price. For example, `o1-mini` matches `o1` and is charged the full o1 price. File order is the only safeguard.
5. **Passive ingest runs aren't priced by Agent Prism.** The run routes store the cost the agent reports (see [passive ingest](passive-ingest.md)). Runs with tokens but no cost show as $0, and their cost doesn't follow edits to the price list.
6. **`POST /api/usage` ignores Claude cache tokens.** It only passes input and output tokens, so cache cost is left out, while the gateway and fleet ingest include it.
7. **Costs are worked out once.** A run's cost is calculated and stored when it's saved. Changing a rate affects new usage only; past runs keep their old cost. That's usually what you want, but there's no way to re-price history.
8. **One price list for every workspace.** All workspaces on a server share the same rates. A customer with a negotiated discount can't have its own prices, except Copilot credits, and even that is server-wide.
9. **Rates are edited by hand on the server.** There's no screen, no history of changes and no audit entry when a rate changes.
