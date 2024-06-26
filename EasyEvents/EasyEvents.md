- [1. EasyEvents API Documentation V1.0](#1-easyevents-api-documentation-v10)
  - [1.1. Overview](#11-overview)
  - [1.2. Configuration Details](#12-configuration-details)
    - [1.2.1. Environments](#121-environments)
  - [1.3. Common Response Codes](#13-common-response-codes)
  - [1.4. Error Handling](#14-error-handling)
    - [1.4.1. Key Points](#141-key-points)
  - [1.5. Authentication \& Authorization](#15-authentication--authorization)
    - [1.5.1. Headers](#151-headers)
    - [1.5.2. Authentication Process](#152-authentication-process)
      - [1.5.2.1. Server Authentication](#1521-server-authentication)
      - [1.5.2.2. User Authentication](#1522-user-authentication)
  - [1.6. API Endpoints](#16-api-endpoints)
    - [1.6.1. Ping](#161-ping)
    - [1.6.2. Login](#162-login)
    - [1.6.3. Refresh Token](#163-refresh-token)
    - [1.6.4. Sync and Login](#164-sync-and-login)
    - [1.6.5. Ticket order](#165-ticket-order)
    - [1.6.6. Payment](#166-payment)
    - [1.6.7. Ticket order cancel](#167-ticket-order-cancel)
  - [1.7. IFrame Integration](#17-iframe-integration)
    - [1.7.1. Implementation Steps](#171-implementation-steps)
    - [1.7.2. Xamarin.Forms WebView Example](#172-xamarinforms-webview-example)
  - [1.8. Contributing](#18-contributing)
  - [1.9. Issues and Support](#19-issues-and-support)
  - [1.10. Stay Updated](#110-stay-updated)

# 1. EasyEvents API Documentation V1.0

## 1.1. Overview

This document outlines the integration process with EasyEvents, focusing on the backend authentication and authorization
mechanisms. Integration is twofold: utilizing API endpoints for secure access and embedding an IFrame within your
application for event querying, ticket purchasing, and management functionalities.

## 1.2. Configuration Details

### 1.2.1. Environments

- **Test Environment:**

  - Access APIs at [https://qabeevents.easypay.am](https://qabeevents.easypay.am).
  - Access Swagger UI and OpenAPI specifications at [https://qabeevents.easypay.am/swagger/integration](https://qabeevents.easypay.am/swagger/integration)
  - Access IFrame at [https://qaiframe.easypay.am](https://qaiframe.easypay.am)

- **Production Environment:**
  - Access APIs at [https://events.easypay.am](https://events.easypay.am)
  - Access IFrame at [https://iframe.easypay.com](https://iframe.easypay.com)

## 1.3. Common Response Codes

- `200 OK` - The request was successful.
- `400 Bad Request` - The request was improperly formatted or contained invalid parameters.
- `401 Unauthorized` - Authentication failed.
- `403 Forbidden` - The user does not have permission to access the requested resource.
- `404 Not Found` - The requested resource was not found.
- `500 Internal Server Error` - An error occurred in the server.

## 1.4. Error Handling

Errors return a standardized JSON structure for consistency and ease of debugging:

```json
{
  "TraceId": "Unique request identifier",
  "Instance": "API call context",
  "StatusCode": "HTTP status code",
  "Type": "Error type",
  "Errors": {
    "field": "Error message"
  },
  "Message": "General error description"
}
```

Real-world example:

```json
{
  "TraceId": "0HMVFE0A284AM:00000001",
  "Instance": "POST - 164.54.144.23:443/users/register",
  "StatusCode": 400,
  "Type": "BadRequestException",
  "Errors": {
    "email": "email_address_is_not_in_a_valid_format",
    "password": "password_must_be_at_least_8_characters_long"
  },
  "Message": "the_request_was_invalid_or_cannot_be_otherwise_served."
}
```

### 1.4.1. Key Points

- **TraceId:** A unique identifier for the request, useful for debugging and tracing the request flow.
- **Instance:** Provides context about the API call, including the HTTP method, the client's IP address, and the
  endpoint accessed.
- **StatusCode:** The HTTP status code associated with the error (e.g., `400` for bad requests).
- **Type:** A brief description of the error type, such as `BadRequestException`.
- **Errors:** A detailed breakdown of specific errors encountered. This section can include multiple key-value pairs,
  with each key representing a field that caused an error and its corresponding error message.
- **Note:** The `Errors` object can contain multiple entries, and the fields listed will depend on the nature of the
  error.
- **Message:** A general message describing the error, often indicating why the request was invalid or could not be
  processed.

Ensure to check the `StatusCode` and `Errors` object for debugging and resolving issues. Utilize the `TraceId` when
seeking support.

## 1.5. Authentication & Authorization

### 1.5.1. Headers

Our Identity server relies on specific headers for authentication and contextual information. Below is a detailed explanation of each header and its usage:

| Header Name       | Type        | Description                                                                           | Example                                |
| ----------------- | ----------- | ------------------------------------------------------------------------------------- | -------------------------------------- |
| `Client-Type`     | Required    | Identifies the client device type. Accepts predefined numeric values.                 | `4` (Windows)                          |
| `Authorization`   | Conditional | Required unless `Client-Type` is `1`. Contains the access token signature.            | `4bba6963-0bc4-4e6c-a17d-71fd1582c59a` |
| `Refresh-Token`   | Conditional | Required unless `Client-Type` is `1`. Contains the refresh token signature.           | `4bba6963-0bc4-4e6c-a17d-71fd1582c59a` |
| `Device`          | Conditional | Required unless `Client-Type` is `1`. Unique identifier for the client device.        | `4e6c-0bc4` (Any unique id)            |
| `Device-Name`     | Optional    | Name of the client device. Provides additional context for analytics.                 | `Iphone 15 Pro Max`                    |
| `Latitude`        | Optional    | Geographic latitude of the client device. Used for location-based services.           | `42.231`                               |
| `Longitude`       | Optional    | Geographic longitude of the client device. Used for location-based services.          | `42.231`                               |
| `Accuracy`        | Optional    | Geolocation accuracy of the client device. Enhances the reliability of location data. | `29.2`                                 |
| `Accept-Language` | Optional    | Preferred language for the client. Defaults to Armenian if not specified.             | `en-US`                                |

- `Client-Type` header accepts values from 1 to 7 only.

1. Browser
2. Ios
3. Android
4. Windows
5. Mac
6. Linux
7. Other

- `Client-Type`: This header is crucial for identifying the type of client that is making the request. Valid values range from 1 to 7 as shown above, each representing a different platform. In browser contexts (`Client-Type` 1), authentication tokens are managed via `Secure` and `HttpOnly` cookies, negating the need for explicit `Authorization`, `Refresh-Token`, and `Device` headers.

- `Authorization` and `Refresh-Token`: These headers are pivotal for managing session lifecycles. The `Authorization` header carries the access token for authenticating API requests, while the `Refresh-Token` header is used when requesting new access tokens upon expiry.

- `Device`, `Device-name`, `Latitude`, `Longitude`, and `Accuracy`: These headers provide additional context about the client device, enhancing the API's ability to deliver personalized and location-aware responses. While optional, providing these details can improve user experience and service quality.

- `Accept-Language:` This header allows the client to specify a preferred language, enabling localized responses from the API. The API currently supports Armenian (`hy-AM`), Russian (`ru-RU`), and English (`en-US`), with Armenian as the default language if this header is not provided.

Please ensure these headers are correctly implemented in your API requests to ensure seamless authentication and a tailored user experience with us.

### 1.5.2. Authentication Process

The authentication process is essential for securing access to the API and is designed to ensure that both
your server and your system's users can interact securely with the API.

#### 1.5.2.1. Server Authentication

1. **Initial Login** To authenticate your server, send a `POST` request
   to `/api/v1/integration/authentication/server/login`
   with your server's credentials. It's imperative to store these credentials securely. The response will include both
   an access token and a refresh token.
   - **Access Token:** Grants temporary access to the API. Typically valid for 10 minutes, but this duration is
     configurable and specified in the login response.
   - **Refresh Token:** Used to obtain new access tokens once the current access token expires. Has a much longer
     validity period than the access token.
2. **Token Refresh** When the access token expires, use the refresh token to request new tokens. Send a `GET` request to
   `/api/v1/integration/authentication/refresh-token` with the refresh token in the `Refresh-Token` header. The response
   will include new access and refresh tokens.

#### 1.5.2.2. User Authentication

To authenticate your system's users, follow these steps:

1. **Server Prerequisite** Ensure that your server is authenticated and has obtained an access token.
2. **User Login** Send a `POST` request to `/api/v1/integration/authentication/sync-and-login` with the user's full
   information. with the necessary user information. This step keeps the user's details up-to-date in the EasyEvents
   system. The response includes user-specific access and refresh tokens.
3. **Token Management** Transfer the user-specific tokens securely to your client application. The client application
   will provide these tokens to the IFrame, enabling it to make API requests on behalf of the user, such as querying
   events, purchasing tickets, and managing ticket views.
4. **IFrame Integration** The IFrame, embedded in your application, will handle token refreshes for user sessions,
   ensuring uninterrupted access to EasyEvents functionalities.

This authentication workflow ensures a secure and seamless integration with EasyEvents, allowing both your server and
your users to interact with the API confidently. For more detailed information on API endpoints and additional
functionalities, refer to the respective sections below.

## 1.6. API Endpoints

### 1.6.1. Ping

- **Path:** `/above-board/ping`
- **Method:** `GET`
- **Description:** For monitoring connection with the server
- **Response:**

```text
"pong"
```

### 1.6.2. Login

- **Path:** `/api/v1/integration/authentication/server/login`
- **Method:** `POST`
- **Description:** Login for authentication and authorization retrieval
- **Request:**

```json
{
  "email": "easyserver@easypay.am",
  "password": "Qwerty123"
}
```

- **Response:**

```json
{
  "accessTokenSignature": "132d5e92-a17f-427e-b5fd-d1892c95eaf7",
  "accessTokenExpiration": "2024-02-22T12:36:17.6088053Z",
  "refreshTokenSignature": "e384b714-86e3-4cde-a658-00a321045735",
  "refreshTokenExpiration": "2024-02-23T05:06:17.6088053Z",
  "userId": "1",
  "counterpartyId": "1",
  "counterpartyTypes": [3],
  "forcePasswordChange": false,
  "permissions": []
}
```

- **Possible Errors:**
  - `400 Bad Request` - you_cannot_login_with_this_device.
  - `400 Bad Request` - wrong_credentials.
  - `400 Bad Request` - Client type is not found in request header.
  - `400 Bad Request` - Client type is not valid.
  - `400 Bad Request` - Unique id per device is required.
  - `400 Bad Request` - Device name is required.
  - `400 Bad Request` - username_invalid_format.
  - `400 Bad Request` - password_min_requirements_not_met.

Note: You can safely ignore `permissions`, `forcePasswordChange`, `counterpartyTypes`, `counterpartyId` and `userId`
fields.

### 1.6.3. Refresh Token

- **Path:** `/api/v1/integration/authentication/refresh-token`
- **Method:** `GET`
- **Description:** Refresh tokens for authentication and authorization retrieval
- **Request:** Make sure that `Refresh-Token` header is set with the refresh token signature
- **Response:**

```json
{
  "accessTokenSignature": "132d5e92-a17f-427e-b5fd-d1892c95eaf7",
  "accessTokenExpiration": "2024-02-22T12:36:17.6088053Z",
  "refreshTokenSignature": "e384b714-86e3-4cde-a658-00a321045735",
  "refreshTokenExpiration": "2024-02-23T05:06:17.6088053Z",
  "userId": "1",
  "counterpartyId": "1",
  "counterpartyTypes": [3],
  "permissions": []
}
```

- **Possible Errors:**
  - `400 Bad Request` - Client type is not found in request header.
  - `400 Bad Request` - Client type is not valid.
  - `400 Bad Request` - refresh_token_is_required.
  - `400 Bad Request` - refresh_token_is_invalid.
  - `400 Bad Request` - refresh_token_is_expired.
  - `400 Bad Request` - unique_id_per_device_is_required.
  - `400 Bad Request` - you_cannot_refresh_token_with_this_device.

Note: You can safely ignore `permissions`, `counterpartyTypes`, `counterpartyId` and `userId` fields. Be careful
with `refreshTokenSignature` as it will be changed after each refresh.

Hint: You can create middleware in your server to refresh token automatically when access token is expired, without
cancelling any call.

### 1.6.4. Sync and Login

- **Path:** `/api/v1/integration/authentication/sync-and-login`
- **Method:** `POST`
- **Description:** Login for authentication and authorization retrieval and update your user in EasyEvents
- **Request:**

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

Required fields in the request:

- `externalUserId` - Unique identifier of the user in your system
- `phoneNumber` or `email` - At least one of them is required. Note that PhoneNumber should be formatted as
  `(374)93919101` and email should be in a valid format.
- `firstName` - User's first name
- `lastName` - User's last name
- `middleName` - User's father name
- `clientType` - Client device type. Accepts values from 1 to 7 only. Refer values in the `Supported headers` section
- `uniqueIdPerDevice` - Client device unique identifier

All other fields are optional.

- **Response:**

```json
{
  "accessTokenSignature": "132d5e92-a17f-427e-b5fd-d1892c95eaf7",
  "accessTokenExpiration": "2024-02-22T12:36:17.6088053Z",
  "refreshTokenSignature": "e384b714-86e3-4cde-a658-00a321045735",
  "refreshTokenExpiration": "2024-02-23T05:06:17.6088053Z"
}
```

- **Possible Errors:**
  - `400 Bad Request` - Invalid phone number format.
  - `400 Bad Request` - Invalid email format.
  - `400 Bad Request` - Either Email or Phone Number is required.
  - `400 Bad Request` - Invalid Armenian SSN format.
  - `400 Bad Request` - user_is_disabled_or_deleted.
  - `400 Bad Request` - you_cannot_login_with_this_device.

Note: Access token and refresh token is required to be transferred to your client which will provide it to IFrame. This
token is not related to the server's token and will be refreshed by the IFrame.

### 1.6.5. Ticket order

- **Path:** `/api/v1/integration/order/{ticketOrderId}`
- **Method:** `GET`
- **Description:** After a user clicks the order button in the IFrame, a `TicketOrderId` will be sent in the return message from the IFrame. This ID should be provided to this endpoint by the server to retrieve the order details. For more information about the IFrame, please refer to the section below.
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

  [Description("ԷՅՉ-ԷՍ-ԲԻ-ՍԻ ԲԱՆԿ ՀԱՅԱՍՏԱՆ")]
  HSBCBankArmenia,
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

### 1.6.6. Payment

- **Path:** `/api/v1/integration/payment`
- **Method:** `POST`
- **Description:** After the user successfully pays the order in full, the payment information should be sent to this endpoint. `ExternalSystemPaymentId` is a unique identifier for the payment in your system.
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

- **Possible Errors:**
  - `400 Bad Request` - Invalid price. Please try again with actual price
  - `400 Bad Request` - Invalid commission. Please try again with actual commission
  - `400 Bad Request` - Duplicate `ExternalSystemPaymentId`

### 1.6.7. Ticket order cancel

- **Path:** `/api/v1/integration/order/{ticketOrderId}`
- **Method:** `DELETE`
- **Description:** This endpoint is designed to cancel an existing ticket order identified by the `{ticketOrderId}`. It is particularly useful in scenarios where a user decides not to proceed with the ticket purchase or exits the payment process prematurely. Upon successful execution, the endpoint releases any seats reserved as part of the order, making them available for future purchases. This action is irreversible; once an order is canceled, it cannot be reinstated.
- **Request:** /api/v1/integration/order/asd
- **Response:** `200` Ok

- **Possible Errors:**
  - `400 Bad Request` - It's impossible to cancel already paid ticket order.
  - `404 Not Found`

## 1.7. IFrame Integration

### 1.7.1. Implementation Steps

Integrating an IFrame within your application provides a seamless experience for event querying, ticket purchasing, and management functionalities. This section focuses on the implementation details of embedding the IFrame and ensuring smooth interaction between your application and the IFrame content.

1. **Embedding the IFrame:** Insert the IFrame into your application's designated UI component. Ensure the IFrame's `src` attribute points to the correct URL provided by EasyEvents. Adjust the width and height to fit your UI design.

```html
<!-- Example HTML snippet for embedding the IFrame -->
<iframe
  id="easyEventsIframe"
  src="https://iframe.easyevents.com"
  style="width:100%; height:100%; border:none;"
></iframe>
```

2. **Setting the Access Token:** Prioritize security by storing the access token in the local storage of the IFrame rather than sending it through postMessage. This ensures the token is securely communicated and accessible by the IFrame for authorization purposes.

```js
// Example JavaScript code for setting the access token in the IFrame's local storage
window.localStorage.setItem("token", JSON.stringify(  {
    accessToken: "string";
    accessTokenExpiration: "Date";
    refreshToken: "string";
    refreshTokenExpirationTime: "Date";
    deviceId "string";
  }));
```

1. **Closing the IFrame:** To close the IFrame, either when a user completes an action or chooses to exit, your application should listen for a specific message or trigger from the IFrame. This can be done using the `window.postMessage` API for secure cross-origin communication. When the IFrame needs to be closed, it will send a message (`-1` or a predefined string ID), which your application will listen for and act upon by removing or hiding the IFrame.

```js
function startPollingMessages() {
  const intervalId = setInterval(() => {
    // Assuming 'getMessage' retrieves and clears the first message from the queue
    const message = window.getMessage();

    if (message !== undefined) {
      switch (message) {
        case '-1':
          // User decided not to buy a ticket, close the IFrame
          closeIFrame();
          clearInterval(intervalId); // Stop the polling
          break;
        default:
          // Handle the order based on the message (e.g., order ID)
          handleOrder(message);
          clearInterval(intervalId); // Stop the polling
          break;
      }
    }
  }, 100); // Polling every 100ms

  return () => clearInterval(intervalId); // Return a cleanup function to stop polling when needed
}

function closeIFrame() {
  // Logic to close or hide the IFrame
  const iframe = document.getElementById('easyEventsIframe');
  if (iframe) {
    iframe.style.display = 'none'; // or use iframe.remove() to remove it from the DOM
  }
}

function handleOrder(orderId) {
  // Handle the order based on orderId
  console.log('Handle order with ID:', orderId);
  // Implement the order handling logic here
}

// Start polling for messages from the IFrame
const stopPolling = startPollingMessages();

// Later, if you need to stop polling for messages
// stopPolling();
```

### 1.7.2. Xamarin.Forms WebView Example

For applications developed using Xamarin.Forms, integrating the IFrame can be achieved using the `WebView` control. Below is an example that demonstrates how to embed the IFrame within a Xamarin.Forms page and interact with it. This example is **not** following Xamarin best coding practices its only for illustration purposes:

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

We encourage contributions to improve our API documentation. If you have suggestions or corrections, please feel free to
open a pull request or an issue.

## 1.9. Issues and Support

If you encounter any problems or have questions regarding a specific API, please use the 'Issues' section of this
repository. Our team will do its best to assist you.

## 1.10. Stay Updated

We regularly update our API documentation to reflect the latest changes and improvements. Keep an eye on this repository
for the most current information.

---

Thank you for using PandaTech's APIs! We hope this documentation helps you in your development journey.
