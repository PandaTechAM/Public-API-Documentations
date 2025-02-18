- [1. e-bak API Documentation](#1-e-bak-api-documentation)
  - [1.1. Overview](#11-overview)
  - [1.2. Environments](#12-environments)
  - [1.3. Common Response Codes](#13-common-response-codes)
  - [1.4. Error Handling](#14-error-handling)
  - [1.5. Authentication \& Authorization](#15-authentication--authorization)
    - [1.5.1. User Authentication](#151-user-authentication)
    - [1.5.2. Obtaining an Access Token](#152-obtaining-an-access-token)
    - [1.5.3. Using the Token](#153-using-the-token)
    - [1.5.4. Token Expiration and Renewal](#154-token-expiration-and-renewal)
  - [1.6. API Endpoints](#16-api-endpoints)
    - [1.6.1. Ping](#161-ping)
    - [1.6.2. Login](#162-login)
    - [1.6.3. Common models](#163-common-models)
    - [1.6.4. Debt Retrieval by Estate](#164-debt-retrieval-by-estate)
    - [1.6.5. Debt Retrieval by Owner Unique Identifier](#165-debt-retrieval-by-owner-unique-identifier)
    - [1.6.6. Debt Commission Calculation](#166-debt-commission-calculation)
    - [1.6.7. Debt Payment](#167-debt-repayment)
    - [1.6.8. Search Cities](#168-search-cities)
    - [1.6.9. Search Condominium Association](#169-search-condominium-association)
    - [1.6.10. Search Buildings](#1610-search-buildings)
    - [1.6.11. Search Districts](#1611-search-districts)
    - [1.6.12. Search Estates](#1612-search-estates)
  - [1.7. Contributing](#17-contributing)
  - [1.8. Issues and Support](#18-issues-and-support)
  - [1.9. Stay Updated](#19-stay-updated)

# 1. e-bak API Documentation

## 1.1. Overview

The e-bak API allows secure and programmatic access to condominium association debt retrieval and repayment functionalities. This RESTful interface enables you to:

- Retrieve outstanding debts for a specific estate or owner.
- Calculate and apply commissions on debts.
- Process debt repayment transactions.
- Search and query related information such as cities, buildings, districts, and more.

With these endpoints, you can seamlessly integrate condominium fee management functionality into your own applications.

**Language Support:**
Requests should include an `Accept-Language` header with a supported ISO language code. Valid options include:

- `en-US` for English (US)
- `hy-AM` for Armenian (default if none specified or invalid)
- `ru-RU` for Russian

## 1.2. Environments

**Test Environment (for development and QA)**

- **Base URL:**
  [https://qabe-ca.pandatech.it](https://qabe-ca.pandatech.it).
- **OpenAPI Spec:**
  [https://qabe-ca.pandatech.it/openapi/integration.json](https://qabe-ca.pandatech.it/openapi/integration-v1.json)
- **Scalar UI (Test Only):**
  [https://qabe-ca.pandatech.it/scalar/integration](https://qabe-ca.pandatech.it/docs/integration-v1)
- **Swagger UI (Test Only):**
  [https://qabe-ca.pandatech.it/swagger/integration](https://qabe-ca.pandatech.it/swagger/integration-v1)

> **Note:** Scalar UI and Swagger UI are not available in the production environment.

**Production Environment**

- **Base URL:**
  [https://be.e-bak.am](https://be.e-bak.am)

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

Valid user credentials and an access token are required to interact with most endpoints.

### 1.5.1. User Authentication

Obtain and **securely** store user credentials. You must have a valid user account to request a token.

### 1.5.2. Obtaining an Access Token

Make a `POST` request to `/api/v1/integration/login` with your credentials. A successful response includes an `accessToken` and an `expirationDate`.

### 1.5.3. Using the Token

Include the token in the `Authorization` header for subsequent requests. Tokens refresh automatically with each successful API call, so active usage extends token validity.

### 1.5.4. Token Expiration and Renewal

If you receive a `401 Unauthorized` error, your token may have expired. Re-authenticate via `/api/v1/integration/login` to obtain a new token before retrying your request.

## 1.6. API Endpoints

### 1.6.1. Ping

- **Endpoint:** `GET /above-board/ping`
- **Description:** Checks connectivity
- **Response:** `"pong"`

### 1.6.2. Login

- **Endpoint:** `POST /api/v1/integration/login`
- **Description:** Authenticates a user and returns an access token.
- **Request:**
  ```json
  {
    "login": "easypay@easypay.am",
    "password": "Qwerty123!"
  }
  ```
- **Response:**
  ```json
  {
    "accessToken": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "expirationDate": "2023-12-21T08:34:20.996Z"
  }
  ```

### 1.6.3. Common models

**Estate Type Enum**

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

**
Check Commission Enum**

```csharp
public enum CheckCommission
{
   Yes = 1,
   No = 2
}
```

### 1.6.4. Debt Retrieval by Estate

- **Endpoint:** `GET /api/v4/integration/debts/{estateId}`
- **Description:** Retrieves all outstanding debts for a given estate.
- **Note:**
- **Response:**
  ```json
  {
    "partnerId": 8,
    "partnerName": "Անուն Ազգանուն",
    "estateId": 1000270,
    "estateType": 5,
    "estateAddress": "Գյուլբենկյան 33, 1",
    "primaryEstateOwnerFullName": "Անուն Ազգանուն",
    "checkCommission": 1,
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
  > `primaryEstateOwnerFullName` may be `null` if not available

### 1.6.5. Debt Retrieval by Owner Unique Identifier

- **Endpoint:** `GET /api/v4/integration/debts/owner/{uniqueDocumentId}`
- **Description:** Retrieves all outstanding debts associated with an owner identified by SSN or Tax Code.
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
        "checkCommission": 1,
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
        "checkCommission": 1,
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
        "checkCommission": 2,
        "balance": 0,
        "debts": []
      }
    ]
  }
  ```
  > `primaryEstateOwnerFullName` may be `null` if not available

### 1.6.6. Debt Commission Calculation

- **Endpoint:** `POST /api/v2/integration/debts/commission`
- **Description:** Calculates commissions for the specified debts.
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

### 1.6.7. Debt Payment

- **Path:** `POST /api/v2/integration/debts/payments`
- **Description:** Repays debts using a FIFO approach unless a specific `debtId` is provided. Provide an `outerPaymentId` for tracking this payment in your system.

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
    "transactions": [
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

**Bank Enum Values:**

```csharp
public enum Bank
{
   AcbaBank = 0,
   AraratBank = 1,
   Ameriabank = 2,
   AmioBank = 3,
   IDBank = 4,
   Ardshinbank = 5,
   Armswissbank = 6,
   Artsakhbank = 7,
   BiblosBankArmenia = 8,
   HSBCBankArmenia = 9,
   Evocabank = 10,
   Inecobank = 11,
   ConverseBank = 12,
   Armeconombank = 13,
   MellatBank = 14,
   Unibank = 15,
   VTBArmeniaBank = 16,
   FastBank = 17
}
```

### 1.6.8. Search Cities

- **Endpoint:** `GET /api/v2/integration/search/cities`
- **Description:** Retrieves all cities.
- **Request:** https://qabe-ca.pandatech.it/api/v2/integration/search/cities
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

### 1.6.9. Search Condominium Association

- **Endpoint:** `GET /api/v3/integration/search/condominium-associations?cityId={cityId}&Page={page}&PageSize={pageSize}`
- **Description:** Retrieves condominium associations filtered by city, with pagination.
- **Request:** https://qabe-ca.pandatech.it/api/v3/integration/search/condominium-associations?cityId=1&Page=1&PageSize=4
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

### 1.6.10. Search Buildings

- **Endpoint:** `GET /api/v2/integration/search/buildings...`
- **Description:** Retrieves buildings based on one of the following filters: `PartnerId`, `CityId`, or `DistrictId`. At least one of these parameters is required. Also supports pagination.
- **Request Examples:**
  1. https://qabe-ca.pandatech.it/api/v2/integration/search/buildings?PartnerId=1&Page=1&PageSize=1
  2. https://qabe-ca.pandatech.it/api/v2/integration/search/buildings?CityId=1&Page=1&PageSize=1
  3. https://qabe-ca.pandatech.it/api/v2/integration/search/buildings?DistrictId=1&Page=1&PageSize=1
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

### 1.6.11. Search Districts

- **Endpoint:** `GET /api/v1/integration/search/districts?CityId={cityId}`
- **Description:** Retrieves all districts within city.
- **Request:** Retrieves districts in a given city.
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

### 1.6.12. Search Estates

- **Endpoint:** `GET /api/v2/integration/search/estates`
- **Description:** Retrieves estates within a specific building, including estate types and pagination.
- **Request:** https://qabe-ca.pandatech.it/api/v2/integration/search/estates?BuildingId=1&Page=1&PageSize=1
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

We welcome contributions and improvements to this documentation. Submit a pull request or open an issue for suggestions.

## 1.8. Issues and Support

For issues or support inquiries, use the repository's “Issues” section. Our team will respond to help resolve your problem.

## 1.9. Stay Updated

We regularly update this documentation. Check the repository for the latest changes, new features, and improvements.

---

Thank you for using PandaTech's APIs! We hope this documentation helps you in your development journey.
