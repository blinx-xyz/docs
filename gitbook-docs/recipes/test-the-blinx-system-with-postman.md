# Test the Blinx system with Postman

{% hint style="info" %}
The full request/response contract is in the latest OpenAPI spec: [blinx-api.yaml](https://openapi.gitbook.com/o/qIJeTD9Swqwc0lE9nsEa/spec/blinx-api.yaml).
{% endhint %}

You can download the Blinx API Postman collection [here](../.gitbook/assets/Blinx-API.postman_collection.json).

## Setting up

In order to use the postman, you need to add some variables in the collection variables:

- `api_key`: this is your API key provided by blinx. You can get it from the Blinx Dashboard.
- `api_secret`: this is your API secret provided by blinx. You can get it from the Blinx Dashboard.

Every request is signed automatically by the collection's pre-request script — you only set the variables. See [Signing requests in Postman](../signing-requests-in-postman.md) for how it works.

### Other variables

| Variable | Default | Used by |
| --- | --- | --- |
| `sandbox_url` | `https://sandbox.api.blinx.xyz` | every request |
| `country` | `BW` | payment limits, country data |
| `receive_country` / `receive_currency` | `NG` / `NGN` | the receive requests — the sender's country and its currency |
| `receive_channel` | `bank` | which kind of network (`bank` or `momo`) the receive networks request picks |
| `send_country` / `send_currency` | `NG` / `NGN` | the send requests — the receiver's country and its currency |
| `send_channel` | `bank` | which kind of network the send networks request picks |

These are filled in by the requests themselves, so run them in order: `payment_id`, `payment_code`, `receive_amount_in` / `receive_amount_out` / `receive_rate`, `source_network_id`, `receive_id`, `send_amount_in` / `send_amount_out` / `send_rate`, `destination_network_id`, `send_id`, `webhook_id`.

## Requests

### payments

* **create payment** — `POST /api/merchant/payments`. Creates a payment link for a payer. The response carries `redirectUrl` (the page the payer opens) and `id`; the script saves them as `payment_code` and `payment_id` and shows a copy button in the *Visualize* tab. See [Create a Payment](create-a-payment.md).
* **get payments** — `GET /api/merchant/payments`. Lists your payments and receives, most recent first.
* **get payments csv** — `GET /api/merchant/payments?format=csv`. The same list as a CSV file.
* **get payment** — `GET /api/merchant/payments/{{payment_id}}`. One payment, with its line items, in any status.
* **get payment status** — `GET /api/merchant/payments/{{payment_id}}/status`. Only the payment's `status` — use it to poll.
* **get payment limits** — `GET /api/merchant/payments/limits/{{country}}`. The minimum and maximum amount a payment can have in a country.

### quote

* **quote** — `POST /api/merchant/quote`. The best available price for a currency pair and amount: send `amountIn` (or `amountOut`), `currencyIn`, `currencyOut` and `country`. Fiat → USD is a receive/payment quote; USD → fiat is a send quote.

### receive

Run these in order — each one uses what the previous one saved. See [Receive Funds](receive-funds.md).

* **quote** — `POST /api/merchant/quote`. What the sender pays in `receive_currency` against the USD you receive. The script saves `amountIn`, `amountOut` and `rate` for the **receive** request.
* **networks** — `GET /api/merchant/network/{{receive_country}}`. The banks and mobile-money networks a sender can pay from. The script saves the first network of `receive_channel` as `source_network_id`.
* **verification** — `POST /api/merchant/verification`. Checks the sender's account on `source_network_id` and returns the holder's name when known. In the sandbox, `1111111111` always verifies.
* **receive** — `POST /api/merchant/receive`. Creates the receive, with the **quote**'s `amountIn`, `amountOut` and `rate`. For a bank network the response carries `bankInfo` — the details to forward to the sender. The script saves `receive_id`.
* **get receive** — `GET /api/merchant/receive/{{receive_id}}`. The receive, in any status.
* **get receive status** — `GET /api/merchant/receive/{{receive_id}}/status`. Only its `status` — poll until `settled`.

### send

Run these in order — each one uses what the previous one saved. A send moves real (sandbox) funds out of your vault. See [Send Funds](send-funds.md).

* **balance** — `GET /api/merchant/balance`. What your vault can pay out, e.g. `{ "balance": "31.58", "currency": "usd" }`.
* **quote received** — `POST /api/merchant/quote` with `amountOut`: the amount the receiver must get, in `send_currency`; the quote tells you the USD that leaves your vault.
* **quote sent** — `POST /api/merchant/quote` with `amountIn`: the USD you send; the quote tells you what the receiver gets. Run **either** quote — the script saves `amountIn`, `amountOut` and `rate` for the **send** request.
* **networks** — `GET /api/merchant/network/{{send_country}}?side=send`. The banks and mobile-money networks a send can pay out to. The script saves the first network of `send_channel` as `destination_network_id`.
* **verification** — `POST /api/merchant/verification?side=send`. Checks the receiver's account on `destination_network_id`. In the sandbox, `1111111111` always verifies.
* **send** — `POST /api/merchant/send`. Creates the send with the quote's figures. `amountIn` must not exceed your balance, otherwise `402 PAY_402_INSUFFICIENT_FUNDS`. The script saves `send_id`.
* **get send** — `GET /api/merchant/send/{{send_id}}`. The send, in any status.
* **get send status** — `GET /api/merchant/send/{{send_id}}/status`. Only its `status` — poll until `settled`.

### webhooks

* **create webhook** — `POST /api/merchant/webhooks`. Registers your endpoint for `PAYMENT_UPDATE`, so status changes are pushed to you instead of polled. Replace the example `uri` with your own https URL. The script saves `webhook_id`.
* **get webhooks** — `GET /api/merchant/webhooks`. Your webhook subscriptions.
* **update webhook** — `PATCH /api/merchant/webhooks/{{webhook_id}}`. Changes a subscription's topics or turns it off (`active: false`).

### data

* **country data** — `GET /api/merchant/data/{{country}}`. A country's currency, phone format, transfer rules and payment limits. See [Country Data](country-data.md).
