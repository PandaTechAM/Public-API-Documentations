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
    - [1.9.1. Implementation Steps](#191-implementation-steps)
    - [1.9.2. Support \& Troubleshooting](#192-support--troubleshooting)

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
  [https://events.easypay.am](https://events.easypay.am)
- **IFrame URL**
  [https://iframe.easypay.com](https://iframe.easypay.com)

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
   Perform a `sync-and-login` to obtain per-user access and refresh tokens. The client-side IFrame automatically handles token usage and refresh.

### 1.5.1. Server Authentication

**Header Format**

```txt
Authorization: HMAC <API_KEY>:<BASE64_SIGNATURE>
```

- `<API_KEY>` is provided by EasyEvents.
- `<BASE64_SIGNATURE>` is an HMAC-SHA256 hash computed over relevant fields (e.g., `externalUserId`, `phoneNumber`, `email`) concatenated into a string. The hash is then Base64-encoded.

**Example**

```txt
apiKey     = "ABCD123"
secretKey  = "Secret"
inputData  = "externalUserId + phoneNumber + email"
signature  = Base64(HMACSHA256(inputData, secretKey))

// In final header:
Authorization: HMAC ABCD123:ccXVrWSaE243axjtQnMUAYyWNMAlv/VzdKNAkzDPvqs=
```

> **Note:** You can verify results with an online HMAC-SHA256 generator (e.g., https://easypay.am/hmac-generator).

### 1.5.2. User-Level Authentication

**Sync and Login**

- **Endpoint:** `POST /api/v1/integration/authentication/sync-and-login`
- **Purpose:** Synchronize user details and retrieve access/refresh tokens.
- **Usage:**
  - Pass `externalUserId` and at least one of `phoneNumber` or `email`.
  - Include the HMAC-based `Authorization` header.
  - On success, the user-specific tokens (access/refresh) are returned, which the IFrame uses for subsequent requests.

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
- **Description:** Synchronizes the user to EasyEvents (creating or updating if necessary) and returns refresh tokens.
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
    "accessTokenSignature": "c384b714-86e3-4cde-a658-00a321045735",
    "accessTokenExpiration": "2025-03-28T08:48:15.404Z",
    "refreshTokenSignature": "e384b714-86e3-4cde-a658-00a321045735",
    "refreshTokenExpiration": "2025-03-28T08:48:15.404Z"
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
- **Request:** /api/v1/integration/order/asd
- **Response:**
  ```json
  {
    "ticketOrderId": "asd",
    "price": 12000,
    "commission": 200,
    "bankName": 1,
    "bankAccount": "1450013254687513"
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
- **Description:** Submit payment details for a completed ticket purchase.
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

  - No body; a `200 OK` implies success.

- **Error Cases:**
  - `400 Bad Request` - Invalid price or commission mismatch.
  - `400 Bad Request` - Duplicate `ExternalSystemPaymentId`

### 1.8.5. Ticket Order Cancel

- **Endpoint:** `DELETE /api/v1/integration/order/{ticketOrderId}`
- **Description:** Cancel a pending ticket order, releasing any reserved seats.
- **Request:** /api/v1/integration/order/asd
- **Response:**

  - No body; a `200 OK` implies success.

- **Error Cases:**
  - `400 Bad Request` - Already paid ticket orders cannot be canceled.
  - `404 Not Found` – Order does not exist.

## 1.9. IFrame Integration

EasyEvents provides an embeddable IFrame for ticket browsing, purchase, and immediate creation of ticket orders.

### 1.9.1. Implementation Steps

## Overview

This document describes how a host application integrates with an external event management iFrame. Communication from the iFrame to the application is done using message-passing (typically postMessage). Each message is sent independently and is treated as a separate command with its own purpose.

## API Endpoint

The host application retrieves the iFrame configuration by sending a GET request to: `/api/mobile/v1/integrations/easy-events/sync-and-login`.

## Purpose of URL Construction

To securely embed an authenticated session in an iFrame, the application must append
authentication-related query parameters to the base iFrame URL. These parameters include access
and refresh tokens, as well as a unique device identifier.

**Parameters Appended to URL**

- `token`: A temporary access token for user authentication
- `refresh_token`: Token used to renew the access token
- `device_id`: Identifier of the device making the request (used to session tracking)

**How It Works**
- Parse the original iFrame URL received from the server.
- Extract existing query parameters.
- Merge existing parameters with required authentication parameters.
- Create and return the updated URL as a string.

## Example in Flutter (Dart)

```dart
String _addQueryParamsToUrl(IFrameEntity iFrameEntity) {
  final Uri uri = Uri.parse(iFrameEntity.iFrameUrl);
  final updatedUri = uri.replace(queryParameters: {
    ...uri.queryParameters,
    'token': iFrameEntity.accessTokenSignature,
    'refresh_token': iFrameEntity.refreshTokenSignature,
    'device_id': iFrameEntity.deviceId,
  });
  return updatedUri.toString();
}
```
## JavaScript Channel Name

The JavaScript channel is used to receive messages from the iFrame to the host application. In this
integration, the channel name is defined as `'parent'`.

**Usage:**

```dart
..addJavaScriptChannel(
   'parent',
    onMessageReceived: (JavaScriptMessage message) {
    _handleUrlChange(message.message, context);
  },
)
```
**On the iFrame (Web) side:**

```javascript
window.parent.postMessage(JSON.stringify({ checkout_id: '123' }), '*');
// or for WebView channel
Parent.postMessage('{"checkout_id":"123"}');
```
## Security Considerations

- Always use HTTPS for URL construction and WebView loading.
- Avoid exposing sensitive tokens in URLs that could be logged or shared.
- Validate all received messages before performing actions on them.

## Use Case

This approach is used in applications embedding third-party services in an iFrame, requiring a
secure and dynamic session initialization along with two-way communication via JavaScript
channels.

## Command Format (JSON Message).

Messages received from the iFrame are JSON objects. Each message contains one or more of the following keys, each representing a distinct command that the application handles separately: selected.

```json
{
"checkout_id": "<ticket_order_id>", 
"link": "<external_url>", 
"pdf": "<base64_encoded_pdf>", 
"goBack": "true"
}
```

These keys may appear individually or together, but each triggers a specific handler.

## Command Description

  - `checkout_id` - Indicates that the user has selected a ticket and is ready to proceed to payment. The application navigates to a checkout page using the given ticket ID.
  - `link` - Represents a URL that should be opened in the device's external browser (e.g., YouTube, Maps).
  - `pdf` - Contains a Base64-encoded PDF (e.g., a ticket). The application decodes and shares this file via local sharing options.
  - `goBack` - A boolean flag (`true` or `false`) that instructs the application to return to the previous screen (e.g., to cancel or exit the ticket selection process).

## Command Handling Strategy

Each command should be processed independently even if multiple commands appear in the same message.

- **Messages may arrive in any order.**
- **Commands are optional and can be sent one at a time.**
- **No assumption should be made about message sequence or completeness.**

## Security ad Validation

- **Ensure all received messages are valid JSON and keys are sanitized before use** 
- **Always validate the content of `checkout_id`, `link`, and `pdf` before executing any action**
- **Use secure connections (HTTPS) and validate external links before opening. selected**

## Use Cases

- **Online event booking and checkout** 
- **External redirection to media or map services** 
- **Sharing event tickets via PDF** 
- **Providing a back action to gracefully exit the flow**



### 1.9.2. Support & Troubleshooting

For additional assistance or to report issues, please contact your designated EasyEvents integration manager. When troubleshooting, include both the `RequestId` and `TraceId` from the error responses to help expedite the resolution process.
