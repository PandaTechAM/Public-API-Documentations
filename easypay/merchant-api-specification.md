# EasyPay Merchant API Specification

This document specifies the REST API that merchants must implement to integrate with EasyPay. In this integration model, **EasyPay calls your API** — you provide the endpoints, and EasyPay sends requests to them.

---

## Introduction

When a merchant service is configured for direct integration, EasyPay initiates HTTP calls to your server for:

- **Balance inquiry** — check a customer's outstanding debt or account balance
- **Payment** — process a payment on behalf of the customer
- **Refund** — reverse a previously completed payment
- **Ping** — verify your service is reachable

Your API must validate all incoming requests using HMAC-SHA256, process them within **8 seconds**, and sign all responses so EasyPay can verify their authenticity.

---

## Prerequisites

| Item                | Description                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------- |
| **Base URL**        | The root URL of your API (e.g., `https://api.yourcompany.com`). Provided during merchant onboarding. |
| **HMAC Secret Key** | Shared secret for request/response signing. Delivered out-of-band. Store securely.                   |
| **TLS 1.2+**        | Required for all connections.                                                                        |
| **IP Allowlist**    | Restrict inbound traffic to EasyPay's IP ranges (provided during onboarding).                        |

---

## Authentication

EasyPay signs every request with HMAC-SHA256. Your server must validate the signature before processing.

### Request headers from EasyPay

```
Nonce: <UUIDv4>
Authorization: HMAC <base64_signature>
Accept-Language: en-US | hy-AM | ru-RU
```

> **Note:** The merchant API uses `HMAC <signature>` without an API key prefix. EasyPay is the only caller, so caller identification is not needed.

### How to validate incoming requests

1. Extract the `Nonce` and `Authorization` headers.
2. Strip the `HMAC ` prefix from the Authorization value to get the received signature.
3. Concatenate the endpoint-specific parameters (documented per endpoint below).
4. Compute `HMAC-SHA256(your_secret_key, concatenated_string)`.
5. Base64-encode the result.
6. Compare your computed signature with the received signature. Reject the request with `401` if they don't match.

### How to sign responses

EasyPay validates the `Authorization` header on your responses. You must sign every successful response:

1. Concatenate the response-specific parameters (documented per endpoint below).
2. Compute `HMAC-SHA256(your_secret_key, concatenated_string)`.
3. Base64-encode and return as: `Authorization: HMAC <base64_signature>`.

> If EasyPay cannot verify your response signature, the transaction is treated as failed.

### Input concatenation

When HMAC parameters include `Inputs`, concatenate each input as `Type:Value:TechnicalIndex:` (note the trailing colon per input).

**Example:** Given inputs `[{Type: 16, Value: "12345", TechnicalIndex: 1}, {Type: 3, Value: "98765", TechnicalIndex: 2}]`, the concatenated string is `16:12345:1:3:98765:2:`.

---

## Error Handling

When your API encounters an error, return the appropriate HTTP status code. EasyPay recognizes:

| Code | When to use                                           |
| ---- | ----------------------------------------------------- |
| 200  | Request processed successfully.                       |
| 400  | Invalid request (bad input, business rule violation). |
| 401  | HMAC signature validation failed.                     |
| 404  | Account or resource not found.                        |
| 500  | Unexpected server error.                              |

For `400` and `500` responses, include a JSON body with a human-readable message:

```json
{
  "Message": "Account not found for the provided identifier"
}
```

EasyPay will forward relevant error messages to the end user (localized via `Accept-Language`).

---

## Endpoints

You must implement the following endpoints at your base URL.

### Ping

EasyPay calls this endpoint periodically to verify your service is healthy.

```
GET /api/ping
```

**Request HMAC:** None required.

**Response:**

```json
{
  "Status": "OK"
}
```

Return `200` if your service is operational. Any non-`200` response triggers an alert on EasyPay's monitoring dashboard.

---

### Balance Inquiry

Called when a customer wants to check their balance or outstanding debt before making a payment.

```
POST /api/balance-inquiry
```

**Request headers:**

```
Nonce: 69f77f9c-b9b5-43ac-9e6b-516863b8a451
Authorization: HMAC 5EXOva...soTQ=
Accept-Language: en-US
```

**Request HMAC parameters:** `BalanceInquiryId + MerchantServiceIdentifierId + Inputs + Nonce`

**Request body:**

```json
{
  "BalanceInquiryId": 12345,
  "MerchantServiceIdentifierId": 67890,
  "Inputs": [
    { "TechnicalIndex": 1, "Type": 1, "Value": "12345" },
    { "TechnicalIndex": 2, "Type": 3, "Value": "98765" }
  ]
}
```

**Successful response (200):**

```json
{
  "Amount": 500.75,
  "Properties": [{ "Key": "Debt for 09.24", "Value": "500.75" }]
}
```

**Response HMAC parameters:** `Amount + Properties[].Value (concatenated in order) + Request Nonce`

**Response header:**

```
Authorization: HMAC <base64_signature>
```

**Implementation notes:**

- `Amount` is the total outstanding balance. Positive means the customer owes money. Zero means no debt.
- `Properties` provide additional context displayed to the customer (due dates, account details, etc.). Each property's `Value` is included in the response HMAC in order.
- If the customer identifier is not found, return `404`.
- The `BalanceInquiryId` is EasyPay's internal reference — store it for traceability but do not use it for business logic.

---

### Payment

Called when a customer confirms a payment. This endpoint must process the payment and return a unique reference ID.

```
POST /api/payments
```

**Request headers:**

```
Nonce: 04ba5422-e021-4dfb-a716-bdd1440e91b4
Authorization: HMAC 5EXOva...soTQ=
Accept-Language: ru-RU
```

**Request HMAC parameters:** `OrderId + Amount + BalanceInquiryId + MerchantServiceIdentifierId + Inputs + Nonce`

> If `BalanceInquiryId` is `null`, use an empty string in the HMAC calculation.

**Request body:**

```json
{
  "OrderId": "434dd03f-ede8-4e55-b71f-f81cb4120cba",
  "Amount": 500.75,
  "BalanceInquiryId": 12345,
  "MerchantServiceIdentifierId": 67890,
  "Inputs": [
    { "TechnicalIndex": 1, "Type": 1, "Value": "12345" },
    { "TechnicalIndex": 2, "Type": 3, "Value": "98765" }
  ]
}
```

**Successful response (200):**

```json
{
  "PaymentId": "xyz-12345"
}
```

**Response HMAC parameters:** `PaymentId + Request Nonce`

**Response header:**

```
Authorization: HMAC <base64_signature>
```

**Critical implementation requirements:**

- **Idempotency:** Store the `OrderId` with a **unique index**. If EasyPay sends the same `OrderId` again (e.g., due to a network retry), return the original `PaymentId` without processing a duplicate payment.
- **Atomicity:** Either fully process the payment or reject it. Do not leave partial state.
- **Timeout:** EasyPay waits up to **8 seconds**. If your processing takes longer, respond immediately with an acknowledgment and process asynchronously, or return an error.
- `PaymentId` is your internal reference for this payment. EasyPay stores it for reconciliation.

---

### Refund

Called when a payment needs to be reversed (customer dispute, cancellation, or error correction).

```
POST /api/refunds
```

**Request headers:**

```
Nonce: a1b2c3d4-e5f6-7890-abcd-ef1234567890
Authorization: HMAC 8FGHij...klMN=
Accept-Language: en-US
```

**Request HMAC parameters:** `OrderId + Nonce`

**Request body:**

```json
{
  "OrderId": "434dd03f-ede8-4e55-b71f-f81cb4120cba"
}
```

**Successful response (200):**

```json
{
  "RefundId": "ref-67890"
}
```

**Response HMAC parameters:** `RefundId + Request Nonce`

**Response header:**

```
Authorization: HMAC <base64_signature>
```

**Implementation requirements:**

- **Idempotency:** If the same `OrderId` is submitted for refund multiple times, return the original `RefundId` without processing a duplicate refund.
- **Validation:** Verify that the `OrderId` corresponds to a completed payment on your side. Return `404` if the payment is not found, or `400` if the payment was already refunded.
- `RefundId` is your internal reference for the refund. EasyPay stores it for reconciliation.

---

## Security Best Practices

- **IP allowlisting** — Only accept requests from EasyPay's IP ranges. Reject all other inbound traffic to your integration endpoints.
- **Replay protection** — Cache recent `Nonce` values and reject duplicates within a reasonable window (e.g., 5 minutes).
- **Timeout handling** — If your downstream systems are slow, fail fast rather than letting EasyPay's 8-second timeout expire. Return a clear error so the operation can be retried.
- **Idempotency** — Both `/api/payments` and `/api/refunds` must handle duplicate requests safely. Use the `OrderId` as the idempotency key.
- **Audit logging** — Log every request with the `Nonce`, `Authorization` header, request body, and your response. This is critical for dispute resolution.
- **Credential rotation** — If you suspect the HMAC key has been compromised, contact your EasyPay integration manager immediately for key rotation.

---

## Testing

1. Request sandbox credentials from your EasyPay integration manager.
2. Implement all four endpoints (ping, balance inquiry, payment, refund).
3. Run integration tests covering:
   - Successful balance inquiry with properties
   - Successful payment with HMAC response validation
   - Duplicate `OrderId` handling (must return original `PaymentId`, not process twice)
   - Refund of a completed payment
   - Refund of a non-existent order (expect `404`)
   - Invalid HMAC signature (expect `401`)
   - Timeout behavior (response within 8 seconds)
4. Pass EasyPay's automated certification checks.
5. Coordinate go-live with your integration manager.

---

## Support

For integration support, contact your EasyPay integration manager. When reporting issues, include the `Nonce` from the request and your server-side logs.

---

_Last updated: 16-03-2026_
