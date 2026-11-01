# Blinx Merchant Dashboard

The merchant dashboard is the merchant-facing area of the Blinx admin dashboard. It lets a seller collect payments, send payouts, follow money movement, manage their vault, and configure integrations — all scoped to the seller's own tenant.

### Who this is for

A **seller** is a merchant-tier user tied to a single **tenant** (your organisation). After signing in, every page you see and every request the dashboard makes is limited to your tenant's data. You do not see other tenants or platform-wide administration.

### Signing in

Access is passwordless, via a one-time code sent to your email:

1. Enter your email address.
2. Blinx emails you a one-time verification code (OTP).
3. Enter the code to sign in.

Your session stays active across page reloads and refreshes automatically. Use **Sign out** from the account menu (top-right) to end the session.

### The workspace

Every page sits inside the same frame:

* **Sidebar** (left) — the main navigation, with a utility group pinned at the bottom.
* **Header** (top) — a menu toggle, your **tenant name**, a **SANDBOX** badge when you're in the sandbox environment, and your account menu (with **Sign out**) on the right.

The sidebar groups the pages like this:

| Section | Page | Route | What it's for |
|---|---|---|---|
| Main | **Overview** | `/tenant/overview` | Available funds, 7-day money movement, and the items that need attention |
| Main | **Payments** | `/tenant/payments` | One list of every incoming payment and outgoing payout |
| Main | **Vault** | `/tenant/vault` | Vault assets, balances, and vault-transfer history |
| Main | **Contacts** | `/tenant/payers` | Reusable payer and recipient accounts |
| Main | **Integrations** | `/tenant/webhooks` | Webhook endpoints and delivery health |
| Settings | **General** | `/tenant/settings` | Business name, settlement destination, readiness, API secret |
| Settings | **Users** | `/tenant/settings/users` | The people who use this tenant and their verification state |
| Utility | **Service status** | `/tenant/healthcheck` | Service connectivity and version |

The money-moving flows are reached from buttons on those pages rather than from the sidebar:

| Flow | Route | Reached from |
|---|---|---|
| **Create payment link** | `/tenant/payments/new` | Overview, Payments |
| **Receive by bank transfer** | `/tenant/receive` | Overview, Payments |
| **Send payout** | `/tenant/send` | Overview, Payments, a contact's **Send payout** action |

> **Gating.** Collecting and paying out require an **approved KYB** and at least one **KYC-approved user**. Until both are in place, the money-moving buttons are disabled and a banner explains what's missing (see **Settings — General › Business readiness**).

When you sign in, the dashboard opens on your **Overview**.
