# Paywall & Stripe — the complete lifecycle, step by step

> Picks up the moment the respondent answers the **last survey question** and follows every step
> through to money in the Stripe account and the events that record it.
>
> Every API call, every file, every Stripe request, in order. Traced from source on 2026-08-17.
>
> Companions: [02-api-reference.md](./02-api-reference.md) (route list) ·
> [../02-features-and-optimizations/stripe.md](../02-features-and-optimizations/stripe.md) (Stripe integration reference) ·
> [../02-features-and-optimizations/payment-methods.md](../02-features-and-optimizations/payment-methods.md) (which methods show where)

---

## Contents

- [0. What must already be configured](#0-what-must-already-be-configured)
- [1. The 30-second version](#1-the-30-second-version)
- [Step 1 — Leaving the last question](#step-1--leaving-the-last-question)
- [Step 2 — Warm-up during the break screen](#step-2--warm-up-during-the-break-screen)
- [Step 3 — `PaywallScreen` mounts and fetches prices](#step-3--paywallscreen-mounts-and-fetches-prices)
- [Step 4 — Wallet prefetch (the expensive one)](#step-4--wallet-prefetch-the-expensive-one)
- [Step 5 — The plan step is on screen](#step-5--the-plan-step-is-on-screen)
- [Step 6 — Continue → the checkout step](#step-6--continue--the-checkout-step)
- [Step 7 — Embedded Checkout mounts](#step-7--embedded-checkout-mounts)
- [Step 8 — Payment and the redirect back](#step-8--payment-and-the-redirect-back)
- [Step 9 — `payment-confirm`](#step-9--payment-confirm)
- [Step 10 — The webhook, in parallel](#step-10--the-webhook-in-parallel)
- [Event timeline](#event-timeline)
- [Stripe call budget](#stripe-call-budget)
- [From "I made a plan in Stripe" to "the user sees it"](#from-i-made-a-plan-in-stripe-to-the-user-sees-it)
- [Failure modes](#failure-modes)

---

## 0. What must already be configured

Nothing below works unless all four of these exist. This is the checklist when a paywall renders empty.

| # | Thing | Where | Consequence if missing |
|---|---|---|---|
| 1 | **Stripe secret + publishable key** | `site.stripeSecretKey`, `site.stripePublishableKey` (falls back to `STRIPE_SECRET_KEY`) | `503 stripe_unconfigured` |
| 2 | **A Product or a Price in Stripe** | Stripe Dashboard → recorded on `paywall.stripeProductId` **or** `paywall.stripePriceId` | `422 no_stripe_price` |
| 3 | **A `paywall` row wired to the flow** | `flow.paywallId` (and/or `exitPaywallId`, `paymentExitPaywallId`) | `404 no_paywall` |
| 4 | **A default Payment Method Configuration** | Stripe Dashboard → Settings → Payment methods | Silently degrades to card-only (see [`resolvePaywallPaymentConfig`](../../backend/src/flows/flows.service.ts#L301) — the `catch` returns `embeddedMethodIds: ["card"]`) |

**Product vs Price is the key branch.** Set `stripeProductId` → the backend lists *all* active recurring prices on that product, and the user gets a plan picker. Set only `stripePriceId` → one price, no picker.

---

## 1. The 30-second version

```
last question answered
   └─ POST /api/flows/:id/next   → { nextQuestionId: null, isComplete: true }
        │
        ├─ (optional) before_paywall break screen  ──► warmPaywallInBackground()
        │                                              prefetch prices + wallets + JS + Stripe.js
        │
        └─ stage = "paywall"  →  <PaywallScreen>
             │
             ├─ GET  /api/flows/:id/prices           ← 3+ Stripe calls
             │      └─ event: paywall_loaded
             │
             ├─ POST /api/flows/:id/checkout × N     ← 4 Stripe calls EACH (wallet prefetch)
             │      (only when Express Checkout is on)
             │      └─ event: express_checkout_shown
             │
             ├─ [plan step] user picks a plan, hits Continue
             │      └─ events: plan_selected, more_payment_methods_clicked
             │
             ├─ [checkout step] POST /api/flows/:id/checkout   ← 4 Stripe calls
             │      └─ events: stripe_page_loaded, stripe_iframe_loaded
             │
             └─ user pays → Stripe redirects to
                /start?flow_id=N&payment=cs_xxx
                  │
                  ├─ POST /api/flows/:id/payment-confirm  ← 1–2 Stripe calls
                  │      └─ event: user_subscribed
                  │
                  └─ (async, separate) Stripe → POST /webhook/{uuid}
```

---

## Step 1 — Leaving the last question

**File:** [SurveyRunner.tsx](../../frontend/src/components/survey/SurveyRunner.tsx) · **API:** `POST /api/flows/:id/next`

The client never decides the survey is over. It posts the answer and the backend's branching engine
([`domain/engine.ts`](../../backend/src/domain/engine.ts)) replies `isComplete: true`.

The runner then picks one of three destinations, in this priority order:

```ts
if (beforePaywallBreak && !wasBeforePaywallBreakSeen(flow.id)) {
  enterLifecycleBreak("before_paywall", true);   // → Step 2
} else {
  setStage("paywall");                            // → Step 3
  replaceState({ stage: "paywall", paywallStep: "select_plan" }, "#paywall");
}
```

Two things worth noting:

- **`wasBeforePaywallBreakSeen(flow.id)`** — the break screen shows once per flow per session, so a back-press into the paywall doesn't replay the loader.
- **The history entry is `replaceState`, not `pushState`.** The paywall *replaces* the last question in the history stack. A back-press from the paywall goes to the question before it, which is what the exit-paywall interception hooks into.

`stage === "paywall"` immediately fires **`payment_gateway_redirected`** ([SurveyRunner.tsx:1218](../../frontend/src/components/survey/SurveyRunner.tsx#L1218)) — despite the name, this fires on *entering the paywall screen*, not on redirecting to Stripe. Guarded by a `trackedStagesRef` set so React Strict Mode's double-invoke can't double-fire it.

---

## Step 2 — Warm-up during the break screen

**File:** [lib/paywall-preload-cache.ts](../../frontend/src/lib/paywall-preload-cache.ts)

This is a pure optimization — skip it and everything still works, just slower. If the break screen's
config asks for it (`shouldPreloadPaywallOnBreak`), `warmPaywallInBackground()` runs while the loader
animation is on screen and does **four** things in parallel:

```ts
preloadPaywallPrices(flowId, exitPaywall, publicHost);   // 1. GET /prices → in-memory cache
void import("@/components/survey/PaywallScreen");        // 2. pull the paywall JS chunk
if (key) void loadStripe(key);                           // 3. load Stripe.js
// 4. once prices land → startWalletPrefetch(...) → POST /checkout per price
```

The cache is a module-level `Map` keyed `{flowId}:main` or `{flowId}:exit`, holding the in-flight
promise as well as the settled data — so a consumer arriving mid-flight awaits the same request
rather than firing a second one.

**`consume*` deletes the entry as it reads it.** One-shot by design: a stale price list must never be
served to a second paywall mount.

⚠️ **The payment-exit paywall deliberately opts out** of this cache. Its keying only distinguishes
main vs exit, so serving it a cached payload would risk handing it the *regular* exit paywall's
prices. It always live-fetches — see the guard `!isPaymentExitPaywall` in `PaywallScreen`'s loader.

---

## Step 3 — `PaywallScreen` mounts and fetches prices

**File:** [PaywallScreen.tsx](../../frontend/src/components/survey/PaywallScreen.tsx) → `loadPaywall()`
**API:** `GET /api/flows/:id/prices` (+ `?exitPaywall=true` / `?paymentExitPaywall=true`)

### Frontend

```ts
const cached = await consumePaywallPreload(flowIdNum, isExitPaywall);   // Step 2's cache
if (raw == null) {
  const res = await fetch(`/api/flows/${flowId}/prices${pricesQs}`, {
    headers: { ...publicHostHeaders() },     // X-Public-Host — mandatory
  });
  raw = await res.json();
}
```

### Backend — [`FlowsService.prices()`](../../backend/src/flows/flows.service.ts#L1201)

Step by step:

**1. Authorize the flow against the host.** `assertFlowActiveOnTeam(flowId, publicHost)` — the flow must be `ACTIVE` **and** belong to the team that `X-Public-Host` resolves to. This is the entire authorization model for the public surface.

**2. Pick which paywall.** Three-way, with fallback:

```ts
const paywallId = opts?.paymentExitPaywall
  ? (flow?.paymentExitPaywallId ?? flow?.exitPaywallId)   // note the fallback
  : opts?.exitPaywall
    ? flow?.exitPaywallId
    : flow?.paywallId;
```

**3. Read trial/discount JSON via `$queryRaw`.** Not through Prisma — a nested `try/catch` ladder that degrades from `(trials + discounts)` → `(trials only)` → `undefined`. The columns may not exist on an environment that hasn't migrated yet, and a paywall that renders without discounts beats one that 500s.

**4. Resolve the Stripe key** — `stripeSecretForPublicHost()`: `site.stripeSecretKey` first, `STRIPE_SECRET_KEY` second. A **new `Stripe` client is constructed per request** — there's no shared singleton, because the key varies per site.

**5. Resolve payment methods** — `resolvePaywallPaymentConfig()` → **2 Stripe calls**:
```
stripe.paymentMethodConfigurations.list({ limit: 100 })   → find is_default
stripe.paymentMethodConfigurations.retrieve(def.id)
```
Then it intersects what **Stripe** has enabled with what the **paywall** has enabled (`paywall.enabledPaymentMethods`), splitting the result into `expressMethodIds` (wallet buttons) and `embeddedMethodIds` (inside the Stripe iframe). Both sides must say yes for a method to appear.

**6. Fetch prices** — **1 Stripe call**, branching on config:

```ts
if (stripeProductId) {
  stripe.prices.list({ product, active: true, type: "recurring", expand: ["data.product"] })
} else if (stripePriceId) {
  stripe.prices.retrieve(stripePriceId, { expand: ["product"] })
}
```

**7. Sort → order → filter → enrich:**

| Stage | Function | Does |
|---|---|---|
| sort | inline | ascending by `unitAmount` |
| order | `sortOptionsByPlanSelectorPriceOrder` | applies the editor's manual drag order from `planSelectorUi` |
| filter | `filterOptionsByPlanPriceEnabled` | drops prices toggled off in the editor |
| enrich | `enrichPriceOptionsWithCouponDisplay` | **+1 Stripe call per distinct coupon** (deduped via a local `Map`) → adds `discountedUnitAmountCents` |

**8. Compute trial days per price** — `trialDaysForCheckoutPrice()`, pure, no Stripe call. The precedence rule:

```
per-price row exists for THIS price   → use it (enabled ? days : null)
any price_* key exists in the JSON    → null          ← legacy is fully disabled
otherwise                             → legacy trialEnabled/trialDays
```

That middle line is the subtle one: **the moment you configure a per-price trial for *any* price, the legacy `trialEnabled`/`trialDays` stops applying to *every* price.** It's all-or-nothing, not a per-price fallback.

### The response

```jsonc
{
  "options": [ { "id": "price_...", "unitAmount": 999, "currency": "usd",
                 "interval": "week", "intervalCount": 1,
                 "discountedUnitAmountCents": 499 } ],
  "trialDaysByPriceId":   { "price_...": 7 },
  "trialToggleEnabled":   true,          // from template, not paywall
  "trialToggleLabel":     "Start with a {days}-day free trial",
  "expressCheckoutEnabled": true,
  "expressMethodIds":     ["apple_pay", "google_pay"],
  "expressCheckoutButtonTheme": { "mode": "auto" },
  "embeddedMethodIds":    ["card", "link"]
}
```

On success the client calls `setLoadingPrices(false)` and fires **`paywall_loaded`** with
`{ price_count, express_checkout_enabled }`.

---

## Step 4 — Wallet prefetch (the expensive one)

Only when `expressCheckoutEnabled && expressMethodIds.length > 0 && stripePublishableKey && !skipPlanPicker`.

Apple Pay / Google Pay buttons are Stripe **Elements**, and Elements needs a `client_secret` before
it can render. So the paywall creates a **Checkout Session per price, up front**, so switching plans
doesn't re-initialise the wallet buttons:

```ts
const results = await Promise.allSettled(
  list.map((price) => fetchWalletSecretFromApi(price.id))   // POST /checkout, checkoutUi: "elements"
);
```

**This is the single most expensive thing the paywall does.** With 3 plans, that's 3 concurrent
`POST /checkout` calls → **12 Stripe API calls** (4 each, see the budget table) — before the user has
touched anything.

`Promise.allSettled`, so one failing plan doesn't kill the rest; `walletError` is only surfaced when
the failure is on the *default* price.

Each secret is rendered into a memoized `WalletPanelSlot`. All panels stay mounted; only the selected
one is visible. From the comment in the code:

> selecting a different plan only re-renders the two panels whose `isSelected` actually flips — it
> never re-renders (and thus never re-mounts) the others' `PaywallCheckoutPanel`, so Stripe's
> `ExpressCheckoutElement` keeps its state instead of re-initialising on every plan switch.

When the wallet buttons actually appear, `onReady` fires **`express_checkout_shown`** (once per
paywall session, `expressCheckoutShownFiredRef`).

### The trial toggle re-fetches everything

`trial_period_days` is baked into the Checkout Session **at creation**. So flipping the toggle
invalidates every prefetched secret:

```ts
useEffect(() => {
  if (isFirstWantsTrialRender.current) { isFirstWantsTrialRender.current = false; return; }
  setCheckoutKey((k) => k + 1);                       // force embedded checkout remount
  // …re-POST /checkout for every price
}, [wantsTrial]);
```

Every toggle flip = another N × 4 Stripe calls. The first-render guard is what stops it running on mount.

---

## Step 5 — The plan step is on screen

Rendered by the layout from the registry: `const { PlanStep } = getPaywallTheme(paywallTheme)`.

Two things decide whether the plan picker appears at all:

```ts
const hidePlanPicker = initialPriceId is a valid price_ in the list;   // survey answer locked a plan
const shouldSkipPlanPage = prices.length === 0 || hidePlanPicker;
```

`initialPriceId` comes from a `PRICE_PLAN` survey answer — a respondent can pick their plan *during
the survey* and land straight on the card form.

The plan ↔ checkout sub-step is mirrored into `window.history.state.paywallStep`, with a `popstate`
listener keeping React in sync, so browser Back moves checkout → plan picker instead of leaving the
paywall entirely.

---

## Step 6 — Continue → the checkout step

`goToPayment()`:

```ts
trackPlanSelectedProceed();                              // event: plan_selected
onEvent?.("more_payment_methods_clicked", { … });        // event
setStep("checkout");
window.history.pushState({ stage: "paywall", paywallStep: "checkout" }, "", "#paywall-checkout");
// deliberately does NOT bump checkoutKey — no Embedded Checkout is mounted yet, and
// bumping can race Stripe's singleton
```

`plan_selected` carries the full plan context from `planSelectedTrackingProps(price, productName, trialDays)`.

---

## Step 7 — Embedded Checkout mounts

Stripe allows **only one Embedded Checkout instance per page**, so the mount is deferred to let the
previous teardown finish:

```ts
const t = wantsEmbeddedCheckout ? window.setTimeout(() => setEmbedMountAllowed(true), 100) : null;
```

That 100 ms is what makes React Strict Mode's double-mount and the plan→pay transition safe.

Then `EmbeddedCheckoutProvider` calls `fetchClientSecret` → **`POST /api/flows/:id/checkout`** with
`checkoutUi: "embedded"`.

### The request body — [`buildFlowCheckoutBody`](../../frontend/src/lib/flow-checkout-body.ts)

One shared builder for both the wallet and embedded paths, so Stripe metadata stays identical:

```jsonc
{
  "priceId": "price_...",
  "userId": "<respondentToken>",         // ← becomes metadata.app_user_id
  "checkoutUi": "embedded",
  "wantsTrial": false,                   // only when toggled off
  "exitPaywall": true,                   // or paymentExitPaywall
  "moment": "back",                      // close | back — picks branding template
  "utmCampaign": "...", "utmSource": "...",
  "metaFbc": "...", "metaFbp": "...",    // read from _fbc/_fbp cookies
  "amplitudeSessionId": "...",
  "awTrackingSessionId": "..."           // sessionStorage aw_tracking_session_id
}
```

**Why the tracking ids travel to Stripe:** webhooks arrive with no cookies and no browser context.
Stashing them in session metadata is the only way the server-side Meta CAPI / Amplitude / RevenueCat
calls can attribute the purchase back to the browser session that produced it.

### Backend — [`FlowsService.checkout()`](../../backend/src/flows/flows.service.ts#L1382)

1. `assertFlowActiveOnTeam` again
2. Pick the paywall (same three-way as `/prices`)
3. `resolveMomentBrandingTemplate(flowId, template, moment)` — `close`/`back` swap in the override template's colours/font for Stripe's hosted page, falling back to the main template. Also `$queryRaw`, same "column may predate the client" reason.
4. `isPlanPriceEnabled(...)` → `422 price_disabled` — **re-validated server-side**, so a tampered `priceId` for a disabled plan is rejected
5. Read trial + discount JSON (`$queryRaw`, three-level fallback ladder)
6. Resolve the Stripe key, construct `new Stripe(key)`
7. **Stripe call:** `stripe.prices.retrieve(priceId, { expand: ["product"] })` → for `plan` / `plan_frequency` analytics. Wrapped in `try/catch` with a `"subscription"`/`"monthly"` default — analytics must never block a sale.
8. Build the metadata (see below)
9. **Stripe calls ×2:** `resolvePaywallPaymentConfig()` again
10. `checkoutDiscountsForPrice(priceId, discountsJson)` → `[{ coupon }]` or `[{ promotion_code }]`
11. **Stripe call:** `stripe.checkout.sessions.create({...})`

### Two `ui_mode`s from one endpoint

| | `checkoutUi: "elements"` (wallets) | `checkoutUi: "embedded"` (card form) |
|---|---|---|
| `ui_mode` | `"elements"` | `"embedded_page"` |
| `payment_method_types` | `expressMethodIds` | `embeddedMethodIds` |
| `branding_settings` | — | background, button colour, `rounded`, font from the template |
| Rendered by | `ExpressCheckoutElement` | `<EmbeddedCheckout />` iframe |

Both get `mode: "subscription"`, the same `line_items`, the same `return_url`, and the same metadata.

### Metadata — written to two places

```ts
subscription_data.metadata = { flow_id, app_user_id, utm_campaign, utm_source,
                               meta_fbc, meta_fbp, amplitude_session_id,
                               aw_tracking_session_id, trial_days }
session.metadata          = { …the same… , plan, plan_frequency }
```

Session metadata is what `payment-confirm` reads on return; subscription metadata is what
**renewal** webhooks carry months later, when there is no checkout session in sight.

There's also a defensive backfill here — if the `response` row somehow has `sessionId: null`, it's
filled from `awTrackingSessionId`. Guarded on `sessionId: null` so an existing pairing is never
overwritten.

### Return URL

```
{origin}/start?flow_id={flowId}&payment={CHECKOUT_SESSION_ID}
```

`{CHECKOUT_SESSION_ID}` is a **Stripe template variable**, substituted by Stripe at redirect time.

Once mounted, two events fire: **`stripe_page_loaded`** (on `embedMountAllowed`) and
**`stripe_iframe_loaded`** — the latter via a `MutationObserver` watching for the iframe insertion,
because `EmbeddedCheckoutProvider` exposes no `onReady`.

---

## Step 8 — Payment and the redirect back

Whichever surface the user paid on, Stripe does a **real full-page redirect** to the `return_url`.
That means a brand-new document: React state is gone, and only `sessionStorage` survives.

`/start?flow_id=3&payment=cs_test_...` → `StartFlowHost` sees `payment=cs_…` and sets
`needsPaymentVerify`.

---

## Step 9 — `payment-confirm`

**File:** [StartFlowHost.tsx:265](../../frontend/src/components/start/StartFlowHost.tsx#L265) · **API:** `POST /api/flows/:id/payment-confirm`

```ts
body: { sessionId: paymentSession, metaFbc, metaFbp }   // fbc/fbp re-read from cookies
```

### Backend — [`confirmPayment()`](../../backend/src/flows/flows.service.ts#L1646)

**1. Stripe call:** `stripe.checkout.sessions.retrieve(sessionId, { expand: ["customer", "line_items.data.price"] })`

**2. Is it actually paid?**

```ts
let paidLike = payment_status === "paid" || payment_status === "no_payment_required";
if (!paidLike && mode === "subscription" && status === "complete" && subscription) {
  // Stripe call #2 — a trial subscription reports payment_status "unpaid"
  const sub = await stripe.subscriptions.retrieve(sid);
  paidLike = sub.status === "trialing" || sub.status === "active";
}
if (!paidLike) return { error: "not_paid" };
```

That second call exists because **a free-trial signup is not "paid"** — without it, every trial start
would be rejected.

**3. Resolve the email — three-level fallback:**

```
customer_details.email    → typed at Stripe checkout
customer_email            → legacy field
customer.email            → expanded Customer — the wallet case: a returning Apple Pay
                            customer never types an email, but Stripe has one on file
```

All three null → the frontend shows `PostPaymentEmailCapture`.

**4. Recover the canonical tracking session.** `metadata.app_user_id` → look up `response.sessionId`
in MySQL as the source of truth, falling back to `metadata.aw_tracking_session_id`. This is what
self-heals a tab whose `sessionStorage` was wiped by an in-app browser across the redirect.

**5. Three side effects, all failure-isolated** (each `.catch()`s and logs — none can break the response):

| Side effect | Service |
|---|---|
| RevenueCat receipt sync | `revenueCat.notifyStripeReceipt(appUserId, subscriptionId, keys)` |
| Confirmation email | `paymentConfirmationEmail.trySendForStripeSession(...)` |
| Meta CAPI `Purchase` | `metaConversions.sendPurchaseForCheckoutSession(...)` |

**6. Response** → `{ customerEmail, value, currency, plan, plan_frequency, respondentToken, trackingSessionId }`

### Back on the client

`setPaymentConfirmed(true)`, `setSessionId(body.trackingSessionId)` (the self-heal, applied *before*
`SurveyRunner` mounts), then **`user_subscribed`** fires with
`{ source: "stripe_redirect", stripe_checkout_session_id, value, currency, plan, plan_frequency }`.

For Meta, the event id **is** the Checkout Session id — that's what deduplicates the browser Pixel
event against the server CAPI event for the same purchase.

`sessionStorage[flow_paid_{flowId}] = "1"` makes the thank-you screen sticky, and an extra
`history.pushState` gives the first back-press something to pop to in this fresh document.

---

## Step 10 — The webhook, in parallel

Completely independent of Steps 8–9 — it fires even if the user closes the tab mid-redirect.

**`POST /webhook/{site.stripeInboundWebhookUuid}`** → [site-stripe-inbound-webhook.controller.ts](../../backend/src/webhooks/site-stripe-inbound-webhook.controller.ts)

1. Look up the site by the URL's uuid
2. `Stripe.webhooks.constructEvent(rawBody, signature, secret)` — invalid signature → `400`, nothing runs
3. Emit the full event to ClickHouse (`analytics.site_stripe_webhook_event_log_dist`) via Fluent
4. `webhookAutomations.processInboundEvent(site, event)` — fire-and-forget → Meta CAPI / Amplitude per the site's rules

> ⚠️ The MySQL write to `site_stripe_webhook_event_log` is **commented out** in the controller.
> ClickHouse is the only destination today, despite what [ai-context/paywall.md](../../ai-context/paywall.md) says.

The raw-body preservation that makes signature verification possible is set up in
[main.ts](../../backend/src/main.ts) via `express.json({ verify })`. Replace that with the default
body parser and every webhook fails verification.

---

## Event timeline

Every event fired from paywall entry onward, in order:

| # | Event | Fires when | Fired from |
|---|---|---|---|
| 1 | `payment_gateway_redirected` | `stage` becomes `"paywall"` | `SurveyRunner.tsx:1218` |
| 2 | `paywall_loaded` | `/prices` resolved | `PaywallScreen` `loadPaywall()` |
| 3 | `express_checkout_shown` | wallet buttons first render (`onReady`) | `handleExpressCheckoutShown` |
| 4 | `express_checkout_button_click` | user taps a wallet button, **before** the payment sheet | `handleExpressCheckoutButtonFocus` |
| 5 | `express_checkout_clicked` | Stripe's `onClick` with the resolved method | `handleExpressCheckoutClick` |
| 6 | `plan_selected` | Continue pressed, or a wallet clicked from the plan step | `trackPlanSelectedProceed` |
| 7 | `more_payment_methods_clicked` | Continue → card form | `goToPayment` |
| 8 | `stripe_page_loaded` | Embedded Checkout allowed to mount | `embedMountAllowed` effect |
| 9 | `stripe_iframe_loaded` | Stripe's iframe `load` event | `MutationObserver` effect |
| 10 | `user_subscribed` | `payment-confirm` returns OK | `SurveyRunner.tsx:1172` |
| 11 | `app_install_prompted` | thank-you screen with a deep link | `SurveyRunner.tsx:1222` |
| — | `payment_failed` | `/checkout` errors on the embedded path | `createCheckoutSessionForPrice` |
| — | `exit_paywall_shown` / `payment_exit_paywall_shown` | exit-intent paywall renders | buffered, flushed on render |

Note #4 vs #5: `express_checkout_button_click` fires on the **tap**; `express_checkout_clicked` on
Stripe's callback. With one visible method they're redundant; with several, #4 reports
`payment_type: "unknown"` and you correlate with #5 to learn which was actually chosen.

All of these route through `track()` → up to four destinations, each independently gated by
`tracking_event_configs`. See [../02-features-and-optimizations/events-tracking_new.md](../02-features-and-optimizations/events-tracking_new.md).

---

## Stripe call budget

Per endpoint:

| Endpoint | Stripe calls | Breakdown |
|---|---|---|
| `GET /prices` | **3 + C** | PMC list (1) + PMC retrieve (1) + prices list-or-retrieve (1) + one per distinct coupon (C, deduped) |
| `POST /checkout` | **4** | price retrieve (1) + PMC list (1) + PMC retrieve (1) + session create (1) |
| `POST /payment-confirm` | **1–2** | session retrieve (1) + subscription retrieve (1, only for trials) |

### A realistic paywall load

3 plans, Express Checkout on, no coupons:

```
GET  /prices                    →  3 calls
POST /checkout × 3 (wallets)    → 12 calls
POST /checkout   (embedded)     →  4 calls
POST /payment-confirm           →  2 calls
                                  ─────────
                                   21 Stripe API calls for one conversion
```

Flip the trial toggle once and add 12 more.

**Where the waste is:** `resolvePaywallPaymentConfig()` runs on **every** `/prices` and **every**
`/checkout` — 2 calls each — and the Payment Method Configuration changes maybe once a month. Eight of
those 21 calls are the same PMC fetched over and over. There's no cache today; the Stripe client
itself is reconstructed per request. Worth knowing before you debug a rate-limit or a slow paywall.

---

## From "I made a plan in Stripe" to "the user sees it"

The exact path a newly-created Stripe price takes to a plan card:

```
① Stripe Dashboard
     Product "Pro"  →  Price $9.99/week (recurring)  →  price_1Abc...

② Admin dashboard → Paywalls → edit
     paywall.stripeProductId = "prod_Xyz"       ← list ALL its recurring prices
     — or —
     paywall.stripePriceId   = "price_1Abc"     ← exactly one price, no picker

③ (optional) same editor
     planSelectorUi          → drag order + per-price on/off toggles
     stripePriceTrials       → { "price_1Abc": { enabled: true, days: 7 } }
     stripePriceDiscounts    → { "price_1Abc": { enabled: true, couponId: "INTRO" } }
     enabledPaymentMethods   → { "apple_pay": true, "card": true }

④ Admin dashboard → Flows → edit
     flow.paywallId = <that paywall>

⑤ Runtime — GET /api/flows/:id/prices
     stripe.prices.list({ product: "prod_Xyz", active: true, type: "recurring" })
        ↓ mapStripePriceToOption
        ↓ sort by amount
        ↓ sortOptionsByPlanSelectorPriceOrder   (editor drag order)
        ↓ filterOptionsByPlanPriceEnabled       (editor toggles)
        ↓ enrichPriceOptionsWithCouponDisplay   (strikethrough price)
        ↓ trialDaysForCheckoutPrice             ("7-day trial" copy)

⑥ PlanStep renders the card
```

**Four filters sit between Stripe and the screen.** A price that exists in Stripe but doesn't appear
on the paywall has failed one of them — walk them in order:

| Not showing? | Check |
|---|---|
| Price is one-off, not recurring | `type: "recurring"` in the list call excludes it |
| Price is archived in Stripe | `active: true` excludes it |
| Wrong product | `paywall.stripeProductId` |
| Toggled off in the editor | `planSelectorUi` → `filterOptionsByPlanPriceEnabled` |
| Only `stripePriceId` is set | Single-price mode — the product's other prices are never listed |
| Trial not showing | Per-price rows exist for *other* prices → legacy trial disabled for all (see Step 3.8) |

---

## Failure modes

| Symptom | Cause | Where to look |
|---|---|---|
| `503 stripe_unconfigured` | No `site.stripeSecretKey` and no `STRIPE_SECRET_KEY`; or `X-Public-Host` missing so the Site never resolved | `stripeSecretForPublicHost()` |
| `404 no_paywall` | `flow.paywallId` is null — or the exit variant for an exit-paywall request | `prices()` step 2 |
| `422 no_stripe_price` | Paywall has neither `stripeProductId` nor `stripePriceId` | `prices()` tail |
| `422 price_disabled` | The requested price is toggled off in `planSelectorUi` | `isPlanPriceEnabled()` |
| Only card, no wallets | PMC fetch threw → `catch` returns `embeddedMethodIds: ["card"]`; or the paywall has express off; or Stripe has the method off | `resolvePaywallPaymentConfig()` |
| Wallets never appear on the plan step | `walletsVisibleByPriceId[selected] === false` — the device genuinely can't offer them (no Apple Pay on desktop Chrome) | `handleWalletAvailabilityChange` |
| Trial silently missing | A per-price row exists for some *other* price → legacy trial disabled everywhere | `trialDaysForCheckoutPrice()` |
| Checkout iframe blank | Two Embedded Checkouts raced; the 100 ms defer or `checkoutKey` bump misfired | `embedMountAllowed` effect |
| Paid, but the thank-you screen doesn't stick | `sessionStorage` lost across the redirect (in-app browser) | `flow_paid_{id}` + the `trackingSessionId` self-heal |
| Webhook 400s | Signature mismatch — wrong `whsec_`, or the raw body was consumed | `main.ts` `express.json({ verify })` |
| Purchase counted twice in Meta | Pixel + CAPI not deduplicating — both must use the Checkout Session id as event id | `sendPurchaseForCheckoutSession` |

---

## File index

| File | Role |
|---|---|
| [components/survey/SurveyRunner.tsx](../../frontend/src/components/survey/SurveyRunner.tsx) | Funnel state machine; decides when the paywall shows; fires most events |
| [components/survey/PaywallScreen.tsx](../../frontend/src/components/survey/PaywallScreen.tsx) | Prices, plan step, wallet panels, embedded checkout |
| [components/survey/PaywallCheckoutPanel.tsx](../../frontend/src/components/survey/PaywallCheckoutPanel.tsx) | `ExpressCheckoutElement` wrapper |
| [components/start/StartFlowHost.tsx](../../frontend/src/components/start/StartFlowHost.tsx) | Post-redirect `payment-confirm` |
| [lib/paywall-preload-cache.ts](../../frontend/src/lib/paywall-preload-cache.ts) | Break-screen warm-up |
| [lib/flow-checkout-body.ts](../../frontend/src/lib/flow-checkout-body.ts) | Shared checkout POST body |
| [lib/plan-analytics.ts](../../frontend/src/lib/plan-analytics.ts) | Plan naming for events |
| [paywall-themes/index.ts](../../frontend/src/paywall-themes/index.ts) | Layout registry — `getPaywallTheme()` |
| [flows/flows.service.ts](../../backend/src/flows/flows.service.ts) | `prices()`, `checkout()`, `confirmPayment()` |
| [paywalls/stripe-payment-methods.util.ts](../../backend/src/paywalls/stripe-payment-methods.util.ts) | PMC → express/embedded method ids |
| [paywalls/stripe-price-discounts.util.ts](../../backend/src/paywalls/stripe-price-discounts.util.ts) | Coupon display + checkout discounts |
| [webhooks/site-stripe-inbound-webhook.controller.ts](../../backend/src/webhooks/site-stripe-inbound-webhook.controller.ts) | Per-site webhook ingest |
