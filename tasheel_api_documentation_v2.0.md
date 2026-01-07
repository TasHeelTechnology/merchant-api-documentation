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

**Processing Flow:**

* Validates `merchant_uuid` exists
* Validates user by `mobile`
* Blocks assisted checkout if user has overdue installments
* Creates a `MerchantTempAuth` session
* Sends OTP via SMS
* Applies short cooldown to prevent spamming

**Response (example):**

```json
{
  "success": true,
  "message": "OTP sent successfully",
  "otp_id": 98
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

**Processing Flow:**

* Validates temp token (and expiry)
* Resolves user from token
* Validates `cart_uuid` exists and belongs to that user
* Validates cart is not expired
* Validates the selected saved card belongs to the user
* Calculates downpayment using `TasHeel_Algorithm(...)`
* Creates a `TasHeelPayment` record (pending)
* Calls Paymob saved card / intention logic
* Returns a **checkout session** payload (`client_secret`, `checkout_url`, etc.)

**Response**

```json
{
  "success": true,
  "message": "Assisted checkout initialized",
  "data": {
    "payment_track": "YC3ANYBR5GVG",
    "ompay_order_id": 1831184,
    "amount": 75,
    "currency": "OMR",
    "client_secret": "omn_csk_test_...beeb4c",
    "checkout_url": "https://oman.paymob.com/unifiedcheckout/?publicKey=...&clientSecret=...",
    "quotation_id": 118,
    "cart_uuid": "d97843bb-8589-4c89-8b63-9a6b12232605"
  }
}
```

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

