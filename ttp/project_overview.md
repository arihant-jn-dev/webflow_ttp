# Platform Flow: How Everything Works

## What Is This Platform?

This is a **listicle serving and monetization platform**. A *listicle* is a ranked list article — "Top 10 Air Fryers of 2026", "Best Budget Laptops Under $500", etc. Each listicle shows a list of products with descriptions, rankings, and a button that sends users to a merchant (like Amazon) to buy. When a purchase happens, we earn an **affiliate commission**.

The platform runs multiple such websites from a single codebase. Each website is a *site* or *brand* with its own domain, design, and product catalog.

---

## Glossary of Terms

| Term | What it means |
|------|---------------|
| **Listicle** | A ranked list article (e.g. "Top 10 Blenders") — the core content unit |
| **Product** | An item in the listicle, usually an Amazon product (has ASIN, title, price, image, description) |
| **ASIN** | Amazon Standard Identification Number — the unique ID Amazon uses for every product |
| **Affiliate tag** | A tracking code appended to Amazon links (e.g. `tag=mytag-20`) that identifies us so Amazon pays commission |
| **CTA** | Call To Action — the button on a product card (usually "VIEW ON AMAZON") |
| **CTA URL** | The affiliate link the CTA button points to |
| **Slug** | The URL-friendly identifier for a listicle (e.g. `best-air-fryers-2026`) |
| **Site** | A brand/domain unit in the platform (e.g. top10picks.co) |
| **Domain** | The hostname (`top10picks.co`) that maps to a Site |
| **Template** | A React component layout for rendering a listicle page — different templates = different visual designs |
| **Experience Config** | A JSON blob that controls template, colors, layout for a visitor's session |
| **UTM** | Urchin Tracking Module — URL params (like `utm_source`, `utm_exp_id`) used to track traffic source |
| **utm_exp_id** | Our custom parameter that identifies which "experience" (template/layout/colors) a visitor should see |
| **A/B Test** | Showing different template variants to different users to see which one converts better |
| **Mapping** | The relationship between a listicle and a product — defines rank, badge, CTA text |
| **Override** | Per-listicle JSON that deep-merges on top of the base product data (used to customize without editing the product itself) |
| **Multi-tenant** | One codebase, many brands — the same backend and frontend serve all sites |
| **SSR** | Server-Side Rendering — the page HTML is generated on the server before reaching the browser (important for SEO) |
| **Middleware** | Code that runs before every page request — used to resolve experience config |
| **Badge** | A label on a product card like "BEST VALUE" or "EDITOR'S CHOICE" |
| **Listicle Metrics** | Impressions, clicks, and conversions tracked per template per day |
| **Cookie ID** (`tmm_cookie_id`) | A unique anonymous ID assigned to each visitor for tracking |
| **Homepage Builder** | An admin tool to arrange blocks (components) on the home page of a site |
| **GTM** | Google Tag Manager — fires analytics and marketing tags client-side |
| **Meta Pixel** | Facebook/Instagram tracking pixel for ad attribution |

---

## How We Make Money

The business model is **affiliate marketing**:

1. A user reads a listicle article on our site
2. They see a product they like and click "VIEW ON AMAZON"
3. They land on Amazon with our affiliate tag in the URL (e.g. `?tag=mytag-20`)
4. If they buy *anything* on Amazon within 24 hours, Amazon pays us a commission (typically 1–10% of the sale price)

The platform is optimized end-to-end to maximize **clicks on CTA buttons** and ultimately **conversions** (purchases). Every feature — A/B testing, experience configs, template variants — exists to increase that click rate.

The **CTA URL** for each product is configurable per site. It can point to Amazon (via ASIN), Shopnomics, or any other merchant using a URL template with macros. This flexibility means the same content can be monetized through different merchant programs without code changes.

---

## The Full Request Flow

### Step 0: Who sets up the content?

Before any user sees anything, an admin (us) has:
1. Created a **Site** record in the database for the brand
2. Mapped a **domain** to that site
3. Imported or created **listicles** with ranked **products**
4. Set up the **CTA config** (which merchant, what affiliate tag)
5. Configured **experience configs** for different traffic sources
6. Set up **templates** (visual designs) for the site

This is all done via the `/admin` panel in the frontend.

---

### Step 1: User Arrives (Browser → Next.js Middleware)

A user clicks a link or types a URL like:
```
https://top10picks.co/l/best-air-fryers-2026?utm_exp_id=holiday-sale
```

Before the page component even starts rendering, **Next.js middleware** runs:

1. **Domain resolution**: Reads the `Host` header (`top10picks.co`). Passes this to the backend.

2. **UTM Experience ID resolution** — 3-tier priority:
   - First, check if `utm_exp_id` is in the URL (`holiday-sale` in the example above)
   - If not in URL, check `localStorage` (was it stored from a previous visit?)
   - If not in localStorage, check the `utm_exp_id` cookie
   - If found in URL, automatically persist it to both localStorage and a 30-day cookie (so the user maintains the same experience across the whole session)

3. **Experience config fetch**: Calls `POST /user-experience` on the NestJS backend with the domain and resolved `utm_exp_id`. The backend looks up the experience config JSON for that ID (or falls back to the site's default config).

4. **Attach config to request**: Sets `x-experience_config` as a request header and sets `tmm_cookie_id` (a unique visitor ID) and encrypted `tmm_cookie_data` as cookies.

Now the page component runs with that config already available.

---

### Step 2: Page Renders (Next.js SSR → NestJS API)

The listicle page component (`/app/l/[slug]/page.tsx`) runs **server-side**:

1. **Fetch listicle data**: Calls `GET /api/listicle/:slug` on the backend.

   The backend does a lot here:
   - Looks up the listicle by slug
   - Loads all active product mappings (which products are in this list, in what rank order)
   - Applies any **product overrides** (per-listicle JSON that customizes a product's data without editing the product itself)
   - Runs `deepMerge` to combine base product data + overrides
   - Resolves the **CTA URL** for each product using the site's URL template and macros
   - Picks a **template** for this listicle (see Template A/B section below)
   - Returns the full payload: `{ ok: true, listicle, products, template }`
   - This payload is **cached** — subsequent requests for the same slug return the cached version without hitting the DB

2. **Template loading**: The page reads the `x-experience_config` header (set by middleware) and the `template_key` from the API response. It then does a dynamic import to load the correct React component from:
   ```
   app/domains/top10picks-co/templates/listiclepage/{template_key}.tsx
   ```
   If that doesn't exist, it falls back to a default template path.

3. **Render HTML**: The chosen template component receives the listicle and products as props and renders the full HTML.

4. **Send to browser**: The browser receives a fully-rendered HTML page with all product data, images, and copy already in it.

This SSR approach is critical for SEO — search engine crawlers see the full content immediately.

---

### Step 3: Browser Loads (Client-Side Hydration)

React "hydrates" the server-rendered HTML — it attaches event listeners without re-fetching data. Now the page is interactive.

Several client-side systems activate:

#### Tracking
- The `useTracking` hook starts listening for clicks on elements with `tmm-data-track="ComponentName"` attributes
- An impression event is fired via `GET /api/track/pixel`
- Click events are sent via `POST /api/track/beacon`
- Events carry the `tmm_cookie_id` for anonymous user identification
- **Google Tag Manager** fires from the site's GTM container ID (configured per site)
- **Meta Pixel** fires for ad attribution (Facebook/Instagram), reading `_fbc` and `_fbp` cookies

#### Experience ID Persistence
- The `getUtmExpId()` utility runs to ensure the experience ID is persisted in localStorage and cookie if it came in via URL

---

### Step 4: User Clicks a Product (The Money Moment)

The user reads the list and clicks "VIEW ON AMAZON" on, say, the #2 ranked blender.

1. **CTA URL** — The link was built server-side by the backend using the site's CTA config:
   ```
   https://www.amazon.com/dp/B09XYZ123?tag=top10picks-20
   ```
   - `B09XYZ123` is the product's ASIN
   - `tag=top10picks-20` is our affiliate tag

2. **Click tracking** — Before the redirect, the tracking system fires a click beacon to `/api/track/beacon`. Template A/B metrics are incremented for this template.

3. **User lands on Amazon** — Amazon records the affiliate tag. If the user buys anything within 24 hours, we get commission.

---

### Step 5: Admin Monitors and Optimizes

Back in the admin panel:

- **Template metrics** show which template variant is getting more clicks (impressions vs clicks per template per day)
- **A/B tests** can be adjusted: shift allocation weights to send more traffic to the winning template
- **Cache invalidation**: After editing a listicle or product, the admin hits the cache bust endpoint so the next request gets fresh data
- **Experience configs**: New `utm_exp_id` experiments can be created, each pointing different UTM traffic sources to different templates/layouts/colors

---

## Multi-Site Architecture: One Codebase, Many Brands

The platform serves multiple websites simultaneously. Here's how they stay isolated:

```
top10picks.co  →  domain_master (domain_name = "top10picks.co") 
                      ↓
               sites_master (id=1, site_config JSON)
                      ↓
               app/domains/top10picks-co/templates/...

supershoppr.com → domain_master (domain_name = "supershoppr.com")
                      ↓
               sites_master (id=2, site_config JSON)
                      ↓
               app/domains/supershoppr-com/templates/...
```

Each site has its own:
- **Templates** (visual design)
- **CTA config** (which merchant, what affiliate tag)
- **Marketing tags** (GTM container ID, Meta Pixel ID)
- **Experience configs** (A/B test setups)
- **Default experience** (what a visitor gets with no UTM parameter)
- **Homepage layout** (configured via the homepage builder)

But they share:
- The same NestJS backend
- The same Prisma database
- The same tracking infrastructure
- The same caching layer
- The same deployment pipeline

---

## Template A/B Testing: How It Works

This is how we optimize click-through rates without deploying new code:

1. **Register templates**: Each visual design is a React component. Its `template_key` is saved in the `templates` DB table.

2. **Create variants**: In `listicle_template_variants`, assign allocation weights per listicle:
   - Template A: weight 70 (70% of visitors)
   - Template B: weight 30 (30% of visitors)

3. **Assignment**: When a visitor loads a listicle for the first time:
   - The backend checks if a `chosen_template_{listicleId}` cookie exists
   - If not, it does a weighted random selection
   - Sets a 30-day cookie so the visitor always sees the same template (consistency)
   - Admin can preview any template with `?preview_template=template_key`

4. **Metrics tracking**: Every impression, click, and conversion is recorded in `listicle_template_metrics` per day per template. This shows which template converts better.

5. **Winner takes more**: Shift allocation weights to 90/10 once a winner is clear.

---

## Experience Config: Personalizing Per Traffic Source

The experience config system lets us show completely different pages to visitors arriving from different sources — without them ever knowing.

**Example**:
- Regular organic visitor → default template, standard layout
- Visitor from a holiday email campaign (`utm_exp_id=holiday-2026`) → holiday-themed template, red/green colors, "Gift Guide" badge style
- Visitor from a "best budget" social ad (`utm_exp_id=budget-audience`) → budget-focused template, price-prominent layout

The experience config JSON controls:
- Which template to load (`listiclePageTemplate`, `homePageTemplate`, etc.)
- Color scheme (primary, secondary, accent colors)
- Typography (heading and body fonts)
- Feature flags (show/hide badges, prices, animations)
- CTA button label and style
- Any other UI customization the template supports

This all happens in middleware *before* SSR — the page renders correctly on the first request with zero client-side flicker.

---

## Homepage Builder

Each site's home page is assembled from **components** (blocks) configured by an admin — no code deployment needed.

The admin picks:
- Which component types to show (featured listicles, category grids, hero banner, etc.)
- Which specific listicles appear in each component
- The order of components on the page

The `GET /api/homepage/config` endpoint returns the assembled layout, which the homepage template renders.

---

## Data Import Flow

New listicle content enters the system via JSON import:

1. JSON files dropped into `data/listicle/`
2. `npm run import:listicles` (or via `/api/admin/import` in the admin panel) reads the JSON and upserts:
   - Listicle record
   - Product records
   - Mapping records (which products, what rank, what badge)
3. Cache is busted
4. The new listicle is immediately live at `/l/{slug}`

---

## Summary: What Happens on Every Request

```
User URL
   │
   ▼
Next.js Middleware
   ├── Resolve domain → site
   ├── Resolve utm_exp_id (URL → localStorage cookie → cookie)
   ├── POST /user-experience → get experience config JSON
   └── Set x-experience_config header + tmm_cookie_id

   │
   ▼
Page SSR (server component)
   ├── GET /api/listicle/:slug
   │     ├── Load listicle + mappings
   │     ├── Deep-merge product overrides
   │     ├── Resolve CTA URL per product (affiliate links)
   │     ├── Pick template (cookie → weighted A/B random)
   │     └── Return cached payload
   └── loadTemplate() → dynamic import domain template component

   │
   ▼
HTML sent to browser (fully rendered, SEO-ready)

   │
   ▼
Browser hydrates (React attaches event listeners)
   ├── useTracking activates (impression pixel fires)
   ├── GTM fires (Google Analytics, etc.)
   └── Meta Pixel fires

   │
   ▼
User clicks CTA button
   ├── Click beacon → /api/track/beacon
   └── Redirect to merchant with affiliate tag
             │
             ▼
       Purchase → Commission earned
```
