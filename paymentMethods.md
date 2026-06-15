# Paywall payment methods & Stripe flow

Reference for how Stripe payment methods are configured, which APIs are called, and how the live paywall works.

> This is the focused deep-dive on **which payment methods show where** and the live paywall checkout sequence. For the whole Stripe integration (per-site keys, webhooks, cancellation/refunds, lifecycle automation), see [stripe.md](./stripe.md).

**Data-source legend** (used throughout): 🗄️ **DB** = our MySQL/Prisma (merchant config we store) · 💳 **Stripe** = lives in Stripe, fetched live · 🌐 **Client** = sent by the browser at request time. See [stripe.md → Who owns what](./stripe.md#who-owns-what--the-full-data-map) for the full map.

> Core idea for this doc: **the list of methods is 💳 Stripe's (PMC); which of them are turned on is 🗄️ our DB (paywall toggles); a method is live only when both agree.**

---

## Overview

Each paywall can enable payment methods per merchant. Methods come from **Stripe’s Payment Method Configuration (PMC)** — we don’t hardcode a list. At runtime the paywall uses **two Stripe checkout modes**:

| Step | URL hash | What the user sees | Stripe mode |
|------|----------|-------------------|-------------|
| **Plan step** | `#paywall` | Plan cards + wallet buttons + Continue | `ui_mode: "elements"` (Express Checkout) |
| **Checkout step** | `#paywall-checkout` | Stripe Embedded Checkout iframe | `ui_mode: "embedded_page"` |

**🗄️ DB fields on `paywall`** (our config — what the merchant toggled):
- `express_checkout_enabled` — master toggle for wallet buttons on plan step
- `enabled_payment_methods` — JSON map of PMC key → `true` / `false`

**💳 From Stripe** (not stored by us, read live):
- The PMC itself — which methods exist and are enabled in the merchant's Stripe account
- Prices/amounts and the Checkout Session `clientSecret`

---

## Express vs Embedded — which methods go where

Methods are split into two sections based on what Stripe supports:

| Section | PMC keys | Where shown | Stripe API |
|---------|----------|-------------|------------|
| **Express** | `link`, `apple_pay`, `google_pay`, `amazon_pay`, `paypal` | Plan step wallet buttons | `ExpressCheckoutElement` (Stripe.js) |
| **Embedded** | Everything else (card, Klarna, iDEAL, ACH, etc.) | Checkout step iframe | Embedded Checkout `payment_method_types` |

A method is **active** on the live paywall only when:
1. 💳 Stripe PMC has it enabled (`available` + `display_preference: on`), **and**
2. 🗄️ Paywall toggle is on (`enabled_payment_methods[id] === true`).

If no saved config exists yet, all Stripe-enabled methods default to **on**. (So: 💳 Stripe proposes, 🗄️ our DB disposes.)

**Special mappings** (handled in [stripe-payment-methods.util.ts](../backend/src/paywalls/stripe-payment-methods.util.ts)):
- `apple_pay` / `google_pay` → checkout session type `card`
- `amazon_pay` → Express preference `"auto"` (Stripe decides when to show it; not forced like other wallets)

---

## ⚠️ Which PMC? — the `is_default` one (currently RevenueCat's)

Our code never stores a PMC id. It lists all PMCs and picks **whichever has `is_default: true`** (`fetchDefaultPaymentMethodConfiguration`). `is_default` is a flag on the PMC object, **not** the dashboard's "Your account" / "RevenueCat account" grouping.

**Verified against the live account:** the default is the **"Default" under _RevenueCat account_** (`pmc_1TKFip…RL001`), *not* "Your account / Default". RevenueCat sets one of its configs as the account default when Stripe is connected, and our code follows that flag.

➡️ **So enable/disable methods in the RevenueCat-account "Default" PMC** — changes in "Your account / Default" or the standalone `apple pay`/`amazon pay` PMCs do **nothing** for our app. This is fragile (nothing pinned); see [stripe.md → Which PMC do we read?](./stripe.md#which-pmc-do-we-read--the-is_default-one-currently-revenuecats) for the full caveat + verification command.

## Dashboard flow (merchant setup)

```
Stripe Dashboard (enable methods on the DEFAULT PMC = RevenueCat's, pmc_1TKFip…)
        │
        ▼
GET /api/paywalls/:id/payment-methods
  → Fetches the is_default Stripe PMC (RevenueCat's) + merges saved paywall toggles
  → Returns { expressCheckoutEnabled, express: [...], embedded: [...] }
        │
        ▼
PaymentMethodsEditor (PaywallEditorClient)
  → Merchant toggles Express section + individual methods
        │
        ▼
PUT /api/paywalls/:id (save paywall)
  → Saves express_checkout_enabled + enabled_payment_methods to DB
```

| API | Purpose | Code |
|-----|---------|------|
| `GET /api/paywalls/:id/payment-methods` | Sync methods from Stripe PMC for editor | [paywalls.controller.ts](../backend/src/paywalls/paywalls.controller.ts) → [paywalls.service.ts](../backend/src/paywalls/paywalls.service.ts) |
| `PUT /api/paywalls/:id` | Save paywall toggles | [paywalls.controller.ts](../backend/src/paywalls/paywalls.controller.ts) |

---

## Live paywall flow (end user)

### 1. User reaches paywall (`#paywall`)

**Frontend:** [PaywallScreen.tsx](../frontend/src/components/survey/PaywallScreen.tsx)

```
GET /api/flows/:id/prices
  → Returns plan options + expressCheckoutEnabled + expressMethodIds + embeddedMethodIds
        │
        ▼ (if Express wallets enabled)
POST /api/flows/:id/checkout  × N plans (parallel prefetch)
  body: { priceId, userId, checkoutUi: "elements" }
  → Returns { clientSecret } per plan
        │
        ▼
Stripe.js loads → ExpressCheckoutElement mounts per plan (pre-warmed, stacked in DOM)
  → Wallet buttons render (Apple Pay, Amazon Pay, etc.)
        │
        ▼
User taps wallet → Stripe confirms subscription inline
  OR taps Continue → go to step 2
```

| API | When | Body | Returns | Backend |
|-----|------|------|---------|---------|
| `GET /api/flows/:id/prices` | Paywall mount | — | `options` (💳 Stripe prices), `trialDaysByPriceId` (🗄️ DB), `expressCheckoutEnabled` + `expressMethodIds` + `embeddedMethodIds` (🗄️ DB toggles ∩ 💳 PMC) | `flows.service.ts` → `prices()` |
| `POST /api/flows/:id/checkout` | Wallet prefetch (per plan) + embedded checkout | `{ priceId` 💳`, userId` 🗄️`, checkoutUi: "elements" }` | `{ clientSecret }` (💳 Stripe) | `flows.service.ts` → `checkout()` creates Stripe session with `ui_mode: "elements"` |

**Wallet UX notes:**
- One Stripe checkout session (and `clientSecret`) is created **per plan** for wallets.
- All plan wallet panels mount in the background before the paywall is revealed (avoids flicker on plan switch).
- Switching plans does **not** call the API again if secrets were prefetched — it only swaps which pre-mounted Stripe panel is visible.

### 2. User taps Continue (`#paywall-checkout`)

```
POST /api/flows/:id/checkout
  body: { priceId, userId, checkoutUi: "embedded", utm/meta/amplitude fields... }
  → Returns { clientSecret }
        │
        ▼
EmbeddedCheckoutProvider + EmbeddedCheckout (Stripe iframe)
  → Shows embedded methods (card, Klarna, etc.) with paywall branding
```

| API | When | Body | Stripe session |
|-----|------|------|----------------|
| `POST /api/flows/:id/checkout` | User opens checkout step | `{ checkoutUi: "embedded" }` | `ui_mode: "embedded_page"` + `payment_method_types` from embedded methods + branding colors/font |

### 3. After payment

```
Stripe redirects to /start?flow_id=…&payment={CHECKOUT_SESSION_ID}
        │
        ▼
POST /api/flows/:id/payment-confirm
  body: { sessionId }
  → Verifies session, provisions subscription (RevenueCat), fires analytics
        │
        ▼ (optional)
POST /api/flows/:id/payment-confirm-email
  → Captures email if missing from wallet payment
```

| API | Purpose |
|-----|---------|
| `POST /api/flows/:id/payment-confirm` | Confirm payment + provision access |
| `POST /api/flows/:id/payment-confirm-email` | Backfill email after wallet checkout |

---

## What happens inside `POST /checkout` (backend)

`flows.service.ts` → `checkout()`:

1. Resolve active paywall for the flow — 🗄️ DB (`Flow` → `Paywall`).
2. Load PMC (💳 Stripe) + `enabled_payment_methods` (🗄️ DB) → intersect → split into `expressMethodIds` / `embeddedMethodIds`.
3. Create Stripe Checkout Session (💳 Stripe), passing:
   - **elements:** `ui_mode: "elements"`, `payment_method_types` from express methods, subscription line item for `priceId` (💳).
   - **embedded:** `ui_mode: "embedded_page"`, `payment_method_types` from embedded methods, plus `branding_settings` (🗄️ colors/font from paywall/template).
   - `subscription_data.metadata` ← 🗄️ `flow_id`/`app_user_id` + 🌐 UTM/Meta/Amplitude (see [stripe.md Step 2](./stripe.md)).
4. Return `clientSecret` (💳 Stripe) to frontend.

Both modes use `mode: "subscription"` with trial metadata (🗄️ DB) when configured.

---

## Why wallet buttons can differ per plan

Your app sends the **same** `expressMethodIds` to every plan. Buttons can still differ because:

- Each plan gets its **own Stripe checkout session** (different price, amount, billing interval).
- Stripe’s `ExpressCheckoutElement` reports `availablePaymentMethods` per session — wallets have eligibility rules (amount, currency, device, subscription type).
- `amazon_pay` uses `"auto"` — Stripe may show/hide it per session.

This is expected Stripe behavior, not different paywall config per plan.

---

## Delay on wallet buttons — what causes it

There is **no API that returns wallet buttons**. The sequence is:

1. **API:** `GET /prices` then `POST /checkout` (per plan) → get `clientSecret`s.
2. **Client:** Stripe.js loads + `ExpressCheckoutElement` initializes per plan.
3. **Stripe:** Decides which wallets to show and renders buttons.

The spinner on first load waits for all plan wallet panels to finish Stripe init before revealing plans + buttons together.

---

## Key files

| Area | File |
|------|------|
| PMC discovery, express/embedded split, checkout type mapping | [backend/src/paywalls/stripe-payment-methods.util.ts](../backend/src/paywalls/stripe-payment-methods.util.ts) |
| Dashboard payment-methods API | [backend/src/paywalls/paywalls.controller.ts](../backend/src/paywalls/paywalls.controller.ts) |
| Live prices + checkout + payment confirm | [backend/src/flows/flows.service.ts](../backend/src/flows/flows.service.ts) |
| Live API routes | [backend/src/flows/flows.controller.ts](../backend/src/flows/flows.controller.ts) |
| Dashboard editor UI | [frontend/src/components/paywall/PaymentMethodsEditor.tsx](../frontend/src/components/paywall/PaymentMethodsEditor.tsx) |
| Live paywall orchestration | [frontend/src/components/survey/PaywallScreen.tsx](../frontend/src/components/survey/PaywallScreen.tsx) |
| Wallet buttons (Express Checkout) | [frontend/src/components/survey/PaywallCheckoutPanel.tsx](../frontend/src/components/survey/PaywallCheckoutPanel.tsx) |
| Frontend payment-method helpers | [frontend/src/lib/payment-methods.ts](../frontend/src/lib/payment-methods.ts) |

---

## Adding a new Stripe method

**Usually no code change:**
1. Enable method in Stripe Dashboard.
2. Sync in paywall editor (`GET /payment-methods`).
3. Toggle on for the paywall and save.

**Code change only if** Stripe adds a new Express Checkout wallet (rare) — add its PMC key to `EXPRESS_CHECKOUT_PMC_KEYS` in [stripe-payment-methods.util.ts](../backend/src/paywalls/stripe-payment-methods.util.ts) (and frontend mirror in [payment-methods.ts](../frontend/src/lib/payment-methods.ts)).

---

## Quick reference — APIs

| API | Who calls it | Reads from | Writes to |
|-----|--------------|------------|-----------|
| `GET /api/paywalls/:id/payment-methods` | Dashboard editor | 💳 Stripe PMC + 🗄️ saved toggles | — |
| `PUT /api/paywalls/:id` | Dashboard editor | — | 🗄️ DB (toggles) |
| `GET /api/flows/:id/prices` | `PaywallScreen` on load | 🗄️ paywall config + 💳 Stripe prices/PMC | — |
| `POST /api/flows/:id/checkout` (`elements`) | `PaywallScreen` wallet prefetch | 🗄️ paywall + 🌐 client meta | 💳 Stripe (creates Elements session) |
| `POST /api/flows/:id/checkout` (`embedded`) | `PaywallScreen` checkout step | 🗄️ paywall + 🌐 client meta | 💳 Stripe (creates Embedded session) |
| `POST /api/flows/:id/payment-confirm` | `StartFlowHost` after redirect | 🌐 session id → 💳 Stripe session/subscription | 🔌 RevenueCat / Meta / email |
| `POST /api/flows/:id/payment-confirm-email` | `PostPaymentEmailCapture` | 🌐 typed email | 💳 Stripe customer + 🔌 email/CRM |
