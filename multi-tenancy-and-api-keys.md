# Multi-tenancy, bootstrap and API keys: keeping each customer's data separate

## Introduction

Agent Prism can host several customers on one installation. Each customer gets its own **workspace**, called a *tenant* in the code. A workspace has its own users, API keys, connectors, runs, sessions, budgets and audit log. Nobody in one workspace can see or change another's data.

This document covers how that works:

- **Bootstrap**: how the first workspace is created on a new installation.
- **Provisioning**: how the operator adds more workspaces.
- **API keys** (`acp_…`): how agents, scripts and integrations prove which workspace they belong to.
- **`requireTenant`**: the single check every workspace route goes through, which decides *who is calling, from which workspace, with which role*.

There are two kinds of "admin", and they're different things:

| | Who | Credential | Can do |
| --- | --- | --- | --- |
| **Platform operator** | Whoever runs the installation (us, or the customer's IT for a self-hosted install) | The `X-Admin-Secret` header (`ACP_ADMIN_SECRET`) | Bootstrap, create workspaces, set plans, issue keys for any workspace, set any user's password or role |
| **Workspace admin / owner** | A person inside one workspace | Their dashboard login, or an API key | Manage that workspace only: users, keys, connectors, settings |

Example: on a new installation, the operator calls `POST /api/bootstrap` with the company name and admin email. The answer includes the first workspace, its owner, and an API key `acp_3f9a…`. A CI agent then sends runs with `x-api-key: acp_3f9a…`, and they land in that workspace only.

This document describes how it works **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | What it does |
| --- | --- |
| `POST /api/bootstrap` | Creates the **first** workspace, once per installation |
| `POST /api/admin/tenants` | Creates **more** workspaces (repeatable) |
| `GET /api/admin/tenants` | Lists all workspaces and the installation's license |
| `POST /api/admin/tenant/plan` | Changes a workspace's plan |
| `POST /api/admin/api-keys` | Operator issues a key for a workspace |
| `GET / POST / DELETE /api/tenant/api-keys…` | Workspace admins list, create, revoke and delete their own keys, and set a key's IP allowlist |
| `GET /api/tenant` | The calling workspace: users, connectors, plan, limits and usage |
| `requireTenant` (`server.js`) | Works out the workspace and role of every request |
| `requireAdmin` (`server.js`) | Checks the operator's admin secret |
| `src/auth.js` | Creates and checks API keys, password hashes and session tokens |
| `src/permissions.js` | The role table: viewer, member, admin, owner |

## What it does

- **Creates a workspace in one step**: the workspace, its first user (role **owner**), a default API key, a GitHub Copilot connector placeholder, and the workspace's own data encryption key.
- **Identifies the caller** on every request, from an API key, a short-lived token made from a key, or a dashboard login.
- **Scopes every read and write to the caller's workspace.** Every storage function takes the workspace id, and every query filters by it.
- **Applies the caller's role**: read-only keys can only read; a full key acts with the role of the person who created it.
- **Applies per-key limits**: IP allowlist and request rate.

## Where the data comes from

### What's stored

| Record | Main fields | Notes |
| --- | --- | --- |
| Workspace (`tenants`) | `id` (`tenant_…`), `name`, `slug` (from the name), `plan`, `status`, `keyManagement` (`platform` or customer-managed), wrapped encryption key | `slug` must be unique |
| User (`users`) | `tenantId`, `email`, `name`, `role`, password hash (PBKDF2-SHA256, 210,000 rounds) | First user is always `owner` |
| API key (`api_keys`) | `tenantId`, `name`, `prefix` (first 12 characters), `key_hash` (SHA-256), `status` (`active`/`revoked`), `readOnly`, `role`, `ip_allowlist`, `lastUsedAt` | **The full key is never stored**, only its hash |

Every other table (connectors, runs, sessions, audit logs, budgets and so on) has a `tenant_id` column pointing at its workspace. In Postgres, deleting a workspace deletes its rows too (`ON DELETE CASCADE`).

### What an API key looks like

```text
acp_ + 48 random hex characters (24 random bytes)
e.g. acp_3f9a1c07b2e4…
```

- Shown **once**, when created. After that only the prefix (`acp_3f9a1c07`) is visible.
- Stored as a SHA-256 hash and checked with a timing-safe comparison.
- Log lines mask anything that looks like a key (`[REDACTED_API_KEY]`, `src/middleware/logger.js`).

## How it works

### Bootstrap: the first workspace

`POST /api/bootstrap` (operator only):

1. Checks the admin secret, and a rate limit per IP address.
2. Checks the body (`companyName`, `adminEmail`, optional `adminName`, `adminPassword`, `plan`).
3. Refuses with `409 already_bootstrapped` if **any** workspace exists already.
4. Creates the workspace (default plan **free**), owner, default key ("Default ingest key"), Copilot connector placeholder and encryption key.
5. Writes an audit entry "Tenant Bootstrapped".
6. If a password was given, also signs the owner in to the dashboard (sets the session cookie).
7. Returns `201` with the workspace, user, connector and **the API key in plain text**.

`GET /api/bootstrap/status` (public) says whether the installation has been set up and how many workspaces it has. The dashboard uses it to show the first-run screen.

### Provisioning more workspaces

`POST /api/admin/tenants` (operator only) does the same as bootstrap but can be called again and again:

- **Default plan**: `enterprise-trial`, or the license's tier if the license doesn't cover a trial.
- **License ceiling**: a plan above what the installation's license covers is refused (`403 plan_exceeds_license`). The same check applies to `POST /api/admin/tenant/plan`.
- A duplicate name or admin email is refused with `409 conflict` (Postgres).
- The Copilot connector placeholder is created too, as in bootstrap, but it isn't included in the response.

On a **single-tenant license** (a customer's own installation) every workspace gets the license's tier, whatever plan is stored. On a **multi-tenant license** (our hosted service) each workspace gets its own plan, capped at the license tier (`effectivePlanName` in `src/license.js`).

### Identifying the caller: `requireTenant`

Every workspace route starts with `requireTenant`. It tries, in this order:

1. **A token made from an API key** (a JWT from `POST /api/oauth/token`). It must be signed with the server's signing key (`JWT_SECRET`, HS256) and be an access token. The key it came from must still be **active**, so revoking a key also kills its tokens. The token inherits that key's read-only flag, role and IP allowlist.
2. **An API key**, from the `x-api-key` header or `Authorization: Bearer …`. It's looked up by prefix (Postgres) and checked against the stored hash. `lastUsedAt` is updated.
3. **A dashboard login**, from the session cookie or the `X-Session-Token` header. A write (POST, PUT, DELETE) authenticated only by the cookie must come from Agent Prism's own web address; otherwise it's refused (`403 csrf_rejected`).

If none works: `401 unauthorized`.

Then, for the caller it found:

| Check | Rule | Refused with |
| --- | --- | --- |
| Read-only | A `viewer` user or a read-only key may only GET or HEAD | `403 read_only` |
| IP allowlist | If the key has an allowlist, the caller's IP must match one entry (exact address or IPv4 range). No allowlist means everyone is allowed. Not applied to dashboard logins | `403 ip_not_allowed` |
| Rate limit | API traffic: the plan's requests per minute, per workspace. People on the dashboard: a separate, generous per-person limit (one page load makes about 25 requests) | `429` |

The result (`auth`) holds the workspace, the user (for dashboard logins) and the key. Each route then calls `requirePermission(auth, …)` where it needs a specific role.

### Roles of API keys

From `roleOf` in `src/permissions.js`:

| Key | Acts as |
| --- | --- |
| Read-only key | `viewer`, read only |
| Full key created by a person | **That person's role**, so an admin's key is an admin key, never an owner key |
| Key with no recorded role: bootstrap and provisioning default keys, operator-issued keys, keys from before key roles existed | **`owner`** |

Managing keys needs the `keys.manage` permission (admin or owner).

### Managing keys inside a workspace

| Route | What it does |
| --- | --- |
| `GET /api/tenant/api-keys` | Lists keys: name, prefix, status, read-only, role, dates. No hashes |
| `POST /api/tenant/api-keys` | Creates a key. `name` (≤ 128 characters) and `readOnly` (true/false) are optional. Returns the key in plain text, once |
| `DELETE /api/tenant/api-keys/{id}` | **Revokes** a key: it stops working but stays in the list |
| `DELETE /api/tenant/api-keys/{id}/permanent` | **Deletes** a key completely |
| `DELETE /api/tenant/api-keys` | Deletes **every** key except the one making the request |
| `PUT /api/tenant/api-keys/{id}/ip-allowlist` | Sets or clears a key's IP allowlist |

You can't revoke or delete the key you're using for the request (`400`). Every change writes an audit entry: "Tenant API Key Created / Revoked / Deleted", "All API Keys Deleted".

### The operator's admin secret

`requireAdmin` compares `X-Admin-Secret` with `ACP_ADMIN_SECRET` using a timing-safe comparison. Too many wrong guesses from one IP address within a minute lock that address out for a while (`429` with `Retry-After`). At start-up, `validateConfig()` refuses to run in production with the default secret.

## API

Bootstrap request and response:

```json
POST /api/bootstrap          X-Admin-Secret: …
{ "companyName": "Acme Corp", "adminEmail": "ops@acme.com", "adminName": "Ops", "adminPassword": "…" }

201
{ "tenant": { "id": "tenant_9c1e…", "name": "Acme Corp", "slug": "acme-corp", "plan": "free", "status": "active" },
  "user": { "id": "user_…", "email": "ops@acme.com", "role": "owner" },
  "connector": { "provider": "github-copilot", "status": "ready" },
  "apiKey": "acp_3f9a1c07b2e4…" }
```

Creating a key inside a workspace:

```json
POST /api/tenant/api-keys    x-api-key: acp_…   (admin or owner)
{ "name": "CI pipeline", "readOnly": false }

201
{ "apiKey": "acp_…",
  "key": { "id": "key_…", "name": "CI pipeline", "prefix": "acp_51d0e8a2", "status": "active",
           "readOnly": false, "role": "admin", "createdAt": "…", "lastUsedAt": null } }
```

Common errors:

| Status | Error | When |
| --- | --- | --- |
| `401` | `unauthorized` | No key, wrong key, revoked key or token, wrong admin secret |
| `403` | `read_only` | Write with a read-only key or viewer login |
| `403` | `ip_not_allowed` | Caller's IP not in the key's allowlist |
| `403` | `csrf_rejected` | Cookie-only write from another website |
| `403` | `plan_exceeds_license` | Plan above the license |
| `403` | permission error | Role too low for the action (`requirePermission`) |
| `409` | `already_bootstrapped` / `conflict` | Bootstrap run twice / duplicate workspace |
| `429` | `rate_limit_exceeded` | Request limit, or too many wrong admin-secret guesses |

### Settings

| Setting | What it does |
| --- | --- |
| `ACP_ADMIN_SECRET` | The operator's secret for `X-Admin-Secret`. Must not be the default in production |
| `JWT_SECRET` | Signs the tokens made from API keys. If not set, it's derived from `ACP_ADMIN_SECRET`; in production it must be set and different from it |
| `ENCRYPTION_KEY` | Master key that wraps each workspace's data encryption key |

## Dashboard / UI

- **First-run screen**: shown while `GET /api/bootstrap/status` says the installation isn't set up.
- **Admin tab → API keys**: lists the workspace's keys, creates keys (full or read-only), revokes or deletes them. The full key is shown once, right after it's created. Only visible to admins and owners. IP allowlists have no screen; they're set through `PUT /api/tenant/api-keys/{id}/ip-allowlist` only.
- **Workspace header**: the workspace name and plan, from `GET /api/tenant`.

The operator routes (`/api/admin/*`) have no screen in the product. The vendor-only owner console (`npm run owner-console`) uses some of them.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does for multi-tenancy |
| --- | --- |
| `server.js` | `requireTenant`, `requireAdmin`, `getApiKey`, `requirePermission`; the bootstrap, admin and key routes; `GET /api/tenant` |
| `src/auth.js` | `createApiKey`, `verifyApiKey`, `hashPassword`, `verifyPassword`, `createSessionToken`, `createId` |
| `src/permissions.js` | Roles, the permission table, `roleOf` |
| `src/plans.js`, `src/license.js` | Plans per workspace, license ceiling, `effectivePlanName`, `planAllowedByLicense` |
| `src/security/ip-allowlist.js` | `ipAllowed`, `isValidAllowlistEntry` |
| `src/middleware/rate-limiter.js` | Per-workspace, per-person, bootstrap and admin-failure limits |
| `src/middleware/timing-safe-compare.js` | Comparing the admin secret |
| `src/middleware/logger.js` | Masking keys in logs |
| `src/crypto.js`, `src/tenant-keys.js` | The workspace's own data encryption key (separate design doc) |
| `src/saas-store.js` + `src/stores/*` | `bootstrapSaas`, `provisionTenant`, `createTenantApiKey`, `listTenantApiKeys`, `revokeTenantApiKey`, `deleteTenantApiKey`, `deleteAllTenantApiKeys`, `authenticateTenantApiKey`, `getApiKeyStatus`, `getApiKeyAccess`, `listTenants`, `updateTenantPlan` |
| `db/schema.sql`, `db/migrations/000_base_schema.sql` | `tenants`, `users`, `api_keys`, and `tenant_id` on every other table |

## Tests

What's tested today:

- `test/tenant-admin.test.js`: keys can be listed and revoked, and hashes are never returned.
- `test/user-management.test.js`: key roles (`roleOf`: a full key acts as its creator's role, old keys act as owner); an admin's key is an admin, directly or as a token; the role table; read-only viewers; IP allowlist entries; the dashboard isn't throttled by the plan's API limit.
- `test/ip-allowlist.test.js`: exact addresses, IPv4 ranges, empty allowlist allows everyone, valid entries.
- `test/admin-lockout.test.js`: wrong admin-secret guesses are locked out.
- `test/license.test.js`: plans above the license are refused.
- `test/auth-refresh.test.js`: token refresh.
- BDD: `features/06_cost_leak_tenant_isolation.feature`: one workspace's cost leaks never show in another.

Not tested yet, and worth adding:

- Bootstrap twice → `409`, and the first answer contains a working key.
- A revoked key and a token made from it are both refused.
- A broad isolation test: workspace A's key can't read or change any of workspace B's runs, connectors, sessions or keys, on every route.
- `DELETE /api/tenant/api-keys` keeps the calling key.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **A workspace can't be suspended or deleted.** Workspaces have a `status`, but there's no route to change it. Password login, dashboard sessions, invites and token refresh (Postgres) check it, but API keys and the access tokens made from them don't. Even if a workspace were marked inactive in the database, its keys would keep working.
2. **Default keys act as owner.** The key returned by bootstrap and provisioning, and any key the operator issues, has no recorded role, so it has full owner power, including erasing data. They're meant for first setup, but nothing prompts anyone to replace them with narrower keys.
3. **`POST /api/admin/api-keys` without a `tenantId` picks the first workspace.** On a multi-workspace installation, a forgotten `tenantId` silently gives a key for the wrong customer.
4. **Keys never expire.** There's no expiry date and no rotation reminder. `lastUsedAt` is recorded but nothing warns about old or unused keys.
5. **No limits on scopes.** Every key has the scope `*`. Beyond read-only and role, a key can't be limited to, say, ingest only.
6. **One admin secret for the whole installation.** The operator credential is a single shared secret, with no per-person operator accounts and no audit of *which* operator acted (entries say "Admin").
7. **The file store checks every key on every request.** It hashes the incoming key and compares it with all stored keys, then rewrites the whole state file to update `lastUsedAt`. That's fine for demos but slow with many keys or much traffic. Postgres looks keys up by prefix instead.
8. **Provisioning doesn't check the license's multi-tenant flag.** A single-tenant license can still have extra workspaces created; they all get the license tier. Whether that's allowed is a licensing decision that isn't enforced here.
9. **Workspace isolation mostly depends on every query including the workspace id.** Postgres row-level security exists on the main tables, but on most of them it doesn't apply when the app connects as the table owner, and newer tables have none (see [storage backends](storage-backends-and-migrations.md)). The file backend has no second layer at all. A future query that forgets `tenant_id` could leak data, and only tests would catch it.
10. **On Postgres, the bootstrap response shows the wrong plan.** The workspace is stored with the plan sent, or **free** by default, but the response always says `"plan": "enterprise-trial"`. The file backend returns the stored plan. Anyone reading the bootstrap response on Postgres is told they're on a trial plan that the workspace doesn't have.
