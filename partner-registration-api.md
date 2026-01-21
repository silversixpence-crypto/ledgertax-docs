# Partner Registration API

> This is the **current production-ready** partner endpoint used for automated user provisioning and source creation.

## Endpoint
`POST /api/v1/partner/register`

## Authentication
Use a Bearer token provided by LedgerTax:

```
Authorization: Bearer <partner_token>
```

Each partner receives a unique token. Tokens are rotated on request.

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
  "discount_applied": {
    "code": "XagoTax2026",
    "amount": 500,
    "currency": "ZAR",
    "valid_until": "2026-02-15T23:59:59Z"
  },
  "redirect_url": "https://ledgertax.com/onboarding",
  "message": "Account created. Email sent with login credentials."
}
```

## Success response (existing user)
```json
{
  "status": "success",
  "user_created": false,
  "clerk_id": "user_2abc123def",
  "email": "user@example.com",
  "sources_added": 1,
  "sources_skipped": 1,
  "discount_applied": null,
  "discount_message": "User already has active subscription",
  "redirect_url": "https://ledgertax.com/dashboard",
  "message": "Sources added to existing account."
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

## Notes
- Sources are created immediately and start syncing asynchronously.
- Duplicate sources are deduplicated using `(user_id, exchange_name, api_key_hash)`.
- If the partner code and discount partner mismatch, LedgerTax will still accept the discount but log a warning.
- Onboarding data (name/surname/jurisdiction) is prefilled for first login.

## XAGO Partner Setup

### Required configuration
- LedgerTax provides a dedicated bearer token for XAGO.
- Store it as an environment variable on your server:
  - `PARTNER_TOKEN_XAGO=<provided_by_ledgertax>`

### Recommended discount setup
- Coupon code: `XagoTax2026`
- Validity: 30 days from user registration
- Currency: ZAR

### Sample request (XAGO)
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

### Support
For integration support, contact: `support@ledgertax.io`