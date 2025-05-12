- [1. EasyEvents Integration Guide](#1-easyevents-integration-guide)
  - [1.1. Introduction](#11-introduction)
  - [1.2. Key Terminology](#12-key-terminology)
  - [1.3. Overview](#13-overview)
    - [1.3.1. Environments](#131-environments)
  - [1.4. Prerequisites](#14-prerequisites)
  - [1.5. Security \& Authentication](#15-security--authentication)
    - [1.5.1. Server Authentication](#151-server-authentication)
    - [1.5.2. User-Level Authentication](#152-user-level-authentication)
    - [1.5.3. Client Type Header (Mandatory)](#153-client-type-header-mandatory)
  - [1.6. Response Codes \& Error Handling](#16-response-codes--error-handling)
    - [1.6.1. Standard HTTP Response Codes](#161-standard-http-response-codes)
  - [1.7. Error Response Structure](#17-error-response-structure)
  - [1.8. API Endpoints](#18-api-endpoints)
    - [1.8.1. Ping](#181-ping)
    - [1.8.2. Sync and Login](#182-sync-and-login)
    - [1.8.3. Ticket Order](#183-ticket-order)
    - [1.8.4. Payment](#184-payment)
    - [1.8.5. Ticket Order Cancel](#185-ticket-order-cancel)
  - [1.9. IFrame Integration](#19-iframe-integration)
    - [1.9.1. Introduction](#191-introduction)
    - [1.9.2. Integration Flow](#192-integration-flow)
    - [1.9.3. Building the IFrame URL](#193-building-the-iframe-url)
    - [1.9.4. Communication Mechanism (JavaScript Channel)](#194-communication-mechanism-javascript-channel)
    - [1.9.5. Handling IFrame Commands](#195-handling-iframe-commands)
    - [1.9.6. Example: Constructing \& Loading the IFrame in Flutter (Dart)](#196-example-constructing--loading-the-iframe-in-flutter-dart)
    - [1.9.7. Security Considerations](#197-security-considerations)
    - [1.9.8. Additional Use Cases](#198-additional-use-cases)
  - [1.10. Support \& Troubleshooting](#110-support--troubleshooting)

# 1. EasyEvents Integration Guide

## 1.1. Introduction

Welcome to the EasyEvents Integration Guide. This document outlines how to:

1. **Integrate Back-end Services**
   Perform secure authentication, create and manage ticket orders, handle payments, and manage cancellations.

2. **Embed the IFrame**
   Allow end-users to browse events, select seats, and initiate ticket purchases directly within your UI.

By following this guide, you’ll seamlessly integrate EasyEvents into your existing or new software solutions, ensuring secure, efficient ticketing operations.

## 1.2. Key Terminology

- **Base URL:** The root address for API calls.
- **API Key / Secret:** Credentials provided by EasyEvents for server-to-server authentication (HMAC-based).
- **HMAC Signature:** A cryptographic hash used to verify request integrity.
- **TicketOrderId:** A unique identifier for a created ticket order, returned by the IFrame or retrieved from the server.
- **IFrame:** A front-end component allowing end-users to search and purchase tickets within your application UI.
- **Refresh Tokens:** User-level tokens returned by `sync-and-login`, used by the IFrame for authentication.

## 1.3. Overview

EasyEvents offers two primary integration pathways:

1. **Server Integration**

   - Obtain server credentials (API key and secret).
   - Implement user synchronization (sync-and-login), ticket order lookups, and payment operations.

2. **IFrame Integration**

   - Embed the EasyEvents IFrame into your UI.
   - The IFrame automatically uses user-level tokens for searching, reserving seats, and initiating orders.
   - You receive a `TicketOrderId` upon successful reservation, which your server can use to finalize or cancel the order.

For testing, use the **Test Environment**. Once verified, switch your base URLs to the **Production Environment**.

### 1.3.1. Environments

**Test Environment**

- **Base URL:**
  [https://qabeevents.easypay.am](https://qabeevents.easypay.am).
- **IFrame URL:**
  [https://qaiframe.easypay.am](https://qaiframe.easypay.am)
- **OpenAPI Spec:**
  [https://qabeevents.easypay.am/openapi/integration-v1.json](https://qabeevents.easypay.am/openapi/integration-v1.json)
- **Scalar UI (Test Only):**
  [https://qabeevents.easypay.am/scalar/integration-v1](https://qabeevents.easypay.am/docs/integration-v1)
- **Swagger UI (Test Only):**
  [https://qabeevents.easypay.am/swagger/integration-v1](https://qabeevents.easypay.am/swagger/integration-v1)

**Production Environment**

- **Base URL:**
  [https://beevents.easypay.am](https://beevents.easypay.am) (api url)
- **IFrame URL**
  [https://iframe.easypay.com](https://iframe.easypay.com) (front-end url)

> **Note:** Scalar UI and Swagger UI are not available in the production environment.

## 1.4. Prerequisites

1. **Valid EasyEvents Credentials**

   - **API Key and Secret:** Used for HMAC-SHA256 server authentication.
   - **Provisioned External User:** Provided by EasyEvents for your server.

2. **Accept-Language Header**
   Include a valid ISO language code in the `Accept-Language` header:

- `en-US` for English
- `hy-AM` for Armenian (default if missing or invalid)
- `ru-RU` for Russian

3. **Network Access to EasyEvents**
   Ensure your environment can reach the test or production URLs.

## 1.5. Security & Authentication

EasyEvents uses a two-layered authentication strategy:

1. **Server Authentication**
   Secure each server-to-server request with HMAC-SHA256 using the keys provided by EasyEvents.
2. **User-Level Tokens**
   Perform a `sync-and-login` to obtain per-user refresh token. The client-side IFrame automatically handles token usage and refresh.

### 1.5.1. Server Authentication

**Header Format**

```txt
Authorization: HMAC <API_KEY>:<BASE64_SIGNATURE>
```

- `<API_KEY>` is provided by EasyEvents.
- `<BASE64_SIGNATURE>` is an HMAC-SHA256 hash computed over relevant fields (e.g., `externalUserId`, `phoneNumber`, `email`,`ticketOrderId`) concatenated into a string. The hash is then Base64-encoded.

**Example**

```txt
apiKey     = "ABCD123"
secretKey  = "Secret"
inputData  = "externalUserId + phoneNumber + email" or ("ticketOrderId")
signature  = Base64(HMACSHA256(inputData, secretKey))

// In final header:
Authorization: HMAC ABCD123:ccXVrWSaE243axjtQnMUAYyWNMAlv/VzdKNAkzDPvqs=
```

> **Note:** You can verify results with an online HMAC-SHA256 generator (e.g., https://easypay.am/hmac-generator).

### 1.5.2. User-Level Authentication

**Sync and Login**

- **Endpoint:** `POST /api/v1/integration/authentication/sync-and-login`
- **Purpose:** Synchronize user details and retrieve refresh token.
- **Usage:**
  - Pass `externalUserId` and at least one of `phoneNumber` or `email`.
  - Include the HMAC-based `Authorization` header.
  - On success, the user-specific refresh token is returned, which the IFrame uses for subsequent requests.

### 1.5.3. Client Type Header (Mandatory)

Along with the standard headers (`Authorization`, `Accept-Language`), all API requests must include a header indicating the `Client-Type`:

```txt
client-type: <int value from the ClientType enum>
```

Where `ClientType` is defined as:

```cs
public enum ClientType
{
   Browser = 1,
   Ios = 2,
   Android = 3,
   Windows = 4,
   Mac = 5,
   Linux = 6,
   Other = 7
}
```

1. **For Server Requests**
   Use the appropriate OS type: `Windows (4)`, `Mac (5)`, or `Linux (6)`.
   If you simply cannot classify the server, use `Other (7)`.

2. **For Mobile Clients**
   Use `Ios (2)` or `Android (3)`.

This header ensures the EasyEvents platform correctly classifies and tracks the source environment. If omitted or invalid, requests may be rejected.

## 1.6. Response Codes & Error Handling

### 1.6.1. Standard HTTP Response Codes

| Code | Description                                       |
| :--- | :------------------------------------------------ |
| 200  | Request succeeded.                                |
| 202  | Request accepted (e.g. order enqueued).           |
| 400  | Invalid request parameters or duplicate requests. |
| 401  | Authentication failed.                            |
| 403  | Insufficient permissions.                         |
| 404  | Resource not found.                               |
| 500  | Server encountered an unexpected error.           |

## 1.7. Error Response Structure

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

## 1.8. API Endpoints

Below are the primary endpoints for server-side integration. In all cases, include the `Authorization: HMAC ...` header and, the `Accept-Language` header.

### 1.8.1. Ping

- **Endpoint:** `GET /above-board/ping`
- **Description:** Check connectivity
- **Response:**
  ```json
  "pong"
  ```

### 1.8.2. Sync and Login

- **Endpoint:** `POST /api/v1/integration/authentication/sync-and-login`
- **Description:** Synchronizes the user to EasyEvents (creating or updating if necessary) and returns refresh token.
- **Request:**
  Include at least `externalUserId` and either `phoneNumber` or `email`. Also provide `Authorization` header with the HMAC signature.

  Example:

  ```json
  {
    "externalUserId": "Asdkfj123!zax",
    "phoneNumber": "(374)93919101",
    "email": "vardan.vardanyan@gmail.com",
    "firstName": "Vardan",
    "lastName": "Vardanyan",
    "middleName": "Vazgeni",
    "device": {
      "clientType": 2,
      "name": "Iphone 11 Pro",
      "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36",
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

- **Response:**

  ```json
  {
    "refreshTokenSignature": "e384b714-86e3-4cde-a658-00a321045735"
  }
  ```

- **Common Errors:**
  - `400 Bad Request` - Invalid phone number format.
  - `400 Bad Request` - Invalid email format.
  - `400 Bad Request` - Either Email or Phone Number is required.
  - `400 Bad Request` - Invalid Armenian SSN format.
  - `400 Bad Request` - user_is_disabled_or_deleted.
  - `400 Bad Request` - you_cannot_login_with_this_device.

### 1.8.3. Ticket Order

- **Endpoint:** `GET /api/v1/integration/order/{ticketOrderId}`
- **Description:** Retrieve order details after the user has created an order in the IFrame.
  - Note: You must retrieve the ticketOrderId from the IFrame before the user is redirected, as it is required to call this endpoint.
  - Also provide `Authorization` header with the HMAC signature (inputData  = "ticketOrderId").
- **Request:** /api/v1/integration/order/asd
- **Response:**
  ```json
  {
    "ticketOrderId": "asd",
    "price": 12000,
    "commission": 200,
    "bankName": 1,
    "bankAccount": "1450013254687513",
    "tickets": [
      {
        "eventName": "Stand Up",
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
- **BankName enum:**

  ```csharp
  public enum BankType
  {
    [Description("ԱԿԲԱ ԲԱՆԿ")] AcbaBank,
    [Description("ԱՐԱՐԱՏԲԱՆԿ")] AraratBank,
    [Description("ԱՄԵՐԻԱԲԱՆԿ")] Ameriabank,
    [Description("ԱՄԻՕ ԲԱՆԿ")] AmioBank,
    [Description("ԱՅԴԻ ԲԱՆԿ")] IDBank,
    [Description("ԱՐԴՇԻՆԲԱՆԿ")] Ardshinbank,
    [Description("ԱՐՄՍՎԻՍԲԱՆԿ")] Armswissbank,
    [Description("ԱՐՑԱԽԲԱՆԿ")] Artsakhbank,
    [Description("ԲԻԲԼՈՍ ԲԱՆԿ ԱՐՄԵՆԻԱ")] BiblosBankArmenia,
    [Description("ԷՅՉ-ԷՍ-ԲԻ-ՍԻ ԲԱՆԿ ՀԱՅԱՍՏԱՆ")] HSBCBankArmenia,
    [Description("ԷՎՈԿԱԲԱՆԿ")] Evocabank,
    [Description("ԻՆԵԿՈԲԱՆԿ")] Inecobank,
    [Description("ԿՈՆՎԵՐՍ ԲԱՆԿ")] ConverseBank,
    [Description("ՀԱՅԷԿՈՆՈՄԲԱՆԿ")] Armeconombank,
    [Description("ՄԵԼԼԱԹ ԲԱՆԿ")] MellatBank,
    [Description("ՅՈՒՆԻԲԱՆԿ")] Unibank,
    [Description("ՎՏԲ-ՀԱՅԱՍՏԱՆ ԲԱՆԿ")] VTBArmeniaBank,
    [Description("ՖԱՍԹ ԲԱՆԿ")] FastBank,
  }
  ```

- **Error Cases:**
  - `404 Not Found` if the specified order is not found.

### 1.8.4. Payment

- **Endpoint:** `POST /api/v1/integration/payment`
- **Description:** Submit payment details for a completed ticket purchase. Also provide `Authorization` header with the HMAC signature (inputData  = "ticketOrderId").
- **Request Body:**

  ```json
  {
    "ticketOrderId": "asd",
    "price": 12000,
    "commission": 200,
    "externalSystemPaymentId": "d97cf534-a7cd-473a-ba86-71bbeb31b872"
  }
  ```

- **Response:**

  ```json
  {
    "partnerFullName": "Partners Name",
    "uniqueDocumentId": "5454545445",
    "bankAccount": "1450013254687513"
  }
  ```

- **Error Cases:**
  - `400 Bad Request` - Invalid price or commission mismatch.
  - `400 Bad Request` - Duplicate `ExternalSystemPaymentId`

### 1.8.5. Ticket Order Cancel

- **Endpoint:** `DELETE /api/v1/integration/order/{ticketOrderId}`
- **Description:** Cancel a pending ticket order, releasing any reserved seats.Also provide `Authorization` header with the HMAC signature (inputData  = "ticketOrderId").
- **Request:** /api/v1/integration/order/asd
- **Response:**

  - No body; a `200 OK` implies success.

- **Error Cases:**
  - `400 Bad Request` - Already paid ticket orders cannot be canceled.
  - `404 Not Found` – Order does not exist.

## 1.9. IFrame Integration

EasyEvents provides an embeddable IFrame for ticket browsing, purchase, and immediate creation of ticket orders.

### 1.9.1. Introduction

EasyEvents provides an embeddable IFrame that allows end-users to:

1. Browse available events.
2. Select seats (if applicable).
3. Immediately create a ticket order.

Once an order is created in the IFrame, the host application is notified (via JavaScript channel or similar messaging mechanism) and receives a `ticketOrderId`. That ID can then be used by your backend to finalize or cancel the order, as needed.

### 1.9.2. Integration Flow

A typical IFrame integration flow looks like this:

1. **Obtain User Tokens (Server-Side)**

   - On your server, call **Sync and Login** (`/api/v1/integration/authentication/sync-and-login`) to retrieve the `refreshTokenSignature`.
   - Store this token securely and associate it with the current user/session in your application.

2. **Construct the IFrame URL**

   - Start with the EasyEvents IFrame base URL (Test or Production).
   - Append the user’s `refreshTokenSignature` as `refresh_token`.
   - Also append a `device_id` (unique device identifier) or any other required query parameters.

3. **Load the IFrame in Your Front-End**

   - Open a WebView or IFrame component in your front-end.
   - Pass the constructed URL to the component, ensuring that HTTPS is used.

4. **Handle Messages from the IFrame**

   - The IFrame sends events to the host application via JavaScript messages (e.g., `window.parent.postMessage`, or the equivalent channel mechanism on your platform).

   - Parse and validate these messages. If a `ticketOrderId` (or `checkout_id` key) is present, use it to retrieve order details, process payments, etc.

5. **Finalize or Cancel Orders via API**

   - Once you have the `ticketOrderId`, you can call EasyEvents endpoints such as:

     - `GET /api/v1/integration/order/{ticketOrderId}` to retrieve details,

     - `POST /api/v1/integration/payment` to submit a payment, and

     - `DELETE /api/v1/integration/order/{ticketOrderId}` to cancel.

### 1.9.3. Building the IFrame URL

When embedding the IFrame, you need to append essential authentication parameters to the base URL. The two most critical parameters are:

1. `refresh_token`

   - Comes from the `refreshTokenSignature` field, obtained via **Sync and Login**.
   - Used by the IFrame to fetch or refresh the user’s access token.

2. `device_id`

   - A unique identifier representing the device or session.
   - Required by EasyEvents for security, logging, and session tracking.

> Important: You may have other parameters required by your business logic or additional query params from EasyEvents. Always ensure these parameters are merged without overwriting each other.

Example base URLs:

- **Test:** `https://qaiframe.easypay.am`
- **Production:** `https://iframe.easypay.com`

If you start with `https://qaiframe.easypay.am?lang=hy-AM`, for example, and need to append `refresh_token` and `device_id`, your final URL might look like:

```txt
https://qaiframe.easypay.am?lang=hy-AM&refresh_token=<YOUR_REFRESH_TOKEN>&device_id=<UNIQUE_DEVICE_ID>
```

### 1.9.4. Communication Mechanism (JavaScript Channel)

To communicate back to your application, the IFrame uses the `postMessage` API (or a similar mechanism), typically targeting a specific channel name. In Flutter/Dart’s WebView, this channel is often referred to as `JavaScriptChannel`. In other frameworks (React Native, Xamarin, etc.), the approach is analogous:

- The IFrame calls `window.parent.postMessage(...)` to send data.
- Your host application listens on the appropriate channel and processes the incoming JSON messages.

Below is a Flutter-specific example, but the concept is identical in other platforms:

```dart
webView.addJavaScriptChannel(
  'parent',
  onMessageReceived: (JavaScriptMessage message) {
    final String rawJson = message.message;
    // Parse, validate, and handle the JSON message
  },
);
```

```js
window.parent.postMessage(JSON.stringify({ checkout_id: '123' }), '*');
```

Or, if using a named channel inside a WebView context:

```js
Parent.postMessage('{"checkout_id":"123"}');
```

> Tip: Always validate or sanitize the data in the received JSON before acting on it.

### 1.9.5. Handling IFrame Commands

All communication from the IFrame arrives as JSON messages that may contain one or more command keys. Each command key signals a specific action:

- `checkout_id:`
  Indicates that a user has selected a ticket order for checkout.
  Example:

  ```json
  { "checkout_id": "<ticket_order_id>" }
  ```
- `event_id:`
  To enable proper navigation within the application, provide the parent event identifier to establish a deeplink navigation path.
  Example:

  ```json
  { "event_id": "<event_id>" }
  ```

  _Host Action:_ Navigate to a payment screen or retrieve the order details.  <br><br>

- `link`
  A URL that the application should open in a browser (e.g., YouTube link, maps directions).
  Example:

  ```json
  { "link": "https://myspecialoffer.example.com" }
  ```

  _Host Action:_ Open an external browser or in-app browser with the provided link. <br><br>

- `pdf`
  A Base64-encoded PDF, e.g., a ticket or receipt.
  Example:

  ```json
  { "pdf": "<base64_encoded_pdf_data>" }
  ```

  _Host Action:_ Decode and display/share the PDF file. <br><br>

- `goBack`
  A boolean string (`"true"`/`"false"`) instructing the host app to navigate back or close the IFrame.
  Example:

  ```json
  { "goBack": "true" }
  ```

  _Host Action:_ Close or pop the current screen to return to a previous view.

> Important: Messages may arrive in any order and can contain multiple keys. Treat each key independently.

### 1.9.6. Example: Constructing & Loading the IFrame in Flutter (Dart)

Below is a concise Flutter/Dart snippet showing how to append query parameters to the base IFrame URL and then load it into a `WebView`. You can do the same (albeit with different APIs) in React Native, Xamarin, native Android/iOS, or any other platform.

```dart
String buildIFrameUrl({
  required String baseIFrameUrl,
  required String refreshTokenSignature,
  required String deviceId,
  Map<String, String>? additionalParams,
}) {
  final uri = Uri.parse(baseIFrameUrl);
  final updatedUri = uri.replace(queryParameters: {
    // Keep original parameters
    ...uri.queryParameters,
    // Add required ones
    'refresh_token': refreshTokenSignature,
    'device_id': deviceId,
    // Add any extras, if provided
    ...?additionalParams,
  });

  return updatedUri.toString();
}

...

// Usage in code:
final iframeUrl = buildIFrameUrl(
  baseIFrameUrl: 'https://qaiframe.easypay.am',
  refreshTokenSignature: 'e384b714-86e3-4cde-a658-00a321045735',
  deviceId: 'device-uuid-1234',
);

WebView(
  initialUrl: iframeUrl,
  javascriptMode: JavascriptMode.unrestricted,
  onWebViewCreated: (controller) {
    controller.addJavaScriptChannel(
      'parent',
      onMessageReceived: (JavaScriptMessage message) {
        _handleIFrameMessage(message.message);
      },
    );
  },
);
```

### 1.9.7. Security Considerations

1. **HTTPS Only**
   Always use `https://` for the IFrame URL to avoid sending tokens over unencrypted connections.

2. **Secure Token Handling**

   - The `refresh_token` must be protected. Do not store it in plaintext or persist it in logs.

   - Always validate your tokens on the server side before performing sensitive operations.

3. **Sanitize Incoming Messages**

   - Parse the JSON from the IFrame carefully.

   - Reject or ignore unrecognized keys, suspicious links, or invalid data.

4. **Minimal Exposure**

   - Include only the parameters that are strictly necessary in the query string.

   - If your platform or environment exposes the full URLs in logs, consider additional measures (like ephemeral short-lifetime tokens).

### 1.9.8. Additional Use Cases

- **Native iOS/Android:** Use a web component (`WKWebView` in iOS or `WebView` in Android) and set up a similar “bridge” for message handling.

- **React Native:** Implement `WebView` from `react-native-webview` and the `onMessage` prop to capture postMessage events from the IFrame.

- **Xamarin:** Use the built-in WebView or third-party libraries. The messaging concept remains the same: calls from JavaScript back to C#.

## 1.10. Support & Troubleshooting

For additional assistance or to report issues, please contact your designated EasyEvents integration manager. When troubleshooting, include both the `RequestId` and `TraceId` from the error responses to help expedite the resolution process.
