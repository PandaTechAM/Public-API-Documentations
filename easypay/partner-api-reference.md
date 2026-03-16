# EasyPay Partner API Reference

Integrate with EasyPay to offer merchant payment services through your platform. This reference covers authentication, available endpoints, and implementation guidance for a secure and reliable integration.

---

## Introduction

EasyPay provides a REST API that allows payment partners (kiosk operators, payment aggregators, financial institutions) to:

- Query available merchant services and their configurations
- Check customer balances and outstanding debts
- Calculate customer commissions before payment
- Create, monitor, and cancel payment orders

All communication happens over **HTTPS** with **HMAC-SHA256** authentication. Every request must include a valid signature and a unique nonce to prevent replay attacks.

---

## Prerequisites

Before you begin, ensure you have:

| Item                | Description                                                                   |
| ------------------- | ----------------------------------------------------------------------------- |
| **API Base URL**    | Provided during onboarding (e.g., `https://api.easypay.am`).                  |
| **API Key**         | Identifies your application. Included in the `Authorization` header.          |
| **HMAC Secret Key** | Used to compute request signatures. Store in a secure vault — never hardcode. |
| **TLS 1.2+**        | Required for all connections.                                                 |

**Optional header:** `Accept-Language: en-US | hy-AM | ru-RU` — localizes error messages and display text.

---

## Authentication

Every request must include two headers:

```
Nonce: <UUIDv4>
Authorization: HMAC <API_KEY>:<BASE64_SIGNATURE>
```

### How to compute the signature

1. Generate a random **UUIDv4** for the `Nonce` header.
2. Concatenate the endpoint-specific parameters (documented per endpoint below).
3. Compute `HMAC-SHA256(secret_key, concatenated_string)`.
4. Base64-encode the result.
5. Format the header: `HMAC <your_api_key>:<base64_signature>`.

**Example:**

```
API Key: ffa5be93-35e6-4186-8b84-44882d6dbb30
HMAC Key: your-secret-key
Parameters: ["1234567890", "16:12345:1:3:98765:2:", "deadd8fc-58fc-4f12-8a57-808711f6319d"]
Concatenated: "123456789016:12345:1:3:98765:2:deadd8fc-58fc-4f12-8a57-808711f6319d"
Base64 HMAC: iVGljbFol+xfKGW/uw3B9wxP5timuV5YYyzYkL3cicU=
Header: HMAC ffa5be93-35e6-4186-8b84-44882d6dbb30:iVGljbFol+xfKGW/uw3B9wxP5timuV5YYyzYkL3cicU=
```

> You can verify your implementation using the HMAC calculator at `https://easypay.am/hmac-generator`.

### Input concatenation for HMAC

When an endpoint requires `Inputs` in the HMAC, concatenate each input as `IdentifierDetailType:Value:TechnicalIndex:` (note the trailing colon).

**Example:** Given inputs `[{Type: 16, Value: "12345", TechnicalIndex: 1}, {Type: 3, Value: "98765", TechnicalIndex: 2}]`, the concatenated string is `16:12345:1:3:98765:2:`.

---

## Identifier Detail Types

Identifier types define what information a merchant service requires to identify a payment account.

| Code | Name                            |
| ---- | ------------------------------- |
| 1    | Id                              |
| 2    | Name                            |
| 3    | Phone Number                    |
| 4    | Social Security Number          |
| 5    | Passport                        |
| 6    | Estate Number                   |
| 7    | Plate Number                    |
| 8    | Registration Certificate Number |
| 9    | Date                            |
| 10   | Code                            |
| 11   | Description                     |
| 12   | Email                           |
| 13   | Tax Code                        |
| 14   | Customer Id                     |
| 15   | Username                        |
| 16   | Bank Account                    |
| 17   | Bank Card                       |
| 18   | Payer Name                      |
| 19   | Course                          |
| 20   | Address                         |
| 21   | Count                           |
| 22   | Loan Id                         |
| 23   | Investment Id                   |
| 24   | Surname                         |
| 25   | Full Name                       |
| 26   | Technical Id                    |
| 27   | Unique Document Id              |

---

## Order Statuses

| Code | Name        | Description                                                                                  |
| ---- | ----------- | -------------------------------------------------------------------------------------------- |
| 1    | Enqueued    | Order received and queued for processing. Poll the Order Details endpoint to track progress. |
| 2    | Initialized | Order received and being processed synchronously.                                            |
| 3    | Charged     | Payment successfully collected from the customer.                                            |
| 5    | Canceled    | Order was canceled by the client or an administrator.                                        |
| 6    | Success     | Order completed successfully. This is a terminal state.                                      |
| 7    | Rejected    | Order rejected due to a validation error or merchant-side problem. This is a terminal state. |

---

## Error Handling

### HTTP Status Codes

| Code | Meaning                                                            |
| ---- | ------------------------------------------------------------------ |
| 200  | Request succeeded.                                                 |
| 202  | Request accepted (order enqueued for async processing).            |
| 400  | Invalid parameters, duplicate request, or business rule violation. |
| 401  | Authentication failed (invalid API key or HMAC signature).         |
| 403  | Insufficient permissions.                                          |
| 404  | Resource not found.                                                |
| 500  | Unexpected server error.                                           |

### Error Response Body

```json
{
  "RequestId": "unique-request-id",
  "TraceId": "unique-trace-id",
  "Instance": "contextual-api-info",
  "StatusCode": 400,
  "Type": "BadRequestException",
  "Errors": {
    "Amount": "Amount must be greater than zero"
  },
  "Message": "One or more validation errors occurred"
}
```

When contacting support, always include the `RequestId` and `TraceId` from the error response.

---

## Endpoints

All endpoints use the base URL provided during onboarding. Every request must include `Nonce`, `Authorization`, and optionally `Accept-Language` headers.

### Ping

Verify connectivity to the EasyPay API.

```
GET /above-board/ping
```

**Response:**

```
"pong"
```

No authentication required.

---

### Get Terminal Information

Retrieve your terminal configuration including currency and payment limits.

```
GET /api/external/v1/terminals
```

**HMAC parameters:** `Nonce`

**Response:**

```json
{
  "id": 123,
  "name": "Terminal Name",
  "currencyCode": "USD",
  "dailyMaxPaymentSum": 11000,
  "dailyCurrentPaymentSum": 10,
  "dailyMaxPaymentCount": 50,
  "dailyCurrentPaymentCount": 2,
  "monthlyMaxPaymentSum": null,
  "monthlyCurrentPaymentSum": null,
  "monthlyMaxPaymentCount": null,
  "monthlyCurrentPaymentCount": null
}
```

> `null` limits mean no restriction is configured for that period.

---

### Get Agent Balance

Check your current deposit balance. Payments are deducted from this balance.

```
GET /api/external/v1/agents/balance
```

**HMAC parameters:** `Nonce`

**Response:**

```json
{
  "balance": 978648.14
}
```

> When the balance reaches zero (or exceeds the credit limit), your terminal cannot accept new payments.

---

### List Merchant Services

Retrieve all merchant services available to your terminal.

```
GET /api/external/v1/merchant-services
```

**HMAC parameters:** `Nonce`

**Response:**

```json
[
  {
    "id": 123,
    "name": "Mobile Top-up",
    "description": "Recharge mobile balance"
  }
]
```

---

### Get Merchant Service Details

Fetch configuration, required identifiers, and payment constraints for a specific merchant service.

```
GET /api/external/v1/merchant-services/{merchantServiceId}
```

**HMAC parameters:** `merchantServiceId + Nonce`

**Response:**

```json
{
  "config": {
    "minSoloPaymentAmount": 10,
    "maxSoloPaymentAmount": 500,
    "allowedLessThanBalanceInquiry": true,
    "allowedMoreThanBalanceInquiry": true
  },
  "identifiers": [
    {
      "id": 1,
      "enabled": true,
      "details": [
        {
          "technicalIndex": 1,
          "identifierDetailType": 3,
          "required": true,
          "name": "Phone number",
          "prefix": "0",
          "placeHolder": "Enter phone number",
          "hint": "Type only +374 prefixed phone number",
          "defaultValue": "077123456",
          "regex": "^\\+374[0-9]{8}$",
          "options": ["+37491234567", "+37498123456"],
          "inputBlackList": ["+37490000000"]
        }
      ]
    }
  ]
}
```

**Implementation notes:**

- When `options` is provided, restrict user input to those values only.
- When `inputBlackList` is provided, reject any matching input.
- Use `regex` for client-side validation before making API calls.
- `config.allowedLessThanBalanceInquiry` / `allowedMoreThanBalanceInquiry` control whether the customer can pay less or more than the balance inquiry amount.

---

### Balance Inquiry

Check a customer's outstanding balance or debt for a merchant service. This may require a two-step process when the initial input doesn't uniquely identify the account.

```
POST /api/external/v1/merchant-services/{merchantServiceId}/balance-inquiry
```

**HMAC parameters:** `merchantServiceId + merchantServiceIdentifierId + Inputs + Nonce`

#### Step 1: Initial Inquiry

**Request:**

```json
{
  "merchantServiceIdentifierId": 1,
  "inputs": [
    {
      "identifierDetailType": 3,
      "technicalIndex": 0,
      "value": "+37491234567"
    }
  ]
}
```

**Response (multiple results — disambiguation needed):**

```json
{
  "balanceInquiryId": 12345,
  "data": [
    {
      "amount": 150,
      "inputs": [
        { "identifierDetailType": 3, "technicalIndex": 0, "value": "+37491234567" },
        { "identifierDetailType": 10, "technicalIndex": 1, "value": "12345678" }
      ],
      "properties": [{ "key": "Due Date", "value": "2025-02-10" }]
    },
    {
      "amount": 240,
      "inputs": [
        { "identifierDetailType": 3, "technicalIndex": 0, "value": "+37491234567" },
        { "identifierDetailType": 10, "technicalIndex": 1, "value": "478956315" }
      ],
      "properties": [{ "key": "Due Date", "value": "2025-03-10" }]
    }
  ]
}
```

#### Step 2: Detailed Inquiry (if needed)

If Step 1 returns multiple results with additional inputs, let the customer select the correct account and re-submit with the complete input set.

**Request:**

```json
{
  "merchantServiceIdentifierId": 1,
  "inputs": [
    { "identifierDetailType": 3, "technicalIndex": 0, "value": "+37491234567" },
    { "identifierDetailType": 10, "technicalIndex": 1, "value": "12345678" }
  ]
}
```

**Response (single result with full details):**

```json
{
  "balanceInquiryId": 12346,
  "minAmount": 100,
  "maxAmount": 500,
  "data": [
    {
      "amount": 150,
      "inputs": [
        { "identifierDetailType": 3, "technicalIndex": 0, "value": "+37491234567" },
        { "identifierDetailType": 10, "technicalIndex": 1, "value": "12345678" }
      ],
      "properties": [
        { "key": "Due Date", "value": "2025-02-10" },
        { "key": "Account Type", "value": "Savings" },
        { "key": "Overdue Amount", "value": "50" }
      ]
    }
  ]
}
```

> The second step is uncommon. Most merchant services resolve to a single account on the first call. When `minAmount` / `maxAmount` are returned, enforce these limits on the payment amount.

---

### Check Customer Commission

Calculate the commission fee the customer will pay before creating an order.

```
POST /api/external/v1/orders/commissions
```

**HMAC parameters:** `merchantServiceId + amount + Nonce`

**Request:**

```json
{
  "merchantServiceId": 123,
  "amount": 2450
}
```

**Response:**

```json
{
  "commission": 15.41
}
```

> The commission is automatically added on top of the payment amount during order creation. Display it to the customer before confirming.

---

### Create Payment Order

Submit a payment order. The inputs must match the latest balance inquiry results exactly.

```
POST /api/external/v1/orders
```

**HMAC parameters:** `merchantServiceId + merchantServiceIdentifierId + amount + Inputs + Nonce`

**Request:**

```json
{
  "id": "unique-id-from-your-system",
  "merchantServiceId": 123,
  "merchantServiceIdentifierId": 1,
  "balanceInquiryId": 12346,
  "amount": 2540,
  "inputs": [
    { "identifierDetailType": 3, "technicalIndex": 0, "value": "+37491234567" },
    { "identifierDetailType": 10, "technicalIndex": 1, "value": "12345678" }
  ]
}
```

**Response:**

```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

**Important notes:**

- The `id` field is **your** unique order identifier. EasyPay enforces uniqueness — if the same ID is submitted again, the existing order's ID is returned with `200 OK` (no duplicate payment is created). This makes the endpoint safe to retry on network failures.
- The `amount` excludes commission — it is added automatically.
- `balanceInquiryId` is optional but recommended. Pass it when a balance inquiry was performed.
- Response may return `200 OK` (processed synchronously or duplicate) or `202 Accepted` (enqueued). For `202`, poll the Order Details endpoint to determine the final outcome.

---

### Get Order Details

Retrieve the current status and details of a payment order.

```
GET /api/external/v1/orders/{orderId}
```

**HMAC parameters:** `Nonce`

**Response:**

```json
{
  "finTechName": "EasyPay",
  "finTechTaxCode": "12345678",
  "finTechPhoneNumber": "+37491234567",
  "finTechAddress": "Yerevan, Armenia",
  "merchantServiceName": "Mobile Top-up",
  "date": "2025-04-04T07:52:47.666Z",
  "inputs": [{ "technicalIndex": 0, "type": 1, "value": "1", "nameKey": "ID" }],
  "amount": 1000,
  "customerCommission": 100,
  "alphabeticIsoCode": "AMD",
  "orderId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": 6,
  "cancelable": true
}
```

> Check the `cancelable` field before attempting to cancel an order.

---

### Cancel Order

Cancel a payment order. Only orders where `cancelable` is `true` in the Order Details response can be canceled.

```
DELETE /api/external/v1/orders/{orderId}
```

**HMAC parameters:** `orderId + Nonce`

Returns `200 OK` on success. Refer to the error handling section for failure scenarios.

---

## Implementation Checklist

- [ ] Store the API Key and HMAC Key in a secure vault. Never hardcode credentials.
- [ ] Generate a fresh UUIDv4 nonce for every request.
- [ ] Enforce unique order IDs on your side — index the `id` field and prevent duplicate submissions.
- [ ] Handle both `200` and `202` responses for order creation. Poll Order Details for async orders.
- [ ] Display the customer commission (from the commission check endpoint) before confirming payment.
- [ ] Respect `minAmount` / `maxAmount` from balance inquiry responses.
- [ ] Implement timeout handling — EasyPay has an **8-second** processing limit.
- [ ] Log `RequestId` and `TraceId` from error responses for support escalation.

---

## Support

For integration support or to report issues, contact your EasyPay integration manager. Always include `RequestId` and `TraceId` when reporting errors.

---

_Last updated: 16-03-2026_
