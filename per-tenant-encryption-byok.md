# Per-workspace encryption and customer-managed keys (BYOK)

## Introduction

Agent Prism stores provider credentials for each workspace: Anthropic, OpenAI and Azure keys for the gateway, the GitHub token for Copilot, and the SIEM signing secret. These are **encrypted** using **envelope encryption**:

- Each workspace has its own random **data encryption key (DEK)**. Connector secrets are encrypted with it using AES-256-GCM.
- The DEK is stored only in **wrapped** (encrypted) form. What wraps it depends on the workspace:

| Mode | DEK is wrapped with | Who can decrypt |
| --- | --- | --- |
| **Platform-managed** (default) | The installation's master key, `ENCRYPTION_KEY` | The server, at any time |
| **Customer-managed (BYOK)** | A key derived from a secret **only the customer has** | The server only while the workspace is **unsealed**: after an owner enters the secret, for 30 minutes, in that server process's memory |

Agent Prism never stores a BYOK workspace's secret. After a restart, or after 30 minutes, the workspace is **sealed** again. Its stored credentials can't be decrypted, so the gateway returns `423` until an owner unseals it.

Example: Acme's owner sends `POST /api/tenant/encryption-key` with a 64-character hex key from `openssl rand -hex 32`. The workspace's DEK is re-wrapped with that key and the workspace becomes customer-managed. It stays unsealed for 30 minutes. The next morning, after a redeploy, Acme's agents get `423 tenant_encryption_sealed` from `/v1/messages` until the owner sends `POST /api/tenant/encryption-key/unseal` with the same key.

This document describes encryption **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | What it does |
| --- | --- |
| `src/crypto.js` | AES-256-GCM (`enc:v1:<iv>:<tag>:<ciphertext>`), master key, DEK generate, wrap and unwrap, customer key derivation, `resolveTenantDek`, `encryptConnectorConfig`/`decryptConnectorConfig` |
| `src/tenant-keys.js` | In-memory cache of unsealed DEKs (`TENANT_KEY_UNSEAL_TTL_MS`, default 30 minutes) |
| `server.js` | `POST /api/tenant/encryption-key`, `/unseal`, `/seal`; `sendGatewayKeyError` (423 when sealed) |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | `resolveTenantDekById` / `resolveTenantDek` (create a DEK the first time and re-encrypt old connector secrets), `setTenantEncryptionKey`, `unsealTenantKey`, connector create and read |
| `src/startup.js` | Production won't start without `ENCRYPTION_KEY`; warns if it isn't 64 hex characters |
| Storage | `tenants.wrapped_dek`, `tenants.key_management` (`platform` or `customer`), `tenants.key_derivation_salt` |

## What it does

- Keeps provider keys unreadable in the database, in backups and in the JSON file.
- Gives each workspace its own key, so one workspace's secrets can't be decrypted with another's key.
- Lets a customer hold the key themselves (BYOK): without them, the operator can't read their stored credentials.
- Detects tampering: GCM refuses changed ciphertext.

## Where the data comes from

| Input | Use |
| --- | --- |
| `ENCRYPTION_KEY` | The master key. 64 hex characters are used directly; anything else is hashed with SHA-256. **Required in production** |
| `customerKey` (in the request) | The BYOK secret. 16 or more characters. 64 hex characters are used directly; anything else goes through PBKDF2-SHA256 (210,000 rounds) with a random salt saved for the workspace |
| `TENANT_KEY_UNSEAL_TTL_MS` | How long an unsealed DEK stays in memory (default 1,800,000 ms) |

**What's encrypted:** only the connector `apiKey` field, which also holds the GitHub token and the SIEM secret. Everything else is stored as plain data: runs, prompt captures, audit log, users, connector URLs and org names.

## How it works

### 1. A workspace's DEK

- The first time a workspace's DEK is needed, a random 32-byte DEK is created, wrapped with the master key and saved. Saving uses a "claim if still empty" update, so two requests at once can't create two DEKs.
- Connector secrets saved under the **older scheme** (encrypted straight with the master key) are re-encrypted with the new DEK at that point.

### 2. Saving and reading a connector secret

- **Save:** resolve the DEK, encrypt `apiKey`, store.
- **Read:** resolve the DEK, decrypt `apiKey`. When the DEK can't be resolved (sealed, or no master key), the connector comes back with `apiKey` removed and `_sealed: true`. One connector failing to decrypt doesn't mark the others sealed.

**Resolving the DEK:**

| Mode | How |
| --- | --- |
| Platform | Unwrap with `ENCRYPTION_KEY` every time |
| Customer | Take it from the in-memory cache. If it isn't there or has expired: error `sealed` |

### 3. Switching to BYOK (`POST /api/tenant/encryption-key`)

1. The caller must be signed in to the dashboard, be an owner or admin, **and** hold `data.erase`. That permission is owner-only, so in practice only owners pass.
2. `customerKey` must be 16 or more characters, or the request is refused with `422`.
3. The workspace's **current** DEK is resolved: unwrapped with the master key, or taken from the cache if the workspace is already customer-managed.
4. A new salt is made, the wrapping key is derived from `customerKey`, and the **same DEK** is re-wrapped with it. `key_management` is set to `customer`.
5. The DEK is cached, so the workspace stays unsealed for 30 minutes.
6. `Tenant Encryption Key Set (BYOK)` is audited. If the key wasn't hex, the response suggests using a 64-character hex key.

Because the DEK stays the same, existing connector secrets don't need re-encrypting. Calling this again while unsealed **changes** the customer key.

### 4. Unseal and seal

- **Unseal** (`/api/tenant/encryption-key/unseal`, same permission rules): the wrapping key is derived with the stored salt (or the legacy unsalted SHA-256 for workspaces set up before salts existed), and the DEK is unwrapped. A wrong key fails the GCM check and returns `401 invalid_key`; a platform-managed workspace returns `409 not_customer_managed`. On success the DEK is cached and `Tenant Encryption Unsealed` is audited.
- **Seal** (`/api/tenant/encryption-key/seal`): the DEK is removed from the cache at once, and `Tenant Encryption Sealed` is audited.

### 5. While sealed

| Feature | What happens |
| --- | --- |
| Gateway (`/v1/*`) | `423 tenant_encryption_sealed`: "An owner/admin must unseal…" |
| Copilot sync | The connector has no token, so it's reported as not configured |
| SIEM delivery | Skipped without notice (see [audit, GDPR and SIEM](audit-gdpr-and-siem.md)) |
| Passive ingest, dashboard, reports | Unaffected: they don't need connector secrets |

## API

| Method and path | Auth | Body | Returns |
| --- | --- | --- | --- |
| `POST /api/tenant/encryption-key` | Signed-in owner | `{ customerKey }` (16 or more characters) | `{ status: "customer_managed", message, recommendation? }` |
| `POST /api/tenant/encryption-key/unseal` | Signed-in owner | `{ customerKey }` | `{ status: "unsealed" }`, `401 invalid_key`, `409 not_customer_managed` |
| `POST /api/tenant/encryption-key/seal` | Signed-in owner | — | `{ status: "sealed" }` |

## Dashboard / UI

There's no screen for this. BYOK is set up and unsealed through the API only, and the dashboard doesn't show whether a workspace is sealed, apart from gateway errors.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does for encryption |
| --- | --- |
| `src/crypto.js` | Ciphers, keys, wrap and unwrap, key derivation |
| `src/tenant-keys.js` | Unsealed DEK cache |
| `src/startup.js` | `ENCRYPTION_KEY` checks |
| `server.js` | BYOK routes, 423 handling |
| `src/stores/postgres-store.js`, `src/stores/file-store.js` | DEK creation and migration, BYOK storage, connector encrypt and decrypt |

## Tests

What's tested today:

- `test/byok-key-derivation.test.js`: the same secret and salt give the same key; a different salt gives a different key; a 64-character hex key is used directly; no salt falls back to legacy SHA-256; salted and legacy keys differ; salts are random.
- `test/byok-file-store.test.js`: set then unseal works with a fresh salt; the wrong secret is refused; a legacy unsalted workspace still unseals with its original secret.

Not tested yet, and worth adding:

- The three routes, including the owner-only check and `409`/`401`.
- The gateway returning `423` while sealed, and working again after unseal.
- The TTL expiry and sealing.
- Changing the key while unsealed, and calling the set route while **sealed** (see limitation 4).
- The Postgres migration of old connector secrets to the DEK.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **BYOK workspaces seal themselves every 30 minutes.** The unsealed DEK expires after `TENANT_KEY_UNSEAL_TTL_MS`, even while agents are using it. The gateway then returns `423` until an owner unseals it again by hand, and nothing re-unseals automatically. For a busy customer this means regular outages.
2. **Unsealing works for one server process only.** The cache is in memory. With `npm run cluster` or several instances, the unseal request reaches one worker, and the others stay sealed. Every restart or deploy seals the workspace.
3. **BYOK only protects connector secrets.** Runs, prompt captures, audit logs, users, connector URLs and Copilot org names aren't encrypted by Agent Prism at all. "Customer-managed encryption" can easily be read as covering the customer's data, and it doesn't.
4. **Changing the key needs the workspace unsealed, and a failure isn't handled.** Calling `POST /api/tenant/encryption-key` while sealed throws the `sealed` error inside the route, which isn't caught, so the caller gets a `500` instead of a clear message.
5. **No way back, and the DEK is never replaced.** A workspace can't go back to platform-managed. Changing the key only re-wraps the **same** DEK, so anyone holding an old wrapped DEK and the old customer key can still decrypt. There's no rotation of the master `ENCRYPTION_KEY` either: changing it makes every platform-managed workspace's secrets unreadable.
6. **A lost customer key means lost connector secrets.** There's no recovery. The connectors have to be re-entered after resetting the workspace's encryption, and there's no route to do that reset.
7. **Admins can't use these routes, although the messages say they can.** The routes need `data.erase` (owner) and then also check for owner **or admin**. Admins always fail the first check.
8. **No screen and no status.** BYOK is API-only, nothing shows whether a workspace is sealed, and there's no warning before the 30 minutes run out.
9. **A weak passphrase is allowed.** Any secret of 16 or more characters is accepted; a 64-character hex key is only *recommended*. Unseal attempts aren't rate-limited beyond the general limits, although they need a signed-in owner.
10. **Key derivation blocks the server briefly.** PBKDF2 with 210,000 rounds runs synchronously on each set and unseal, pausing other requests for that moment.
11. **Development installs store DEKs unprotected.** Without `ENCRYPTION_KEY` (allowed outside production), the "wrapped" DEK is stored as plain base64, so the encryption is only nominal. Startup warns about this.
