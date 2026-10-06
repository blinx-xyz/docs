# Create a Payment

## Create and check a payment

A practical guide to creating a **payment**, handing it off to the payer, and learning its outcome. For the exact request/response contract see the OpenAPI spec (`seller-payments.openapi.yaml`, tag _Merchant Payments_); for signing see the main README and Sign a payment request.

### Concepts

* **Payment** — a request for a payer to pay you. You create it in the local **`fiat`** currency the payer pays; you (the merchant) are settled in the **`token`** you choose. `amount` is the decimal amount in that `fiat` currency.
* **`redirectUrl`** — the hosted URL returned on create. The payer completes the payment there; your integration's only job is to get the payer to it. Its last path segment is the payment **`code`** (see below), not the payment `id`.
* **`code`** — a short, human-readable payer code (format `word-word-####C`, e.g. `foam-bark-95162`) returned alongside `id` on create. It is what appears in `redirectUrl`; the internal payment `id` never leaks into the payer link.
* **Corridor** — the `(country, fiat)` pair. It decides which amount limits apply and whether the payment can be created at all. Currencies shared across countries (XOF, XAF) are bounded **per country**, so pass `country` to disambiguate; it falls back to `fiat` when omitted.
* **`status`** — where the payment is in its lifecycle. One of `created`, `waiting`, `processing`, `validated`, `refunding`, `refunded`, `expired`, `rejected`, `failed`, `cancelled`, `settled`. **Terminal**: `settled` (success) and `failed` / `rejected` / `expired` / `cancelled` (not paid). The rest are in-flight.
* **`reference`** — your own unique reference for the payment. It travels back on reads so you can reconcile against your records.
* **`purpose`** — a human-readable description shown to the payer; it maps to the transfer memo and comes back on the `Payment`.
* **`type`** — the payment category, one of `gift`, `bills`, `groceries`, `travel`, `health`, `entertainment`, `housing`, `school-fees`, `other`. Defaults to `other` when omitted.

### Lifecycle

```
POST /api/merchant/payments          → { data: { redirectUrl, validUntil, code, id } }   (status: created)
      │
      ▼  give the payer the redirectUrl
payer completes the hosted payment
      │
      ▼  learn the outcome
GET /api/merchant/payments/{id}       → { data: Payment }  (poll until terminal)
      │
      ▼
status ∈ { settled | failed | rejected | expired | cancelled }   (terminal)
```

Create the payment, give the payer its `redirectUrl`, then wait for a terminal `status` by polling `GET /api/merchant/payments/{id}` until it reaches a terminal state.

### Endpoints

All endpoints are under `/api/merchant/payments` and require the standard apiKey + apiSecret HMAC signature.

| Method & path                     | Purpose                                                              |
| --------------------------------- | -------------------------------------------------------------------- |
| `POST /api/merchant/payments`     | Create a payment; returns the hosted `redirectUrl`                   |
| `GET /api/merchant/payments/{id}` | Get one payment (full `Payment`), in any status                      |
| `GET /api/merchant/payments`      | List the merchant's payments — JSON array, or CSV with `?format=csv` |

### Walkthrough

The examples below assume you sign each request. The easiest way is the CLI signer, which can also print a runnable `curl`:

```bash
ts-node scripts/sign-payment-request.ts \
  --api-key "$BLINX_API_KEY" --api-secret "$BLINX_API_SECRET" \
  --method POST --url /api/merchant/payments \
  --body '{"amount":"5000","fiat":"NGN","token":"USDC","country":"NG","reference":"abc","purpose":"seize global content","type":"other"}' \
  --curl --base-url http://localhost:3001
```

> **Content-Type matters.** The body must be sent as `application/json`. If your HTTP client sends a stringified JSON body without `Content-Type: application/json` (browser `fetch` defaults to `text/plain`), the server never parses the body, the signed body hash won't match, and you get `401`. Always set the header.
>
> **Sign the exact path and bytes.** The signature covers the request target (path + query) **exactly as sent** and the **verbatim** body bytes — don't add a trailing slash, reorder query params, or re-serialize the JSON after signing.

#### 1. Create the payment

```bash
curl -X POST http://localhost:3001/api/merchant/payments \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amount":"5000","fiat":"NGN","token":"USDC","country":"NG","reference":"abc","purpose":"seize global content","type":"other"}'
```

The body is a `PaymentRequest`: `amount` (decimal string, in the `fiat` currency), `fiat` (local currency, e.g. `NGN`), `token` (the settlement token you receive, e.g. `USDC`), `country` (ISO 3166-1 alpha-2), `reference`, `purpose`, `type`, and an optional `validUntil` (ISO 8601 expiry). `type` defaults to `other` when omitted. See the OpenAPI `PaymentRequest` schema for the full field list and the `type` enum.

`201` on success:

```json
{ "data": {
    "redirectUrl": "https://app.blinx.example/payment/foam-bark-95162",
    "validUntil": "2026-08-01T00:00:00.000Z",
    "code": "foam-bark-95162",
    "id": "7b2c1e90-…" } }
```

* **Give the payer the `redirectUrl`.** They complete the payment there; the payment starts at status `created`. The URL's last path segment is the payer `code`, not the `id`.
* **Capture the `id`** returned alongside `redirectUrl` and `code` — it's the payment id you poll with in step 2. `validUntil` is when the payment expires.
* A request that fails validation → `400 PAY_INVALID_QUOTE_REQUEST`, with an `errors` **array** of `{ field, value, message }` entries (every failing field reported at once): a non-positive `amount`, a `fiat` that does not match `country`, an unsupported `token`, an invalid `network`/`type`/`validUntil`, a missing/unsafe `reference`, or an **amount too low to produce a payout** (an `amount` that is within the corridor range but whose settlement token amount, after fees, rounds to `0` — reported as an `amount` field error, message `amount is too low to produce a payout`). **Validate the amount against the corridor range up front by calling `POST /api/merchant/quote`** — it rejects an out-of-range amount with `400 PAY_AMOUNT_OUT_OF_RANGE`, its message carrying the bounds, and an in-range-but-too-small amount with `400 PAY_LOW_AMOUNT` (same "too low to produce a payout" reason). The range is the **country-wide envelope**: the top-level `payments.min` / `payments.max` from `GET /api/merchant/data/{country}`, **not** the per-channel `payments.bank` / `payments.momo` bounds (a payment does not pick a payout channel at creation; the per-channel bounds are informational only).
* An unsupported `country`/`fiat` → `400 PAY_COUNTRY_NOT_SUPPORTED` (here `errors` is a single `{ field, value }` object, not an array).
* Reusing a `reference` that still belongs to a live or settled payment → `409 DB_409_DUPLICATE_ENTRY`. A reference frees up only once its payment reaches a failed terminal state (`expired`/`rejected`/`failed`/`cancelled`).

#### 2. Check one payment

Create hands you the payment `id`, but the outcome arrives later. Read the full `Payment` by that id:

```bash
curl http://localhost:3001/api/merchant/payments/7b2c1e90-… \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
```

`200`, with the payment wrapped in `data`:

```json
{
  "data": {
    "id": "7b2c1e90-…",
    "status": "waiting",
    "amount": "300",
    "currencyIn": "USDC",
    "currencyOut": "NGN",
    "fees": "0",
    "type": "other",
    "reference": "abc",
    "purpose": "seize global content",
    "country": "NG",
    "source": {
      "currency": "USDC"
    },
    "destination": {
      "currency": "NGN",
      "country": "NG",
      "recipient": {
        "accountName": "Ada Lovelace",
        "institution": "GTB",
        "accountIdentifier": "0123456789"
      }
    },
    "createdAt": "2026-07-27T10:15:00.000Z"
  }
}
```

* Poll this until `status` reaches a terminal value — `settled` (paid) or `failed` / `rejected` / `expired` / `cancelled` (not paid). Everything else is in-flight; keep waiting.
* A payment not owned by the caller's tenant → `404 PAY_404_NOT_FOUND`. The same 404 covers unknown ids: existence is never disclosed.
* **Null/unknown fields are omitted, not sent as `null`.** The API strips null/undefined values from every response, so fields like `amountInUsd` or `cancelledAt` are simply absent until they have a value — read a missing key as "not set", and don't expect an explicit `null`.
* `source`/`destination` are the internal transfer shape and may carry more fields than shown; see the OpenAPI `Payment` schema.

Poll `GET /api/merchant/payments/{id}` on a timer until `status` reaches a terminal state.

#### 3. List payments (bulk)

For reconciliation over many payments, list them instead of fetching one by one:

```bash
# JSON: { "data": [ PaymentSummary, … ] }  (wrapped in `data`)
curl http://localhost:3001/api/merchant/payments \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"

# CSV export — remember to sign the path *with* its query string
curl 'http://localhost:3001/api/merchant/payments?format=csv' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
```

Each row is a condensed `PaymentSummary` (flattened for list/CSV display), not the full `Payment`; retrieve the full description per id with step 2. `?format=csv` returns the same rows as an RFC 4180 CSV file (`text/csv`, `Content-Disposition: attachment`). See the OpenAPI `PaymentSummary` schema and the `getTransfers` operation for the exact columns.

### Errors

| Status / code                                                 | When                                                                                                                                                                                                                                                           |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `400 PAY_INVALID_QUOTE_REQUEST`                               | Request failed validation; `errors` is an array of `{ field, value, message }` (bad `amount`, mismatched `fiat`/`country`, unsupported `token`, invalid `network`/`type`/`validUntil`, missing/unsafe `reference`, or an `amount` too low to produce a payout) |
| `400 PAY_AMOUNT_OUT_OF_RANGE`                                 | Quote only: `amount` is outside the corridor's country-wide `payments.min`/`payments.max` envelope (message carries the bounds)                                                                                                                                |
| `400 PAY_LOW_AMOUNT`                                          | Quote only: `amount` is within range but its settlement-token payout, after fees, rounds to `0` (`amount is too low to produce a payout`)                                                                                                                      |
| `400 PAY_COUNTRY_NOT_SUPPORTED`                               | `country`/`fiat` maps to no supported corridor                                                                                                                                                                                                                 |
| `409 DB_409_DUPLICATE_ENTRY`                                  | `reference` already belongs to a live or settled payment                                                                                                                                                                                                       |
| `401 AUTH_401_INVALID_SIGNATURE` / `AUTH_401_INVALID_API_KEY` | Bad/missing signature, key, or stale timestamp (see the Content-Type note above)                                                                                                                                                                               |
| `403 PAY_UNAUTHORIZED_ACCESS`                                 | Tenant valid but has no merchant user                                                                                                                                                                                                                          |
| `404 PAY_404_NOT_FOUND`                                       | Payment unknown, or not owned by the caller's tenant — existence is not disclosed                                                                                                                                                                              |

### Notes

* **Create returns the `id` directly** (alongside `redirectUrl`, `code`, and `validUntil`); capture it to poll `GET /api/merchant/payments/{id}` later. The `id` is **not** in `redirectUrl` — that URL ends in the payer `code` — but you can recover the `id` from your `reference` via the list endpoint.
* **Terminal is terminal.** Once `status` is `settled`, `failed`, `rejected`, `expired`, or `cancelled`, it won't change again — stop polling.
* **Sign query strings too.** For the CSV export, the signed `path` includes `?format=csv` exactly as sent — sign the full target, not the bare path.
* **`amount` currencies differ across views.** On create, `amount` is in the `fiat` the payer pays; in `PaymentSummary` the `amount`/`currency` are the destination (settlement) currency. Don't compare them blindly.

_Updated: 21 September 2026 12:57_
