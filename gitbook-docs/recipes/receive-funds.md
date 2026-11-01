# Receive Funds

{% hint style="info" %}
The full request/response contract is in the latest OpenAPI spec: [blinx-api.yaml](https://openapi.gitbook.com/o/qIJeTD9Swqwc0lE9nsEa/spec/blinx-api.yaml).
{% endhint %}

### Receive money from a third-party sender

A practical guide to **receiving** funds from a third party — a customer, a partner, anyone who pays you in their local currency — straight into your vault, with no Blinx payer page in between. You create the receive, forward the deposit instructions (or let the provider collect a mobile-money payment), and follow the status until it settles. For the exact request/response contract see the OpenAPI spec (`blinx-openapi.yaml`, tag _Merchant Receive_); for signing see the main README and Sign a request.

#### Concepts

* **Receive** — a direct collection from a **sender**. The sender pays in their local **fiat** (`currencyIn`); you are settled in **USDC** to your vault address.
* **Sender** — the third party who pays. You describe their account as the receive **`source`**: `{ accountName, accountIdentifier, networkId }`.
* **Network** — the bank or mobile-money (momo) network the sender pays from, identified by a **`networkId`** from `GET /api/merchant/network/{country}`. Every `source` names its network by `networkId`.
* **`accountIdentifier`** — the sender's bank account number, or their mobile number (with a leading `+`) on a momo network. A mobile number on a **bank** network is refused.
* **`bankInfo`** — returned for a **bank** collection: the account, institution and reference the sender must pay to. A **momo** collection returns no `bankInfo` — the provider collects from the sender's wallet directly.
* **`status`** — the same lifecycle as a payment: in-flight (`created`, `waiting`, `processing`, …) until **terminal** `settled` (paid) or `failed` / `rejected` / `expired` / `cancelled`.
* **`reference`** — your own unique reference. It travels back on reads and webhooks, and is the reference a bank sender quotes on their transfer.

#### Lifecycle

```
Quote     POST /api/merchant/quote                 → { data: QuoteResponse }        (side: receive)
             │
Source    GET  /api/merchant/network/{country}     → { data: Network[] }
          POST /api/merchant/verification          → { data: { success, accountName? } }
             │
Receive   POST /api/merchant/receive               → { data: { id, bankInfo? } }
             │
             ├─ bank?  you forward bankInfo to the sender ─┐
             └─ momo?  the provider collects from them ────┤
                                                           ▼
Settle    the sender pays the provider
             │
Check     GET  /api/merchant/receive/{id}/status   → { data: { status } }   (or a PAYMENT_UPDATE webhook)
             ▼
          status ∈ { settled | failed | rejected | expired | cancelled }
```

#### Endpoints

All endpoints are under `/api/merchant` and require the standard apiKey + apiSecret HMAC signature.

| Method & path                            | Purpose                                                          |
| ---------------------------------------- | ---------------------------------------------------------------- |
| `POST /api/merchant/quote`               | Price the receive (fiat in → token out)                          |
| `GET /api/merchant/network/{country}`    | Networks (bank / momo) a sender in `country` can pay from        |
| `POST /api/merchant/verification`        | Verify the sender's account and resolve the holder's name        |
| `POST /api/merchant/receive`             | Create the receive; returns `id` and, for bank, `bankInfo`       |
| `GET /api/merchant/receive/{id}`         | Get one receive, in any status                                   |
| `GET /api/merchant/receive/{id}/status`  | Get only its `status` (lightweight polling)                      |
| `POST /api/merchant/webhooks`            | Register a webhook for `PAYMENT_UPDATE` status changes           |

#### Before you start

* Your tenant has passed **KYB and KYC** — `POST /receive` answers `403 GATE_403_KYB_REQUIRED` / `GATE_403_KYC_REQUIRED` otherwise.

#### Signing requests

Every request below must be **signed**: send `x-api-key`, `x-timestamp` and `x-signature` (an HMAC-SHA256 of the request keyed by your apiSecret). See [Signing a request from the CLI](../signing-a-request-from-the-cli.md) for the signature and a ready-made signing script, or [Signing requests in Postman](../signing-requests-in-postman.md).

In the snippets, `blinx(method, path, body)` stands for a signed call that returns the response's `data`, and `$KEY` / `$TS` / `$SIG` are the signed headers for that request.

#### Walkthrough

**1. Quote the receive**

The quote is a `receive` (the sender's fiat → USD). Send **one** of the two amounts — the quote solves the other, fees included:

| You know…                                  | Send                          | The quote tells you                                  |
| ------------------------------------------ | ----------------------------- | ---------------------------------------------------- |
| the USD you want to receive                | `amountOut` (in USD)          | `amountIn` — how much the sender must send, in fiat  |
| the local amount the sender will pay       | `amountIn` (in the local fiat) | `amountOut` — how much you will receive, in USD     |

**a) Fixed USD — how much should the sender send?**

```js
const quote = await blinx('POST', '/api/merchant/quote', {
  amountOut: '10',          // the USD you want to receive
  currencyIn: 'NGN',
  currencyOut: 'USD',
  country: 'NG',
});
// { currencyIn: 'NGN', currencyOut: 'USD', amountIn: '13509.75', amountOut: '10.00',
//   rate: '1344.22', localFee: '67.55', side: 'receive', ttl: 1800 }
// → the sender sends 13,509.75 NGN
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/quote" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amountOut":"10","currencyIn":"NGN","currencyOut":"USD","country":"NG"}'
```

**b) Fixed local amount — how much will you receive?**

```js
const quote = await blinx('POST', '/api/merchant/quote', {
  amountIn: '5000',         // what the sender pays, in NGN
  currencyIn: 'NGN',
  currencyOut: 'USD',
  country: 'NG',
});
// { currencyIn: 'NGN', currencyOut: 'USD', amountIn: '5000.00', amountOut: '3.66',
//   rate: '1344.22', localFee: '25.00', side: 'receive', ttl: 1800 }
// → you receive 3.66 USD
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/quote" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amountIn":"5000","currencyIn":"NGN","currencyOut":"USD","country":"NG"}'
```

* Either way, keep the quote's `amountIn`, `amountOut` and `rate`: the receive in step 4 uses them. Figures above are illustrative — rates and fees move.
* `localFee` is the provider fee in the local fiat; on a fixed-USD quote it is added on top, so the sender covers it.
* `USD` stands for USDC (the receive settles in USDC), and tokens come back by their fiat denomination.
* An amount no provider serves → `400 PAY_AMOUNT_OUT_OF_RANGE` (bounds in the message); no provider able to quote → `503 PAY_FAILED_QUOTE` (retry later). See Create a Payment → How quotes work.

**2. List the sender's networks**

The networks a sender in a country can pay from:

```js
const networks = await blinx('GET', '/api/merchant/network/NG');
// [ { networkId: '5f1af11b-…', name: 'Access Bank', channel: 'bank', country: 'NG' },
//   { networkId: '30f11b2e-…', name: 'MTN',         channel: 'momo', country: 'NG' }, … ]

const network = networks.find((n) => n.channel === 'bank' && n.name === 'Access Bank');
```

```bash
curl "$BLINX_BASE_URL/api/merchant/network/NG" \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
```

Let the sender pick their bank or mobile-money operator from this list; keep its `networkId`.

**3. Verify the sender's account**

Check the account on its network before creating the receive. The provider resolves the holder's name when it can:

```js
const check = await blinx('POST', '/api/merchant/verification', {
  networkId: network.networkId,
  accountIdentifier: '1111111111',   // bank account number, or +<mobile> on a momo network
  country: 'NG',
});
// { success: true, accountName: 'Ken Adams' }
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/verification" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"networkId":"5f1af11b-305f-4420-8fce-65ed2725a409","accountIdentifier":"1111111111","country":"NG"}'
```

* `success: false` (still `200`) means the provider could not confirm the account — ask the sender to check it.
* A mobile number on a **bank** network → `400 PAY_400_INVALID_REQUEST` on `accountIdentifier` (`accountIdentifier is a mobile number but Access Bank is a bank`). A `networkId` that is not a network of the country → `400` on `networkId`.
* In the sandbox, `0000000000` and `1111111111` always verify (as `Ken Adams` on a bank network).

**4. Create the source and the receive**

Build the `source` from the network and the verified account, then create the receive with the quote's figures:

```js
const source = {
  accountName: check.accountName ?? 'Jane Doe',   // the verified name, or the one the sender gave
  accountIdentifier: '1111111111',
  networkId: network.networkId,
};

const receive = await blinx('POST', '/api/merchant/receive', {
  amountIn: quote.amountIn,
  currencyIn: 'NGN',
  amountOut: quote.amountOut,
  rate: quote.rate,
  country: 'NG',
  reference: 'inv-0001',        // unique per tenant
  memo: 'Invoice 0001',
  source,
});
// bank → { id: 'c83ab522-…', bankInfo: { accountName: 'Ken Adams', accountNumber: '3401018190',
//          institution: 'PAGA', reference: 'inv-0001', amount: '5000.00', currency: 'NGN' } }
// momo → { id: '0d1f7a52-…' }
```

```bash
curl -X POST "$BLINX_BASE_URL/api/merchant/receive" \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG" \
  -d '{"amountIn":"5000.00","currencyIn":"NGN","amountOut":"3.66","rate":"1344.22","country":"NG","reference":"inv-0001","memo":"Invoice 0001","source":{"accountName":"Ken Adams","accountIdentifier":"1111111111","networkId":"5f1af11b-305f-4420-8fce-65ed2725a409"}}'
```

The body is a `ReceiveRequest`:

| Field        | Required | Description                                                                                       |
| ------------ | -------- | ------------------------------------------------------------------------------------------------- |
| `amountIn`   | yes      | What the sender pays, in `currencyIn` — from the quote.                                           |
| `currencyIn` | yes      | The sender's fiat (`country`'s currency).                                                         |
| `amountOut`  | yes      | What you receive, in USDC — from the quote.                                                       |
| `rate`       | yes      | The quote rate.                                                                                   |
| `source`     | yes      | `{ accountName, accountIdentifier, networkId }` — all three required.                             |
| `country`    | no       | The sender's country. Defaults to your merchant user's country.                                   |
| `reference`  | no       | Your unique reference; generated when omitted. Reusing a live one → `409 DB_409_DUPLICATE_ENTRY`.  |
| `memo`       | no       | Free text kept on the receive.                                                                    |

* The receive always settles in **USDC** to your vault address — a `currencyOut` in the body is ignored.
* Every invalid field is reported at once: `400 PAY_400_INVALID_REQUEST` with an `errors` array (e.g. `source.networkId`, `source.accountIdentifier`).
* **Keep the `id`** — you check the receive with it in step 6.

**5. The sender pays**

What happens next depends on the network's channel:

```js
if (receive.bankInfo) {
  // Bank: forward the deposit instructions to the sender — they make the transfer.
  await notifySender({
    payTo: receive.bankInfo.accountName,
    accountNumber: receive.bankInfo.accountNumber,
    bank: receive.bankInfo.institution,
    amount: `${receive.bankInfo.amount} ${receive.bankInfo.currency}`,
    reference: receive.bankInfo.reference,   // the sender must quote it on the transfer
  });
} else {
  // Mobile money: the provider sends the sender a collection prompt on their phone.
  await notifySender({ message: 'Approve the payment request on your phone.' });
}
```

`notifySender` is yours — email, SMS, your app. Blinx never contacts the sender for a bank collection.

**6. Check the receive**

Poll the status until it is terminal:

```js
const TERMINAL = ['settled', 'failed', 'rejected', 'expired', 'cancelled'];

async function waitForReceive(id, { everyMs = 30_000, timeoutMs = 24 * 3600_000 } = {}) {
  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    const { status } = await blinx('GET', `/api/merchant/receive/${id}/status`);
    if (TERMINAL.includes(status)) return status;
    await new Promise((r) => setTimeout(r, everyMs));
  }
  throw new Error('receive still in flight');
}

const status = await waitForReceive(receive.id);   // 'settled' when the funds reached your vault
```

```bash
curl "$BLINX_BASE_URL/api/merchant/receive/c83ab522-94a2-483a-9927-b8c393a38782/status" \
  -H "x-api-key: $KEY" -H "x-timestamp: $TS" -H "x-signature: $SIG"
# → { "data": { "status": "processing" } }
```

`GET /api/merchant/receive/{id}` returns the full receive (`type: "topup"`, amounts, `payerName`, `reference`, timestamps). An id that is not a receive of your tenant → `404 PAY_404_NOT_FOUND`. Receives also appear in `GET /api/merchant/payments`.

**…or get the status pushed (webhook)**

Instead of polling, register a webhook once for `PAYMENT_UPDATE`. Blinx POSTs every status change of your payments and receives to it:

```js
await blinx('POST', '/api/merchant/webhooks', {
  uri: 'https://merchant.example.com/blinx/webhooks',
  topics: ['PAYMENT_UPDATE'],
});
```

Each delivery is `{ topic, tenantId, data }`, signed with your **apiSecret** in `X-Blinx-Signature: t=<timestamp>,v1=<hex HMAC-SHA256 of "<timestamp>.<raw body>">`. Verify it, then answer **`202 Accepted`** — any other status counts as undelivered:

```js
const express = require('express');
const { createHmac, timingSafeEqual } = require('crypto');

const app = express();
app.post('/blinx/webhooks', express.raw({ type: 'application/json' }), (req, res) => {
  const header = req.get('X-Blinx-Signature') ?? '';
  const { t, v1 } = Object.fromEntries(header.split(',').map((kv) => kv.split('=')));
  const expected = createHmac('sha256', process.env.BLINX_API_SECRET)
    .update(`${t}.${req.body.toString('utf8')}`)
    .digest('hex');
  if (!v1 || v1.length !== expected.length || !timingSafeEqual(Buffer.from(v1), Buffer.from(expected))) {
    return res.sendStatus(401);
  }
  const event = JSON.parse(req.body.toString('utf8'));   // { topic, tenantId, data }
  // event.data carries the receive's id and status — update your records, then:
  res.sendStatus(202);
});
```

#### Errors

| Status | Code                           | When                                                                                   |
| ------ | ------------------------------ | -------------------------------------------------------------------------------------- |
| 400    | `PAY_400_INVALID_REQUEST`      | A bad field (every one listed in `errors`): amounts, `source`, unknown `networkId`, a mobile number on a bank network |
| 400    | `PAY_AMOUNT_OUT_OF_RANGE`      | No provider serves the amount (bounds in the message)                                  |
| 400    | `PAY_COUNTRY_NOT_SUPPORTED`    | Unknown country on the networks list                                                   |
| 403    | `GATE_403_KYB_REQUIRED` / `GATE_403_KYC_REQUIRED` | The tenant has not passed KYB / KYC                                 |
| 404    | `PAY_404_NOT_FOUND`            | The id is not a receive of your tenant                                                 |
| 409    | `DB_409_DUPLICATE_ENTRY`       | `reference` already used by a live or settled transfer                                 |
| 409    | `PAY_409_NO_SETTLEMENT_ASSET`  | Your vault has no active address to settle to                                          |
| 503    | `PAY_FAILED_QUOTE`             | No provider can quote right now — retry later                                          |

#### Notes

* **Quote first, then receive.** The receive is priced with the figures you send; take them from a fresh quote so the sender pays what the provider expects.
* **One reference per receive.** A reference frees up only once its transfer fails (`expired` / `rejected` / `failed` / `cancelled`).
* **Senders are saved as your recipients.** The `source` of a receive is stored as one of your recipients, so a returning sender is recognised.
