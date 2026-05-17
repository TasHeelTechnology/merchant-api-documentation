# Tasheel Travel Now Pay Later - Merchant API Integration Guide

## 1. Overview

Tasheel Assisted Checkout enables authorized merchants to facilitate a Tasheel user’s BNPL downpayment **within the merchant checkout experience**, without redirecting the user to the Tasheel app.

This flow is explicitly authorized by the user through **OTP-based verification**. After OTP verification, Tasheel issues a **short-lived temporary access token**. Using this token, the merchant platform can:

* Securely list the user’s saved cards
* Allow card selection or default assignment
* Initialize and complete the BNPL downpayment using the selected saved card

---

## 2. Authentication & Authorization

Assisted checkout uses **OTP-based temporary authorization** initiated by the merchant and approved by the user.

### 2.1 Request OTP (User Consent Initiation)

```
POST /merchant/temp-auth/request
```
**Headers:**
```
Content-Type: application/json
Accept: application/json
```
**Body:**

```json
{
  "merchant_uuid": "84b5edc9-766d-4fcc-9eef-dadcc5a7c6c7",
  "mobile": "9689378439758"
}
```

**Response (example):**

```json
{
  "success": true,
  "message": "OTP sent successfully",
  "otp_id": 98
}
```

#### OTP Cooldown Behaviour

To prevent OTP abuse and SMS flooding, the system enforces a cooldown period.

- Only **one OTP can be generated within a 2-minute window**.
- If another request is made during this cooldown period, the API **will not generate a new OTP**.
- Instead, the API returns the **existing OTP session**.
- The **remaining wait time is provided inside the `message` field** of the response.

The merchant should **reuse the returned `otp_id`** when calling the OTP verification endpoint.

#### Example Response During Cooldown

```json
{
  "success": true,
  "message": "OTP already sent. Please wait 101 seconds before requesting again.",
  "otp_id": 688,
  "expires_at": "2026-03-08T06:16:14.000000Z"
}
```

---

**Notes:**

* A new OTP is not generated during cooldown.
* The previously issued OTP remains valid until expires_at.
* Merchants should proceed with /merchant/temp-auth/verify using the same otp_id.

---
## Expected Error Responses

**400 – Validation Error**
Occurs when required fields are missing or invalid.

```json
{
    "status": "error",
    "code": "VALIDATION_FAILED",
    "message": "Validation failed",
    "errors": {
        "mobile": [
            "The mobile field is required."
        ]
    }
}
```

---

**404 – User Not Found**

```json
{
    "status": "error",
    "code": "NOT_FOUND",
    "message": "No user found for this mobile",
    "errors": {
        "not_found": [
            "User not found for this mobile"
        ]
    }
}
```

---

**403 – User Not Eligible (Overdue Installments)**

```json
{
    "status": "error",
    "code": "HAS_OVERDUE",
    "message": "User has overdue installments. Assisted checkout blocked.",
    "errors": {
        "HAS_OVERDUE": [
            "User has overdue installments"
        ]
    }
}
```

---

**500 – OTP Generation Failure**

```json
{
    "status": "error",
    "code": "OTP_CREATION_FAILED",
    "message": "OTP creation failed",
    "errors": {
        "OTP_CREATION_FAILED": [
            "OTP creation failed"
        ]
    }
}
```
---

### 2.2 Verify OTP & Issue Temporary Access Token

```
POST /merchant/temp-auth/verify
```
**Headers:**
```
Content-Type: application/json
Accept: application/json
```
**Body:**

```json
{
  "mobile": "9689378439758",
  "otp": "111111",
  "merchant_uuid": "84b5edc9-766d-4fcc-9eef-dadcc5a7c6c7",
  "otp_id": 372
}
```

**Processing Flow:**

* Confirms OTP session belongs to the user and matches the merchant
* Ensures OTP is not already used and not expired
* Marks OTP as consumed
* Issues `temp_token` valid for **10 minutes**

### OTP Verification Attempt Limit

For security purposes, OTP verification attempts are restricted.

* Maximum **2 incorrect OTP attempts** are allowed per OTP session.
* If the limit is reached, the OTP session becomes invalid.
* The merchant must request a **new OTP** using `/merchant/temp-auth/request`.

**Response (example):**

```json
{
  "success": true,
  "temp_token": "80428f...1b5ea5",
  "expires_at": "2026-01-07T06:39:49.000000Z"
}
```
---

## Expected Error Responses

**400 – Validation Error**

```json
{
    "status": "error",
    "code": "VALIDATION_FAILED",
    "message": "Validation failed",
    "errors": {
        "otp": [
            "The otp must be 6 digits."
        ]
    }
}
```

---

**422 – Validation Error**

```json
{
    "status": "error",
    "code": "VALIDATION_FAILED",
    "message": "Validation failed",
    "errors": {
        "merchant_uuid": [
            "The selected merchant uuid is invalid."
        ]
    }
}
```

---

**422 – Validation Error**

```json
{
    "status": "error",
    "code": "VALIDATION_FAILED",
    "message": "Validation failed",
    "errors": {
        "merchant_uuid": [
            "The merchant uuid field is required."
        ]
    }
}
```

---

**422 – Validation Error**

```json
{
    "status": "error",
    "code": "VALIDATION_FAILED",
    "message": "Validation failed",
    "errors": {
        "mobile": [
            "The mobile field is required."
        ]
    }
}
```

---

**404 – User Not Found**

```json
{
    "status": "error",
    "code": "NOT_FOUND",
    "message": "No user found for this mobile",
    "errors": {
        "NOT_FOUND": [
            "No user found for this mobile"
        ]
    }
}
```

---

**404 – Invalid OTP Session**

```json
{
    "status": "error",
    "code": "INVALID_SESSION",
    "message": "Invalid OTP session",
    "errors": {
        "INVALID_SESSION": [
            "Invalid OTP session"
        ]
    }
}
```

---

**400 – Wrong OTP**


```json
{
    "status": "error",
    "code": "INVALID_OTP",
    "message": "Wrong OTP entered.",
    "errors": {
        "INVALID_OTP": [
            "Wrong OTP entered."
        ],
        "remaining_attempts": [
            1
        ]
    }
}

```

---

**429 – Too Many Wrong OTP Attempts**

```json
{
    "status": "error",
    "code": "TOO_MANY_WRONG_ATTEMPTS",
    "message": "Too many wrong attempts. Please request a new OTP.",
    "errors": {
        "TOO_MANY_ATTEMPTS": [
            "Too many wrong attempts."
        ]
    }
}
```

---

**403 – Merchant Mismatch**

```json
{
    "status": "error",
    "code": "MERCHANT_MISMATCH",
    "message": "Merchant mismatch",
    "errors": {
        "MERCHANT_MISMATCH": [
            "Merchant mismatch"
        ]
    }
}
```

---

**400 – OTP Already Used**

```json
{
    "status": "error",
    "code": "SESSION_ALREADY_USED",
    "message": "OTP session already used",
    "errors": {
        "SESSION_ALREADY_USED": [
            "OTP session already used"
        ]
    }
}
```

---

**400 – OTP Expired**

```json
{
    "status": "error",
    "code": "OTP_EXPIRED",
    "message": "OTP expired",
    "errors": {
        "OTP_EXPIRED": [
            "OTP expired"
        ]
    }
}
```

---

## 3. Base URL & Environment

Example environments:

* **Staging:** `https://staging.tasheelfs.om/api`
* **Production:** `https://tasheelfs.om/api` (or your production domain)

All assisted checkout requests must include the temporary token header:

```
temp-user-access: <temp_token>
```

---

## 4. Threshold

The assisted checkout flow applies the following restrictions.

### OTP Request

* Only **one OTP can be generated within a 2-minute cooldown period**.
* If another request is made during the cooldown window, the API returns the **existing OTP session** instead of generating a new OTP.

### OTP Verification

* Each OTP session allows a maximum of **2 incorrect attempts**.
* If the attempt limit is reached, the OTP session becomes invalid and a new OTP must be requested.


### OTP Request

* Only **one OTP can be generated within a 2-minute cooldown period**.
* If another request is made during the cooldown window, the API returns the **existing OTP session** instead of generating a new OTP.

## 5. Temporary Access Token

After successful OTP verification, the system issues a **temporary access token** used to authorize the assisted checkout request.

### Token Behavior

- A **temporary access token** is generated once the OTP is successfully verified.
- The token must be included in subsequent assisted checkout requests using the request header:

```
temp-user-access: <temporary_token>
```

### Token Expiry

- The temporary token remains valid for **10 minutes**.
- After the expiration time, the token becomes invalid and the merchant must **restart the OTP authorization flow**.


---

## 5. Endpoints

### 5.1 List User’s Saved Cards

```
GET /assisted/cards
```

**Headers:**

```
Content-Type: application/json
temp-user-access: <temp_token>
```

**Response (example):**

```json
{
  "success": true,
  "data": [
    {
      "id": 3,
      "token": "tok_125",
      "last4": "4248",
      "brand": "Master",
      "exp_month": 12,
      "exp_year": 2029,
      "is_default": true
    }
  ]
}
```

Notes:

* Cards are returned from your DB (`saved_cards` table).
* Only masked card metadata is returned.
---
## Expected Error Responses

**401 – Temp Token Missing**

```json
{
    "status": "error",
    "code": "TOKEN_MISSING",
    "message": "Temp token missing",
    "errors": {
        "TOKEN_MISSING": [
            "Temp token missing"
        ]
    }
}
```

---

**401 – Invalid or Expired Token**

```json
{
    "status": "error",
    "code": "INVALID_TOKEN",
    "message": "Invalid or expired token",
    "errors": {
        "INVALID_TOKEN": [
            "Invalid or expired token"
        ]
    }
}
```

---

### 5.2 Set Default Card

```
POST /assisted/cards/{id}/make-default
```

**Headers:**

```
Content-Type: application/json
temp-user-access: <temp_token>
```

**Response (example):**

```json
{
  "success": true,
  "message": "Default card updated",
  "default_card_id": "3"
}
```

Notes:

* Ensures card belongs to the authorized user.
* Updates default selection via `SavedCardRepository::setDefault()`.

---
## Expected Error Responses

**401 – Temp Token Missing**

```json
{
    "status": "error",
    "code": "TOKEN_MISSING",
    "message": "Temp token missing",
    "errors": {
        "TOKEN_MISSING": [
            "Temp token missing"
        ]
    }
}
```

---

**401 – Invalid or Expired Token**

```json
{
    "status": "error",
    "code": "INVALID_TOKEN",
    "message": "Invalid or expired token",
    "errors": {
        "INVALID_TOKEN": [
            "Invalid or expired token"
        ]
    }
}
```

---

**404 – Card Not Found**

```json
{
    "status": "error",
    "code": "NOT_FOUND",
    "message": "Card not found",
    "errors": {
        "NOT_FOUND": [
            "Card not found"
        ]
    }
}
```

---

### 5.3 Assisted Checkout Confirm

This endpoint **initializes the downpayment payment session** and returns Paymob Unified Checkout details.

```
POST /assisted/checkout/confirm
```

**Headers:**

```
Content-Type: application/json
temp-user-access: <temp_token>
```

**Body:**

```json
{
  "cart_uuid": "728ef50f-6d14-42df-83da-7b5bba574af2",
  "default_card_id": 13
}
```

**Response**

```json
{
  "success": true,
  "message": "Payment successful",
  "data": {
    "payment_track": "YC3ANYBR5GVG",
    "amount": 75,
    "currency": "OMR",
    "quotation_id": 118,
    "cart_uuid": "d97843bb-8589-4c89-8b63-9a6b12232605",
    "transaction_id": "PMB-987654321"
  }
}
```
---
#### Assisted Checkout Confirm Retry & Cooldown 

* Each user + quotation + card combination can attempt **payment only once**.

* If a payment fails:
  * The API enforces a **1-minute cooldown** before retrying with the same card.
  * Only **one retry** is allowed per failed payment.
* Attempt Limits: Payment attempts are tracked using the following scope:
  ```json
  user_id + quotation_id + card_id
  ```
* If a retry attempt is made **before the cooldown expires**, the API returns **HTTP 429**:

---
## Expected Error Responses

**429 – Too Many Requests**

```json
{
    "status": "error",
    "code": "PAYMENT_FAILED",
    "message":"Previous payment failed. Please wait 37 seconds before retrying with the same card.",
    "errors": {
        "retry_after_seconds": [
            37,
        ],
    }
}
```
---

If the retry limit is already used, the API returns HTTP 429:

**429 – Too Many Requests**

```json
{
    "status": "error",
    "code": "TOO_MANY_ATTEMPTS",
    "message":"Retry already used for this payment attempt with the selected card.",
    "errors": {
        "TOO_MANY_ATTEMPTS": [
            "Retry count 3 exceeded allowed retry count of 2.",
        ],
    }
}
```
---

**422 – Validation Error**

```json
{
    "status": "error",
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "errors": {
        "cart_uuid": [
            "The cart uuid field is required."
        ]
    }
}
```

---

**422 – Validation Error**

```json
{
    "status": "error",
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "errors": {
        "default_card_id": [
            "The default card id field is required."
        ]
    }
}
```

---

**401 – Temp Token Missing**

```json
{
    "status": "error",
    "code": "TOKEN_MISSING",
    "message": "Temp token missing",
    "errors": {
        "TOKEN_MISSING": [
            "Temp token missing"
        ]
    }
}
```

---

**401 – Invalid or Expired Token**

```json
{
    "status": "error",
    "code": "INVALID_TOKEN",
    "message": "Invalid or expired token",
    "errors": {
        "INVALID_TOKEN": [
            "Invalid or expired token"
        ]
    }
}
```

---

**404 – User Not Found**

```json
{
    "status": "error",
    "code": "NOT_FOUND",
    "message": "User not found",
    "errors": {
        "NOT_FOUND": [
            "User not found"
        ]
    }
}
```

---

**404 – Invalid Cart Reference**

```json
{
    "status": "error",
    "code": "INVALID_CART_REF",
    "message": "Invalid cart reference",
    "errors": {
        "INVALID_CART_REF": [
            "Invalid cart reference"
        ]
    }
}
```

---

**403 – Cart Ownership Violation**

```json
{
    "status": "error",
    "code": "INVALID_CART",
    "message": "Cart does not belong to this user",
    "errors": {
        "INVALID_CART": [
            "Cart does not belong to this user"
        ]
    }
}
```

---

**400 – Cart Expired**

```json
{
    "status": "error",
    "code": "EXPIRED_CART",
    "message": "Cart has expired",
    "errors": {
        "EXPIRED_CART": [
            "Cart has expired"
        ]
    }
}
```

---

**400 – Card Not Found**

```json
{
    "status": "error",
    "code": "NOT_FOUND",
    "message": "Card not found",
    "errors": {
        "NOT_FOUND": [
            "Card not found"
        ]
    }
}
```

---

**400 – No Saved Cards Available**

```json
{
    "status": "error",
    "code": "NOT_FOUND",
    "message": "No card found, please add card to your Tasheel account",
    "errors": {
        "NOT_FOUND": [
            "No card found, please add card to your Tasheel account"
        ]
    }
}
```

---

**400 – Invalid Payment Amount**

```json
{
    "status": "error",
    "code": "INVALID_AMOUNT",
    "message": "Invalid payment amount",
    "errors": {
        "INVALID_AMOUNT": [
            50
        ]
    }
}
```

---

**402 – Payment Failed**

```json
{
    "status": "error",
    "code": "PAYMENT_FAILED",
    "message": "Previous payment failed. Please wait 60 seconds before retrying with the same card.",
    "errors": {
        "retry_after_seconds": [
            50
        ]
    }
}
```
---

**400 – Payment Gateway Not Available**

```json
{
    "status": "error",
    "code": "PAYMENT_GATEWAY_NOT_FOUND",
    "message": "Payment gateway not available",
    "errors": {
        "PAYMENT_GATEWAY_NOT_FOUND": [
            "Payment gateway not available"
        ]
    }
}
```

---

**500 – Gateway Configuration Error**

```json
{
    "status": "error",
    "code": "CONFIGURATION_ERROR",
    "message": "Payment gateway configuration error (Paymob keys missing)",
    "errors": {
        "CONFIGURATION_ERROR": [
            "Payment gateway configuration error"
        ]
    }
}
```
---

**500 – Unexpected Processing Error**

```json
{
    "status": "error",
    "code": "PAYMENT_FAILED",
    "message": "Payment failed. Please wait 1 minute before retrying with the same card.",
    "errors": {
        "PAYMENT_FAILED": [
            "Payment failed."
        ]
    }
}
```

---
### 6 Get Status
```
GET /checkout/get-status
```
Use the same JWT access token generated in 2.2 Issue Access Token.

**Headers:**
```
temp-user-access: <temp_token>
Content-Type: application/json
```
```
**Body Parameters:**
- `refCode` (optional): Merchant reference code used for idempotency. It equals `merchantId` concatenated with the merchant’s order number (e.g., `tasheel123-ORD98765`). Must exist in `tasheel_payment_links` table.
- `cartId` (optional): The cart UUID returned from Create Cart (e.g., `cart_id`). Must exist in `tasheel_payment_links` table.
- `txnId` (optional): Transaction ID provided by Tasheel in the webhook request to your `callback_url` when Tasheel calls the merchant’s callback. Use if `refCode` or `cartId` is unavailable. Must exist in `payments` table.

**Sample Request:**
```json
{
  "refCode": "ref_1234567890_1627056000",
  "cartId": "1b796418-d380-431c-a083-830fe2ec295c",
  "txnId": "RPBK3THJQ7RB"
}
```
**Response:**
```json
{
  "message": "Cart status",
  "data": {
    "uuid": "UUID123",
    "referenceCode": "tasheel123-ORD98765",
    "payment": {
      "txn_id": "TXN123456",
      "created_at": "2025-11-25T10:30:00Z",
      "try": 1,
      "status": "completed", // Status options: 1 => "completed", 2 => "pending", 3 => "cancel", default => "unknown".
      "note": "Downpayment received",
      "amount": 50.00,
      "final_amount": 50.00,
      "currency": "OMR",
      "ref_code": "tasheel123-ORD98765"
    },
    "quotation": {
      "title": "Cart for John Doe",
      "description": "Travel package",
      "package": {
        "name": "Holiday Package",
        "details": "Includes flight and hotel"
      },
      "plan": {
        "installments": [
          {"amount": 50, "due_date": "2025-12-01"},
          {"amount": 50, "due_date": "2026-01-01"}
        ]
      },
      "base_amount": 150.00,
      "total_amount": 150.00,
      "vat_rate": 5,
      "currency": "OMR",
      "quotationNumber": "QTN98765"
    },
    "user": {
      "uuid": "USR123",
      "branch": {"name": "Main Branch"},
      "branchStaff": {"name": "Staff Name"},
      "accountNumber": "ACC12345",
      "firstName": "John",
      "lastName": "Doe",
      "username": "johndoe",
      "email": "john@example.com",
      "countryCode": "+968",
      "mobile": "91234567",
      "balance": 500.00,
      "image": "https://cdn.tasheel.com/user123.jpg",
      "address": "Muscat, Oman",
      "status": "active",
      "isCompany": false,
      "malaaCreditScore": 720
    },
    "requestData": {
      "merchantId": "tasheel123",
      "customerPhone": "96891234567",
      "cartItems": [
        {"productId": "FLIGHT001", "quantity": 1, "price": 150, "total": 150}
      ],
      "totalAmount": 150,
      "referenceCode": "tasheel123-ORD98765"
    }
  }
}
```

#### Status interpretation for payment.status
- completed: Downpayment paid successfully
- pending: Payment initialized but still pending
- cancel: Payment rejected or declined
- unknown: Treated as cancel

## 7. Security Considerations


* OTP explicitly confirms user consent before any card/payment actions.
* Temp token is short-lived (10 minutes) and cannot be used after expiry.
* Card exposure is limited to masked metadata (last4, brand, expiry).
* Cart ownership validation prevents cross-user payment attempts.

---


