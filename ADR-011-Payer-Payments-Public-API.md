# ADR-011: Payer Payments Public API  
  
**Date:** 2026-06-08  
**Status:** Proposed  


> [!WARNING] to be updated when we have a full set of requirements

---  
## Context  
  
Blinx exposes an external-facing REST API that allows tenants to accept inbound fiat payments from their customers. This ADR defines the **public API contract** for that surface: authentication, rate discovery, payment-method discovery, payer management, and payment lifecycle management.  
  
The API is consumed by tenant back-ends and mobile/web checkout flows. It is deliberately provider-agnostic — callers interact with Blinx resources only; the underlying collection infrastructure is an implementation detail.  
  
Auth follows the dual-layer model established in ADR-008 / ADR-009:  
  
1. **`x-api-key`** — GCP API Gateway key, identifies the tenant, must be present on every request.  
2. **Bearer JWT** — user-scoped access token issued via the auth endpoints below, must be present on all resource endpoints.  
  
---  
## Authentication Flows    
### Challenge-response (wallet signature)  

```
Client                          Service  
  |                                 |  
  |--GET /auth/message?address=---->|   provide EVM address  
  |<---{message}------------------- |   challenge tied to address  
  |                                 |  
  |--POST /auth/token-------------->|   {address, message, signature, chainId}  
  |   x-api-key: <tenant key>       |  
  |<---{token, refreshToken, ...}---|
```
  
### Token rotation  
```  
Client                          Service  
  |                                 |  
  |--POST /auth/refresh------------>|   {refreshToken}  
  |   x-api-key: <tenant key>       |  
  |<---{token, refreshToken, ...}---|
```  
  
Both tokens are rotated on every refresh call. See ADR-004 for payload structure and expiry rules.  
  
---  
## Resource Model  
  
```  
tenants  
  └─ payers          — registered senders that initiate payments to a tenant       
	  └─ payments   — individual inbound payment requests / instructions
```  
  
A `payer` is a named entity (person or organisation) associated with a tenant. One payer may create many payments over time. Payer identity is supplied by the tenant on first payment and reused across subsequent requests via `payerId`.  
  
A `payment` represents a single inbound transfer request. It carries payment instructions (bank-account details or mobile-money prompt) that the payer uses to complete the transfer. Status flows: `CREATED → PENDING → PROCESSING → COMPLETED | FAILED | EXPIRED`.  
  
---  
## OpenAPI Specification  
  
The paths are summarised below.    
### Auth (x-api-key only — no Bearer required)  
  
| Method | Path | Purpose |  
|--------|------|---------|  
| `GET` | `/auth/message` | Fetch a sign-in challenge for a wallet address |  
| `POST` | `/auth/token` | Exchange a signed challenge for access + refresh tokens |  
| `POST` | `/auth/refresh` | Rotate a valid refresh token into a new token pair |  
  
### Rates & Discovery (x-api-key + Bearer)  
  
| Method | Path | Purpose |  
|--------|------|---------|  
| `GET` | `/rates` | Exchange rates and supported countries |  
| `GET` | `/payment-ramps` | Payment methods available per country |  
  
### Payers (x-api-key + Bearer)  
  
| Method | Path | Purpose |  
|--------|------|---------|  
| `GET` | `/payers` | Paginated payer list; filter by `payerId`, `payerName` |  
| `GET` | `/payers/{id}` | Fetch a single payer record |  
  
### Payments (x-api-key + Bearer)  
  
| Method | Path | Purpose |  
|--------|------|---------|  
| `POST` | `/payments` | Create a receive-payment request, returns payment instructions |  
| `GET` | `/payments` | Paginated payment list with filters and sorting |  
| `DELETE` | `/payments/{id}` | Cancel and remove a payment (CREATED or PENDING only) |  
  
---  
## Key Design Decisions  
  
| Decision                                         | Rationale                                                                                                                                                                                                            |     |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| Dual-auth on every request                       | Matches ADR-008/009: API key identifies the tenant; JWT identifies the user. A stolen JWT alone cannot reach the API — the tenant key acts as a second factor.                                                       |     |
| `x-api-key` on auth endpoints too                | Tenant scoping must be enforced even during login, so the JWT is issued with the correct `tenantId` claim (ADR-009 Option 3).                                                                                        |     |
| Challenge-response login (wallet signature)      | Consistent with the existing auth model (ADR-004). The frontend already handles this flow.                                                                                                                           |     |
| `payers` as a first-class resource               | Separating payer identity from payment data avoids duplicating customer fields on every payment record and enables cross-payment analytics per sender.                                                               |     |
| `payerName` filter on payments via JOIN          | Filtering payments by name is more ergonomic for dashboards than requiring callers to resolve payer IDs first. The query joins `payments → payers` on `tenant_id + payer_id`.                                        |     |
| `reference` as idempotency key on POST /payments | Safe to retry on network failure. Matches the Unlimit/CardPay convention. Duplicate references return `409 Conflict` with the existing payment record.                                                               |     |
| Delete restricted to `CREATED` / `PENDING`       | Payments that have reached `PROCESSING` have been forwarded to the collection provider; reversal is outside the scope of this API.                                                                                   |     |
| Sorting via `sortBy` + `sortOrder` query params  | Two independent parameters are easier to document and validate than a combined `sort=createdAt:desc` string.                                                                                                         |     |
| Provider-agnostic response shapes                | No upstream-provider identifiers in the public response body. Internal `collectionId` is stored in the DB but not surfaced to callers, allowing the underlying provider to be swapped without a breaking API change. |     |
  
---  
  
## Implementation Checklist  
  
- [ ] `GET /auth/message` — existing route; verify `x-api-key` middleware is applied  
- [ ] `POST /auth/token` — existing route; verify `x-api-key` middleware is applied  
- [ ] `POST /auth/refresh` — existing route; verify `x-api-key` middleware is applied  
- [ ] `GET /rates` — new route; calls collection provider for rates + country list  
- [ ] `GET /payment-ramps` — new route; calls collection provider for available channels  
- [ ] `src/services/payer/` — Payer entity, repository, service (CRUD)  
- [ ] Migration: `payers` table (`id`, `tenant_id`, `name`, `email`, `phone`, `country`, `created_at`, `updated_at`)  
- [ ] `GET /payers` — paginated list with `payerId` / `payerName` filters  
- [ ] `GET /payers/{id}` — single payer fetch  
- [ ] `POST /payments` — create receive-payment; upsert payer on `reference`  
- [ ] `GET /payments` — paginated list with all filters + sort  
- [ ] `DELETE /payments/{id}` — status guard + soft-delete or hard-delete TBD  
- [ ] `src/dtos/payer.ts` — request/response DTOs  
- [ ] `src/dtos/payment.ts` — request/response DTOs  
- [ ] `src/controllers/payer.ts` / `src/controllers/payment.ts`  
- [ ] `src/routes/payer.ts` / `src/routes/payment.ts`  
- [ ] Wire routes in `src/app.ts` behind `resolveTenant` + `authenticateUser`  
- [ ] Tests: `test/services/payer/` + `test/services/payment/`  
  
---  
  
## References  
  
- ADR-004 — JWT Authentication Flow  
- ADR-008 — API Gateway Key Auth  
- ADR-009 — Multi-Tenant System  
- ADR-010 — Receive Payment Integration  
- [Unlimit (CardPay) API Reference](https://integration.unlimit.com/api-reference)
