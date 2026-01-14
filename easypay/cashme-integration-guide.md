- [1. CashMe Integration Guide (External APIs)](#1-cashme-integration-guide-(external-APIs))
    - [1.1. Prerequisites](#11-prerequisites)
        - [1.1.1. Required Headers](#111-required-headers)
        - [1.1.2 HMAC Signature Generation](#112-hmac-signature-generation)
    - [1.2. API Endpoints](#12-api-endpoints)
        - [1.2.1. Get Wallet User Payload](#121-get-wallet-user-payload)
        - [1.2.2. Disburse Loan to Wallet](#122-disburse-loan-to-wallet)




# 1. CashMe Integration Guide (External APIs)

These endpoints allow **CashMe** to interact with wallet loan sessions.

- Retrieve wallet user data (`/user-payload`)
- Disburse loan to wallet (`/disburse`)

---

## 1.1. Prerequisites

Before integrating with FinHub, ensure you have:

1. **Valid FinHub Credentials**

- **HMAC Key:** Must be stored securely (in a vault or key management system).

2. **Base URL**
   - The root address for FinHub's API calls (e.g., https://your-env.easypay.am/).

3. Each request **must** include:

- A valid **Authorization** header with an HMAC signature
- A **Timestamp** header in UTC
- A request body specific to the endpoint(Token)

### 1.1.1. Required Headers

| Header        | Value                         | Notes                                  |
|--------------|-------------------------------|----------------------------------------|
| Authorization | `HMAC <CashMe>:<SIGNATURE>`  | HMAC-SHA256 over `Timestamp:Token`     |
| Timestamp     | `YYYY-MM-DDTHH:mm:ssZ`        | UTC time, ISO 8601 format              |

### 1.1.2 HMAC Signature Generation

Siganture is generated using HMAC-SHA256 algorithm with a **shared secret key** over the string: `Timestamp:Token`


---

## 1.2. API Endpoints

### 1.2.1. Get Wallet User Payload

**Endpoint**  
`POST /api/external/v1/cashme/user-payload`

**Summary**  
Returns wallet user data for a given token.

#### Request Body Example

```json
{ 
 "token": "yFqEHF7lNoXVg44-oVg1B18tIqLVrRBMw8TSQB4rJm0" 
}
```

#### Response Example

```json
{
    "accountId": 1111,
    "firstName": "Test",
    "lastName": "Test",
    "middleName": "Test",
    "passportNum": "AA1111111",
    "socialCard": "1111111111",
    "passportFrom": "111",
    "passportCreated": "1900-01-01",
    "passportExp": "1900-01-01",
    "accountCreated": "1900-01-01T00:00:00.000000Z",
    "sex": "F",
    "birthday": "1900-01-01",
    "address": "Test",
    "phone": "+37400000000",
    "email": "test@mailinator.com",
    "citizenCountryId": "051",
    "amount": 10000.00000000000000000000
}
```
#### Error Responses and Messages
`400 Bad Request`
- TOKEN_IS_NOT_FOUND: When the provided token does not exist.
- ACCESS_TOKEN_EXPIRED: When the provided token has expired.
- INVALID_SESSION_STATUS: When the session status is not valid for this operation.
- PHONE_NUMBER_NOT_FOUND: When the phone number associated with the user is missing.
- EMAIL_IS_REQUIRED: When the email associated with the user is missing.
- DOCUMENT_NOT_FOUND: When the user don't have any verified identification document.

`401 Unauthorized`: Invalid HMAC or expired timestamp


#### Response Example

```json
{
  "requestId": "007",
  "traceId": "5b4d6fafe053f9b6d8aaa8e20",
  "instance": "GET - Test",
  "statusCode": 400,
  "type": "BadRequestException",
  "errors": null,
  "message": "TOKEN_IS_NOT_FOUND"
}
```


### 1.2.2. Disburse Loan to Wallet

**Endpoint**  
`POST /api/external/v1/cashme/disburse`

**Summary**  
Top up wallet balance using a previously created loan session token

#### Request Body Example

```json
{ 
 "token": "yFqEHF7lNoXVg44-oVg1B18tIqLVrRBMw8TSQB4rJm0",
 "amount": 10000,
 "contractNumber": "Test"
}
```

#### Response Example

In case of success, returns the order ID:
```json
"019b6a24-0486-787b-b908-6fe3a9e08dba"
```
#### Error Responses

`400 Bad Request`
- TOKEN_IS_NOT_FOUND: When the provided token does not exist.
- ACCESS_TOKEN_EXPIRED: When the provided token has expired.
- INVALID_SESSION_STATUS: When the session status is not valid for this operation.
- NOT_ENOUGH_FUNDS: When there are insufficient funds in the CashMe balance to disburse the loan.

`401 Unauthorized`: Invalid HMAC or expired timestamp

#### Response Example

```json
{
  "requestId": "007",
  "traceId": "5b4d6fafe053f9b6d8aaa8e20",
  "instance": "GET - Test",
  "statusCode": 400,
  "type": "BadRequestException",
  "errors": null,
  "message": "TOKEN_IS_NOT_FOUND"
}
```


