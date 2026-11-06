# Node.js SDK and CLI

## Introduction

Agent Prism includes a small **Node.js SDK** (`src/sdk/index.js`) and a **CLI** (`bin/cli.js`) so a developer can send agent runs without writing HTTP calls themselves.

- **CLI.** `agent-prism login` asks for a workspace API key (`acp_…`) and saves it with an endpoint to `~/.agent-prism/credentials.json`. `agent-prism logout` deletes that file.
- **SDK.** `new AgentPrism()` finds the API key (from options, environment variables or the CLI file). It exchanges the key for a 1-hour access token at `/api/oauth/token` and reuses it. Its one method, `logRun(payload)`, sends a run to `POST /api/ingest`.

The demo scripts (`real_demo_agent.js`, `real_demo_openai_agent.js`, `real_demo_copilot_agent.js`, `setup_gateway.js`, `setup_openai_gateway.js`) use the SDK to find credentials. Some then call the gateway or `/api/connectors` themselves, using `prism.clientSecret` and `prism.endpoint`.

Example:

```js
import { AgentPrism } from "./src/sdk/index.js";

const prism = new AgentPrism();                       // reads ~/.agent-prism/credentials.json
await prism.logRun({
  agentName: "invoice-bot", status: "success", environment: "staging",
  model: "claude-haiku-4-5", tokensIn: 8000, tokensOut: 900, costUsd: 0.0125,
  toolManifest: [{ name: "search_invoices" }, { name: "send_email" }]
});
```

The first call gets an access token, and later calls within the hour reuse it. The run then goes through normal passive ingest: it's normalised, tool danger is classified, the plan and certification checks run, and it's saved (see [passive ingest](passive-ingest.md)).

Both are described as they are in the repo today.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/sdk/index.js` | `AgentPrism` class: `constructor(options)`, `_authenticate()`, `logRun(payload)` |
| `bin/cli.js` | `login`, `logout`. Exposed as `agent-prism` in `package.json` `bin` |
| `real_demo_*.js`, `setup_*gateway.js` | Examples that use the SDK for credentials |
| Server | `POST /api/oauth/token` (see [auth](auth-sso-and-scim.md)), `POST /api/ingest` (see [passive ingest](passive-ingest.md)) |

## What it does

- Lets a developer report agent runs with two lines of code.
- Keeps the API key out of code and `.env` files by storing it once on the developer's machine.
- Uses a short-lived access token on the wire, not the API key, for each run.

## Where the data comes from

**Credentials**, in this order:

| Setting | From |
| --- | --- |
| API key (`clientSecret`) | `options.clientSecret` → `AGENT_PRISM_API_KEY` → `~/.agent-prism/credentials.json` `apiKey` |
| Endpoint | `options.endpoint` → `AGENT_PRISM_ENDPOINT` → the credentials file's `endpoint` → `http://127.0.0.1:3000` |
| Client id | `options.clientId` → `AGENT_PRISM_CLIENT_ID` → `"cli_user"` (the server ignores it) |

If no API key is found, the constructor throws: "Not authenticated. Run `npx agent-prism login` or provide clientSecret."

**The CLI file** is written with permissions `0600`: `{ "apiKey": "<pasted key>", "endpoint": "<AGENT_PRISM_URL or https://agent-prism.onrender.com>" }`.

**The run payload** is passed to `/api/ingest` as it is. The fields that matter are the generic run fields (agent name, status, model, tokens, cost, budget, latency, environment and so on), plus `toolManifest` and `humanApprovals` for certification (see [agent certification](agent-certification.md)).

## How it works

### 1. Getting a token (`_authenticate`)

1. If there's an access token that expires **more than 5 minutes** from now, use it.
2. Otherwise send `POST {endpoint}/api/oauth/token` with `{ client_id, client_secret, grant_type: "client_credentials" }`.
3. On error, throw "Agent Prism Authentication Failed: <response text>".
4. Store `access_token` and its expiry (`expires_in` × 1000 from now). The `refresh_token` in the response isn't used.

### 2. Sending a run (`logRun`)

1. Get a token.
2. Send `POST {endpoint}/api/ingest` with the payload as JSON and `Authorization: Bearer <token>`.
3. If the status isn't 2xx, throw "Agent Prism Telemetry Push Failed (<status>): <response text>".
4. Return the response JSON: `ingested` or `duplicate`, or `202 queued` when the server saved it to the retry queue.

### 3. CLI

- `login`: prints instructions, asks for the API key, creates `~/.agent-prism/` if needed, and saves the file. The endpoint comes from `AGENT_PRISM_URL`, or defaults to the hosted demo URL.
- `logout`: deletes the file if it's there.
- Anything else: prints usage.

## API

| SDK | Calls | Notes |
| --- | --- | --- |
| `new AgentPrism({ clientId?, clientSecret?, endpoint? })` | — | Throws when there's no API key |
| `prism.logRun(payload)` | `POST /api/oauth/token` (when needed), `POST /api/ingest` | Returns the ingest response; throws when the status isn't 2xx |
| `prism.clientSecret`, `prism.endpoint` | — | Used by the demos to call the gateway and connectors directly |

| CLI | Effect |
| --- | --- |
| `agent-prism login` | Saves the API key and endpoint |
| `agent-prism logout` | Deletes saved credentials |

## Dashboard / UI

None. The developer copies an API key from **Admin → API keys** and pastes it into `agent-prism login`.

## Changes by file

The SDK and CLI code:

| File | What it does |
| --- | --- |
| `src/sdk/index.js` | SDK |
| `bin/cli.js` | CLI |
| `package.json` | `bin.agent-prism` |
| `real_demo_*.js`, `setup_*gateway.js` | Examples |

## Tests

Nothing tests the SDK or the CLI. `test/privacy.test.js` leaves `src/sdk` out of its outbound-host scan on purpose.

Worth adding:

- Credential order (options → environment → file).
- Token reuse and refresh 5 minutes before expiry.
- `logRun` error messages for `401`, `402`, `403` and `5xx`.
- CLI login writes the file with `0600`, and logout removes it.

## Known limitations

Problems a new SDK user is likely to hit:

1. **The first `logRun` fails unless the run says `environment`.** Ingest treats a run without `environment` as **production**, and an agent without a production certificate is refused with `403 agent_not_certified` (see [agent certification](agent-certification.md)). A new user following the simple example gets an error until they add `environment: "staging"` or certify the agent. The SDK doesn't mention this.
2. **The CLI sends telemetry to the hosted demo by default.** Without `AGENT_PRISM_URL`, `login` saves `https://agent-prism.onrender.com` as the endpoint and tells the user to get their key from that site. A customer running their own installation who doesn't set the variable sends runs, or fails to authenticate, against the demo server. The SDK's own default is `http://127.0.0.1:3000`, which is different.
3. **The SDK isn't really published.** It lives inside the server package (`agent-prism`, with all server dependencies). There's no separate package, no `exports` entry, no type definitions and no version of its own. The demos import it by relative path, so "`npx agent-prism login`" and "install the SDK" only work from a copy of this repository.
4. **It does one thing.** There's only `logRun`, sent to `/api/ingest`. There's nothing for sessions or usage, the Claude, Copilot or generic ingest routes, fleet snapshots, or reading data back, and no help for the gateway beyond exposing the raw key.
5. **No timeout, retry or idempotency.** `fetch` has no timeout, so an unreachable server hangs the caller. Nothing is retried on network errors, `429` or `5xx`. No `Idempotency-Key` or run id is generated, so a caller that retries by itself can store the same run twice unless it sets `id`.
6. **A revoked or expired token isn't recovered.** If the server revokes the key or the token is refused, `logRun` throws. It doesn't clear the token and try again.
7. **The token exchange adds little.** The SDK holds the long-lived `acp_` key anyway and asks for a new access token with it every hour, ignoring the refresh token. Each exchange creates a refresh token that's never used and writes an `OAuth Token Issued` audit entry, for every running process, every hour.
8. **The demos hand the raw API key to other libraries.** `real_demo_agent.js` passes `prism.clientSecret` to the Anthropic SDK as its `apiKey`, so the long-lived key goes on every gateway call, not a short-lived token.
9. **Credentials are a plain-text file.** `~/.agent-prism/credentials.json` holds the key unencrypted. `0600` protects it on Linux and macOS but means nothing on Windows. There's one profile per machine user, so you can't keep two workspaces.
10. **Errors include the whole response body**, which may be long or contain HTML from a proxy.
