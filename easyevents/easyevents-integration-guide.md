# EasyEvents Ticketing API

Integrate with EasyEvents to sell event tickets through your platform. This guide covers server-side API endpoints, IFrame embedding for the ticket browsing experience, and the complete purchase flow.

---

## Introduction

EasyEvents provides two integration pathways for financial institutions:

**Server-side API** — authenticate users, retrieve order details, submit payments, and cancel reservations.

**Embeddable IFrame** — a hosted front-end where end-users browse events, select seats, and create ticket orders. Your application receives a `ticketOrderId` when the user completes a selection, then your server finalizes the payment.

After payment, EasyEvents handles everything automatically — ticket confirmation, email delivery, and IFrame state updates.

---

## Environments

|                | API Base URL                        | IFrame URL                             |
| -------------- | ----------------------------------- | -------------------------------------- |
| **Staging**    | `https://ti-api-staging.easypay.am` | `https://ti-iframe-staging.easypay.am` |
| **Production** | `https://beevents.easypay.am`       | `https://iframe.easypay.am`            |

**Staging-only resources:**

- OpenAPI Spec: `{API Base URL}/openapi/integration-v1.json`
- Scalar UI: `{API Base URL}/scalar/integration-v1`
- Swagger UI: `{API Base URL}/swagger/integration-v1`

> These interactive docs are not available in production.

---

## Prerequisites

| Item                          | Description                                                                   |
| ----------------------------- | ----------------------------------------------------------------------------- |
| **API Key**                   | Identifies your application. Part of the `Authorization` header.              |
| **HMAC Secret Key**           | Used to compute request signatures. Store in a secure vault — never hardcode. |
| **Provisioned External User** | Provided by EasyEvents for your server during onboarding.                     |
| **TLS 1.2+**                  | Required for all connections.                                                 |

### Required Headers

Every API request must include:

| Header            | Description                                                                        |
| ----------------- | ---------------------------------------------------------------------------------- |
| `Authorization`   | `HMAC <API_KEY>:<BASE64_SIGNATURE>` (see Authentication below).                    |
| `Accept-Language` | `en-US`, `hy-AM` (default), or `ru-RU`. Localizes error messages and display text. |
| `Client-Type`     | Integer indicating the source platform (see table below).                          |

**Client-Type values:**

| Value | Platform | When to use               |
| ----- | -------- | ------------------------- |
| 1     | Browser  | Web applications          |
| 2     | iOS      | iOS mobile apps           |
| 3     | Android  | Android mobile apps       |
| 4     | Windows  | Server running on Windows |
| 5     | Mac      | Server running on macOS   |
| 6     | Linux    | Server running on Linux   |
| 7     | Other    | Unclassified environments |

For server-to-server calls, use the value matching your server OS (4, 5, or 6). For mobile clients embedding the IFrame, use 2 or 3.

---

## Authentication

EasyEvents uses a two-layer authentication model:

1. **Server authentication (HMAC)** — secures all server-to-server API calls.
2. **User-level tokens** — returned by sync-and-login, used by the IFrame for the end-user session.

### HMAC Signature

```
Authorization: HMAC <API_KEY>:<BASE64_SIGNATURE>
```

**How to compute:**

1. Concatenate the endpoint-specific parameters into a single string (documented per endpoint).
2. Compute `HMAC-SHA256(secret_key, concatenated_string)`.
3. Base64-encode the result.
4. Format: `HMAC <your_api_key>:<base64_signature>`.

**Example:**

```
API Key:    ABCD123
Secret Key: your-secret-key
Input:      "ext-user-123" + "(374)93919101" + "user@example.com"
Signature:  Base64(HMAC-SHA256(secret_key, "ext-user-123(374)93919101user@example.com"))

Header:     HMAC ABCD123:ccXVrWSaE243axjtQnMUAYyWNMAlv/VzdKNAkzDPvqs=
```

> Verify your implementation at `https://easypay.am/hmac-generator`.

### Null Handling in HMAC

When a field used in HMAC calculation is `null`, substitute an **empty string**. For example, if `phoneNumber` is provided but `email` is null:

```
HMAC input: externalUserId + phoneNumber + ""
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning                                        |
| ---- | ---------------------------------------------- |
| 200  | Request succeeded.                             |
| 400  | Invalid parameters or business rule violation. |
| 401  | Authentication failed.                         |
| 403  | Insufficient permissions.                      |
| 404  | Resource not found.                            |
| 500  | Unexpected server error.                       |

### Error Response Body

```json
{
  "RequestId": "unique-request-id",
  "TraceId": "unique-trace-id",
  "Instance": "contextual-api-info",
  "StatusCode": 400,
  "Type": "BadRequestException",
  "Errors": {
    "field": "Error message"
  },
  "Message": "General error description"
}
```

Always include `RequestId` and `TraceId` when contacting support.

---

## API Endpoints

### Ping

Verify connectivity to the EasyEvents API.

```
GET /above-board/ping
```

**Response:**

```
"pong"
```

No authentication required.

---

### Sync and Login

Synchronize a user to EasyEvents and obtain a refresh token for the IFrame session.

```
POST /api/v1/integration/authentication/sync-and-login
```

**HMAC parameters:** `externalUserId + (phoneNumber ?? "") + (email ?? "")`

> At least one of `phoneNumber` or `email` is required. If a field is null, use an empty string in the HMAC calculation.

**Request:**

```json
{
  "externalUserId": "ext-user-123",
  "phoneNumber": "(374)93919101",
  "email": "user@example.com",
  "firstName": "Vardan",
  "lastName": "Vardanyan",
  "middleName": "Vazgeni",
  "device": {
    "clientType": 2,
    "name": "iPhone 15 Pro",
    "userAgent": "Mozilla/5.0 ...",
    "uniqueIdPerDevice": "077cc5f5-337a-46bf-a815-a2ea90ec24f2"
  },
  "authenticationHistory": {
    "latitude": "44.456",
    "longitude": "46.584",
    "accuracy": 1.5,
    "userNetworkAddress": "143.156.250.11",
    "forwardedData": "X-Forwarded-For: 143.156.250.11, 198.51.100.1"
  }
}
```

**Response:**

```json
{
  "refreshTokenSignature": "e384b714-86e3-4cde-a658-00a321045735"
}
```

Store the `refreshTokenSignature` securely — it's used to authenticate the IFrame session.

**Error cases:**

- `400` — Invalid phone number or email format.
- `400` — Neither email nor phone number provided.
- `400` — User is disabled or deleted.
- `400` — Device not allowed for this user.

---

### Get Ticket Order

Retrieve order details after the user creates an order via the IFrame.

```
GET /api/v1/integration/order/{ticketOrderId}
```

**HMAC parameters:** `ticketOrderId`

**Response:**

```json
{
  "ticketOrderId": "abc123",
  "price": 12000,
  "commission": 200,
  "bankName": 1,
  "bankAccount": "1450013254687513",
  "tickets": [
    {
      "eventName": "Stand Up Show",
      "price": 1000,
      "areaName": "Area 1",
      "areaId": 100,
      "row": "1",
      "seat": 3,
      "seatId": 12537,
      "discountedPrice": 10
    }
  ]
}
```

**Response fields:**

| Field         | Description                                                                                                       |
| ------------- | ----------------------------------------------------------------------------------------------------------------- |
| `price`       | Total order amount (sum of ticket prices after discounts).                                                        |
| `commission`  | Service commission charged to the customer.                                                                       |
| `bankName`    | Bank of the entertainment provider (see Bank Types below). Required by financial institutions for wire transfers. |
| `bankAccount` | Bank account number of the entertainment provider.                                                                |

> **Important:** Reservations expire after **15 minutes**. If payment is not submitted within this window, the order is automatically canceled and seats are released. Retrieve the order promptly after receiving the `ticketOrderId` from the IFrame.

**Bank Types:**

| Code | Bank                |
| ---- | ------------------- |
| 0    | ACBA Bank           |
| 1    | Araratbank          |
| 2    | Ameriabank          |
| 3    | Amio Bank           |
| 4    | ID Bank             |
| 5    | Ardshinbank         |
| 6    | Armswissbank        |
| 7    | Artsakhbank         |
| 8    | Byblos Bank Armenia |
| 9    | HSBC Bank Armenia   |
| 10   | Evocabank           |
| 11   | Inecobank           |
| 12   | Converse Bank       |
| 13   | Armeconombank       |
| 14   | Mellat Bank         |
| 15   | Unibank             |
| 16   | VTB Armenia Bank    |
| 17   | Fast Bank           |

These are provided so your financial institution can route the wire transfer to the entertainment provider's account.

**Error cases:**

- `404` — Order not found.

---

### Submit Payment

Submit payment confirmation for a ticket order.

```
POST /api/v1/integration/payment
```

**HMAC parameters:** `ticketOrderId`

**Request:**

```json
{
  "ticketOrderId": "abc123",
  "price": 12000,
  "commission": 200,
  "externalSystemPaymentId": "d97cf534-a7cd-473a-ba86-71bbeb31b872"
}
```

| Field                     | Description                                                                  |
| ------------------------- | ---------------------------------------------------------------------------- |
| `ticketOrderId`           | The order to pay for.                                                        |
| `price`                   | Must match the order's `price` from the Get Ticket Order response.           |
| `commission`              | Must match the calculated commission. EasyEvents validates this server-side. |
| `externalSystemPaymentId` | Your unique payment reference. Used for reconciliation.                      |

**Response:**

```json
{
  "partnerFullName": "Entertainment Corp LLC",
  "uniqueDocumentId": "5454545445",
  "bankAccount": "1450013254687513",
  "TinNumber": "1234567",
  "ArmSoftAccount": "4567891"
}
```

**Response fields — settlement details for the entertainment provider:**

| Field              | Description                                                     |
| ------------------ | --------------------------------------------------------------- |
| `partnerFullName`  | Legal name of the entertainment provider.                       |
| `TinNumber`        | Tax identification number of the entertainment provider.        |
| `bankAccount`      | Bank account to wire ticket revenue to.                         |
| `ArmSoftAccount`   | Accounting system account number of the entertainment provider. |
| `uniqueDocumentId` | Unique document reference for the settlement record.            |

Use these fields to initiate the wire transfer from your financial institution to the entertainment provider.

**Error cases:**

- `400` — Commission mismatch (recalculate and retry with the current value).
- `400` — Duplicate `externalSystemPaymentId`.
- `400` — Order is not in `Reserved` or `Frozen` status.

---

### Cancel Ticket Order

Cancel a pending reservation, releasing the held seats.

```
DELETE /api/v1/integration/order/{ticketOrderId}
```

**HMAC parameters:** `ticketOrderId`

Returns `200 OK` on success (no response body).

**Error cases:**

- `400` — Already paid orders cannot be canceled.
- `404` — Order not found.

> Reservations auto-expire after 15 minutes, but explicitly canceling frees seats immediately for other customers.

---

## IFrame Integration

The EasyEvents IFrame is a hosted front-end that allows end-users to browse events, select seats, and create ticket orders — all embedded within your application.

### Integration Flow

1. **Obtain a refresh token** — call Sync and Login on your server to get `refreshTokenSignature`.
2. **Build the IFrame URL** — append authentication parameters to the base IFrame URL.
3. **Load the IFrame** — open a WebView or IFrame component with the constructed URL.
4. **Handle messages** — listen for `postMessage` events from the IFrame to receive order IDs and navigation commands.
5. **Finalize via API** — use the `ticketOrderId` from the IFrame to get order details, submit payment, or cancel.

### Building the IFrame URL

Append `refresh_token` and `device_id` as query parameters:

```
{IFrame Base URL}?refresh_token={refreshTokenSignature}&device_id={uniqueDeviceId}&lang={languageCode}
```

**Example:**

```
https://ti-iframe-staging.easypay.am?refresh_token=e384b714-86e3-4cde-a658-00a321045735&device_id=device-uuid-1234&lang=hy-AM
```

**Special routes:**

| URL path           | Purpose                       |
| ------------------ | ----------------------------- |
| `/`                | Event listing (default)       |
| `/event/{eventId}` | Deep-link to a specific event |
| `/owner-tickets`   | User's ticket history         |

All routes accept the same query parameters (`refresh_token`, `device_id`, `lang`).

### IFrame Messages

The IFrame communicates with your application via `postMessage`. Messages are JSON objects with one or more command keys:

| Key              | Payload                       | Action                                                                                               |
| ---------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------- |
| `checkout_id`    | `string` — ticket order ID    | Navigate to your payment screen. Use this ID to call Get Ticket Order and Submit Payment.            |
| `event_id`       | `string` — event ID           | User tapped "Share". Build a deep link: `{IFrame URL}/event/{event_id}?...`                          |
| `link`           | `string` — URL                | Open the URL in an external or in-app browser (e.g., YouTube, maps).                                 |
| `pdf`            | `string` — Base64-encoded PDF | Decode and display/share the PDF (ticket or receipt).                                                |
| `goBack`         | `"true"` / `"false"`          | Close the IFrame or navigate back.                                                                   |
| `sync_and_login` | `"true"` / `"false"`          | Session expired. Call Sync and Login again to get a new refresh token. On failure, close the IFrame. |

> Messages may arrive in any order and can contain multiple keys. Handle each key independently.

### Listening for Messages

**Flutter/Dart:**

```dart
String buildIFrameUrl({
  required String baseUrl,
  required String refreshToken,
  required String deviceId,
  String? lang,
}) {
  final uri = Uri.parse(baseUrl);
  return uri.replace(queryParameters: {
    ...uri.queryParameters,
    'refresh_token': refreshToken,
    'device_id': deviceId,
    if (lang != null) 'lang': lang,
  }).toString();
}

// In your WebView setup:
controller.addJavaScriptChannel(
  'parent',
  onMessageReceived: (message) {
    final data = jsonDecode(message.message);
    if (data['checkout_id'] != null) {
      navigateToPayment(data['checkout_id']);
    }
    if (data['goBack'] == 'true') {
      Navigator.pop(context);
    }
    // Handle other keys...
  },
);
```

**Web (JavaScript):**

```js
window.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  if (data.checkout_id) {
    // Navigate to payment with data.checkout_id
  }
});
```

**React Native:** Use `onMessage` prop on `WebView` from `react-native-webview`.

**Native iOS:** Set up `WKScriptMessageHandler` on `WKWebView`.

**Native Android:** Use `addJavascriptInterface` on `WebView`.

### IFrame Security

- **HTTPS only** — never load the IFrame over HTTP.
- **Protect the refresh token** — do not log it or persist it in plaintext.
- **Sanitize messages** — validate JSON from the IFrame before acting on it. Ignore unrecognized keys.
- **Minimal query parameters** — only include what's required (`refresh_token`, `device_id`, `lang`).

---

## Implementation Checklist

- [ ] Store API Key and HMAC Secret in a secure vault.
- [ ] Implement Sync and Login and securely store the refresh token per user session.
- [ ] Build the IFrame URL with `refresh_token` and `device_id`.
- [ ] Handle all IFrame message types (`checkout_id`, `goBack`, `sync_and_login`, `pdf`, `link`, `event_id`).
- [ ] On `checkout_id`: call Get Ticket Order within **15 minutes** (before the reservation expires).
- [ ] Submit Payment with the exact `price` and `commission` from the order details.
- [ ] Store `externalSystemPaymentId` with a unique index to prevent duplicate payments.
- [ ] Handle the `sync_and_login` message by refreshing the token and reloading the IFrame.
- [ ] Include `RequestId` and `TraceId` in all support requests.

---

## Support

For integration support, contact your EasyEvents integration manager. Always include `RequestId` and `TraceId` from error responses when reporting issues.

---

_Last updated: 16-03-2026_
