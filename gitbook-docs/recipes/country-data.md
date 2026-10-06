---
description: 'updated: 27 August 2026'
---

# Country Data

A practical guide to reading **reference data for a supported country** — its currency and dialling info, the travel-rule threshold that governs transfers, and the amount limits per payout channel. For the exact request/response contract see the OpenAPI spec (`seller-payments.openapi.yaml`, tag _Merchant_); for signing see the main README and Sign a request.

### Concepts

* **Country code** — the ISO country code (`NG`, `KE`, `TZ`, …) identifying a supported country. It is the only path parameter and is matched against the set of countries Blinx pays out to; an unknown or unsupported code is rejected with `400 PAY_COUNTRY_NOT_SUPPORTED`.
* **Travel rule** — every transfer must carry a `travelRuleData` object; it is required **unconditionally**, regardless of amount. `transfer.minAmount` is the transaction floor (the smallest amount a transfer may be), **not** a threshold that switches travel-rule data on. `transfer.dataFields` names the fields that object must include.
* **Payout channels** — a country pays out over one or more channels (e.g. `bank`, `momo`), each with its own `{ min, max }` amount bounds. A country need not have every channel — a missing channel key means that channel is **absent**, not unbounded. `payments.min` / `payments.max` give the widest envelope across whatever channels exist.

### Endpoints

Under `/api/merchant/data`, requiring the standard apiKey + apiSecret HMAC signature.

| Method & path                      | Purpose                                  |
| ---------------------------------- | ---------------------------------------- |
| `GET /api/merchant/data/{country}` | Reference data for one supported country |

### Walkthrough

The examples below assume you sign each request. The easiest way is the CLI signer, which can also print a runnable `curl`:

```bash
ts-node scripts/sign-payment-request.ts \
  --api-key "$BLINX_API_KEY" --api-secret "$BLINX_API_SECRET" \
  --method GET --url /api/merchant/data/NG \
  --curl --base-url http://localhost:3001
```

> **Sign the exact path.** This is a GET with **no body**, so there is nothing to serialize — but the signature still covers the request target (path + query) **exactly as sent**. Don't add a trailing slash or change the case of the country code after signing, or the hash won't match and you get `401`. No `Content-Type` header is needed since there is no body.

#### Read a country's data

```bash
curl http://localhost:3001/api/merchant/data/NG \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
```

The response is `200` with the country object inside the standard `data` envelope — the global response wrapper wraps every success body, so this endpoint is no exception:

```json
{
  "data": {
    "code": "NG",
    "label": "Nigeria · NGN",
    "currency": "NGN",
    "dialCode": "+234",
    "phoneLength": 9,
    "transfer": {
      "minAmount": 1000,
      "dataFields": "[{\"displayName\":\"First Name\",\"fieldType\":\"string\",\"required\":true,\"payloadName\":\"firstName\"},{\"displayName\":\"Last Name\",\"fieldType\":\"string\",\"required\":true,\"payloadName\":\"lastName\"}]"
    },
    "payments": {
      "bank": { "min": 2500, "max": 30000000 },
      "momo": { "min": 500, "max": 250000 },
      "min": 500,
      "max": 30000000
    }
  }
}
```

Field by field:

* `code` / `label` / `currency` / `dialCode` / `phoneLength` — country identity and the info needed to normalise a local phone number (`dialCode` prefix, `phoneLength` national digits).
* `transfer.minAmount` — the transaction floor: the smallest amount a transfer may be. It is **not** a travel-rule threshold — `travelRuleData` is required on every transfer regardless of amount.
* `transfer.dataFields` — a JSON-encoded array of field descriptors (`{displayName, fieldType, required, payloadName}`), passed through verbatim from the provider. Each descriptor's `payloadName` is a field your `travelRuleData` object must include on every transfer.
* `payments.bank` / `payments.momo` — `{ min, max }` amount bounds for that payout channel, in the country's `currency`. **Either may be absent** when the country has no such channel — treat a missing key as "channel not available", never as an unlimited range.
* `payments.min` / `payments.max` — the widest amount envelope across the country's available channels.

### Errors

| Status / code                                                 | When                                                                        |
| ------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `401 AUTH_401_INVALID_SIGNATURE` / `AUTH_401_INVALID_API_KEY` | Bad/missing signature, key, or stale timestamp (see the signing note above) |
| `400 PAY_COUNTRY_NOT_SUPPORTED`                               | The `{country}` code is unknown or not a supported payout country           |

### Notes

* **Read-only reference data.** This endpoint takes no body and never mutates anything; it is safe to call and cache.
* **Standard `data` envelope.** Like the rest of the API, the country object is wrapped — read `data.code`, `data.payments`, and so on.
* **Channel keys are optional.** Always check whether `payments.bank` / `payments.momo` are present before reading their bounds; only `payments.min` / `payments.max` are guaranteed.

_Updated: 19 September 2026 12:22_
