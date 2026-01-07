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

---
## Expected Error Responses

**400 – Validation Error**
Occurs when required fields are missing or invalid.

```json
{
  "success": false,
  "message": "The mobile field is required."
}
```

---

**404 – User Not Found**

```json
{
  "success": false,
  "message": "No user found for this mobile"
}
```

---

**403 – User Not Eligible (Overdue Installments)**

```json
{
  "success": false,
  "message": "User has overdue installments. Assisted checkout blocked."
}
```

---

**429 – OTP Cooldown Active**

```json
{
  "success": false,
  "message": "OTP already sent. Please wait before requesting again."
}
```

---

**500 – OTP Generation Failure**

```json
{
  "success": false,
  "message": "OTP creation failed"
}
```
---

### 2.2 Verify OTP & Issue Temporary Access Token

```
POST /merchant/temp-auth/verify
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
  "success": false,
  "message": "The otp must be 6 digits."
}
```

---

**404 – User Not Found**

```json
{
  "success": false,
  "message": "No user found for this mobile"
}
```

---

**404 – Invalid OTP Session**

```json
{
  "success": false,
  "message": "Invalid OTP session"
}
```

---

**400 – Invalid OTP Target**

```json
{
  "success": false,
  "message": "Invalid OTP target"
}
```

---

**403 – Merchant Mismatch**

```json
{
  "success": false,
  "message": "Merchant mismatch"
}
```

---

**400 – OTP Already Used**

```json
{
  "success": false,
  "message": "OTP session already used"
}
```

---

**400 – OTP Expired**

```json
{
  "success": false,
  "message": "OTP session expired"
}
```

---

**400 – Incorrect OTP**

```json
{
  "success": false,
  "message": "Invalid OTP"
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

## 4. Rate Limits

* OTP request: throttled using cooldown per mobile (example: ~15 seconds)
* OTP verify: limited by OTP manager rules and expiry
* Assisted APIs (`/assisted/cards`, `/assisted/checkout/confirm`): naturally constrained by **10-minute token expiry**

---

## 5. Idempotency

Assisted checkout prevents replay through:

* **Single-use OTP verification**
* **Short-lived temp token**
* **Cart ownership validation** (cart must belong to the authorized user)

If a client retries confirm requests, the backend will create new payment sessions unless you additionally enforce idempotency on `cart_uuid` + `user_id` at your application layer.

---

## 6. Endpoints

### 6.1 List User’s Saved Cards

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
  "success": false,
  "message": "Temp token missing"
}
```

---

**401 – Invalid or Expired Token**

```json
{
  "success": false,
  "message": "Invalid or expired token"
}
```

---

### 6.2 Set Default Card

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
  "success": false,
  "message": "Temp token missing"
}
```

---

**401 – Invalid or Expired Token**

```json
{
  "success": false,
  "message": "Invalid or expired token"
}
```

---

**404 – Card Not Found**

```json
{
  "success": false,
  "message": "Card not found"
}
```

---

### 6.3 Assisted Checkout Confirm

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
## Expected Error Responses

**400 – Validation Error**

```json
{
  "success": false,
  "message": "The cart uuid field is required."
}
```

---

**401 – Temp Token Missing**

```json
{
  "success": false,
  "message": "Temp token missing"
}
```

---

**401 – Invalid or Expired Token**

```json
{
  "success": false,
  "message": "Invalid or expired token"
}
```

---

**404 – User Not Found**

```json
{
  "success": false,
  "message": "User not found"
}
```

---

**404 – Invalid Cart Reference**

```json
{
  "success": false,
  "message": "Invalid cart reference"
}
```

---

**403 – Cart Ownership Violation**

```json
{
  "success": false,
  "message": "Cart does not belong to this user"
}
```

---

**400 – Cart Expired**

```json
{
  "success": false,
  "message": "Cart has expired"
}
```

---

**400 – Card Not Found**

```json
{
  "success": false,
  "message": "Card not found"
}
```

---

**400 – No Saved Cards Available**

```json
{
  "success": false,
  "message": "No card found, please add card to your Tasheel account"
}
```

---

**400 – Invalid Payment Amount**

```json
{
  "success": false,
  "message": "Invalid payment amount"
}
```

---

**400 – Payment Gateway Not Available**

```json
{
  "success": false,
  "message": "Payment gateway not available"
}
```

---

**500 – Gateway Configuration Error**

```json
{
  "success": false,
  "message": "Payment gateway configuration error (Paymob keys missing)"
}
```

---

**400 – Payment Failed**

```json
{
  "success": false,
  "message": "Payment failed"
}
```

---

**500 – Unexpected Processing Error**

```json
{
  "success": false,
  "message": "Failed to process payment"
}
```

---

## 7. Laravel Middleware Notes

* `ValidateTempUserAccess` middleware:

  * Requires header: `temp-user-access`
  * Validates token exists and not expired
  * Injects `temp_user_id` into request context (`$request->temp_user_id`)
* All assisted card + checkout endpoints are protected by this middleware.

---

## 8. Security Considerations

* OTP explicitly confirms user consent before any card/payment actions.
* Temp token is short-lived (10 minutes) and cannot be used after expiry.
* Card exposure is limited to masked metadata (last4, brand, expiry).
* Cart ownership validation prevents cross-user payment attempts.

---

## 9. Versioning, Timeouts & Retries

* Versioning: `/`
* Token validity: 10 minutes
* HTTP client timeout: 30s, connect timeout: 10s
* Gateway errors are handled as failures and payment record is marked rejected where applicable.

---

