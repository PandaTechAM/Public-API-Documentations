- [1. FinHub Payment Gateway Integration Guide](#1-finhub-payment-gateway-integration-guide)
  - [1.1. Introduction](#11-introduction)
  - [1.2. Key Terminology](#12-key-terminology)
  - [1.3. Overview](#13-overview)
  - [1.4. Prerequisites](#14-prerequisites)
  - [1.5. Security \& Authentication](#15-security--authentication)
    - [1.5.1. Overview](#151-overview)
    - [1.5.2. HMAC Calculation](#152-hmac-calculation)
  - [1.6. Response Codes \& Error Handling](#16-response-codes--error-handling)
    - [1.6.1. Standard HTTP Response Codes](#161-standard-http-response-codes)
  - [1.7. Error Response Structure](#17-error-response-structure)
  - [1.8. Prerequisites](#18-prerequisites)
  - [1.9. API Endpoints](#19-api-endpoints)
    - [1.9.1. Connectivity (Ping)](#191-connectivity-ping)
    - [1.9.2. Terminal Information](#192-terminal-information)
    - [1.9.3. Agent Balance](#193-agent-balance)
    - [1.9.4. Merchant Services](#194-merchant-services)
    - [1.9.5. Merchant Service Details](#195-merchant-service-details)
    - [1.9.6. Balance Inquiry](#196-balance-inquiry)
      - [1.9.6.1. Step 1: Initial Inquiry](#1961-step-1-initial-inquiry)
      - [1.9.6.2. Step 2: Detailed Inquiry (If Needed)](#1962-step-2-detailed-inquiry-if-needed)
    - [1.9.7. Customer Commission Check](#197-customer-commission-check)
    - [1.9.8. Payment Order Creation](#198-payment-order-creation)
    - [1.9.9. Order Status Check (Drafted only)](#199-order-status-check-drafted-only)
    - [1.9.10. Additional Considerations](#1910-additional-considerations)
    - [1.9.11. Support \& Troubleshooting](#1911-support--troubleshooting)

# 1. FinHub Payment Gateway Integration Guide

## 1.1. Introduction

Welcome to FinHub’s Payment Gateway Integration Guide. This document walks you through:

- Authenticating requests via HMAC-SHA256.
- Querying terminals, balances, and service details.
- Checking commissions and performing balance inquiries.
- Creating and monitoring payment orders.

By following this guide, you’ll seamlessly integrate FinHub’s APIs into your existing or new payment solutions, ensuring secure and efficient payment transactions.

## 1.2. Key Terminology

- **API Base URL:** The root address for all FinHub endpoints. This URL is provided to you upon registration.
- **API Key:** Identifies your application to FinHub.
- **HMAC Key:** A secret key used to calculate the HMAC-SHA256 signature for requests.
- **Nonce:** A UUIDv4 string used once per request to maintain uniqueness and prevent replay attacks.
- **Merchant Service:** Represents a specific service (e.g., mobile top-up) offered via FinHub.
- **Identifier / Identifier Detail Type:** Fields required by specific merchant services to identify the payment account (e.g., phone number, social security number).

## 1.3. Overview

This guide provides a comprehensive overview of how to integrate with FinHub’s payment gateway. It details the required authentication, input formatting for HMAC calculations, and descriptions of each API endpoint. Whether you are setting up your terminal information or processing payment orders, this document is designed to be clear and accessible.

## 1.4. Prerequisites

Before integrating with FinHub, ensure you have:

1. **Valid FinHub Credentials**

- **API Key:** Used as part of the Authorization header.
- **HMAC Key:** Must be stored securely (in a vault or key management system).

2. **Base URL**
   - The root address for FinHub’s API calls (e.g., https://your-env.easypay.am/).
3. **Language Header**
   Include an `Accept-Language` HTTP header with a valid ISO language code:
   - `en-US` (English, default),
   - `hy-AM` (Armenian),
   - `ru-RU` (Russian).

## 1.5. Security & Authentication

### 1.5.1. Overview

FinHub enforces HMAC-SHA256-based authentication. Every request must include:

- **Nonce** (unique UUIDv4)
- **Authorization** header in the form:

```txt
HMAC <API_KEY>:<BASE64_SIGNATURE>
```

> Storage Recommendation: Keep both the API and HMAC keys secure (avoid hardcoding them).

### 1.5.2. HMAC Calculation

Some endpoints require you to compute an HMAC hash over specific parameters. The general approach is:

1. **Concatenate Inputs:**
   For simple fields, concatenate the values. For arrays or objects (such as “inputs”), concatenate their properties using the colon (`:`) as a separator.
   **Example Input:**

   ```json
   [
     { "IdentifierDetailType": 16, "Value": "12345", "TechnicalIndex": 1 },
     { "IdentifierDetailType": 3, "Value": "98765", "TechnicalIndex": 2 }
   ]
   ```

   **Concatenation Format:**
   `IdentifierDetailType:Value:TechnicalIndex:`
   **Result:**
   `16:12345:1:3:98765:2:`

2. **Calculate the HMAC:**

   - Combine the concatenated string with any additional required fields (e.g., Nonce, Merchant Service IDs).
   - Use the HMAC-SHA256 algorithm with your secret key.
   - Base64-encode the resulting byte array.
   - Prefix the output with HMAC and your API key.
   - **HMAC Calculation Example:**
     ```txt
     Api Key: ffa5be93-35e6-4186-8b84-44882d6dbb30
     Combined String: 1234567890 + "16:12345:1:3:98765:2:" + deadd8fc-58fc-4f12-8a57-808711f6319d
     Key: your HMAC secret key
     Base64-encoded HMAC: iVGljbFol+xfKGW/uw3B9wxP5timuV5YYyzYkL3cicU=
     Final Header: "HMAC ffa5be93-35e6-4186-8b84-44882d6dbb30:EQiqUTkEQOGNkfcW4MU6XooOm+rL1REz5njbkvq40bA="
     ```

3. **Identifier Detail Types:**
   The following list describes common identifier types. You might use these when constructing the HMAC string:
   | Code | Description |
   | :--- | :------------------------------------------------ |
   |1|Id|
   |2|Name|
   |3|Phone Number|
   |4|Social Security Number|
   |5|Passport|
   |6|Estate Number|
   |7|Plate Number|
   |8|Registration Certificate No.|
   |9|Date|
   |10|Code|
   |11|Description|
   |12|Email|
   |13|Tax Code|
   |14|Customer Id|
   |15|Username|
   |16|Bank Account|
   |17|Bank Card|
   |18|Payer Name|
   |19|Course|
   |20|Address|
   |21|Count|
   |22|Loan Id|
   |23|Investment Id|
   |24|Surname|
   |25|Full Name|
   |26|Technical Id|
   |27|Unique Document Id|

> **Note:**
> You can verify results with an online HMAC-SHA256 generator (e.g., https://easypay.am/hmac-generator).

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

## 1.8. Prerequisites

Before you begin, ensure you have:

- **API Base URL**: Provided upon registration.
- **API Key**: A unique key that identifies your integration.
- **HMAC Key**: A secret key used as salt for HMAC calculations. Store these keys securely (e.g., in a vault).

## 1.9. API Endpoints

All endpoints assume the base URL provided by FinHub. Every request must include the headers: `Nonce`, `Authorization`, and `Accept-Language`.

### 1.9.1. Connectivity (Ping)

Validate connectivity.

- **Endpoint:** `GET /above-board/ping`
- **Response:**
  ```txt
  "pong"
  ```

### 1.9.2. Terminal Information

Retrieve your terminal details such as currency and limits.

- **URL:** `GET /api/external/v1/terminals`
- **HMAC Calculation:**
  `HMAC(Nonce)`
- **Response Example:**
  ```json
  {
  "id": 123,
  "name": "Terminal Name",
  "currencyCode": "USD",
  "dailyMaxPaymentSum": 11000,
  "dailyCurrentPaymentSum": 10,
  "dailyMaxPaymentCount": 50,
  "dailyCurrentPaymentCount": 2,
  "monthlyMaxPaymentSum": null,
  "monthlyCurrentPaymentSum": null,
  "monthlyMaxPaymentCount": null,
  "monthlyCurrentPaymentCount": null
}
  ```

### 1.9.3. Agent Balance

Check your current deposit balance.

- **URL:** `GET /api/external/v1/agent-balance`
- **HMAC Calculation:**
  `HMAC(Nonce)`
- **Response Example:**
  ```json
  {
    "balance": 978648.14
  }
  ```

### 1.9.4. Merchant Services

Retrieve all available merchant services for your terminal.

- **URL:** `GET /api/external/v1/merchant-services`
- **Headers:**
- **HMAC Calculation:**
  `HMAC(Nonce)`
- **Response Example:**
  ```json
  [
    {
      "id": 123,
      "name": "Mobile Top-up",
      "description": "Recharge mobile balance"
    }
  ]
  ```

### 1.9.5. Merchant Service Details

Fetch detailed configurations, input identifiers, and transaction constraints for a specific merchant service.

- **URL:** `GET /api/external/v1/merchant-services/{merchantServiceId}`
- **Path Parameters:**
  `merchantServiceId` (`integer`) – Unique identifier for the merchant service.
- **HMAC Calculation:**
  `HMAC(merchantServiceId + Nonce)`
- **Response Example:**
  ```json
  {
    "config": {
      "minSoloPaymentAmount": 10,
      "maxSoloPaymentAmount": 500,
      "allowedLessThanBalanceInquiry": true,
      "allowedMoreThanBalanceInquiry": true
    },
    "identifiers": [
      {
        "id": 1,
        "enabled": true,
        "details": [
          {
            "technicalIndex": 1,
            "identifierDetailType": 3,
            "required": true,
            "name": "Phone number",
            "prefix": "0",
            "placeHolder": "Enter phone number",
            "hint": "Type only +374 prefixed phone number",
            "example": "+37491234567",
            "defaultValue": "077123456",
            "regex": "^\\+374[0-9]{8}$",
            "options": ["+37491234567", "+37498123456"],
            "inputBlackList": ["+37490000000"]
          }
        ]
      }
    ]
  }
  ```

> **Note:** When options are provided, restrict input to those values. Similarly, prevent entries that match the blacklist.

### 1.9.6. Balance Inquiry

FinHub uses a two-step balance inquiry process when the provided input is not uniquely identifying the account.

- **URL**: `POST /api/external/v1/merchant-services/{merchantServiceId}/balance-inquiry`
- **HMAC Calculation:**
  `HMAC(merchantServiceId + MerchantServiceIdentifierId + Inputs + Nonce)`

#### 1.9.6.1. Step 1: Initial Inquiry

- **Request Example:**
  ```json
  {
    "merchantServiceIdentifierId": 1,
    "inputs": [
      {
        "identifierDetailType": 3,
        "technicalIndex": 1,
        "value": "+37491234567"
      }
    ]
  }
  ```
- **Response Example**:
  ```json
  {
    "balanceInquiryId": 12345,
    "data": [
      {
        "amount": 150,
        "inputs": [
          {
            "identifierDetailType": 3,
            "technicalIndex": 1,
            "value": "+37491234567"
          },
          {
            "identifierDetailType": 10,
            "technicalIndex": 2,
            "value": "12345678"
          }
        ],
        "properties": [{ "key": "Due Date", "value": "2025-02-10" }]
      },
      {
        "amount": 240,
        "inputs": [
          {
            "identifierDetailType": 3,
            "technicalIndex": 1,
            "value": "+37491234567"
          },
          {
            "identifierDetailType": 10,
            "technicalIndex": 2,
            "value": "478956315"
          }
        ],
        "properties": [{ "key": "Due Date", "value": "2025-02-10" }]
      }
    ]
  }
  ```

#### 1.9.6.2. Step 2: Detailed Inquiry (If Needed)

If the response from Step 1 includes additional inputs (i.e. the `inputs` property is not null), make a second call using the complete set of inputs returned.

- **Request Example:**

  ```json
  {
    "merchantServiceIdentifierId": 1,
    "inputs": [
      {
        "identifierDetailType": 3,
        "technicalIndex": 1,
        "value": "+37491234567"
      },
      {
        "identifierDetailType": 10,
        "technicalIndex": 2,
        "value": "12345678"
      }
    ]
  }
  ```

- **Response Example:**

```json
{
  "balanceInquiryId": 12346,
  "minAmount": 100,
  "maxAmount": 500,
  "data": [
    {
      "amount": 150,
      "inputs": [
        {
          "identifierDetailType": 3,
          "technicalIndex": 1,
          "value": "+37491234567"
        },
        {
          "identifierDetailType": 10,
          "technicalIndex": 2,
          "value": "12345678"
        }
      ],
      "properties": [
        { "key": "Due Date", "value": "2025-02-10" },
        { "key": "Account Type", "value": "Savings" },
        { "key": "Overdue Amount", "value": "50" },
        { "key": "Last Payment Date", "value": "2025-01-20" },
        { "key": "Billing Cycle", "value": "Monthly" }
      ]
    }
  ]
}
```

> **Tip:** It is uncommon to require the second inquiry, but it is necessary when the initial input does not uniquely identify the account.

### 1.9.7. Customer Commission Check

Before processing a payment, determine the commission fee that the customer will incur.

- **URL:** `GET /api/external/v1/commissions`
- **HMAC Calculation:**
  `HMAC(merchantServiceId + MerchantServiceIdentifierId + Inputs + Nonce)`
- **Request Example:**
  ```json
  {
    "merchantServiceId": 123,
    "merchantServiceIdentifierId": 1,
    "balanceInquiryId": 12346,
    "amount": 2450,
    "inputs": [
      {
        "identifierDetailType": 3,
        "technicalIndex": 1,
        "value": "+37491234567"
      },
      {
        "identifierDetailType": 10,
        "technicalIndex": 2,
        "value": "12345678"
      }
    ]
  }
  ```
- **Response Example:**
  ```json
  {
    "commission": 15.41
  }
  ```

### 1.9.8. Payment Order Creation

Create a new payment order based on the latest balance inquiry results.

> **Important:** The inputs provided must match exactly those from the last inquiry. The amount specified here excludes the commission fee (which is automatically added).

- **URL:** `POST /api/external/v1/orders`
- **Description:** Creates a new payment order based on the latest balance inquiry results.
- **HMAC Calculation:**
  `HMAC(merchantServiceId + MerchantServiceIdentifierId + Inputs + Nonce)`
- **Request Example:**
  ```json
  {
    "id": "unique_id_generated_by_your_system",
    "merchantServiceId": 123,
    "merchantServiceIdentifierId": 1,
    "balanceInquiryId": 12346,
    "amount": 2540,
    "inputs": [
      {
        "identifierDetailType": 3,
        "technicalIndex": 0,
        "value": "+37491234567"
      },
      {
        "identifierDetailType": 26,
        "technicalIndex": 1,
        "value": "12"
      }
    ]
  }
  ```
- **Response Example:**
  ```json
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  }
  ```

> **Note:** Depending on your terminal configuration, a successful order may return HTTP status `200 OK` or `202 Accepted`. If the order is enqueued, call the status endpoint to determine the outcome.

### 1.9.9. Order Status Check (Drafted only)

Monitor the status of a payment order to confirm whether it was processed successfully and other details such as whether cancel operation permited.

- **URL:** `GET /api/external/v1/orders/{orderId}`
- **HMAC Calculation:**
  `HMAC(orderId + Nonce)`

- **Response Example:**
  ```json
  {
  "orderId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": 6,
  "cancelable" : false
}
  ```

### 1.9.10. Additional Considerations

- **Unique Order ID Enforcement:**
  FinHub treats the order ID as unique. Reusing an ID will result in a 400 Bad Request with the error message `ORDER_ALREADY_EXISTS`.

- **Duplicate Requests:**
  To avoid duplicate payments, ensure that your system indexes the order IDs and prevents repeated submissions.

- **HMAC Integrity:**
  Consistent HMAC validation across endpoints secures your integration. Always verify that all input values and the nonce are correctly concatenated before generating the signature.

### 1.9.11. Support & Troubleshooting

For additional assistance or to report issues, please contact your designated FinHub integration manager. When troubleshooting, include both the `RequestId` and `TraceId` from the error responses to help expedite the resolution process.

---

This guide is designed to provide a clear, step-by-step overview of FinHub’s payment gateway integration. By following the instructions for authentication, input formatting, and endpoint usage, you can ensure a secure and efficient integration experience.
