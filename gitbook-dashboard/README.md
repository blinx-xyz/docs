# Blinx Merchant Dashboard

The merchant dashboard is the merchant-facing area of the Blinx admin dashboard. It lets a seller request payments from payers, review incoming payments, and manage the payers they work with — all scoped to the seller's own tenant.

### Who this is for

A **seller** is a merchant-tier user tied to a single **tenant** (your organisation). After signing in, every page you see and every request the dashboard makes is limited to your tenant's data. You do not see other tenants or platform-wide administration.

### Signing in

Access is passwordless, via a one-time code sent to your email:

1. Enter your email address.
2. Blinx emails you a one-time verification code (OTP).
3. Enter the code to sign in.

Your session stays active across page reloads and refreshes automatically. Use **Sign out** from the account menu (top-right) to end the session.

### Pages available to a seller

| Page                    | Route               | What it's for                                              |
| ----------------------- | ------------------- | ---------------------------------------------------------- |
| Home / Tenant Dashboard | `/tenant/dashboard` | Overview of your tenant, API credentials, and users        |
| Payers                  | `/payers`           | Review the payers (recipients) linked to your tenant       |
| Payments In             | `/payments-in`      | Review incoming payments and export them                   |
| New Payment             | `/payments-in/new`  | Create a payment request link for a payer                  |
| Webhooks                | `/webhooks`         | Subscribe your endpoints to payment/transfer events        |

When you sign in, the dashboard opens on your **Tenant Dashboard**.
