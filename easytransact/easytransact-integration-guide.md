# EasyTransact API

EasyTransact is the API gateway financial institutions use to access EasyPay's 500+ payment providers (utilities, telcos, banks, card-to-card transfers, etc.) without integrating directly with EasyPay. This guide covers everything you need to integrate.

## Base URL

| Environment | Base URL |
| --- | --- |
| Production | `https://betransact.easypay.am/api` |

All endpoints below are prefixed with `/v1/`. For example, the login endpoint is `https://betransact.easypay.am/api/v1/login`.

## Response envelope

Every endpoint returns a `ServiceResponse<T>` envelope. Non-paginated:

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": { /* T */ }
  }
}
```

Paginated (`ServiceResponsePaged<T>`):

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": [ /* T[] */ ],
    "page": 1,
    "pageSize": 20,
    "totalCount": 142
  }
}
```

Endpoints that perform an action without returning data (e.g. logout) reply with the envelope minus `responseData.data`.

## Errors

Errors come back with `success: false`, a human-readable `message`, and a typed `responseStatus`. Inspect `responseStatus` for programmatic handling.

```json
{
  "success": false,
  "message": "Invalid credentials",
  "responseStatus": "BadRequest",
  "responseData": null
}
```

`responseStatus` values you should handle:

| Status | HTTP | Meaning |
| --- | --- | --- |
| `Success` | 200 | Request succeeded. |
| `BadRequest` | 400 | Validation, business-rule, or upstream rejection. The `message` is safe to display. |
| `Unauthorized` | 401 | Token missing, invalid, or expired. Re-login. |
| `NotFound` | 404 | Resource does not exist. |
| `ServiceUnavailable` | 503 | Upstream provider temporarily unreachable. Retry with backoff. |

## Headers

| Header | Required | Description |
| --- | --- | --- |
| `token` | Required after login | Session token issued by `POST /v1/login`. UUID format. Sent on every protected call. |
| `language` | Optional | `en` or `hy`. Affects provider hints returned by `/v1/user-providers`. Defaults to `en`. |
| `Content-Type` | Required for body endpoints | `application/json`. |

## Authentication model

EasyTransact uses opaque session tokens (UUIDs) backed by a database row, **not JWTs**. There is no refresh token: when a token expires you receive `401 Unauthorized` and must re-authenticate.

The token slides forward on every authenticated call: each successful call extends its expiration window. Idle sessions expire after the configured TTL (production default: very long; treat 401 as the only authoritative expiration signal).

### Login flow

1. Call `POST /v1/login` with username and password.
2. The response carries:
   - A session `token` (UUID) ready for use **OR**
   - A one-time token (`token`) plus `forceToChangePassword: true` if this is the user's first login.
3. If `forceToChangePassword` is `true`, call `PATCH /v1/change-own-password-forced` with that one-time token in the `token` header and a new password in the body. The response of that call returns a fresh session token.
4. Use the session token in the `token` header for every subsequent request.
5. On `401 Unauthorized` from any endpoint, the token is no longer valid; re-run the login flow.

### Logout

`POST /v1/logout` invalidates the current token. Subsequent requests with that token receive `401`.

## Endpoints

### Login

```
POST /api/v1/login
```

Authenticate a user and obtain a session token.

**Request body**

```json
{
  "username": "panda.user",
  "password": "S3cret!Pass"
}
```

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": {
      "id": "1k3ZA",
      "token": "f6816910-914e-48a1-a40a-06cb6aead7e9",
      "fullName": "Panda User",
      "username": "panda.user",
      "forceToChangePassword": false,
      "role": "User"
    }
  }
}
```

Notes:
- `id` is the user's id encoded as a base-36 string (the underlying type is a 64-bit integer).
- `role` is one of `User`, `Admin`, `SuperAdmin`. Integrators are typically `User`.
- If `forceToChangePassword` is `true`, the returned `token` is single-use and only valid against `/v1/change-own-password-forced`.

**Errors**
- `BadRequest` — username/password invalid, or user is disabled.
- `Unauthorized` — too many failed attempts (3 failures in 30 seconds blocks further attempts).

### Logout

```
POST /api/v1/logout
```

Invalidate the current session token.

**Headers** — `token` (required).

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success"
}
```

### Change own password (forced)

```
PATCH /api/v1/change-own-password-forced
```

Used immediately after a login that returned `forceToChangePassword: true`. Sends the one-time token and a new password; the response returns a fresh session token (login again with the same flow afterwards is not required).

**Headers** — `token` (the one-time token from login).

**Request body**

```json
{
  "password": "N3wS3cret!Pass"
}
```

**Response** — same shape as `POST /v1/login`, with `forceToChangePassword: false` and a usable session token.

### Change own password

```
PATCH /api/v1/change-own-password
```

Routine password change for the authenticated user.

**Headers** — `token` (required).

**Request body**

```json
{
  "oldPassword": "S3cret!Pass",
  "newPassword": "N3wS3cret!Pass"
}
```

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success"
}
```

### Get current user

```
GET /api/v1/user-identify
```

Return the authenticated user's profile. Useful after login if you discarded part of the login response, or to detect role changes.

**Headers** — `token` (required).

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": {
      "id": "1k3ZA",
      "token": "f6816910-914e-48a1-a40a-06cb6aead7e9",
      "fullName": "Panda User",
      "username": "panda.user",
      "forceToChangePassword": false,
      "creationDate": "2024-09-12T10:11:31.000Z",
      "role": "User"
    }
  }
}
```

### List available providers

```
GET /api/v1/user-providers
```

Returns the providers this user is entitled to use, with their input-field metadata, amount limits, and verification requirement.

**Headers** — `token` (required), `language` (optional, `en` or `hy`).

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": [
      {
        "id": 17667,
        "name": "EasyPay Wallet",
        "isEnabled": true,
        "minAmount": 10,
        "maxAmount": 200000,
        "input1Object": {
          "hint": "Phone number",
          "minLength": 12,
          "maxLength": 12,
          "regexp": "",
          "prefix": "+374"
        },
        "input2Object": null,
        "input3Object": null,
        "input4Object": null,
        "requiresVerification": false
      },
      {
        "id": 18900,
        "name": "Card-to-card transfer (KYC)",
        "isEnabled": true,
        "minAmount": 100,
        "maxAmount": 1000000,
        "input1Object": {
          "hint": "Card number",
          "minLength": 16,
          "maxLength": 16,
          "regexp": "",
          "prefix": ""
        },
        "input2Object": null,
        "input3Object": null,
        "input4Object": null,
        "requiresVerification": true
      }
    ]
  }
}
```

Important fields:
- `requiresVerification` — when `true`, you must verify the payer's identity (see [Payment identity verification](#payment-identity-verification)) before calling `/v1/pay`. The flag is sourced live from EasyPay; it can flip without notice.
- `inputNObject` — describes a form field this provider expects in `/v1/check` and `/v1/pay`. Up to 4 inputs per provider; `null` means the field is unused. `prefix` is a fixed string prepended to user input (e.g. `+374` for Armenian phone numbers).

### Identity lookup

```
POST /api/v1/identity-lookup
```

Identify a payer by SSN or document number. On success EasyTransact persists the resolved identity internally and returns a single-use `reference` UUID. **No personal fields are returned** — by design, so the endpoint cannot be used as an identity-harvesting oracle. Pass the `reference` to `/v1/pay`; the persisted identity is forwarded to the upstream KYC step server-side.

Use this as the primary path. Fall back to `/v1/identity-manual` when the lookup cannot identify the payer (foreign citizen with no local record, off-roll, or a typo in the document number).

**Headers** — `token` (required).

**Request body**

```json
{
  "lookupType": "Ssn",
  "value": "1234567890"
}
```

`lookupType` accepts `"Ssn"` or `"Document"`. `value` is the SSN or document number.

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": {
      "reference": "8c2f6f4a-9c1e-4e7d-9b8f-2c4f1c7f6d9a"
    }
  }
}
```

`reference` is **single-use**: passing it to `/v1/pay` consumes it. To pay for the same payer again, look them up again.

**Errors**
- `BadRequest` — value missing, lookup type invalid, no supported document found, document type unsupported.
- `NotFound` — no record matching the value. Most often a foreign citizen, an off-roll payer, or a typo. Fall back to `/v1/identity-manual`.
- `ServiceUnavailable` — the lookup service is unreachable. Retry later or fall back to manual entry.

### Identity manual entry

```
POST /api/v1/identity-manual
```

Submit identity fields entered by the operator. Use when the automatic lookup could not identify the payer (foreign citizen, off-roll, or document mistyped) or when the lookup service is unavailable.

Returns a `reference` UUID **plus** the submitted fields echoed back as a `person` object — only what the caller already supplied. Nothing extra is leaked here.

**Headers** — `token` (required).

**Request body**

```json
{
  "firstName": "John",
  "lastName": "Doe",
  "ssn": "1234567890",
  "document": "AB1234567",
  "documentType": 1,
  "documentIssuedBy": "003",
  "documentIssuedDate": "2019-02-13",
  "documentValidityDate": "2029-02-13"
}
```

Field rules:
- All string fields are required.
- `documentType`: `1` (Passport), `2` (Biometric Passport), or `3` (ID Card). Anything else is `BadRequest`.
- `documentIssuedDate` and `documentValidityDate` are ISO `yyyy-MM-dd` strings.
- `documentValidityDate` must be on or after `documentIssuedDate`.

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": {
      "reference": "8c2f6f4a-9c1e-4e7d-9b8f-2c4f1c7f6d9a",
      "person": {
        "firstName": "John",
        "lastName": "Doe",
        "ssn": "1234567890",
        "document": "AB1234567",
        "documentType": 1,
        "documentIssuedBy": "003",
        "documentIssuedDate": "2019-02-13",
        "documentValidityDate": "2029-02-13",
        "source": "Manual"
      }
    }
  }
}
```

`documentType`: `1` = Passport (non-biometric), `2` = Biometric Passport, `3` = ID Card.

`source` is always `"Manual"` for this endpoint. (`"Auto"` is reserved for internal records resolved by `/v1/identity-lookup`, but that endpoint does not expose the person object.)

### Check (pre-payment validation)

```
POST /api/v1/check
```

Validate a payment with the upstream provider before charging. Returns a session id you must include in `/v1/pay`. For providers with `requiresVerification: true`, Check also performs the KYC authorization against the upstream provider, so the identity reference is required here (not on Pay alone).

**Headers** — `token` (required).

**Request body**

```json
{
  "providerId": 17667,
  "input1": "+37477777777",
  "input2": null,
  "input3": null,
  "input4": null,
  "identityReference": null
}
```

`input1` through `input4` correspond to the provider's `inputNObject.hint` from `GET /v1/user-providers`. Send `null` for unused inputs. If the provider's `inputNObject.prefix` is non-empty, prepend it to the user's value (e.g. `+374` for Armenian phones).

`identityReference` — required iff the provider's `requiresVerification` is `true`. Send the UUID from `/v1/identity-lookup` or `/v1/identity-manual`. Send `null` for providers that don't require KYC. The reference is validated (exists, not already consumed) and forwarded to the upstream KYC authorization step before the Check call. It is not consumed here: consumption happens on a successful `/v1/pay`.

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": "f7e6d5c4-b3a2-4910-8f7e-6d5c4b3a2910"
  }
}
```

`responseData.data` is the upstream session id. Pass it as `sessionID` to `/v1/pay`.

**Errors**
- `BadRequest` — provider rejected the input (e.g. unknown subscriber); identity required but not provided; identity reference not found; identity reference already consumed.
- `NotFound` — `providerId` doesn't exist or isn't enabled for this user.
- `ServiceUnavailable` — upstream provider or KYC authorization unreachable.

Identity-related error messages you may want to detect here:
- `identity verification required for this provider` — the provider's live `requiresVerification` is `true` but you sent `identityReference: null`.
- `identity reference not found` — the UUID you sent doesn't exist.
- `identity reference already consumed` — the UUID was used by an earlier successful pay; verify again to get a fresh one.

### Pay

```
POST /api/v1/pay
```

Charge the payer and create a transaction. Use only after `/v1/check` returned a session id, and (for verified providers) after `/v1/identity-lookup` or `/v1/identity-manual` returned a reference.

**Headers** — `token` (required).

**Request body**

```json
{
  "providerId": 17667,
  "input1": "+37477777777",
  "input2": null,
  "input3": null,
  "input4": null,
  "amount": 1000,
  "sessionID": "f7e6d5c4-b3a2-4910-8f7e-6d5c4b3a2910",
  "identityReference": null
}
```

- `sessionID` — required, from the matching `/v1/check`.
- `amount` — required, between the provider's `minAmount` and `maxAmount`.
- `identityReference` — required iff the provider's `requiresVerification` is `true`. Send the **same** UUID you used on `/v1/check` for this session. It is validated and linked to the created transaction, then consumed (cannot be reused). Send `null` for providers that don't require KYC.

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": "1m4cP2"
  }
}
```

`responseData.data` is the transaction id (base-36 encoded long). Use it with `/v1/receipt`.

**Errors**
- `BadRequest` — amount out of range; identity required but not provided; identity reference not found; identity reference already consumed; upstream provider rejected the payment.
- `NotFound` — `providerId` not enabled for this user.
- `ServiceUnavailable` — upstream provider unreachable. Safe to retry: EasyPay deduplicates by `sessionID`.

Identity-related error messages you may want to detect:
- `identity verification required for this provider` — the provider's live `requiresVerification` is `true` but you sent `identityReference: null`.
- `identity reference not found` — the UUID you sent doesn't exist.
- `identity reference already consumed` — the UUID was used by an earlier successful pay; verify again to get a fresh one.

### Get balance

```
GET /api/v1/balance
```

Return the agent (user) balance from the upstream EasyPay account. Only available for `User` role; admins receive `Unauthorized`.

**Headers** — `token` (required).

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": {
      "balance": 45348.156
    }
  }
}
```

### List transactions

```
GET /api/v1/user-transactions?page=1&pageSize=50&dataRequest=%7B%7D
```

Paginated list of the authenticated user's transactions.

**Headers** — `token` (required).

**Query parameters**
- `page` — 1-indexed page number.
- `pageSize` — items per page (max 5000 in practice).
- `dataRequest` — URL-encoded JSON describing filters/sorting. Pass `%7B%7D` (encoded `{}`) for no filters.

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": [
      {
        "id": "1m4cP2",
        "providerId": 17667,
        "providerName": "EasyPay Wallet",
        "userId": "1k3ZA",
        "amount": 1000,
        "creationDate": "2026-04-17T08:21:43.757Z",
        "comment": null
      }
    ],
    "page": 1,
    "pageSize": 50,
    "totalCount": 142
  }
}
```

Identity fields (FirstName, LastName, SSN, Document, …) are **never** returned in transaction lists, even for transactions that required KYC. They are AES-encrypted at rest and only ever decrypted internally during the Pay flow.

### List transactions (no filters helper)

```
GET /api/v1/available-user-transactions?page=1&pageSize=50
```

Same shape as `/v1/user-transactions` but without the `dataRequest` query parameter. Equivalent to passing `dataRequest={}`.

### Export transactions

```
GET /api/v1/user-transactions-export?token=<token>&dataRequest=%7B%7D&exportType=Csv
```

Stream the user's transactions as a downloadable file. The token goes in the **query string** here (not the header), so the URL can be used in a download anchor.

**Query parameters**
- `token` — session token.
- `dataRequest` — URL-encoded filter JSON (`%7B%7D` for none).
- `exportType` — `Csv` (other formats are not currently emitted by this endpoint).

**Response** — `text/csv` body with `Content-Disposition: attachment; filename="Transactions.Csv"`.

A sister endpoint `/v1/available-user-transactions-export` accepts the same `token` + `exportType` and skips `dataRequest`; it supports `Csv`, `Pdf`, and `Xlsx`.

### Get receipt

```
GET /api/v1/receipt?transactionId=1m4cP2
```

Return the upstream receipt lines for a single transaction.

**Headers** — `token` (required).

**Query parameters**
- `transactionId` — base-36 transaction id (the `id` returned by `/v1/pay`).

**Response**

```json
{
  "success": true,
  "message": null,
  "responseStatus": "Success",
  "responseData": {
    "data": [
      "EasyPay receipt",
      "Provider: EasyPay Wallet",
      "Amount: 1000.00 AMD",
      "Reference: ABC123"
    ]
  }
}
```

Receipt lines are emitted by the upstream provider; render them as plain text or in a fixed-width font for fidelity.

## Payment identity verification

Some EasyPay providers (typically card-to-card transfers and high-value categories) require KYC: each payment must carry the payer's identity. EasyTransact handles this via two endpoints (`/v1/identity-lookup` for automatic identification, `/v1/identity-manual` for operator-entered data). Both return a single-use `reference` UUID that you pass to `/v1/pay`.

### When verification is required

`GET /v1/user-providers` returns `requiresVerification: true` on providers that need KYC. The flag is read live from EasyPay on every call; do not cache it.

### End-to-end flow

1. **Discover** — `GET /v1/user-providers` and read the chosen provider's `requiresVerification`.
2. **Verify** — call `POST /v1/identity-lookup` (auto-identification by SSN or document number); if it returns `NotFound` or `ServiceUnavailable`, fall back to `POST /v1/identity-manual`. Stash the returned `reference` UUID.
3. **Check** — call `POST /v1/check` with `identityReference` set to the UUID from step 2. EasyTransact authorizes the upstream session with the payer's identity before validating inputs, and returns a `sessionID`.
4. **Pay** — call `POST /v1/pay` with the same `sessionID` and the same `identityReference`. On success the reference is consumed and the transaction is created.

Once consumed, a `reference` cannot be reused. To pay for the same payer again, run step 2 again to mint a new one.

For providers where `requiresVerification` is `false`, skip step 2 and omit (or `null`) `identityReference` on both Check and Pay.

### Identity payload contract

The 8 fields that make up an identity (forwarded to EasyPay's authorize-session call during Check for KYC providers) are:

| Field | Type | Notes |
| --- | --- | --- |
| `firstName` | string | |
| `lastName` | string | |
| `ssn` | string | Armenian Social Security Number. Patterns: `\d{10}` or `Տ\d{3}/\d{5}` or `S\d{3}A\d{5}`. |
| `document` | string | Document number. For passport: `[A-Z]{2}\d{7}` or `\d{9}`. For ID card: `\d{9}`. |
| `documentType` | integer | `1` Passport, `2` Biometric Passport, `3` ID Card. |
| `documentIssuedBy` | string | Issuing authority code (e.g. `"003"`). |
| `documentIssuedDate` | ISO date | `yyyy-MM-dd`. |
| `documentValidityDate` | ISO date | `yyyy-MM-dd`. Must be on or after `documentIssuedDate`. |

These fields are the body of `/v1/identity-manual` (operator submits, response echoes back). For `/v1/identity-lookup`, none of these fields appear on the wire: the resolved identity stays server-side and only the `reference` UUID comes back. EasyTransact stores the full payload AES-256 encrypted; only the Check flow decrypts it (to build the upstream KYC authorization payload). List/export endpoints never return identity fields.

### Recovering from stale references

If `/v1/pay` returns `BadRequest` with one of these messages, re-run identity verification and retry:

- `identity reference already consumed`
- `identity reference not found`

Both indicate the `reference` UUID is no longer usable. Re-mint and retry the same `sessionID` only if your `/v1/check` is still recent (sessions also expire upstream). Safest is to redo Check + Pay together.

## Quick start

A minimal end-to-end integration:

```bash
BASE=https://betransact.easypay.am/api

# 1. Login
TOKEN=$(curl -s -X POST $BASE/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"panda.user","password":"S3cret!Pass"}' \
  | jq -r '.responseData.data.token')

# 2. Pick a provider
PROVIDER_ID=$(curl -s $BASE/v1/user-providers \
  -H "token: $TOKEN" \
  | jq -r '.responseData.data[0].id')

# 3. (KYC providers only) Verify the payer
REFERENCE=$(curl -s -X POST $BASE/v1/identity-lookup \
  -H "token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"lookupType":"Ssn","value":"1234567890"}' \
  | jq -r '.responseData.data.reference')

# 4. Check (pass identityReference for KYC providers, omit or null otherwise)
SESSION_ID=$(curl -s -X POST $BASE/v1/check \
  -H "token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"providerId\":$PROVIDER_ID,\"input1\":\"+37477777777\",\"identityReference\":\"$REFERENCE\"}" \
  | jq -r '.responseData.data')

# 5. Pay (same identityReference as Check)
curl -s -X POST $BASE/v1/pay \
  -H "token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"providerId\":$PROVIDER_ID,\"input1\":\"+37477777777\",\"amount\":1000,\"sessionID\":\"$SESSION_ID\",\"identityReference\":\"$REFERENCE\"}"
```

## Changelog

- **2026-04** — Payment identity verification. `GET /v1/user-providers` now returns `requiresVerification` per provider. New endpoints `POST /v1/identity-lookup` and `POST /v1/identity-manual`. Both `POST /v1/check` and `POST /v1/pay` now accept an optional `identityReference`, required when the selected provider has `requiresVerification: true`. The upstream KYC authorization step runs during Check; Pay consumes the reference on success.

## Support

For issues with the API, file a ticket at the EasyTransact support channel of your contract. For documentation corrections, open a pull request against the [Public-API-Documentations](https://github.com/PandaTechAM/Public-API-Documentations) repository.
