# LedgerTax — Distribution Partner Integration Guide  
**Rev 0 (Draft)**  
**Audience:** Distribution partners (exchanges, brokers, trading platforms, bots, wallets) who want to offer **LedgerTax** to their users via an embedded “Tax Reporting” experience.

> This document is intentionally a mix of narrative + technical guidance. It should be readable by product people and directly usable by engineers.

---

## Table of Contents
1. [What is LedgerTax?](#what-is-ledgertax)
2. [What does “distribution integration” mean?](#what-does-distribution-integration-mean)
3. [Integration models](#integration-models)
4. [Recommended UX patterns](#recommended-ux-patterns)
5. [Security model](#security-model)
6. [Partner API (server-to-server)](#partner-api-server-to-server)
7. [Redirect + session handoff](#redirect--session-handoff)
8. [Data sources and coverage](#data-sources-and-coverage)
9. [Pricing, coupons, revenue share](#pricing-coupons-revenue-share)
10. [Testing & environments](#testing--environments)
11. [Support & troubleshooting](#support--troubleshooting)
12. [Partner checklist](#partner-checklist)
13. [Appendix A — Example integration pages (mockups)](#appendix-a--example-integration-pages-mockups)
14. [Appendix B — Example payloads](#appendix-b--example-payloads)
15. [Appendix C — Webhooks (optional)](#appendix-c--webhooks-optional)

---

## What is LedgerTax?

LedgerTax is a SaaS product that helps users generate **tax reports** from crypto activity across exchanges and wallets. LedgerTax:
- Imports transaction history (via exchange API keys, CSV exports, and/or wallet addresses)
- Categorizes events (trades, transfers, deposits/withdrawals, fees, rewards, etc.)
- Calculates taxable events (cost basis methods vary per jurisdiction)
- Produces a **PDF tax report** suitable for submission and/or sharing with an accountant

Distribution partners integrate LedgerTax to provide a simple, high-trust route for their users to generate compliant tax reports based on their activity on the partner platform.

---

## What does “distribution integration” mean?

A distribution integration typically includes:
1. A **“Tax Reporting”** entry point in your UI (menu item / button / banner / “Tax season” flow)
2. A **handoff** into LedgerTax (redirect + optional server-to-server provisioning)
3. Optional: automated preloading of the user’s transaction history into LedgerTax
4. Optional: a **partner coupon** (discount code) + partner attribution (for revenue share)

LedgerTax is designed so the integration can be:
- **lightweight** (just a link + coupon), or
- **high-conversion** (one-click provisioning + preloaded transactions + seamless redirect)

---

## Integration models

### Model 0 — Link-only (fastest, lowest effort)
You add a “Tax Reporting” link to LedgerTax and optionally provide a coupon code.

**Pros:** Minimal engineering  
**Cons:** Lower conversion, user must import history manually  
**Best for:** Partners without strong backend resources or without export APIs

---

### Model 1 — Link + coupon + UTM/referral attribution
You link to LedgerTax with:
- a partner code (for attribution)
- a coupon code (for discount)

**Pros:** Still simple; better tracking  
**Cons:** Still requires manual import unless user already has exports  
**Best for:** Partners who want tracking without handling any user data

---

### Model 2 — One-click provisioning (recommended for advanced partners)
You do a **server-to-server call** to LedgerTax to:
- create or match a user
- create a **Source** for your platform
- optionally attach exchange API keys / export pointers (depending on your system)
- return a short-lived **handoff token**
Then redirect the user to LedgerTax where the data is already loading.

**Pros:** Best UX, highest conversion  
**Cons:** Requires backend work + secure handling  
**Best for:** Exchanges, brokers, bots, custodians

---

### Model 3 — White-label / embedded (future / advanced)
LedgerTax can be embedded (e.g., iframe or partner-hosted UI calling LedgerTax APIs).
This is not currently the default approach, but can be supported for larger partners.

---

## Recommended UX patterns

### A) “Tax Reporting” page inside your platform (recommended)
A simple page that explains:
- what LedgerTax does
- what the user gets (SARS-ready / compliant report, etc.)
- discount code (if applicable)
- a clear CTA button: **“Go to LedgerTax”**

**Include:**
- 2–4 bullet value props
- one hero illustration / mock UI
- discount section (if enabled)
- FAQ section (optional)

> Image placeholders and examples are in [Appendix A](#appendix-a--example-integration-pages-mockups).

---

### B) One-click handoff UX (best conversion)
When the user clicks **Go to LedgerTax**, your backend:
1. Calls LedgerTax Partner API to provision
2. Receives a handoff token
3. Redirects user to LedgerTax
4. LedgerTax shows a loading/progress state while syncing

---

### C) Manual import UX (fallback)
If you can’t provision automatically:
- send users to LedgerTax with partner attribution + coupon
- show clear “How to export from <Partner>” steps (optional)
- optionally offer a downloadable export file directly from your platform

---

## Security model

LedgerTax integrations commonly involve sensitive data:
- personal identifiers (email/name)
- transaction history
- potentially exchange API keys (if you support API-based preloading)

This guide assumes:
- **All partner-to-LedgerTax calls are server-to-server**
- Calls are authenticated with **HMAC signatures** (recommended) or mutual TLS (enterprise)

### Security goals
- Prevent a third party from forging provisioning requests
- Prevent replay attacks
- Minimize exposure of secrets (API keys, tokens)
- Ensure auditability via correlation IDs

### Required partner controls
1. Store partner secret keys securely (vault / KMS)
2. Sign all requests (HMAC SHA-256)
3. Use timestamps + nonces to prevent replays
4. Never send secrets in URLs (query strings)
5. Never log secrets (mask keys and tokens in logs)
6. Encrypt any user API keys you store at rest (if applicable)

---

## Partner API (server-to-server)

> **Base URLs** (placeholders — confirm with LedgerTax team)  
- Sandbox: `https://sandbox.api.ledgertax.io`  
- Production: `https://api.ledgertax.io`

All requests:
- `Content-Type: application/json`
- `X-Partner-Id: <your_partner_id>`
- `X-Signature: <hmac_signature>`
- `X-Timestamp: <unix_seconds>`
- `X-Nonce: <random_uuid>`
- Optional: `X-Correlation-Id: <uuid>` (recommended)

### HMAC signature format
We recommend signing the canonical string:

```
<timestamp>.<nonce>.<http_method>.<path>.<sha256(body)>
```

Then:

- `signature = HMAC_SHA256(partner_secret, canonical_string)`
- send as hex or base64 (choose one and standardize)

#### Example (Node.js)
```js
import crypto from "crypto";

function sha256Hex(input) {
  return crypto.createHash("sha256").update(input).digest("hex");
}

export function signRequest({ secret, timestamp, nonce, method, path, bodyJson }) {
  const body = bodyJson ? JSON.stringify(bodyJson) : "";
  const canonical = `${timestamp}.${nonce}.${method.toUpperCase()}.${path}.${sha256Hex(body)}`;
  return crypto.createHmac("sha256", secret).update(canonical).digest("hex");
}
```

---

### Partner Registration API (LedgerTax)

> **📋 [View detailed Partner Registration API documentation](./partner-registration-api.md)**

This is the current **production-ready** partner endpoint with **passwordless sign-in flow** for automated user provisioning and source creation.

#### Key Features
- ✅ **Passwordless onboarding** - 30-day secure sign-in links (no password management for partners)
- ✅ **Duplicate detection** - Prevents creating duplicate exchange connections  
- ✅ **Detailed feedback** - Per-source status with skip reasons
- ✅ **Self-service password setup** - Users set their own password during onboarding
- ✅ **Automatic discounts** - Partner discount codes applied at registration

#### Quick Integration
```bash
POST /api/v1/partner/register
Authorization: Bearer <partner_token>

{
  "email": "user@example.com",
  "name": "First",
  "surname": "Last",
  "partner_source": "XAGO",
  "discount_code": "XagoTax2026",
  "jurisdiction": "ZA",
  "api_credentials": [
    {
      "exchange_name": "LUNO",
      "api_key": "...",
      "api_secret": "..."
    }
  ]
}
```

**Response includes:**
- `redirect_url` - Secure sign-in link (valid 30 days)
- `link_expires_at` - Link expiration timestamp
- `source_details` - Per-source status (added/skipped)
- `onboarding_instructions` - Password setup guidance

#### Integration Flow
1. Partner calls API with user credentials
2. API returns secure sign-in link in `redirect_url`
3. Partner redirects user to sign-in link
4. User automatically signed in (no password needed)
5. User sets password during onboarding
6. Future access via email + password

#### Documentation
The dedicated documentation covers:
- Complete endpoint specification
- Authentication requirements
- Request/response schemas with all new fields
- Field validation rules
- Passwordless sign-in flow
- Password setup implementation guide
- Duplicate source detection
- Error handling
- XAGO-specific setup instructions
- Supported exchanges list

---

### POST `/v1/partners/provision` (Legacy)

> **⚠️ Note:** This is a legacy endpoint. New integrations should use **`POST /api/v1/partner/register`** which includes passwordless sign-in flow and improved duplicate detection. See [Partner Registration API documentation](./partner-registration-api.md) for details.

Creates or matches a LedgerTax user and attaches one or more **Sources**.

#### Typical use cases

* Exchange user clicks “Tax Reporting” inside partner UI
* Bot platform wants to preload user history from multiple exchanges (if it already has keys)
* Partner wants to ensure attribution + coupon is set at creation time

#### Request body (schema)

```json
{
  "partner_id": "xago",
  "partner_user_id": "123456", 
  "user": {
    "email": "user@example.com",
    "first_name": "First",
    "last_name": "Last",
    "jurisdiction": "ZA"
  },
  "attribution": {
    "source": "partner",
    "campaign": "tax-season-2026",
    "coupon_code": "XagoTax2026"
  },
  "sources": [
    {
      "type": "partner_account",
      "name": "Xago",
      "external_account_id": "123456",
      "capabilities": ["csv", "api"],
      "notes": "Provisioned from Xago tax button"
    }
  ],
  "handoff": {
    "return_url": "https://partner.example.com/app/tax",
    "preferred_landing": "overview"
  }
}
```

#### Notes on key fields

* `partner_user_id` is your internal ID for the user (helps reconciliation)
* `jurisdiction` is a short code (e.g., `ZA`, `US`) — if unknown, omit or set `ZA` for SA-only partners
* `coupon_code` is optional
* `sources` can contain **one** primary partner source, plus optional additional sources if you manage them (e.g., bot platform controlling multiple exchange connections)

#### Response body

```json
{
  "user_id": "ltx_usr_abc123",
  "handoff_token": "ltx_handoff_...",
  "handoff_url": "https://app.ledgertax.io/handoff/ltx_handoff_...",
  "status": "ok"
}
```

---

### POST `/v1/partners/provision-with-keys` (optional / advanced)

> Only use this if you are already storing user exchange API keys (or you are a bot platform that manages keys).

```json
{
  "partner_id": "idatco",
  "partner_user_id": "u_789",
  "user": {
    "email": "user@example.com",
    "jurisdiction": "ZA"
  },
  "sources": [
    {
      "type": "exchange_api",
      "exchange": "luno",
      "label": "Luno via IDATCO",
      "credentials": {
        "api_key": "*****",
        "api_secret": "*****"
      }
    },
    {
      "type": "exchange_api",
      "exchange": "kraken",
      "label": "Kraken via IDATCO",
      "credentials": {
        "api_key": "*****",
        "api_secret": "*****"
      }
    }
  ],
  "attribution": {
    "coupon_code": "IDATCOTax2026"
  }
}
```

**Security requirement:** credentials must be encrypted at rest in LedgerTax, never logged, and must be least-privilege (read-only where possible).

---

### Error responses

LedgerTax uses standard HTTP codes plus machine-readable error bodies:

```json
{
  "status": "error",
  "code": "INVALID_SIGNATURE",
  "message": "Signature validation failed",
  "correlation_id": "..."
}
```

Common codes:

* `INVALID_SIGNATURE`
* `REPLAY_DETECTED`
* `VALIDATION_ERROR`
* `USER_BLOCKED`
* `RATE_LIMITED`
* `INTERNAL_ERROR`

---

## Redirect + session handoff

### Modern Approach: Passwordless Sign-In (Recommended)

When using **`POST /api/v1/partner/register`**, redirect users to the `redirect_url` in the response:

```json
{
  "redirect_url": "https://ledgertax.com/sign-in#/?__clerk_ticket=eyJhbG...",
  "link_expires_at": "2026-02-20T12:00:00Z"
}
```

**Key Features:**
- ✅ 30-day validity (vs 60-180 seconds for legacy tokens)
- ✅ No password required for initial access
- ✅ Users set their own password during onboarding
- ✅ Cryptographically signed secure tokens

> See [Partner Registration API](./partner-registration-api.md) for details.

---

### Legacy Approach: Short-Lived Handoff Tokens

For legacy endpoints, redirect the user to:

* `handoff_url` returned by `/provision`, or
* `https://app.ledgertax.io/handoff/<token>` (depending on configuration)

### Legacy Handoff Token Behavior

* Short-lived (60–180 seconds)
* Single-use recommended
* Creates/continues an authenticated LedgerTax session
* Sets partner attribution (if not already set)
* Sends user to the correct landing screen (`overview`, `sources`, etc.)

> **⚠️ Note:** New integrations should use the passwordless flow via `/api/v1/partner/register`.

### Return URL

If you supply a `return_url`, LedgerTax can show a “Back to <Partner>” link in the UI.

---

## Data sources and coverage

### Source concept

In LedgerTax, a **Source** is one origin of transaction data. Examples:

* “Xago account export”
* “Luno API connection”
* “Wallet: 0xabc…”

Partners typically create **one Source** representing the partner platform activity.

### Data ingestion approaches (partner-side)

Depending on your capabilities, choose one:

1. **Server-to-server API provisioning** (best)
2. **CSV export** (good)
3. **Wallet-only** (limited, but useful for self-custody products)

### CSV export (recommended format guidance)

If you provide exports, aim for:

* ISO timestamps in UTC
* explicit fees
* explicit base/quote assets for trades
* stable transaction identifiers

> If you already have an existing CSV schema, share it with the LedgerTax team — we can map it.

---

## Pricing, coupons, revenue share

### Coupons

Partners often display a discount code on the Tax Reporting page.

Example (as seen in mockups):

* `XagoTax2026`
* `IDATCOTax2026`

LedgerTax can configure coupons to:

* apply only for users attributed to your partner
* apply to first report only (common)
* expire after a set period

### Attribution

Attribution determines partner revenue share eligibility. Typical rules:

* Attribution is set at first provisioning (recommended “first touch wins”)
* Subsequent partner clicks should not override attribution unless explicitly allowed

### Revenue share (high level)

Revenue share models vary (referral / revenue split / licensing). This doc focuses on the technical primitives:

* `partner_id` attribution
* `coupon_code` usage tracking
* optional webhooks for “report purchased” events (see Appendix C)

---

## Testing & environments

### Environments

* **Sandbox**: safe for testing with dummy data
* **Production**: real users and real billing

### Recommended partner test plan

1. Provision user (new) → verify redirect works
2. Provision same user again → idempotency (no duplicates)
3. Invalid signature → rejected
4. Replay test (same nonce) → rejected
5. Coupon applied correctly
6. Source appears in LedgerTax
7. User can generate a report end-to-end

### Idempotency

Partners should provide a stable idempotency key:

* `Idempotency-Key: <uuid>`
  or rely on `(partner_id + partner_user_id + email)` as dedupe identity.

---

## Support & troubleshooting

### Include correlation IDs

Always log the `X-Correlation-Id` you send and store it with your internal user ID.
LedgerTax will echo this in responses and logs, making support much faster.

### Common integration issues

* **Signature mismatch**: canonical string differences (method/path/body hash)
* **Clock drift**: timestamp too far from LedgerTax time window
* **Nonce reuse**: retry logic accidentally reuses nonce
* **User email mismatch**: partner email differs from user’s LedgerTax email

### Partner support channel

> Replace with your preferred public support endpoint(s).

* Email: `support@ledgertax.io`
* Slack/Discord: *(optional)*
* GitHub Issues: *(optional, if you want partners to open issues publicly)*

---

## Partner checklist

### Product & UX

* [ ] Add “Tax Reporting” menu item or call-to-action
* [ ] Add Tax Reporting page (copy + bullets + CTA)
* [ ] Display coupon code (if provided)
* [ ] Include basic FAQ and privacy disclaimer
* [ ] Include “Back to <Partner>” link behavior (optional)

### Engineering

* [ ] Store partner secret in vault/KMS
* [ ] Implement HMAC signing + replay protection
* [ ] Call `/v1/partners/provision` on CTA click
* [ ] Redirect user using returned `handoff_url`
* [ ] Pass `partner_user_id` for reconciliation
* [ ] Log correlation IDs

### Compliance

* [ ] Confirm user consent to share transaction data with LedgerTax
* [ ] Update privacy policy terms if needed
* [ ] Ensure secure handling of any API credentials (if used)

---

## Appendix A — Example integration pages (mockups)

> Replace these images with your final assets.
> Suggested paths assume a repo structure like `assets/mockups/...`.

### Example: Xago integration page

![Xago Tax Reporting Mockup](./assets/mockups/xago-tax-reporting.png)

**Placeholders used in mock:**

* Title: “Simplify Your Crypto Tax Reporting”
* Bullets:

  * Automatically import your Xago trades and transactions
  * Calculate crypto gains/losses and taxable events
  * Generate SARS-ready tax reports
* Coupon block: `XagoTax2026`
* CTA button: “Go to LedgerTax”

### Example: IDATCO integration page

![IDATCO Tax Reporting Mockup](./assets/mockups/idatco-tax-reporting.png)

**Placeholders used in mock:**

* “Automatically import your IDATCO trade history”
* Coupon block: `IDATCOTax2026`

### Optional: Wireframe / UX flow diagram

> Add a simple flow diagram showing the recommended one-click provisioning.

![Partner Handoff Flow](./assets/diagrams/partner-handoff-flow.png)

---

## Appendix B — Example payloads

### Minimal provisioning (Model 2)

```json
{
  "partner_id": "xago",
  "partner_user_id": "123456",
  "user": {
    "email": "user@example.com",
    "first_name": "First",
    "last_name": "Last",
    "jurisdiction": "ZA"
  },
  "attribution": {
    "source": "partner",
    "campaign": "tax-season-2026",
    "coupon_code": "XagoTax2026"
  },
  "sources": [
    {
      "type": "partner_account",
      "name": "Xago",
      "external_account_id": "123456",
      "capabilities": ["csv"]
    }
  ],
  "handoff": {
    "preferred_landing": "overview",
    "return_url": "https://xago.example.com/app/tax"
  }
}
```

### Multi-source provisioning (bot platform pattern)

```json
{
  "partner_id": "idatco",
  "partner_user_id": "u_789",
  "user": {
    "email": "user@example.com",
    "jurisdiction": "ZA"
  },
  "attribution": {
    "coupon_code": "IDATCOTax2026"
  },
  "sources": [
    {
      "type": "exchange_api",
      "exchange": "luno",
      "label": "Luno via IDATCO",
      "credentials": {
        "api_key": "*****",
        "api_secret": "*****"
      }
    },
    {
      "type": "exchange_api",
      "exchange": "valr",
      "label": "VALR via IDATCO",
      "credentials": {
        "api_key": "*****",
        "api_secret": "*****"
      }
    }
  ]
}
```

---

## Appendix C — Webhooks (optional)

If partners want billing/report status updates, LedgerTax can send webhooks.

### Events (examples)

* `user.provisioned`
* `source.sync.started`
* `source.sync.completed`
* `report.generated`
* `report.purchased`

### Webhook security

* Sign webhook payloads using HMAC
* Include timestamp + nonce

### Webhook payload example

```json
{
  "event": "report.purchased",
  "timestamp": "2026-01-06T10:05:00Z",
  "partner_id": "xago",
  "partner_user_id": "123456",
  "ledgertax_user_id": "ltx_usr_abc123",
  "report_id": "ltx_rpt_987",
  "amount": {
    "currency": "ZAR",
    "value": 2000
  },
  "coupon_code": "XagoTax2026"
}
```

---

## Change log

* Rev 0: Initial draft for partner distribution integrations.

