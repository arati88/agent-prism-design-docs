# Dashboard and analytics: control score, cost leaks and the dashboard snapshot

## Introduction

The main dashboard is built from one response, `GET /api/dashboard`. The server builds it from the workspace's runs every time. Nothing is stored ahead of time.

The main pieces:

- **Control score.** A 0–100 score for each run that blends success, cost against budget, speed, autonomy, guardrails and retries. Each run also gets a health label: Strong, Stable, Watch or Critical.
- **Cost leaks** ("Cost Leak Radar"). Runs where money was probably wasted: over budget, stuck retrying, poor outcome for the cost, or statistically unusual.
- **Token Coach.** The token mix (input against output), retry waste, an efficiency score and up to 5 suggestions, each with an estimated monthly saving.
- **Comparisons and trends.** Providers ranked against each other, cost per workflow, a profile per agent, an activity feed, the cost trend and a 30-day forecast.

Example: a run cost $1.20 against a $1.00 budget, succeeded, took 3 seconds, ran at autonomy 3, had no policy violations and 1 retry.

```text
success 100 × 0.25 = 25.0
budget   80 × 0.20 = 16.0   (20% over budget → 100 − 20)
latency  75 × 0.15 = 11.25  (100 − 3,000 ms ÷ 120)
autonomy 60 × 0.15 =  9.0   (3 × 20)
guards  100 × 0.15 = 15.0
retries  82 × 0.10 =  8.2   (100 − 1 × 18)
control score = 84.45 → 84 → "Stable"
```

It also cost more than its budget, so it shows on the Cost Leak Radar as a **Budget breach**.

This is how the dashboard works in the code right now. The machine-learning models it can use (logistic regression and isolation forest) have their own document. ROI and the savings report are covered separately.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/scoring.js` | `computeControlScoreFixed`, `computeControlScore` (uses the ML model when it's ready), `explainScore`, `classifyHealth`, `summarizeStatus`, `average` |
| `src/store.js` | `buildDashboardSnapshot`, `queryCostLeaks`, `detectCostLeaks`, `detectModelMismatches`, `buildTokenEfficiency` (Token Coach), `buildMLAnalytics` (trend, anomalies, forecast, clusters) |
| `server.js` | `GET /api/dashboard` (`cachedDashboard`, `buildDashboardPayload`), `GET /api/leaks`, `GET /api/runs` |
| `getRunTotals` (both stores) | Headline totals counted over **every** run |
| `public/app.js` | Overview, Token Coach, provider scorecard, Cost Leak Radar, agent panels, activity feed |

## What it does

- Gives every run, agent, workflow and provider one comparable score.
- Shows headline numbers: total runs, success rate, total cost, budget used, average latency, average control score, average satisfaction and projected monthly cost.
- Finds the runs where money is leaking and says what to do about each one.
- Tells the team how to spend fewer tokens, with an estimated saving for each suggestion.
- Ranks providers, so a buyer can see which one gives the best mix of reliability, quality, cost and speed.
- Shows whether cost is rising, falling or stable, and flags runs with unusual token counts.

## Where the data comes from

Everything is worked out from the workspace's **runs** (see [passive ingest](passive-ingest.md) and [gateway proxy](gateway-proxy.md)). The run fields that matter:

| Field | Used for |
| --- | --- |
| `status` | Success (`success` or `completed`), failed, running |
| `costUsd`, `budgetUsd` | Cost, budget use, leaks, score |
| `latencyMs` | Score, averages |
| `autonomyLevel` | Score (defaults to 3 on ingest) |
| `policyViolations` | Score |
| `retryCount` | Score, leaks, retry waste |
| `userSatisfaction` | Leaks, average satisfaction (defaults to 4 on ingest) |
| `tokensIn`, `tokensOut` | Token Coach, trend, anomalies |
| `userPromptTokens`, `systemPromptTokens`, `contextTokens`, `toolResultTokens`, `memoryTokens` | Token Coach prompt breakdown |
| `provider`, `model`, `agentName`, `workflow`, `team`, `breadcrumbs`, `notes` | Grouping, profiles, activity feed |

**Two sets of runs are used:**

- **Headline totals** (total runs, success rate, total cost, budget used, average latency) are counted by the database over **all** runs (`getRunTotals`).
- **Everything else** (scores, leaks, Token Coach, comparisons, trend) is worked out from the **most recent 10,000 runs** (`MAX_RUNS_PER_QUERY`). The response says how many runs were used in `sample.runsAnalyzed`.

## How it works

### 1. Control score per run

When the workspace's ML model is ready (20 or more labelled runs), the score is the model's prediction × 100 (see the ML document). Otherwise the fixed formula is used:

| Part | Weight | Score (each kept between 0 and 100) |
| --- | --- | --- |
| Success | 25% | 100 if succeeded, otherwise 35 |
| Budget efficiency | 20% | 100 − (cost ÷ budget × 100 − 100). Full marks at or under budget, 0 at twice the budget |
| Latency | 15% | 100 − latency ms ÷ 120. Reaches 0 at 12 seconds |
| Autonomy | 15% | autonomy level × 20 |
| Guardrails | 15% | 100 − 30 per policy violation |
| Retries | 10% | 100 − 18 per retry |

The result is rounded. Each run also gets `scoringMode` (`fixed` or `ml`).

Health label:

| Score | Health |
| --- | --- |
| 85 or more | Strong |
| 70–84 | Stable |
| 55–69 | Watch |
| Below 55 | Critical |

### 2. Headline metrics

| Metric | How |
| --- | --- |
| `totalRuns`, `successRate`, `totalCostUsd`, `budgetUsedPercent`, `averageLatencyMs` | From all runs, in the database |
| `averageControlScore`, `averageSatisfaction` | Average over the sample |
| `projectedMonthlyCost` | From Token Coach (see section 5) |

### 3. Cost leaks

A run is a leak if **any** of these are true:

| Rule | Condition |
| --- | --- |
| Over budget | cost > budget |
| Retry heavy | 3 or more retries **and** cost > $1 |
| Low value | satisfaction 2 or less **and** cost > $1.50 |
| Statistical anomaly | The workspace's isolation forest flags the run as unusual (only once it has been trained) |

Each leak gets a type and a recommendation, checked in this order:

| Type | When | Recommendation |
| --- | --- | --- |
| Budget breach | Over budget | Add budget guardrails and early stopping |
| Retry spiral | 3 or more retries | Tighten tool permissions, add checkpoints |
| Statistical anomaly | Anomaly score above 0.62 | Inspect the run's telemetry |
| Low-value spend | Otherwise | Review prompt quality and handoff logic |

Each leak also records `detectionMethod` (`threshold` or `isolation_forest`) and its `anomalyScore`. Leaks are sorted by cost, highest first.

There are two versions:

- `queryCostLeaks`, used by the dashboard and the ROI summary, returns **every** leak each time.
- `detectCostLeaks`, used by `GET /api/leaks`, is an **alert feed**. Each run is returned only once per hour (`LEAK_SUPPRESS_MS`, default 3,600,000 ms), so the same leak doesn't alert twice.

### 4. Provider comparison

For each provider: runs, cost, success rate, average control score, average latency, tokens, cost per run, cost per 1,000 tokens and retries per run. Then a composite score:

```text
composite = success rate × 0.35
          + average control score × 0.30
          + cost score × 0.20      (cost score    = (1 − cost per 1k ÷ highest cost per 1k) × 100)
          + latency score × 0.10   (latency score = (1 − latency ÷ slowest latency) × 100)
          + retry score × 0.05     (retry score   = (1 − retries per run, at most 1) × 100)
```

Providers are sorted by composite score, and the top one is marked `isWinner`.

### 5. Token Coach (`tokenEfficiency`)

Only runs with tokens count here.

| Value | How |
| --- | --- |
| Input / output mix | Input tokens ÷ total, output tokens ÷ total |
| Prompt breakdown | Totals of the five prompt buckets. The rest of the input is "uncategorised" |
| Retry waste tokens | For each run: tokens × retries ÷ (retries + 1) |
| Waste % | Retry waste ÷ total tokens |
| Cost per 1k tokens | Cost ÷ tokens × 1,000 |
| Projected monthly cost | Cost ÷ runs × (runs × 30). This always works out to **total cost × 30** (see limitation 1) |
| Efficiency score | 100, minus 25 or 12 for a high input mix (above 70% / 65%), minus 20 or 10 for a high output mix (above 45% / 40%), minus 25 or 12 for waste (above 10% / 5%), and minus 20 or 10 when the top agent averages more than 15,000 / 8,000 tokens a run |

Suggestions are made in this order, and at most 5 are shown. Each has a diagnosis, what to change, how to test it and a target:

| Suggestion | When | Estimated saving |
| --- | --- | --- |
| Run an agent to unlock token coaching | No tokens yet | — |
| Trim repeated context | Input mix above 65% and more than 3,000 tokens | 30% of input tokens |
| Cap verbose responses | Output mix above 40% and more than 3,000 tokens | 35% of output tokens |
| Eliminate retry waste | Any retry waste | All retry waste tokens |
| Right-size the top agent | Top agent averages more than 8,000 tokens a run | Tokens above 6,000 a run |
| Stabilise the top workflow | The top workflow has retries | Not estimated |
| Route routine tasks to cheaper models | Fewer than 3 suggestions so far | 60% of total cost |

Savings are turned into dollars at the workspace's average cost per 1,000 tokens, then scaled to a month in the same way as the projected monthly cost.

### 6. Trend, anomalies and forecast (`mlAnalytics`)

This needs at least 3 runs with tokens. Runs are put in time order.

- **Trend.** A straight line is fitted to cost per run. A slope above +$0.0005 a run is "rising", below −$0.0005 is "falling", and anything else is "stable". `costR2` says how well the line fits.
- **Token anomalies.** A run is flagged when its token count is more than **2 standard deviations** from the average (`ANOMALY_ZSCORE_THRESHOLD`). The top 5 are listed.
- **Burn timeline.** Tokens, cost, a 3-run moving average and the trend line for each run, cut down to at most 400 points for the chart. A point keeps its anomaly flag when points are merged.
- **Agent clusters.** Agents are ranked by average tokens a run. The lowest third are "Efficient", the middle third "Moderate" and the top third "Wasteful".
- **30-day forecast.** The trend line's cost for the run 30 places after the latest one, × 30.

### 7. Agent profiles and activity feed

- **Agent profile**, one per agent, taken from its latest run: provider, model, team, workflow, status (Running or its health label), current task (`notes` or `taskType`), control score, tasks done, total tokens and average latency. `progressPercent` is fixed by status: 96 if it succeeded, 64 if running, 42 otherwise.
- **Activity feed**, the latest 24 events. A run with `breadcrumbs` gives one event per breadcrumb, 15 seconds apart. The level is error if the run failed, warn if the text says "retry", info if it says fetched, parsed or loaded, and otherwise success or tool. A run without breadcrumbs gives a "Started" and a "Completed" or "Failed" event, with tokens, cost and latency.

### 8. Model mismatches

`detectModelMismatches` lists runs whose model is a poor fit for their task type (`mismatch` or `suboptimal`), most expensive first. The fitness rules are in `src/model-classifier.js`, covered in the advisor document.

### 9. Caching

The dashboard is built once and reused for **5 seconds** for each workspace, and requests that arrive together share the same build. New runs appear within 5 seconds. The cache is per server process.

## API

| Method and path | Auth | Returns |
| --- | --- | --- |
| `GET /api/dashboard` | Tenant | The full snapshot below |
| `GET /api/leaks` | Tenant | `{ leaks }`: leaks not already returned in the last hour |
| `GET /api/runs?limit=&offset=` | Tenant | Recent runs, newest first, paged (`limit` 1–1,000, default 200) |

`GET /api/dashboard` response fields:

| Field | Contents |
| --- | --- |
| `headlineMetrics` | See section 2 |
| `status` | Total, success, failed, running (all runs) |
| `sample` | `runsAnalyzed`, `totalRuns` |
| `providerComparison` | See section 4 |
| `workflowInsights` | For each workflow: cost, average latency, average score, failures. Most expensive first |
| `costLeaks` | See section 3 |
| `tokenEfficiency` | See section 5 |
| `mlAnalytics` | See section 6, or `null` with fewer than 3 runs |
| `agentProfiles`, `selectedAgent`, `activityFeed` | See section 7 |
| `recentRuns` | The latest 12 runs, with score and health |
| `modelMismatches` | See section 8 |
| `pricing` | Current Claude, OpenAI and Copilot rates (see [cost and pricing](cost-and-pricing.md)) |
| `tenant`, `usp` | Workspace details and the product tagline |

## Dashboard / UI

- **Overview.** Headline tiles (runs, success rate, cost, budget used, latency, control score, active agents), Cost Leak Radar, provider scorecard with the winner marked, agent profiles.
- **Token Coach.** Efficiency score, input and output mix, retry waste, cost trend arrow, anomaly count, burn chart, top agents, workflow hotspots and the suggestion cards.
- **Activity.** The event feed and paged recent runs.

## Changes by file

Files behind the dashboard:

| File | What it does for the dashboard |
| --- | --- |
| `src/scoring.js` | Control score, health label, status counts |
| `src/store.js` | Snapshot, leaks, Token Coach, trend analytics, model mismatches |
| `src/ml/logistic-regression.js`, `src/ml/isolation-forest.js` | Optional ML score and anomaly flag |
| `src/model-classifier.js` | Model fitness for mismatches |
| `server.js` | `/api/dashboard` cache and payload, `/api/leaks`, `/api/runs` |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | `listTenantContext` (10,000-run sample), `getRunTotals`, `listRecentRuns` |
| `public/app.js` | Every dashboard panel |

## Tests

What's tested today:

- `test/scoring.test.js`: each part of the score moves the right way (success, budget, latency, autonomy, guardrails, retries); no crash with 0 runs, all-failed runs, 0 or negative latency, or a budget of 0; the score stays between 0 and 100; a perfect run is near 100 and a worst-case run near 0; the fixed formula is used when there's no model.
- `test/store.test.js`: over-budget runs are a budget breach; retry-heavy runs are a retry spiral; healthy runs aren't flagged; snapshot sums are correct; workspaces don't see each other's runs; empty run sets give zeros; the Cost Leak Radar and ROI summary aren't emptied by the alert feed's once-an-hour rule; dashboard costs match the cost calculators.
- `test/token-efficiency.test.js`: the snapshot includes Token Coach suggestions.

One test in `test/scoring.test.js` is still named "currently fails - see scoring.js bug", for a run with no budget at all. That bug is already fixed (`scoring.js` now falls back to a valid number), so only the test's name and comment are out of date.

Not tested yet, and worth adding:

- Projected monthly cost and the 30-day forecast against a known number of days.
- The provider composite score with one provider, and with several.
- The low-value and anomaly leak rules.
- The 5-second dashboard cache.

## Known limitations

Places where the dashboard numbers can be off:

1. **"Projected monthly cost" is always total cost × 30.** It treats every run in the sample as one day's runs (`runsPerDay = number of runs`), so a sample covering 3 months is still multiplied by 30. Every Token Coach monthly saving is scaled the same way, so they're all inflated too.
2. **Two different scales for the same score.** The fixed formula reaches 0 latency marks at 12 seconds, but `explainScore` and the ML model use 120 seconds, and they score budget differently. When a workspace's model becomes ready, scores can jump for the same runs. The "explain" view in fixed mode shows values from the ML scale, not from the formula that produced the score.
3. **`/api/leaks` empties itself.** Each leak is returned only once an hour, and the memory of what was returned belongs to one server process. A second call within the hour gets nothing. With `npm run cluster`, each worker keeps its own memory, so results depend on which worker answers.
4. **Headline numbers and detail panels use different runs.** Totals count every run. The average score, satisfaction, leaks, Token Coach, trends and projected cost use only the latest 10,000. Once a workspace passes 10,000 runs, they stop agreeing.
