# 1. EasyTransact API Documentation V1.0

- [1. EasyTransact API Documentation V1.0](#1-easytransact-api-documentation-v10)
  - [1.1. Overview](#11-overview)
  - [1.2. Base URL](#12-base-url)
  - [1.3. Common Response Codes](#13-common-response-codes)
  - [1.4. Authentication and Authorization](#14-authentication-and-authorization)
    - [1.4.1. User Authentication](#141-user-authentication)
    - [1.4.2. Obtaining an Access Token](#142-obtaining-an-access-token)
    - [1.4.3. Using the Token](#143-using-the-token)
    - [1.4.4. Token Expiration and Renewal](#144-token-expiration-and-renewal)
  - [1.5. Authentication](#15-authentication)
    - [1.5.1. Login](#151-login)
  - [1.6. Transaction Operations](#16-transaction-operations)
    - [1.6.1. Export Transactions](#161-export-transactions)
    - [1.6.2. Get Transactions](#162-get-transactions)
  - [1.7. Provider Information](#17-provider-information)
    - [1.7.1. Get Providers](#171-get-providers)
  - [1.8. Transaction Verification and Payment](#18-transaction-verification-and-payment)
    - [1.8.1. Check](#181-check)
    - [1.8.2. Pay](#182-pay)
    - [1.8.3. Balance](#183-balance)
  - [1.9. Contributing](#19-contributing)
  - [1.10. Issues and Support](#110-issues-and-support)
  - [1.11. Stay Updated](#111-stay-updated)

## 1.1. Overview

The EasyTransact API provides programmatic access to our service, allowing users to retrieve payment providers and make
payments. This API is designed to be RESTful and is intended to be used by developers to integrate EasyTransact's
capabilities into their applications.

## 1.2. Base URL

All API requests should be made to the base URL: [https://betransact.easypay.am](https://betransact.easypay.am)

## 1.3. Common Response Codes

- `200 OK` - Request succeeded.
- `400 Bad Request` - Invalid request format.
- `401 Unauthorized` - Authentication failed or user doesn’t have permissions for requested operation.
- `404 Not Found` - Resource not found.
- `500 Internal Server Error` - An error occurred in the server.

## 1.4. Authentication and Authorization

### 1.4.1. User Authentication

To use the API, you must have a registered user account in the system. It's crucial to
`store your credentials securely`.

### 1.4.2. Obtaining an Access Token

- To receive an access token, make a call to the `api/v1/login` endpoint using your credentials.
- Upon successful authentication, you will receive a token.

### 1.4.3. Using the Token

- This token is should be always included in the header on each call.
- The token is refreshed automatically with each API call, extending its validity.

### 1.4.4. Token Expiration and Renewal

- The token is not valid indefinitely. Over time, it will expire.
- If you receive a `401 Unauthorized` error, it indicates that your token has expired.
- In case of a `401` error, you should re-authenticate using the `api/v1/login` endpoint to obtain a new token and
  proceed with your API calls.

## 1.5. Authentication

### 1.5.1. Login

- **Endpoint:** `POST /api/v1/login`
- **Description:** Authenticate the user and obtain a token for API access.
- **Request:**

  ```json
  {
    "Username": "string",
    "Password": "string"
  }
  ```

- **Response:**

  ```json
  {
    "responseData": {
      "data": {
        "userId": "61adcbf5-1755-4b64-ae7b-ccbcfa647609",
        "token": "f6816910-914e-48a1-a40a-06cb6aead7e9",
        "fullName": "Panda",
        "username": "Admin",
        "role": "SuperAdmin"
      }
    },
    "success": true,
    "message": "",
    "responseStatus": "Ok"
  }
  ```

## 1.6. Transaction Operations

### 1.6.1. Export Transactions

- **Endpoint:** `GET /api/v1/available-user-transactions-export`
- **Description:** Export user transaction data in following formats: `.csv`, `.xlsx`, `.pdf`.
- **Query parameters** `dataRequest`, `exportType`
- **Request
  ** https://betransact.easypay.am/api/v1/user-transactions-export?token=d62266b3-c5bb-4b82-bca4-c22b8e58337a&dataRequest=%7B%7D&exportType=CSV

### 1.6.2. Get Transactions

- **Endpoint:** `GET /api/v1/available-user-transactions`
- **Description:** Retrieve user transaction data.
- **Query parameters** `page`, `pageSize`
- **Request** https://betransact.easypay.am/api/v1/available-user-transactions?page=1&pageSize=5000
- **Response**

  ```json
  {
    "success": true,
    "message": "string",
    "responseStatus": "Ok",
    "responseData": {
      "data": [
        {
          "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "providerId": 0,
          "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "inputs": "string",
          "creationDate": "2023-10-18T11:21:43.757Z",
          "amount": 0,
          "comment": "string",
          "getProviderDto": {
            "id": 0,
            "providerName": "string"
          }
        }
      ],
      "page": 0,
      "pageSize": 0,
      "totalCount": 0
    }
  }
  ```

## 1.7. Provider Information

### 1.7.1. Get Providers

- **Endpoint:** `GET /api/v1/user-providers`
- **Language:** Provide language in header `language` "am", "ru", "en" to get proper hint
- **Description:** Retrieve information about available providers.
- **Response:**

  ```json
  {
  "responseData": {
    "data": [
      {
        "id": 17667,
        "input1Object": {
          "hint": "Հեռախոսահամար",
          "minLength": 12,
          "maxLength": 12,
          "regexp": "",
          "prefix": "+374"
        },
        "input2Object": null,
        "input3Object": null,
        "input4Object": null,
        "isEnabled": true,
        "maxAmount": 200000,
        "minAmount": 10,
        "name": "Easy Pay Wallet New For Getway"
      },
      {
        "id": 18473,
        "input1Object": {
          "hint": "Քարտի համար",
          "minLength": 16,
          "maxLength": 16,
          "regexp": "",
          "prefix": ""
        },
        "input2Object": null,
        "input3Object": null,
        "input4Object": null,
        "isEnabled": true,
        "maxAmount": 399000,
        "minAmount": 100,
        "name": "Փոխանցում քարտին coin - Bitcoin Armenia"
      },
      {
        "id": 18474,
        "input1Object": {
          "hint": "Հեռախոսահամար",
          "minLength": 12,
          "maxLength": 12,
          "regexp": "",
          "prefix": "+374"
        },
        "input2Object": null,
        "input3Object": null,
        "input4Object": null,
        "isEnabled": true,
        "maxAmount": 200000,
        "minAmount": 10,
        "name": "Փոխանցում դրամապանակին coin - Bitcoin Armenia"
      }
    ]
  },
  "success": true,
  "message": "",
  "responseStatus": "Ok"
  }
  ```

## 1.8. Transaction Verification and Payment

### 1.8.1. Check

- **Endpoint:** `POST /api/v1/check`
- **Description:** Perform pre-transaction checks.
- **Request:**

  ```json
  {
    "providerId": 1,
    "input1": "077777777",
    "input2": null,
    "input3": null,
    "input4": null
  }
  ```

- **Response:**

  ```json
  {
    "success": true,
    "message": "string",
    "responseStatus": "Ok",
    "responseData": {
      "data": "string"
    }
  }
  ```

### 1.8.2. Pay

- **Endpoint:** `POST /api/v1/pay`
- **Description:** Process a payment.
- **Request:**
  ```json
  {
    "amount": 1,
    "providerId": 1,
    "input1": "+37477777777",
    "input2": null,
    "input3": null,
    "input4": null,
    "sessionID": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  }
  ```

- **Response:**
  ```json
  {
    "success": true,
    "message": "string",
    "responseStatus": "Ok",
    "responseData": {
    "data": 0
    }
  }
  ```

### 1.8.3. Balance

- **Endpoint:** `GET /api/v1/balance`
- **Description:** Get agent balance.
- **Request:**

  ```json
  {
    "responseData": {
      "data": {
        "balance": 45348.156
      }
    },
    "success": true,
    "message": "",
    "responseStatus": "Ok"
  }
  ```


### 1.8.4. Get cheque

- **Endpoint:** `GET /api/v1/receipt?transactionId=1`
- **Description:** Get transaction cheque by transaction id.
- **Request:**
  
  ```json
  {
    "success": true,
    "message": "string",
    "responseStatus": "Ok",
    "responseData": {
      "data": [
        "string"
      ]
    }
  }
  ```

## 1.9. Contributing

We encourage contributions to improve our API documentation. If you have suggestions or corrections, please feel free to
open a pull request or an issue.

## 1.10. Issues and Support

If you encounter any problems or have questions regarding a specific API, please use the 'Issues' section of this
repository. Our team will do its best to assist you.

## 1.11. Stay Updated

We regularly update our API documentation to reflect the latest changes and improvements. Keep an eye on this repository
for the most current information.

---

Thank you for using PandaTech's APIs! We hope this documentation helps you in your development journey.
