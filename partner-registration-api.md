# Partner Registration API

> This is the **current production-ready** partner endpoint used for automated user provisioning and source creation with **passwordless sign-in flow**.

## Endpoint
`POST /api/v1/partner/register`

## Authentication
Use a Bearer token provided by LedgerTax:

```
Authorization: Bearer <partner_token>
```

Each partner receives a unique token. Tokens are rotated on request.

## Key Features
- ✅ **Passwordless onboarding** - Users receive secure 30-day sign-in links
- ✅ **Duplicate source detection** - Prevents creating duplicate exchange connections
- ✅ **Detailed feedback** - Per-source status with skip reasons
- ✅ **Password setup flow** - Users set their own password during onboarding
- ✅ **Discount management** - Automatic discount code application

## Request body
```json
{
  "email": "user@example.com",
  "name": "John",
  "surname": "Doe",
  "partner_source": "XAGO",
  "discount_code": "XagoTax2026",
  "jurisdiction": "ZA",
  "api_credentials": [
    {
      "exchange_name": "Luno",
      "api_key": "abc123xyz789",
      "api_secret": "secretkey456",
      "api_passphrase": null
    }
  ]
}
```

## Field validation
| Field | Required | Type | Constraints | Default |
|------|----------|------|-------------|---------|
| email | Yes | string | Valid email, max 255 chars | - |
| name | No | string | Max 100 chars | Extracted from email |
| surname | No | string | Max 100 chars | null |
| partner_source | Yes | string | Partner code (e.g., XAGO) | - |
| discount_code | No | string | Max 50 chars, alphanumeric | null |
| jurisdiction | No | string | ZA / US | ZA |
| api_credentials | Yes | array | 1–5 objects | - |
| └─ exchange_name | Yes | string | Supported exchange | - |
| └─ api_key | Yes | string | Min 10 chars | - |
| └─ api_secret | Yes | string | Min 10 chars | - |
| └─ api_passphrase | No | string | Max 100 chars | null |

## Success response (new user)
```json
{
  "status": "success",
  "user_created": true,
  "clerk_id": "user_2abc123def",
  "email": "user@example.com",
  "sources_added": 1,
  "sources_skipped": 0,
  "source_details": [
    {
      "exchange_name": "LUNO",
      "status": "added"
    }
  ],
  "discount_applied": {
    "code": "XagoTax2026",
    "amount": 500,
    "currency": "ZAR",
    "valid_until": "2026-02-15T23:59:59Z"
  },
  "discount_message": null,
  "discount_error_code": null,
  "redirect_url": "https://ledgertax.com/sign-in#/?__clerk_ticket=eyJhbG...",
  "link_expires_at": "2026-02-20T23:59:59Z",
  "message": "Account created successfully. Redirect to the provided link to access your account.",
  "onboarding_instructions": "Click the redirect_url to sign in. You will be prompted to set a password during your first login for future access."
}
```

### Key Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `user_created` | boolean | `true` for new users, `false` for existing |
| `redirect_url` | string | **Secure sign-in link** valid for 30 days (no password needed) |
| `link_expires_at` | string | ISO 8601 timestamp when sign-in link expires |
| `source_details` | array | Per-source status with reasons for skipped sources |
| `onboarding_instructions` | string | Guidance for password setup (new users only) |
| `discount_applied` | object\|null | Applied discount details or null |

## Success response (existing user)
```json
{
  "status": "success",
  "user_created": false,
  "clerk_id": "user_2abc123def",
  "email": "user@example.com",
  "sources_added": 1,
  "sources_skipped": 1,
  "source_details": [
    {
      "exchange_name": "LUNO",
      "status": "added"
    },
    {
      "exchange_name": "VALR",
      "status": "skipped_duplicate",
      "reason": "Source with these API credentials already exists for VALR"
    }
  ],
  "discount_applied": null,
  "discount_message": "User already has discount applied",
  "discount_error_code": null,
  "redirect_url": "https://ledgertax.com/sign-in#/?__clerk_ticket=eyJhbG...",
  "link_expires_at": "2026-02-20T23:59:59Z",
  "message": "Existing account found. 1 source(s) added, 1 duplicate source(s) skipped."
}
```

## Warnings (invalid/expired discount)
The account is still created, but discount fields return an error code:
```json
{
  "discount_applied": null,
  "discount_message": "Discount code 'XagoTax2026' is invalid or expired",
  "discount_error_code": "INVALID_DISCOUNT_CODE"
}
```

## Passwordless Sign-In Flow

### How It Works
1. **Partner calls API** → LedgerTax creates account with temporary password
2. **API returns sign-in link** → Secure, one-time token in `redirect_url` (valid for 30 days)
3. **Partner redirects user** → User clicks link and is automatically signed in
4. **User sets password** → Frontend prompts password setup during onboarding
5. **Future access** → User can sign in with email + password

### Benefits
- ✅ No password management for partners
- ✅ Frictionless onboarding with one-click sign-in
- ✅ 30-day validity gives users plenty of time
- ✅ Secure cryptographically signed tokens via Clerk
- ✅ Multiple access methods after password setup

### Password Setup (Frontend Implementation Required)
Users set their own password during first-time onboarding:
1. Frontend checks `user.publicMetadata.requires_password_setup` flag
2. Shows password creation form
3. Calls `user.updatePassword({ newPassword })` (Clerk SDK)
4. Clears the metadata flag
5. User can now use email + password for future sign-ins

> **📋 See [PASSWORD_SETUP_FLOW.md](../ledgertax-api/PASSWORD_SETUP_FLOW.md) in the API repository for complete implementation guide and code examples.**

## Source Management

### Duplicate Detection
- Sources are deduplicated using `(user_id, exchange_name, api_key_hash)`
- Duplicate sources return `status: "skipped_duplicate"` with reason
- New sources return `status: "added"`
- All sources sync asynchronously after creation

### Source Status Responses
```json
"source_details": [
  {
    "exchange_name": "VALR",
    "status": "added"
  },
  {
    "exchange_name": "LUNO",
    "status": "skipped_duplicate",
    "reason": "Source with these API credentials already exists for LUNO"
  }
]
```

## Additional Notes
- Onboarding data (name/surname/jurisdiction) is prefilled for first login
- If partner code and discount partner mismatch, LedgerTax accepts discount but logs warning
- Sign-in links can be regenerated by making another registration call with same email
- After 30 days, partners can request new sign-in links for users

## Supported Exchanges

Currently supported exchange integrations:
- **LUNO** - South African exchange
- **VALR** - South African exchange  
- **KRAKEN** - International exchange
- **BINANCE** - International exchange
- **CRYPTOCOM** - International exchange
- **OKX** - International exchange
- **DERIBIT** - Derivatives exchange

> Exchange names are case-insensitive and normalized to uppercase (e.g., "Luno" → "LUNO")

## XAGO Partner Setup

### Required Configuration
- LedgerTax provides a dedicated bearer token for XAGO
- Store securely as environment variable: `PARTNER_TOKEN_XAGO=<provided_by_ledgertax>`
- Use in `Authorization: Bearer <token>` header

### Discount Setup
- Coupon code: `XagoTax2026`
- Amount: R500 ZAR
- Validity: Through 2026-12-31
- Auto-applied for new users
- Skipped for users with existing discounts

### Integration Flow
1. **User clicks "Tax Reporting" in XAGO platform**
2. **XAGO backend calls** `/api/v1/partner/register` with user info + credentials
3. **API returns** `redirect_url` (30-day sign-in link)
4. **XAGO redirects user** to the `redirect_url`
5. **User automatically signed in** to LedgerTax (no password needed)
6. **User completes onboarding** and sets their own password
7. **Future access** via email + password or new sign-in links

### Sample Request (XAGO)
```json
{
  "email": "user@example.com",
  "name": "Jane",
  "surname": "Doe",
  "partner_source": "XAGO",
  "discount_code": "XagoTax2026",
  "jurisdiction": "ZA",
  "api_credentials": [
    {
      "exchange_name": "Luno",
      "api_key": "luno_key_123",
      "api_secret": "luno_secret_456",
      "api_passphrase": null
    }
  ]
}
```

### Sample Response (New XAGO User)
```json
{
  "status": "success",
  "user_created": true,
  "clerk_id": "user_2XagoAbc123",
  "email": "user@example.com",
  "sources_added": 1,
  "sources_skipped": 0,
  "source_details": [
    {
      "exchange_name": "LUNO",
      "status": "added"
    }
  ],
  "discount_applied": {
    "code": "XagoTax2026",
    "amount": 500,
    "currency": "ZAR",
    "valid_until": "2026-12-31T23:59:59Z"
  },
  "redirect_url": "https://ledgertax.com/sign-in#/?__clerk_ticket=eyJhbG...",
  "link_expires_at": "2026-02-20T12:00:00Z",
  "message": "Account created successfully. Redirect to the provided link to access your account.",
  "onboarding_instructions": "Click the redirect_url to sign in. You will be prompted to set a password during your first login for future access."
}
```

### Implementation Checklist
- [ ] Store `PARTNER_TOKEN_XAGO` securely
- [ ] Call API with user email + exchange credentials
- [ ] Redirect user to `response.redirect_url` immediately
- [ ] Handle error responses gracefully
- [ ] Monitor sign-in link expiration (30 days)
- [ ] Test with sandbox environment first

### Support & Resources
- **Integration Support:** `support@ledgertax.io`
- **Technical Documentation:** [API Testing Guide](../ledgertax-api/docs/API_TESTING_GUIDE.md)
- **Password Setup Guide:** [PASSWORD_SETUP_FLOW.md](../ledgertax-api/PASSWORD_SETUP_FLOW.md)
- **Test Scripts:** Available in `ledgertax-api/` directory