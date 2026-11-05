# AI providers: the LLM switch for Agent Prism's own AI features

## Introduction

Agent Prism's own AI features are the **AI Advisor**, the **Prompt Advisor**, the optional **task classifier**, and an admin **connection test**. They all go through one place, `src/llm/provider.js`, and use **one model service per installation**, chosen with environment variables:

| `AI_PROVIDER` | Service | Sign-in without a stored key |
| --- | --- | --- |
| `none` (default) | AI features off | — |
| `azure-openai` | Azure OpenAI / Azure AI Foundry | Managed or workload identity, service principal |
| `aws-bedrock` | Amazon Bedrock (Converse API) | IAM role, IRSA, instance profile, SSO profile |
| `gcp-vertex` | Google Vertex AI (OpenAI-compatible endpoint) | Service account, Workload Identity |
| `openai-compatible` | Any OpenAI-compatible endpoint: LiteLLM, an internal AI gateway, vLLM, NIM, Databricks… | — |
| `openai`, `anthropic` | Public APIs | — |
| `ollama`, `openrouter` | Development only | — |

Nothing is sent anywhere until a provider is set. This is **separate** from the gateway (`/v1/*`), which forwards **customers' agents'** calls to **their** providers (see [gateway proxy](gateway-proxy.md)).

The user guide is [docs/AI-PROVIDERS.md](../AI-PROVIDERS.md). This document is the design view: how the switch works, and where the guide and the code don't match.

Example: a bank runs Agent Prism on AWS. It sets `AI_PROVIDER=aws-bedrock`, `AWS_REGION=eu-west-1`, `AI_MODEL=<inference-profile ARN>` and `AWS_BEDROCK_ENDPOINT=<PrivateLink URL>`, and gives the task role `bedrock:Converse`. There's no key to store. Admin → AI service shows "Amazon Bedrock · platform identity (no static key)", and **Test connection** answers "OK" in 640 ms.

This document describes the switch **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/llm/provider.js` | `llmSettings` (reads the environment on every call), `isLlmConfigured`, `llmStatus` (safe to show), `completeText` (one system and one user message → text) |
| `src/ai-advisor.js` | AI Advisor and `callLlm` for the Prompt Advisor (see [AI Advisor and Prompt Advisor](ai-advisor-and-prompt-advisor.md)) |
| `src/ml/llm-classifier.js`, `src/privacy.js` | Task classifier, on only with `LLM_CLASSIFIER_ENABLED=true` (see [machine learning](machine-learning.md)) |
| `server.js` | `GET /api/ai/status`, `POST /api/ai/test`, plan checks on the advisor routes, classifier wiring at startup |
| `package.json` | `@azure/identity`, `@aws-sdk/client-bedrock-runtime`, `google-auth-library`, loaded only when that provider is used |

## What it does

- Lets each installation use the AI service its security team has already approved, including private endpoints and corporate gateways.
- Avoids storing keys on the major clouds, by using the platform's identity.
- Keeps AI off by default, so nothing leaves the installation unless an operator turns it on.
- Shows exactly which setting is missing, and lets an admin test the connection, without ever showing a key.

## Where the data comes from

**Settings** (read again on every call). The older `AI_ADVISOR_PROVIDER`, `AI_ADVISOR_MODEL` and `AI_ADVISOR_TIMEOUT_MS` still work.

| Provider | Required | Optional |
| --- | --- | --- |
| all | `AI_PROVIDER` | `AI_MODEL`, `AI_TIMEOUT_MS` (30,000) |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_DEPLOYMENT` (or `AI_MODEL`) | `AZURE_OPENAI_API_VERSION` (2024-10-21), `AZURE_OPENAI_API_KEY`; without a key, `DefaultAzureCredential` |
| `aws-bedrock` | `AWS_REGION` (or `AWS_DEFAULT_REGION`), `AI_MODEL` | `AWS_BEDROCK_ENDPOINT` (PrivateLink); the standard AWS credential chain |
| `gcp-vertex` | `GOOGLE_CLOUD_PROJECT`, `AI_MODEL` | `GOOGLE_CLOUD_LOCATION` (us-central1, or `global`); Application Default Credentials |
| `openai-compatible` | `AI_BASE_URL`, `AI_MODEL` | `AI_API_KEY` |
| `openai` | `OPENAI_API_KEY`, `AI_MODEL` | — |
| `anthropic` | `ANTHROPIC_API_KEY` | `AI_MODEL` (default `claude-haiku-4-5-20251001`) |
| `ollama` | — | `OLLAMA_BASE_URL` (http://127.0.0.1:11434), `AI_MODEL` (llama3.1) |
| `openrouter` | `OPENROUTER_API_KEY`, `AI_MODEL` | `OPENROUTER_BASE_URL` |

**Who calls it:**

| Feature | Max tokens back | Sent |
| --- | --- | --- |
| AI Advisor | 1,024 | A workspace telemetry summary (see [AI Advisor](ai-advisor-and-prompt-advisor.md)) |
| Prompt Advisor | 300 | One prompt text, with instructions |
| Task classifier | 12 | Up to 800 characters of prompt, with personal data removed and wrapped as data |
| Connection test | 5 | "Reply with exactly the word OK." |

## How it works

### 1. Status (`llmStatus`)

`{ provider, label, model, configured, missing[], auth }`. `configured` is true when the provider isn't `none` and nothing is missing. `auth` is "platform identity (no static key)" or "API key". Keys and secret URLs are never included. An unknown `AI_PROVIDER` shows up as a missing item.

### 2. A call (`completeText({ system, user, maxTokens })`)

1. If the provider is `none` or a setting is missing, throw with the reason.
2. Call the provider:

| Provider | Request |
| --- | --- |
| Azure OpenAI | `POST {endpoint}/openai/deployments/{deployment}/chat/completions?api-version=…`, with header `api-key` or `Authorization: Bearer <Entra token>` (scope `cognitiveservices.azure.com`) |
| Bedrock | AWS SDK `ConverseCommand` (system, one user message, maxTokens, temperature 0.2), with an abort signal at the timeout |
| Vertex | `POST https://{location}-aiplatform.googleapis.com/v1/projects/{p}/locations/{l}/endpoints/openapi/chat/completions` (host `aiplatform.googleapis.com` when the location is `global`), with a Google access token |
| Anthropic | `POST https://api.anthropic.com/v1/messages`, `anthropic-version: 2023-06-01` |
| Others | `POST {baseUrl}/chat/completions`, with a Bearer key when there is one |

3. Every HTTP call uses the timeout (`AI_TIMEOUT_MS`) and **refuses redirects**. Chat calls use temperature 0.2.
4. A non-2xx answer throws "`<provider> returned HTTP <status>: <first 200 characters>`".
5. Return the text, joining the content parts.

Credential objects (the Azure credential, Google auth, the Bedrock client) are made once and reused; each SDK refreshes its own tokens.

### 3. Turning features on

| Feature | On when |
| --- | --- |
| AI Advisor, Prompt Advisor | A provider is configured **and** the workspace's plan has `aiAdvisor` (Starter and above) |
| Task classifier | A provider is configured **and** `LLM_CLASSIFIER_ENABLED=true` at startup |
| Connection test | A provider is configured; admin only |

## API

| Method and path | Auth | Returns |
| --- | --- | --- |
| `GET /api/ai/status` | `settings.manage` (admin) | `llmStatus()` |
| `POST /api/ai/test` | `settings.manage`, rate limited for each workspace | `{ ok, status, latencyMs, reply }` or `{ ok: false, error }` (error scrubbed, at most 300 characters) |

The AI Advisor and Prompt Advisor routes are in [AI Advisor and Prompt Advisor](ai-advisor-and-prompt-advisor.md).

## Dashboard / UI

- **Admin → AI service:** provider, model and how it signs in, any missing settings, and **Test connection**.
- **Token Coach → AI Advisor panel:** the provider and model, or why it's unavailable.
- **Admin → Machine learning model status:** the classifier's provider when it's on.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does |
| --- | --- |
| `src/llm/provider.js` | Settings, status, calls |
| `src/ai-advisor.js`, `src/ml/llm-classifier.js` | Callers |
| `src/privacy.js` | Classifier switch |
| `server.js` | Status and test routes, plan checks, classifier wiring |
| `docs/AI-PROVIDERS.md` | Operator guide |

## Tests

What's tested today:

- `test/llm-provider.test.js`: off by default; status lists what's missing and never shows keys; the old setting names still work; request shapes for Azure OpenAI (key), OpenAI-compatible gateways and Anthropic; provider errors name the provider and status; a real AWS SDK call (SigV4-signed Converse) against a private endpoint.
- `test/ai-service.test.js`: status shows the provider without its key; the connection test makes one real call and reports latency; only admins can see or test the AI service.

Not tested yet, and worth adding:

- Azure without a key, and Vertex: token use.
- Timeout behaviour for each provider.
- Refused redirects.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **One provider for the whole installation.** Every workspace's AI Advisor and Prompt Advisor data goes to the same model service, chosen by the operator. On a shared installation a customer can't choose their own provider, keep their data out of the operator's provider, or turn AI off for themselves; only the plan decides.
2. **The user guide doesn't match the code on what's sent:**
   - It says the Prompt Advisor sends "a prompt the user chose to submit". In fact the dashboard sends breadcrumb, notes or captured prompt text from a page of recent runs automatically, without removing personal data (see [AI Advisor and Prompt Advisor](ai-advisor-and-prompt-advisor.md)).
   - It says the AI Advisor sends "aggregate metrics only". The summary also includes the workspace's id and name, workflow names and details of the last 12 runs.
3. **Agent Prism's own AI spend isn't tracked.** The calls made by the advisors and classifier aren't recorded as runs or cost anywhere, there's no budget or limit on them, and the AI Advisor runs on every dashboard load (see [AI Advisor and Prompt Advisor](ai-advisor-and-prompt-advisor.md)).
4. **No retry.** One timeout or `429` and the feature reports "unavailable". The default timeout is 30 seconds, and the dashboard waits for it.
5. **Provider errors are shown to users.** Up to 200 characters of the provider's response are put in the error. The AI Advisor panel shows that message to anyone in the workspace, which can reveal deployment names, account ids or policy details.
6. **Fixed request settings.** Temperature 0.2, no JSON mode, one user message only. Prompt instructions and the response format are written into each feature.
7. **Old model defaults.** `anthropic` defaults to `claude-haiku-4-5-20251001` and `ollama` to `llama3.1`. The task classifier names a model it doesn't actually use (see [machine learning](machine-learning.md)).
8. **The Bedrock path ignores the corporate proxy.** `NODE_USE_ENV_PROXY`/`HTTPS_PROXY` applies to `fetch`, but the AWS SDK uses its own HTTP stack, so Bedrock needs a VPC endpoint or direct egress. The guide mentions this.
9. **Endpoints aren't checked.** `AI_BASE_URL` and `AZURE_OPENAI_ENDPOINT` can be any address, including plain HTTP. It's operator configuration, but nothing warns about an unencrypted endpoint.
10. **The classifier switch is read only at startup**, while the other settings are read on every call. Turning the classifier on or off needs a restart.
