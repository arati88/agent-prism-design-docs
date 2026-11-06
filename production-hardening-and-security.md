# Production hardening and request security

## Introduction

Agent Prism started as a proof of concept, with no input checks, no rate limits, a path-traversal hole in the static file server, no security headers and a "View Sales Story" demo link. **Production hardening** added the defensive layers that every request goes through, using only Node's standard library:

| Layer | What it does |
| --- | --- |
| CORS | Only origins listed in `CORS_ALLOWED_ORIGINS` get CORS headers; `*` is refused at startup; only HTTPS origins in production |
| Body size | Requests over `MAX_BODY_BYTES` (1 MB) get `413` and the connection is closed |
| Content type | A body that isn't `application/json` gets `415` |
| Input validation | `src/validation.js` schemas, `422 validation_error` with the fields that failed |
| Path traversal | Static paths are decoded, resolved and must stay inside `public/`, otherwise `403` |
| Security headers | CSP, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, HSTS in production |
| Rate limits | Per workspace, per key, per signed-in user, sign-in, bootstrap and admin failures (fixed one-minute windows, in memory) |
| Structured logs | One JSON line per request, errors with secrets masked |
| Startup checks | Refuses unsafe production settings |
| Graceful shutdown | Drains in-flight requests on SIGTERM/SIGINT |
| Bootstrap lock | `POST /api/bootstrap` works once (`409` afterwards) and is limited to 5 attempts an hour per IP |
| Docker | Non-root `node:24-alpine`, `npm ci` from the lockfile, health check |

This document replaces the openspec change `openspec/changes/production-hardening-and-security/` in our format. It describes what's in the code **today** and where that differs from the plan. Startup, shutdown, logs and health checks are covered in more detail in [health, metrics and resilience](health-metrics-and-resilience.md).

Example: a client sends `POST /api/sessions` with `Content-Type: text/plain` and gets `415`. It resends as JSON without `platform` and gets `422 { fields: [{ field: "platform", message: … }] }`. A browser on `https://evil.example` sends a preflight and gets `403 CORS origin not allowed`. `GET /..%2f..%2fetc/passwd` gets `403`. Every one of these responses carries `X-Content-Type-Options: nosniff` and the CSP.

This describes the hardening that's in the code today.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/middleware/cors.js` | `applyCors`, `isAllowedOrigin` |
| `src/middleware/security-headers.js` | `setSecurityHeaders`, `generateCspNonce`, `buildCsp` |
| `src/middleware/rate-limiter.js` | `RateLimiter` (request counts), `TokenLimiter` (token or failure counts), the shared limiter instances |
| `src/middleware/logger.js`, `request-id.js`, `client-ip.js`, `timing-safe-compare.js` | Logs and secret masking, request id, client IP (`TRUST_PROXY`), constant-time compare |
| `src/validation.js` | `validate(schema, data)`, `SCHEMAS` |
| `src/startup.js`, `src/shutdown.js` | Startup checks, graceful shutdown |
| `server.js` | `readBodyBytes` (size limit), `parseBody` (content type, JSON object only, `\u0000` removed), `sendValidationError`, `serveStatic` (path containment), `requireBasicAuth`, the global error handler |
| `Dockerfile`, `.dockerignore`, `docker-compose.yml` | Container hardening |

## What it does

- Stops malformed, oversized or hostile input before it reaches business logic.
- Stops the browser running injected script, or loading anything from other origins.
- Limits how hard any one caller can push the server.
- Refuses to run in production with known-unsafe settings.

## Where the data comes from

| Setting | Default | Purpose |
| --- | --- | --- |
| `CORS_ALLOWED_ORIGINS` | empty (no cross-origin access) | Comma-separated origins; `*` stops startup |
| `MAX_BODY_BYTES` | 1,048,576 | Body size limit |
| `RATE_LIMIT_REQUESTS_PER_MINUTE` | 300 | Workspace default when the plan has no limit of its own |
| `RATE_LIMIT_KEY_RPM`, `RATE_LIMIT_KEY_TPM` | 60, 100,000 | Gateway per-key requests and tokens (the plan can set its own) |
| `RATE_LIMIT_DASHBOARD_USER_RPM` | 600 | Per signed-in user |
| `RATE_LIMIT_LOGIN_PER_MINUTE` | 20 | Sign-in, invite, OAuth token, per IP |
| `RATE_LIMIT_ADMIN_FAILURES_PER_MINUTE` | 10 | Wrong `X-Admin-Secret` attempts, per IP |
| `SHUTDOWN_TIMEOUT_MS` | 30,000 in production, 10,000 otherwise | Drain time |
| `DASHBOARD_USERNAME`, `DASHBOARD_PASSWORD` | — | Basic Auth for `/dashboard` (required in production) |
| `TRUST_PROXY` | off | Use `X-Forwarded-For` for the client IP |

## How it works

### 1. Order on every request

```text
in-flight count + request id → CORS (preflight answered here) → route match →
  parseBody: content type (415) → size (413) → JSON object, \u0000 removed (400) → validate (422) →
  requireTenant: credentials → read-only → IP allowlist → rate limit (429) →
  handler → security headers on the response → log line
```

### 2. CORS

- Without an `Origin` header, nothing is added.
- In production, a non-HTTPS origin gets no CORS headers, and its preflight gets `403`.
- An allowed origin gets `Access-Control-Allow-Origin: <origin>`, `Allow-Credentials: true` and `Vary: Origin`. A preflight gets methods `GET, POST, PATCH, DELETE, OPTIONS`, headers `Content-Type, Authorization, X-API-Key, X-Admin-Secret` and a max age of 1 day.
- Any other origin's preflight gets `403`.

### 3. Security headers

| Header | Value |
| --- | --- |
| `Content-Security-Policy` | `default-src 'self'; style-src 'self' 'unsafe-inline'; font-src 'self'; script-src 'self' [nonce]; img-src 'self' data:; connect-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'self'` |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `SAMEORIGIN` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains`, production only |

HTML pages get a fresh nonce added to their own inline `<script>` tags. Inline event handlers were replaced by `data-action` delegation, so `script-src` never needs `'unsafe-inline'`.

### 4. Rate limits

Each limiter counts in a fixed 60-second window per key: the first request starts the window, and requests over the limit get `429` with `Retry-After` set to the seconds left. Expired windows are cleaned up every 60 seconds.

| Who | Key | Limit |
| --- | --- | --- |
| API key or token | Workspace | Plan's requests per minute (Free 20, Starter 60, Pro 300, Enterprise none) |
| Signed-in user | User | 600 a minute |
| Gateway key | Key | Requests and tokens per minute (plan or settings) |
| Sign-in, invite, OAuth token | IP | 20 a minute |
| Bootstrap | IP | 5 an hour |
| Wrong admin secret | IP | 10 failures a minute, checked **before** the secret is compared |

### 5. Static files

`serveStatic` takes the pathname (query string removed), maps `/` to `index.html`, resolves it against `public/`, and refuses anything that resolves outside it. The comparison uses the OS path separator, so it also works on Windows.

### 6. What changed from the plan

| Plan (openspec) | Today |
| --- | --- |
| Middleware pipeline `(req, res, next)` | Helper functions called in order inside the one handler; no pipeline object |
| Sliding window, default 300/min for ingest, 60 for metrics | Fixed window per key; the workspace limit comes from the **plan**; separate per-user, per-key, sign-in and admin-failure limiters |
| CSP with Google Fonts, `unsafe-inline` to start | No third-party origins at all (self-hosted fonts); scripts never inline; nonce for page scripts |
| `Retry-After: 60` | Seconds left in the window |
| Startup checks: admin secret, dashboard credentials, `NODE_ENV` | Also `ENCRYPTION_KEY`, `JWT_SECRET`, CORS `*`, database TLS, pool size, webhook secrets (see [health and resilience](health-metrics-and-resilience.md)) |
| "View Sales Story" link removed | Removed from the main bar; `storyline.html` is still served if opened directly |
| Non-goal: OAuth / OIDC | Built since (see [auth](auth-sso-and-scim.md)) |
| Non-goal: audit to an external system | SIEM delivery built since (see [audit and SIEM](audit-gdpr-and-siem.md)) |

## API

These aren't routes of their own. Every route can return `413 payload_too_large`, `415 unsupported_media_type`, `422 validation_error` (`{ error, message, fields[] }`), `429 rate_limit_exceeded` (with `Retry-After`), or `403` (CORS preflight, path traversal).

## Dashboard / UI

None. The dashboard runs under the CSP above.

## Changes by file

Hardening code, by file:

| File | What it does |
| --- | --- |
| `src/middleware/*` | CORS, headers, rate limits, logs, request id, client IP, constant-time compare |
| `src/validation.js` | Schemas |
| `src/startup.js`, `src/shutdown.js` | Startup checks, shutdown |
| `server.js` | Body reading and parsing, validation responses, static containment, Basic Auth, error mapping |
| `Dockerfile`, `.dockerignore`, `docker-compose.yml` | Containers |
| `openspec/changes/production-hardening-and-security/*` | Original plan, replaced by this document |

## Tests

- `test/security.test.js`: validation (missing field, bad enum, too long, valid body, session update); startup exits with the default secret in production; security headers (nosniff, frame options, referrer policy, no HSTS outside production); CORS allowed and refused; rate limiter (under the limit, blocks at limit+1, separate keys, window resets); path traversal (old code vulnerable, new code contained, normal paths fine).
- `test/hardening.test.js`: malformed input classes against a live server (see [architecture and load testing](architecture-and-load-testing.md)).

Not tested yet, and worth adding:

- HTTP-level `413` and `415`.
- The CSP nonce on HTML pages.
- Bootstrap's 5-an-hour limit and the admin-failure limiter.

## Known limitations

Hardening gaps that are still open:

1. **Rate limits are kept in each process's memory.** With cluster mode or several replicas, each limit is multiplied by the number of processes, and counts start again on restart. That's noted in the code but not fixed.
2. **Validation covers only some routes.** `SCHEMAS` exists for sessions, usage, login, bootstrap, connectors, ingest, team assignments, plan config, promotions and a few others. Many routes check their input by hand or not at all, and nothing makes sure every route is covered.
3. **Storyline and other demo pages are still served.** `storyline.html`, `leadership.html` and `admin.html` are reachable by URL in production, without sign-in.
