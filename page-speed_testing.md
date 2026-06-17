# Page Speed Changes — Testing & Production Rollout Guide

This is the companion to [page-speed-optimization.md](page-speed-optimization.md). It covers **everything that changed**, **how each part works now**, **exactly what to test and where**, and **what to do before/during production rollout** (especially the image migration + backfill).

Read the image section carefully — it changed the data model and needs a migration + backfill in every environment.

---

## TL;DR — what changed

| # | Change | Risk | Needs prod action? |
|---|--------|------|--------------------|
| 1 | **Optimistic SSR render** (no more black screen) | Low | No |
| 2 | **`start/loading.tsx`** streaming fallback | None | No |
| 3 | **Theme code-split** (lazy registry) | Low | No |
| 4 | **Images: base64 → file (`/api/images/:id`)** | Medium | **YES — migration + backfill** |
| 5 | **Defer GTM / Meta Pixel** | Low | Verify tracking still fires |
| 6 | **Split Stripe/Paywall out of SurveyRunner** | Low | No |
| 7 | **Font `display: "swap"`** | None | No |
| 8 | **Flow resolution caching (60s)** | Medium | Know your instance count |

Test page used for all PageSpeed checks: `https://staging.funnelwork.ai/start?flow_id=<id>` (mobile).

---

## 1. Optimistic SSR render

### How it works now
`StartFlowHost` used to render `null` (blank/black screen) until a client-side A/B resolve finished. Now it renders the **server-resolved entry flow immediately**, then swaps to the A/B variant in the background if there is one. The payment-verify path (`?payment=cs_...`) is unchanged — it still shows the "Restoring your progress…" loader.

### What to test
1. Open `/start?flow_id=<id>` on a **throttled** mobile connection (DevTools → Network → Slow 4G).
   - ✅ Landing screen appears **immediately** — no black/blank gap.
2. **Flow with NO experiment** → page paints once, nothing changes. No flicker.
3. **Flow WITH an experiment** (entry flow is a target):
   - You may briefly see the entry (control) flow, then it swaps to the variant flow. This is expected.
   - If entry and variant landers look identical → no visible change.
4. **Payment return**: complete a Stripe checkout so you come back with `?payment=cs_...`.
   - ✅ Shows the loader, then the correct post-payment screen. (Must NOT optimistically render the survey here.)
5. **Back button** during loading still works (sentinel / exit-paywall logic unchanged).

### Pass criteria
No black screen; A/B still assigns the same variant per visitor (sticky); payment flow unaffected.

---

## 2. `start/loading.tsx`

### How it works now
While the server is resolving the flow (`await resolveFlow` in `start/page.tsx`), Next.js streams a neutral spinner instead of nothing.

### What to test
- Hard-refresh `/start?flow_id=<id>` (Cmd+Shift+R) with a throttled connection → you may briefly see a centered gray spinner, then the real landing screen.
- Most visible on cold starts / slow API. On a fast local API the window is tiny.

### Pass criteria
No regression; spinner is neutral (not wrongly themed), then replaced by the real screen.

---

## 3. Theme code-split

### How it works now
`themes/index.ts` used to statically bundle all 6 themes. Now `default` + `dark` stay eager (tiny; `default` is the fallback) and the 4 heavy themes (`chat`, `insta_a/b/c`) load as **separate chunks** only when a flow uses them.

### What to test
For each theme, open a flow that uses it and confirm it renders correctly:
- `default`, `dark`, `chat`, `insta_a`, `insta_b`, `insta_c`.
- In DevTools → Network (JS), confirm a flow on `default` does NOT download the `chat`/`insta_*` chunks.
- Switch a flow's theme in the admin and re-open `/start` → correct theme renders.

### Pass criteria
Every theme still renders identically; unused theme JS is not downloaded.

---

## 4. Images: base64 → file (THE BIG ONE)

### How it works now — the full picture

**Before:** images were stored as base64 text inside the flow/template/question rows and shipped inside the flow JSON on every visit.

**Now:** images are uploaded to the existing `POST /api/images` endpoint, stored as real files in the `image` table, and the template/question stores only an **id**. They render via `/api/images/:id` (served with a 1-year immutable cache header).

**Backward compatibility:** every render prefers the id, and **falls back to base64** when no id is set. So old/un-migrated rows keep working.

#### The 5 migrated image fields
| Field | Lives on | Where it shows |
|-------|----------|----------------|
| `logoImageId` | template | Top banner (landing / questions / thank-you) |
| `surveyBackgroundImageId` | template | Full-page survey background |
| `landingHeroImageId` | template | Landing hero (the **LCP** image) |
| `paywallHeroImageId` | template | Paywall hero (hero layout) |
| `questionHeroImageId` | question | Image above a question |

#### How resolution works (the rule)
Central helper `resolveImageSrc(id, base64)` in `frontend/src/lib/image-src.ts`:

```
if id is set      → use  /api/images/:id     (kind: "url")
else if base64    → use  data:...;base64,... (kind: "base64")
else              → null (render nothing)
```

Example for the landing hero:
- New flow: `landingHeroImageId = 42` → renders `<img src="/api/images/42">` (cached 1 year).
- Old flow not yet backfilled: `landingHeroImageId = null`, `landingHeroImageBase64 = "data:..."` → renders the base64 inline (still works).

Upload limits:
- **Editor UI:** 750 KB max per image (template + question heroes).
- **Backend `/api/images`:** 2 MB hard cap; allowed types **jpeg, png, webp, gif, svg**.
- **SVG note:** SVGs render via a plain `<img>` (never `next/image` optimization), so we never enable `dangerouslyAllowSVG` (XSS-safe).

### What to test — UPLOAD (admin side)

Go to the dashboard editors:
- **Template images** → `/templates/<id>/edit` (Branding section): top banner, survey background, landing hero, paywall hero (paywall hero only shows when paywall theme uses a hero).
- **Question hero** → survey builder → a question → "Image above question".

For **each** uploader:
1. Upload a normal PNG/JPG (< 750 KB).
   - ✅ Preview appears.
   - In DevTools → Network, confirm a `POST /api/images` returns `{ id, url }`.
2. **Save** the template/survey.
   - In DevTools, the save `PATCH /api/templates/<id>` (or `/api/surveys/<id>`) body should contain `landingHeroImageId: <number>` (etc.) — NOT a giant base64 string.
3. Reload the editor → image still shows (id round-tripped from DB).
4. **Remove** the image → preview clears; save → field is nulled.
5. **Too big**: upload a > 750 KB file → UI alert, no upload.
6. **Bad type / too big at backend**: (optional) a > 2 MB or unsupported type → server error surfaced as an alert.
7. **SVG**: upload an SVG logo → renders (via plain `<img>`).

### What to test — RENDER (public side)

Open `/start?flow_id=<id>` for a flow whose template/questions have images:
1. Landing hero, top banner, background all show correctly.
2. DevTools → Network: the images load from `/api/images/<id>` with response header `Cache-Control: public, max-age=31536000, immutable`.
3. **Reload** the page → images are served from browser cache (Network shows "(disk cache)" / 304), NOT re-downloaded.
4. Question images show as you advance through the survey.
5. Reach the paywall (hero layout) → paywall hero image shows.
6. **Old/un-migrated flow** (one with base64 but no backfill yet): images still render (base64 fallback). Confirm nothing is broken.

### What to test — BACKFILL (existing data)

Script: `backend/scripts/backfill-image-ids.ts`. It converts existing base64 → `image` rows → sets ids. It does **not** delete base64 (safe + reversible) and is **idempotent**.

```bash
# from repo root, with backend env loaded
set -a && . ./backend/.env && set +a

# 1) Dry run — reports counts, writes nothing
npx tsx backend/scripts/backfill-image-ids.ts --dry-run

# 2) Real run
npx tsx backend/scripts/backfill-image-ids.ts

# 3) Re-run — should be all skipped (idempotent)
npx tsx backend/scripts/backfill-image-ids.ts
```

Expected output shape:
```
templates: migrated=N skipped=M failed=0
questions: migrated=N skipped=M failed=0
```

Verify in DB:
- `SELECT id, landing_hero_image_id, logo_image_id FROM template WHERE landing_hero_image_id IS NOT NULL;` → ids populated.
- `SELECT COUNT(*) FROM image;` → grew by the number migrated.
- Open the corresponding `/start?flow_id=<id>` → images now load from `/api/images/:id` instead of base64.

> Already run on the **local** dev DB during development (6 images migrated, 0 failed, re-run skipped all). Staging/prod still need it.

### Pass criteria
Upload → save → reload → render all work; new images are URLs + cached; old images fall back to base64; backfill is idempotent and reports 0 failed.

---

## 5. Defer GTM / Meta Pixel

### How it works now
`SurveyRunner` used to inject GTM (`googletagmanager.com/gtm.js`) and Meta Pixel (`connect.facebook.net/fbevents.js`) immediately on mount. Now it waits for the **first user interaction** (tap/scroll/keydown) OR a `requestIdleCallback` / 2s fallback — whichever comes first. PageView still fires; `initBrowserMarketingTags` is idempotent. Amplitude was already lazy and is unchanged.

### What to test
1. Open `/start?flow_id=<id>` with a flow that HAS a GTM container / Pixel id configured (on its Site).
2. DevTools → Network: on **initial paint**, `gtm.js` / `fbevents.js` should **NOT** load yet.
3. Tap/scroll the page → they load now.
4. If you don't interact, they still load within ~2–3s (idle/timeout fallback).
5. **PageView**: use Meta Pixel Helper / GTM Preview → confirm a PageView still fires.
6. Conversion events (e.g. purchase) still fire on the paywall/checkout.

### Pass criteria
Tags don't load on first paint; they DO load on interaction or shortly after; PageView and conversions still tracked.

---

## 6. Split Stripe / Paywall out of SurveyRunner

### How it works now
`PaywallScreen` (which imports the Stripe SDK), `ThankYouScreen`, and `PostPaymentEmailCapture` are lazy-loaded via `next/dynamic`. They only download when the user reaches that stage — Stripe is no longer in the initial `/start` bundle.

### What to test
1. Open `/start?flow_id=<id>` → DevTools Network: Stripe (`js.stripe.com`) and the paywall chunk are **not** loaded on the landing screen.
2. Advance to the paywall → the paywall chunk + Stripe load now; a brief spinner may show.
3. Complete a checkout → Stripe Elements / Express wallets work as before.
4. Thank-you screen + post-payment email capture (for wallet payments with no email) still render.

### Pass criteria
Stripe loads only at the paywall; full checkout (card + wallets) works; thank-you/email-capture unaffected.

---

## 7. Font `display: "swap"`

### How it works now
`Inter` now loads with `display: "swap"` — fallback text shows immediately instead of blank text while the font loads.

### What to test
- Throttle network, load `/start` → text is visible immediately (may briefly be a system font, then swaps to Inter).
- No layout breakage from the swap.

### Pass criteria
No invisible-text flash; no major layout shift.

---

## 8. Flow resolution caching (60s)

### How it works now
`resolveFlow` / `resolveFlowBySlug` used `cache: "no-store"` (fresh API+DB call every visit). Now they use the Next.js Data Cache with `revalidate: 60`. Cache keys are **host-scoped** (a `_host` query param + `flow-<host>-<id>` tags) — critical for multi-tenancy.

### ⚠️ The consistency tradeoff (READ THIS)
Admin flow edits happen in the **NestJS** backend, which has no hook into the **Next.js** cache. So an edit can take **up to 60 seconds** to appear on the public `/start` page. This is intentional (bounded eventual consistency, no cross-stack wiring). To make edits instant later, have the backend call a Next route handler that runs `revalidateTag(flowCacheTag(host, id))`.

### What to test
1. **Caching works**: open `/start?flow_id=<id>` twice within 60s → second load is faster / TTFB lower (served from cache).
2. **Edit visibility**: edit a flow's landing headline in the admin, save, then open `/start` → the change appears **within 60s** (may not be instant). Confirm it does appear after the window.
3. **Multi-tenant safety (important)**: if you have two hosts/sites with the **same** `flow_id` resolving to **different** content, open both:
   - ✅ Each host shows ITS OWN flow. They must NOT share a cached entry. (This is the leak we guarded against.)

### Pass criteria
Repeat visits are cached; edits appear within 60s; **no cross-host content bleed**.

### How to revert if 60s staleness is unacceptable
In `frontend/src/lib/flow-resolver.ts`, swap the two `next: { revalidate... }` blocks back to `cache: "no-store"`. The helper functions are harmless if left.

---

## Production rollout checklist

Do these **in order** per environment (staging first, then prod).

### A. Pre-deploy
- [ ] Confirm **how many Next.js instances** run in prod.
  - **1 instance** → flow caching is fully fine.
  - **Multiple instances** → caches are per-instance; the 60s window applies per instance, and there's no shared invalidation (would need Redis + a revalidate endpoint). Decide if 60s eventual consistency is acceptable; if not, revert item 8 before deploy.
- [ ] Review the image migration SQL: `backend/prisma/migrations/20260617120000_add_image_id_columns/migration.sql` (5 `ADD COLUMN` statements, all nullable — non-destructive).

### B. Deploy
- [ ] Deploy code (frontend + backend).
- [ ] **Apply the migration** on the target DB:
  ```bash
  npx prisma migrate deploy
  ```
  (If the shadow-DB/baseline issue blocks it as it did locally, apply the 5 `ADD COLUMN` statements directly — they're nullable and safe.)
- [ ] Regenerate Prisma client if your deploy doesn't do it automatically (`npx prisma generate`).

### C. Post-deploy (before backfill)
- [ ] Smoke test `/start?flow_id=<id>` for a few live flows → everything renders (old images via base64 fallback). The site must be fully working **even before** the backfill runs.
- [ ] Upload a new image in the admin → confirm it stores as an id and renders via `/api/images/:id`.

### D. Backfill (migrate existing images)
- [ ] **Dry run** first:
  ```bash
  set -a && . ./backend/.env && set +a
  npx tsx backend/scripts/backfill-image-ids.ts --dry-run
  ```
- [ ] Review counts; `failed` should be 0 (any failures still render from base64 — no data loss, but investigate).
- [ ] **Real run**:
  ```bash
  npx tsx backend/scripts/backfill-image-ids.ts
  ```
- [ ] Spot-check a few migrated flows on `/start` → images now load from `/api/images/:id`.
- [ ] Re-run the script → should report all `skipped` (idempotent).

### E. Verify tracking & payments (don't skip)
- [ ] GTM PageView + a conversion event still fire (item 5).
- [ ] A full Stripe checkout (card + at least one wallet) completes (item 6).
- [ ] Payment-return screen (`?payment=cs_...`) works (item 1).

### F. Things to watch in prod
- [ ] **DB size**: images now live as blobs in the `image` table. Watch DB growth / backup size. If it grows a lot, the next step is a CDN in front of `/api/images/*` or moving blobs to object storage (S3/GCS).
- [ ] **`/api/images/:id` load**: first request per image hits the DB (then cached 1 year by the browser). A CDN in front of this path removes most DB hits for cold visitors.
- [ ] **Flow edit latency**: remember the up-to-60s window for public visibility of flow edits (item 8).

### G. Rollback notes
- **Images**: base64 columns are **kept** — render falls back automatically. To fully roll back, revert the frontend render changes; data is intact. The new `*ImageId` columns are nullable and can stay.
- **Flow caching**: one-line revert to `cache: "no-store"` (item 8).
- **Everything else** (P0, loading, themes, defer-tags, Stripe split, font) is frontend-only and reverts cleanly with no data implications.

---

## Known unrelated issue (pre-existing, not from this work)

`npm run build` fails at the TypeScript check on `frontend/src/app/api/auth/[...nextauth]/route` with a `NextRequest` type mismatch. This is caused by a **duplicate `next` install** (there's a nested `frontend/package-lock.json` + `frontend/node_modules` alongside the root). It exists on the branch independent of these changes. The app **compiles** fine (`✓ Compiled successfully`) — only the type-check step trips. Resolve by removing the nested lockfile/`node_modules` so there's a single `next` install. Not part of the page-speed work, but it will block a clean production build until fixed.
