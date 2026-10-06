# Test the Blinx system with Postman

A step-by-step walkthrough for a **merchant** to exercise the whole flow — onboarding and payments — from Postman, end to end. It assumes you have **not** used Postman much before, so the first section gets you set up; the rest is a checklist you can follow top to bottom.

Everything here drives the ready-made collection `Blinx Merchant API.postman_collection.json` in this folder. You do **not** need to build requests by hand or write any code.

For deeper detail on individual topics, see the companion docs: create & check a payment · request signing · the Postman pre-request script · OpenAPI spec.

***

#### 0. One-time Postman setup

1. **Install Postman** — download it from postman.com and open the desktop app (a free account is enough; you can skip sign-in for local testing).
2. **Download Postman collection** above
3. **Import the collection** — click **Import** (top-left), drag in `Blinx Merchant API.postman_collection.json`, and confirm. A collection named **“Blinx Merchant API”** appears in the left sidebar with a **payments** folder (plus a top-level **healthcheck** request).
4. **Set the base URL** — the collection includes a **`sandbox_url`** variable for the API base. On the collection's **Variables** tab, set `sandbox_url` to the environment you're testing against — the Blinx **sandbox** URL you were given, or `http://localhost:3001` for a service running locally — and **Save**. Make sure that service is reachable before you send anything.
5. **Test the connection** — run the top-level **healthcheck (test sandbox)** request (`GET {{sandbox_url}}/healthcheck`). A **200** confirms `sandbox_url` is correct and the service is up. This request needs no API key — it's the quickest way to verify your setup before onboarding.

**How authentication works (you don't sign anything by hand)**

Merchant endpoints are protected by a signed-request scheme (`authenticateSeller`). The collection already contains a **collection-level Pre-request Script** that, before every request, computes an HMAC-SHA256 signature and injects the `x-api-key`, `x-timestamp`, and `x-signature` headers for you. You only have to supply your key and secret (next step). If you're curious how it works, read postman-pre-request.md.

> Because timestamps are signed, your machine's clock must be within **±5 minutes** of the server's, or requests are rejected as replays.

***

#### 1. Onboarding — set your API key & secret

Blinx issues each merchant an **API key** and an **API secret**.

1. In the sidebar, click the **Blinx Merchant API** collection → **Variables** tab.
2. Set the **current value** of:
   * `api_key` — your API key (sent as `x-api-key`).
   * `api_secret` — your API secret (used only to sign; **never sent**).
3. Click **Save**.

`payment_id` is **not** filled in automatically — when a later step needs it, copy the `id` from the response into that collection variable (Variables tab → set the current value → **Save**).

> Tip: keep the secret in the collection/environment variable only. Never paste it into a request body or URL.

***

#### 2. Payment — create a request and take the payer through it

Open the **payments** folder. A payment is a request for a **payer** (your buyer) to pay you; you hand them a hosted URL and then poll for the outcome.

**2a. Create the payment**

1. **create payment** — `POST /api/merchant/payments` Body sends `amount`, `fiat`, `token`, `country`, `type`, `reference`, `purpose`, `validUntil`. **Check:** `201` and the response contains a **`redirectUrl`** (the hosted payment page). Copy the returned payment `id` into the `payment_id` variable so _get payment_ / _peek payment_ can use it.
2. Copy the **`redirectUrl`** and open it in a browser — this is the payer's view.

**2b. Payer denies the request**

1. On the payment page, the payer **rejects** the request.
2. Back in Postman, run **get payments** (`GET /api/merchant/payments`). **Check:** the payment's **status is `rejected`**.

**2c. Payer happy path (settled)**

1. On the payment page, the payer **accepts**.
2. They fill in payout details: choose **bank or mobile** and use **`1111111111`** as the account identifier (this is the sandbox “success” number), then **Continue**.
3. In Postman, run **get payments** repeatedly. **Check:** the status advances and ends at **`settled`**.

**2d. Payer unhappy path (failed)**

1. On the payment page, the payer **accepts**.
2. They fill in payout details: choose **bank or mobile** and use **`0000000000`** as the account identifier (the sandbox “failure” number), then **Continue**.
3. In Postman, run **get payments** repeatedly. **Check:** the status ends at **`failed`**.

> You can also inspect a single payment with **get payment** (`GET /api/merchant/payments/{{payment_id}}`), or preview the payer's view with **peek payment** (`GET /api/payer/payment/{{payment_id}}`). Payment statuses progress through `created → waiting → … → settled | rejected | failed`. More in create-and-check-payment.md.

***

#### 3. Payments reports

From the **payments** folder:

1. **get payments** — `GET /api/merchant/payments` → returns **JSON** (the default).
2. **get payments csv** — `GET /api/merchant/payments?format=csv` → returns a **CSV** download (`Content-Type: text/csv`). In Postman, the CSV body shows in the response pane; use **Save Response → Save to a file** to download it.

***

#### Troubleshooting

* **401 Unauthorized** — check `api_key`/`api_secret` are set on the collection **Variables** tab (current value) and saved; check your machine clock is within ±5 minutes of the server; confirm `sandbox_url` points at a reachable service (e.g. `http://localhost:3001` for a local run).
* **A `{{...}}` variable is empty** — for `payment_id`, copy the `id` from the creating request's response into the variable before using it.
* **Payment won't create** — the `(country, fiat)` corridor may be unsupported or the amount outside its limits; see create-and-check-payment.md.

_Updated: 19 September 2026 12:22_
