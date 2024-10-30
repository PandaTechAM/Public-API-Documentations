- [1. EasyPay Merchant Integration Guide](#1-easypay-merchant-integration-guide)
  - [1.1. Overview](#11-overview)
  - [1.2. Prerequisites](#12-prerequisites)
  - [1.3. Authentication and Security](#13-authentication-and-security)
    - [1.3.1. HMAC Authentication](#131-hmac-authentication)
  - [1.4. Standard HTTP Response Codes](#14-standard-http-response-codes)
  - [1.5. Input Formatting and HMAC Calculation](#15-input-formatting-and-hmac-calculation)
  - [1.6. API Endpoints](#16-api-endpoints)
    - [1.6.1. Balance Inquiry](#161-balance-inquiry)
    - [1.6.2. Make Payment](#162-make-payment)
  - [1.7. Security Recommendation](#17-security-recommendation)
  - [1.8. Summary](#18-summary)

# 1. EasyPay Merchant Integration Guide

## 1.1. Overview

This document provides detailed instructions on how merchants can integrate with EasyPay’s system through secure API endpoints. The integration supports balance inquiries and payment processing with standardized security protocols such as `HMAC-based authentication`.

## 1.2. Prerequisites

- **API Base URL**: Provided by the merchant upon registration.
- **HMAC Key**: A unique key shared securely by EasyPay.
  > **Security Note:** Merchants are advised to whitelist EasyPay’s IP addresses to ensure only trusted requests are processed.

## 1.3. Authentication and Security

### 1.3.1. HMAC Authentication

1. **Algorithm:** HMAC-SHA256 is required to authenticate requests.
2. **Key Storage:** Store the HMAC key securely (e.g., AES-256 encryption) in a vault or encrypted database, not in code.
3. **Headers:**
   - **Authorization:** Format - `Authorization: HMAC <HMAC hash of concatenated parameters>`
   - **Nonce:** Format - `Nonce: <UUIDv4>`
     > Note: Ensure the string representation of UUIDv4 is used to avoid encoding errors.
4. **Language Header:** Optional but recommended for multi-language support:
   - `Accept-Language`: Set to `hy-AM` (Armenian), `en-US` (English), or `ru-RU` (Russian) based on the response language preference.

## 1.4. Standard HTTP Response Codes

All API responses must follow standard HTTP status codes:

- 200: Success
- 400: Bad Request
- 401: Unauthorized
- 404: Not Found
- 500: Internal Server Error

## 1.5. Input Formatting and HMAC Calculation

Each input provided must follow the below structure:

Example Input:

```json
[
  { "Type": 16, "Value": "12345", "TechnicalIndex": 1 },
  { "Type": 3, "Value": "98765", "TechnicalIndex": 2 }
]
```

**Concatenation for HMAC:**
Inputs should be concatenated in the format `Type:Value:TechnicalIndex:`, separated by colons.

**Result:**
`16:12345:1:3:98765:2:`

> Usage: The concatenated string forms part of the HMAC calculation, ensuring message integrity.

**List of types:**

```txt
1: Id
2: Name
3: PhoneNumber
4: SocialSecurityNumber
5: Passport
6: EstateNumber
7: PlateNumber
8: RegistrationCertificateNumber
9: Date
10: Code
11: Description
12: Email
13: TaxCode
14: CustomerId
15: Username
16: BankAccount
17: BankCard
18: PayerName
19: Course
20: Address
21: Count
22: LoanId
23: InvestmentId
24: Surname
```

## 1.6. API Endpoints

### 1.6.1. Balance Inquiry

**Endpoint:**
`POST /api/balance-inquiry`

**Headers:**

```http
Nonce: 69f77f9c-b9b5-43ac-9e6b-516863b8a451
Authorization: HMAC 5EXOvak7ZZWG6ZCiliXK6TNKRIEgTu1Clw8E8ilsoTQ=
Accept-Language: en-US
```

**HMAC Calculation:**
`HMAC(BalanceInquiryId + MerchantServiceIdentifierId + Inputs + Nonce)`

**Request Body:**

```json
{
  "BalanceInquiryId": 12345,
  "MerchantServiceIdentifierId": 67890,
  "Inputs": [
    { "TechnicalIndex": 1, "Type": 1, "Value": "12345" },
    { "TechnicalIndex": 2, "Type": 3, "Value": "98765" }
  ]
}
```

**Response Example:**

```json
{
  "Debt": 500.75,
  "Properties": [{ "Key": "Debt for 09.24", "Value": "500.75" }]
}
```

**Response Headers**

```http
Authorization: HMAC euAmWwAGbPkV1YXQffhpNFBh8/1Y5my3ZsZ+uNmooKo=
```

**HMAC Calculation For Response:**
`HMAC(Debt + Properties.Values + Request's Nonce)`

### 1.6.2. Make Payment

**Endpoint:**
`POST /api/payments`

**Headers:**

```http
Nonce: 04ba5422-e021-4dfb-a716-bdd1440e91b4
Authorization: HMAC 5EXOvak7ZZWG6ZCiliXK6TNKRIEgTu1Clw8E8ilsoTQ=
Accept-Language: ru-RU
```

**HMAC Calculation:**
`HMAC(OrderId + Amount + BalanceInquiryId + MerchantServiceIdentifierId + Inputs + Nonce)`

**Request Body:**

```json
{
  "OrderId": "434dd03f-ede8-4e55-b71f-f81cb4120cba",
  "Amount": 500.75,
  "BalanceInquiryId": 12345,
  "MerchantServiceIdentifierId": 67890,
  "Inputs": [
    { "TechnicalIndex": 1, "Type": 1, "Value": "12345" },
    { "TechnicalIndex": 2, "Type": 3, "Value": "98765" }
  ]
}
```
> Important Notice: To ensure smooth processing and avoid duplicate payments, please store the `OrderId` in your database and enforce a unique index constraint on it. This will prevent duplication issues and facilitate efficient payment reconciliation during the accounting process.


**Response Example:**

```json
{
  "PaymentId": "xyz-12345"
}
```

**Response Headers**

```http
Authorization: HMAC 4Q3soI9dTLuCLl3kWnVqPcCnayHwreKYUBwtLor3LRI=
```

**HMAC Calculation For Response:**
`HMAC(PaymentId + Request's Nonce)`

## 1.7. Security Recommendation

Merchants should persist HMAC signatures temporarily to detect duplicate or replay attacks. If a subsequent request arrives with the same signature, it should be treated as a duplicate request or a potential replay attack, and appropriate action should be taken (e.g., rejecting the request and logging the incident).

## 1.8. Summary

This document outlines how to integrate and securely communicate with EasyPay's platform using HMAC-based authentication. Following the instructions ensures robust security and smooth interoperability between systems.

For further assistance, contact your integration manager at EasyPay.
