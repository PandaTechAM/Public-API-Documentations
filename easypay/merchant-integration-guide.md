# 1. EasyPay merchant integration guide

Integrate with EasyPay quickly and securely using our HMAC‑protected REST API.  
This guide walks you through authentication, request/response formats, and best‑practice safeguards.

---

## 1.1. Overview

EasyPay exposes three endpoints—**balance inquiry**, **payment**, and **ping**—over HTTPS.  
All requests are authenticated with _HMAC‑SHA‑256_ and must finish within **8 seconds**.

---

## 1.2. Prerequisites

- **API base URL** – supplied after merchant onboarding.
- **HMAC secret key** – delivered out‑of‑band; store only in an encrypted vault.
- **IP allow‑list** – add EasyPay’s IP ranges to reject untrusted traffic.
- **TLS 1.2+** – required for all connections.

---

## 1.3. Authentication & security

### 1.3.1. HMAC workflow

1. **Generate a nonce** – random UUID v4 per request (header `Nonce`).
2. **Concatenate fields** (order defined per endpoint).
3. **Hash** the string with `HMAC‑SHA‑256(secret key, payload)`.
4. **Base64‑encode** the hash.
5. **Send** the result in `Authorization: HMAC <signature>`.

**Optional header**  
`Accept‑Language: hy‑AM | en‑US | ru‑RU` – localizes error/messages.

---

## 1.4. Response Codes & Error Handling

### 1.4.1. Standard HTTP Response Codes

| Code | Description                                       |
| :--- | :------------------------------------------------ |
| 200  | Request succeeded.                                |
| 202  | Request accepted (e.g. order enqueued).           |
| 400  | Invalid request parameters or duplicate requests. |
| 401  | Authentication failed.                            |
| 403  | Insufficient permissions.                         |
| 404  | Resource not found.                               |
| 500  | Server encountered an unexpected error.           |

### 1.4.2. Error Response Structure

All errors are returned in a standard JSON format to help with troubleshooting:

```json
{
  "RequestId": "Unique request ID",
  "TraceId": "Unique trace ID",
  "Instance": "Contextual API information",
  "StatusCode": 400,
  "Type": "Short error descriptor (e.g. BadRequestException)",
  "Errors": {
    "field": "Error message"
  },
  "Message": "General error description"
}
```

**Key points:**

- Use the `RequestId` and `TraceId` when contacting support.
- The `Errors` property gives field-specific details.
- The `Instance` field offers additional context for the operation.

## 1.5. Input formatting & HMAC calculation

### 1.5.1. Technical input list

```text
1: Id                9:  Date                17: BankCard
2: Name              10: Code               18: PayerName
3: PhoneNumber       11: Description        19: Course
4: SocialSecurityNumber
                     12: Email              20: Address
5: Passport          13: TaxCode            21: Count
6: EstateNumber      14: CustomerId         22: LoanId
7: PlateNumber       15: Username           23: InvestmentId
8: RegistrationCertificateNumber
                     16: BankAccount        24: Surname
```

### 1.5.2. Concatenation rule

`Type:Value:TechnicalIndex:` (repeat for each element).

Example payload:

```json
[
  { "Type": 16, "Value": "12345", "TechnicalIndex": 1 },
  { "Type": 3, "Value": "98765", "TechnicalIndex": 2 }
]
```

Concatenated string → `16:12345:1:3:98765:2:`

Full hash input (Balance Inquiry example): `BalanceInquiryId + MerchantServiceIdentifierId + 16:12345:1:3:98765:2: + Nonce`

---

## 1.6. API endpoints

### 1.6.1. Balance inquiry

| Method | Path                   |
| ------ | ---------------------- |
| POST   | `/api/balance-inquiry` |

#### 1.6.1.1. Required headers

```http
Nonce: 69f77f9c-b9b5-43ac-9e6b-516863b8a451
Authorization: HMAC 5EXOva…soTQ=
Accept-Language: en-US
```

#### 1.6.1.2. Request body

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

#### 1.6.1.3. Successful response

```json
{
  "Debt": 500.75,
  "Properties": [{ "Key": "Debt for 09.24", "Value": "500.75" }]
}
```

_Response HMAC_ → `HMAC(Debt + Properties.Values + Request Nonce)`.

---

### 1.6.2. Make payment

| Method | Path            |
| ------ | --------------- |
| POST   | `/api/payments` |

#### 1.6.2.1. Required headers

```http
Nonce: 04ba5422-e021-4dfb-a716-bdd1440e91b4
Authorization: HMAC 5EXOva…soTQ=
Accept-Language: ru-RU
```

#### 1.6.2.2. Request body

```json
{
  "OrderId": "434dd03f-ede8-4e55-b71f-f81cb4120cba",
  "Amount": 500.75,
  "BalanceInquiryId": 12345, // optional
  "MerchantServiceIdentifierId": 67890,
  "Inputs": [
    { "TechnicalIndex": 1, "Type": 1, "Value": "12345" },
    { "TechnicalIndex": 2, "Type": 3, "Value": "98765" }
  ]
}
```

> **Important:** store `OrderId` with a **unique index** to avoid duplicate payments.  
> On repeat `OrderId`, respond with the original `PaymentId`.

#### 1.6.2.3. Successful response

```json
{ "PaymentId": "xyz-12345" }
```

_Response HMAC_ → `HMAC(PaymentId + Request Nonce)`.

---

### 1.6.3. Ping

| Method | Path        |
| ------ | ----------- |
| GET    | `/api/ping` |

Returns service health.

```json
{ "Status": "OK" }
```

---

## 1.7. Additional security best practices

- **Replay defense** – cache recent HMAC signatures; reject duplicates.
- **Timeout handling** – abort all downstream work after EasyPay’s **8 s** limit.
- **Idempotency** – use the unique `OrderId` strategy above for safe retries.
- **Audit logging** – log `RequestId`/`TraceId`, status, and timing for each call.

---

## 1.8. Next steps

1. Generate test credentials in the EasyPay sandbox.
2. Implement the HMAC helper in your preferred language.
3. Run integration‑test scenarios: balance‑only, payment, timeout, duplicate order.
4. Move to production after passing all automated checks.

For questions, contact your EasyPay integration manager.
