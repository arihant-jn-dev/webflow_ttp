# Tech Stack: Why Next.js + NestJS and What We Use

## The Big Picture

This platform is a **multi-tenant listicle serving system** — it runs several affiliate commerce websites from one codebase. A visitor arrives on `top10picks.co` or `supershoppr.com`, lands on a "Best 10 Air Fryers" type page, and clicks through to buy a product. We earn a commission on that sale.

Because the same infrastructure serves multiple brands, every layer of the stack had to be:
- **Fast to render** (SEO depends on it)
- **Easy to extend** per-brand without duplicating code
- **Scalable** across domains and traffic spikes
- **Maintainable** by a small team

Next.js (frontend) and NestJS (backend) were chosen because together they cover all of those requirements while keeping the language (TypeScript) consistent end-to-end.

---

## Frontend: Why Next.js

### What is Next.js?
Next.js is a React framework built by Vercel. It adds server-side rendering (SSR), static generation, file-based routing, and a middleware layer on top of React — things React alone does not give you.

### Why it fits this project

| Problem | How Next.js solves it |
|---------|----------------------|
| SEO is critical for affiliate traffic | SSR renders full HTML so Google indexes the page instantly |
| Each brand needs its own look | App Router + domain-aware template folders let each site have its own components without a separate codebase |
| Per-user experience (A/B tests, UTM config) | Middleware runs *before* the page renders, so every SSR response already has the right config baked in |
| Admin panel lives in the same app | `/app/admin/*` routes sit alongside the public pages, sharing the same auth cookies and API client |
| TypeScript everywhere | Next.js is TS-first, matching the backend |

### What we use from Next.js

#### App Router (not Pages Router)
We use the newer **App Router** (`/app` directory). Every route is a folder with a `page.tsx` and optional `layout.tsx`. This gives us:
- **Server Components** by default — data fetching happens on the server with no client JS bundle cost
- **Layouts** that share chrome (headers, navbars) across routes without re-rendering

Key routes:
```
/l/[slug]          → Listicle page (primary product list)
/a/[slug]          → Article-style listicle variant
/c/[category]      → Category listing
/admin/*           → Full admin panel
/app/domains/*/    → Per-brand template components
```

#### Middleware (`frontend/middleware.ts`)
Next.js middleware runs on the Edge *before* any page renders. We use it to:
1. Read the `Host` header to identify which site/domain is being requested
2. Read `utm_exp_id` from the URL, localStorage-persisted cookie, or encrypted cookie
3. Call the backend `/user-experience` API to get the experience config JSON for this visitor
4. Attach the config as a request header (`x-experience_config`) that SSR page components then read

This is the key architectural decision that makes A/B testing and per-brand templates work transparently — the page never has to know how to find its config, the middleware just puts it there.

#### Template System
Each brand has its own folder under `frontend/app/domains/`. For example:
- `app/domains/top10picks-co/templates/listiclepage/default.tsx`
- `app/domains/supershoppr-com/templates/homepage/hero-v2.tsx`

The `template-loader.ts` utility reads the experience config to know *which* template key to load, then does a dynamic import. If the template doesn't exist, it falls back gracefully to a static path. This means:
- New brands get their own design without touching other brands' code
- A/B tests are just different template keys with different allocation weights

#### Data Fetching
- **SSR pages** use `cache: 'no-store'` for listicle data — content must always be fresh
- `backendFetch()` in `src/lib/api-client.ts` is the standard way to call the backend from the server
- `adminFetch.ts` is the authenticated version used by admin panel pages
- No client-side data fetching on public pages — everything is in the HTML on first load

#### React 19
We use React 19 which brings improved Server Components, better hydration, and the new `use()` hook for async data in components.

#### Tailwind CSS
All styling uses Tailwind utility classes. No CSS modules or styled-components. This means:
- No naming collisions across domains
- Fast to write, easy for anyone to read
- Per-brand theming is handled at the experience config level (color scheme keys), not in CSS files

---

## Backend: Why NestJS

### What is NestJS?
NestJS is a Node.js framework that brings **Angular-style architecture** (modules, controllers, services, decorators, dependency injection) to the backend. It sits on top of Express and adds structure, TypeScript support, and a large ecosystem of official packages.

### Why it fits this project

| Problem | How NestJS solves it |
|---------|---------------------|
| Many features (listicles, tracking, auth, homepage builder, A/B tests) | Module system keeps each feature isolated with its own controller + service |
| Admin routes must be protected | `AuthGuard` decorator on entire controller classes, no per-route boilerplate |
| Caching is critical for performance | `@nestjs/cache-manager` integrates cleanly with `@CacheKey` and `@CacheTTL` decorators |
| TypeScript on both ends | NestJS is TS-native; shared types/interfaces are consistent |
| Future-proof | Dependency injection makes swapping implementations (e.g. DB, cache) easy |

### What we use from NestJS

#### Module Architecture
Every feature is a **module** — a self-contained unit with:
- `*.module.ts` — declares what's in the module and what it imports
- `*.controller.ts` — HTTP routes (GET, POST, PATCH, DELETE)
- `*.service.ts` — business logic
- Optional DTOs for request validation

Modules we have:
```
listicle/              → Public listicle API (the core)
domain/                → Domain → site resolution
user-experience/       → UTM experience config lookup
tracking/              → Event pixels and beacons
homepage/              → Public homepage config
abtest/                → A/B test assignment
auth/                  → Admin login/logout (JWT)
admin/admin-listicles/ → CRUD for content
admin/admin-products/  → CRUD for products
admin/admin-categories/
admin/admin-sites/     → Per-site config including CTA and marketing tags
admin/admin-homepage/  → Homepage builder
admin/admin-tracking/  → Tracking config
admin/admin-abtest/    → A/B test management
```

#### `@Global()` PrismaModule
Prisma (our ORM) is wrapped in a `@Global()` module. This means you just inject `PrismaService` in any service file — you don't have to import `PrismaModule` everywhere. It's a single database connection pool shared across all modules.

#### `@UseGuards(AuthGuard)`
Admin controllers are decorated with `@UseGuards(AuthGuard)`. The guard reads the JWT from an httpOnly cookie and validates it. If invalid, returns 401. This is applied once at the controller level, not on every method.

#### Caching with `@nestjs/cache-manager`
A global `CacheModule` is registered in `app.module.ts`. The listicle service caches the fully-merged product payload per slug. Domain lookups are cached for 1 hour. This means:
- The DB is not hit on every page load
- Cache keys are managed by `CacheKeyHelper` in `src/config/cache.config.ts`
- Admin can bust the cache via `DELETE /admin/cache` endpoints after editing content

#### Express underneath
NestJS uses Express as its HTTP adapter. This means:
- Standard Express middleware works (e.g. `cookie-parser` for httpOnly cookies)
- 50MB JSON body limit is set in `main.ts` for large import payloads
- CORS is configured in `main.ts` to allow the frontend origin

---

## Database: Prisma + SQLite (dev) / MySQL (prod)

### Why Prisma?
Prisma is a TypeScript-first ORM. You define models in `schema.prisma`, and Prisma generates a fully-typed client. This means:
- No raw SQL strings for CRUD operations
- TypeScript autocomplete for every table and column
- Easy to switch database providers (we literally switch between SQLite and MySQL via a script)

### The SQLite ↔ MySQL Switch
This is a deliberate design choice:
- **Dev**: SQLite — zero setup, just a file (`dev.db`), runs anywhere
- **Prod**: MySQL — handles concurrent writes, has triggers for history, scales horizontally

The `switch-db.js` script edits the `provider` line in `schema.prisma` and regenerates the Prisma client. It's not dynamic at runtime — it's a build-time decision.

### 18 Models

The database is the source of truth for all content and configuration:

| Category | Models |
|----------|--------|
| Sites & Domains | `SitesMaster`, `domain_master` |
| Content | `listicles`, `products`, `authors`, `category_master` |
| Product mapping | `mapping_listicle_products`, `listicle_product_overrides` |
| Templates | `templates`, `listicle_template_variants`, `listicle_template_metrics` |
| Homepage | `homepage_components`, `homepage_component_listicles` |
| Experience / A/B | `user_experience_config`, `ab_test_config`, `ab_test_variations` |
| Analytics | `event_logs`, `event_tracking_config` |

---

## Why TypeScript End-to-End

Both Next.js and NestJS are TypeScript-first. This project uses TypeScript everywhere — from DB schema types (auto-generated by Prisma) to API response shapes to React component props.

Benefits for this codebase:
- Catch bugs at compile time before they hit production
- Refactoring is safe — the compiler tells you every call site that broke
- Onboarding is faster — types document the shape of data without separate docs
- Frontend and backend can share interfaces for API payloads

---

## Infrastructure & Deployment

| Layer | Technology | Why |
|-------|-----------|-----|
| Containerization | Docker (multi-stage builds) | Reproducible builds; same image runs in dev and prod |
| Orchestration | AWS EKS (Kubernetes) | Scales pods independently for backend and frontend |
| Registry | AWS ECR | Stores Docker images; integrates with EKS |
| CI/CD | GitHub Actions (`.github/workflows/deploy.yml`) | Builds and pushes images on push to `main` or `staging` |
| DNS / Routing | Kubernetes Ingress | Routes `top10picks.co` and `supershoppr.com` to the same frontend pods |
| Alternative PaaS | Heroku (Procfile present) | Simpler deploys for staging/demo environments |

### Why Split Images?
`Dockerfile.backend` and `Dockerfile.frontend` build separate Docker images. This means:
- Backend and frontend can be scaled independently (e.g. more backend pods during high API traffic)
- A frontend-only change doesn't rebuild the backend image
- EKS can rolling-deploy just the frontend

### Node 24 LTS
Both images use Node 24 LTS as the base. This is the latest LTS version and includes V8 performance improvements relevant to SSR rendering throughput.

---

## Full Tech Stack Summary

### Frontend
| Tool | Version | Role |
|------|---------|------|
| Next.js | App Router | SSR framework, routing, middleware |
| React | 19 | UI component model |
| TypeScript | — | Language |
| Tailwind CSS | — | Utility-first styling |
| `backendFetch` / `adminFetch` | custom | API client helpers |

### Backend
| Tool | Version | Role |
|------|---------|------|
| NestJS | 11 | API framework |
| Express | (via NestJS) | HTTP server |
| Prisma | 5.20 | ORM + schema management |
| `@nestjs/cache-manager` | — | In-memory caching |
| JWT + httpOnly cookies | — | Admin auth |
| TypeScript | — | Language |

### Database
| Environment | DB | Notes |
|------------|-----|-------|
| Development | SQLite | File-based, zero setup |
| Production | MySQL | Concurrent writes, triggers, performance |

### Infrastructure
| Tool | Role |
|------|------|
| Docker | Containerization |
| AWS EKS | Kubernetes orchestration |
| AWS ECR | Image registry |
| GitHub Actions | CI/CD |
| Heroku (optional) | Simple PaaS deploys |
