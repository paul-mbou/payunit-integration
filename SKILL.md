---
name: payunit-integration
description: >-
  Integrates Payunit as a PaymentGateway adapter (Mobile Money, cards, hosted
  checkout) so the app can switch PSPs without rewriting checkout. Always scans
  architecture first. Use when the user mentions Payunit, flutter_payunit,
  @payunit/nodejs-sdk, gateway.payunit.net, mobile money, XAF payments, or
  payment webhooks.
---

# Payunit integration

Guide for implementing [Payunit](https://developer.payunit.net/) — payment aggregator for Mobile Money and online payments (primarily **XAF**, countries like **CM**).

**Official docs:** https://developer.payunit.net/  
**Last verified:** 2026-08-31 (REST initialize/make-payment/overview **2024-12-01**; Node SDK **2025-11-26**; Flutter SDK **2025-12-19**; checkout process **2025-03-03**)

This skill is **stack- and architecture-agnostic**. Always adapt file layout, state management, and HTTP client to the **target project** — never assume a specific framework unless the codebase uses it.

**Mandatory:** Payunit is an **adapter** behind the shared `PaymentGateway` port.
Checkout, orders, and UI must not import Payunit types. Read and apply
`~/.cursor/skills/payment-gateway/SKILL.md` before writing code.

- Port already in the repo → add `PayunitAdapter` only, register it, keep fulfillment.
- No port → create the port + this adapter (see payment-gateway skill).
- Switching later (Stripe, …) = new adapter + `PAYMENT_PROVIDER` — not a rewrite.

**Mandatory first action:** launch a **project-scan subagent** before any Payunit design or code (see Step 0).

---

## Step 0: Project scan subagent (MANDATORY)

When this skill is invoked, the parent agent **must not** guess architecture or start implementing until a subagent has scanned the repo.

### Launch the subagent

Use the **Task** tool:

| Parameter | Value |
|-----------|--------|
| `subagent_type` | `explore` |
| `description` | `Scan project architecture` |
| `readonly` | `true` |
| `run_in_background` | `false` |

### Subagent prompt (copy and adapt)

```
Full Repository Path: <absolute path to project root>

Task: Scan this codebase and define its structure/architecture for a future Payunit payment integration.

Do NOT implement Payunit. Read-only exploration only.

Investigate:
1. Stack (mobile/web/backend/fullstack/monorepo) and frameworks
2. Folder layout and architecture style (feature-first, layered, clean, MVC, etc.)
3. HTTP client, state management, routing, DI patterns
4. How env vars / secrets are loaded
5. Existing payment, order, or checkout code (if any)
6. Whether a **PaymentGateway port** already exists (interface + adapters)
7. One reference feature/module to mirror for new payment code (cite real paths)

If a PaymentGateway port exists, recommend **adapter-only** placement (do not rewrite checkout).

Return findings using the template in:
~/.cursor/skills/payunit-integration/project-scan-template.md

Be specific: real file paths, not generic advice.
If monorepo, focus on the app/package relevant to the user's request.
```

### After subagent returns

1. **Present the scan summary** to the user (stack, recommended paths, reference feature).
2. **Confirm integration path** (Step 1) based on scan — e.g. backend proxy if server exists in repo.
3. **Only then** proceed to implementation, placing files at the paths recommended in the scan.

### Skip subagent only if

- The user explicitly says architecture is already documented in the chat **and** provides paths; or
- A completed scan for the **same repo** exists earlier in the **same conversation** (reuse it, do not re-scan).

If the scan is ambiguous, ask one clarifying question before coding.

Output template: [project-scan-template.md](project-scan-template.md)

---

## Step 1: Choose integration path

| Path | Best for | Server required? |
|------|----------|------------------|
| **A — Platform SDK** | Fast mobile MVP (`flutter_payunit`, Native Android SDK) | `notify_url` should hit your backend |
| **B — REST + hosted page** | Redirect user to Payunit hosted UI | **Yes** |
| **C — Checkout API** | Product cart / invoice link (15 min validity) | **Yes** |
| **D — REST + custom UI** | List providers → make payment → poll status | **Yes** |
| **E — One-step MoMo** | Single API call (SDK `initiateAndMakePaymentMobileMoney`) | **Yes** |

**Production default:** **B or D via backend proxy** — Payunit credentials (`api_user`, `api_password`, `x-api-key`) belong on the **server**, not in client bundles or mobile binaries. Client SDKs that embed credentials are acceptable for **sandbox/prototype only**.

**Node backends:** prefer the official **`@payunit/nodejs-sdk`** wrapped in `PayunitAdapter` rather than raw HTTP when using Node 18+.

Payunit also documents SDKs for [Node.js](https://developer.payunit.net/sdk-and-plugins/nodejs), [Flutter](https://developer.payunit.net/sdk-and-plugins/flutter), [Native Android](https://developer.payunit.net/sdk-and-plugins/native-android), [Python](https://developer.payunit.net/sdk-and-plugins/python), [PHP](https://developer.payunit.net/sdk-and-plugins/php), and [WooCommerce](https://developer.payunit.net/sdk-and-plugins/woocommerce).

---

## Step 2: Prerequisites

```
Payunit setup:
- [ ] Merchant account created
- [ ] Application created (PAYMENT COLLECTION for in-app integration)
- [ ] Application activated (dashboard → ⋮ → Activate)
- [ ] At least one PSP subscribed (Orange Money, MTN, etc.)
- [ ] Sandbox keys obtained
- [ ] KYC submitted if more payment methods needed (go-live)
```

| Credential | Source |
|------------|--------|
| `api_user` / `api_password` | Dashboard → API CREDENTIALS |
| `application_token` (`x-api-key`) | Application → APPLICATION DETAIL (sandbox or live) |
| `mode` | REST / Node SDK: `test` or `live`; Flutter SDK widget: `sandbox` or `live` |
| `base_url` | `https://gateway.payunit.net` (ignore `gateway.paynit.net` typo in older docs) |

Store credentials in the project's existing secret mechanism — **never commit to git**.

---

## Step 3: Mandatory Payunit rules

- Callback URLs (`return_url`, `notify_url`, `success_url`, `cancel_url`, `image_url`) must be **HTTPS**.
- `transaction_id` must be **unique** per attempt; **alphanumeric only** (no special characters — breaks Orange Money); **max 18 characters** for Orange Money (Payunit support; MTN tolerates longer — keep ≤18 for all).
- Primary currency: **`XAF`**.
- Confirm payment via **`notify_url` webhook** and/or **`GET /paymentstatus/{transactionID}`** — never trust UI callback alone.
- Webhook fulfillment uses **`data.transaction_status`**, not top-level envelope `status` alone.
- Card payments **not available in sandbox**.
- Checkout links expire after **15 minutes**.
- Gateway shortcodes are PSP-specific — common: `CM_ORANGE`, `CM_MTNMOMO` (verify subscribed PSPs in dashboard).

---

## Step 4: Core payment flow

### Authentication (every REST call)

All calls need **both** Basic auth and application token:

```
Content-Type: application/json
Authorization: Basic Base64(api_user:api_password)
x-api-key: <application_token>
mode: test | live
```

### REST sequence (paths B/D)

```
1. POST /api/gateway/initialize
   Optional: pay_with, redirect_on_failed, custom_fields, payment_country
2. Redirect to transaction_url  OR  GET /api/gateway/gateways → POST /api/gateway/makepayment
3. GET /api/gateway/paymentstatus/{transactionID}
4. POST notify_url (webhook) → idempotent order update on your server
```

Full payloads: [reference.md](reference.md).

### One-step MoMo (path E — Node SDK)

```
client.collections.initiateAndMakePaymentMobileMoney({ total_amount, gateway, phone_number, ... })
```

Still requires `notify_url` + status polling for reliability.

### Checkout sequence (path C)

```
POST /api/gateway/checkout/initialize → data.redirect (15 min)
Mobile money: POST /api/gateway/invoice/process
Card:         POST /api/gateway/checkout/process
```

Checkout `mode`: `payment` or `subscription`.

---

## Step 5: Map layers to any architecture

Use this logical split; **rename folders to match the project**. Payunit lives
**only** in the adapter layer — see `~/.cursor/skills/payment-gateway/SKILL.md`.

| Layer | Responsibility |
|-------|----------------|
| **Port (domain)** | `PaymentGateway`, `Charge`, `PaymentEvent` — **no Payunit types** |
| **Config** | `PAYMENT_PROVIDER` + Payunit env (`baseUrl`, `mode`, callback URLs) |
| **PayunitAdapter** | Translate port ↔ Payunit HTTP/SDK; parse webhook; map statuses |
| **Use-case** | `createCharge` / `getStatus` / `applyPaymentEvent` (shared across PSPs) |
| **Webhook** | `POST /webhooks/payments/payunit` → adapter → `applyPaymentEvent` |
| **UI** | Reads `Charge.nextAction` only (redirect / await phone) |

### Client ↔ server boundary

```
┌─────────────┐   POST /payments/charges        ┌─────────────┐   PayunitAdapter      ┌──────────┐
│ Mobile/Web  │ ──────────────────────────────► │ Your API    │ ────────────────────► │ Payunit  │
│   (client)  │ ◄── Charge + nextAction         │  (port)     │ ◄── notify webhook ── │          │
└─────────────┘                                 └─────────────┘                        └──────────┘
```

Client calls **your** API only in production. Never expose `/payments/payunit/*`
as the app contract. Server holds Payunit secrets and forwards to `gateway.payunit.net`.

### Webhook handler requirements

1. Route: `POST /webhooks/payments/payunit` (register this as `notify_url`).
2. `PayunitAdapter.parseWebhook` validates payload shape; match `data.transaction_id` to a stored pending row (no documented HMAC).
3. Fulfill from **`data.transaction_status`** (`SUCCESS` → grant access).
4. `applyPaymentEvent` updates the order **idempotently** (same `transaction_id` twice = no double fulfillment).
5. Return HTTP 200 quickly; heavy work async if needed.
6. Whitelist route on firewall/load balancer before go-live.

Webhook payload shape: [reference.md](reference.md#webhook-payload-notify_url).  
Status map: `~/.cursor/skills/payment-gateway/adapters.md`.

### Payment state machine (canonical)

```
pending → awaiting_user → succeeded | failed | cancelled | expired
```

Map Payunit `PENDING|SUCCESS|FAILED|CANCELLED` in the adapter (see payment-gateway
`adapters.md`). Persist `provider` + `providerPaymentId` (`transaction_id`) so
status can be polled if the user closes the app or tab.

---

## Step 6: Platform notes

| Platform | Integration hint |
|----------|------------------|
| **Node / Express / Nest / Firebase Functions** | **`@payunit/nodejs-sdk`** or raw HTTP; webhook route; wrap in `PayunitAdapter`. |
| **Flutter** | `flutter_payunit` for prototype (`mode: sandbox\|live`); production → backend proxy. See [examples.md](examples.md). |
| **React / Vue / Angular** | `fetch`/`axios` to your backend; open `transaction_url` in same tab or WebView. |
| **Python / Django / FastAPI** | `requests` or Python SDK; webhook as POST view. |
| **PHP / Laravel** | PHP SDK or Guzzle; webhook route excluded from CSRF. |
| **Native Android** | Payunit Android SDK per docs. |

Always use the project's existing HTTP client, DI, and routing — do not introduce a parallel stack.

---

## Step 7: Error handling

| HTTP | Meaning | Action |
|------|---------|--------|
| 400 | Provider unsupported / transaction exists | New `transaction_id`; check PSP subscription |
| 404 | Transaction not found | Verify ID |
| 422 | Integrity violation | Review payload; amount/duplicate mismatch |

`transaction_status`: `PENDING` | `SUCCESS` | `FAILED` | `CANCELLED`.

---

## Go-live checklist

- [ ] `mode: live` + live `x-api-key`
- [ ] `notify_url` = `https://<api>/webhooks/payments/payunit` (HTTPS, tested)
- [ ] At least one PSP subscribed
- [ ] KYC completed if needed
- [ ] No Payunit secrets in client builds or git
- [ ] Alphanumeric `transaction_id` ≤ 18 chars for Orange Money
- [ ] Amount consistent between initialize and makepayment
- [ ] `PAYMENT_PROVIDER=payunit`; UI has no Payunit types
- [ ] Webhook reads `data.transaction_status`, not envelope `status` alone

---

## Agent checklist before finishing

- [ ] **Project scan subagent** completed (or valid scan reused from same conversation)
- [ ] Scan summary shown; Payunit files placed at recommended paths
- [ ] Integration path chosen and documented
- [ ] Webhook or status polling implemented (not UI-callback-only)
- [ ] Sandbox tested with subscribed PSP
- [ ] All callback URLs use HTTPS
- [ ] `transaction_id` strategy documented
- [ ] Credentials stored via project's existing secret pattern
- [ ] No unnecessary new dependencies or architectural layers
- [ ] **PaymentGateway port** used (or created); UI/domain have **no** Payunit types
- [ ] `PayunitAdapter` maps statuses; fulfillment only via `applyPaymentEvent`
- [ ] App API is `POST /payments/charges` + `GET /payments/{id}` (not `/payments/payunit/...`)
- [ ] Switching to another PSP would be adapter + `PAYMENT_PROVIDER` only

## Additional resources

- Shared port (mandatory): `~/.cursor/skills/payment-gateway/SKILL.md`
- Project scan output template: [project-scan-template.md](project-scan-template.md)
- API reference: [reference.md](reference.md)
- Multi-stack examples: [examples.md](examples.md)
- Docs home: https://developer.payunit.net/
- Getting started: https://developer.payunit.net/getting-started
- Node.js SDK: https://developer.payunit.net/sdk-and-plugins/nodejs
- Flutter SDK: https://developer.payunit.net/sdk-and-plugins/flutter
