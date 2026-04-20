# Tochka Bank API — Authentication

Two authentication flows: JWT (simple, single-company) and OAuth 2.0 with Consents (multi-tenant). Pick JWT unless building a SaaS that authorizes access to other people's accounts.

## Table of Contents

- [JWT (recommended for own-account automation)](#jwt-recommended-for-own-account-automation)
- [OAuth 2.0 + Consent flow (multi-tenant)](#oauth-20--consent-flow-multi-tenant)
- [Sandbox](#sandbox)
- [Permissions](#permissions)
- [Token lifecycle and 403 handling](#token-lifecycle-and-403-handling)

## JWT (recommended for own-account automation)

### Generation steps (in online banking)

1. Sign in to https://enter.tochka.com (web online-banking)
2. Open **Интеграции и API** → **Подключить**
3. Choose **Сгенерировать JWT-ключ**
4. Set:
   - Token name (any label)
   - TTL (срок доступа токена) — pick the shortest workable duration
   - Permissions (see [Permissions](#permissions)) — uncheck what you don't need
   - Companies — by default all the user's companies are included
5. Confirm via SMS code
6. Copy both the **JWT** and the **client_id** (client_id is needed for webhook setup)

The JWT cannot be re-displayed after closing the page. Save it to a secret manager immediately.

### Using the JWT

Send on every request:
```
Authorization: Bearer <jwt>
```

Example:
```bash
curl -H "Authorization: Bearer eyJhbGciOi..." \
     https://enter.tochka.com/uapi/open-banking/v1.0/accounts
```

### Notes

- Editing token permissions is not supported — any change requires reissuance (which produces a new JWT and new `client_id`)
- Revocation: delete the token in online banking — subsequent calls return HTTP 403
- Reissuance reminder appears about 1 week before TTL expiry
- Source: https://developers.tochka.com/docs/tochka-api/algoritm-raboty-s-jwt-tokenom

## OAuth 2.0 + Consent flow (multi-tenant)

Use only when an end user (a different person/company than the integrator) needs to grant your app access to their Tochka account. Standard OAuth 2.0 with two grant types: `client_credentials` (for app-to-app calls) and `authorization_code` (for delegated user access).

### High-level flow

1. **Register the app** with Tochka — receive `client_id` and `client_secret`
2. **Get an app token** via `client_credentials`:
   ```
   POST https://enter.tochka.com/connect/token
   Content-Type: application/x-www-form-urlencoded
   grant_type=client_credentials&client_id=...&client_secret=...&scope=accounts payments
   ```
3. **Create a Consent** (defines what the user is being asked to authorize):
   ```
   POST https://enter.tochka.com/uapi/consent/v1.0/consents
   Authorization: Bearer <app_token>
   Content-Type: application/json
   { "Data": { "permissions": ["ReadAccountsBasic", "ReadStatements"] } }
   ```
   Response includes `Data.consentId`.

   Canonical path per swagger (v1.90.4-stable): `/uapi/consent/v1.0/consents`. The legacy alias `/uapi/v1.0/consents` (no `consent/` segment) still resolves in production as of 2026-04-20, but new code should use the canonical form.
4. **Redirect the user** to authorize the consent:
   ```
   https://enter.tochka.com/connect/authorize?
     response_type=code&
     client_id=<your_client_id>&
     redirect_uri=<your_callback>&
     scope=accounts statements&
     consent_id=<consentId>
   ```
5. **Exchange the returned `code` for tokens**:
   ```
   POST https://enter.tochka.com/connect/token
   grant_type=authorization_code&code=...&redirect_uri=...&client_id=...&client_secret=...
   ```
6. Use the resulting `access_token` (24h) for API calls. Refresh with the `refresh_token` (30d).

### Token TTLs

| Token | Lifetime |
|-------|----------|
| Access token | 24 hours |
| Refresh token | 30 days |
| JWT (online-banking) | User-configurable |

Source: https://developers.tochka.com/docs/tochka-api/algoritm-raboty-po-oauth-2.0

## Sandbox

Base URL: `https://enter.tochka.com/sandbox/v2`

Sandbox requires no registration. Use the documented sandbox JWT (typically `Bearer working_token` — confirm against current sandbox doc page: https://developers.tochka.com/docs/tochka-api/pesochnica). Sandbox returns mock data and accepts mock writes, but not all production endpoints are mirrored — verify presence before integrating.

## Permissions

Granular permissions selectable when generating a JWT or requesting an OAuth consent. The list below is **authoritative** — copied verbatim from Tochka's Consent API validation response on 2026-04-20. Any value not in this list is rejected.

### Full accepted set (19 values)

```
ReadAccountsBasic, ReadAccountsDetail, ReadBalances, ReadStatements,
ReadTransactionsBasic, ReadTransactionsCredits, ReadTransactionsDebits,
ReadTransactionsDetail, ReadCustomerData, ReadSBPData, EditSBPData,
CreatePaymentForSign, CreatePaymentOrder, ReadAcquiringData,
MakeAcquiringOperation, ManageInvoiceData, ManageWebhookData,
MakeCustomer, ManageGuarantee
```

(Source: error body from `POST /uapi/v1.0/consents` when an invalid permission is submitted — Tochka helpfully lists all accepted values. Tochka's public docs at https://developers.tochka.com/docs/tochka-api/api/rabota-s-razresheniyami are out of date; the response above is the ground truth.)

### Grouped by use case

**Accounts & balances**
- `ReadAccountsBasic` — list accounts (numbers, currencies)
- `ReadAccountsDetail` — full account details
- `ReadBalances` — account balances
- `ReadCustomerData` — legal entity data (**not** `ReadCustomers` as older docs say)

**Statements & transactions**
- `ReadStatements` — full statement (Open Banking async flow)
- `ReadTransactionsBasic` — basic transaction data
- `ReadTransactionsCredits` — incoming transactions only
- `ReadTransactionsDebits` — outgoing transactions only
- `ReadTransactionsDetail` — full transaction details

**Outgoing payments**
- `CreatePaymentForSign` — draft payment orders (land in «На подпись»)
- `CreatePaymentOrder` — direct payment order creation

**Invoices (счета на оплату)**
- `ManageInvoiceData` — create / read / update / delete invoices (covers all — there is **NO** `ReadInvoiceData`)

**SBP (Система быстрых платежей)**
- `ReadSBPData` — read merchant SBP data
- `EditSBPData` — register / edit QR codes, merchant settings

**Acquiring (интернет-эквайринг)**
- `ReadAcquiringData` — read acquiring operations
- `MakeAcquiringOperation` — create / manage acquiring operations (**not** `EditAcquiringData`)

**Webhooks**
- `ManageWebhookData` — configure webhook subscriptions

**Specialized (usually not needed for single-company automation)**
- `MakeCustomer` — create customers (partner-bank use case)
- `ManageGuarantee` — bank guarantees

**Principle of least privilege:** only request permissions you actually need — for OAuth, changing a consent after creation means recreating it and re-running user authorization. `scripts/tochka_client.py` bakes in 17 of these (all except `MakeCustomer` and `ManageGuarantee`) as the `DEFAULT_OAUTH_PERMISSIONS` constant.

## Token lifecycle and 403 handling

When an API call returns 403:

1. Check token expiry (JWT — decode the `exp` claim; OAuth — check refresh date)
2. If expired:
   - JWT: regenerate in online banking
   - OAuth: refresh via `grant_type=refresh_token`
3. If not expired, the permission scope is wrong — verify the token includes the permission for the endpoint being called
4. Sandbox sometimes returns 403 on endpoints it doesn't implement — try the same call against production with a minimal-permission token

For OAuth, refresh:
```bash
curl -X POST https://enter.tochka.com/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token&refresh_token=<rt>&client_id=...&client_secret=..."
```

## Wizards and OAuth app registration

### Why the `!` prefix is mandatory

Both `init` and `init --oauth` read secrets with `getpass` (and `input()` for confirmations). These require a real terminal (`termios`); Claude Code's Bash tool doesn't provide one, so running through the agent hangs silently or fails with `EOFError`. The wizard now detects this on startup and exits with an actionable message.

**Correct pattern:** the agent instructs the user to run the command in their own shell via the `!` prefix:

```
! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init
```

With `!` the command runs in the user's live TTY, so the JWT / OAuth secret is entered directly into their terminal and never passes through the agent's tool-use channel or the conversation transcript. The agent still sees the wizard's stdout and can continue.

**Never ask the user to paste a production JWT into chat** — it would land in transcript backups. If the user has already pasted one in error, immediately rotate: ЛК Точки → Интеграции и API → delete the token → generate a new one. For the sandbox `working_token` pasting is fine; only the production token warrants the `!` flow.

### `init` wizard (personal JWT)

The wizard:
1. Explains how to generate a JWT in the Точка online banking (Интеграции и API → Сгенерировать JWT-ключ) and which permissions to tick.
2. Reads the JWT with hidden input (`getpass` — not recorded in shell history).
3. **Validates** the token against PROD by calling `GET /accounts` — only saves on success.
4. **Stores the JWT in the OS credential store** (encrypted at rest, password/biometry-gated):
   - macOS → Keychain (`security add-generic-password`)
   - Linux → `secret-tool` (GNOME Keyring / KWallet via libsecret)
   - Windows → Credential Manager
   - WSL → falls back to Windows Credential Manager via `powershell.exe`
5. If the credential store isn't available, falls back to `~/.config/tochka-bank-api/token` with `chmod 600` (and warns that this is less secure).
6. Prints the `customerCode` + `accountId` you'll need for invoice/statement commands.
7. Saves defaults to `~/.config/tochka-bank-api/config.json`.

Force file-only storage (e.g. in a headless environment): `init --storage file`.

For sandbox: prefix with `TOCHKA_SANDBOX=1`. Sandbox uses a shared `working_token` — easier to just `export TOCHKA_TOKEN=working_token TOCHKA_SANDBOX=1` without running init.

### OAuth app registration (needed only for `init --oauth`)

One-time step at https://i.tochka.com/bank/services/m/integration/new — required if you want Invoice API, closing documents, or the full Open Banking statement. Verified pitfalls (2026-04-20):

- **`Redirect URL` must be HTTPS.** Plain `http://` is rejected.
- **`localhost` is not accepted** in the Redirect URL field (despite what Tochka's own example docs show). Use `127.0.0.1` or a real domain. The form silently rejects `https://localhost:8443/callback` but accepts `https://127.0.0.1:8443/callback`.
- Recommended template: `https://127.0.0.1:8443/callback`. Port can be anything free; `/callback` is arbitrary but fixed once registered.
- `client_secret` **cannot be re-viewed** after the registration form closes — if you lose it, you must regenerate or recreate the app.

`init --oauth` defaults to `https://127.0.0.1:8443/callback`. Pass `--redirect-url <full URL>` if your registered URL differs (missing this flag = `invalid_redirect_uri` at the browser stage, then a silent 5-min callback-server timeout — always pre-check the match). The wizard starts a local HTTPS server (using `mkcert` if installed, otherwise self-signed — expect a browser warning; click through), opens the browser to `connect/authorize`, exchanges the code, and stores `client_id` / `client_secret` / `refresh_token` in Keychain.

Refresh token lifetime is 30 days. If nothing calls an OAuth endpoint for that long the token dies silently — symptom is 401 on `list-statement` / `create-invoice`. Fix: re-run `init --oauth`.
