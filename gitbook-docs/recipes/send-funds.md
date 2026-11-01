# Send Funds

{% hint style="info" %}
The full request/response contract is in the latest OpenAPI spec: [blinx-api.yaml](https://openapi.gitbook.com/o/qIJeTD9Swqwc0lE9nsEa/spec/blinx-api.yaml).
{% endhint %}

### Pay a third-party receiver from your vault

A practical guide to **sending** funds from your vault to a third party — a supplier, a partner, a payee — paid out in their local currency to their bank or mobile-money account. You check your balance, quote the payout, pick and verify the receiver's account, create the send, and follow its status until it settles. For the exact request/response contract see the OpenAPI spec (tag _Merchant Send_).

#### Concepts

* **Send** — a payout from your vault to a **receiver**. It leaves your vault in **USD** (USDC, `currencyIn`) and reaches the receiver in their local **fiat** (`currencyOut`).
* **Balance** — what your vault holds, from `GET /api/merchant/balance`. A send's `amountIn` must not exceed it.
* **Receiver** — the third party you pay. You describe their account as the send **`destination`**: `{ accountName, accountIdentifier, networkId }`.
* **Network** — the bank or mobile-money (momo) network the receiver is paid on, identified by a **`networkId`** from `GET /api/merchant/network/{country}?side=send`.
* **`accountIdentifier`** — the receiver's bank account number, or their mobile number (with a leading `+`) on a momo network. A mobile number on a **bank** network is refused.
* **`status`** — the same lifecycle as a payment: in-flight (`created`, `waiting`, `processing`, …) until **terminal** `settled` (paid out) or `failed` / `rejected` / `expired` / `cancelled`.
* **`reference`** — your own unique reference. It travels back on reads and webhooks.

#### Lifecycle

```
Balance      GET  /api/merchant/balance                      → { data: { balance, currency } }
                │
Quote        POST /api/merchant/quote                        → { data: QuoteResponse }        (side: send)
                │
Destination  GET  /api/merchant/network/{country}?side=send  → { data: Network[] }
             POST /api/merchant/verification?side=send       → { data: { success, accountName? } }
                │
Send         POST /api/merchant/send                         → { data: { id, reference, status } }
                │
Settle       the provider pays the receiver
                │
Check        GET  /api/merchant/send/{id}/status             → { data: { status } }   (or a PAYMENT_UPDATE webhook)
                ▼
             status ∈ { settled | failed | rejected | expired | cancelled }
```

#### Endpoints

All endpoints are under `/api/merchant` and require the standard apiKey + apiSecret HMAC signature.

| Method & path                                    | Purpose                                                         |
| ------------------------------------------------ | --------------------------------------------------------------- |
| `GET /api/merchant/balance`                      | Your vault balance — what a send can pay out                    |
| `POST /api/merchant/quote`                       | Price the send (token in → fiat out)                            |
| `GET /api/merchant/network/{country}?side=send`  | Networks (bank / momo) a send can pay out to in `country`       |
| `POST /api/merchant/verification?side=send`      | Verify the receiver's account and resolve the holder's name     |
| `POST /api/merchant/send`                        | Create the send                                                 |
| `GET /api/merchant/send/{id}`                    | Get one send, in any status                                     |
| `GET /api/merchant/send/{id}/status`             | Get only its `status` (lightweight polling)                     |
| `POST /api/merchant/webhooks`                    | Register a webhook for `PAYMENT_UPDATE` status changes          |

#### Before you start

* Your tenant has passed **KYB and KYC** — `POST /send` answers `403 GATE_403_KYB_REQUIRED` / `GATE_403_KYC_REQUIRED` otherwise.
* Your vault holds enough **USDC** for the send — fund it first, e.g. with a [receive](receive-funds.md).

#### Signing requests

Every request below must be **signed**: send `x-api-key`, `x-timestamp` and `x-signature` (an HMAC-SHA256 of the request keyed by your apiSecret). See [Signing a request from the CLI](../signing-a-request-from-the-cli.md) for the signature and a ready-made signing script, or [Signing requests in Postman](../signing-requests-in-postman.md).

In the snippets, `blinx(method, path, body)` stands for a signed call that returns the response's `data`, and `$KEY` / `$TS` / `$SIG` are the signed headers for that request.

#### Walkthrough

**1. Check your balance**

What your vault can pay out:

```js
const { balance, currency } = await blinx('GET', '/api/merchant/balance');
// { balance: '31.58', currency: 'usd' }
```

```bash
curl "$BLINX_BASE_URL/api/merchant/balance" \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
# → { "data": { "balance": "31.58", "currency": "usd" } }
```

* `balance` is cut to 2 decimals (truncated, never rounded up); `currency` is your vault asset by its fiat denomination (`USDC` → `usd`).
* No vault address → `404 PAY_VAULT_NOT_FOUND`; balance temporarily unavailable → `503 PAY_503_VAULT_PROVIDER_UNAVAILABLE` (retry later).

**2. Quote the send**

The quote is a `send` (USD → the receiver's fiat). Send **one** of the two amounts — the quote solves the other, fees included:

| You know…                                  | Send                            | The quote tells you                                   |
| ------------------------------------------ | ------------------------------- | ----------------------------------------------------- |
| the USD you want to send                   | `amountIn` (in USD)             | `amountOut` — how much the receiver gets, in fiat     |
| the local amount the receiver must get     | `amountOut` (in the local fiat) | `amountIn` — how much USD leaves your vault           |

**a) Fixed USD — how much will the receiver get?**

```js
const quote = await blinx('POST', '/api/merchant/quote', {
  amountIn: '10',           // the USD you send
  currencyIn: 'USD',
  currencyOut: 'NGN',
  country: 'NG',
});
// { currencyIn: 'USD', currencyOut: 'NGN', amountIn: '10.00', amountOut: '13240.57',
//   rate: '1344.22', localFee: '201.63', side: 'send', ttl: 1800 }
// → the receiver gets 13,240.57 NGN
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/quote" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amountIn":"10","currencyIn":"USD","currencyOut":"NGN","country":"NG"}'
```

**b) Fixed local amount — how much USD does it take?**

```js
const quote = await blinx('POST', '/api/merchant/quote', {
  amountOut: '20000',       // what the receiver must get, in NGN
  currencyIn: 'USD',
  currencyOut: 'NGN',
  country: 'NG',
});
// { currencyIn: 'USD', currencyOut: 'NGN', amountIn: '15.11', amountOut: '20000.00',
//   rate: '1344.22', localFee: '304.57', side: 'send', ttl: 1800 }
// → 15.11 USD leaves your vault
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/quote" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amountOut":"20000","currencyIn":"USD","currencyOut":"NGN","country":"NG"}'
```

* Either way, keep the quote's `amountIn`, `amountOut` and `rate`: the send in step 5 uses them. Figures above are illustrative — rates and fees move.
* `localFee` is the provider fee in the local fiat; it is taken out of the payout, so check `amountOut` is what the receiver should get.
* Make sure `amountIn` fits your balance from step 1.
* An amount no provider serves → `400 PAY_AMOUNT_OUT_OF_RANGE` (bounds in the message); no provider able to quote → `503 PAY_FAILED_QUOTE` (retry later).

**3. List the receiver's networks**

The networks a send can pay out to in the receiver's country — note `?side=send`:

```js
const networks = await blinx('GET', '/api/merchant/network/NG?side=send');
// [ { networkId: '5f1af11b-…', name: 'Access Bank', channel: 'bank', country: 'NG' }, … ]

const network = networks.find((n) => n.channel === 'bank' && n.name === 'Access Bank');
```

```bash
curl "$BLINX_BASE_URL/api/merchant/network/NG?side=send" \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
```

Pick the receiver's bank or mobile-money operator from this list; keep its `networkId`. Remember to sign the path **with** its query string.

**4. Verify the receiver's account**

Check the account on its network before sending. The provider resolves the holder's name when it can:

```js
const check = await blinx('POST', '/api/merchant/verification?side=send', {
  networkId: network.networkId,
  accountIdentifier: '1111111111',   // bank account number, or +<mobile> on a momo network
  country: 'NG',
});
// { success: true, accountName: 'Ken Adams' }
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/verification?side=send" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"networkId":"5f1af11b-305f-4420-8fce-65ed2725a409","accountIdentifier":"1111111111","country":"NG"}'
```

* `success: false` (still `200`) means the provider could not confirm the account — check it with the receiver before sending.
* A mobile number on a **bank** network → `400 PAY_400_INVALID_REQUEST` on `accountIdentifier`. A `networkId` that is not a payout network of the country → `400` on `networkId`.
* In the sandbox, `0000000000` and `1111111111` always verify (as `Ken Adams` on a bank network).

**5. Create the destination and the send**

Build the `destination` from the network and the verified account, then create the send with the quote's figures:

```js
const destination = {
  accountName: check.accountName ?? 'Jane Doe',   // the verified name, or the one the receiver gave
  accountIdentifier: '1111111111',
  networkId: network.networkId,
};

const send = await blinx('POST', '/api/merchant/send', {
  amountIn: quote.amountIn,
  currencyIn: 'USD',
  amountOut: quote.amountOut,
  currencyOut: 'NGN',
  rate: quote.rate,
  country: 'NG',
  reference: 'payout-0001',     // unique per tenant
  memo: 'Payout 0001',
  destination,
});
// { id: '628f5d94-47d8-4538-9049-ca1107ddef85', reference: 'payout-0001', status: 'created' }
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/send" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amountIn":"10.00","currencyIn":"USD","amountOut":"13240.57","currencyOut":"NGN","rate":"1344.22","country":"NG","reference":"payout-0001","memo":"Payout 0001","destination":{"accountName":"Ken Adams","accountIdentifier":"1111111111","networkId":"5f1af11b-305f-4420-8fce-65ed2725a409"}}'
```

The body is a `SendRequest`:

| Field         | Required | Description                                                                                       |
| ------------- | -------- | ------------------------------------------------------------------------------------------------- |
| `amountIn`    | yes      | What leaves your vault, in USD — from the quote. Must not exceed your balance.                    |
| `currencyIn`  | yes      | Your vault token: `USD` (or `USDC`).                                                              |
| `amountOut`   | yes      | What the receiver gets, in `currencyOut` — from the quote.                                        |
| `currencyOut` | yes      | The receiver's fiat (`country`'s currency).                                                       |
| `rate`        | yes      | The quote rate (informational — the send executes at the provider's quote).                      |
| `destination` | yes      | `{ accountName, accountIdentifier, networkId }` — all three required.                             |
| `country`     | no       | The receiver's country. Defaults to your merchant user's country.                                 |
| `reference`   | no       | Your unique reference; generated when omitted. Reusing a live one → `409 DB_409_DUPLICATE_ENTRY`.  |
| `memo`        | no       | Free text kept on the send.                                                                       |

* The send always pays out from your vault address — you never pass a token or network.
* `amountIn` above your balance → `402 PAY_402_INSUFFICIENT_FUNDS`, with an `amountIn` error carrying the available balance. Nothing is created.
* Every invalid field is reported at once: `400 PAY_400_INVALID_REQUEST` with an `errors` array (e.g. `destination.networkId`, `currencyIn`).
* **Keep the `id`** — you check the send with it in step 7.

**6. The provider pays the receiver**

Nothing to do on your side: the provider pays the receiver's bank or mobile-money account from your vault funds. The send moves through `processing` while the payout is in flight.

**7. Check the send**

Poll the status until it is terminal:

```js
const TERMINAL = ['settled', 'failed', 'rejected', 'expired', 'cancelled'];

async function waitForSend(id, { everyMs = 30_000, timeoutMs = 24 * 3600_000 } = {}) {
  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    const { status } = await blinx('GET', `/api/merchant/send/${id}/status`);
    if (TERMINAL.includes(status)) return status;
    await new Promise((r) => setTimeout(r, everyMs));
  }
  throw new Error('send still in flight');
}

const status = await waitForSend(send.id);   // 'settled' when the receiver was paid
```

```bash
curl "$BLINX_BASE_URL/api/merchant/send/628f5d94-47d8-4538-9049-ca1107ddef85/status" \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
# → { "data": { "status": "processing" } }
```

`GET /api/merchant/send/{id}` returns the full send:

```json
{ "data": {
    "id": "628f5d94-47d8-4538-9049-ca1107ddef85", "reference": "payout-0001",
    "status": "processing", "type": "send",
    "amount": "13240.57", "currencyIn": "USDC", "amountUsd": "10", "currencyOut": "NGN",
    "fees": "0.10", "country": "NG", "paymentType": "bank", "payerName": "Ken Adams",
    "createdAt": "2026-11-01T16:32:27.263Z", "updatedAt": "2026-11-01T16:32:38.199Z" } }
```

`amount` is what the receiver gets (`currencyOut`); `amountUsd` is what left your vault. An id that is not a send of your tenant → `404 PAY_404_NOT_FOUND`.

**…or get the status pushed (webhook)**

Register a webhook once for `PAYMENT_UPDATE`: every status change of your payments, receives and sends is POSTed to it. See [Receive Funds](receive-funds.md), step 6 ("…or get the status pushed"), for the registration and how to verify the `X-Blinx-Signature`.

#### Errors

| Status | Code                                | When                                                                                   |
| ------ | ----------------------------------- | -------------------------------------------------------------------------------------- |
| 400    | `PAY_400_INVALID_REQUEST`           | A bad field (every one listed in `errors`): amounts, `currencyIn`/`currencyOut`, `destination`, unknown `networkId`, a mobile number on a bank network |
| 400    | `PAY_AMOUNT_OUT_OF_RANGE`           | No provider serves the amount (bounds in the message)                                  |
| 402    | `PAY_402_INSUFFICIENT_FUNDS`        | `amountIn` exceeds your vault balance                                                  |
| 403    | `GATE_403_KYB_REQUIRED` / `GATE_403_KYC_REQUIRED` | The tenant has not passed KYB / KYC                                      |
| 404    | `PAY_VAULT_NOT_FOUND`               | Your tenant has no vault address                                                       |
| 404    | `PAY_404_NOT_FOUND`                 | The id is not a send of your tenant                                                    |
| 409    | `DB_409_DUPLICATE_ENTRY`            | `reference` already used by a live or settled transfer                                 |
| 503    | `PAY_503_VAULT_PROVIDER_UNAVAILABLE` | Your balance could not be read right now — retry later                                |
| 503    | `PAY_FAILED_QUOTE`                  | No provider can quote right now — retry later                                          |

#### Notes

* **Balance, quote, then send.** Read the balance, take the figures from a fresh quote, and send them — the provider's quote decides the payout.
* **One reference per send.** A reference frees up only once its transfer fails (`expired` / `rejected` / `failed` / `cancelled`).
* **Receivers are saved as your recipients.** The `destination` of a send is stored as one of your recipients, so a returning receiver is recognised.
* **Sandbox.** Account `1111111111` on a bank network is the success account; a send moves real sandbox funds out of your vault.
