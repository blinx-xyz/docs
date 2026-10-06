# Create a Payment

### Create and check a payment

A practical guide to creating a **payment**, handing it off to the payer, and learning its outcome. For the exact request/response contract see the OpenAPI spec (`seller-payments.openapi.yaml`, tag _Merchant Payments_); for signing see the main README and Sign a payment request.

#### Concepts

* **Payment** — a request for a payer to pay you. You create it in the local **`fiat`** currency the payer pays; you (the merchant) are settled in the **`token`** you choose. `amount` is the decimal amount in that `fiat` currency.
* **`redirectUrl`** — the hosted URL returned on create. The payer completes the payment there; your integration's only job is to get the payer to it. Its last path segment is the payment **`code`** (see below), not the payment `id`.
* **`code`** — a short, human-readable payer code (format `word-word-####C`, e.g. `foam-bark-95162`) returned alongside `id` on create. It is what appears in `redirectUrl`; the internal payment `id` never leaks into the payer link.
* **Corridor** — the `(country, fiat)` pair. It decides which amount limits apply and whether the payment can be created at all. Currencies shared across countries (XOF, XAF) are bounded **per country**, so pass `country` to disambiguate; it falls back to `fiat` when omitted.
* **`items`** — optional line items (what the payer is buying), shown to the payer on the hosted page. See Line items.
* **Quote** — the price of converting the payer's `fiat` into your `token`, and the choice of payout **provider** that will execute it. See How quotes work.
* **`status`** — where the payment is in its lifecycle. One of `created`, `viewed`, `waiting`, `processing`, `validated`, `refunding`, `refunded`, `expired`, `rejected`, `failed`, `cancelled`, `settled`. **Terminal**: `settled` (success) and `failed` / `rejected` / `expired` / `cancelled` (not paid). The rest are in-flight; `viewed` means the payer opened the link but has not confirmed yet.
* **`reference`** — your own unique reference for the payment. It travels back on reads so you can reconcile against your records.
* **`purpose`** — a human-readable description shown to the payer; it maps to the transfer memo and comes back on the `Payment`.
* **`type`** — the payment category, one of `gift`, `bills`, `groceries`, `travel`, `health`, `entertainment`, `housing`, `school-fees`, `other`. Defaults to `other` when omitted.

#### Lifecycle

```
POST /api/merchant/quote             → { data: QuoteResponse }   (optional: check price & limits)
      │
      ▼
POST /api/merchant/payments          → { data: { redirectUrl, validUntil, code, id } }   (status: created)
      │
      ▼  give the payer the redirectUrl
payer opens the hosted page (status: viewed), sees purpose + items, confirms
      │
      ▼  learn the outcome
GET /api/merchant/payments/{id}/status → { data: { status } }   (poll until terminal)
      │
      ▼
status ∈ { settled | failed | rejected | expired | cancelled }   (terminal)
```

Create the payment, give the payer its `redirectUrl`, then wait for a terminal `status` by polling `GET /api/merchant/payments/{id}/status` (or the full `GET /api/merchant/payments/{id}`).

#### Endpoints

All endpoints are under `/api/merchant` and require the standard apiKey + apiSecret HMAC signature, except where noted.

| Method & path                                 | Purpose                                                              |
| --------------------------------------------- | -------------------------------------------------------------------- |
| `POST /api/merchant/quote`                    | Best quote across providers for a currency pair and amount           |
| `POST /api/merchant/payments`                 | Create a payment; returns the hosted `redirectUrl`                   |
| `GET /api/merchant/payments/{id}`             | Get one payment, in any status                                       |
| `GET /api/merchant/payments/{id}/status`      | Get only the payment's `status` (lightweight polling)                |
| `GET /api/merchant/payments`                  | List the merchant's payments — JSON array, or CSV with `?format=csv` |
| `GET /api/merchant/payments/limits/{country}` | Amount limits for a country — **not signed**, no auth headers needed |

#### How quotes work

Every payment converts the payer's **fiat** into your **token** through a payout **provider**. Several providers can serve the same corridor at different prices, so Blinx asks all of them and keeps the best one.

```
                 QuoteRequest { amount, currencyIn, currencyOut, country }
                                       │
                                       ▼
                  ┌──────────────── derive side ────────────────┐
                  │ fiat  → token  = receive   (payment case)   │
                  │ token → fiat   = send                       │
                  │ token → token  = transfer                   │
                  │ fiat  → fiat   = rejected (400)             │
                  └─────────────────────────────────────────────┘
                                       │
                                       ▼
                 validate: country/currency match, amount > 0, supported token
                                       │
                                       ▼
           token not USDC (USDT, EURe)? ──yes──► quote the providers in USDC,
                                       │         bridge with a swap quote
                                       ▼
          ┌────────────────────┬───────┴───────────┬────────────────────┐
          ▼                    ▼                   ▼                    ▼
     Provider "1"         Provider "2"            ...          (only `providers`                         
     amountOut, fee       amountOut, fee                        if you pass it)
          └────────────────────┴───────┬───────────┴────────────────────┘
                                       ▼
                      pick the highest amountOut  ──► providerId
                                       │
                         none succeeded? ──► 400 PAY_AMOUNT_OUT_OF_RANGE /
                                       │      PAY_LOW_AMOUNT / PAY_FAILED_QUOTE
                                       ▼
          QuoteResponse { amountIn, amountOut, rate, localFee, providerId, ttl }
                     (cached for ttl seconds — default 1800)
```

What this means for a **payment**:

```
POST /quote   ── indicative ──► shows you price, fee and limits; binds nothing
POST /payments ─────────────► takes its OWN fresh best quote → stores providerId
                               │
                               └─ no provider could quote right now?
                                    payment is still created without a provider
PUT  (payer confirms) ───────► provider stored? use it
                               otherwise take a fresh best quote now;
                               if that fails the confirm fails, the payment
                               stays confirmable and the payer can retry
```

* **The quote you see is not a reservation.** Rates move; the provider and price that apply are fixed when the payment is created (or, failing that, when the payer confirms).
* **Use the quote to check an amount before creating.** An out-of-range amount fails with `PAY_AMOUNT_OUT_OF_RANGE` (the message carries the bounds), and an amount too small to pay out anything after fees fails with `PAY_LOW_AMOUNT`.
* **`rate`** is fiat per token unit; **`localFee`** is the provider fee in the fiat; **`amountOut`** is what you would receive in `currencyOut` after fees.

#### Walkthrough

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

**1. (Optional) Quote the payment**

Check the price and that the amount is payable for the corridor:

```bash
curl -X POST http://localhost:3001/api/merchant/quote \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amount":"5000","currencyIn":"NGN","currencyOut":"USDC","country":"NG"}'
```

For a payment, `currencyIn` is the `fiat` the payer pays and `currencyOut` is your settlement `token`. `200`:

```json
{ "data": {
    "currencyIn": "NGN", "currencyOut": "USDC",
    "amountIn": "5000", "amountOut": "3.02",
    "rate": "1650.00", "localFee": "25",
    "providerId": "2", "ttl": 1800 } }
```

Optional body fields: `channel` (`bank` | `momo`) and `providers` (e.g. `["2"]`) to restrict which providers compete.

**2. Create the payment**

```bash
curl -X POST http://localhost:3001/api/merchant/payments \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amount":"5000","fiat":"NGN","token":"USDC","country":"NG","reference":"abc","purpose":"seize global content","type":"other","items":[{"name":"Annual subscription","description":"Global content plan, 12 months","price":"4000","units":"1"},{"name":"Setup fee","price":"1000","units":"1"}]}'
```

The body is a `PaymentRequest`:

| Field        | Required | Description                                                                                                               |
| ------------ | -------- | ------------------------------------------------------------------------------------------------------------------------- |
| `amount`     | yes      | Decimal string, in the `fiat` currency. Must fall within the corridor's `payments.min`/`payments.max`.                    |
| `fiat`       | yes      | Local currency the payer pays (e.g. `NGN`). Must be `country`'s currency.                                                 |
| `token`      | yes      | Settlement token you receive: `USDC` or `USDT`.                                                                           |
| `country`    | yes      | ISO 3166-1 alpha-2 code (e.g. `NG`).                                                                                      |
| `reference`  | yes      | Your unique reference.                                                                                                    |
| `purpose`    | yes      | Description shown to the payer.                                                                                           |
| `type`       | no       | Payment category (see Concepts). Defaults to `other`.                                                                     |
| `validUntil` | no       | Future ISO 8601 expiry. Defaults to 24 hours after creation.                                                              |
| `network`    | no       | Settlement network; your vault must hold an active `{token}_{network}` address. Defaults to your vault's default address. |
| `items`      | no       | Line items shown to the payer — see below.                                                                                |

**Line items**

`items` is an optional list describing what the payer is paying for. Each item is shown on the hosted payment page under the payment's `purpose`.

| Item field    | Required | Type / limit                   | Meaning                                             |
| ------------- | -------- | ------------------------------ | --------------------------------------------------- |
| `name`        | yes      | string, non-empty, ≤ 255 chars | Item label, e.g. `Annual subscription`              |
| `description` | no       | string, no length limit        | Longer text, e.g. `Global content plan, 12 months`  |
| `price`       | no       | numeric string, ≤ 255 chars    | Unit price, in the payment's `fiat` (e.g. `"4000"`) |
| `units`       | no       | string, ≤ 255 chars            | Quantity, free text (e.g. `"1"`, `"2 kg"`)          |

* **Items are informational.** They are not summed or checked against `amount` — the payer is always charged `amount`. Keep them consistent with it yourself.
* **1 to 200 items.** If you send `items`, it must be a non-empty array of at most 200 entries. Omit the field entirely for a payment without items.
* **Read them back** on `GET /api/merchant/payments/{id}` (not on the list endpoint).
* **All or nothing.** Every item is validated before the payment is created; one bad item (missing `name`, non-numeric `price`, an over-long field) fails the whole request with an `items` field error, so a payment is never created with only some of its items.

`201` on success:

```json
{ "data": {
    "redirectUrl": "https://app.blinx.example/payment/foam-bark-95162",
    "validUntil": "2026-08-01T00:00:00.000Z",
    "code": "foam-bark-95162",
    "id": "7b2c1e90-…" } }
```

* **Give the payer the `redirectUrl`.** They complete the payment there; the payment starts at status `created`. The URL's last path segment is the payer `code`, not the `id`.
* **Capture the `id`** returned alongside `redirectUrl` and `code` — it's the payment id you poll with in step 3. `validUntil` is when the payment expires.
* A request that fails validation → `400 PAY_INVALID_QUOTE_REQUEST`, with an `errors` **array** of `{ field, value, message }` entries (every failing field reported at once): a non-positive or out-of-range `amount`, a `fiat` that does not match `country`, an unsupported `token`, an invalid `network`/`type`/`validUntil`, a missing/unsafe `reference`/`purpose`, malformed `items`, or an **amount too low to produce a payout** (reported as an `amount` field error, message `amount is too low to produce a payout`). The amount range is the **country-wide envelope**: the top-level `payments.min` / `payments.max` from `GET /api/merchant/data/{country}` (or `GET /api/merchant/payments/limits/{country}`), **not** the per-channel `payments.bank` / `payments.momo` bounds.
* An unsupported `country`/`fiat` → `400 PAY_COUNTRY_NOT_SUPPORTED` (here `errors` is a single `{ field, value }` object, not an array).
* Reusing a `reference` that still belongs to a live or settled payment → `409 DB_409_DUPLICATE_ENTRY`. A reference frees up only once its payment reaches a failed terminal state (`expired`/`rejected`/`failed`/`cancelled`).

**3. Check one payment**

Create hands you the payment `id`, but the outcome arrives later. For polling, read just the status:

```bash
curl http://localhost:3001/api/merchant/payments/7b2c1e90-…/status \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
# → { "data": { "status": "processing" } }
```

Or read the full `Payment`:

```bash
curl http://localhost:3001/api/merchant/payments/7b2c1e90-… \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
```

`200`, with the payment wrapped in `data`:

```json
{
  "data": {
    "id": "7b2c1e90-…",
    "status": "processing",
    "type": "payment",
    "amount": "5000",
    "currencyIn": "NGN",
    "currencyOut": "USDC",
    "fees": "25",
    "country": "NG",
    "reference": "abc",
    "purpose": "seize global content",
    "paymentType": "bank",
    "payerName": "Ada Lovelace",
    "tenant": "Acme",
    "seller": "seller@example.com",
    "destinationAddress": "0x8439cca53a7f7bcfa3bad61a73e8eceae12d1589",
    "destinationNetwork": "arbitrum",
    "items": [
      { "name": "Annual subscription", "description": "Global content plan, 12 months", "price": "4000", "units": "1" },
      { "name": "Setup fee", "price": "1000", "units": "1" }
    ],
    "createdAt": "2026-07-27T10:15:00.000Z",
    "updatedAt": "2026-07-27T10:21:40.000Z"
  }
}
```

* Poll until `status` reaches a terminal value — `settled` (paid) or `failed` / `rejected` / `expired` / `cancelled` (not paid). Everything else is in-flight; keep waiting. When a payment fails or expires, `failure` carries the reason.
* `items` echoes the line items sent on create (absent when the payment has none). The list endpoint does not include them.
* `paymentType` / `payerName` appear once the payer has confirmed (they describe the payer's bank or mobile-money account).
* A payment not owned by the caller's tenant → `404 PAY_404_NOT_FOUND`. The same 404 covers unknown ids: existence is never disclosed.
* **Null/unknown fields are omitted, not sent as `null`.** Fields like `amountUsd`, `failure` or `cancelledAt` are simply absent until they have a value — read a missing key as "not set".

**4. List payments (bulk)**

For reconciliation over many payments, list them instead of fetching one by one:

```bash
# JSON: { "data": [ PaymentSummary, … ] }  (wrapped in `data`)
curl http://localhost:3001/api/merchant/payments \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"

# CSV export — remember to sign the path *with* its query string
curl 'http://localhost:3001/api/merchant/payments?format=csv' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
```

Rows have the same flat shape as the single-payment read. The list also includes your vault top-ups — filter on `type` (`payment` | `topup`). Payments still `created`/`viewed` past their `validUntil` are moved to `expired` as the list is read. `?format=csv` returns the same rows as an RFC 4180 CSV file (`text/csv`, `Content-Disposition: attachment`). See the OpenAPI `PaymentSummary` schema for the exact columns.

#### Errors

| Status / code                                                 | When                                                                                                                                                                                                                                                                                                                                                                |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `400 PAY_INVALID_QUOTE_REQUEST`                               | Create: request failed validation; `errors` is an array of `{ field, value, message }` (bad/out-of-range `amount`, mismatched `fiat`/`country`, unsupported `token`, invalid `network`/`type`/`validUntil`, missing/unsafe `reference`/`purpose`, malformed `items`, or an `amount` too low to produce a payout). Quote: non-positive `amount` or unsupported token |
| `400 PAY_400_INVALID_REQUEST`                                 | Quote only: fiat → fiat pair, or a fiat that does not match `country`                                                                                                                                                                                                                                                                                               |
| `400 PAY_AMOUNT_OUT_OF_RANGE`                                 | Quote only: `amount` is outside the corridor's country-wide envelope (message carries the bounds)                                                                                                                                                                                                                                                                   |
| `400 PAY_LOW_AMOUNT`                                          | Quote only: `amount` is within range but its payout, after fees, rounds to `0`                                                                                                                                                                                                                                                                                      |
| `400 PAY_FAILED_QUOTE`                                        | Quote only: no provider could quote the request                                                                                                                                                                                                                                                                                                                     |
| `400 PAY_COUNTRY_NOT_SUPPORTED`                               | `country`/`fiat` maps to no supported corridor                                                                                                                                                                                                                                                                                                                      |
| `409 DB_409_DUPLICATE_ENTRY`                                  | `reference` already belongs to a live or settled payment                                                                                                                                                                                                                                                                                                            |
| `401 AUTH_401_INVALID_SIGNATURE` / `AUTH_401_INVALID_API_KEY` | Bad/missing signature, key, or stale timestamp (see the Content-Type note above)                                                                                                                                                                                                                                                                                    |
| `403 PAY_UNAUTHORIZED_ACCESS`                                 | Tenant valid but has no merchant user                                                                                                                                                                                                                                                                                                                               |
| `404 PAY_404_NOT_FOUND`                                       | Payment unknown, or not owned by the caller's tenant — existence is not disclosed                                                                                                                                                                                                                                                                                   |

#### Notes

* **Create returns the `id` directly** (alongside `redirectUrl`, `code`, and `validUntil`); capture it to poll later. The `id` is **not** in `redirectUrl` — that URL ends in the payer `code` — but you can recover the `id` from your `reference` via the list endpoint.
* **Terminal is terminal.** Once `status` is `settled`, `failed`, `rejected`, `expired`, or `cancelled`, it won't change again — stop polling.
* **Sign query strings too.** For the CSV export, the signed `path` includes `?format=csv` exactly as sent — sign the full target, not the bare path.
* **Quotes are indicative.** The provider and price are fixed when the payment is created, not when you quote — see How quotes work.
* **`amount` is always in the payer's `fiat`.** On create and on reads, `amount` is denominated in `currencyIn` (the fiat); `currencyOut` is your settlement token.

_Updated: 5 October 2026_
