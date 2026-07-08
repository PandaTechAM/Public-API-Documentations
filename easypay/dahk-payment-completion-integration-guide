# 1. DAHK Payment Completion Integration Guide (External APIs)

This endpoint allows **DAHK** to notify EasyPay about successful payment completion.

After the payment is successfully completed on DAHK side, DAHK must call this API to inform EasyPay about the completion of the payment and provide related payment information.

---

## 1.1. Prerequisites

Before integrating with EasyPay, ensure you have:

1. **Valid EasyPay Credentials**

   - **HMAC Key:** Must be stored securely and must not be exposed publicly.

2. **Base URL**

   - The root address for EasyPay external API calls.

3. Each request **must** include:

   - A valid **Authorization** header with an HMAC signature
   - A **Timestamp** header in UTC
   - A request body specific to the endpoint

### 1.1.1. Required Headers

| Header        | Value                     | Notes                                                        |
| ------------- | ------------------------- | ------------------------------------------------------------ |
| Authorization | `HMAC <DAHK>:<SIGNATURE>` | HMAC-SHA256 signature generated using the shared secret key. |
| Timestamp     | `YYYY-MM-DDTHH:mm:ssZ`    | UTC time, ISO 8601 format.                                   |
| Content-Type  | `application/json`        | Request body must be sent in JSON format.                    |

### 1.1.2. HMAC Signature Generation

Signature is generated using the **HMAC-SHA256** algorithm with a shared secret key over the string:

```text
Timestamp:PayOrderNum:InquestId
```

Example:

```text
2026-07-07T10:15:30Z:PO-123456:INQ-789456
```

The generated signature must be sent in the `Authorization` header:

```http
Authorization: HMAC DAHK:<SIGNATURE>
Timestamp: 2026-07-07T10:15:30Z
Content-Type: application/json
```

---

## 1.2. API Endpoints

### 1.2.1. Complete Payment

**Endpoint**

```http
POST /api/external/v1/dahk/payment-completions
```

**Summary**

Notifies EasyPay that a payment has been successfully completed on DAHK side.

The request is processed only once per `PayOrderNum`.

If the same `PayOrderNum` is sent again, EasyPay will not process the payment completion again, but will still return `200 OK`.

---

#### Request Body

| Field                  | Type                | Required | Description                                               |
| ---------------------- | ------------------- | -------- | --------------------------------------------------------- |
| PayOrderNum            | string              | Yes      | Unique payment order number. Used as the idempotency key. |
| RestrictionReleaseDate | date/time, nullable | No       | Restriction release date, if available.                   |
| CaseCloseDate          | date/time, nullable | No       | Case close date, if available.                            |
| RemainingAmount        | decimal, nullable   | No       | Remaining amount after payment completion, if available.  |
| InquestId              | string              | Yes      | Inquest identifier related to the payment.                |

---

#### Request Body Example

```json
{
  "payOrderNum": "PO-123456",
  "restrictionReleaseDate": "2026-07-07T00:00:00Z",
  "caseCloseDate": "2026-07-07T00:00:00Z",
  "remainingAmount": 1500.25,
  "inquestId": "INQ-789456"
}
```

#### Request Body Example With Nullable Fields

```json
{
  "payOrderNum": "PO-123456",
  "restrictionReleaseDate": null,
  "caseCloseDate": null,
  "remainingAmount": null,
  "inquestId": "INQ-789456"
}
```

---

#### Success Response

In case of success, EasyPay returns an empty response body with status code `200 OK`.

```http
200 OK
```

#### Duplicate Request Handling

The request is idempotent by `PayOrderNum`.

If EasyPay already handled a request with the same `PayOrderNum`, the duplicate request will not be processed again, but EasyPay will return `200 OK`.

```http
200 OK
```

---

#### Error Responses and Messages

`400 Bad Request`

- `PAY_ORDER_NUM_NOT_FOUND`: When the provided `PayOrderNum` does not exist.
- `INVALID_REQUEST_BODY`: When request body is malformed or contains invalid field values.

`401 Unauthorized`

- Invalid HMAC signature.
- Expired or invalid timestamp.

`403 Forbidden`

- Partner does not have permission to use this endpoint.
- Provided HMAC credentials are valid, but access to DAHK payment completion API is not allowed.

---

#### Error Response Example

```json
{
  "requestId": "007",
  "traceId": "5b4d6fafe053f9b6d8aaa8e20",
  "instance": "POST - /api/external/v1/dahk/payment-completions",
  "statusCode": 400,
  "type": "BadRequestException",
  "errors": null,
  "message": "PAY_ORDER_NUM_NOT_FOUND"
}
```

#### Forbidden Response Example

```json
{
  "requestId": "007",
  "traceId": "5b4d6fafe053f9b6d8aaa8e20",
  "instance": "POST - /api/external/v1/dahk/payment-completions",
  "statusCode": 403,
  "type": "ForbiddenException",
  "errors": null,
  "message": "FORBIDDEN"
}
```
