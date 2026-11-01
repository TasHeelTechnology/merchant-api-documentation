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
  "status" : "success",
  "code" : "OTP_SENT",
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
  "status" : "success",
  "code" : "OTP_ALREADY_SENT",
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
  "status" : "success",
  "code" : "OTP_VERIFIED",
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
  "status" : "success",
  "code" : "CARDS_RETRIEVED",
  "data": [
    {
      "id": 38,
      "last4": "3343",
      "brand": "VISA",
      "exp_month": 12,
      "exp_year": 2029,
      "is_default": true,
      "gateway": "OMPay"
    },
    {
      "id": 37,
      "last4": "2346",
      "brand": "MasterCard",
      "exp_month": 12,
      "exp_year": 2029,
      "is_default": false,
      "gateway": "Paymob"
    }
  ]
}
```

Notes:

* The effective default card is returned first, followed by the most recent other supported card. The current implementation returns at most two cards.
* Only masked metadata for cards owned by a supported provider is returned. Card tokens are never returned.
* A legacy card with missing or incorrect provider ownership is excluded until corrected.
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
  "status" : "success",
  "code" : "DEFAULT_CARD_UPDATED",
  "message": "Default card updated",
  "default_card_id": "3"
}
```

Notes:

* Ensures card belongs to the authorized user.
* Updates default selection via `SavedCardRepository::setDefault()`.
* The card must belong to a supported provider. Making it the default does not migrate it to the currently primary gateway.

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

This endpoint **initializes the downpayment payment session** and routes the selected card through its owning provider.

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

**Paymob saved-card success (HTTP 200)**

```json
{
  "success" : true,
  "status" : "success",
  "code" : "PAYMENT_SUCCESS",
  "message": "Payment successful",
  "data": {
    "payment_track": "YC3ANYBR5GVG",
    "amount": 75,
    "currency": "OMR",
    "quotation_id": 118,
    "cart_uuid": "d97843bb-8589-4c89-8b63-9a6b12232605",
    "transaction_id": "PMB-987654321"
    "provider": "paymob",
    "method_code": "125",
    "ompay_order_id": "PAYMOB_ORDER_ID",
    "checkout_url": null,
    "redirect_url": null
  }
}
```

`ompay_order_id` is a legacy field name and may contain a Paymob order identifier.

**OMPay frictionless saved-card success (HTTP 200)**

```json
{
  "success": true,
  "status": "success",
  "code": "PAYMENT_SUCCESS",
  "message": "Payment successful",
  "data": {
    "payment_track": "YC3ANYBR5GVG",
    "amount": 75,
    "currency": "OMR",
    "quotation_id": 118,
    "cart_uuid": "728ef50f-6d14-42df-83da-7b5bba574af2",
    "provider": "ompay",
    "method_code": "100",
    "ompay_order_id": "8a87bb37-b103-4d5e-af91-5480d38e01d2",
    "transaction_id": "8a87bb37-b103-4d5e-af91-5480d38e01d2",
    "checkout_url": null,
    "redirect_url": null
  }
}
```

**OMPay 3DS authentication required (HTTP 202)**

```json
{
  "success": true,
  "status": "action_required",
  "code": "PAYMENT_ACTION_REQUIRED",
  "message": "Additional payment authentication is required.",
  "data": {
    "payment_track": "YC3ANYBR5GVG",
    "amount": 75,
    "currency": "OMR",
    "quotation_id": 118,
    "cart_uuid": "728ef50f-6d14-42df-83da-7b5bba574af2",
    "provider": "ompay",
    "method_code": "100",
    "ompay_order_id": "8a87bb37-b103-4d5e-af91-5480d38e01d2",
    "transaction_id": "8a87bb37-b103-4d5e-af91-5480d38e01d2",
    "checkout_url": "https://api.sandbox.truepay.ompay.om/api/v1/transactions/8a87bb37-b103-4d5e-af91-5480d38e01d2/3ds",

    "otp_required": true
  }
}
```

Open `checkout_url` in a browser or supported web view. Tasheel does not collect the issuer OTP. The return URL supplied to OMPay must be publicly reachable over HTTPS; `localhost` cannot be opened from the customer's phone.

**Paymob safely falls back to OMPay (HTTP 200)**

```json
{
    "success": true,
    "status": "success",
    "code": "PAYMENT_FALLBACK_INITIALIZED",
    "message": "Paymob is unavailable. Continue payment through OMPay.",
    "data": {
        "payment_track": "EVVEXOEF2RAP",
        "amount": 3,
        "currency": "OMR",
        "quotation_id": 1726,
        "cart_uuid": "76d9f940-e8ae-48be-a514-c40540e604de",
        "provider": "ompay",
        "method_code": "100",
        "ompay_order_id": null,
        "transaction_id": "07295e53-dd34-4356-be46-b672e9420e41",
        "checkout_url": "https://api.sandbox.truepay.ompay.om/api/v1/transactions/07295e53-dd34-4356-be46-b672e9420e41/pay-page",
        "redirect_url": "https://api.sandbox.truepay.ompay.om/api/v1/transactions/07295e53-dd34-4356-be46-b672e9420e41/pay-page",
        "fallback_triggered": true,
        "fallback_from": "paymob",
        "fallback_to": "ompay",
        "fallback_reason": "GATEWAY_UNAVAILABLE",
        "retry_requires_card_details": true
    }
}
```

The fallback creates a fresh OMPay bank-hosted checkout. The Paymob saved-card token is never sent to OMPay.

**Ambiguous provider result (HTTP 202)**

```json
{
  "status": "pending",
  "code": "PAYMENT_PENDING",
  "message": "Payment status is being verified.",
  "data": { "payment_track": "YC3ANYBR5GVG" }
}
```

Do not retry or switch providers. Poll the documented status endpoint with `payment_track` until Tasheel reconciles the authoritative result.
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

**503 – Secondary Gateway Unavailable**

```json
{
  "status": "error",
  "code": "SECONDARY_GATEWAY_UNAVAILABLE",
  "message": "OMPay secondary gateway is unavailable or disabled.",
  "errors": {
    "SECONDARY_GATEWAY_UNAVAILABLE": [
      "Provider initialization error"
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

## 7. BRS Booking Status

This endpoint allows BRS to report the final booking status after the Tasheel downpayment has been completed. For `NOT_TICKETED` bookings, Tasheel initiates a refund for every successful downpayment capture associated with the cart's quotation, using the provider recorded on each original payment.

```
POST /api/checkout/booking-status
```

**Headers:**

```http
Authorization: Bearer <BRS access token>
Content-Type: application/json
Accept: application/json
```

The bearer token must authenticate a merchant through the existing `api_branch_staff` guard. The authenticated merchant must own the specified cart.

**Body Parameters:**

- `cartId` (optional): Payment-link UUID. Supply `cartId` or `refCode`; `cartId` is preferred.
- `refCode` (optional): Merchant reference code associated with the cart. If both identifiers are supplied, they must identify the same cart.
- `status` (required): Must be either `TICKETED` or `NOT_TICKETED`.
- `bookingReference` (optional): Booking reference or PNR supplied by BRS.
- `reason` (optional): Reason associated with the booking result.
- `timestamp` (optional): ISO-8601-compatible date/time. UTC (`Z`) is preferred.

**Sample Request:**

```json
{
  "cartId": "payment-link-uuid",
  "refCode": "optional-reference-that-must-match-the-cart",
  "status": "TICKETED",
  "bookingReference": "PNR123456",
  "timestamp": "2026-10-29T10:30:00Z"
}
```

**Response (example):**

```json
{
  "status": "success",
  "code": "BOOKING_STATUS_RECORDED",
  "message": "Booking status recorded successfully",
  "data": {
    "cartId": "payment-link-uuid",
    "bookingStatus": "TICKETED",
    "refund": null,
    "duplicate": false
  }
}
```

### 7.1 Booking Status and Refund Behaviour

- For `TICKETED`, the booking status is recorded and no refund is created.
- For `NOT_TICKETED`, Tasheel refunds every successful downpayment capture for the cart's quotation through its original provider (Paymob or OMPay).
- The refund includes the captured service fee and does not apply the normal non-refundable-fee calculation.
- Future unpaid installments are not refunded.
- An unconfirmed gateway result remains `pending` and is retried through the existing pending-gateway-refund scheduler.
- Repeated API requests resume the same refund request rather than creating another one.

A successful `NOT_TICKETED` response includes a `refund` object:

```json
{
    "status": "success",
    "code": "BOOKING_STATUS_RECORDED",
    "message": "Booking failure recorded and refund initiated",
    "data": {
        "cartId": "3db6af17-ef2c-4309-a229-0f6a4cc56c20",
        "bookingStatus": "NOT_TICKETED",
        "refund": {
            "refundRequestId": 172,
            "amount": "13.000",
            "currency": "OMR",
            "status": "completed"
        },
        "duplicate": false
    }
}
```

The refund amount is returned as a three-decimal OMR value. The refund status is either `pending` or `completed` and reflects the confirmed gateway result; a pending refund is never reported as completed.

### 7.2 Idempotency and Conflicts

- An identical retry returns HTTP `200` with code `BOOKING_STATUS_ALREADY_PROCESSED` and `duplicate: true`.
- If a terminal booking status has already been recorded, a request with the opposite status returns HTTP `409` with code `BOOKING_STATUS_CONFLICT`.
- Ordinary BRS requests cannot reverse a terminal booking status.

```json
{
    "status": "success",
    "code": "BOOKING_STATUS_ALREADY_PROCESSED",
    "message": "This booking status has already been processed",
    "data": {
        "cartId": "3db6af17-ef2c-4309-a229-0f6a4cc56c20",
        "bookingStatus": "NOT_TICKETED",
        "refund": {
            "refundRequestId": 172,
            "amount": "13.000",
            "currency": "OMR",
            "status": "completed"
        },
        "duplicate": true
    }
}
```

### 7.3 Expected Error Responses

**401 – Unauthenticated**

```json
{
  "status": "error",
  "code": "UNAUTHENTICATED",
  "message": "Invalid or missing access token"
}
```

Returned when the bearer token is missing, expired, invalid, or cannot authenticate through the `api_branch_staff` guard.

---

**403 – Unauthorized**

```json
{
  "status": "error",
  "code": "UNAUTHORIZED",
  "message": "Merchant not authorized for this cart"
}
```

Returned when the cart belongs to a different merchant from the merchant authenticated by the bearer token.

---

**404 – Cart Not Found**

```json
{
  "status": "error",
  "code": "CART_NOT_FOUND",
  "message": "The specified cart could not be found"
}
```

Returned when no active payment link matches the supplied `cartId` or `refCode`, or when the cart has been deleted.

---

**409 – Downpayment Not Successful**

```json
{
  "status": "error",
  "code": "DOWN_PAYMENT_NOT_SUCCESSFUL",
  "message": "The cart has no successful down payment"
}
```

Returned when Tasheel has no successful quotation or downpayment payment for the cart. Confirm the payment status and retry only after it has successfully settled.

---

**409 – Booking Status Conflict**

```json
{
  "status": "error",
  "code": "BOOKING_STATUS_CONFLICT",
  "message": "The booking has already been processed with a different status"
}
```

Returned when the cart already has a terminal booking status and BRS sends the opposite status.

---

**422 – Invalid Booking Status**

```json
{
  "status": "error",
  "code": "INVALID_BOOKING_STATUS",
  "message": "Status must be TICKETED or NOT_TICKETED",
  "errors": {
    "status": [
      "The selected status is invalid."
    ]
  }
}
```

Returned when `status` is missing or is not exactly `TICKETED` or `NOT_TICKETED`. It can also occur when the JSON body is malformed or sent with an incorrect content type.

---

**422 – Validation Failed**

```json
{
  "status": "error",
  "code": "VALIDATION_FAILED",
  "message": "Validation failed",
  "errors": {
    "cartId": [
      "The cart id field is required when ref code is not present."
    ]
  }
}
```

Returned when required identifiers are missing, a value has the wrong type, a field exceeds its maximum length, or `timestamp` is not a valid date.

---

**422 – Cart Identifier Mismatch**

```json
{
  "status": "error",
  "code": "CART_IDENTIFIER_MISMATCH",
  "message": "cartId and refCode do not identify the same cart"
}
```

Returned when both identifiers are supplied but the `refCode` does not belong to the cart selected by `cartId`.

---

**422 – Refund Initiation Failed**

```json
{
  "status": "error",
  "code": "REFUND_INITIATION_FAILED",
  "message": "No successful down payment found for this cart",
  "data": {
    "cartId": "payment-link-uuid",
    "bookingStatus": "NOT_TICKETED"
  }
}
```

Returned only for `NOT_TICKETED` when Tasheel cannot find a successful downpayment capture that can be refunded. A locally successful payment without its provider transaction identifier requires reconciliation before retrying.

---

**500 – Booking Status Processing Failed**

```json
{
  "status": "error",
  "code": "BOOKING_STATUS_PROCESSING_FAILED",
  "message": "Booking status could not be processed"
}
```

Returned for an unexpected database or application failure while resolving, locking, or recording the booking status. BRS may retry the same request because identical requests are idempotent.

### 7.4 Framework-Level Responses

- HTTP `429 Too Many Requests` is returned when the checkout endpoint rate limit is exceeded. BRS should retry with backoff.
- A maintenance response may be returned while Tasheel maintenance middleware is active.
- These responses may use the standard Laravel/Tasheel middleware format rather than the endpoint-specific JSON structures above.

---

## 8. Security Considerations


* OTP explicitly confirms user consent before any card/payment actions.
* Temp token is short-lived (10 minutes) and cannot be used after expiry.
* Card exposure is limited to masked metadata (last4, brand, expiry).
* Cart ownership validation prevents cross-user payment attempts.

---


