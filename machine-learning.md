# Machine learning: run value, unusual runs and task classification

## Introduction

Agent Prism has three small machine-learning pieces. All of them run inside the server, with no outside ML service.

- **Value model** (logistic regression). It learns, from the team's 👍/👎 labels, which runs were worth what they cost. Once a workspace has 20 labels, it replaces the fixed control-score formula (see [dashboard and analytics](dashboard-and-analytics.md)).
- **Unusual-run detector** (isolation forest). It learns what a normal run looks like for the workspace (tokens, cost, duration, retries) and flags runs far outside that. No labels are needed. Flagged runs appear on the Cost Leak Radar and in the Run value panel.
- **Task classifier.** It labels each run's task as code, reasoning, summarization, creative, data, multi_tool, simple_qa or general. It uses the organisation's LLM when that's switched on, and keyword rules otherwise. The label is used to judge whether the run used the right model.

The **Run value** panel (Admin) turns the first two into money: how much spend had no business return, which agents it came from, the main reason and a fix.

Example: over 30 days a workspace spent $400 on 900 runs. 120 of them (13%) scored below 0.5, adding up to $62 of spend without return. Most of that ($41, 66%) came from runs that went over budget, so the advice is to set a per-run budget and stop runs early. The detector also flags 4 runs. One reads "Input tokens 14× the usual", which is likely a runaway prompt.

These are the ML pieces as they're coded today.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/ml/logistic-regression.js` | `extractFeatures` (6 values per run, each 0–1), `LogisticRegression` (train, predict, save, load), `MIN_SAMPLES = 20` |
| `src/ml/isolation-forest.js` | `IsolationForest` (fit, score, isAnomaly). 100 trees, 256 runs per tree, needs 30 runs |
| `src/ml/value-report.js` | `buildValueReport`: the Run value panel's numbers, reasons, fixes and labelling queue |
| `src/ml/llm-classifier.js` | `classifyTaskWithLLM`: LLM classification with a 4-second timeout and a cache, falling back to keyword rules |
| `src/model-classifier.js` | `classifyTask`: the keyword rules. Also model fitness (covered in the advisor document) |
| `src/store.js` | Holds each workspace's models in memory (`setLrModel`, `setIsoForest`) and uses them in the score and the cost leaks |
| `server.js` | Feedback route (retrains), `/api/ml/value-report`, `/api/ml/status`, start-up loading, the hourly rebuild, the classifier hook on ingest |
| Storage | Run columns `user_outcome` and `task_type`. Saved weights in `ml_model_weights` (Postgres) or `mlModelWeights` (file store) |

## What it does

- Lets the team teach Agent Prism what a valuable run is for **them**, with one click per run.
- Puts a dollar figure on spend that brought no return, with the agents responsible and what to change.
- Spots runaway runs (loops, huge prompts, misconfiguration) without anyone setting thresholds.
- Labels each run's task type, so model-fit advice knows what the run was trying to do.

## Where the data comes from

| Data | Source | Used by |
| --- | --- | --- |
| `status`, `costUsd`, `budgetUsd`, `latencyMs`, `autonomyLevel`, `policyViolations`, `retryCount` | Every run | Value model |
| `tokensIn`, `tokensOut`, `costUsd`, `latencyMs`, `retryCount` | Every run | Unusual-run detector |
| `userOutcome` (1 = worth it, 0 = not) | 👍/👎 in the Activity feed or the Run value panel | Value model training |
| `messages` or `prompt` in the ingest body | `/api/ingest/claude`, `/api/ingest/generic` | Task classifier |

Which runs are used:

| For | Runs |
| --- | --- |
| Training the value model | The latest 5,000 labelled runs |
| Training the detector | Runs from the last 30 days, at most 2,000 (start-up and hourly) or 5,000 (Run value panel) |
| Run value report | Runs from the last 30 days, at most 5,000 |

## How it works

### 1. The six features

`extractFeatures(run)` turns a run into six numbers between 0 and 1:

| Feature | Value |
| --- | --- |
| success | 1 if succeeded, otherwise 0 |
| budgetEfficiency | 1 − cost ÷ budget, with the ratio capped at 3. 0 once a run reaches its budget. A run with cost but no budget counts as 1.5 × over |
| latencyScore | 1 − latency ÷ 120 seconds |
| autonomyScore | autonomy level ÷ 5 |
| guardrailScore | 1 − policy violations ÷ 5 |
| retryScore | 1 − retries ÷ 10 |

### 2. The value model (logistic regression)

```text
P(worth it) = sigmoid(bias + w1·success + w2·budget + … + w6·retries)
```

- **Training.** Every time someone gives 👍/👎 (`POST /api/runs/:id/feedback`), and the workspace then has **20 or more** labels, the model trains on all of them: 200 passes of gradient descent, learning rate 0.05, L2 penalty 0.01. It stops early when the loss stops improving. Training continues from the current weights.
- **Saving.** The weights are saved to `ml_model_weights` under the name `control_score_lr`.
- **Use.** Once it's ready, the dashboard's control score is P(worth it) × 100, and the Run value report runs in "learned" mode.

### 3. The unusual-run detector (isolation forest)

1. **Fit.** It needs at least 30 runs. Each of the five values (input tokens, output tokens, cost, duration, retries) is scaled to 0–1 using the minimum and maximum in the training runs. It builds 100 random trees, each from up to 256 runs, by repeatedly splitting on a random value at a random point.
2. **Score.** A run that gets isolated in few splits is unusual. The score is 2^(−average path length ÷ c(256)), between 0 and 1. Closer to 1 is more unusual.
3. **Flag.** A score above **0.62** counts as unusual.

When it's built:

| Backend | When |
| --- | --- |
| Postgres | 5 seconds after start-up, then every hour, for every workspace with 30 or more recent runs. Also on the first Run value request if none exists yet |
| File store | Only on the first Run value request, then never rebuilt |

### 4. The Run value report (`buildValueReport`)

Each run gets a value between 0 and 1:

| Mode | When | Value |
| --- | --- | --- |
| `rules` | Fewer than 20 labels | 0 if the run finished without success (running runs excluded), otherwise the fixed control score ÷ 100 |
| `learned` | 20 or more labels | The value model's P(worth it) |

- A run with value **below 0.5** is low value, and its cost counts as "spend without business return".
- **Main reason** for each low-value run: the feature costing it most (weight × (1 − value)). Only features with a positive weight count. Reasons are added up by spend, overall and for each agent:

| Feature | Reason shown | Fix shown |
| --- | --- | --- |
| success | Runs that fail | Find the most common failure and fix it |
| budgetEfficiency | Runs going over budget | Set a per-run budget and stop runs early |
| latencyScore | Slow runs | Try a faster model or send less context |
| autonomyScore | Runs needing a lot of human help | Clearer instructions, or the tools it keeps asking for |
| guardrailScore | Policy violations | Tighten tool permissions, review the violating steps |
| retryScore | Too many retries | Cap retries at 3 and add a checkpoint |

- **Agents:** the top 10 by low-value spend, each with its own main reason.
- **Labelling queue:** the 8 most expensive runs without a label, because they teach the model the most.
- **Unusual runs**, when the detector is ready: the count, their total cost and the 5 most expensive. Each gets a reason naming the value furthest above the workspace's median, for example "Cost 6.2× the usual".

### 5. The task classifier

On `/api/ingest/claude` (when the body has `messages`) and `/api/ingest/generic` (when it has `messages` or `prompt`), after the run is saved and **without waiting**:

1. If the run used 2 or more tools, the answer is `multi_tool`.
2. The first 4 messages are joined and cut to 800 characters. If that's empty, the answer is `general`.
3. If the same text was classified before, the cached answer is used. The cache holds 2,000 entries per process.
4. If the LLM classifier is off, the keyword rules decide.
5. Otherwise, personal data is removed from the text, it's wrapped in `<agent_content>` tags with an instruction that it is data to classify, not instructions, and it's sent to the organisation's LLM (up to 12 tokens back).
6. If the answer is one of the 8 types, it's used and cached. A different answer, an error, or no reply within **4 seconds** falls back to the keyword rules.
7. The run's `task_type` is updated.

The LLM classifier is **off by default**. It needs `LLM_CLASSIFIER_ENABLED=true` and a configured LLM provider, because it sends prompt text (with personal data removed) outside the deployment.

## API

| Method and path | Auth | What it does |
| --- | --- | --- |
| `POST /api/runs/:id/feedback` `{ outcome: 0 or 1 }` | member | Saves the label and retrains when 20 or more labels exist. Returns `modelReady`, `sampleCount`, and `loss` or `samplesNeeded` |
| `GET /api/ml/value-report` | Tenant | The Run value report: `mode`, `labels`, `totals`, `topDriver`, `agents`, `labelQueue`, `anomalies` |
| `GET /api/ml/status` | Tenant | Value model readiness, label counts, weights, last training and loss; detector trees, sample size and when it was trained; classifier on/off, provider and cache size |

## Dashboard / UI

- **Admin → Run value.** Three cards: spend without business return, share of AI spend ("under 10% is good"), and unusual runs. Then a note on the mode, the main reason callout, a "Where it goes" table for each agent, the unusual runs list and the "Teach it what's worth it" queue with 👍/👎.
- **ML status panel.** Value model ACTIVE, or "x/20 labels"; detector state; classifier state. When the model is trained, its weights are shown in green or red.
- **Activity feed.** 👍/👎 on each run, showing how many labels are still needed.

## Changes by file

ML code is in:

| File | What it does for ML |
| --- | --- |
| `src/ml/logistic-regression.js` | Features and the value model |
| `src/ml/isolation-forest.js` | Unusual-run detector |
| `src/ml/value-report.js` | Run value report |
| `src/ml/llm-classifier.js`, `src/model-classifier.js` | Task classification |
| `src/privacy.js` | `isLlmClassifierEnabled` |
| `src/store.js`, `src/scoring.js` | In-memory models, ML score, anomaly cost leaks |
| `server.js` | Routes, start-up load, hourly rebuild, classifier hook |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | Labels, saved weights, runs for ML, `updateRunTaskType` |
| `public/app.js` | Run value panel, ML status, feedback buttons |

## Tests

What's tested today:

- `test/run-value.test.js`: rules mode puts a dollar figure on low-value runs, for each agent, with a reason and a fix; learned mode follows the team's labels; unusual runs are counted with their cost and a reason; no runs gives an empty report; `GET /api/ml/value-report` works against a real server and is scoped to the workspace.
- `test/ml-persistence.test.js`: saved weights load back; nothing saved returns `null`; workspaces are kept separate; saving again replaces the old weights; weights survive a restart.
- `test/ml-feedback.test.js`: labels save and load back; an unknown run gives "not found"; only labelled runs come back, for one workspace; `updateRunTaskType` updates in place and ignores unknown runs; runs for ML cover the last 30 days and respect the limit.
- `test/llm-classifier-injection.test.js`: content is wrapped in tags; a fake closing tag can't break out; normal content still classifies; 2 or more tools skip the LLM.

Not tested yet, and worth adding:

- The detector on small workspaces: how many ordinary runs are flagged at 30, 60 and 100 runs (see limitation 1).
- Value model training: does it learn the right direction from clearly separated labels?
- The classifier's fallback on timeout and on an invalid answer.
- The startup load and the hourly rebuild.

## Known limitations

Things to be aware of with the models:

1. **The detector flags most runs in small workspaces.** The score is always scaled by the figure for 256 runs, even when a tree was built from fewer. A quick simulation on ordinary runs with nothing unusual flagged about **97% at 30 runs, 40% at 60 and 15% at 100**, falling to under 1% only from 256 runs. Every flagged run also appears on the Cost Leak Radar as a "Statistical anomaly", so a new workspace's radar fills with false leaks. The "≈5% false-positive rate" in the code comment only holds for large workspaces.
2. **Models live in each server's memory.** A label retrains the model only in the process that received it. With `npm run cluster`, other workers keep the old model until restart. On Postgres, saved weights are loaded only at start-up.
3. **One label can change every score.** At 20 labels, every control score in the workspace switches from the fixed formula to P(worth it), on a different scale. Twenty labels is very few, nothing checks that both 👍 and 👎 are present, and if all 20 are 👍 the model learns that everything is valuable.
4. **Any member can train the model.** Labels need only `agents.operate`, and one person's labels change the score for the whole workspace. Nothing records who labelled what.
