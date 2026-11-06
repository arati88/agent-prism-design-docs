# Licensing and plans

## Introduction

Agent Prism ships as **one image for every edition**. What an installation may use is set by two things:

- **Plans** (`src/plans.js`) define the limits for each edition: Free, Starter, Pro, Enterprise and Enterprise Trial. They cover agents, runs per month, seats, gateway, AI Advisor, prompt capture, data retention, and API requests and tokens per minute.
- **A signed licence** (`src/license.js`) sets the **ceiling** for the installation. It's an Ed25519-signed text, `APL1.<payload>.<signature>`, that only the product owner can issue. It's checked offline against public keys built into the image, so it works on air-gapped sites and can't be forged or edited.

A licence is either:

- **Single-workspace** (a customer's own installation): every workspace gets the licence's tier and seats.
- **Multi-workspace** (our hosted service): the operator gives each workspace a plan, up to the licence's tier, with `POST /api/admin/tenant/plan`.

Running out of licence is meant to be **never destructive**: after expiry and 14 days' grace, workspaces drop to Free limits, but no data is deleted.

The operator and product-owner guide is [docs/LICENSING.md](../LICENSING.md), which covers issuing licences, the owner console and installing a licence. This document is the design view.

Example: Contoso gets a single-workspace **Pro** licence with **50 seats**, expiring 2027-10-31. Every workspace on Contoso's installation gets Pro: 100 agents, 50,000 runs a month, the gateway, the AI Advisor and 365-day retention. Seats are capped at 50 across the installation. From 2027-10-01 admins see a renewal banner. Between 2027-10-31 and 2027-11-14 everything still works. After that, Free limits apply: 5 agents, 500 runs and no gateway, until a new licence is installed.

This reflects how licensing and plans work in the current code.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/plans.js` | `PLANS`, `getPlan`, `getPlanForTenant` (the effective plan), `computeUsage`, `checkIngestAllowed`, `checkGatewayAllowed`, upgrade prompts |
| `src/license.js` | `verifyLicenseText`, `evaluateLicense` (licence state), `currentLicense` (from the environment, cached per minute), `effectivePlanName`, `planAllowedByLicense` |
| `src/license-keys.js` | Trusted public keys by key id (`ap-2026-1`) |
| `scripts/license/*`, `scripts/owner-console/*` | Product-owner tools: key generation, issuing, inspecting, and the console. **Not in the image** |
| `server.js` | `checkPlanAllows`, `gatewayPlanGuard`, `seatCheck`, rate limits in `requireTenant`, `retentionDaysFor`, `GET /api/license`, `POST /api/admin/tenant/plan`, `GET /api/admin/tenants`, licence line at startup |
| Storage | `tenants.plan` (the stored plan for each workspace) |

`/api/tenant/plan-config` is a different thing: the customer's **own AI-tool plan** (seats and price) used for plan-utilisation advice, not Agent Prism's edition.

## What it does

- Lets one image be sold as four editions, with limits set by a licence the customer can't change.
- Lets the hosted service sell a plan per workspace, up to the hosted licence.
- Warns admins before expiry, gives 14 days' grace, and never deletes data because of licensing.
- Enforces every limit on the server; the dashboard only shows them.

## Where the data comes from

**Plans:**

| | Free | Starter | Pro | Enterprise | Enterprise Trial |
| --- | --- | --- | --- | --- | --- |
| Agents | 5 | 20 | 100 | ∞ | ∞ |
| Runs per month | 500 | 5,000 | 50,000 | ∞ | ∞ |
| Seats (`teamMembers`) | 1 | 5 | 25 | ∞ | ∞ |
| Gateway | — | ✓ | ✓ | ✓ | ✓ |
| AI Advisor, Prompt Advisor | — | ✓ | ✓ | ✓ | ✓ |
| Prompt capture allowed | — | — | ✓ | ✓ | ✓ |
| Data retention (days) | 30 | 90 | 365 | ∞ | 90 |
| API requests per minute (per workspace) | 20 | 60 | 300 | ∞ | ∞ |
| Tokens per minute (per gateway key) | 10,000 | 100,000 | 1,000,000 | ∞ | ∞ |

An unknown stored plan is treated as Free.

**Licence**, from `AGENT_PRISM_LICENSE` (the text) or `AGENT_PRISM_LICENSE_FILE` (a path, re-read when the file changes). Payload: `{ kid, licenseId, customer, tier (free|starter|pro|enterprise), seats (a whole number, or null for the plan default), multiTenant, issuedAt, expiresAt }`.

## How it works

### 1. Licence state (`evaluateLicense`)

| Situation | Mode / state | Ceiling |
| --- | --- | --- |
| No licence, production | unlicensed / none | Free, single-workspace |
| No licence, `NODE_ENV` isn't `production` | development / none | Enterprise, multi-workspace, no limits ("not for production") |
| Bad format, unknown key id, bad signature, unknown tier, missing fields, bad seat count, or file can't be read | invalid | Free in production, no limits in development |
| More than 30 days to expiry | licensed / valid | Licence tier |
| 30 days or fewer to expiry | expiring | Licence tier, with a renewal banner |
| Up to 14 days after expiry | grace | Licence tier, with "stops on …" |
| More than 14 days after expiry | expired | Free (seats back to the plan default) |

`currentLicense()` re-checks the licence at most once a minute for the same source, so expiry follows the clock and a replaced file takes effect within a minute.

### 2. A workspace's effective plan (`effectivePlanName`)

```text
single-workspace licence → the licence tier (the stored plan is ignored)
multi-workspace licence  → the stored plan, lowered to the licence tier if it's higher
                           (rank: free 0, starter 1, pro 2, enterprise and enterprise-trial 3)
```

Every limit and feature check uses `getPlanForTenant`, never the stored plan directly.

### 3. Limits and where they're checked

| Limit | Checked | When it's hit |
| --- | --- | --- |
| Runs per month | Each run on `/api/ingest*` and the gateway: runs since the 1st of the month (UTC) | `402 run_limit_reached` |
| Agents | Each run with a **new** agent name: number of distinct agent names in all stored runs | `402 agent_limit_reached` |
| Gateway | Each `/v1/*` call | `402 gateway_not_available` |
| AI Advisor and Prompt Advisor | Each call | `402 feature_not_in_plan`, or `unavailable` |
| Requests per minute | Each request with an API key or token (dashboard users have their own limit instead) | `429` |
| Tokens per minute | Gateway calls, per key (see [gateway proxy](gateway-proxy.md)) | `429` |
| Seats | Before adding an active user: invite, reactivate, SCIM create or reactivate, first SSO sign-in | `402 seat_limit_reached` (SCIM `403`; SSO `no_seats`) |
| Retention | Daily | Old runs, captures and finished sessions deleted |

Each `402` includes an **upgrade prompt** with the next plan and `UPGRADE_URL` (default `https://agentprism.io/pricing`).

**Seats** (`seatCheck`). The workspace limit is the licence's seats on a single-workspace licence, otherwise the plan's `teamMembers`. If the licence sets seats, there's also an **installation-wide** limit. Seats are active users. Existing users are never cut off; only additions are refused.

**Retention** (`retentionDaysFor`). The plan's days apply **only** under a licence that's valid, expiring or in grace. With no licence, an invalid one, an expired one or in development, nothing is deleted unless `DATA_RETENTION_DAYS` is set. `DATA_RETENTION_DAYS` is always an extra upper limit.

### 4. Changing a hosted workspace's plan

`POST /api/admin/tenant/plan { tenantId, plan }` (platform admin). The plan must be one of `free`, `starter`, `pro`, `enterprise` or `enterprise-trial`, and **no higher than the licence**, otherwise `403 plan_exceeds_license`. Under a single-workspace licence the stored plan has no effect.

## API

| Method and path | Auth | Returns |
| --- | --- | --- |
| `GET /api/license` | Tenant (details: `settings.manage`) | `mode`, `state`, `message`, `plan`. Admins also get tier, customer, licence id, expiry, days left, multi-workspace, error, seats (used and limit, for the workspace and the installation) and features |
| `GET /api/tenant` | Tenant | `plan`: limits and usage (agents, agent names, runs this month), `upgradeUrl` |
| `POST /api/admin/tenant/plan` | Platform admin | `{ ok, tenant }` or `403 plan_exceeds_license` |
| `GET /api/admin/tenants` | Platform admin | Workspaces and the licence summary (no licence text) |

## Dashboard / UI

- **Admin → Plan and license:** plan, tier, seats in use against the limit, expiry, features.
- **A banner** for admins when the licence is expiring, in grace, expired, missing or invalid, or in development mode.
- **Upgrade prompts** when a `402` is returned.
- **Owner console** (`npm run owner-console`, on the product owner's machine only): issue and renew licences, a register of issued licences, hosted workspaces and their plans, and a licence checker.

## Changes by file

Licensing and plan code:

| File | What it does |
| --- | --- |
| `src/plans.js` | Plans and limit checks |
| `src/license.js`, `src/license-keys.js` | Licence verification and state |
| `server.js` | Enforcement points, seats, retention, licence and plan routes |
| `scripts/license/*`, `scripts/owner-console/*` | Product-owner tools (not shipped) |
| `src/stores/*` | `getPlanUsage`, `countActiveUsers`, `updateTenantPlan` |
| `public/app.js` | Plan and licence page, banners, upgrade prompts |

## Tests

What's tested today:

- `test/license.test.js`: a licence from the vendor tool verifies and any edit breaks it; a different signing key is refused even with a known key id; the lifecycle valid → expiring → grace → expired; no or invalid licence means Free in production and no limits (labelled) in development; the effective plan for own installations and hosted plans; production without a licence is Free and the operator can't raise the plan; a forged licence is refused and explained; licensing never deletes data.
- `test/user-management.test.js`, `test/sso.test.js`: seat checks on invite, reactivation and SSO.

Not tested yet, and worth adding:

- Each `402` limit at its edge: the 500th and 501st run, the 6th agent.
- Installation-wide seats across several workspaces.
- Retention days for each plan.

## Known limitations

Licensing gaps worth discussing:

1. **An expired licence stops customers' agents, not just features.** After grace, the gateway isn't in the Free plan, so every agent routed through `/v1/*` gets `402` and stops working. Ingest also stops at 500 runs a month. No data is deleted, but "never destructive" doesn't describe what customers' production agents experience.
2. **A licence can't be withdrawn and isn't tied to an installation.** It's valid until it expires, wherever it's installed. A leaked or shared licence works on any number of servers, and there's no revocation list. Expiry depends on the server's own clock.
3. **A missing `NODE_ENV=production` removes every limit.** Without it, an installation with no licence runs in "development" mode with Enterprise features and no limits. (The strict startup checks are also off.)
4. **A single-workspace licence can be stretched across many workspaces.** Agents, runs and rate limits are counted **per workspace**, and each workspace gets the full tier. An operator who creates more workspaces gets more allowance; only seats are counted across the installation.
