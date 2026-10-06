# Payers

**Route:** `/payers`

The Payers page lists the payers your tenant works with — the recipients recorded against your account — so you can review and search them. It is read-only.

### The payers list

Each entry shows the payer's recorded details — name/label, institution or network, country, and **type** (`personal` or `external`). The list is **paginated**: a page at a time (5 by default, up to 50 per page), with a control to load the next page.

### Filtering

You can narrow the list by:

* **Type** — `personal` or `external`.
* **Country** — the payer's country.
* **Scope** — narrows to a particular grouping of payers.

Backed by `GET /api/dashboard/tenant/payers` (scoped to your tenant; `type`, `country`, `scope`, `limit`, and `offset` query parameters). The response carries the page of payers plus a `total` count and the `next` offset (or `null` when you have reached the end).
