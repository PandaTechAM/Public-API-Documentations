# CashMe Integration Guide (External APIs)

These endpoints allow **CashMe** to interact with wallet loan sessions.

- Retrieve wallet user data (`/user-payload`)
- Disburse loan to wallet (`/disburse`)

---

## Authentication & Request Requirements

Each request **must** include:

- A valid **Authorization** header with an HMAC signature
- A **Timestamp** header in UTC
- A request body specific to the endpoint(Token)

### Required Headers

| Header        | Value                         | Notes                                  |
|--------------|-------------------------------|----------------------------------------|
| Authorization | `HMAC <CashMe>:<SIGNATURE>`  | HMAC-SHA256 over `Timestamp:Token`     |
| Timestamp     | `YYYY-MM-DDTHH:mm:ssZ`        | UTC time, ISO 8601 format              |

### HMAC Signature Generation

Siganture is generated using HMAC-SHA256 algorithm with a **shared secret key** over the string: `Timestamp:Token`


---

## API Endpoints

### Get Wallet User Payload

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
    "phone": "(374)00000000",
    "email": "test@mailinator.com",
    "citizenCountryId": "051",
    "amount": 10000.00000000000000000000
}
```
#### Error Responses

- **400 Bad Request**: Invalid request, expired token or missing fields.
- **401 Unauthorized**: Invalid HMAC or expired timestamp.


### Disburse Loan to Wallet

**Endpoint**  
`POST /api/external/v1/cashme/disburse`

**Summary**  
Top up wallet balance using a previously created loan session token

#### Request Body Example

```json
{ 
 "token": "yFqEHF7lNoXVg44-oVg1B18tIqLVrRBMw8TSQB4rJm0" 
}
```

#### Response Example

In case of success, returns the transaction ID:
```json
"019b6a24-0486-787b-b908-6fe3a9e08dba"
```
#### Error Responses

- **400 Bad Request**: Invalid request, expired token or missing fields.
- **401 Unauthorized**: Invalid HMAC or expired timestamp.


