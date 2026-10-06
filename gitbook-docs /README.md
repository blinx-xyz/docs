---
description: >-
  Welcome to the developer documentation for the Blinx Merchant Payments API —
  the tenant-facing surface for creating and reading payments.
---

# Blinx Merchant API Documentation

{% hint style="warning" %}
**NOTE**\
\
This documentation covers operations performed in **sandbox**\
No real transactions are created
{% endhint %}

### Scope

This documentation covers everything a merchant (tenant) integration needs:

* **API reference** — the full endpoint contract lives in the OpenAPI spec, `seller-payments.openapi.yaml`. Render it with any OpenAPI viewer (Redoc, Swagger UI, Stoplight) for interactive docs.
* **Request authentication** — every request to `/api/merchant/*` is authenticated per call with an **apiKey + apiSecret HMAC signature** (`x-api-key`, `x-timestamp`, `x-signature`). There is no session or bearer token.

### What's covered where

| Topic                                        | Where                          |
| -------------------------------------------- | ------------------------------ |
| Endpoints, schemas, error shapes             | `seller-payments.openapi.yaml` |
| Country reference data (limits, travel rule) | Get country data               |
| Creating & checking payments                 | Create and check a payment     |
| Signing requests in code (CLI)               | Sign a payment request         |
| Signing requests in Postman                  | Postman pre-request script     |

### The signing model in one line

```
signature = hex HMAC-SHA256( apiSecret, `${timestamp}.${METHOD}.${path}.${sha256hex(body)}` )
```

* `timestamp` — unix epoch **milliseconds**, within **±5 minutes** of server time.
* `METHOD` — upper-cased HTTP method.
* `path` — request target (path **+ query**) exactly as sent.
* `sha256hex(body)` — SHA-256 (hex) of the **raw** body bytes; the empty string hashes to `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`.

The signature covers the **exact** body bytes — send the body verbatim; never reformat or re-serialize it after signing, or the hash won't match.

### Ready-made helpers

Both helper scripts are self-contained (Node's `crypto` only — no service dependencies) and live under `scripts/`:

* `sign-payment-request.ts` — sign an API request from the CLI.
* `postman-pre-request.js` — sign requests inside Postman.

_Updated: 19 September 2026 12:22_
