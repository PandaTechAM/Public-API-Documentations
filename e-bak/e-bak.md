# e-bak API Documentation

## Overview

The **e-bak API** provides secure, programmatic access to condominium-fee debt retrieval and payment services. Through these REST endpoints you can:

- Query outstanding debts for a given estate or owner.
- Calculate commissions before processing payments.
- Repay debts in bulk or individually with idempotent requests.
- Retrieve a previously processed payment by its outer payment ID.
- Discover reference data: cities, districts, buildings, condominium associations, and estates.

## Environments

| Environment | Base URL                       | Swagger                          | Scalar UI                       |
| ----------- | ------------------------------ | -------------------------------- | ------------------------------- |
| **Test**    | `https://qabe-ca.pandatech.it` | `{Base URL}/swagger/integration` | `{Base URL}/scalar/integration` |
| **Prod**    | `https://be.e-bak.am`          | Not exposed                      | Not exposed                     |

> **Tip:** Complete all integration testing against the Test environment before switching to Production.

## Language Support

Include an `Accept-Language` header with every request. Supported values:

| Code    | Language               |
| ------- | ---------------------- |
| `hy-AM` | Armenian **(default)** |
| `en-US` | English (US)           |
| `ru-RU` | Russian                |

If the header is missing or contains an unsupported value, the API defaults to Armenian (`hy-AM`).

---

## Authentication

All endpoints (except [Ping](#ping)) require a valid access token.

### Obtaining a Token

Send your credentials to the [Login](#login) endpoint. A successful response returns an `accessToken` and its `expirationDate`.

### Using the Token

Include the token in the `Authorization` header of every subsequent request:

```
Authorization: {accessToken}
```

### Token Lifecycle

- **Auto-refresh:** Each successful API call silently extends the token's expiration on the server side. Active usage keeps the session alive without any action from your side.
- **Expiration:** If you receive a `401 Unauthorized` response, your token has expired. Call the [Login](#login) endpoint again to obtain a fresh token, then retry.

> **Important: Do NOT call login before every API request.** This is a common integration mistake. The login endpoint uses Argon2ID password hashing, which adds ~300ms of overhead per call. Instead, cache your token and implement a middleware/pipeline that intercepts `401` responses, re-authenticates once, and replays the failed request. This keeps your calls fast and avoids unnecessary load on the auth service.

---

## Response Codes and Error Handling

### Standard HTTP Response Codes

| Code | Meaning                                           |
| :--- | :------------------------------------------------ |
| 200  | Request succeeded.                                |
| 202  | Request accepted (e.g., order enqueued).          |
| 400  | Invalid request parameters or duplicate requests. |
| 401  | Authentication failed or token expired.           |
| 403  | Insufficient permissions.                         |
| 404  | Resource not found.                               |
| 500  | Server encountered an unexpected error.           |

### Error Response Structure

All errors follow a consistent JSON format:

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

| Field       | Purpose                                                        |
| ----------- | -------------------------------------------------------------- |
| `RequestId` | Unique identifier for the request. Share with support.         |
| `TraceId`   | Trace identifier for distributed tracing. Share with support.  |
| `Instance`  | Additional context about the operation.                        |
| `Errors`    | Field-level validation errors (may be `null`).                 |
| `Message`   | Human-readable description of the error.                       |

> **Tip:** Always log `RequestId` and `TraceId`. When contacting support, provide both values for faster resolution.

---

## Common Models

These enums are referenced across multiple endpoints.

> **Note:** All enums are serialized as their **string names** in JSON responses (e.g., `"Apartment"`, `"Yes"`, `"Ameriabank"`), not as numeric values. The numeric values listed below are the underlying enum values for reference only.

### Estate Types

| Value | Name             | Armenian           |
| ----- | ---------------- | ------------------ |
| 0     | Apartment        | Բնակարան           |
| 1     | CommercialArea   | Կոմերցիոն տարածք  |
| 2     | ParkingArea      | Կայանման տարածք    |
| 3     | Basement         | Նկուղ              |
| 4     | House            | Տուն               |
| 5     | Other            | Այլ                |
| 6     | Elevator         | Վերելակ              |
| 7     | SolarStation     | Արևային կայան       |

### Check Commission

| Value | Name | Meaning                                                    |
| ----- | ---- | ---------------------------------------------------------- |
| 1     | Yes  | Commission applies. Calculate before payment.              |
| 2     | No   | No commission. Skip the commission calculation step.       |

### Bank

| Value | Name               |
| ----- | ------------------ |
| 0     | AcbaBank           |
| 1     | AraratBank         |
| 2     | Ameriabank         |
| 3     | AmioBank           |
| 4     | IDBank             |
| 5     | Ardshinbank        |
| 6     | Armswissbank       |
| 7     | Artsakhbank        |
| 8     | BiblosBankArmenia  |
| 10    | Evocabank          |
| 11    | Inecobank          |
| 12    | ConverseBank       |
| 13    | Armeconombank      |
| 14    | MellatBank         |
| 15    | Unibank            |
| 16    | VTBArmeniaBank     |
| 17    | FastBank           |

> **Note:** Value `9` is intentionally missing from the Bank enum.

---

## API Endpoints

### Ping

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/above-board/ping` |
| **Auth** | Not required |

Health check endpoint. Returns `"pong"` if the service is reachable.

---

### Login

| | |
| --- | --- |
| **Method** | `POST` |
| **Path** | `/api/v1/integration/login` |
| **Auth** | Not required |

Authenticates a user and returns an access token.

**Request:**

```json
{
  "login": "easypay@easypay.am",
  "password": "Qwerty123!"
}
```

**Response (200):**

```json
{
  "accessToken": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "expirationDate": "2023-12-21T08:34:20.996Z"
}
```

> **Note:** Store the token securely. Never expose it in client-side code or logs. Cache it and reuse until it expires (see [Token Lifecycle](#token-lifecycle)).

---

### Debt Retrieval by Estate

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v4/integration/debts/{estateId}` |
| **Auth** | Required |

Retrieves all outstanding debts for a specific estate.

**Path Parameters:**

| Parameter  | Type    | Description            |
| ---------- | ------- | ---------------------- |
| `estateId` | integer | The ID of the estate.  |

**Response (200):**

```json
{
  "partnerId": 8,
  "partnerName": "Sample Condominium LLC",
  "estateId": 1000270,
  "estateType": "Other",
  "estateAddress": "Tumanyan 33, 1",
  "primaryEstateOwnerFullName": "John Doe",
  "checkCommission": "Yes",
  "balance": -4900,
  "bank": "Ameriabank",
  "bankAccount": "1500016548794561",
  "debts": [
    {
      "debtId": 75,
      "balance": -4900,
      "date": "2024-04-01T00:00:00Z"
    }
  ]
}
```

**Response fields:**

| Field                        | Type     | Description                                                  |
| ---------------------------- | -------- | ------------------------------------------------------------ |
| `partnerId`                  | integer  | Condominium association ID.                                  |
| `partnerName`                | string   | Condominium association name.                                |
| `estateId`                   | integer  | Estate identifier.                                           |
| `estateType`                 | string   | See [Estate Types](#estate-types).                           |
| `estateAddress`              | string   | Estate address.                                              |
| `primaryEstateOwnerFullName` | string?  | Owner's full name. **Can be `null`.**                        |
| `checkCommission`            | string   | See [Check Commission](#check-commission).                   |
| `balance`                    | decimal  | Total outstanding balance. Negative value indicates debt.    |
| `bank`                       | string   | Bank name. See [Bank](#bank) enum.                           |
| `bankAccount`                | string   | Bank account number.                                         |
| `debts`                      | array    | List of individual debt records.                             |
| `debts[].debtId`             | integer  | Individual debt identifier.                                  |
| `debts[].balance`            | decimal  | Debt amount. Negative value indicates amount owed.           |
| `debts[].date`               | datetime | Date of the debt record (UTC).                               |

---

### Debt Retrieval by Owner

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v4/integration/debts/owner/{uniqueDocumentId}` |
| **Auth** | Required |

Retrieves all outstanding debts across all estates owned by a person, identified by SSN or Tax Code.

**Path Parameters:**

| Parameter          | Type   | Description                |
| ------------------ | ------ | -------------------------- |
| `uniqueDocumentId` | string | Owner's SSN or Tax Code.   |

**Response (200):**

```json
{
  "estates": [
    {
      "partnerId": 8,
      "partnerName": "Sample Condominium LLC",
      "estateId": 1000270,
      "estateType": "Basement",
      "estateAddress": "Tumanyan 1, 1",
      "primaryEstateOwnerFullName": "John Doe",
      "checkCommission": "Yes",
      "balance": -5000,
      "bank": "Ameriabank",
      "bankAccount": "1500016548794561",
      "debts": [
        {
          "debtId": 1,
          "balance": -4900,
          "date": "2024-04-01T00:00:00Z"
        },
        {
          "debtId": 20,
          "balance": -100,
          "date": "2024-04-01T00:00:00Z"
        }
      ]
    },
    {
      "partnerId": 8,
      "partnerName": "Sample Condominium LLC",
      "estateId": 1000271,
      "estateType": "Apartment",
      "estateAddress": "Tumanyan 2, 1",
      "primaryEstateOwnerFullName": "John Doe",
      "checkCommission": "Yes",
      "balance": -15000,
      "bank": "Ameriabank",
      "bankAccount": "1500016548794561",
      "debts": [
        {
          "debtId": 7,
          "balance": -8000,
          "date": "2024-04-01T00:00:00Z"
        },
        {
          "debtId": 5,
          "balance": -7000,
          "date": "2024-04-01T00:00:00Z"
        }
      ]
    },
    {
      "partnerId": 8,
      "partnerName": "Sample Condominium LLC",
      "estateId": 1000272,
      "estateType": "House",
      "estateAddress": "Tumanyan 2, 3",
      "primaryEstateOwnerFullName": "John Doe",
      "checkCommission": "No",
      "balance": 0,
      "bank": "AcbaBank",
      "bankAccount": "2200016548794561",
      "debts": []
    }
  ]
}
```

Each item in the `estates` array follows the same structure as the [Debt Retrieval by Estate](#debt-retrieval-by-estate) response.

> **Note:** `primaryEstateOwnerFullName` may be `null` if the owner information is not available.

---

### Debt Commission Calculation

| | |
| --- | --- |
| **Method** | `POST` |
| **Path** | `/api/v2/integration/debts/commission` |
| **Auth** | Required |

Calculates the total commission for the specified debts before processing payment. This is an **informational** endpoint; the backend does not enforce that you call it before paying. However, the payment endpoint validates that the submitted `commission` value is correct, so use this endpoint to obtain the right amount.

> **Note on commission:** The commission belongs to the integrator (you). e-bak configures and calculates it on your behalf because commission rates vary by condominium association, and maintaining the full list on your side would be impractical. This endpoint gives you the correct value without needing to know the per-condominium rates.

**Request:**

```json
{
  "debtCommissionRequestModels": [
    {
      "estateId": 1000182,
      "amount": 1000,
      "debtId": null
    }
  ]
}
```

**Request fields:**

| Field                                          | Type     | Required | Description                                                       |
| ---------------------------------------------- | -------- | -------- | ----------------------------------------------------------------- |
| `debtCommissionRequestModels`                  | array    | Yes      | List of commission calculation items.                             |
| `debtCommissionRequestModels[].estateId`       | integer  | Yes      | The estate to calculate commission for.                           |
| `debtCommissionRequestModels[].amount`         | decimal  | Yes      | The payment amount.                                               |
| `debtCommissionRequestModels[].debtId`         | integer? | No       | Specific debt ID. Pass `null` to calculate for the estate total.  |

**Response (200):**

```json
{
  "commission": 10
}
```

> **Tip:** Only call this endpoint when the debt's `checkCommission` value is `"Yes"`. If `checkCommission` is `"No"`, the commission is zero and you can skip this step.

---

### Debt Payment

| | |
| --- | --- |
| **Method** | `POST` |
| **Path** | `/api/v2/integration/debts/payments` |
| **Auth** | Required |

Processes debt payments. This endpoint is **idempotent** when accompanied by a unique `outerPaymentId`.

> **Important: Always pay at the estate level, not per individual debt.** Do not build UI that allows users to select and pay specific debts. The system automatically repays debts from oldest to newest (FIFO), which is the correct behavior expected by condominium associations. Allowing per-debt selection leads to situations where users pay newer debts while older ones remain unpaid, which causes disputes with condominiums. The `debtId` field exists in the API but should not be exposed to end users.

**Request:**

```json
{
  "debtPaymentRequestModels": [
    {
      "estateId": 1654896,
      "amount": 452.25,
      "debtId": null
    }
  ],
  "outerPaymentId": "9c98610b-1cd3-4f12-83e2-2a0b86ff4c2e",
  "commission": 10
}
```

**Request fields:**

| Field                                   | Type     | Required | Description                                                          |
| --------------------------------------- | -------- | -------- | -------------------------------------------------------------------- |
| `debtPaymentRequestModels`              | array    | Yes      | List of debt payments to process.                                    |
| `debtPaymentRequestModels[].estateId`   | integer  | Yes      | The estate receiving the payment.                                    |
| `debtPaymentRequestModels[].amount`     | decimal  | Yes      | Payment amount.                                                      |
| `debtPaymentRequestModels[].debtId`     | integer? | No       | Specific debt ID. **Recommended: pass `null`** (see note above).     |
| `outerPaymentId`                        | string   | Yes      | Your unique payment identifier (UUID). Used for idempotency.        |
| `commission`                            | decimal  | Yes      | Total commission amount (from the commission calculation step).      |

> **Important:** Generate a standard UUID with regular hyphens for `outerPaymentId`. Do not reuse this value across different payment operations.

**Response (200 OK):**

```json
{
  "transactions": [
    {
      "estateId": 1654896,
      "debtId": 42,
      "amount": 452.25,
      "transactionId": 443,
      "date": "2023-10-18T11:21:43.757Z",
      "bank": "Ameriabank",
      "bankAccount": "1500016548794561"
    }
  ],
  "outerPaymentId": "9c98610b-1cd3-4f12-83e2-2a0b86ff4c2e",
  "commission": 10
}
```

**Response fields:**

| Field                          | Type     | Description                               |
| ------------------------------ | -------- | ----------------------------------------- |
| `transactions`                 | array    | Completed transaction records.            |
| `transactions[].estateId`      | integer  | Estate that received the payment.         |
| `transactions[].debtId`        | integer  | Debt that was repaid.                     |
| `transactions[].amount`        | decimal  | Amount paid.                              |
| `transactions[].transactionId` | integer  | System-generated transaction ID.          |
| `transactions[].date`          | datetime | Transaction timestamp (UTC).              |
| `transactions[].bank`          | string   | Bank name. See [Bank](#bank) enum.        |
| `transactions[].bankAccount`   | string   | Bank account number.                      |
| `outerPaymentId`               | string   | Echoed back from your request.            |
| `commission`                   | decimal  | Total commission charged.                 |

**Duplicate Request (400 Bad Request):**

```json
{
  "RequestId": "Unique request ID",
  "TraceId": "Unique trace ID",
  "Instance": "Contextual API information",
  "StatusCode": 400,
  "Type": "BadRequestException",
  "Errors": null,
  "Message": "transaction_already_processed"
}
```

> **Important:** A `400` response with `"transaction_already_processed"` means the original payment was already completed successfully. **Treat this as a success.** Do not retry with a different `outerPaymentId` or charge the user again.

---

### Payment Lookup

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v1/integration/debts/payments/{outerPaymentId}` |
| **Auth** | Required |

Retrieves a previously processed transaction by its outer payment ID. Useful for verifying payment status or reconciliation.

**Path Parameters:**

| Parameter        | Type   | Description                                      |
| ---------------- | ------ | ------------------------------------------------ |
| `outerPaymentId` | string | The unique payment identifier used during payment. |

**Response (200):**

Returns the same structure as the [Debt Payment](#debt-payment) response.

---

### Typical Payment Flow

Below is the recommended sequence for processing a payment:

1. **Retrieve debts** using [Debt Retrieval by Estate](#debt-retrieval-by-estate) or [by Owner](#debt-retrieval-by-owner).
2. **Check the `checkCommission` field** on the debt response.
   - If `"Yes"`: call [Debt Commission Calculation](#debt-commission-calculation) to get the total commission amount.
   - If `"No"`: commission is `0`, skip the calculation.
3. **Submit payment** via [Debt Payment](#debt-payment) with the total amount for the estate, the commission, and a unique `outerPaymentId`. Pass `debtId` as `null` to let the system repay debts from oldest to newest automatically.
4. **Handle the response:**
   - `200`: payment successful.
   - `400` with `"transaction_already_processed"`: payment was already processed. Treat as success.
   - Other errors: inspect and handle accordingly.

---

## Search Endpoints

These endpoints provide reference data for navigating the condominium hierarchy: **City > District > Condominium Association > Building > Estate**.

### Search Cities

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v2/integration/search/cities` |
| **Auth** | Required |

Retrieves all cities that have registered buildings.

**Response (200):**

```json
[
  {
    "id": 1,
    "name": "Yerevan"
  },
  {
    "id": 2,
    "name": "Abovyan"
  },
  {
    "id": 3,
    "name": "Agarak"
  },
  {
    "id": 4,
    "name": "Alaverdi"
  }
]
```

> **Note:** This endpoint returns a JSON **array** directly, not wrapped in an object.

---

### Search Districts

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v1/integration/search/districts?cityId={cityId}` |
| **Auth** | Required |

Retrieves all districts within a given city.

**Query Parameters:**

| Parameter | Type    | Required | Description   |
| --------- | ------- | -------- | ------------- |
| `cityId`  | integer | Yes      | The city ID.  |

**Response (200):**

```json
[
  {
    "id": 13548785,
    "name": "Arabkir"
  }
]
```

> **Note:** This endpoint returns a JSON **array** directly, not wrapped in an object.

---

### Search Condominium Associations

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v3/integration/search/condominium-associations?cityId={cityId}&page={page}&pageSize={pageSize}` |
| **Auth** | Required |

Retrieves condominium associations filtered by city, with pagination.

**Query Parameters:**

| Parameter  | Type    | Required | Description                       |
| ---------- | ------- | -------- | --------------------------------- |
| `cityId`   | integer | Yes      | The city ID.                      |
| `page`     | integer | Yes      | Page number (1-based).            |
| `pageSize` | integer | Yes      | Number of results per page.       |

**Response (200):**

```json
{
  "values": [
    {
      "id": 1,
      "name": "A LLC"
    },
    {
      "id": 2,
      "name": "B LLC"
    },
    {
      "id": 3,
      "name": "C LLC"
    }
  ],
  "totalCount": 78
}
```

---

### Search Buildings

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v2/integration/search/buildings` |
| **Auth** | Required |

Retrieves buildings based on filters. Supports pagination. Filters are **combinable**: you can pass multiple to narrow results.

**Query Parameters:**

| Parameter    | Type    | Required | Description                                          |
| ------------ | ------- | -------- | ---------------------------------------------------- |
| `partnerId`  | integer | No*      | Condominium association ID.                          |
| `cityId`     | integer | No*      | City ID.                                             |
| `districtId` | integer | No*      | District ID.                                         |
| `page`       | integer | Yes      | Page number (1-based).                               |
| `pageSize`   | integer | Yes      | Number of results per page.                          |

\* At least one of `partnerId`, `cityId`, or `districtId` is required. They can be combined for more specific queries.

**Example requests:**

- `/api/v2/integration/search/buildings?partnerId=1&page=1&pageSize=1`
- `/api/v2/integration/search/buildings?cityId=1&page=1&pageSize=1`
- `/api/v2/integration/search/buildings?cityId=1&districtId=5&page=1&pageSize=10`

**Response (200):**

```json
{
  "values": [
    {
      "id": 3,
      "name": "Tumanyan 27"
    }
  ],
  "totalCount": 746
}
```

---

### Search Estates

| | |
| --- | --- |
| **Method** | `GET` |
| **Path** | `/api/v2/integration/search/estates` |
| **Auth** | Required |

Retrieves estates within a specific building. Supports pagination.

**Query Parameters:**

| Parameter    | Type    | Required | Description                  |
| ------------ | ------- | -------- | ---------------------------- |
| `buildingId` | integer | Yes      | The building ID.             |
| `page`       | integer | Yes      | Page number (1-based).       |
| `pageSize`   | integer | Yes      | Number of results per page.  |

**Example request:** `/api/v2/integration/search/estates?buildingId=1&page=1&pageSize=1`

**Response (200):**

```json
{
  "values": [
    {
      "type": "Apartment",
      "id": 13548785,
      "name": "21/2"
    }
  ],
  "totalCount": 150
}
```

The `type` field maps to the [Estate Types](#estate-types) enum.

---

## Contributing

We welcome contributions and improvements to this documentation. Submit a pull request or open an issue for suggestions.

## Issues and Support

For issues or support inquiries, use the repository's "Issues" section. Our team will respond to help resolve your problem.

## Stay Updated

We regularly update this documentation. Check the repository for the latest changes, new features, and improvements.
