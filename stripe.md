# Stripe — End-to-End Reference

How Stripe is wired into the Adaptive Survey Platform: what we use from Stripe, how a payment flows through our code, the webhooks we listen to, and what is possible to build on top.

> Scope: This doc covers the **whole** Stripe integration — checkout, subscriptions, wallets, webhooks, cancellation/refunds, lifecycle automation, and the per-site (white-label) key model. For the narrower "which payment methods show where" question, see [payment-methods.md](./payment-methods.md).

### Data-source legend

Throughout this doc, every piece of data is tagged with where it comes from:

| Tag | Source | Meaning |
|-----|--------|---------|
| 🗄️ **DB** | Our MySQL (Prisma) | Stored by us — `Site`, `Paywall`, `Flow`, `Response`, etc. Configured by the merchant in our admin. |
| 💳 **Stripe** | Stripe API | Lives in Stripe's account, fetched live or sent in webhooks. We don't store it (prices, sessions, subscriptions, customers). |
| 🌐 **Client** | Browser / request | Sent by the respondent's browser at request time — `X-Public-Host` header, UTM/cookies, the session id on redirect. Not persisted unless we copy it. |
| 🔌 **External** | Other 3rd party | RevenueCat, Meta CAPI, Amplitude, SMTP/ActiveCampaign — downstream of a Stripe event. |

> The recurring theme: **we store *pointers* (Stripe IDs, keys) in our DB; the *money objects* (prices, sessions, subscriptions) live in Stripe.** A paywall row holds `stripeProductId`/`stripePriceId` 🗄️, but the actual price amount/currency/interval is read from Stripe 💳 every time.

---

## 1. What we use from Stripe

| Stripe capability | What we use it for | SDK / API surface |
|-------------------|--------------------|--------------------|
| **Checkout Sessions** | Every purchase. Two UI modes: Embedded Checkout iframe and Express Checkout (wallets) | `stripe.checkout.sessions.create / .retrieve / .listLineItems` |
| **Subscriptions** | All purchases are `mode: "subscription"` — we don't sell one-time products | `stripe.subscriptions.retrieve / .search / .update / .cancel` |
| **Prices & Products** | Plan options shown on the paywall are read live from a Stripe Product (multiple prices) or a single Price | `stripe.prices.list / .retrieve` |
| **Customers** | Email fallback for wallet payments; cancellation lookup | `stripe.customers.retrieve` |
| **Payment Method Configuration (PMC)** | Source of truth for which payment methods a merchant has enabled. We never hardcode the method list | `stripe.paymentMethodConfigurations.*` (via `fetchDefaultPaymentMethodConfiguration`) |
| **Webhooks (outbound→us)** | Provision/revoke access (global) + lifecycle analytics (per-site) | `stripe.webhooks.constructEvent`, `stripe.webhookEndpoints.create` |
| **Refunds & Invoices** | Refund-on-cancel within a configurable window | `stripe.invoicePayments.list`, `stripe.refunds.create`, `stripe.invoices` |
| **Branding settings** | Embedded Checkout iframe colors/font derived from the template | `branding_settings` on session create |
| **Trials** | Per-price free-trial days via `subscription_data.trial_period_days` | session create |

SDK version: `stripe@^21` (server-side, backend) and `@stripe/stripe-js` + `@stripe/react-stripe-js` (browser, Express Checkout + Embedded Checkout components).

---

## 2. The per-site key model (white-label)

This is the single most important architectural fact about Stripe here: **Stripe keys are resolved per-site, not globally.**

The `Site` model holds:
- `stripeSecretKey` — server key used to create/retrieve sessions
- `stripePublishableKey` — returned to the browser to init Stripe.js
- `stripeInboundWebhookUuid` — unique path segment for that site's inbound webhook URL
- `stripeInboundWebhookSigningSecret` (`whsec_…`) — verifies that site's inbound webhooks
- `stripeProductId` / `stripePriceId` live on the **paywall**, not the site

Environment variables (`STRIPE_SECRET_KEY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`) are **global fallbacks only**. Production tenants each run their own Stripe account on one deployment.

**How a request finds the right key:** public endpoints carry no auth. The site is resolved from the `X-Public-Host` header → `SiteDomain.host` → `Site`. See [`backend/src/flows/flows.service.ts`](../backend/src/flows/flows.service.ts) → `stripeSecretForPublicHost()`.

> Rule: never read `process.env.STRIPE_SECRET_KEY` directly in feature code — go through the public-host resolution helper so white-label tenants get their own key. The global env is the fallback path only.

### Who owns what — the full data map

| Data | Source | Where it lives / how we get it |
|------|--------|-------------------------------|
| `stripeSecretKey`, `stripePublishableKey` | 🗄️ DB | `Site` row (env is fallback) |
| `stripeInboundWebhookUuid`, `stripeInboundWebhookSigningSecret` | 🗄️ DB | `Site` row (UUID we generate; signing secret returned by Stripe on register, then stored) |
| `stripeProductId`, `stripePriceId` | 🗄️ DB | `Paywall` row — **just the IDs**, the price details are NOT stored |
| `trialEnabled`, `trialDays`, `stripePriceTrials` | 🗄️ DB | `Paywall` row — trial config is ours |
| `expressCheckoutEnabled`, `enabledPaymentMethods` | 🗄️ DB | `Paywall` row — method toggles are ours |
| Paywall branding (colors, font, headline) | 🗄️ DB | `Paywall` / template rows — passed into Stripe `branding_settings` |
| Price amount, currency, interval, nickname | 💳 Stripe | `stripe.prices.list/retrieve` — read live every time |
| `tierLabel`, `marketingHeadline`, `compareAtCents` | 💳 Stripe | Read from the Stripe **price/product metadata** (merchant sets in Stripe dashboard) |
| Available payment methods (PMC) | 💳 Stripe | `paymentMethodConfigurations` — never hardcoded |
| Checkout Session, `clientSecret`, payment status | 💳 Stripe | Created/retrieved per request; not persisted |
| Subscription status (`trialing`/`active`/`canceled`) | 💳 Stripe | `stripe.subscriptions.retrieve/search` |
| Customer email | 💳 Stripe | `customer_details.email` / `customer.email` from the session |
| `X-Public-Host` (→ which site/keys) | 🌐 Client | Request header, set by the browser |
| Session id on return (`?payment=…`) | 🌐 Client | Stripe redirect URL → browser → our confirm endpoint |
| UTM, `_fbc`/`_fbp`, Amplitude/tracking session ids | 🌐 Client | Captured in browser, sent at checkout, copied into Stripe metadata |
| `app_user_id` (respondent token) | 🗄️ DB→💳 | We generate it (`Response.respondentToken`), then stamp it into Stripe subscription metadata |
| Entitlement grant/revoke | 🔌 External | RevenueCat — we push to it after Stripe confirms |
| Purchase event, confirmation email, CRM sync | 🔌 External | Meta CAPI, SMTP/ActiveCampaign — downstream of a Stripe event |

> Read this table top to bottom as the rule of thumb: **rows we configure → 🗄️ DB; anything about money/payment state → 💳 Stripe; per-visit context → 🌐 Client; side effects → 🔌 External.**

---

## 3. The payment flow (end user)

This is the **full journey** from the moment the respondent lands, through the survey, to a completed payment. The runner is a single stage machine in [`SurveyRunner.tsx`](../frontend/src/components/survey/SurveyRunner.tsx): `landing → questions → complete → paywall`, where `paywall` has two sub-steps (`select_plan` → `checkout`). Each step is tagged with where its data comes from: 🗄️ DB · 💳 Stripe · 🌐 Client · 🔌 External.

```
 USER LANDS  ──►  GET /start/[slug]  (or /start?flow_id=…)                         🌐 Client host
      │           Next.js SSR resolves the flow by slug + X-Public-Host → team/site  🌐→🗄️
      ▼
 ┌─ LANDING (#) ───────────────────────────────────────────────────────────────┐
 │  Hero / "Start" screen from the template       🗄️ DB (flow.template)         │
 │  (skippable via template.skipLandingScreen)                                   │
 └───────────────────────────────────────────────────────────────────────────────┘
      │ user taps Start
      │   POST /api/flows/:id/session  → { respondentToken }   🗄️ DB (we mint it; Response row)
      ▼
 ┌─ QUESTIONS (#q-<id>) ─────────────────────────────────────────────────────────┐
 │  One question at a time, branching.            🗄️ DB (survey/questions/rules) │
 │  Each answer:                                                                  │
 │    POST /api/flows/:id/next  body { respondentToken, questionId, optionIds }   │
 │        ▲ respondentToken 🗄️  ▲ answers 🌐 Client → stored on Response 🗄️       │
 │        ▼ returns next question OR "go to paywall"                              │
 └───────────────────────────────────────────────────────────────────────────────┘
      │ last question answered → stage becomes "paywall"
      ▼
 ╔═ PAYWALL · sub-step select_plan (#paywall) ══════════════════════════════════╗
 ║  GET /api/flows/:id/prices                                                    ║
 ║     ← plan cards: amount/currency/interval 💳 Stripe  +  layout/trials 🗄️ DB  ║
 ║     ← which wallets/embedded methods are active = 💳 PMC ∩ 🗄️ toggles        ║
 ║                                                                               ║
 ║  IF Express wallets enabled (expressCheckoutEnabled 🗄️):                      ║
 ║     POST /api/flows/:id/checkout  ×N plans   body { priceId 💳, checkoutUi:   ║
 ║        "elements", userId 🗄️, utm/meta/amplitude 🌐 }                          ║
 ║        → { clientSecret } 💳  per plan (pre-warmed)                            ║
 ║     ExpressCheckoutElement renders wallet buttons (Apple/Google/Amazon/…) 💳  ║
 ║                                                                               ║
 ║  Screen = plan cards + wallet buttons + a "Continue" button                   ║
 ╚═══════════════════════════════════════════════════════════════════════════════╝
      │                                          │
      │ taps a WALLET button                     │ taps CONTINUE (pay by card etc.)
      │ (pays inline, no step change)            ▼
      │                          ╔═ PAYWALL · sub-step checkout (#paywall-checkout) ═══════════╗
      │                          ║  POST /api/flows/:id/checkout  body { priceId 💳,           ║
      │                          ║     checkoutUi: "embedded", userId 🗄️, utm/meta 🌐 }        ║
      │                          ║     → { clientSecret } 💳, ui_mode "embedded_page"          ║
      │                          ║  branding_settings (colors/font) 🗄️ DB (paywall/template)  ║
      │                          ║                                                             ║
      │                          ║  EmbeddedCheckout iframe = full payment form 💳            ║
      │                          ║  (card, Klarna, iDEAL, ACH … = active embedded methods)    ║
      │                          ╚═════════════════════════════════════════════════════════════╝
      │                                          │
      └──────────────┬───────────────────────────┘
                     ▼  payment succeeds → Stripe redirects to
                        /start?flow_id=…&payment={CHECKOUT_SESSION_ID}     🌐 Client (session id)
                     ▼
 ┌─ CONFIRM (handled by StartFlowHost) ──────────────────────────────────────────┐
 │  POST /api/flows/:id/payment-confirm  body { sessionId 🌐 }                    │
 │    → retrieve session/subscription 💳 Stripe (verify paid/trialing)           │
 │    → grant access in RevenueCat 🔌  (keyed by app_user_id, read back from 💳)  │
 │    → confirmation email 🔌  +  Meta CAPI Purchase 🔌 (uses _fbc/_fbp 🌐)        │
 │    → returns { confirmed, customerEmail, value, plan } 💳-derived → analytics  │
 │                                                                                │
 │  POST /api/flows/:id/payment-confirm-email  (optional)                         │
 │    backfill email 🌐 if a wallet payment produced none → update customer 💳    │
 └───────────────────────────────────────────────────────────────────────────────┘
      │
      ▼  ThankYou / success screen 🗄️ DB (template)

 (Aside: an "exit paywall" #exit-paywall can appear on back/refresh — same prices/checkout APIs.)
```

**One-line summary of the data movement:**
- Landing + questions are **all 🗄️ DB** (your template, survey, branching) with answers **🌐 Client** written back to 🗄️ DB.
- The paywall **merges 🗄️ DB (layout/toggles/trials) with 💳 Stripe (prices, methods, session)**.
- Checkout is the **hand-off**: 🗄️ + 🌐 data is copied *into* the 💳 Stripe session metadata so it survives to webhooks/confirm.
- Confirm reads it **back out of 💳 Stripe** and fans out to **🔌 External** systems (RevenueCat, email, Meta).

> The steps below detail only the **payment portion** (the paywall sub-steps onward). The landing + questions stages above are the survey runner, not Stripe — see `ai-context/` for the flow/session/branching internals.

### Step 1 — Plan step prices (`GET /api/flows/:id/prices`)
[`FlowsService.prices()`](../backend/src/flows/flows.service.ts). Resolves the site's Stripe key, reads the paywall's `stripeProductId` (lists all prices) or `stripePriceId` (single price), maps each to a plan option, and returns the active Express/Embedded method ids + per-price trial days. Returns `{ error: "stripe_unconfigured" }` if the site has no key, `{ error: "no_stripe_price" }` if the paywall has neither product nor price.

**Data sources in this step:**
- 🌐 Client → `X-Public-Host` header picks the site → 🗄️ DB Stripe key.
- 🗄️ DB → `Paywall.stripeProductId`/`stripePriceId`, trial config, payment-method toggles, plan-selector ordering.
- 💳 Stripe → the actual price amount/currency/interval/nickname, plus `tierLabel`/`marketingHeadline`/`compareAtCents` from price-or-product **metadata**, and the PMC list of available methods.
- So a "plan card" the user sees is a **merge**: layout/toggles/trials from our DB, the price + marketing metadata from Stripe.

### Step 2 — Create a Checkout Session (`POST /api/flows/:id/checkout`)
[`FlowsService.checkout()`](../backend/src/flows/flows.service.ts). The single endpoint that creates **both** session types based on `checkoutUi`:

- **`elements`** (wallets, plan step): `ui_mode: "elements"`, `payment_method_types` from the active Express methods. One session is created **per plan** so each wallet panel can pre-warm.
- **`embedded`** (iframe, checkout step): `ui_mode: "embedded_page"`, `payment_method_types` from the active Embedded methods, plus `branding_settings` (colors + font derived from the template).

Both share:
- `mode: "subscription"`, `line_items: [{ price: priceId, quantity: 1 }]` — `priceId` is 💳 Stripe's (came from Step 1).
- `subscription_data.trial_period_days` 🗄️ DB — trial config is ours.
- `branding_settings` 🗄️ DB — colors/font from the paywall/template.
- `subscription_data.metadata` — **this is where our + client data gets copied into Stripe** so it survives to webhooks (which have no cookies): `flow_id` 🗄️, `app_user_id` (the respondent token) 🗄️, UTM fields / Meta `_fbc`/`_fbp` / Amplitude session id / `aw_tracking_session_id` 🌐 Client, `trial_days` 🗄️.
- Returns `{ clientSecret }` 💳 Stripe for the browser to mount the Stripe component.

> Step 2 is the **hand-off point**: data flows 🗄️ DB + 🌐 Client → into 💳 Stripe (as session params + metadata). After this, that context lives inside the Stripe session/subscription.

> **Why `app_user_id` matters:** it is the respondent's `respondentToken`. It's stamped into subscription metadata at checkout so every later webhook (and the post-redirect confirm) can map a Stripe subscription back to our respondent and to RevenueCat — even though respondents have no accounts.

### Step 3 — Confirm after redirect (`POST /api/flows/:id/payment-confirm`)
[`FlowsService.confirmPayment()`](../backend/src/flows/flows.service.ts). The browser returns from Stripe with the session id and calls this. It:

1. Takes the session id 🌐 Client (from the `?payment=…` redirect) and retrieves the session 💳 Stripe (expanding `customer` + `line_items.data.price`).
2. Decides "paid-like" from 💳 Stripe `payment_status`, **or** a 💳 Stripe subscription whose status is `trialing`/`active` (trials have no payment yet but should grant access).
3. Resolves the customer email via a 3-level fallback, all 💳 Stripe: `customer_details.email` → `customer_email` → expanded `customer.email`. Wallet payments (Apple/Google Pay) often have no typed email — hence the fallback; if all are null the frontend shows `PostPaymentEmailCapture`.
4. **Provisions access** 🔌 External: notifies RevenueCat (`notifyStripeReceipt`) keyed by `app_user_id` (read back out of 💳 Stripe metadata, where we put it in Step 2).
5. Sends the payment-confirmation email 🔌 External (SMTP / ActiveCampaign), idempotently.
6. Fires Meta CAPI `Purchase` 🔌 External (fired here, not from the webhook, because only the browser 🌐 has the `_fbc`/`_fbp` cookies).
7. Returns `{ confirmed, customerEmail, currency, value, plan, plan_frequency }` (💳 Stripe-derived) for client-side analytics.

> The redirect-confirm path is deliberately the primary provisioning path (not the webhook) because it runs with browser context. The webhook is the backup for cases where the user never returns.

### Step 4 — Email backfill (`POST /api/flows/:id/payment-confirm-email`)
Captures an email after a wallet payment that produced none, so the confirmation email and CRM sync can still happen.

Frontend orchestration: [`PaywallScreen.tsx`](../frontend/src/components/survey/PaywallScreen.tsx) (plan/checkout steps), [`PaywallCheckoutPanel.tsx`](../frontend/src/components/survey/PaywallCheckoutPanel.tsx) (Express Checkout wallets), [`StartFlowHost.tsx`](../frontend/src/components/start/StartFlowHost.tsx) (handles the post-redirect confirm).

---

## 4. Payment methods: Express vs Embedded

We don't hardcode the method list — it comes from the merchant's Stripe **Payment Method Configuration (PMC)**, intersected with per-paywall toggles. Methods split into two buckets ([`stripe-payment-methods.util.ts`](../backend/src/paywalls/stripe-payment-methods.util.ts)):

- **Express** (`EXPRESS_CHECKOUT_PMC_KEYS`): `link`, `apple_pay`, `google_pay`, `amazon_pay`, `paypal` → rendered as wallet buttons on the plan step via `ExpressCheckoutElement`.
- **Embedded**: everything else (card, Klarna, iDEAL, ACH, …) → rendered in the Embedded Checkout iframe.

A method is live only if **(1)** the Stripe PMC has it enabled **and** **(2)** the paywall toggle is on. Merchants configure this in the paywall editor (`GET /api/paywalls/:id/payment-methods` to sync from Stripe, `PUT /api/paywalls/:id` to save). Full detail of this slice lives in [payment-methods.md](./payment-methods.md).

### ⚠️ Which PMC do we read? — the `is_default` one (currently RevenueCat's)

We do **not** store a PMC id. [`fetchDefaultPaymentMethodConfiguration`](../backend/src/paywalls/stripe-payment-methods.util.ts#L146) lists all PMCs and picks **whichever has `is_default: true`** (falling back to the first):

```ts
const list = await stripe.paymentMethodConfigurations.list({ limit: 100 });
const def = list.data.find((c) => c.is_default) ?? list.data[0];
```

`is_default` is a property of the PMC object, **not** the dashboard's "Your account" / "RevenueCat account" grouping — `list()` returns everything in one flat array and we take the default flag.

**As verified against the live account, the PMC carrying `is_default` is the "Default" under _RevenueCat account_ (`pmc_1TKFip…RL001`), NOT "Your account / Default" (`pmc_1TKFNz…`).** RevenueCat sets one of its configs as the account default when you connect Stripe, and our code follows that flag.

**Consequences:**
- To change which methods our app offers, edit the **RevenueCat-account "Default"** PMC (`pmc_1TKFip…`) in Stripe — toggling methods in "Your account / Default" or the standalone `apple pay`/`amazon pay` PMCs has **no effect** on our app.
- This is **fragile**: nothing is pinned. If RevenueCat (or anyone) later moves the `is_default` flag to a different PMC, our app silently starts reading a different method list, with no alert. (Candidate tech-debt: let each `Site` pin a PMC id instead of trusting the moving default.)
- Verify the current default any time with: `stripe payment_method_configurations list --limit 100 | jq -r '.data[]|"\(.is_default)\t\(.id)\t\(.name)"'` — the `true` row is what our code uses.

---

## 5. Webhooks (Stripe → us)

There are **two distinct inbound webhook systems**. Both verify the `Stripe-Signature` and require the raw request body (Nest is configured with `rawBody: true` in `main.ts`).

### 5a. Global webhook — `POST /api/webhooks/stripe`
[`WebhooksController`](../backend/src/webhooks/webhooks.controller.ts). Uses the global `STRIPE_WEBHOOK_SECRET`. Handles two events for entitlement provisioning:

The event arrives from 💳 Stripe; `app_user_id`/`flow_id` are read back out of 💳 Stripe metadata; keys are resolved from 🗄️ DB; the grant/revoke goes to 🔌 RevenueCat.

| Event (💳 Stripe) | Action |
|-------|--------|
| `checkout.session.completed` | Resolve RevenueCat keys 🗄️ DB, `notifyStripeReceipt(app_user_id, subscriptionId)` 🔌, send confirmation email 🔌. Backup for users who don't return to the redirect-confirm path. |
| `customer.subscription.deleted` | Look up the customer 💳, resolve keys 🗄️, `revokeEntitlement(app_user_id)` 🔌 so access is removed in RevenueCat. |

> Most other Stripe webhook event handlers are intentionally stubs (see Known Technical Debt in `ai-context/conventions-and-patterns.md`). The two above are the ones that drive entitlement state.

### 5b. Per-site inbound webhook — `POST /webhook/:uuid`
[`SiteStripeInboundWebhookController`](../backend/src/webhooks/site-stripe-inbound-webhook.controller.ts). This is the white-label path: each site gets a unique URL `https://<host>/webhook/<stripeInboundWebhookUuid>`, verified with that **site's own** `stripeInboundWebhookSigningSecret`.

- A `GET /webhook/:uuid` probe returns 2xx for a known site UUID (Stripe reachability checks use GET).
- On `POST`, if no signing secret is stored yet it returns 2xx without logging (the secret arrives only after `webhookEndpoints.create` succeeds — we mustn't reject before it's set).
- Registered with a **curated** event set only (not `*`): `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid` ([`stripe-webhook-curated.ts`](../backend/src/webhooks/stripe-webhook-curated.ts)).
- Drives **lifecycle analytics automation** ([`StripeWebhookAutomationService`](../backend/src/webhooks/stripe-webhook-automation.service.ts)): raw Stripe events are mapped to **semantic lifecycle events** — `trial_start`, `first_payment_recorded`, `subscription_cancelled`, `subscription_ended` ([`stripe-lifecycle-semantics.ts`](../backend/src/webhooks/stripe-lifecycle-semantics.ts)) — then fanned out to destinations (ClickHouse/tracking by default; Amplitude and Meta CAPI per-row opt-in).

**Managing the per-site webhook** (admin, [`sites.controller.ts`](../backend/src/sites/sites.controller.ts)):
- `POST /api/sites/:siteId/stripe/register-inbound-webhook` — calls `stripe.webhookEndpoints.create` against the curated events, stores the returned `whsec_…`.
- `POST /api/sites/:siteId/stripe/delete-inbound-webhook` and `.../clear-inbound-webhook-signing-secret`.
- `GET /api/sites/:siteId/stripe/inbound-webhook-status`.
- UI: [`frontend/src/app/(dashboard)/sites/`](../frontend/src/app/(dashboard)/sites/) (`stripe-inbound-webhook-*`, `site-webhook-automations-drawer.tsx`).

---

## 6. Cancellation & refunds — `POST /api/cancel`
[`CancelService`](../backend/src/cancel/cancel.service.ts). A self-serve cancel endpoint:

1. Finds the active subscription via `stripe.subscriptions.search` (scoped by `app_user_id` metadata — ownership check).
2. If the subscription was created within the site's `refundWindowDays` (default 30): cancel immediately (`subscriptions.cancel`) and attempt a refund of the latest invoice payment (`invoicePayments.list` → `refunds.create`).
3. Outside the window: `cancel_at_period_end: true`, no refund.

The `customer.subscription.deleted` webhook then revokes the RevenueCat entitlement.

---

## 7. Subscriptions, trials, and analytics metadata

- **Everything is a subscription.** No one-time payments. "Paid" includes `trialing` — a free trial grants access immediately.
- **Trials** are per-price (`stripe_price_trials` JSON on the paywall) and applied via `subscription_data.trial_period_days`.
- **Plan analytics** (`plan`, `plan_frequency`, amount, currency) are derived from the Stripe Price via [`plan-analytics.util.ts`](../backend/src/flows/plan-analytics.util.ts) and threaded into both checkout metadata and confirm responses for Amplitude/Meta.
- **Mobile entitlements** are never granted from the browser — RevenueCat is always called server-side from the confirm path or webhooks.

---

## 8. What you can build / extend with Stripe here

| Goal | How |
|------|-----|
| Add a new payment method | Usually **no code** — enable it in Stripe PMC, sync in the paywall editor, toggle on. Code only if Stripe adds a new *Express wallet* → add its key to `EXPRESS_CHECKOUT_PMC_KEYS`. |
| Add/replace plans | Change the paywall's `stripeProductId`/`stripePriceId` and Stripe prices — no deploy. |
| Add a free trial to a plan | Set per-price trial in the paywall editor (`stripe_price_trials`). |
| React to a new Stripe event for analytics | Add it to `STRIPE_WEBHOOK_CURATED_EVENT_TYPES`, map it to a semantic lifecycle event, re-register the per-site endpoint. |
| Provision access on a new event | Extend the global `WebhooksController` switch (most non-handled events are currently stubs). |
| Change refund policy | `Site.refundWindowDays`. |
| Onboard a new white-label tenant | Set the tenant's keys on its `Site`, register its inbound webhook. No shared code changes. |

---

## 9. Key files

| Area | File |
|------|------|
| Prices / checkout / confirm (the core flow) | [`backend/src/flows/flows.service.ts`](../backend/src/flows/flows.service.ts) |
| Public flow routes | [`backend/src/flows/flows.controller.ts`](../backend/src/flows/flows.controller.ts) |
| Plan analytics from Stripe Price | [`backend/src/flows/plan-analytics.util.ts`](../backend/src/flows/plan-analytics.util.ts) |
| PMC discovery + Express/Embedded split | [`backend/src/paywalls/stripe-payment-methods.util.ts`](../backend/src/paywalls/stripe-payment-methods.util.ts) |
| Global webhook (provision/revoke) | [`backend/src/webhooks/webhooks.controller.ts`](../backend/src/webhooks/webhooks.controller.ts) |
| Per-site inbound webhook | [`backend/src/webhooks/site-stripe-inbound-webhook.controller.ts`](../backend/src/webhooks/site-stripe-inbound-webhook.controller.ts) |
| Lifecycle automation + semantics | [`backend/src/webhooks/stripe-webhook-automation.service.ts`](../backend/src/webhooks/stripe-webhook-automation.service.ts), [`stripe-lifecycle-semantics.ts`](../backend/src/webhooks/stripe-lifecycle-semantics.ts), [`stripe-webhook-curated.ts`](../backend/src/webhooks/stripe-webhook-curated.ts) |
| Cancel + refund | [`backend/src/cancel/cancel.service.ts`](../backend/src/cancel/cancel.service.ts) |
| RevenueCat provisioning | [`backend/src/webhooks/revenuecat.service.ts`](../backend/src/webhooks/revenuecat.service.ts) |
| Site Stripe webhook management | [`backend/src/sites/sites.controller.ts`](../backend/src/sites/sites.controller.ts) |
| Frontend Stripe helper (session verify) | [`frontend/src/lib/stripe.ts`](../frontend/src/lib/stripe.ts) |
| Live paywall UI | [`frontend/src/components/survey/PaywallScreen.tsx`](../frontend/src/components/survey/PaywallScreen.tsx), [`PaywallCheckoutPanel.tsx`](../frontend/src/components/survey/PaywallCheckoutPanel.tsx) |
| Post-redirect confirm host | [`frontend/src/components/start/StartFlowHost.tsx`](../frontend/src/components/start/StartFlowHost.tsx) |
| Payment-method editor | [`frontend/src/components/paywall/PaymentMethodsEditor.tsx`](../frontend/src/components/paywall/PaymentMethodsEditor.tsx) |

---

## 10. Environment variables (Stripe)

| Variable | Service | Role |
|----------|---------|------|
| `STRIPE_SECRET_KEY` | Nest | **Global fallback** server key — overridden by `Site.stripeSecretKey` |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Next | Global fallback publishable key — overridden by `Site.stripePublishableKey` |
| `STRIPE_WEBHOOK_SECRET` | Nest | Signature secret for the **global** `/api/webhooks/stripe` endpoint only (per-site endpoints use `Site.stripeInboundWebhookSigningSecret`) |

> Do **not** set `NEXTAUTH_URL` / `NEXT_PUBLIC_URL` — Stripe return URLs are built from the dynamically-derived request host.
