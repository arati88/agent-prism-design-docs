# Sign-in, SSO, SCIM and user management

## Introduction

This document covers how **people** get into Agent Prism and how their accounts are managed:

- **Password sign-in.** Email and password, giving a 7-day dashboard session.
- **Single sign-on (SSO).** OpenID Connect with the customer's identity provider (Okta, Entra ID, Keycloak and so on). Roles can come from the provider's groups.
- **Invites.** An owner or admin adds a person, who sets their own password through a one-time link. The same link is used for password resets.
- **SCIM 2.0.** The identity provider creates, updates, deactivates and deletes users automatically.
- **Demo sign-in.** A shared login, shown on the sign-in screen, for prospects.
- **OAuth token exchange.** For SDKs and machines: swap an API key for a 1-hour access token and a 30-day refresh token.

How a request is matched to a workspace, API keys, roles and permissions is in [multi-tenancy and API keys](multi-tenancy-and-api-keys.md).

Example: Acme connects Okta. A new engineer signs in through Okta for the first time. Okta says the address is verified, and the engineer is in the `ap-admins` group, which is mapped with `OIDC_ADMIN_GROUPS`. Agent Prism creates the account in Acme's workspace, checks there's a free seat, gives the account the **admin** role and starts a session. Months later Okta's SCIM sends `active: false`, and the engineer's sessions stop working on the next request.

This document describes sign-in and accounts **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/auth.js` | Password hashing (PBKDF2-SHA256, 210,000 rounds, random salt), `acp_` API keys and `aps_` session tokens (only SHA-256 hashes stored) |
| `src/permissions.js` | Roles, permissions, `canAssignRole`, `roleFromSsoGroups`, `demoUserRole` |
| `server.js` | Sign-in, sign-out, SSO, invites, workspace users, SCIM, platform-admin user routes, OAuth, session cookie, CSRF check |
| `src/middleware/rate-limiter.js` | `loginLimiter`: 20 a minute per IP (`RATE_LIMIT_LOGIN_PER_MINUTE`) |
| Storage | `users` (password hash, role, active, external id, invite token hash and expiry, last login), `dashboard_sessions`, `refresh_tokens` |
| `public/app.js` | Sign-in screen, set-password screen, Users page |

## What it does

- Lets people sign in with a password or the company's identity provider.
- Lets the company's directory decide who's in and what role they get (SSO groups), and turns accounts off when people leave (SCIM).
- Lets admins add, invite, change and deactivate people without a platform operator.
- Makes role changes and deactivations take effect immediately.
- Never lets a workspace lose its last owner.

## Where the data comes from

| Setting | Purpose |
| --- | --- |
| `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_REDIRECT_URI` | SSO provider. Its endpoints are read from `<issuer>/.well-known/openid-configuration` |
| `OIDC_DEFAULT_TENANT_ID` | The workspace SSO users land in. Without it, SSO works only when there's exactly one active workspace |
| `OIDC_GROUPS_CLAIM`, `OIDC_OWNER_GROUPS`, `OIDC_ADMIN_GROUPS`, `OIDC_MEMBER_GROUPS`, `OIDC_VIEWER_GROUPS` | Optional mapping from groups to roles |
| `SSO_ONLY` | Hides the password form on the sign-in screen |
| `DEMO_EMAIL`, `DEMO_PASSWORD`, `DEMO_ROLE` | Shared demo login (role member by default, never owner) |
| `DEMO_VIEWER_EMAIL`, `DEMO_VIEWER_PASSWORD` | A view-only demo login in the same workspace |
| `JWT_SECRET` | Signs OAuth access tokens (HS256) |
| `RATE_LIMIT_LOGIN_PER_MINUTE` | Sign-in rate limit per IP |

## How it works

### 1. Password sign-in (`POST /api/auth/login`)

1. Rate limit: 20 a minute per IP, shared with the other sign-in routes.
2. The email must be 3–256 characters and the password 8–256.
3. If the email is the demo email or the demo viewer email, that demo account is created or refreshed first (section 6).
4. Find **active** users with that email (any workspace) whose password matches, in an **active** workspace.
5. Create a dashboard session: a random `aps_` token, stored as a hash, valid for **7 days**.
6. Set the cookie `aps_session` (HttpOnly, SameSite=Lax, `Secure` in production, 7 days). The token is also returned as `sessionToken`, for use as the `X-Session-Token` header when cookies aren't available.
7. Record the last-login time and a `Dashboard Login` audit event.

A wrong email or password always gives `401 "Invalid email or password."`.

**CSRF:** a write request (anything other than GET or HEAD) authenticated only by the cookie must come from the same origin, or it's refused with `403 csrf_rejected`. Header-based credentials don't need this check.

**Sign-out** (`POST /api/auth/logout`) revokes the session, clears the cookie and writes `Dashboard Logout`.

### 2. SSO (OpenID Connect)

**Start** (`GET /auth/sso/login`):

1. Read the provider's settings (cached after the first time).
2. Make a random `state` and `nonce` and keep them in memory for 10 minutes. At most 10,000 sign-ins can be pending; the oldest are dropped first.
3. Set the cookie `aps_sso_state` (HttpOnly, SameSite=Lax, path `/auth/sso`), then redirect to the provider with scope `openid email profile`.

**Callback** (`GET /auth/sso/callback`). Any failure redirects to `/?sso_error=<code>`:

| Check | Error |
| --- | --- |
| The provider returned an error | its code |
| `state` is known, not expired, and matches the browser's state cookie | `invalid_state` |
| A `code` was returned | `missing_code` |
| The code exchange at the token endpoint worked | `token_exchange_failed` |
| An `id_token` came back | `missing_id_token` |
| `iss` is the issuer, `aud` includes our client id, `nonce` matches, `exp` hasn't passed | `invalid_id_token` |
| `email_verified` isn't false | `email_not_verified` |
| There's an `email` (or `preferred_username`) | `no_email_claim` |
| Group mapping is set and the person is in none of the groups | `not_authorized` |
| A target workspace can be found | `user_not_provisioned` |
| The account isn't deactivated | `account_disabled` |
| A new account fits within the seats | `no_seats` (the new account is deactivated) |

The `id_token`'s signature isn't checked again, because it came straight from the provider's token endpoint over TLS (allowed by OIDC Core 3.1.3.7). Its claims are checked as above.

**Account:** the person is matched by email **within the target workspace only**. If there's no match, a new account is created with role `member` (just-in-time sign-up). If the email exists in a different workspace, a warning is logged. If an SSO sign-in matches an account that has a password, that's logged too. When group mapping gives a role different from the stored one, the role is updated and audited. Then a session is created, as for a password sign-in, with an `SSO Login` audit event.

**Group mapping:** when any `OIDC_*_GROUPS` is set, the person gets the **highest** role whose group they're in. Someone in none of them is refused. With no mapping, roles are managed in Agent Prism.

### 3. Users, invites and password resets

Owners and admins manage people on the Users page (`users.manage`):

| Action | Rules |
| --- | --- |
| Add (`POST /api/tenant/users`) | Valid email up to 256 characters. Role defaults to member, and you can only give a role you're allowed to assign. There must be a free seat. Duplicates get `409`. Returns a **set-password link** valid for **7 days** |
| New link (`POST /api/tenant/users/:id/invite`) | For a first invite again, or a password reset. A new link cancels the old one. Not for deactivated users |
| Change (`PATCH /api/tenant/users/:id`) | Role, active and name (1–128 characters) |

The role rules:

- Admins manage members, viewers and other admins. Only owners can touch owners.
- Nobody can change their own role or deactivate themselves.
- A workspace must always keep one active owner.
- Reactivating someone needs a free seat.
- A role change or deactivation **signs the person out everywhere** straight away.

The set-password link carries its token in the URL **fragment** (`#invite=…`), so it never reaches server logs. `GET /api/auth/invite?token=` shows who the link is for. `POST /api/auth/accept-invite` sets the password, which must be **12–256 characters**. The link works once, and accepting it signs the person out of every other session and into a new one.

### 4. SCIM 2.0

The identity provider uses a workspace API key (`acp_…`) as its SCIM bearer token. The key needs `users.manage`, so it must be an admin or owner key.

| Route | What it does |
| --- | --- |
| `GET /scim/v2/ServiceProviderConfig` | Capabilities: patch yes, filter yes (up to 200), bulk, sort, etag and password change no |
| `GET /scim/v2/Users?filter=userName eq "x"` | List users. That's the only filter supported |
| `POST /scim/v2/Users` | Create, with `userName` or `emails[0].value`. Always role **member**. Needs a seat if active. Duplicates get `409 uniqueness` |
| `GET /scim/v2/Users/:id` | One user |
| `PUT /scim/v2/Users/:id` | Replace name, active and external id. Reactivating needs a seat |
| `PATCH /scim/v2/Users/:id` | `replace`/`add` of `active`, or a no-path value with `active` and `name`. Other operations are ignored |
| `DELETE /scim/v2/Users/:id` | Delete the user |

Every change is audited (`SCIM User Provisioned/Updated/Deactivated/Deleted`). A deactivated user's sessions stop working on the next request, because session checks require an active user. Roles can't be set over SCIM: use SSO groups, the Users page, or the platform-admin route below.

### 5. Platform-admin user routes (`X-Admin-Secret`)

| Route | What it does |
| --- | --- |
| `POST /api/admin/users/password` `{ tenantId, email, password }` | Sets a password (8–256 characters). Audited |
| `POST /api/admin/users/role` `{ tenantId, userId, role }` | Sets any role, including owner. Signs the person out everywhere. Audited |

### 6. Demo sign-in

- `GET /api/login-config` is public and returns `ssoEnabled`, `ssoOnly`, `demoEmail` and **`demoPassword`**. When both demo values are set, the sign-in screen fills them in and **signs in automatically**.
- `POST /api/auth/demo-login`, or a password sign-in with the demo email, runs `ensureDemoUser`:
  - If an account with that email exists (in any workspace), its password is reset to `DEMO_PASSWORD`, its role is capped at `DEMO_ROLE`, and its workspace's plan is raised from free or starter to **enterprise-trial**.
  - Otherwise the account is created in the **oldest active workspace**.
- The demo viewer login works the same way, with role viewer, in the same workspace.

### 7. OAuth token exchange (for SDKs)

- `POST /api/oauth/token` `{ grant_type: "client_credentials", client_secret: "<acp_ key>" }`. Rate limited and checked against the key's IP allowlist. It returns a **1-hour** HS256 access token (workspace, key id, scopes) and a **30-day** refresh token. `OAuth Token Issued` is audited. `client_id` is ignored.
- `POST /api/oauth/refresh` `{ refresh_token }` works only while the token, its API key and the workspace are all active. It **rotates** the refresh token (the old one is revoked) and returns a new access token.
- A request with an access token keeps the key's read-only flag, and is refused once the key is revoked (see [multi-tenancy](multi-tenancy-and-api-keys.md)).

## API

| Method and path | Auth | Purpose |
| --- | --- | --- |
| `POST /api/auth/login` | Public, rate limited | Password sign-in |
| `POST /api/auth/demo-login` | Public, rate limited | Demo sign-in |
| `POST /api/auth/logout` | Session cookie | Sign out |
| `GET /auth/sso/login`, `GET /auth/sso/callback` | Public | SSO |
| `GET /api/login-config` | Public | Sign-in screen settings |
| `GET /api/me` | Tenant | Workspace, user, auth type, role, permissions |
| `GET/POST /api/tenant/users`, `PATCH /api/tenant/users/:id`, `POST /api/tenant/users/:id/invite` | `users.manage` | Manage people |
| `GET /api/auth/invite`, `POST /api/auth/accept-invite` | Public, rate limited | Set-password link |
| `/scim/v2/*` | Admin or owner API key | Provisioning |
| `POST /api/admin/users/password`, `POST /api/admin/users/role` | Platform admin | Operator fixes |
| `POST /api/oauth/token`, `POST /api/oauth/refresh` | API key / refresh token | SDK tokens |

## Dashboard / UI

- **Sign-in screen.** "Sign in with SSO" when SSO is set; the email and password form unless `SSO_ONLY`; demo values filled in, with an automatic sign-in when set.
- **Set-password screen**, opened from an invite or reset link. It shows who the link is for and asks for a password of 12 or more characters.
- **Admin → Users.** List with role, active, invite pending and its expiry, last sign-in; add a person; copy the set-password link; change role; deactivate or reactivate.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does for sign-in and accounts |
| --- | --- |
| `src/auth.js` | Password hashing, tokens, key hashes |
| `src/permissions.js` | Roles, assignment rules, SSO group mapping, demo role |
| `src/validation.js` | `login`, `setUserPassword`, `setUserRole` schemas |
| `src/middleware/rate-limiter.js` | Sign-in rate limit |
| `server.js` | All routes above, session cookie, CSRF check |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | Users, sessions, invites, SSO upsert, SCIM, demo user, refresh tokens |
| `public/app.js` | Sign-in, set-password and Users screens |

## Tests

What's tested today:

- `test/auth-login.test.js`: a password hash verifies only its own password; session tokens give only a token and its hash.
- `test/sso.test.js`: a sign-in completes on the browser that started it; a callback without that browser's state cookie is refused (login CSRF); a wrong nonce, audience or issuer is refused; an unverified email can't sign in.
- `test/user-management.test.js`: what each key acts as; each role's permissions; only owners touch owners; SSO groups (highest wins, no match refused); password sign-in no longer hands out an owner API key; invite and set-password; `/api/me`; what members, viewers and admins can and can't do; an admin's key acts as an admin, also as a JWT; IP allowlist format; the demo login is never an owner; role changes apply straight away; deactivation signs out and blocks sign-in; a password reset signs out other sessions; the last owner is protected; duplicate and bad emails are rejected.
- `test/sessions.test.js` and `test/hardening.test.js` also touch sessions and SCIM.

Not tested yet, and worth adding:

- Each SCIM route against a real IdP's request shapes (Okta, Entra ID).
- OAuth refresh: rotation, and refusal after the key is revoked.
- `SSO_ONLY` (see limitation 1).
- The demo login's effect on an existing workspace (see limitation 2).

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **`SSO_ONLY` only hides the password form.** The server still accepts `POST /api/auth/login`, so anyone with a password, including one set by invite or by the platform admin, can still sign in with it. A customer that requires SSO (and its MFA) doesn't really get it.
2. **The demo login can expose and change a real workspace.** When `DEMO_EMAIL` and `DEMO_PASSWORD` are set, the password is public (`/api/login-config`) and the sign-in screen uses it automatically. The demo account is created in the **oldest active workspace**, which on a shared installation is a real customer's, and that workspace's plan is raised to enterprise-trial. If `DEMO_EMAIL` matches an existing person, their password is reset and their role lowered.
3. **SSO serves one workspace per installation.** SSO users land in `OIDC_DEFAULT_TENANT_ID`, or in the only active workspace. With several workspaces and no default, SSO fails with `user_not_provisioned`. There's one identity provider per installation, not one per customer.
4. **Anyone who can sign in to the identity provider gets an account.** Without group mapping, every person the provider lets through is created as a member (seats permitting). Nothing requires them to have been provisioned or invited first.
5. **SSO breaks with more than one server process.** The pending `state` and `nonce` are kept in memory. With `npm run cluster`, the callback may reach a different worker and fail with `invalid_state`.
6. **Weak spots in SCIM.** Only `userName eq` filtering, no paging (`startIndex` and `count` are ignored), no groups and no roles. SCIM can deactivate or delete the **last owner**: the last-owner check exists only on the Users page. The seat limit is reported as `403 invalidValue`, which IdPs may not expect.
7. **Password rules differ.** Invite and reset links need 12 or more characters, but the platform-admin route and the sign-in form accept 8. There's no check against common or breached passwords, and no lockout apart from the rate limit of 20 a minute per IP.
8. **The platform-admin password route doesn't sign the person out**, unlike a reset link.
9. **Sessions last 7 days with no idle timeout**, whatever the customer's policy. Only signing out, a role change, deactivation or a password reset ends them early.
10. **One email in two workspaces signs in to whichever matches first.** Password sign-in looks at every workspace for that email and takes the first account whose password matches.
11. **OAuth details.** `client_id` is ignored, and a refreshed access token always has scopes `["*"]` instead of the key's own. Refreshing isn't rate limited or audited. Access tokens are signed with one shared secret (HS256).
12. **SSO errors can leak internals.** An unexpected error in the callback puts its message in the redirect URL (`/?sso_error=…`).
13. **No multi-factor authentication for password accounts.** MFA is available only through the identity provider when SSO is used.
