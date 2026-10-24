# Payunit integration — examples

Architecture-agnostic patterns. Replace placeholders (`your-api.com`, `your-app.com`, folder paths) with values from the **project scan subagent** (Step 0).

**Last verified:** 2026-08-31

---

## Workflow reminder

1. Launch **explore subagent** → architecture scan ([project-scan-template.md](project-scan-template.md))
2. Apply `~/.cursor/skills/payment-gateway/SKILL.md` (port first)
3. Place **PayunitAdapter** at **recommended paths** from the scan
4. Keep examples' HTTP client / naming aligned with the project

---

## Example 1 — Backend proxy (recommended, any client)

### Your API (server → Payunit)

```http
POST https://gateway.payunit.net/api/gateway/initialize
Authorization: Basic <base64(api_user:api_password)>
x-api-key: <application_token>
mode: test
Content-Type: application/json

{
  "total_amount": 10000,
  "currency": "XAF",
  "transaction_id": "order20260613001",
  "return_url": "https://your-app.com/payments/return",
  "notify_url": "https://your-api.com/webhooks/payments/payunit",
  "payment_country": "CM",
  "pay_with": "CM_MTNMOMO",
  "redirect_on_failed": "yes",
  "custom_fields": {
    "order_id": "order-uuid-123"
  }
}
```

### Public app API (client → your server) — PSP-agnostic

```http
POST https://your-api.com/payments/charges
Content-Type: application/json

{
  "orderId": "order-uuid-123",
  "amountMinor": 10000,
  "currency": "XAF",
  "country": "CM",
  "method": "hosted",
  "idempotencyKey": "order20260613001"
}
```

Response (`Charge`):

```json
{
  "paymentId": "pay_abc",
  "status": "awaiting_user",
  "nextAction": { "type": "redirect", "url": "https://gateway.payunit.net/..." }
}
```

Client follows `nextAction` (browser, WebView, `url_launcher`). Never branch on "Payunit".

---

## Example 2 — Node.js official SDK (server)

```javascript
import { PayunitClient } from '@payunit/nodejs-sdk';

const client = new PayunitClient({
  apiKey: process.env.PAYUNIT_API_KEY,
  apiUsername: process.env.PAYUNIT_API_USER,
  apiPassword: process.env.PAYUNIT_API_PASSWORD,
  mode: process.env.PAYUNIT_MODE || 'test',
});

// Hosted redirect (path B)
const payment = await client.collections.initiatePayment({
  total_amount: 10000,
  currency: 'XAF',
  transaction_id: 'order20260613001',
  return_url: process.env.PAYUNIT_RETURN_URL,
  notify_url: process.env.PAYUNIT_NOTIFY_URL,
  payment_country: 'CM',
});

// payment.data.transaction_url → redirect user

// One-step mobile money (path E)
const moMo = await client.collections.initiateAndMakePaymentMobileMoney({
  total_amount: 10000,
  currency: 'XAF',
  transaction_id: 'order20260613002',
  gateway: 'CM_ORANGE',
  phone_number: '6XXXXXXX',
  notify_url: process.env.PAYUNIT_NOTIFY_URL,
});
```

Wrap SDK calls inside `PayunitAdapter` — checkout still uses `PaymentGateway`.

---

## Example 3 — Express webhook

```javascript
// routes/payments.js — adapt to Nest, Fastify, Koa, etc.
app.post('/webhooks/payments/payunit', express.json(), async (req, res) => {
  const { data } = req.body;
  if (!data?.transaction_id) return res.sendStatus(400);

  const order = await orderRepo.findByTransactionId(data.transaction_id);
  if (!order) return res.sendStatus(404);

  // Idempotent: skip if already final
  if (order.paymentStatus === 'SUCCESS') return res.sendStatus(200);

  // Use data.transaction_status — not req.body.status alone
  if (data.transaction_status === 'SUCCESS') {
    await orderRepo.markPaid(order.id, data);
  } else if (['FAILED', 'CANCELLED'].includes(data.transaction_status)) {
    await orderRepo.markFailed(order.id, data.transaction_status);
  }

  res.sendStatus(200);
});
```

---

## Example 4 — Python / FastAPI initialize + webhook

```python
import os, base64, httpx
from fastapi import APIRouter, Request

router = APIRouter(prefix="/payments")

def payunit_headers():
    token = base64.b64encode(
        f"{os.environ['PAYUNIT_API_USER']}:{os.environ['PAYUNIT_API_PASSWORD']}".encode()
    ).decode()
    return {
        "Authorization": f"Basic {token}",
        "x-api-key": os.environ["PAYUNIT_API_KEY"],
        "mode": os.environ.get("PAYUNIT_MODE", "test"),
        "Content-Type": "application/json",
    }

@router.post("/initialize")
async def initialize_payment(body: dict):
    async with httpx.AsyncClient(base_url="https://gateway.payunit.net") as client:
        r = await client.post("/api/gateway/initialize", json={
            "total_amount": body["amount"],
            "currency": "XAF",
            "transaction_id": body["transaction_id"],
            "return_url": os.environ["PAYUNIT_RETURN_URL"],
            "notify_url": os.environ["PAYUNIT_NOTIFY_URL"],
            "payment_country": body.get("country", "CM"),
        }, headers=payunit_headers())
    return r.json()

@router.post("/webhooks/payments/payunit")
async def payunit_webhook(request: Request):
    payload = await request.json()
    # Fulfill from payload["data"]["transaction_status"]
    return {"ok": True}
```

---

## Example 5 — React (calls your backend)

```tsx
async function startPayment(orderId: string, amount: number) {
  const res = await fetch('/api/payments/charges', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      orderId,
      amountMinor: amount,
      currency: 'XAF',
      country: 'CM',
      method: 'hosted',
      idempotencyKey: `${orderId}attempt1`,
    }),
  });
  const charge = await res.json();
  const url = charge.nextAction?.url;
  if (charge.nextAction?.type === 'redirect' && url) {
    window.location.href = url;
  }
}
```

Adapt `fetch` → `axios`, SWR mutation, TanStack Query, etc. per project.

---

## Example 6 — Flutter SDK (sandbox prototype only)

```dart
import 'package:flutter_payunit/flutter_payunit.dart';

PayUnitButton(
  apiUsername: const String.fromEnvironment('PAYUNIT_API_USER'),
  apiPassword: const String.fromEnvironment('PAYUNIT_API_PASSWORD'),
  apiKey: const String.fromEnvironment('PAYUNIT_API_KEY'),
  mode: 'sandbox', // Flutter SDK: sandbox | live (REST uses test | live)
  paymentCountry: 'CM',
  notifyUrl: 'https://your-api.com/webhooks/payments/payunit',
  returnUrl: 'https://your-app.com/payments/return',
  totalAmount: '10000',
  currency: 'XAF',
  onPaymentInitiated: (transactionId, phone, gateway) {
    // Persist pending transaction locally
  },
  actionAfterProccess: (transactionId, status, phone, gateway) {
    // UI feedback only — confirm via backend webhook
  },
)
```

Production Flutter: replace with a call to your backend + `url_launcher` / WebView.

---

## Example 7 — Status polling (any HTTP client)

```typescript
async function getPaymentStatus(transactionId: string) {
  const auth = Buffer.from(`${API_USER}:${API_PASSWORD}`).toString('base64');
  const res = await fetch(
    `https://gateway.payunit.net/api/gateway/paymentstatus/${transactionId}`,
    {
      headers: {
        Authorization: `Basic ${auth}`,
        'x-api-key': API_KEY,
        mode: MODE, // test | live
      },
    },
  );
  const json = await res.json();
  return json.data.transaction_status; // PENDING | SUCCESS | FAILED | CANCELLED
}
```

Dart, Python, PHP variants: same headers and URL — use the project's HTTP library.

---

## Example 8 — `transaction_id` generation

```
Good:  order20260613001, pu1734567890, SDK4587295  (≤ 18 chars for Orange)
Bad:   order-#123, pu_abc-def (hyphens/special chars break Orange Money)
Bad:   a1b2c3d4e5f6789012345678abcdef01  (32 hex chars — Orange rejects > 18)
```

Language-agnostic rule: **alphanumeric only** and **max 18 characters** if Orange Money is a target PSP (Payunit support; MTN tolerates longer).

---

## Example 9 — Checkout API (product link)

```json
POST /api/gateway/checkout/initialize
{
  "cancel_url": "https://your-app.com/pay/cancel",
  "success_url": "https://your-app.com/pay/success",
  "notify_url": "https://your-api.com/webhooks/payments/payunit",
  "currency": "XAF",
  "mode": "payment",
  "transaction_id": "checkout20260613001",
  "total_amount": 5000,
  "payment_country": "CM",
  "items": [{
    "price_description": { "unit_amount": 5000 },
    "product_description": {
      "name": "Product name",
      "image_url": "https://your-cdn.com/product.png"
    },
    "quantity": 1
  }],
  "customer": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "6XXXXXXX"
  },
  "meta": {
    "phone_number_collection": true,
    "address_collection": false
  }
}
```

Redirect user to `data.redirect` within **15 minutes**.

Mobile money completion: `POST /api/gateway/invoice/process`  
Card completion: `POST /api/gateway/checkout/process`

---

## Mapping to common architectures

Payunit is always a **`PayunitAdapter`**. Checkout talks to `PaymentGateway`.
See `~/.cursor/skills/payment-gateway/SKILL.md`.

| Architecture | Where Payunit lives | What stays PSP-agnostic |
|--------------|---------------------|-------------------------|
| **Clean / hexagonal** | `infra/payments/payunit/` | `domain/PaymentGateway` + `applyPaymentEvent` |
| **MVC / Rails / Laravel** | `PayunitAdapter` + webhook route | `PaymentsController` using the port |
| **Feature-first (Flutter/RN)** | `payment/data/payunit/` | `payment/domain/` + UI |
| **Microservice** | Adapter inside `payments-service` | Orders consume canonical events |
| **Serverless** | `POST /webhooks/payments/payunit` | `POST /payments/charges` |
| **Monolith SPA** | Server adapter only | Frontend payment page |

App contract: `POST /payments/charges`, `GET /payments/{id}` — never `/payments/payunit/*`.

Always place files where similar features already live in the repo.

---

## Sandbox testing

1. Sandbox application + `mode: test` + sandbox `x-api-key`.
2. Subscribe test PSPs in dashboard.
3. Cards unavailable in sandbox — use Mobile Money.
4. Swagger from dashboard: [getting started](https://developer.payunit.net/getting-started).

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Payunit secrets in client app | Backend proxy |
| HTTP callback URLs | HTTPS only |
| Trust UI callback only | Webhook + status API |
| Fulfill from `body.status` not `body.data.transaction_status` | Read nested `data` |
| Duplicate `transaction_id` | New ID per attempt |
| Special chars in `transaction_id` | Alphanumeric only |
| `transaction_id` longer than 18 chars (Orange) | Truncate / generate ≤ 18 alphanumeric |
| Amount mismatch | Same value in initialize and makepayment |
| Expired checkout link | Re-initialize after 15 min |
| Wrong gateway shortcode | Check dashboard PSP list (`CM_MTNMOMO`, `CM_ORANGE`, …) |
| Using `gateway.paynit.net` | Use `https://gateway.payunit.net` |
| Ignoring project conventions | Run project scan subagent first (Step 0) |
| Checkout imports Payunit types | Use `PaymentGateway` / `Charge.nextAction` only |
| App API `/payments/payunit/*` | `POST /payments/charges` + webhook `/webhooks/payments/payunit` |
