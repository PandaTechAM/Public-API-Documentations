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
    - [1.6.5. Contributing](#165-contributing)
    - [1.6.6. Issues and Support](#166-issues-and-support)
  - [1.7. Stay Updated](#17-stay-updated)


# 1. EasyEvents API Documentation V1.0

## 1.1. Overview

This document outlines the integration process with EasyEvents, focusing on the backend authentication and authorization
mechanisms. Integration is twofold: utilizing API endpoints for secure access and embedding an IFrame within your
application for event querying, ticket purchasing, and management functionalities.

## 1.2. Configuration Details

### 1.2.1. Environments

- **Test Environment:** Make all API requests to the base
  URL: [https://test.events.easypay.am](https://test.events.easypay.am). Access Swagger UI and OpenAPI specifications at
  [https://test.events.easypay.am/swagger/integration](https://test.events.easypay.am/swagger/integration)
- **Production Environment:** API requests should be directed to the base URL:
  [https://events.easypay.am](https://events.easypay.am)

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
- **Method:** `/GET`
- **Description:** For monitoring connection with the server
- **Response:**

```text
"pong"
```

### 1.6.2. Login

- **Path:** `/api/v1/integration/authentication/server/login`
- **Method:** `/POST`
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
- **Method:** `/GET`
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
- **Method:** `/POST`
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

### 1.6.5. Contributing

We encourage contributions to improve our API documentation. If you have suggestions or corrections, please feel free to
open a pull request or an issue.

### 1.6.6. Issues and Support

If you encounter any problems or have questions regarding a specific API, please use the 'Issues' section of this
repository. Our team will do its best to assist you.

## 1.7. Stay Updated

We regularly update our API documentation to reflect the latest changes and improvements. Keep an eye on this repository
for the most current information.

---

Thank you for using PandaTech's APIs! We hope this documentation helps you in your development journey.



# Using iFrame in Xamarin.Forms with WebView

## Overview

This documentation provides guidance on integrating an iFrame within a Xamarin.Forms application using the WebView control. The provided example demonstrates how to interact with the content of the iFrame using JavaScript within a Xamarin.Forms app.

## Code Example

Below is a code snippet illustrating how to utilize an iFrame in Xamarin.Forms:

```csharp
<WebView x:Name="View"

         Source="https://iframe.pandatech.it"

         Loaded="View_OnLoaded">
</WebView>
```

```csharp
// Event handler for when the WebView has loaded
private void View_OnLoaded(object? sender, EventArgs e)
{
    // Set the title to indicate that the WebView is loaded
    Data = "any JSON data from iframe";

    // Set local storage item 'token' with AccessTokenSignature value
    View.EvaluateJavaScriptAsync("window.localStorage.setItem('token', '" + AccessTokenSignature + "');");
}
//get data from iframe
async Task Start()
{
    // Create a periodic timer
    var timer = new PeriodicTimer(TimeSpan.FromSeconds(0.1));

    // Continuously evaluate JavaScript in the WebView
    while (await timer.WaitForNextTickAsync())
    {
        try
        {
            // Evaluate JavaScript function 'window.getMessage()' in the WebView
            var message = await View.EvaluateJavaScriptAsync("window.getMessage();");

           //get JSON data from iframe
            Data = message ?? Data;
        }
        catch (Exception e)
        {
            // Display any errors occurred during evaluation
            Data = e.ToString();
        }
    }
}




```

## Iframe Example

On iframe example, we have a simple HTML form with an iframe that contains a simple form. When the button is clicked, the iframe sends a data to the parent window using the `window.getMessage({ price, id, commission })` method.

At the top of the form you can see token value from local storage.

```html

```

