# 1. SampleProject API Documentation V1.0

- [1. SampleProject API Documentation V1.0](#1-sampleproject-api-documentation-v10)
  - [1.1. Overview](#11-overview)
  - [1.2. Base URL](#12-base-url)
  - [1.3. Common Response Codes](#13-common-response-codes)
  - [1.4. Error Message Format](#14-error-message-format)
    - [1.4.1. Error Fields Explained](#141-error-fields-explained)
    - [1.4.2. Handling Errors](#142-handling-errors)
  - [1.5. Authentication and Authorization](#15-authentication-and-authorization)
    - [1.5.1. User Authentication](#151-user-authentication)
    - [1.5.2. Obtaining an Access Token](#152-obtaining-an-access-token)
    - [1.5.3. Using the Token](#153-using-the-token)
    - [1.5.4. Token Expiration and Renewal](#154-token-expiration-and-renewal)
  - [1.6. API Endpoints](#16-api-endpoints)
    - [1.6.1. Ping](#161-ping)
    - [1.6.2. Login](#162-login)
    - [1.6.3. Payments](#163-payments)
    - [1.6.4. Contributing](#164-contributing)
    - [1.6.5. Issues and Support](#165-issues-and-support)
  - [1.7. Stay Updated](#17-stay-updated)

## 1.1. Overview

The SampleProject API provides programmatic access to our service, allowing users to retrieve, create, update, and delete data related to [specific functionality]. This API is designed to be RESTful and is intended to be used by developers to integrate SampleProject's capabilities into their applications.

## 1.2. Base URL

All API requests in `test` environment should be made to the base URL: [https://pandatech.it](https://pandatech.it)
All API requests in `production` environment should be made to the base URL: [https://pandatech.it](https://pandatech.it)

## 1.3. Common Response Codes

- `200 OK` - Request succeeded.
- `400 Bad Request` - Invalid request format.
- `401 Unauthorized` - Authentication failed or user doesn't have permissions for requested operation.
- `404 Not Found` - Resource not found.
- `500 Internal Server Error` - An error occurred in the server.

## 1.4. Error Message Format

Our API uses a standardized error response structure, which differs from the structure used for successful responses. Below is the format of the error message:

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

### 1.4.1. Error Fields Explained

- **TraceId:** A unique identifier for the request, useful for debugging and tracing the request flow.
- **Instance:** Provides context about the API call, including the HTTP method, the client's IP address, and the endpoint accessed.
- **StatusCode:** The HTTP status code associated with the error (e.g., `400` for bad requests).
- **Type:** A brief description of the error type, such as `BadRequestException`.
- **Errors:** A detailed breakdown of specific errors encountered. This section can include multiple key-value pairs, with each key representing a field that caused an error and its corresponding error message.
- **Note:** The `Errors` object can contain multiple entries, and the fields listed will depend on the nature of the error.
- **Message:** A general message describing the error, often indicating why the request was invalid or could not be processed.

### 1.4.2. Handling Errors

When you encounter an error response:

- Check the `StatusCode` to understand the nature of the error (e.g., client-side or server-side error).
- Refer to the `Errors` object for detailed information about what went wrong. This can guide you in correcting the request.
- Use the `TraceId` for any debugging or when contacting support for assistance.

This standardized error structure is designed to provide comprehensive and actionable information to help you quickly identify and resolve issues in your API requests.

## 1.5. Authentication and Authorization

### 1.5.1. User Authentication

To use the API, you must have a registered user account in the system. It's crucial to `store your credentials securely`.

### 1.5.2. Obtaining an Access Token

- To receive an access token, make a call to the `api/v1/login` endpoint using your credentials.
- Upon successful authentication, you will receive a token.

### 1.5.3. Using the Token

- This token is stored in cookies and must be included in the cookie header of each API call.
- The token is refreshed automatically with each API call, extending its validity.

### 1.5.4. Token Expiration and Renewal

- The token is not valid indefinitely. Over time, it will expire.
- If you receive a `401 Unauthorized` error, it indicates that your token has expired.
- In case of a `401` error, you should re-authenticate using the `api/v1/login` endpoint to obtain a new token and proceed with your API calls.

## 1.6. API Endpoints

### 1.6.1. Ping

- **Path:** `api/v1/ping`
- **Method:** `/GET`
- **Description:** For monitoring connection with the server
- **Response Example:**

```json
  {
     "pong"
  }
```

### 1.6.2. Login

- **Path:** `api/v1/login`
- **Method:** `/POST`
- **Description:** Login for authentication and authorization retrieval.

### 1.6.3. Payments

- **Path:** `/payments`
- **Method:** `/GET`
- **Description:** Retrieves payments
- **Response Example:**

```json
{
  "items": [
    { "id": 1, "name": "Item 1", "description": "Description of Item 1" },
    { "id": 2, "name": "Item 2", "description": "Description of Item 2" }
  ]
}
```

- **Additional notes**

### 1.6.4. Contributing

We encourage contributions to improve our API documentation. If you have suggestions or corrections, please feel free to open a pull request or an issue.

### 1.6.5. Issues and Support

If you encounter any problems or have questions regarding a specific API, please use the 'Issues' section of this repository. Our team will do its best to assist you.

## 1.7. Stay Updated

We regularly update our API documentation to reflect the latest changes and improvements. Keep an eye on this repository for the most current information.

---

Thank you for using PandaTech's APIs! We hope this documentation helps you in your development journey.
