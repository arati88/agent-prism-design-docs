# Storage backends and migrations: where Agent Prism keeps its data

## Introduction

Everything Agent Prism remembers is saved through one storage layer: workspaces, users, keys, connectors, runs, sessions, audit logs, Copilot data, ML models and so on. That layer has two interchangeable backends:

- **File** (`STORAGE_BACKEND=file`, the default): everything in one JSON file, `data/app-state.json`. Nothing to install, so it's good for local development and demos. **Not for production.**
- **Postgres** (`STORAGE_BACKEND=postgres` plus `DATABASE_URL`): real tables, indexes, transactions, connection pooling, TLS, and row-level security on the main tables. This is what production uses.

The rest of the code never talks to either backend directly. It calls `src/saas-store.js`, which passes each call on to whichever backend is configured. So a route like `/api/ingest` calls `upsertTenantRuns(...)` and doesn't know or care where the run ends up.

The Postgres tables are created and changed by **numbered migration files** in `db/migrations/`, applied in order and recorded so each runs only once.

Example: a developer runs `npm start` with no settings and gets a working app with data in `data/app-state.json`. The hosted service sets `STORAGE_BACKEND=postgres`, `DATABASE_URL=postgres://…` and `RUN_MIGRATIONS_ON_STARTUP=true`. On boot it applies any new migrations, then serves traffic, and `/api/health/ready` only says "ready" once the database is prepared.

This document describes how storage works **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/saas-store.js` | The one entry point for all storage. Loads the chosen backend once, the first time it's needed. For the file backend, makes every call run one at a time |
| `src/stores/file-store.js` | File backend: reads and writes `data/app-state.json` |
| `src/stores/postgres-store.js` | Postgres backend: pool, `withTenant`, every query, `ensureSchemaPatches` |
| `db/migrations/*.sql` | 29 numbered schema changes, `000_base_schema.sql` to `027_api_key_role.sql` (two files share 016) |
| `db/migrate.js` | Applies pending migrations (`npm run migrate`, or at start-up) |
| `db/schema.sql` | An early snapshot of the schema (see limitations) |
| `src/db-ssl.js` | TLS settings for every Postgres connection |
| `src/config.js` | Backend choice and pool settings |
| `src/startup.js` | Start-up checks on storage settings |
| `src/cluster.js` | Refuses several workers on the file backend |

## What it does

- **Lets the same code run on either backend**, chosen by one setting.
- **Keeps the file backend safe within one process**: atomic writes, a backup copy, and one write at a time.
- **Keeps Postgres safe under load**: limited connection pool, query time limit, fail-fast when the pool is full, survives dropped connections, TLS.
- **Upgrades the database schema by itself** at start-up, even with several servers starting at once.
- **Adds a second layer of workspace isolation** in Postgres (row-level security) on the main tables.
- **Deletes old data** per workspace according to its plan (data retention).

## Where the data comes from

### Choosing a backend

| Setting | Default | What it does |
| --- | --- | --- |
| `STORAGE_BACKEND` | `file` | `file` or `postgres` |
| `DATABASE_URL` | — | Postgres connection string. Required for `postgres`; start-up stops without it |

At start-up (`src/startup.js`): Postgres without `DATABASE_URL` stops the server, and the file backend in production logs a warning ("Data will be lost on restart").

### What's stored

The file backend keeps one JSON object with a list per kind of record: `tenants`, `users`, `apiKeys`, `connectors`, `runs`, `sessions`, `dashboardSessions`, `auditLogs`, `promptCaptures`, `certifications`, `idempotencyKeys`, `failedIngests`, `mlModelWeights` and others.

Postgres has 32 tables. 31 come from the migrations; `ml_model_weights` is only created by the start-up patches (`ensureSchemaPatches`), so a database set up with `npm run migrate` alone doesn't have it until the server starts:

| Area | Tables |
| --- | --- |
| Workspaces and access | `tenants`, `users`, `api_keys`, `dashboard_sessions`, `refresh_tokens`, `tenant_plan_config`, `team_assignments` |
| Runs and ingest | `agent_runs`, `idempotency_keys`, `failed_ingests`, `prompt_captures`, `prompt_analyses` |
| Sessions and usage | `agent_sessions`, `fleet_snapshots`, `daily_usage_rollup`, `daily_usage_by_model` |
| Connectors | `connectors` |
| Certification | `agent_definitions`, `agent_tools`, `agent_certifications`, `certification_checks`, `agent_promotions`, `hitl_approvals` |
| Copilot | `copilot_seats`, `copilot_usage`, `copilot_model_usage`, `copilot_budget_status` |
| Reports and ML | `agent_baselines`, `spend_blocks`, `savings_settings`, `ml_model_weights` |
| Compliance | `audit_logs` (with a hash chain, so tampering shows) |

Every table except `tenants` has a `tenant_id` pointing at its workspace. Deleting a workspace deletes its rows (`ON DELETE CASCADE`).

## How it works

### The storage entry point: `src/saas-store.js`

- On the first storage call, it loads `postgres-store.js` or `file-store.js`, based on `STORAGE_BACKEND`, and keeps it for the life of the process.
- Each exported function (about 120 of them, such as `upsertTenantRuns`, `listTenantContext`, `getMonthlySpend`) just calls the same function on the backend.
- Some functions exist only on one backend. For those, the entry point returns a safe default when the backend doesn't have it, for example `[]` for a list.

### File backend

```text
every call:  read the whole file  →  change it in memory  →  write the whole file
```

- **One call at a time.** Two read-change-write calls overlapping would make the second one wipe out the first. Before this rule, 400 simultaneous ingests all answered 201 but only 7 runs were stored. `serializeFileStore` puts every file-store call in a single queue. This only works **inside one process**.
- **Atomic write.** The new state goes to a temporary file, which then replaces the real one in one step (`rename`). A crash mid-write can't leave a half-written file.
- **Backup.** After every write, the same content is also saved to `app-state.json.bak`. If the main file can't be read as JSON (manual edit, disk problem), the backup is used. If both are broken, every call fails with a clear "Manual recovery required" message.
- **Size caps** on some lists, such as audit logs, prompt captures and the failed-ingest queue (2,000 items), so the file can't grow forever.
- **Several workers are refused.** `npm run cluster` with the file backend and more than one worker stops with an error, because separate processes would overwrite each other's writes.

### Postgres backend

**Connection pool** (`getPool`, one per process):

| Setting | Default | Meaning |
| --- | --- | --- |
| `DB_POOL_MAX` | 80 ÷ number of workers, between 2 and 20 | Most connections per process, so the total stays under the usual 100 limit of managed Postgres |
| `DB_POOL_MIN` | 2 | Connections kept open and ready |
| `DB_IDLE_TIMEOUT_MS` | 30000 | Close an unused connection after this |
| `DB_CONNECTION_TIMEOUT_MS` | 5000 | Give up waiting for a free connection after this, instead of waiting forever |
| `DB_STATEMENT_TIMEOUT_MS` | 30000 | Stop any query running longer than this |

Start-up warns if `DB_POOL_MAX × CLUSTER_WORKERS` is over 80.

**Dropped connections**: error listeners on the pool and on each connection log the problem instead of crashing the server. The query that was running fails and its request gets an error, but the server stays up.

**TLS** (`src/db-ssl.js`), the same for every pool:

| Setting | Effect |
| --- | --- |
| `DB_SSL=false` | No TLS (local database or private network) |
| *(nothing set)* | TLS on, but the server certificate isn't verified |
| `DB_SSL_REJECT_UNAUTHORIZED=true` | TLS on, certificate verified against public CAs |
| `DB_SSL_CA=/path/ca.pem` | TLS on, certificate verified against your own CA (on-prem, or a cloud provider's bundle) |

**Workspace context (`withTenant`)**: most workspace queries run inside a transaction that first sets `app.current_tenant_id` to the caller's workspace, local to that transaction, so a reused connection never carries it over. Postgres row-level security uses this value (next section).

### Row-level security (Postgres)

On top of every query filtering by `tenant_id`, Postgres itself can refuse to return another workspace's rows:

| Tables | Rule |
| --- | --- |
| `agent_runs`, `connectors`, `audit_logs`, `prompt_captures`, `agent_definitions`, `agent_tools`, certification tables, Copilot tables, `agent_sessions`, `prompt_analyses` | Reading and updating only see rows of `app.current_tenant_id`. Inserting and deleting are allowed (the app sets the right workspace; retention deletes across workspaces) |
| `users`, `api_keys`, `dashboard_sessions` | Same, except that with **no** workspace set, all rows are visible, because login has to look up a key or email before the workspace is known. These three are **forced**, so they apply even to the table owner |

Important: on the other tables RLS is only *enabled*, not *forced*. Postgres doesn't apply enabled-only RLS to the **table owner** or a superuser. If the app connects as the owner, which is usual on hosted Postgres, those rules protect other connections (BI tools, consoles) but **not the app itself**, unless the app's database role is set to `NOBYPASSRLS` (migration `004` explains this).

### Migrations

`db/migrate.js`:

1. Creates `schema_migrations` (file name + time applied) if it doesn't exist.
2. Lists `db/migrations/*.sql` in alphabetical order, so the number prefix sets the order.
3. Skips files already recorded, and applies each new one **inside a transaction**. If one fails, it's rolled back and the run stops, so later migrations don't run on top of a broken one.

Two ways to run it:

- **By hand**: `npm run migrate`, which needs `DATABASE_URL`.
- **At start-up**: `RUN_MIGRATIONS_ON_STARTUP=true`, which `docker-compose.yml` and `.env.example` set.

### Start-up preparation (Postgres)

`prepareDatabase()` in `server.js`:

1. Takes a Postgres **advisory lock**, so if several servers or workers start at once, they take turns. The second finds nothing left to do. Without this, simultaneous `CREATE TABLE`s failed with duplicate-object errors.
2. Applies pending migrations, if `RUN_MIGRATIONS_ON_STARTUP=true`.
3. Runs `ensureSchemaPatches()`: a list of `ADD COLUMN IF NOT EXISTS` / `CREATE TABLE IF NOT EXISTS` statements that bring older databases up to date even if some migrations were never applied. It's safe to run on an up-to-date database.
4. Releases the lock.

While this runs, `GET /api/health/ready` answers **503 not ready** (`databasePrepared: false`), so a load balancer doesn't send traffic yet. If preparation fails in production, the server **exits**, so the platform shows the failure and restarts it, rather than serving on a half-upgraded schema. Jobs that touch the database (retention, legacy-key clean-up) wait for preparation to finish.

### Data retention

Once at start-up and then daily, for each workspace:

```text
days kept = the smaller of:  the plan's retention   (Free 30, Starter 90, Pro 365, Enterprise unlimited)
                             DATA_RETENTION_DAYS    (installation-wide maximum, if set)
```

- Plan retention only applies with a **valid license installed**. Without one, or in development, only `DATA_RETENTION_DAYS` deletes anything, so a licensing problem never deletes data.
- Older runs, prompt captures and finished sessions are deleted (`applyDataRetention`). Sessions still running or idle are kept.

## API

Storage has no API of its own. Related endpoints:

| Endpoint | What it says about storage |
| --- | --- |
| `GET /api/health/ready` | `ready` only when the database answers, preparation is done, no provider breaker is open and the server isn't shutting down. `503` otherwise |
| `GET /api/health` | `ok` only when the database answers and no provider breaker is open; includes `storageBackend` and database latency |
| `GET /api/admin/diagnose` (operator) | The database's table list and the backend in use |

### Settings

| Setting | Default | What it does |
| --- | --- | --- |
| `STORAGE_BACKEND` | `file` | Which backend |
| `DATABASE_URL` | — | Postgres connection |
| `RUN_MIGRATIONS_ON_STARTUP` | off | Apply migrations at boot |
| `DB_POOL_MAX`, `DB_POOL_MIN`, `DB_IDLE_TIMEOUT_MS`, `DB_CONNECTION_TIMEOUT_MS`, `DB_STATEMENT_TIMEOUT_MS` | see above | Pool tuning |
| `DB_SSL`, `DB_SSL_REJECT_UNAUTHORIZED`, `DB_SSL_CA` | TLS on, unverified | TLS |
| `DATA_RETENTION_DAYS` | 0 (no extra limit) | Installation-wide retention maximum |
| `CLUSTER_WORKERS` | number of CPUs | Workers in `npm run cluster`; must be 1 with the file backend |

## Dashboard / UI

There's no storage screen. Retention appears as the plan's "data retention" limit (from `GET /api/tenant`). Readiness is used by load balancers and container platforms, not people.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does for storage |
| --- | --- |
| `src/saas-store.js` | The storage entry point; `serializeFileStore` |
| `src/stores/file-store.js` | `readState`, `writeState` (atomic + backup), every store function for the file backend |
| `src/stores/postgres-store.js` | `getPool`, `pingDb`, `withTenant`, `ensureSchemaPatches`, every store function for Postgres |
| `db/migrate.js` | `runMigrations` and the `npm run migrate` command |
| `db/migrations/000_…` – `027_…` | The schema changes |
| `db/schema.sql` | Early schema snapshot |
| `src/db-ssl.js` | `dbSslConfig` |
| `src/config.js` | `storageBackend`, `databaseUrl`, `db.*` pool settings |
| `src/startup.js` | Storage checks at start-up |
| `src/cluster.js` | File-backend worker guard |
| `server.js` | `prepareDatabase` (advisory lock, migrations, patches), `/api/health/ready`, data retention |
| `docker-compose.yml` | Postgres + app with migrations at start-up |

## Tests

What's tested today:

- `test/file-store.test.js`: a valid file reads; every write also writes the backup; a corrupted file recovers from the backup; a corrupted file with no backup gives a clear error.
- `test/db-ssl.test.js`: TLS off, unverified default, public-CA verification, own-CA verification.
- `test/postgres.integration.test.js` (runs only with a test database): three servers starting together on a fresh database all become ready; a run id reused by another workspace can't take over the original; `NaN` and wrong types never reach the database; sessions are scoped per workspace; dashboard totals count every run; the server survives its connections being killed mid-traffic; users, team report, seat counts and per-workspace retention work on Postgres.

Not tested yet, and worth adding:

- Many simultaneous writes to the file backend all being kept (the 400-ingest case).
- A failing migration rolls back and stops the rest.
- Row-level security actually hiding another workspace's rows when the app role has `NOBYPASSRLS`.
- Both backends giving the same answer for the same calls (they're two separate implementations).

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **Two implementations of everything.** Every store function is written twice, once per backend: about 2,100 lines for file and 3,300 for Postgres. They can drift apart, and some functions exist on only one, where the entry point quietly returns an empty default. For example, ML model loading at start-up only happens on Postgres.
2. **`db/schema.sql` is out of date.** It defines 9 tables, against 32 in a current database, yet `CLAUDE.md` and `DEVELOPER.md` point to it as the schema, and `DEVELOPER.md` suggests running it by hand. Running it alone would give a database missing most tables. The migrations are the real source.
3. **Row-level security may not protect the app itself.** On most tables it's enabled but not forced, so it doesn't apply if the app connects as the table owner, which is the usual setup. Nothing checks or warns about this at start-up. Tables added later (`fleet_snapshots`, `daily_usage_*`, `failed_ingests`, `idempotency_keys`, `spend_blocks`, `team_assignments`, `tenant_plan_config` and others) have no row-level security at all.
4. **Migrations only go forward.** There are no "down" migrations, so undoing a change means writing a new migration or restoring a backup.
5. **Two migrations share the number 016** (`016_api_key_read_only.sql`, `016_plan_config_target_multiple.sql`). It works, because order is alphabetical, but numbers are no longer unique.
6. **Schema changes live in two places.** Some changes are both a migration and an entry in `ensureSchemaPatches`, so a new column has to be added to both or older databases miss it.
7. **The file backend reads and writes the whole file on every call**, including every API key check. It gets slower as data grows, and it only works with one process. It's for development and demos only.
8. **TLS isn't verified by default.** Without `DB_SSL_REJECT_UNAUTHORIZED=true` or `DB_SSL_CA`, the connection is encrypted but the database server's identity isn't checked.
9. **Retention covers runs, prompt captures and sessions only.** Audit logs, fleet snapshots, daily usage and Copilot data aren't deleted by plan retention.
