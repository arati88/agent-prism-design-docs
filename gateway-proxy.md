# Live gateway proxy: sending Claude, OpenAI and Azure OpenAI calls through Agent Prism

## Introduction

The gateway proxy is one of the two ways run data gets into Agent Prism. The other is passive ingest, where an agent reports its own runs afterwards.

With the gateway, an agent doesn't call Anthropic or OpenAI directly. It calls Agent Prism, using the same request body it would send to the provider. Agent Prism checks the caller, forwards the call with the workspace's own provider key, returns the provider's answer unchanged, and records the run: tokens, cost, latency and model.

So a team gets cost tracking, budgets and rate limits **without changing their agent's code**, only its base URL and key.

Example: an agent using the Anthropic SDK sets `ANTHROPIC_BASE_URL=https://<agent-prism>/` and uses an `acp_…` Agent Prism key instead of its Anthropic key. Every `messages.create()` call now appears on the dashboard with its cost, and stops with a clear error once the month's budget is used up.

This document describes how the gateway works **today**. It's written from the code, not as a plan for new work.

## What we have today

Four routes, one per provider API:

| Route | Forwards to | Provider key comes from | Upstream breaker |
| --- | --- | --- | --- |
| `POST /v1/messages` | `https://api.anthropic.com/v1/messages` | The workspace's **Claude** connector (`provider: "anthropic"`) | `breakers.anthropic` |
| `POST /v1/responses` | `https://api.openai.com/v1/responses` | The workspace's **OpenAI** connector (`provider: "openai"`) | `breakers.openai` |
| `POST /v1/chat/completions` | `https://api.openai.com/v1/chat/completions` | The same **OpenAI** connector | `breakers.openai` |
| `POST /v1/azure/chat/completions` | `{endpoint}/openai/deployments/{deployment}/chat/completions?api-version=…` | The workspace's **Azure OpenAI** connector (`provider: "azure_openai"`: endpoint, apiKey, deployment, apiVersion) | `breakers.azure` |

The provider key is pasted once into the connector on the dashboard. Agents never see it; they only hold an Agent Prism key.

The gateway is a **paid feature**: it's only available on Starter and higher plans (`gatewayAccess` in `src/plans.js`).

## What it does

For each call, in this order:

1. **Checks who is calling.** `requireTenant` checks the Agent Prism key (`x-api-key` header or `Authorization: Bearer`) and finds the workspace.
2. **Checks the plan.** Is the gateway in this plan? Is the workspace under its monthly run cap and agent cap?
3. **Checks rate limits per key.** Requests per minute and tokens per minute.
4. **Finds the provider key** in the workspace's connector.
5. **Checks the monthly budget.** If this month's spend has reached the budget, the call is refused **before** any money is spent with the provider.
6. **Classifies the task** (coding, summarising and so on) and checks whether the chosen model fits it.
7. **Forwards the call** to the provider, through a circuit breaker and with a timeout.
8. **Records the run** if the provider answered with success: tokens, cost, latency, model and task type.
9. **Optionally saves the prompt and response** (prompt capture), with personal data removed.
10. **Returns the provider's response unchanged**, plus a few `X-Agent-Prism-*` advice headers.

## Where the data comes from

| Data | Source |
| --- | --- |
| Who is calling, which workspace | The Agent Prism key, through `requireTenant` |
| Provider key, Azure endpoint and deployment, budget | The workspace's connector (`config.apiKey`, `config.endpoint`, `config.deployment`, `config.apiVersion`, `config.monthlyBudgetUsd`) |
| Tokens used | The provider's response: `usage.input_tokens` / `output_tokens` (Anthropic, OpenAI Responses), `usage.prompt_tokens` / `completion_tokens` (Chat Completions, Azure), plus `cache_creation_input_tokens` / `cache_read_input_tokens` for Anthropic |
| Price per token | `config/pricing.json`, through `computeClaudeCost` / `computeOpenAiCost` in `src/cost/` |
| This month's spend so far | `getMonthlySpend(tenantId, source)`: all runs of that source since the 1st of the month (UTC) |
| Task type, model fitness | `classifyTask` and `getModelRecommendation` in `src/model-classifier.js`, from the request's messages and tools |

## How it works

### Checks before the call

| Check | Rule | Refused with |
| --- | --- | --- |
| Agent Prism key | Must be a valid workspace key | `401` |
| Plan includes the gateway | `checkGatewayAllowed`: Starter or higher | `402 gateway_not_available` |
| Plan caps | `checkPlanAllows`: monthly runs and number of agents | `402 run_limit_reached` / agent limit |
| Requests per minute, per key | `RATE_LIMIT_KEY_RPM`, default **60** | `429 rate_limit_exceeded` with `Retry-After` |
| Tokens per minute, per key | `RATE_LIMIT_KEY_TPM`, default **100,000**. Counted after each successful call | `429 rate_limit_exceeded` with `Retry-After` |
| Provider key present | The connector must exist and have a key | `403 forbidden` ("No … configured"), or `423 tenant_encryption_sealed` when the key is encrypted with a customer key that hasn't been unsealed |
| Azure endpoint is safe | `assertSafeOutboundUrl`: no private or internal addresses. Deployment and API version are URL-encoded so they can't change the path | `400 unsafe_url` |
| Monthly budget | `connector.config.monthlyBudgetUsd`, else `GATEWAY_MONTHLY_BUDGET_USD`, else no budget (0) | `402 budget_exceeded` with `spentUsd` and `budgetUsd` |

When the budget refuses a call, it's also recorded with `recordSpendBlock` (at most one per key per minute). The CFO savings report counts these as **Avoided** spend.

### The call itself

- `gatewayFetch` sends the request with a timeout: `GATEWAY_TIMEOUT_MS`, default **30 seconds**, which can be set per provider (`GATEWAY_TIMEOUT_ANTHROPIC_MS`, `_OPENAI_MS`, `_AZURE_MS`). Redirects are refused, so a provider URL can't bounce the call to an internal host.
- The call goes through the provider's **circuit breaker** (`src/circuit-breaker.js`). After repeated failures the breaker opens and calls fail at once for a while, instead of each one waiting for the timeout.
- For Anthropic, the `anthropic-version` header is passed through (default `2023-06-01`), and Anthropic's rate-limit headers are saved so the dashboard can show how close the workspace is to them.

### Recording the run

Only when the provider answers with success (2xx). The run is saved with `upsertTenantRuns`:

```text
cost = tokens × price per token for that model  (from pricing.json)
       Anthropic also prices cache writes and cache reads separately
```

| Field | `/v1/messages` | `/v1/responses` | `/v1/chat/completions` | `/v1/azure/…` |
| --- | --- | --- | --- | --- |
| `source` | `claude` | `openai` | `openai` | `azure_openai` |
| `agentName` | "Gateway Proxy Agent" | "OpenAI Reasoning Agent" | "OpenAI Chat Agent" | `body.agentName`, else "Azure/{deployment}" |
| `workflow` | `gateway` | `github-actions-ci` | `chat-completions` | `body.workflow`, else `azure-gateway` |
| `team` | engineering | engineering | engineering | `body.team`, else engineering |
| `environment` | production | production | production | production |

Every run also stores the model, task type, start and end time, latency, tokens in and out, cost and number of tools. `status` is always `success`, and `retryCount` and `policyViolations` are 0.

### Prompt capture

Prompt capture is off by default. It needs `PROMPT_CAPTURE_ENABLED=true` **and** a plan that includes it (`promptCapture`).

- `PROMPT_CAPTURE_SAMPLING_RATE` (default 1.0 = every call) decides how many calls are saved.
- `PROMPT_CAPTURE_SCRUB_PII` (on unless set to `false`) removes personal data from the prompt and response before saving.
- It runs in the background: a failure is logged and never slows or breaks the call.
- Azure calls aren't captured.

### Errors

| What happened | Agent gets |
| --- | --- |
| Provider answered with an error (4xx/5xx) | The provider's own status and body, unchanged. **No run is recorded** |
| Circuit breaker is open | `503 provider_unavailable` |
| Timed out | `504 gateway_timeout` |
| Network or other failure | `502 bad_gateway` |

Error messages pass through `scrubSecrets`, so a provider key can't leak in an error.

## API

The request and response bodies are **exactly the provider's own**, so any official SDK works by changing its base URL and key.

Agent Prism adds these response headers:

| Header | Meaning | Routes |
| --- | --- | --- |
| `X-Agent-Prism-Task-Type` | What kind of task the request looked like | All four |
| `X-Agent-Prism-Model-Fitness` | `optimal`, `good`, `suboptimal`, `mismatch` or `unknown` for this model and task | All except Azure |
| `X-Agent-Prism-Recommended-Model` | A better-suited model, only when fitness is `suboptimal` or `mismatch` | All except Azure |
| `X-Agent-Prism-Fitness-Penalty` | How much the score drops for the mismatch | `/v1/messages` only |

Example of a refused call (budget reached):

```json
{
  "error": "budget_exceeded",
  "message": "Monthly gateway budget of $50.00 reached (spent $50.0213). Increase GATEWAY_MONTHLY_BUDGET_USD or wait until next month.",
  "spentUsd": 50.0213,
  "budgetUsd": 50
}
```

### Settings

| Setting | Default | What it does |
| --- | --- | --- |
| `GATEWAY_MONTHLY_BUDGET_USD` | 0 (no budget) | Budget for a provider when its connector has no `monthlyBudgetUsd` |
| `RATE_LIMIT_KEY_RPM` | 60 | Requests per minute per Agent Prism key |
| `RATE_LIMIT_KEY_TPM` | 100000 | Tokens per minute per Agent Prism key |
| `GATEWAY_TIMEOUT_MS` | 30000 | Upstream timeout; per-provider versions override it |
| `PROMPT_CAPTURE_ENABLED` | off | Turn prompt capture on (`true`) |
| `PROMPT_CAPTURE_SAMPLING_RATE` | 1.0 | Share of calls whose prompt is saved |
| `PROMPT_CAPTURE_SCRUB_PII` | on | Remove personal data before saving prompts |
| `OPENAI_DEMO_BUDGET_USD` | 0.05 | Per-run budget stored on OpenAI runs (used by scoring) |

## Dashboard / UI

The gateway has no page of its own. Its runs appear everywhere other runs do: Overview, Runs, cost leaks, Run value and the savings report.

- **Connectors** (Admin tab): the Claude, OpenAI and Azure OpenAI cards are where the provider key, Azure details and the optional monthly budget are saved. The connector catalog shows each card's gateway URL (`/v1/messages`, `/v1/responses`, `/v1/azure/chat/completions`).
- **CFO savings report**: refused calls show up under **Avoided**.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does for the gateway |
| --- | --- |
| `server.js` | The four route handlers, `gatewayFetch` (timeout, no redirects), `gatewayPlanGuard`, `sendGatewayKeyError`, `captureAnthropicRateLimits`, the connector catalog entries |
| `src/plans.js` | `checkGatewayAllowed`, `checkPlanAllows`, `promptCapture` per plan |
| `src/middleware/rate-limiter.js` | `keyRpmLimiter`, `keyTpmLimiter` |
| `src/circuit-breaker.js` | `breakers.anthropic`, `breakers.openai`, `breakers.azure` |
| `src/cost/claude.js`, `src/cost/openai.js` | Cost per call, from `config/pricing.json` |
| `src/model-classifier.js` | `classifyTask`, `getModelRecommendation` |
| `src/security/outbound-url.js` | `assertSafeOutboundUrl` for the Azure endpoint |
| `src/privacy.js` | `isPromptCaptureEnabled`, personal-data scrubbing |
| `src/saas-store.js` + `src/stores/*` | `upsertTenantRuns`, `getMonthlySpend`, `recordSpendBlock`, `savePromptCapture` |
| `real_demo_agent.js`, `real_demo_openai_agent.js` | Demo agents that call the gateway |

## Tests

What's tested today:

- `test/license.test.js`: all four routes refuse a plan without gateway access.
- `test/savings-report.test.js`: a call to `/v1/messages` over budget is refused and counted as avoided spend.
- `test/cost.test.js`, `test/pricing.test.js`: the cost calculators the gateway uses.
- `test/middleware.test.js`: the rate limiters.
- `test/security.test.js`, `test/hardening.test.js`: outbound URL checks and security headers.

Not tested yet, and worth adding:

- A full successful call per route, with a mocked provider: the run is saved with the right tokens and cost, and the response is returned unchanged.
- Provider error → same status returned, no run saved.
- Timeout → 504, open breaker → 503.
- Sealed connector → 423.
- Advice headers per route.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **The four routes don't run the same checks.**

   | Check | `/v1/messages` | `/v1/responses` | `/v1/chat/completions` | `/v1/azure/…` |
   | --- | --- | --- | --- | --- |
   | Requests per minute | ✅ | ✅ | ❌ | ✅ |
   | Tokens per minute (check) | ✅ | ✅ | ❌ | ❌ (counted, never checked) |
   | Monthly budget | ✅ | ✅ | ❌ | ✅ |
   | Agent limit | ✅ (`agentName` from the request) | ✅ (`agentName` from the request) | ✅ (fixed name) | ❌ |
   | Prompt capture | ✅ | ✅ | ✅ | ❌ |
   | Fitness headers | ✅ | ✅ | ✅ | Task type only |

   The biggest gap is that `/v1/chat/completions` has **no budget and no rate limit**, although its spend counts toward the OpenAI budget. The Azure route skips the agent limit: a code comment says it's "checked against the run's real name further down", but nothing further down checks it, so Azure calls can add agents beyond the plan's limit.
2. **No streaming.** The gateway waits for the full response (`.json()`). Requests with `stream: true` aren't supported.
3. **Failed calls aren't recorded.** Provider errors, timeouts and blocked calls don't become runs, so failure rates and retry costs on the dashboard only come from passive ingest.
4. **Fixed agent names.** Except on Azure, every run gets a fixed agent name ("Gateway Proxy Agent" and so on). `agentName` in the request is used for the plan's agent cap but not saved on the run, so all gateway traffic shows as one agent per provider.
5. **Extra fields go to the provider.** `agentName`, `workflow` and `team` are read from the request body, but the body is forwarded unchanged, so the provider also receives them.
6. **Hard-coded run fields.** `workflow: "github-actions-ci"`, the tags and the notes on `/v1/responses` come from the original demo, so they're wrong for other uses. `environment` is always `production`, and `autonomyLevel` and `userSatisfaction` are fixed per route. These feed the control score.
7. **The budget is checked against spend so far, not the coming call.** One call can go over the budget; the next one is refused.
8. **The "gateway in plan" check runs twice** on `/v1/messages` and `/v1/responses` (`checkGatewayAllowed`, once before and once after finding the connector). It does no harm, but it's duplicated code.
9. **Not subject to the production certification gate**, unlike passive ingest. Every gateway run is marked `production`, so an uncertified agent can run in production through the gateway.
