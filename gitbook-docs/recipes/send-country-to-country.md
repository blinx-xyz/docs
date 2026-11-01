# Send and receive funds country to country

{% hint style="info" %}
The full request/response contract is in the latest OpenAPI spec: [blinx-api.yaml](https://openapi.gitbook.com/o/qIJeTD9Swqwc0lE9nsEa/spec/blinx-api.yaml).
{% endhint %}

### Move money from a sender in one country to a receiver in another

Blinx has no single "country to country" call: you chain a **receive** and a **send** through your vault. A sender pays you in their local currency, the funds settle in your vault as **USDC**, and you pay them out to a receiver in another country, in their local currency.

```
Receive   sender (e.g. Nigeria, NGN)  ──pays──►  your vault (USDC)
              │
              ▼  wait until the receive is settled
Send      your vault (USDC)  ──pays out──►  receiver (e.g. Kenya, KES)
```

#### Main flow

1. **Quote both legs.** Quote the receive (`NGN → USD`, `side: receive`) and the send (`USD → KES`, `side: send`) with `POST /api/merchant/quote`. Chain the amounts: the send's `amountIn` can be at most the receive's `amountOut` (the USD it puts in your vault). → [Receive Funds, step 1](receive-funds.md) · [Send Funds, step 2](send-funds.md)
2. **Receive from the sender.** Pick and verify the sender's network (`GET /network/NG`, `POST /verification`), create the receive (`POST /receive`), and forward the `bankInfo` to the sender (or let the provider collect a mobile-money payment). → [Receive Funds, steps 2–5](receive-funds.md)
3. **Wait for the funds.** Poll `GET /receive/{id}/status` — or listen to a `PAYMENT_UPDATE` webhook — until the receive is `settled`. Only a settled receive is in your vault. → [Receive Funds, step 6](receive-funds.md)
4. **Check your balance.** `GET /balance` must cover the send's `amountIn`; otherwise the send is refused with `402 PAY_402_INSUFFICIENT_FUNDS`. → [Send Funds, step 1](send-funds.md)
5. **Send to the receiver.** Pick and verify the receiver's payout network (`GET /network/KE?side=send`, `POST /verification?side=send`) and create the send (`POST /send`) with a fresh quote's figures. → [Send Funds, steps 3–5](send-funds.md)
6. **Follow the send.** Poll `GET /send/{id}/status`, or use the same `PAYMENT_UPDATE` webhook, until it is `settled`. → [Send Funds, step 7](send-funds.md)

#### Things to keep in mind

* **Two transfers, two references.** The receive and the send are separate transfers with their own `id`, `reference` and status; keep both to reconcile the end-to-end movement.
* **Rates move between the legs.** The receive settles at its quote, the send pays out at a fresh quote taken when you create it — re-quote the send once the receive has settled.
* **Fees apply on both legs.** Each leg's provider fee (`localFee`) is taken in that leg's fiat; the USD left over after the receive is what the send can pay out.
* **Your balance is shared.** The vault holds every receive you have not sent on yet; the send only checks that the balance covers `amountIn`, not which receive the funds came from.
* **One webhook covers both.** A single `PAYMENT_UPDATE` webhook delivers the status changes of both the receive and the send — tell them apart by `data.id` (or `data.type`: `topup` for the receive, `send` for the send).
