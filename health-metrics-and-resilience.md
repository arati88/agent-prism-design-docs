# Health checks, metrics, circuit breaker, startup, shutdown and cluster mode

## Introduction

This document covers how Agent Prism is **run and watched** in production:

- **Startup checks** (`validateConfig`). The server refuses to start with unsafe production settings, such as the default admin secret or no encryption key, and warns about risky ones. On Postgres it then runs migrations and schema patches under a lock.
- **Health checks.** `/api/health/live` (is the process up), `/api/health/ready` (should traffic come here) and `/api/health` (a fuller status).
- **Metrics.** `/metrics` in Prometheus text format: request count, errors, a latency histogram, memory, rate-limiter sizes and circuit states.
- **Circuit breaker.** One per upstream provider (Anthropic, OpenAI, Azure). After 5 failures in a row, gateway calls to that provider fail fast for 30 seconds, then a few test calls decide whether to close it again.
- **Graceful shutdown.** On SIGTERM or SIGINT, the server stops taking new connections, lets in-flight requests finish (up to 30 seconds in production), then exits.
- **Cluster mode** (`npm run cluster`). One worker per CPU behind a primary process, which restarts any worker that exits.
- **Logs.** One JSON line per request, and error lines with secrets masked.

Example: Anthropic's API stops answering. Five gateway calls in a row time out, so the `anthropic` circuit **opens**. An alert goes to `CB_ALERT_WEBHOOK_URL`, and for the next 30 seconds `/v1/messages` fails straight away with "temporarily unavailable". `/api/health/ready` now returns `503`. After 30 seconds the next call is let through as a probe, and after 2 successful probes the circuit **closes** with another alert.

It describes how these work in the code right now.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/startup.js` | `validateConfig()`: fatal errors and warnings about settings at boot |
| `server.js` (end of file) | Licence log line, `prepareDatabase()` (migrations and schema patches under advisory lock `815204117`), background jobs, `server.listen`, shutdown setup |
| `src/shutdown.js` | `setupGracefulShutdown`, `isShuttingDown` |
| `src/cluster.js` | Primary process that forks workers and restarts them with backoff |
| `src/circuit-breaker.js` | `CircuitBreaker`, `breakers.{anthropic, openai, azure}` |
| `src/middleware/metrics.js` | Counters, latency histogram, Prometheus text |
| `src/middleware/logger.js` | `logRequest`, `logError`, `scrubSecrets` |
| `src/middleware/request-id.js` | `X-Request-ID` on every response |
| `server.js` | `/api/health*`, `/metrics`, the request wrapper (in-flight count, metrics, logging), `gatewayFetch` (timeouts, no redirects) |

## What it does

- Stops a misconfigured production server from starting, instead of letting it run unsafely.
- Tells a load balancer or Kubernetes when an instance is alive, when it's ready, and when it's draining.
- Gives the operations team numbers to graph and alert on.
- Protects the gateway from piling up requests on a provider that's down.
- Lets deploys and restarts happen without cutting off requests.
- Uses every CPU core on one machine, and recovers from a crashed worker.

## Where the data comes from

| Setting | Default | Purpose |
| --- | --- | --- |
| `NODE_ENV` | (development) | `production` turns on the strict startup checks and `Secure` cookies |
| `CB_FAILURE_THRESHOLD` | 5 | Failures in a row that open a circuit |
| `CB_SUCCESS_THRESHOLD` | 2 | Successful probes that close it |
| `CB_RESET_TIMEOUT_MS` | 30000 | How long a circuit stays open before probing |
| `CB_ALERT_WEBHOOK_URL` | — | Where to POST open and close alerts |
| `SHUTDOWN_TIMEOUT_MS` | 30000 in production, 10000 otherwise | How long to wait for in-flight requests |
| `CLUSTER_WORKERS` | number of CPUs | Workers in cluster mode |
| `DB_POOL_MAX` | 10 | Postgres connections per worker |

## How it works

### 1. Startup checks (`validateConfig`)

**Stops the server** (exit code 1):

| Condition | When |
| --- | --- |
| `STORAGE_BACKEND=postgres` without `DATABASE_URL` | Always |
| `DB_SSL_CA` points to a missing file | Always |
| `CORS_ALLOWED_ORIGINS=*` | Always |
| `ENCRYPTION_KEY` not set | Production |
| `ACP_ADMIN_SECRET` is the default `change-me-before-production` | Production |
| `DASHBOARD_USERNAME` or `DASHBOARD_PASSWORD` not set, or the password is `change-me` | Production |
| `JWT_SECRET` not set, or the same as `ACP_ADMIN_SECRET` | Production |

**Warns:**

- `NODE_ENV` is missing or unknown.
- Production with Postgres TLS that doesn't verify the server certificate.
- The file backend in production.
- `ENCRYPTION_KEY` isn't 64 hex characters.
- The admin secret is shorter than 16 characters, or the JWT secret is shorter than 32.
- `CORS_ALLOWED_ORIGINS` is empty in production.
- `DB_POOL_MAX × CLUSTER_WORKERS` is more than 80 connections.
- In production, `CLAUDE_WEBHOOK_SECRET` or `COPILOT_WEBHOOK_SECRET` isn't set.

Then, on Postgres, `prepareDatabase()` takes an advisory lock, runs migrations and then schema patches, and sets `databasePrepared = true`. If preparation fails in production, the process exits so the platform restarts it (see [storage backends](storage-backends-and-migrations.md)). The server starts listening straight away. Until preparation finishes, readiness reports not ready.

### 2. Health checks

| Route | Auth | Returns | 200 when |
| --- | --- | --- | --- |
| `GET /api/health/live` | Public | `{ ok, pid, uptime }` | Always (the process can answer) |
| `GET /api/health/ready` | Public | `ready`, `db { ok, latencyMs, error? }`, `circuits`, `bootstrapped`, `databasePrepared` | The database answers, **no circuit is open**, preparation is finished, and the server isn't shutting down. Otherwise `503`. In production the database error is hidden ("database unavailable") |
| `GET /api/health` | Public | `ok`, storage backend, `bootstrapped`, `db` (with the driver's error), `circuits`, uptime, memory in MB, pid | The database answers and no circuit is open. Otherwise `503` |

The database check is `SELECT 1`. The file backend always counts as healthy.

### 3. Metrics (`GET /metrics`, needs `X-Admin-Secret`)

| Metric | Type |
| --- | --- |
| `http_requests_total` | counter: every finished response |
| `http_errors_total` | counter: responses with status 500 or above |
| `http_request_duration_ms` | histogram, buckets 10, 50, 100, 250, 500 and 1,000 ms, then 2.5, 5, 10 and 30 s, and +Inf |
| `process_uptime_seconds`, `process_memory_rss_bytes`, `process_memory_heap_used_bytes`, `nodejs_active_handles` | gauges |
| `rate_limiter_tenant_buckets`, `rate_limiter_key_rpm_buckets`, `rate_limiter_key_tpm_buckets` | gauges: entries in each rate limiter |
| `circuit_breaker_<provider>_open` | gauge: 1 when open |

### 4. Circuit breaker

```text
CLOSED ── failure count reaches 5 ──► OPEN ── 30 s passed, next call ──► HALF_OPEN
  ▲                                     ▲                                   │
  │                                     └──────── a probe fails ────────────┤
  └───────────────────────── 2 probes succeed ──────────────────────────────┘
```

- Each gateway call to a provider goes through `breakers.<provider>.execute(...)`: Anthropic for `/v1/messages`, OpenAI for `/v1/responses` and `/v1/chat/completions`, Azure for `/v1/azure/*`.
- A **failure** is the call **throwing**: a network error, a refused redirect, or a timeout from `gatewayFetch`.
- Any success resets the failure count.
- While open, calls fail at once with `isCircuitOpen` and "Retry after Ns", and the gateway returns an error (see [gateway proxy](gateway-proxy.md)).
- Opening and closing are logged as JSON (`circuit_open`, `circuit_closed`) and sent to `CB_ALERT_WEBHOOK_URL` with a 5-second timeout, best effort.

### 5. Each request

For every request the server:

- counts it as in flight;
- attaches an `X-Request-ID`, using the caller's `X-Request-ID` or `X-Trace-ID` (up to 64 characters) or making a new one;
- sets `Connection: close` while shutting down.

When the response finishes, it writes one JSON log line to stdout (time, method, path, status, latency, workspace id, first 12 characters of the key) and updates the metrics. Errors go to stderr through `logError`, with API keys, bearer tokens, provider keys, GitHub tokens, database passwords and `api_key=` query values masked. Headers are never logged.

### 6. Graceful shutdown

On SIGTERM or SIGINT, once only:

1. Readiness starts returning `503`, so the load balancer stops sending traffic.
2. `server.close()` stops new connections. Idle keep-alive connections are closed at once. Requests already running finish, and their responses carry `Connection: close`.
3. When all connections are closed, the process exits with code 0.
4. If that hasn't happened after `SHUTDOWN_TIMEOUT_MS`, it logs how many requests were still running and exits with code 1.

### 7. Cluster mode (`npm run cluster`)

- It refuses to start with the file backend and more than one worker (exit 1), because workers would overwrite each other's file.
- It forks `CLUSTER_WORKERS` workers, each running `server.js` and sharing the port.
- When a worker exits, it's started again. If it lived less than 10 seconds, the wait before the restart doubles each time (from 1 second up to 30 seconds), and goes back to 0 once a worker stays up.

## API

| Method and path | Auth | Purpose |
| --- | --- | --- |
| `GET /api/health/live` | Public | Liveness probe |
| `GET /api/health/ready` | Public | Readiness probe |
| `GET /api/health` | Public | Status for people |
| `GET /metrics` | Platform admin (`X-Admin-Secret`) | Prometheus metrics |
| `GET /api/admin/diagnose` | Platform admin | Database tables and the demo user (see [auth](auth-sso-and-scim.md)) |

## Dashboard / UI

There's no operations screen. Health and metrics are for the platform team's load balancer, Kubernetes probes and Prometheus/Grafana. `/api/metrics/summary` returns `platformStatus` (see limitation 9).

## Changes by file

Where the operations code lives:

| File | What it does for operations |
| --- | --- |
| `src/startup.js` | Startup checks |
| `src/shutdown.js` | Graceful shutdown |
| `src/cluster.js` | Cluster mode |
| `src/circuit-breaker.js` | Circuit breakers and alerts |
| `src/middleware/metrics.js`, `logger.js`, `request-id.js` | Metrics, logs, request id |
| `server.js` | Health routes, `/metrics`, request wrapper, `gatewayFetch`, database preparation, startup order |
| `src/stores/postgres-store.js` | `pingDb` |

## Tests

What's tested today:

- `test/security.test.js`: `validateConfig` exits with code 1 when the default secret is used in production.
- `test/hardening.test.js`: readiness reports database preparation and never returns 500. The test servers wait on `/api/health/live`.
- `test/postgres.integration.test.js`: a lost database connection gives `503`, not `500`, and liveness keeps answering.

Not tested yet, and worth adding:

- The circuit breaker: opening after 5 failures, failing fast, half-open probing, closing after 2 successes, and the alert.
- Graceful shutdown: in-flight requests finish, readiness goes to 503, and the forced exit happens after the timeout.
- Cluster mode: refusing the file backend, and the restart backoff.
- `/metrics` output format.
- `scrubSecrets` for each pattern.

## Known limitations

Operational risks I found:

1. **One provider outage takes the whole product out of rotation.** Readiness and `/api/health` return `503` whenever **any** circuit is open. A load balancer or Kubernetes then stops sending traffic to every instance, so the dashboard, ingest and the other providers' gateways all go down because, say, Anthropic is unreachable.
2. **Provider error responses never open the circuit.** Only thrown errors (network failures, timeouts) count. A provider returning `500`, `502`, `503`, `529 overloaded` or `429` counts as a **success** and resets the failure count, so the breaker does nothing during the most common kind of outage.
3. **Circuits are shared by all workspaces, and kept per process.** One workspace with a broken network path to Azure can open the `azure` circuit for everyone. With `npm run cluster`, each worker has its own breakers, so the state, readiness and `/metrics` depend on which worker answers. In the half-open state, every request that arrives is let through as a probe, not just one.
4. **Metrics are per process and very coarse.** There's no breakdown by route, status code, workspace or provider, and no gateway upstream latency, database pool, ingest or queue metrics. With cluster mode, a scrape reaches one random worker, and counters restart from 0 when a worker restarts.
5. **Prometheus needs the platform admin secret.** `/metrics` is protected by `X-Admin-Secret`, the same secret that can create workspaces and keys, so the monitoring system has to hold the most powerful credential.
6. **The request id isn't in the logs.** `X-Request-ID` is set on the response but not written to the request or error log lines, so a customer's request id can't be used to find the log entry.
7. **`/api/health` is public and reveals details.** It returns the database driver's error message (which can name internal hosts), the storage backend, the process id and memory, even in production. Readiness hides these, but this route doesn't.
8. **Some startup checks ask for unrelated settings.** `DASHBOARD_USERNAME` and `DASHBOARD_PASSWORD` are required in production even though they only protect the older `/dashboard` page. A badly formatted `ENCRYPTION_KEY` only gets a warning. Missing webhook secrets, which mean unsigned ingest, are only warnings. SSO settings aren't checked at all.
9. **`platformStatus` reports server settings, not the workspace.** It says Claude, OpenAI or Copilot are "configured" when the server has `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` or `GITHUB_TOKEN` set, whatever the workspace's own connectors are.
10. **Shutdown doesn't stop background work cleanly.** Hourly and daily jobs (baselines, detector rebuild, retention) and database pools aren't stopped. A job running at the time is cut off when the process exits.
11. **No last-resort error handlers.** There's no `uncaughtException` or `unhandledRejection` handler, so an unexpected rejection ends the process. Cluster mode restarts it, but a single process (`npm start`) stays down until the platform restarts it.
