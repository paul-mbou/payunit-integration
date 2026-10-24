# Payunit API reference

**Last verified against official docs:** 2026-08-31  
**Docs home:** https://developer.payunit.net/

Base URL: **`https://gateway.payunit.net`**

> **Note:** The getting-started page once listed `gateway.paynit.net` (typo). Always use **`https://gateway.payunit.net`**.

---

## Authentication headers (all REST endpoints)

Every call requires **both** merchant Basic auth **and** the application token:

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |
| `Authorization` | `Basic {Base64(api_user:api_password)}` |
| `x-api-key` | Application token (sandbox or live) |
| `mode` | `test` or `live` |

REST API and the official Node.js SDK use `mode: test | live`.  
The Flutter SDK widget documents `mode: live | sandbox` — treat **`sandbox` as equivalent to REST `test`**.

Doc: https://developer.payunit.net/getting-started

---

## REST API — Gateway flow (Collections)

### Initialize payment

`POST /api/gateway/initialize`  
Doc: https://developer.payunit.net/rest-api/initialize-payment (updated **2024-12-01**)

**Body:**

| Field | Required | Notes |
|-------|----------|-------|
| `total_amount` | Yes | Integer amount (XAF) |
| `currency` | Yes | `XAF` |
| `transaction_id` | Yes | Unique in your system; **alphanumeric only** (special chars break Orange Money); **max 18 characters** for Orange Money |
| `return_url` | Yes | HTTPS redirect after payment |
| `notify_url` | No | HTTPS webhook to your server |
| `payment_country` | No | ISO alpha-2, e.g. `CM` — filters displayed PSPs |
| `pay_with` | No | Pre-select gateway shortcode, e.g. `CM_MTNMOMO`, `CM_ORANGE` |
| `redirect_on_failed` | No | `yes` or `no` — redirect user when payment fails |
| `custom_fields` | No | Object — merchant metadata echoed in dashboard/webhooks |

**Response `data` highlights:**

| Field | Description |
|-------|-------------|
| `t_id`, `t_sum`, `t_url` | Encrypted refs for gateways / makepayment |
| `transaction_id` | Payunit-side transaction id (may differ from yours) |
| `transaction_url` | Hosted payment page URL — redirect user here |
| `providers[]` | Subscribed PSPs: `shortcode`, `name`, `logo`, `status` (`ACTIVE`), `country` |

**`providers[].country` shape (2024+):**

```json
{
  "country_name": "Cameroon",
  "country_code": "CM"
}
```

---

### Get providers

`GET /api/gateway/gateways?t_id={t_id}&t_sum={t_sum}&t_url={t_url}`

Returns array of `{ shortcode, name, logo, country: { country_name, country_code } }`.

Common shortcodes (depend on PSP subscription): `CM_ORANGE`, `CM_MTNMOMO`.

Doc: https://developer.payunit.net/rest-api/get-providers

---

### Make payment (Mobile Money — custom UI)

`POST /api/gateway/makepayment`  
Doc: https://developer.payunit.net/rest-api/make-payment (updated **2024-12-01**)

| Field | Required | Notes |
|-------|----------|-------|
| `gateway` | Yes | Provider shortcode, e.g. `CM_ORANGE`, `CM_MTNMOMO` |
| `transaction_id` | Yes | Same as initialize |
| `phone_number` | Yes | Customer MSISDN, compatible with gateway |
| `amount` | Yes | Must match initialized amount |
| `currency` | Yes | `XAF` |
| `return_url` | Yes | HTTPS |
| `notify_url` | No | HTTPS webhook |
| `paymentType` | Yes | `"button"` |

**Response `data`:** `payment_status` (often `PENDING`), `provider_transaction_id`.

Credit card via REST makepayment is documented; **cards not available in sandbox**.

---

### Get payment status

`GET /api/gateway/paymentstatus/{transactionID}`

`transactionID` = your `transaction_id` from initialize.

**Response `data.transaction_status`:** `PENDING` | `SUCCESS` | `FAILED` | `CANCELLED`

Also returns: `transaction_amount`, `transaction_currency`, `transaction_gateway`, `callback_url`, `notify_url`, `purchaseRef`, `message`.

Doc: https://developer.payunit.net/rest-api/get-payment-status

---

### Webhook payload (`notify_url`)

Payunit POSTs JSON to your server (no documented HMAC — validate against a stored pending payment row):

Doc: https://developer.payunit.net/rest-api/overview (updated **2024-12-01**)

```json
{
  "status": "SUCCESS",
  "statusCode": 200,
  "message": "payment has been collected",
  "data": {
    "transaction_amount": 90000,
    "transaction_status": "SUCCESS",
    "transaction_id": "PU1672929285456",
    "purchaseRef": null,
    "notify_url": "https://your-api.com/webhooks/payments/payunit",
    "callback_url": "https://your-app.com/return",
    "transaction_currency": "XAF",
    "transaction_gateway": "CM_ORANGE",
    "message": "payment has been collected"
  }
}
```

**Fulfillment rule:** use `data.transaction_status` (not top-level `status` alone).  
Top-level `status` reflects API envelope success; payment outcome is in `data`.

---

## Checkout API — Invoice / product link

### Initialize checkout

`POST /api/gateway/checkout/initialize`  
Doc: https://developer.payunit.net/checkout/initialize-payment (updated **2024-04-05**)

| Field | Required |
|-------|----------|
| `total_amount` | Yes |
| `transaction_id` | Yes |
| `currency` | Yes (`XAF`) |
| `mode` | Yes — `payment` or `subscription` |
| `success_url` | Yes (HTTPS) |
| `cancel_url` | Yes (HTTPS) |
| `notify_url` | No |
| `payment_country` | No |
| `items` | Yes (array) |
| `items[].price_description.unit_amount` | Yes |
| `items[].product_description.name` | Yes |
| `items[].product_description.image_url` | Yes (HTTPS) |
| `items[].product_description.about_product` | No |
| `items[].quantity` | Yes |
| `meta.phone_number_collection` | Yes |
| `meta.address_collection` | Yes |
| `customer.name` / `email` / `phone` | No (optional, supported by Node SDK) |

**Response:** `data.redirect` — payment link (**15 min** validity).

---

### Process checkout payment

Two endpoints depending on method (doc updated **2025-03-03**):

| Method | Endpoint |
|--------|----------|
| Mobile Money | `POST /api/gateway/invoice/process` |
| Card | `POST /api/gateway/checkout/process` |

**Mobile money body (`/invoice/process`):**

| Field | Required |
|-------|----------|
| `checkout_id` | Yes — from checkout initialize |
| `customer.email` | Yes |
| `shipping.payment_method` | Yes — e.g. `CM_ORANGE` |
| `shipping.payment_info.phone` | Yes (mobile money) |
| `customer.phone`, `customer.name`, `customer.country` | No |
| `shipping.address`, `shipping.phone` | No |

**Card body (`/checkout/process`):** same structure; `shipping.payment_info.card_info` with card fields.

Doc: https://developer.payunit.net/checkout/make-payment

Once purchase completes or is cancelled, the checkout link **expires**.

---

## Error codes

| Status | Description |
|--------|-------------|
| 400 | Provider not supported; transaction already exists |
| 404 | Transaction does not exist |
| 422 | Integrity problem with transaction |

Doc: https://developer.payunit.net/rest-api/error-description

---

## Official Node.js SDK (recommended for Node backends)

Package: **`@payunit/nodejs-sdk`** (requires Node 18+)  
Doc: https://developer.payunit.net/sdk-and-plugins/nodejs (updated **2025-11-26**)

```javascript
import { PayunitClient } from '@payunit/nodejs-sdk';

const client = new PayunitClient({
  baseURL: 'https://gateway.payunit.net',
  apiKey: process.env.PAYUNIT_API_KEY,
  apiUsername: process.env.PAYUNIT_API_USER,
  apiPassword: process.env.PAYUNIT_API_PASSWORD,
  mode: 'test', // or 'live'
});
```

**Collections (maps to REST gateway flow):**

| SDK method | REST equivalent |
|------------|-----------------|
| `client.collections.initiatePayment(...)` | `POST /api/gateway/initialize` |
| `client.collections.makePayment(...)` | `POST /api/gateway/makepayment` |
| `client.collections.initiateAndMakePaymentMobileMoney(...)` | initialize + makepayment in one call |
| `client.collections.getTransactionStatus(id)` | `GET /api/gateway/paymentstatus/{id}` |

**Checkout:** `client.checkout.initialize`, `processPayment`, `getStatus`  
**Invoices:** `client.invoice.createInvoice`, `payInvoice`, `getInvoice` (NORMAL / INSTALLMENT)  
**Disbursements:** `client.disbursement.createDisbursement`, `confirmDisbursement`, `getDisbursementStatus`

Use the SDK on the **server** only. Wrap it in `PayunitAdapter` behind `PaymentGateway`.

---

## Environment variables summary

| Variable | Description |
|----------|-------------|
| `PAYUNIT_API_USER` | Merchant API user |
| `PAYUNIT_API_PASSWORD` | Merchant API password |
| `PAYUNIT_API_KEY` | Application token (`x-api-key`) |
| `PAYUNIT_BASE_URL` | `https://gateway.payunit.net` |
| `PAYUNIT_MODE` | `test` or `live` |
| `PAYUNIT_NOTIFY_URL` | Full HTTPS webhook URL (optional if derived from API base) |
| `API_PUBLIC_BASE_URL` | Your deployed API base for constructing `notify_url` |

---

## Official SDKs (pick per platform)

| Platform | Doc | Notes |
|----------|-----|-------|
| Node.js | https://developer.payunit.net/sdk-and-plugins/nodejs | `@payunit/nodejs-sdk`, `mode: test\|live` |
| Flutter | https://developer.payunit.net/sdk-and-plugins/flutter | `flutter_payunit`, `mode: live\|sandbox` |
| Native Android | https://developer.payunit.net/sdk-and-plugins/native-android | |
| Python | https://developer.payunit.net/sdk-and-plugins/python | |
| PHP | https://developer.payunit.net/sdk-and-plugins/php | |
| WooCommerce | https://developer.payunit.net/sdk-and-plugins/woocommerce | |

SDKs accelerate prototyping; **production integrations should use server-side credentials and webhooks** unless the threat model explicitly accepts client-side keys (sandbox only).

Swagger for sandbox testing: download from merchant dashboard → Getting started.
