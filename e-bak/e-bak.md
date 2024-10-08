- [1. e-bak API Documentation](#1-e-bak-api-documentation)
  - [1.1. Overview](#11-overview)
  - [1.2. Configuration Details](#12-configuration-details)
    - [1.2.1. Environments](#121-environments)
  - [1.3. Common Response Codes](#13-common-response-codes)
  - [1.4. Error Handling](#14-error-handling)
    - [1.4.1. Key Points](#141-key-points)
  - [1.5. Authentication \& Authorization](#15-authentication--authorization)
    - [1.5.1. User Authentication](#151-user-authentication)
    - [1.5.2. Obtaining an Access Token](#152-obtaining-an-access-token)
    - [1.5.3. Using the Token](#153-using-the-token)
    - [1.5.4. Token Expiration and Renewal](#154-token-expiration-and-renewal)
  - [1.6. API Endpoints](#16-api-endpoints)
    - [1.6.1. Languages](#161-languages)
    - [1.6.2. Estate Types](#162-estate-types)
    - [1.6.3. Ping](#163-ping)
    - [1.6.4. Login](#164-login)
    - [1.6.5. Debt retrieval by Estate](#165-debt-retrieval-by-estate)
    - [1.6.6. Debt retrieval by estate owner unique identifier](#166-debt-retrieval-by-estate-owner-unique-identifier)
    - [1.6.7. Debt Commission](#167-debt-commission)
    - [1.6.8. Debt repayment](#168-debt-repayment)
    - [1.6.9. Search Cities](#169-search-cities)
    - [1.6.10. Search Condominium Association](#1610-search-condominium-association)
    - [1.6.11. Search Buildings Within a Condominium Association](#1611-search-buildings-within-a-condominium-association)
    - [1.6.12. Search Estates Within a Building](#1612-search-estates-within-a-building)
  - [1.7. Contributing](#17-contributing)
  - [1.8. Issues and Support](#18-issues-and-support)
  - [1.9. Stay Updated](#19-stay-updated)


# 1. e-bak API Documentation

## 1.1. Overview

The e-bak API provides programmatic access to our service, allowing users to retrieve debts and repay condominium association fees.
This API is designed to be RESTful and is intended to be used by developers to integrate e-bak's capabilities into their applications.

## 1.2. Configuration Details

### 1.2.1. Environments

- **Test Environment:** Make all API requests to the base
  URL: [https://qabe-ca.pandatech.it](https://qabe-ca.pandatech.it). Access Swagger UI and OpenAPI specifications at
  [https://qabe-ca.pandatech.it/swagger/integration](https://qabe-ca.pandatech.it/swagger/integration)
- **Production Environment:** API requests should be directed to the base URL:
  [https://be.e-bak.am](https://be.e-bak.am)

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

**Example:**

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
- **Instance:** Provides context about the API call, including the HTTP method, the client's IP address, and the endpoint accessed.
- **StatusCode:** The HTTP status code associated with the error (e.g., `400` for bad requests).
- **Type:** A brief description of the error type, such as `BadRequestException`.
- **Errors:** A detailed breakdown of specific errors encountered. This section can include multiple key-value pairs, with each key representing a field that caused an error and its corresponding error message.
- **Message:** A general message describing the error, often indicating why the request was invalid or could not be processed.

Ensure to check the `StatusCode` and `Errors` object for debugging and resolving issues. Utilize the `TraceId` when seeking support.


## 1.5. Authentication & Authorization

### 1.5.1. User Authentication

To use the API, you must have a registered user account in the system. It's crucial to store your credentials securely.

### 1.5.2. Obtaining an Access Token

- To receive an access token, make a call to the `api/v1/integration/login` endpoint using your credentials.
- Upon successful authentication, you will receive a token.

### 1.5.3. Using the Token

- Include this token in the header of each API call.
- The token is refreshed automatically with each API call, extending its validity.

### 1.5.4. Token Expiration and Renewal

- The token will expire over time.
- If you receive a `401 Unauthorized` error, it indicates that your token has expired.
- In case of a `401` error, you should re-authenticate using the `api/v1/integration/login` endpoint to obtain a new token and proceed with your API calls.

## 1.6. API Endpoints

### 1.6.1. Languages

Specify the language via the `Accept-Language` header:

- **Armenian** - "hy-AM"
- **Russian** - "ru-RU"
- **English** - "en-US"

### 1.6.2. Estate Types

```csharp
public enum EstateTypes
{
   Apartment = 0,
   CommercialArea = 1,
   ParkingArea = 2,
   Basement = 3,
   House = 4,
   Other = 5
}
```

### 1.6.3. Ping

- **Path:** `/ping`
- **Method:** `/GET`
- **Description:** For monitoring connection with the server
- **Response Example:**

```json
  {
     "pong"
  }
```

### 1.6.4. Login

- **Path:** `/api/v1/integration/login`
- **Method:** `/POST`
- **Description:** Login for authentication and authorization retrieval.
- **Request**

```json
{
  "login": "easypay@easypay.am",
  "password": "Qwerty123!"
}
```

- **Response**

```json
{
  "accessToken": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "expirationDate": "2023-12-21T08:34:20.996Z"
}
```

### 1.6.5. Debt retrieval by Estate

- **Path:** `/api/v3/integration/debts/{estateId}`
- **Method:** `/GET`
- **Description:** Retrieves estate related all outstanding debts.
- **Note:** `primaryEstateOwnerFullName` can be `null`
- **Response:**

```json
{
  "partnerId": 8,
  "partnerName": "Անուն Ազգանուն",
  "estateId": 1000270,
  "estateType": 5,
  "estateAddress": "Գյուլբենկյան 33, 1",
  "primaryEstateOwnerFullName": "Անուն Ազգանուն",
  "balance": -4900,
  "debts": [
    {
      "debtId": 75,
      "balance": -4900,
      "date": "2024-04-01T00:00:00Z"
    }
  ]
}
```

### 1.6.6. Debt retrieval by estate owner unique identifier

- **Path:** `/api/v3/integration/debts/owner/{uniqueDocumentId}`
- **Method:** `/GET`
- **Description:** Retrieves all outstanding debts by estate owner unique identifier (SSN or Tax Code).
- **Note:** `primaryEstateOwnerFullName` can be `null`
- **Response:**

```json
{
  "estates": [
    {
      "partnerId": 8,
      "partnerName": "Անուն Ազգանուն",
      "estateId": 1000270,
      "estateType": 3,
      "estateAddress": "Գյուլբենկյան 1, 1",
      "primaryEstateOwnerFullName": "Անուն Ազգանուն",
      "balance": -5000,
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
      "partnerName": "Անուն Ազգանուն",
      "estateId": 1000271,
      "estateType": 0,
      "estateAddress": "Գյուլբենկյան 2, 1",
      "primaryEstateOwnerFullName": "Անուն Ազգանուն",
      "balance": -15000,
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
      "partnerName": "Անուն Ազգանուն",
      "estateId": 1000272,
      "estateType": 4,
      "estateAddress": "Գյուլբենկյան 2, 1",
      "primaryEstateOwnerFullName": "Անուն Ազգանուն",
      "balance": 0,
      "debts": []
    }
  ]
}
```

### 1.6.7. Debt Commission

- **Path:** `/api/v2/integration/debts/commission`
- **Method:** `/POST`
- **Description:** Commission for repayment of debts. The commission is calculated based on the amount with commission percentage for each estate. The commission is calculated and returned in the response.
- **Request:**

```json
[
  {
    "estateId": 1000182,
    "amount": 1000,
    "debtId": null
  }
]
```

- **Response:**

```json
{
  "commission": 10
}
```

### 1.6.8. Debt repayment

- **Path:** `/api/v2/integration/debts/payments`
- **Method:** `/POST`
- **Description:** Repaying Debts by Estate. The e-bak system initiates the repayment of debts following the First-In-First-Out (FIFO) method. To target a specific debt for repayment, include its `debtId` in your request. If the `debtId` is left blank, the system defaults to the FIFO method, repaying the oldest debt first and then proceeding sequentially.

- **Request:**

```json
{
  "debtCommissionRequestModels": [
    {
      "estateId": 1654896,
      "amount": 452.25,
      "debtId": 24
    }
  ],
  "outerPaymentId": "3fa85f64",
  "commission": 10
}
```

- **Response:**

```json
{
  "debtCommissionRequestModels": [
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
  "outerPaymentId": "3fa85f64",
  "commission": 10
}
```

### 1.6.9. Search Cities

- **Path:** `/api/v2/integration/search/cities`
- **Method:** `/GET`
- **Description:** Retrieves all cities.
- **Request:** https://be-ca.pandatech.it/api/v2/integration/search/cities
- **Response:**

```json
{
  "values": [
    {
      "id": 1,
      "name": "Երևան"
    },
    {
      "id": 2,
      "name": "Աբովյան"
    },
    {
      "id": 3,
      "name": "Ագարակ"
    },
    {
      "id": 4,
      "name": "Ալավերդի"
    }
  ]
}
```

### 1.6.10. Search Condominium Association

- **Path:** `/api/v3/integration/search/condominium-associations`
- **Method:** `/GET`
- **Description:** Retrieves all condominiums with city query, pagination, and total count.
- **Request:** https://becapublicapi.pandatech.it/api/v3/search/condominium-associations?cityId=1Page=1&PageSize=4
- **Response:**

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

### 1.6.11. Search Buildings Within a Condominium Association

- **Path:** `/api/v2/integration/search/buildings`
- **Method:** `/GET`
- **Description:** Retrieves all buildings within the condominium association with pagination and total count.
- **Request:** https://becapublicapi.pandatech.it/api/v2/search/buildings?PartnerId=a1&Page=1&PageSize=1
- **Response:**

```json
{
  "values": [
    {
      "id": 3,
      "address": "Tumanyan 27"
    }
  ],
  "totalCount": 746
}
```

### 1.6.12. Search Estates Within a Building

- **Path:** `/api/v2/integration/search/estates`
- **Method:** `/GET`
- **Description:** Retrieves all estates within the building with types, pagination, and total count.
- **Request:** https://becapublicapi.pandatech.it/api/v2/search/estates?BuildingId=jk2&Page=1&PageSize=1
- **Response:**

```json
{
  "values": [
    {
      "type": 0,
      "id": 13548785,
      "address": "21/2"
    }
  ],
  "totalCount": 150
}
```

## 1.7. Contributing

We encourage contributions to improve our API documentation. If you have suggestions or corrections, please feel free to open a pull request or an issue.

## 1.8. Issues and Support

If you encounter any problems or have questions regarding a specific API, please use the 'Issues' section of this repository. Our team will do its best to assist you.

## 1.9. Stay Updated

We regularly update our API documentation to reflect the latest changes and improvements. Keep an eye on this repository for the most current information.

---

Thank you for using PandaTech's APIs! We hope this documentation helps you in your development journey.