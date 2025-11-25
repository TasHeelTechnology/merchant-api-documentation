# Tasheel Travel Now Pay Later - Merchant API Integration Guide

## 1. Overview
Tasheel BNPL API enables merchants to create invoices for customers who want to travel now and pay later. Customers can view invoices in the Tasheel app and pay the downpayment. Merchants can:
- Check customer eligibility and payment plan.
- Create invoice (cart) if eligible.
- Authenticate to get JWT token.
- Check payment status after downpayment.

---

## 2. Authentication & Authorization
Merchants authenticate using OAuth flow.

### 2.1 Create OAuth Client
```
POST /createOauthClient
```
**Body:**
```json
{
  "name": "MerchantApp",
  "redirect": "https://merchant.com/callback",
  "grant_type": "client_credentials"
}
```
**Response:**
```json
{
  "client_id": 123,
  "client_secret": "abc123xyz",
  "name": "MerchantApp",
  "redirect": "https://merchant.com/callback",
  "grant_type": "client_credentials"
}
```

### 2.2 Issue Access Token
```
POST /oauth/token
```
**Body:**
```json
{
  "client_id": 123,
  "client_secret": "abc123xyz"
}
```
**Response:**
```json
{
  "access_token": "jwt-token",
  "token_type": "Bearer",
  "expires_at": "2025-12-01T12:00:00Z"
}
```
Use `Authorization: Bearer <jwt-token>` for all subsequent requests.

---

## 3. Base URL & Environment
- Sandbox: `https://sandbox.tasheelbnpl.com/api`
- Production: `https://api.tasheelbnpl.com`

---

## 4. Rate Limits
- `/oauth/token`: 30 requests/minute
- `/checkout/summery`: 20 requests/minute
- `/checkout/status`: 25 requests/minute

---

## 5. Idempotency
Tasheel uses `referenceCode` in the request body for idempotency. Merchants must concatenate their `merchantId` and `merchantOrderId`:
```
"referenceCode": "tasheel123-ORD98765"
```
If the same referenceCode is sent again, Tasheel will return the existing cart instead of creating a new one.

---

## 6. Endpoints

### 6.1 Get Summary
```
POST /checkout/summery
```
**Body:**
```json
{
  "merchantId": "tasheel123",
  "customerPhone": "96891234567",
  "cartItems": [
    {"productId": "FLIGHT001", "quantity": 1, "price": 150, "total": 150}
  ],
  "totalAmount": 150
}
```
**Response:**
```json
{
  "message": "Cart summary",
  "cart": {
    "eligible": true,
    "downpayment": 50,
    "installments": [
      {"amount": 50, "due_date": "2025-12-01"},
      {"amount": 50, "due_date": "2026-01-01"}
    ]
  }
}
```

### 6.2 Create Cart
```
POST /checkout/cart
```
**Body:**
```json
{
  "merchantId": "tasheel123",
  "customerPhone": "96891234567",
  "customerEmail": "customer@example.com",
  "quotationType": "refundable",
  "cartItems": [
    {"productId": "FLIGHT001", "quantity": 1, "price": 150, "total": 150}
  ],
  "totalAmount": 150,
  "referenceCode": "tasheel123-ORD98765",
  "cartValidity": "02:00",
  "callback_url": "https://merchant.com/webhook"
}
```
**Response:**
```json
{
  "message": "Cart created successfully",
  "cart_id": "UUID123",
  "ref_code": "tasheel123-ORD98765",
  "otp_expire_at": "2025-11-25T12:10:00Z",
  "cart_expire_at": "2025-11-25T14:10:00Z",
  "redirect_url": "https://tasheelbnpl.com/pay/UUID123"
}
```

### 6.3 Get Status
```
GET /checkout/status?cartId=UUID123
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
      "status": "completed",
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

### 6.4 Delete Cart
```
DELETE /checkout/cart?cartId=UUID123
```
**Response:**
```json
{
  "message": "Cart deleted successfully"
}
```

---

## 7. Webhooks
Merchant sets `callback_url` in Create Cart request. Tasheel sends POST:
```json
{
  "txnId": "TXN123456",
  "status": "downpayment_paid"
}
```
Merchant then calls Get Status API.

---

## 8. Laravel Middleware Notes
- `auth:passport` for merchant routes
- `auth:api_branch_staff` for status/delete
- Throttle limits applied per route

---

## 9. Versioning, Timeouts & Retries
- Versioning: `/`
- Timeout: 30s
- Retry: Exponential backoff for transient errors

