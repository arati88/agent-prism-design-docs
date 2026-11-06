# AI Advisor, Prompt Advisor and model fitness

## Introduction

Agent Prism has three advice features:

- **AI Advisor.** It reads a summary of the workspace's telemetry (headline numbers, Token Coach, cost leaks, provider and workflow scorecards, the last 12 runs) and asks an LLM for up to 5 recommendations in business language. Each one has why it matters, what to do, the expected impact, an owner and what to check next.
- **Prompt Advisor.** It sends one prompt to an LLM and gets back a score out of 10, its main weakness, a rewritten prompt and an estimate of how many input tokens the rewrite saves (0–60%). The dashboard turns that into a monthly and annual dollar saving. "Apply this rewrite" copies the rewrite and logs the estimated saving.
- **Model fitness.** Fixed rules say whether a run's model suits its task type: a small model for simple questions, a large one for reasoning. Mismatches are listed on the dashboard and counted in `/api/model-fitness`.

The two advisors use **the organisation's own LLM**, set with `AI_PROVIDER` (Azure OpenAI, Amazon Bedrock, Google Vertex AI, an OpenAI-compatible gateway, OpenAI, Anthropic, Ollama or OpenRouter). Nothing is sent anywhere until a provider is set. Both need a plan that includes `aiAdvisor` (Starter and above).

Example: the Prompt Advisor scores "Summarise this" at 3/10. The weakness is that it gives no audience, length or format. The rewrite is "Summarise the attached incident report for an engineering manager in 5 bullet points, under 120 words", with a 25% token saving. The run had 8,000 input tokens and cost $0.04, and its workflow is `customer-ops`, which the dashboard assumes runs 150,000 times a month. So the card shows 2,000 tokens saved a run, $0.04 → $0.03 a run, **$1,500 a month** and $18,000 a year.

Both advisors are described as they're coded right now.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/llm/provider.js` | `completeText`, `llmStatus`, `isLlmConfigured`: the only place Agent Prism calls an LLM for its own features |
| `src/ai-advisor.js` | `buildAdvisorTelemetry`, `generateAiAdvisor`, `parseAdvisorJson`, `callLlm` (used by the Prompt Advisor) |
| `src/model-classifier.js` | `classifyTask` (keyword rules), `scoreFitness`, `getModelRecommendation`, `scrubPii`/`scrubPiiDeep` |
| `src/store.js` | `detectModelMismatches` for the dashboard |
| `server.js` | `GET /api/ai-advisor`, `POST /api/advisor`, `POST /api/advisor/apply`, `GET /api/model-fitness` |
| Storage | Saved prompt analyses (only with prompt capture on); `prompt_captures` with `task_type` and `model_fitness` |
| `public/app.js` | AI Advisor panel, Prompt Advisor cards and savings bar, model mismatch list |

## What it does

- Gives a business owner a short, plain-language list of what to fix next, based on their own telemetry.
- Shows a team how to write each prompt better, and what that's worth in money.
- Flags runs that pay for a large model on a simple task, or risk quality with a small model on a hard one.
- Runs on the LLM service the client's security team has already approved.

## Where the data comes from

| Feature | Input |
| --- | --- |
| AI Advisor | The workspace's latest 10,000 runs, turned into a dashboard snapshot. Sent to the LLM: workspace id, name and plan; headline numbers; Token Coach summary; top 6 cost leaks; top 6 providers and workflows; the last 12 runs (agent, provider, model, task type, workflow, status, tokens, prompt breakdown, cost, budget, retries, violations, satisfaction, start time) |
| Prompt Advisor | The `prompt` text sent by the dashboard. The dashboard picks it from each recent run's first breadcrumb longer than 10 characters, or its `notes`, or from gateway prompt captures. It's cut to 600 characters |
| Prompt Advisor savings | Calculated in the browser from the run's tokens and cost, and a fixed monthly run count for each workflow |
| Model fitness | The run's `model` and `taskType` |

## How it works

### 1. AI Advisor (`GET /api/ai-advisor`)

1. If the plan doesn't include the advisor, return `status: "unavailable"` with `reason: "plan"`.
2. If no LLM provider is set, return `unavailable` and name the missing settings.
3. If there are no runs, return `waiting_for_telemetry`.
4. Build the telemetry summary and send it with a prompt that says: use only this telemetry; don't invent secrets, vendors, incidents or dollar amounts; write for a business and platform owner; return strict JSON in this schema. The answer can be up to 1,024 tokens.
5. Read the JSON from the answer, including from a ```json block. Keep at most 5 recommendations and 4 questions, and fill any missing field with a default. `confidence` must be high, medium or low (default medium), and `priority` must be cost, reliability, governance or quality (default quality).
6. If the call fails or the answer isn't JSON, return `unavailable` with the error.

Response: `summary`, `confidence`, `priority`, `recommendations[]` (`title`, `why`, `action`, `expectedImpact`, `owner`, `nextCheck`), `questions[]`, `provider`, `model`, `generatedAt`, `telemetryWindow`.

### 2. Prompt Advisor (`POST /api/advisor` `{ prompt, runId? }`)

1. Check the plan (`402`), check an LLM is configured (`503`), and check the prompt: it must be present (`400`) and at most 50,000 characters (`413`). The `runId`, if sent, must be at most 128 characters.
2. If `runId` is given and an analysis of the **same prompt text** (SHA-256) was saved, return it with `cached: true`.
3. Send this to the LLM: "You are a prompt engineering expert…", the expected JSON (`score`, `weakness`, `rewrite`, `tokenSavingsPct`), then the prompt text. The answer can be up to 300 tokens.
4. Take the JSON out of the answer. If it can't be read, return `score: 0` and a "Could not parse" weakness.
5. Save the analysis for that run only when **prompt capture** is on, because the rewrite is prompt text.

The dashboard's Prompt Advisor tab analyses up to a page of runs, a few at a time with a pause between batches. For each card:

```text
tokens saved per run = input tokens × savings %          (savings % kept between 0 and 60)
cost per token       = run cost ÷ input tokens
cost per run now     = input tokens × cost per token
cost per run after   = (input tokens − tokens saved) × cost per token
monthly runs         = a fixed number for the workflow (below), otherwise 30,000
monthly saving       = (now − after) × monthly runs;  annual = monthly × 12
```

| Workflow | Monthly runs assumed |
| --- | --- |
| it-ops | 180,000 |
| customer-ops | 150,000 |
| sales-ops | 90,000 |
| analytics-pipeline | 60,000 |
| hr-ops | 45,000 |
| ci-cd-pipeline | 15,000 |
| compliance | 12,000 |
| finance-reporting, market-intelligence | 6,000 |
| legal-review | 4,500 |
| strategic-analysis, research | 3,000 |
| Anything else | 30,000 |

### 3. "Apply this rewrite" (`POST /api/advisor/apply`)

When someone clicks **Apply this rewrite**, clicks **Apply all**, or **copies text from the rewrite box**, the dashboard:

1. Copies the rewrite to the clipboard (on click only).
2. Adds the card's monthly saving to the savings bar.
3. Sends `runId`, `tokensIn`, `runsPerMonth`, `tokenSavingsPct` and `costUsdPerRun` to `/api/advisor/apply`.

The server recalculates the monthly saving from those values in the same way. It also works out a **15% commission** on that saving (`COMMISSION_RATE` in `server.js`), and writes an `Advisor Rewrite Applied` audit event with the saving, the commission and the lengths of both prompts. It returns `{ status: "applied", estimatedMonthlySavingsUsd }`. Nothing about the agent or its prompt actually changes.

### 4. Model fitness

Each model is put in a tier by its name:

| Tier | Model names |
| --- | --- |
| fast | haiku, gpt-4o-mini, gpt-4.1-mini, gpt-3.5, gemini-flash, mistral-7b |
| balanced | sonnet, gpt-4o, gpt-4.1, gpt-4-turbo, gemini-pro |
| powerful | opus, gpt-4, o1, o3 |
| unknown | anything else |

The ideal tier for each task: simple_qa and summarization → fast; reasoning → powerful; everything else → balanced.

| Model vs ideal | Fitness | Penalty |
| --- | --- | --- |
| Same tier | optimal | 0 |
| One step off, and a powerful model on a non-reasoning task | suboptimal | 8 |
| One step off, otherwise | good | 5 |
| Fast model on a reasoning task | mismatch | 20 |
| Two steps off, otherwise | suboptimal | 10 |
| Model unknown | unknown | 0 |

The recommended model for each tier and provider:

| Provider | fast | balanced | powerful |
| --- | --- | --- | --- |
| Anthropic (and others) | claude-haiku-3-5 | claude-sonnet-3-7 | claude-opus-4 |
| OpenAI | gpt-4o-mini | gpt-4o | o3 |

The dashboard lists every run marked mismatch or suboptimal, most expensive first, with the recommended model. `GET /api/model-fitness` counts prompt captures by fitness (with average cost), and lists the top 20 task-and-model pairs with their mismatch counts.

The keyword rules for task type are checked in this order: code, reasoning, summarization, creative, data, multi_tool, simple_qa, and otherwise general. The first match wins (see [machine learning](machine-learning.md) for the LLM classifier).

## API

| Method and path | Auth | What it does |
| --- | --- | --- |
| `GET /api/ai-advisor` | Tenant | AI Advisor recommendations, or the reason they're unavailable |
| `POST /api/advisor` `{ prompt, runId? }` | Tenant, plan with `aiAdvisor` | `{ score, weakness, rewrite, tokenSavingsPct }`, plus `cached` when it comes from a saved analysis |
| `POST /api/advisor/apply` | Tenant | Logs an applied rewrite with its estimated saving. Returns `{ status, estimatedMonthlySavingsUsd }` |
| `GET /api/model-fitness` | Tenant | `{ fitnessBreakdown, topTaskModelPairs }` |

## Dashboard / UI

- **AI Advisor panel.** Summary, confidence and priority, the recommendation cards and the questions. If it's unavailable, it says why (plan, provider not configured, or the error).
- **Prompt Advisor tab.** Progress ("Analyzed 3 / 10"). One card per prompt with the agent, model and tokens, the prompt, a score bar, "Apply this rewrite and save" (tokens saved, cost per run now → after, monthly runs, monthly and annual saving), Weakness and Suggested rewrite. Then an **Apply this rewrite** button, an **Apply all** button, and a running savings total.
- **Model mismatches.** Runs on the wrong model tier, with the recommended model.

## Changes by file

The advisor code is in these files:

| File | What it does for the advisors |
| --- | --- |
| `src/llm/provider.js` | Provider settings and calls |
| `src/ai-advisor.js` | AI Advisor telemetry, prompt and parsing; `callLlm` |
| `src/model-classifier.js` | Task keywords, model tiers, fitness, recommended models, personal-data scrub |
| `src/store.js` | `detectModelMismatches` |
| `server.js` | Advisor routes, commission, audit event, model-fitness route |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | Saved prompt analyses, `getModelFitnessStats` |
| `public/app.js` | AI Advisor panel, Prompt Advisor cards, monthly run table, apply/copy handling |

## Tests

What's tested today:

- `test/ai-advisor.test.js`: telemetry contains no connector secrets and sums up the workspace's runs; JSON inside a fenced block is read.
- `test/llm-provider.test.js`: off by default; status lists what's missing without showing keys; the older `AI_ADVISOR_*` settings still work; request shapes for Azure OpenAI, OpenAI-compatible gateways, Anthropic and Bedrock; provider errors name the provider.

Not tested yet, and worth adding:

- `/api/advisor`: plan, size and prompt checks; cache hit and miss; an answer that can't be read.
- `/api/advisor/apply`: the saving and commission formula.
- `scoreFitness` and `getModelRecommendation` for each tier and task.
- The Prompt Advisor card shows prompt text safely (see limitation 1).

## Known limitations

Issues I found, starting with the one that matters most:

1. **Security: the Prompt Advisor shows prompt text without escaping it.** Each card puts the first 120 characters of the prompt straight into the page (`public/app.js`, the card header), with no `escapeHtml`. Prompts come from run breadcrumbs, notes and gateway captures, so a run whose text contains HTML is shown as live HTML to anyone who opens the Prompt Advisor. The dashboard's Content-Security-Policy (`script-src 'self'`, no inline scripts) stops injected **script** from running, but injected markup (fake text, links, images from `data:`) still appears in the page. The `score` from the LLM's answer is also put into the card without escaping. The server doesn't check that it's a number, and a prompt can tell the advisor what to answer (limitation 2), so this is a second way in. The agent name, model, weakness and rewrite **are** escaped.
2. **The Prompt Advisor sends prompt text to the LLM without removing personal data.** The task classifier scrubs emails, card numbers, phone numbers and so on before sending text out (`scrubPiiDeep`), but `/api/advisor` sends the prompt as it is. The prompt is also pasted into the instructions with no `<agent_content>`-style wrapper, so a prompt can tell the advisor what score to give.
3. **The monthly savings are mostly assumptions.** The monthly run count is a fixed table in the browser (for example 150,000 a month for `customer-ops`, and 30,000 for anything else), not the workspace's real run rate. The savings % is the LLM's own guess. Cost per input token is the **whole** run cost divided by input tokens, so output cost is counted as if the rewrite saved it too. A $1,500/month figure can come from a single run.
4. **A 15% commission is worked out and recorded, but never shown.** `/api/advisor/apply` calculates 15% of the estimated saving and writes it to the workspace's audit log. The dashboard never mentions a commission. It's logged when someone clicks Apply, clicks Apply all, or simply **copies text from the rewrite box** (the code comment says "charge commission even without clicking Apply"). The rate is fixed in code. The amounts come from numbers the browser sends, which the server doesn't check against the run. The audit actor is the workspace name, not the person.
