# Event Tracking

End-to-end analytics for the AnswerBot chat funnel. Events flow from the
browser (or backend) → Express ingest endpoints → Vector → ClickHouse. The
entire pipeline is **fire-and-forget**: a Vector/ClickHouse outage never slows
chat, page loads, or click-outs.

---

## Table of contents

1. [How it works — the full pipeline](#1-how-it-works--the-full-pipeline)
2. [visitor_id and session_id — what they are and how to look up a user](#2-visitor_id-and-session_id--what-they-are-and-how-to-look-up-a-user)
3. [Event catalog](#3-event-catalog)
4. [What every event carries (shared context)](#4-what-every-event-carries-shared-context)
5. [UTM and click-macro handling](#5-utm-and-click-macro-handling)
6. [File map](#6-file-map)
7. [Configuration](#7-configuration)
8. [ClickHouse table schema](#8-clickhouse-table-schema)
9. [Vector configuration](#9-vector-configuration)
10. [Local development](#10-local-development)
11. [Adding a new event](#11-adding-a-new-event)
12. [A/B experiment extension point](#12-ab-experiment-extension-point)
13. [What is NOT tracked](#13-what-is-not-tracked)

---

## 1. How it works — the full pipeline

```
Browser
  │
  │  navigator.sendBeacon('/api/track/beacon')    ← client-side events
  │  fetch('/api/track/beacon', {keepalive:true}) ← fallback
  │
  ▼
Next.js  app/api/track/beacon/route.ts            ← same-origin proxy
         app/api/track/pixel.gif/route.ts
  │  (forwards body verbatim, returns immediately)
  │
  ▼
Express  POST /api/track/beacon   ←── server-side events call trackEvent() directly
         GET  /api/track/pixel.gif
  │
  │  res.end() / res.status(204).end()   ← browser released HERE
  │  setImmediate(() => { ... })         ← ingest happens AFTER response
  │
  ▼
trackingService.ts
  • anonymise IP  (last IPv4 octet / last 80 IPv6 bits zeroed)
  • parse UA      (device type / browser / OS)
  • stamp timestamps in ClickHouse DateTime64(3) format
  │
  ▼
fluentLogger.ts  →  FluentClient  →  TCP :24224
  │
  ▼
Vector  (Fluent Forward source on 0.0.0.0:24224)
  • batches 100 events OR every 2 seconds
  │
  ▼
ClickHouse  analytics.answerbot_events
```

### Key design rules

| Rule | Reason |
|---|---|
| Always `setImmediate` before ingest | Browser/response never waits for tracking |
| `FluentClient` errors caught and swallowed | Vector outage invisible to users |
| Zod invalid events silently dropped | Never 400 a browser into a retry loop |
| IP anonymised before storage | GDPR compliance |
| `sendBeacon` preferred over `fetch` | Works on page unload; no CORS preflight |
| `requestIdleCallback` on client | Tracking never competes with React rendering |

---

## 2. visitor_id and session_id — what they are and how to look up a user

These are the two identity fields attached to events. They serve different
purposes and have different lifetimes.

### user_visitor_id

| Property | Detail |
|---|---|
| What it is | A UUID (e.g. `d02898e2-8db0-4209-8261-8fb8f9076e66`) |
| Where stored | `localStorage` under key `ab_visitor_id` |
| Lifetime | **Permanent** — survives page refresh, tab close, browser restart. Wiped only if the user clears localStorage |
| Scope | Same browser + same origin. Different browsers or incognito = different ID |
| Who sets it | `frontend/lib/tracking/visitorId.ts` → `getVisitorId()` — creates UUID on first visit |
| Present on | All client-side events (`page_view`, `chat_opened`, `disclosure_shown`, etc.) |
| Missing on | Server-side events (`chat_turn`, `chat_completed`, etc.) — server has no localStorage access |

**Use it to:** track a single user across multiple sessions, measure return visits, build a full funnel from page_view → chat_opened → affiliate_click.

### session_id

| Property | Detail |
|---|---|
| What it is | A 64-char hex token (e.g. `99bc1feb9f09ba4cd416a224eb...`) |
| Where stored | React `useRef` in `useChatbot.ts` — **memory only**, never persisted |
| Lifetime | **Per conversation** — wiped on page refresh or new tab. Each new conversation = new session_id |
| Scope | One browser tab, one chat conversation |
| Who sets it | Backend creates it on first chat message → returns as `X-Session-Token` header → stored in ref |
| Present on | Server-side events always. Client-side events only **after** the first message is sent (ref is null before that) |
| Missing on | Client-side events that fire before the first message (e.g. `page_view`, `disclosure_shown`, early `preset_question_clicked`) |

**Use it to:** group all events within a single conversation, join with the MySQL `conversations` table (`conversations.session_token`).

### Why some events have only one of the two

```
Timeline of a typical user visit:
─────────────────────────────────────────────────────────
page loads      → page_view            visitor_id ✓   session_id ✗ (no chat yet)
panel renders   → disclosure_shown     visitor_id ✓   session_id ✗ (no chat yet)
clicks preset   → preset_question_clicked  visitor_id ✓  session_id ✗ (first msg not sent yet)
sends message   → chat_session_started visitor_id ✗   session_id ✓ (server-side event)
                → chat_turn (user)     visitor_id ✗   session_id ✓
                → chat_turn (bot)      visitor_id ✗   session_id ✓
minimises panel → chat_window_minimised visitor_id ✓  session_id ✓ (session exists now)
clicks affiliate→ affiliate_click      visitor_id ✓   session_id ✓
─────────────────────────────────────────────────────────
```

### How to look up a specific user in ClickHouse

**You have the visitor_id** (e.g. from a support request or a debug session):

```sql
-- Full event history for a visitor
SELECT event_timestamp, event_name, session_id, page_slug, props
FROM analytics.answerbot_events
WHERE user_visitor_id = 'd02898e2-8db0-4209-8261-8fb8f9076e66'
ORDER BY event_timestamp
FORMAT PrettyCompact;
```

**You have the session_id** (e.g. from MySQL `conversations.session_token`):

```sql
-- All events in one chat conversation
SELECT event_timestamp, event_name, user_visitor_id, props
FROM analytics.answerbot_events
WHERE session_id = '99bc1feb9f09ba4cd416a224eb2b8d003db5995c5569ae6016b122a11fe8934a'
ORDER BY event_timestamp
FORMAT PrettyCompact;
```

**Find visitor_id from session_id** (to see their full history across sessions):

```sql
-- Get visitor_id from a known session
SELECT DISTINCT user_visitor_id
FROM analytics.answerbot_events
WHERE session_id = '99bc1feb9f09ba4cd416a224eb...'
  AND user_visitor_id != '';
```

**Find all sessions for a visitor** (to see every conversation they've had):

```sql
SELECT DISTINCT session_id, min(event_timestamp) AS started_at
FROM analytics.answerbot_events
WHERE user_visitor_id = 'd02898e2-8db0-4209-8261-8fb8f9076e66'
  AND session_id != ''
GROUP BY session_id
ORDER BY started_at
FORMAT PrettyCompact;
```

**Where to find session_id in MySQL** (to cross-reference):

```sql
-- MySQL conversations table
SELECT session_token, created_at, page_slug
FROM conversations
WHERE session_token = '99bc1feb9f09ba4cd416a224eb...';
```

The `session_id` in ClickHouse equals `conversations.session_token` in MySQL — they are the same value.

---

## 3. Event catalog

### Server-side events
Emitted directly by `backend/src/chatbot/chatRouter.ts` — no browser round-trip.

| Event | When | Key props |
|---|---|---|
| `chat_session_started` | First message of a new conversation | — |
| `chat_turn` (user) | After user message is persisted | `turn_type: "user"`, `turn_index`, `char_count` |
| `chat_turn` (bot) | After stream completes | `turn_type: "bot"`, `turn_index`, `response_time_ms`, `char_count`, `model` |
| `chat_completed` | After bot stream completes | `total_user_turns`, `total_duration_ms` |
| `chat_error` | OpenRouter failure | `error_type`, `error_message`, `turn_index` |
| `recommendation_shown` | Ad card selected and emitted | `card_ids[]`, `offer_ids[]`, `card_count` |

### Client-side events
Emitted by the browser via `sendBeacon`. All wired — no manual plumbing needed.

| Event | Where wired | When fires | Key props |
|---|---|---|---|
| `page_view` | `UrlParamCapture.tsx` | Every page mount / route change | `referrer` |
| `chat_opened` | `HeroChatPanel.tsx` — FAB/bubble click | User expands the chat from minimised state | `trigger: "button"` |
| `disclosure_shown` | `HeroChatPanel.tsx` — useEffect | Once, when panel first opens and disclaimer is present | `placement: "chat_input_bar"` |
| `preset_question_clicked` | `HeroChatPanel.tsx` — starter chip onClick | User clicks a starter question | `question_text`, `question_index` |
| `chat_window_minimised` | `HeroChatPanel.tsx` — header chevron | User taps the header to collapse panel | — |
| `chat_window_expanded` | `HeroChatPanel.tsx` — header chevron + FAB | User taps header to expand, or FAB to reopen | — |
| `chat_window_closed` | `HeroChatPanel.tsx` — scrim/backdrop tap | User taps the dark backdrop behind the panel | — |
| `affiliate_click` | `AdCard.tsx` — link onClick | User clicks the ad card CTA | `offer_id`, `card_id`, `card_position` |
| `background_content_click` | `StackExchangeSection.tsx` — "View on Stack Exchange" links | User clicks out to Stack Exchange | `link_url`, `destination: "stack_exchange"` |
| `user_response_count` | Not yet wired | — | `total_user_turns` |
| `response_helpful_clicked` | Not yet wired (no rating UI exists) | — | `turn_index`, `rating` |

---

## 4. What every event carries (shared context)

These fields attach to **every** event automatically — never pass them manually.

| Field | Source | Notes |
|---|---|---|
| `user_visitor_id` | `localStorage.ab_visitor_id` | UUID, persists across refreshes. See section 2. |
| `session_id` | `conversation.sessionToken` (React ref) | 64-char hex. Null until first chat message. See section 2. |
| `page_slug` | Passed from the chat hook / UrlParamCapture | e.g. `glp1-weight-loss-E` |
| `site_id` | MySQL sites.id | Passed from page context |
| `page_url` | `window.location.href` | Full URL at time of event |
| `referrer` | `document.referrer` | Empty string if direct |
| `utm_source/medium/campaign/content/term/campaign_id` | `sessionStorage` (captured on landing) | Named columns — easy to filter without JSON |
| `query_params` | `sessionStorage` (full landing query string) | JSON blob — includes `gclid`, `fbclid`, `utm_ad_id`, any custom macro |
| `device_type` | UA parse (server-side) | `mobile` / `tablet` / `desktop` / `unknown` |
| `browser` | UA parse (server-side) | `chrome` / `safari` / `firefox` / `edge` / `opera` / `other` |
| `os` | UA parse (server-side) | `windows` / `macos` / `android` / `ios` / `linux` / `other` |
| `country_code` | Vector GeoIP enrichment | Empty until Vector GeoIP is configured |
| `ip_address` | Request IP, anonymised | Last IPv4 octet zeroed (`1.2.3.0`); last 80 IPv6 bits zeroed |
| `event_timestamp` | Backend receive time | `DateTime64(3, 'UTC')` — millisecond precision |
| `stats_date` | Derived from `event_timestamp` | `Date` — used for partition pruning in ClickHouse |

---

## 5. UTM and click-macro handling

### How UTM params are captured

On every page load, `captureUrlParams()` in `frontend/lib/chatbot/urlParams.ts`
reads `window.location.search` and merges all params into `sessionStorage` under
key `chatbot_url_params`. This means:

- Params survive navigation within the tab (where the URL has no params).
- Later params override earlier ones (same key wins).
- The full query string is stored — `gclid`, `fbclid`, `utm_ad_id`, and any
  future macro land automatically with zero code changes.

### Named UTM extraction

`frontend/lib/utm.ts` → `getStoredUtmParams()` returns a typed object with six
fields: `utmSource`, `utmMedium`, `utmCampaign`, `utmContent`, `utmTerm`,
`utmCampaignId`. These become fixed columns in ClickHouse for fast filtering.

### Affiliate click-out UTM forwarding

When a user clicks the ad card, `appendUtmToUrl(url)` in `frontend/lib/utm.ts`:
1. Parses the destination URL.
2. Reads stored UTMs from sessionStorage.
3. Appends any UTM that is non-empty and not already in the URL.
4. Returns the enriched URL.

This runs in `AdCard.tsx` before setting the `href`. The `affiliate_click` event
fires at the same time.

---

## 6. File map

```
backend/src/tracking/
  fluentLogger.ts       Singleton FluentClient. Fluent Forward over TCP.
                        Tag: answerbot.events. Fire-and-forget.
  trackingService.ts    Builds ClickHouse row. IP anonymise, UA parse, timestamps.
  trackRouter.ts        POST /api/track/beacon + GET /api/track/pixel.gif

frontend/lib/
  utm.ts                getStoredUtmParams(), appendUtmToUrl()
  chatbot/urlParams.ts  captureUrlParams(), getStoredParams()  ← pre-existing
  tracking/
    visitorId.ts        getVisitorId() — localStorage ab_visitor_id
    trackingClient.ts   sendEvent() — sendBeacon + fetch(keepalive) + requestIdleCallback
    events.ts           Typed emitters for every client-side event

frontend/app/api/track/
  beacon/route.ts       Next.js proxy → backend POST /api/track/beacon
  pixel.gif/route.ts    Next.js proxy → backend GET  /api/track/pixel.gif

frontend/components/
  UrlParamCapture.tsx            page_view on every route change
  StackExchangeSection.tsx       background_content_click on SE links
  chatbot/AdCard.tsx             affiliate_click + UTM forwarding

frontend/themes/hero/
  HeroChatPanel.tsx              chat_opened, chat_window_*, disclosure_shown,
                                 preset_question_clicked

backend/src/chatbot/chatRouter.ts    Server-side events: chat_session_started,
                                     chat_turn, chat_completed, chat_error,
                                     recommendation_shown
```

---

## 7. Configuration

### Backend env vars

```bash
# backend/.env
FLUENT_HOST=localhost   # use "vector" if backend is inside Docker compose
FLUENT_PORT=24224       # standard Fluent Forward port
```

Both are optional — defaults are `localhost:24224`. If Vector is unreachable,
one warning is logged on the first emit then the logger stays silent. No crash.

---

## 8. ClickHouse table schema

Current DDL (also in `analytics/clickhouse-init.sql`):

```sql
CREATE DATABASE IF NOT EXISTS analytics;
CREATE DATABASE IF NOT EXISTS analytics_stg;

CREATE TABLE IF NOT EXISTS analytics.answerbot_events
(
    event_timestamp   DateTime64(3, 'UTC'),
    stats_date        Date,

    event_name        LowCardinality(String),

    user_visitor_id   String,
    session_id        String,
    page_slug         LowCardinality(String),
    site_id           UInt32,
    page_url          String,
    referrer          String,

    props             String,   -- JSON: event-specific fields

    utm_source        String,
    utm_medium        String,
    utm_campaign      String,
    utm_content       String,
    utm_term          String,
    utm_campaign_id   String,

    query_params      String,   -- JSON: full landing query string

    device_type       LowCardinality(String),
    browser           LowCardinality(String),
    os                LowCardinality(String),
    country_code      LowCardinality(String),
    ip_address        String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(stats_date)
ORDER BY event_timestamp
TTL stats_date + INTERVAL 2 YEAR
SETTINGS index_granularity = 8192;

-- Staging table — identical schema
CREATE TABLE IF NOT EXISTS analytics_stg.answerbot_events AS analytics.answerbot_events;
```

### Useful queries

```sql
-- Daily unique visitors
SELECT stats_date, uniqExact(user_visitor_id) AS visitors
FROM analytics.answerbot_events
WHERE event_name = 'page_view'
GROUP BY stats_date ORDER BY stats_date DESC;

-- CTR: affiliate_clicks / chat_sessions_started
SELECT
    stats_date,
    countIf(event_name = 'affiliate_click')      AS clicks,
    countIf(event_name = 'chat_session_started') AS sessions,
    round(clicks / nullIf(sessions, 0), 4)       AS ctr
FROM analytics.answerbot_events
GROUP BY stats_date ORDER BY stats_date DESC;

-- Average bot response time by model
SELECT
    JSONExtractString(props, 'model')              AS model,
    avg(JSONExtractInt(props, 'response_time_ms')) AS avg_ms,
    count()                                        AS turns
FROM analytics.answerbot_events
WHERE event_name = 'chat_turn'
  AND JSONExtractString(props, 'turn_type') = 'bot'
GROUP BY model ORDER BY avg_ms;

-- Top UTM sources driving affiliate clicks
SELECT utm_source, count() AS clicks
FROM analytics.answerbot_events
WHERE event_name = 'affiliate_click' AND utm_source != ''
GROUP BY utm_source ORDER BY clicks DESC LIMIT 20;

-- Full funnel for one visitor (use visitor_id from localStorage or support lookup)
SELECT event_timestamp, event_name, session_id, page_slug, props
FROM analytics.answerbot_events
WHERE user_visitor_id = '<paste_visitor_id_here>'
ORDER BY event_timestamp FORMAT PrettyCompact;

-- All events in one conversation (use session_id from MySQL conversations table)
SELECT event_timestamp, event_name, user_visitor_id, props
FROM analytics.answerbot_events
WHERE session_id = '<paste_session_token_here>'
ORDER BY event_timestamp FORMAT PrettyCompact;
```

---

## 9. Vector configuration

Local dev config is in `analytics/vector.yaml`. For reference:

```yaml
sources:
  fluent_in:
    type: fluent
    address: "0.0.0.0:24224"

sinks:
  clickhouse_out:
    type: clickhouse
    inputs: [fluent_in]
    endpoint: "http://clickhouse:8123"
    database: analytics
    table: answerbot_events
    auth:
      strategy: basic
      user: default
      password: ""
    compression: gzip
    batch:
      max_events: 100
      timeout_secs: 2
    encoding:
      timestamp_format: unix
```

Run the full local analytics stack (Vector + ClickHouse):

```bash
docker compose -f analytics/docker-compose.analytics.yml up -d
```

See `features/analytics-stack.md` for full details — credentials, health checks,
stopping, wiping, DBeaver setup.

---

## 10. Local development

You do **not** need Vector running to develop. When the FluentClient cannot
connect, it logs one warning then stays silent:

```
[tracking] Fluent emit error (non-fatal): Error: connect ECONNREFUSED 127.0.0.1:24224
```

To verify the ingest route is reachable end-to-end:

```bash
# Beacon direct to backend (should return 204)
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:4000/api/track/beacon \
  -H 'Content-Type: application/json' \
  -d '{"eventName":"page_view","pageSlug":"test"}'

# Beacon via Next.js proxy (should return 204)
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:3000/api/track/beacon \
  -H 'Content-Type: application/json' \
  -d '{"eventName":"page_view","pageSlug":"test"}'
```

---

## 11. Adding a new event

### Client-side event

1. Add a typed emitter to `frontend/lib/tracking/events.ts`:

```typescript
export function trackMyNewEvent(ctx: BaseContext & { myProp: string }): void {
  sendEvent({
    eventName: 'my_new_event',
    props:     { my_prop: ctx.myProp },
    ...ctx,
  });
}
```

2. Call it from your component:

```typescript
import { trackMyNewEvent } from '@/lib/tracking/events';
trackMyNewEvent({ pageSlug, sessionId: sessionIdRef.current ?? undefined, myProp: 'value' });
```

No backend changes, no schema changes — props land in the `props` JSON column.

### Server-side event

```typescript
import { trackEvent } from '../tracking/trackingService';

trackEvent({
  eventName: 'my_server_event',
  props:     { some_field: 'value' },
  context: {
    sessionId: conversation.sessionToken,
    pageSlug,
    siteId:    page.siteId ?? undefined,
    rawIp:     String(req.ip ?? ''),
    userAgent: String(req.headers['user-agent'] ?? ''),
  },
});
```

### Promoting a prop to a dedicated column

If a prop needs fast filtering (no `JSONExtractString`), add the column:

```sql
ALTER TABLE analytics.answerbot_events ADD COLUMN my_prop String DEFAULT '';
ALTER TABLE analytics_stg.answerbot_events ADD COLUMN my_prop String DEFAULT '';
```

Then populate it in `trackingService.ts` inside the `row` object.

---

## 12. A/B experiment extension point

No structural rewrites needed — four targeted additions:

**Step 1 — ClickHouse:**
```sql
ALTER TABLE analytics.answerbot_events
  ADD COLUMN experiment_key String DEFAULT '',
  ADD COLUMN exp_variant_id String DEFAULT '';
ALTER TABLE analytics_stg.answerbot_events
  ADD COLUMN experiment_key String DEFAULT '',
  ADD COLUMN exp_variant_id String DEFAULT '';
```

**Step 2 — Backend (`trackingService.ts`)**, add to `TrackContext` and `row`:
```typescript
experimentKey?: string;   // in TrackContext
exp_variant_id: context.expVariantId ?? '',   // in row object
```

**Step 3 — Frontend (`trackingClient.ts`)**, add to `BeaconPayload` and `buildPayload()`:
```typescript
experimentKey?: string;
expVariantId?:  string;
```

**Step 4** — Read the active experiment assignment (localStorage, React context,
cookie) and pass it into `sendEvent()` calls or set it globally in `buildPayload()`.

The `user_visitor_id` is already the stable subject identifier — use it as the
unit of randomisation.

---

## 13. What is NOT tracked

- **Raw user message content** — never in events (persisted in `messages` table, separate from analytics)
- **Full IP address** — last IPv4 octet / last 80 IPv6 bits always zeroed
- **Email addresses or any PII** — no auth in this system, no PII in the event stream
- **System prompts or model names** — not included in client-side events
