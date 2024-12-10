- [1. EasyEvents API Documentation](#1-easyevents-api-documentation)
  - [1.1. Overview](#11-overview)
  - [1.2. Environments](#12-environments)
  - [1.3. Common Response Codes](#13-common-response-codes)
  - [1.4. Error Handling](#14-error-handling)
  - [1.5. Authentication \& Authorization](#15-authentication--authorization)
    - [1.5.1 Server Authentication](#151-server-authentication)
    - [1.5.2 HMAC-SHA256 Signature Calculation](#152-hmac-sha256-signature-calculation)
  - [1.6. API Endpoints](#16-api-endpoints)
    - [1.6.1. Ping](#161-ping)
    - [1.6.2. Sync and Login](#162-sync-and-login)
    - [1.6.3. Ticket order](#163-ticket-order)
    - [1.6.4. Payment](#164-payment)
    - [1.6.5. Ticket order cancel](#165-ticket-order-cancel)
  - [1.7. IFrame Integration](#17-iframe-integration)
    - [1.7.1. Implementation Steps](#171-implementation-steps)
    - [1.7.2. Xamarin.Forms WebView Example](#172-xamarinforms-webview-example)
  - [1.8. Contributing](#18-contributing)
  - [1.9. Issues and Support](#19-issues-and-support)
  - [1.10. Stay Updated](#110-stay-updated)

# 1. EasyEvents API Documentation

## 1.1. Overview

This documentation describes how to integrate your backend and frontend applications with the EasyEvents platform. Integration is twofold:

1. **Backend Integration:** Use the provided endpoints to securely authenticate, authorize, and manage orders and payments.
2. **IFrame Integration:** Embed an EasyEvents IFrame into your application to allow event searching, ticket purchasing, and order handling directly within your UI.

**Language Support:**
Requests should include an `Accept-Language` header with a supported ISO language code. Valid options include:

`en-US` for English (US)
`hy-AM` for Armenian (default if none specified or invalid)
`ru-RU` for Russian

## 1.2. Environments

**Test Environment (for development and QA)**

- **Base URL:**
  [https://qabeevents.easypay.am](https://qabeevents.easypay.am).
- **IFrame Base URL:**
  [https://qaiframe.easypay.am](https://qaiframe.easypay.am)
- **OpenAPI Spec:**
  [https://qabeevents.easypay.am/openapi/integration-v1.json](https://qabeevents.easypay.am/openapi/integration-v1.json)
- **Scalar UI (Test Only):**
  [https://qabeevents.easypay.am/scalar/integration-v1](https://qabeevents.easypay.am/docs/integration-v1)
- **Swagger UI (Test Only):**
  [https://qabeevents.easypay.am/swagger/integration-v1](https://qabeevents.easypay.am/swagger/integration-v1)

> **Note:** Scalar UI and Swagger UI are not available in the production environment.

**Production Environment**

- **Base URL:**
  [https://events.easypay.am](https://events.easypay.am)
- **IFrame Base URL**
  [https://iframe.easypay.com](https://iframe.easypay.com)

Use the test environment for initial integration and testing. Once stable and production-ready, switch your base URL to the production environment.

## 1.3. Common Response Codes

- `200 OK` - Request succeeded.
- `400 Bad Request` - Invalid request format or parameters.
- `401 Unauthorized` - Authentication failed.
- `403 Forbidden` - Insufficient permissions.
- `404 Not Found` - Resource not found.
- `500 Internal Server Error` - An unexpected server-side error occurred.

## 1.4. Error Handling

All errors return a standardized JSON structure to aid in debugging:

```json
{
  "RequestId": "Unique request ID",
  "TraceId": "Unique trace ID",
  "Instance": "API context info",
  "StatusCode": 400,
  "Type": "Error type (e.g. BadRequestException)",
  "Errors": {
    "field": "Error message"
  },
  "Message": "General error description"
}
```

**Key Fields:**

- **RequestId / TraceId:** Use these IDs for debugging and support requests.
- **Instance:** Provides contextual details about the API operation.
- **StatusCode:** Matches the HTTP status code.
- **Type:** A short error descriptor.
- **Errors:** Field-specific error details.
- **Message:** A general explanation of the problem.

Use `RequestId` and `TraceId` when contacting support to help with troubleshooting.

## 1.5. Authentication & Authorization

EasyEvents requires server-level authentication and user-level tokens for secure operation.

### 1.5.1 Server Authentication

1. **Server Setup:**
   EasyEvents registers an external user (your server) and provides an API key and a secret key. For each server-to-server request, include the `Authorization` header in the format:
   `HMAC <API_KEY>:<BASE64_SIGNATURE>`

The signature is an HMAC-SHA256 hash computed from relevant request data and your secret key.

2. **External User Sync/Login (User-Level Authentication):**
   Send a `POST` request to `/api/v1/integration/authentication/sync-and-login` with user details. This updates user info on EasyEvents and returns user-specific access and refresh tokens.

3. **Token Management:**
   Pass these user-level tokens to your client application. The client will use them (via the IFrame) to access APIs related to events, tickets, and more. The IFrame handles token refresh automatically.

This design ensures secure interaction at both the server and user levels.

### 1.5.2 HMAC-SHA256 Signature Calculation

To compute the HMAC-SHA256 signature:

1. Concatenate relevant fields (e.g., `externalUserId`, `phoneNumber`, `email`) into a single string.
2. Hash this string using the secret key with HMAC-SHA256.
3. Base64-encode the resulting hash to form the `BASE64_SIGNATURE`.

Example:

```text
apiKey = "ABCD123"
hmacSecret = "Secret"
input = "5464eeaa-df9b-473a-af12-a036b3dfdab2(374)12345678useremail@gmail.com"
```

With your secret key, compute the HMAC hash and then Base64-encode it, finally add it in `Authorization` header:

```text
Authorization = HMAC ABCD123:ccXVrWSaE243axjtQnMUAYyWNMAlv/VzdKNAkzDPvqs=
```

> You can verify results with an online HMAC-SHA256 generator (e.g., [https://easypay.am/hmac-generator](https://easypay.am/hmac-generator)).

## 1.6. API Endpoints

### 1.6.1. Ping

- **Endpoint:** `GET /above-board/ping`
- **Description:** Checks connectivity
- **Response:** `"pong"`

### 1.6.2. Sync and Login

- **Endpoint:** `POST /api/v1/integration/authentication/sync-and-login`
- **Description:** Synchronizes and logs in users, returning user-specific tokens for use by the IFrame.
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
    "socialSecurityNumber": "1234567891",
    "address": "ք․ Երևան, Վարդանի փ․, բն․ Վազգեն",
    "dateOfBirth": "1985-02-22T00:00:00.000Z",
    "gender": 1,
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
    "accessTokenSignature": "132d5e92-a17f-427e-b5fd-d1892c95eaf7",
    "accessTokenExpiration": "2024-02-22T12:36:17.6088053Z",
    "refreshTokenSignature": "e384b714-86e3-4cde-a658-00a321045735",
    "refreshTokenExpiration": "2024-02-23T05:06:17.6088053Z"
  }
  ```

- **Common Errors:**
  - `400 Bad Request` - Invalid phone number format.
  - `400 Bad Request` - Invalid email format.
  - `400 Bad Request` - Either Email or Phone Number is required.
  - `400 Bad Request` - Invalid Armenian SSN format.
  - `400 Bad Request` - user_is_disabled_or_deleted.
  - `400 Bad Request` - you_cannot_login_with_this_device.

### 1.6.3. Ticket order

- **Endpoint:** `GET /api/v1/integration/order/{ticketOrderId}`
- **Description:** After a user clicks the order button in the IFrame, a `TicketOrderId` will be sent in the return
  message from the IFrame. This ID should be provided to this endpoint by the server to retrieve the order details. For
  more information about the IFrame, please refer to the section below.
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

- **Possible Errors:**
  - `404 Not Found`

### 1.6.4. Payment

- **Endpoint:** `POST /api/v1/integration/payment`
- **Description:** Submit payment details after the user completes a ticket purchase. `ExternalSystemPaymentId` is a unique identifier for the payment in your system.
- **Request Body:**

  ```json
  {
    "ticketOrderId": "asd",
    "price": 12000,
    "commission": 200,
    "externalSystemPaymentId": "d97cf534-a7cd-473a-ba86-71bbeb31b872"
  }
  ```

- **Response:** The response will only include a status code.

- **Common Errors:**
  - `400 Bad Request` - Invalid price. Please try again with actual price
  - `400 Bad Request` - Invalid commission. Please try again with actual commission
  - `400 Bad Request` - Duplicate `ExternalSystemPaymentId`

### 1.6.5. Ticket order cancel

- **Endpoint:** `DELETE /api/v1/integration/order/{ticketOrderId}`
- **Description:** Cancels a pending ticket order, releasing reserved seats. It is particularly useful in scenarios where a user decides not to proceed with the ticket purchase or exits the payment
  process prematurely. Upon successful execution, the endpoint releases any seats reserved as part of the order, making
  them available for future purchases. This action is irreversible; once an order is canceled, it cannot be reinstated.
- **Request:** /api/v1/integration/order/asd
- **Response:** The response will only include a status code.

- **Possible Errors:**
  - `400 Bad Request` - It's impossible to cancel already paid ticket order.
  - `404 Not Found`

## 1.7. IFrame Integration

Integrate the EasyEvents IFrame into your front-end application to let users browse events, select tickets, and make purchases without leaving your interface.

### 1.7.1. Implementation Steps

1. **Embedding the IFrame:**
   Insert the IFrame into your application's UI, setting the `src` to the appropriate IFrame URL.

   ```html
   <iframe
     id="easyEventsIframe"
     src="https://iframe.easyevents.com"
     style="width:100%; height:100%; border:none;"
   ></iframe>
   ```

2. **Setting Access Token:**
   Store tokens in IFrame-local storage for secure access. This allows the IFrame to make authorized requests.

   ```js
   window.localStorage.setItem("token", JSON.stringify({
       accessToken: "string";
       accessTokenExpiration: "Date";
       refreshToken: "string";
       refreshTokenExpirationTime: "Date";
       deviceId: "string";
   }));
   ```

3. **Closing the IFrame & Handling Responses:**
   The IFrame uses `postMessage` or a `polling` mechanism to communicate back to your application. Messages might contain:

   - `"-1"` indicating the user canceled the transaction.
   - An order ID for a successfully initiated ticket order.

   In response, hide or remove the IFrame if the user cancels, or handle the order if an ID is received.

   **Example JavaScript for Polling:**

   ```js
   function startPollingMessages() {
     const intervalId = setInterval(() => {
       const message = window.getMessage(); // Custom function retrieving messages
       if (message !== undefined) {
         if (message === '-1') {
           closeIFrame();
         } else {
           handleOrder(message);
         }
         clearInterval(intervalId);
       }
     }, 100);
   }

   function closeIFrame() {
     const iframe = document.getElementById('easyEventsIframe');
     if (iframe) iframe.style.display = 'none';
   }

   function handleOrder(orderId) {
     console.log('Handle order with ID:', orderId);
     // Implement order handling logic
   }
   ```

### 1.7.2. Xamarin.Forms WebView Example

For Xamarin.Forms, use a `WebView` to embed and interact with the IFrame.
(This example is for illustration only, not best-practice code.)

```csharp
async Task Start()
{
    var timer = new PeriodicTimer(TimeSpan.FromSeconds(0.1));

    while (await timer.WaitForNextTickAsync())
    {
        try
        {
            var message = await View.EvaluateJavaScriptAsync("window.getMessage();");

            switch (message)
            {
                case null:
                    // there is no new messages from iframe
                    continue;
                case "-1":
                    // user decided not to buy a ticket
                    CloseIFrame();
                    return; // this will also stop the task. you can use cancellation token instead.
                default:
                    HandleOrder(message);
                    return; // this will also stop the task. you can use cancellation token instead.
            }
        }
        catch (Exception e)
        {
            // handle exceptions.
        }
    }
}

void HandleOrder(string orderNumber)
{
    // get order details from server (e.g. total price and so on)
    // and navigate to payment page
}

void CloseIFrame()
{
    // closing iframe
}


private void View_OnLoaded(object? sender, EventArgs e) // this event will be loaded when the iframe's html is fully loaded.
{
    // Pass token data to the WebView
    View.EvaluateJavaScriptAsync("window.localStorage.setItem('token', '" + JsonSerializer.Serialize(token) + "');");

    // starting task, that is listening messages from the iframe
    tTask = Start();
}

public class Token
{
    public string accessToken { get; set; }
    public DateTime accessTokenExpiration { get; set; }
    public string refreshToken { get; set; }
    public DateTime refreshTokenExpirationTime { get; set; }
    public string deviceId { get; set; }
}
```

## 1.8. Contributing

We welcome contributions and improvements to this documentation. Submit a pull request or open an issue for suggestions.

## 1.9. Issues and Support

For issues or support inquiries, use the repository's “Issues” section. Our team will respond to help resolve your problem.

## 1.10. Stay Updated

We regularly update this documentation. Check the repository for the latest changes, new features, and improvements.

---

Thank you for using PandaTech's APIs! We hope this documentation helps you in your development journey.
