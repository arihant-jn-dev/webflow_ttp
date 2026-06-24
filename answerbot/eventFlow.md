# Event Flow — End-to-End Technical Walkthrough

This document explains exactly how an analytics event travels from a user
interaction in the browser all the way to a row in ClickHouse. Every function,
file, and protocol hop is covered.

Two full walkthroughs are included:
- **Client-side event** — `preset_question_clicked` (browser → beacon → Express → Fluent → Vector → ClickHouse)
- **Server-side event** — `chat_turn` (Express handler → trackEvent → Fluent → Vector → ClickHouse)

---

## Table of contents

1. [Architecture overview](#1-architecture-overview)
2. [Client-side event: preset_question_clicked](#2-client-side-event-preset_question_clicked)
3. [Server-side event: chat_turn](#3-server-side-event-chat_turn)
4. [Shared internals used by both paths](#4-shared-internals-used-by-both-paths)
5. [End-to-end timing](#5-end-to-end-timing)
6. [What happens when something is down](#6-what-happens-when-something-is-down)

---

## 1. Architecture overview

```
┌─────────────────────────────────────────────────────────┐
│  BROWSER                                                │
│                                                         │
│  User interaction                                       │
│       │                                                 │
│       ▼                                                 │
│  events.ts          typed emitter fn                    │
│       │                                                 │
│       ▼                                                 │
│  trackingClient.ts  buildPayload() + requestIdleCallback│
│       │                                                 │
│       ▼                                                 │
│  navigator.sendBeacon('/api/track/beacon')              │
└────────────────────────┬────────────────────────────────┘
                         │  HTTP POST (same-origin)
                         ▼
┌─────────────────────────────────────────────────────────┐
│  NEXT.JS  app/api/track/beacon/route.ts                 │
│  Same-origin proxy — forwards body verbatim             │
└────────────────────────┬────────────────────────────────┘
                         │  HTTP POST → localhost:4000
                         ▼
┌─────────────────────────────────────────────────────────┐
│  EXPRESS  backend/src/tracking/trackRouter.ts           │
│                                                         │
│  res.status(204).end()   ← browser released here       │
│  setImmediate(() => {                                   │
│       │                                                 │
│       ▼                                                 │
│  trackingService.ts  buildRow() — IP anon, UA parse     │
│       │                                                 │
│       ▼                                                 │
│  fluentLogger.ts     FluentClient.emit()                │
│  })                                                     │
└────────────────────────┬────────────────────────────────┘
                         │  TCP :24224 (Fluent Forward / msgpack)
                         ▼
┌─────────────────────────────────────────────────────────┐
│  VECTOR   analytics/vector.yaml                         │
│  Fluent source → batch (100 events OR 2s) → CH sink     │
└────────────────────────┬────────────────────────────────┘
                         │  HTTP POST → clickhouse:8123
                         ▼
┌─────────────────────────────────────────────────────────┐
│  CLICKHOUSE  analytics.answerbot_events                 │
└─────────────────────────────────────────────────────────┘
```

**For server-side events** the browser is not involved at all. The Express chat
handler calls `trackEvent()` directly — skipping everything above the Express
box. The path from Express downward is identical.

---

## 2. Client-side event: preset_question_clicked

**Trigger:** user taps one of the starter question chips in the chat panel.

---

### Step 1 — User taps the chip

**File:** [frontend/themes/hero/HeroChatPanel.tsx](../frontend/themes/hero/HeroChatPanel.tsx)

```typescript
// Starter question chip onClick handler
onClick={() => {
  trackPresetQuestionClicked({
    pageSlug,
    sessionId,        // React ref — null before first message, filled after
    questionText: q,
    questionIndex: i,
  });
  handleSend(q);      // sends the chat message — completely independent
}}
```

`trackPresetQuestionClicked` and `handleSend` are called in the same tick but
are completely independent. `handleSend` opens the SSE stream to the backend.
`trackPresetQuestionClicked` starts the analytics path below. Neither waits
for the other.

---

### Step 2 — Typed emitter

**File:** [frontend/lib/tracking/events.ts](../frontend/lib/tracking/events.ts)

```typescript
export function trackPresetQuestionClicked(
  ctx: BaseContext & { questionText: string; questionIndex: number }
): void {
  sendEvent({
    eventName: 'preset_question_clicked',
    props: {
      question_text:  ctx.questionText,   // e.g. "What is GLP-1?"
      question_index: ctx.questionIndex,  // e.g. 0
    },
    pageSlug:  ctx.pageSlug,
    sessionId: ctx.sessionId,
  });
}
```

This function only shapes the data into the standard payload format.
All shared context (visitor id, UTMs, URL) is added downstream in
`buildPayload()` — not here. This keeps each emitter minimal.

`BaseContext` is:
```typescript
interface BaseContext {
  sessionId?: string;
  pageSlug?:  string;
  siteId?:    number;
}
```

---

### Step 3 — sendEvent: defer off the hot path

**File:** [frontend/lib/tracking/trackingClient.ts](../frontend/lib/tracking/trackingClient.ts)

```typescript
export function sendEvent(input: BeaconPayload): void {
  if (typeof window === 'undefined') return;  // SSR guard — never runs on server

  const payload = buildPayload(input);  // assemble full context NOW (values may change later)

  // Defer dispatch until the browser is idle — tracking never competes with
  // React rendering, the chat stream, or the user's next keypress.
  if (typeof requestIdleCallback !== 'undefined') {
    requestIdleCallback(() => dispatch(payload), { timeout: 2000 });
  } else {
    setTimeout(() => dispatch(payload), 0);  // Safari fallback
  }
}
```

`buildPayload` is called **immediately** (before the idle defer) because values
like `window.location.href` and `document.referrer` are correct right now — by
the time the idle callback fires, the user may have navigated away.

`requestIdleCallback` with `timeout: 2000` means: run when idle, but force-run
after 2 seconds maximum even if the browser is still busy.

---

### Step 4 — buildPayload: assemble full context

**File:** [frontend/lib/tracking/trackingClient.ts](../frontend/lib/tracking/trackingClient.ts)

```typescript
function buildPayload(input: BeaconPayload): Record<string, unknown> {
  const utm         = getStoredUtmParams();   // reads sessionStorage
  const queryParams = getStoredParams();      // reads sessionStorage

  return {
    eventName:     input.eventName,           // 'preset_question_clicked'
    props:         input.props ?? {},         // { question_text, question_index }
    userVisitorId: getVisitorId(),            // reads localStorage ab_visitor_id
    sessionId:     input.sessionId ?? '',     // '' if no chat started yet
    pageSlug:      input.pageSlug  ?? '',     // 'glp1-weight-loss-E'
    siteId:        input.siteId    ?? 0,
    pageUrl:       window.location.href,      // full URL at event time
    referrer:      document.referrer,
    utmSource:     utm.utmSource,
    utmMedium:     utm.utmMedium,
    utmCampaign:   utm.utmCampaign,
    utmContent:    utm.utmContent,
    utmTerm:       utm.utmTerm,
    utmCampaignId: utm.utmCampaignId,
    queryParams,                              // full landing params blob
  };
}
```

Three storage reads:

**`getVisitorId()`** — [frontend/lib/tracking/visitorId.ts](../frontend/lib/tracking/visitorId.ts)
```typescript
export function getVisitorId(): string {
  if (typeof window === 'undefined') return '';
  let id = localStorage.getItem('ab_visitor_id');
  if (!id) {
    id = crypto.randomUUID();
    localStorage.setItem('ab_visitor_id', id);
  }
  return id;
}
```
Creates a UUID on first visit, returns the same one every time after.
Persists forever (until localStorage cleared).

**`getStoredUtmParams()`** — [frontend/lib/utm.ts](../frontend/lib/utm.ts)
```typescript
export function getStoredUtmParams(): StoredUtmParams {
  const all = getStoredParams();  // reads sessionStorage chatbot_url_params
  return {
    utmSource:     all['utm_source']      ?? '',
    utmMedium:     all['utm_medium']      ?? '',
    utmCampaign:   all['utm_campaign']    ?? '',
    utmContent:    all['utm_content']     ?? '',
    utmTerm:       all['utm_term']        ?? '',
    utmCampaignId: all['utm_campaign_id'] ?? '',
  };
}
```

**`getStoredParams()`** — [frontend/lib/chatbot/urlParams.ts](../frontend/lib/chatbot/urlParams.ts)
```typescript
// reads the full merged query-param record from sessionStorage
// key: 'chatbot_url_params'
// set by captureUrlParams() which runs in UrlParamCapture.tsx on every page mount
```

---

### Step 5 — dispatch: sendBeacon

**File:** [frontend/lib/tracking/trackingClient.ts](../frontend/lib/tracking/trackingClient.ts)

```typescript
function dispatch(payload: Record<string, unknown>): void {
  if (typeof window === 'undefined') return;

  const body = JSON.stringify(payload);
  const blob = new Blob([body], { type: 'application/json' });

  // sendBeacon: fire-and-forget, works on page unload, no CORS preflight
  if (navigator.sendBeacon && navigator.sendBeacon('/api/track/beacon', blob)) return;

  // Fallback: keepalive fetch — used if sendBeacon returns false
  // (can happen if payload > 64KB or browser disables beacon API)
  fetch('/api/track/beacon', {
    method:    'POST',
    headers:   { 'Content-Type': 'application/json' },
    body,
    keepalive: true,
  }).catch(() => {});  // errors swallowed — never surface to user
}
```

`sendBeacon` returns `true` if the browser accepted the request for sending.
It queues it internally — it will be sent even if the user navigates away
immediately. No Promise, no callback, no way to check the response.

---

### Step 6 — Next.js proxy route

**File:** [frontend/app/api/track/beacon/route.ts](../frontend/app/api/track/beacon/route.ts)

```typescript
const BACKEND = process.env.INTERNAL_API_BASE_URL || 'http://backend:4000';

export async function POST(request: NextRequest) {
  const body = await request.text();  // read raw body

  const upstream = await fetch(`${BACKEND}/api/track/beacon`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body,
    signal:  request.signal,  // forward abort so backend stops if browser disconnects
  });

  return new Response(null, { status: upstream.status });  // pass 204 back
}
```

This route exists purely because `sendBeacon` can only POST to the **same
origin** (`weight.localhost:3000`). The backend runs on a different port
(`localhost:4000`) which is a different origin. The proxy bridges them —
it receives the request on the Next.js origin and forwards it to the backend
over the internal network. No transformation of the body.

---

### Step 7 — Express ingest: respond first, ingest second

**File:** [backend/src/tracking/trackRouter.ts](../backend/src/tracking/trackRouter.ts)

```typescript
router.post('/api/track/beacon', async (req, res) => {
  // 1. Release the browser IMMEDIATELY — nothing below blocks the response
  res.status(204).end();

  // 2. Ingest in the next event loop tick — completely after the response is sent
  setImmediate(() => {
    // Validate with Zod — silently drop invalid payloads (never 400 a browser)
    const parsed = beaconSchema.safeParse(req.body);
    if (!parsed.success) return;

    trackEvent({
      eventName: parsed.data.eventName,  // 'preset_question_clicked'
      props:     parsed.data.props,      // { question_text, question_index }
      context: {
        sessionId:  parsed.data.sessionId  ?? '',
        pageSlug:   parsed.data.pageSlug   ?? '',
        siteId:     parsed.data.siteId,
        rawIp:      String(req.ip ?? ''),
        userAgent:  String(req.headers['user-agent'] ?? ''),
        // client-supplied fields (assembled by buildPayload in the browser)
        userVisitorId: parsed.data.userVisitorId ?? '',
        pageUrl:       parsed.data.pageUrl       ?? '',
        referrer:      parsed.data.referrer      ?? '',
        utmSource:     parsed.data.utmSource     ?? '',
        // ... rest of UTM + queryParams
      },
    });
  });
});
```

`setImmediate` schedules the callback for the next iteration of the Node.js
event loop — after I/O callbacks, after the response has been fully flushed to
the socket. The browser gets its 204 before a single line of tracking logic runs.

---

### Step 8 — trackEvent: build the ClickHouse row

**File:** [backend/src/tracking/trackingService.ts](../backend/src/tracking/trackingService.ts)

```typescript
export function trackEvent(payload: TrackPayload): void {
  const { eventName, props, context } = payload;
  const now = new Date();
  const ua  = parseUserAgent(context.userAgent ?? '');

  const row: Record<string, unknown> = {
    // Timestamps
    event_timestamp: toClickHouseDateTime(now),  // '2026-06-24 07:45:12.483'
    stats_date:      toClickHouseDate(now),       // '2026-06-24'

    event_name: eventName,  // 'preset_question_clicked'

    // Identity
    user_visitor_id: context.userVisitorId ?? '',
    session_id:      context.sessionId     ?? '',  // '' — fired before first message
    page_slug:       context.pageSlug      ?? '',
    site_id:         context.siteId        ?? 0,
    page_url:        context.pageUrl       ?? '',
    referrer:        context.referrer      ?? '',

    // Event-specific data as JSON string
    props: JSON.stringify(props ?? {}),
    // stored as: '{"question_text":"What is GLP-1?","question_index":0}'

    // UTM fields (individual columns for fast filtering)
    utm_source:     context.utmSource     ?? '',
    utm_medium:     context.utmMedium     ?? '',
    utm_campaign:   context.utmCampaign   ?? '',
    utm_content:    context.utmContent    ?? '',
    utm_term:       context.utmTerm       ?? '',
    utm_campaign_id: context.utmCampaignId ?? '',

    // Full query param blob
    query_params: JSON.stringify(context.queryParams ?? {}),

    // Device context — parsed from User-Agent header
    device_type: ua.deviceType,  // 'mobile'
    browser:     ua.browser,     // 'safari'
    os:          ua.os,          // 'ios'

    // IP — anonymised: last IPv4 octet zeroed
    ip_address:  anonymiseIp(context.rawIp),  // '103.21.244.0'
    country_code: '',  // filled by Vector GeoIP if configured
  };

  emitEvent(row);  // hand off to FluentClient
}
```

Two operations only the server can do:

**`anonymiseIp(rawIp)`**
```typescript
// IPv4: '103.21.244.197' → '103.21.244.0'   (last octet zeroed)
// IPv6: keeps first 48 bits, zeros the rest
```

**`parseUserAgent(ua)`**
```typescript
// Regex match on the User-Agent string forwarded from the browser
// Returns: { deviceType: 'mobile', browser: 'safari', os: 'ios' }
```

**`toClickHouseDateTime(date)`**
```typescript
// new Date() → 'YYYY-MM-DD HH:MM:SS.mmm'
// ClickHouse DateTime64(3) expects exactly this format
```

---

### Step 9 — emitEvent: send over Fluent Forward

**File:** [backend/src/tracking/fluentLogger.ts](../backend/src/tracking/fluentLogger.ts)

```typescript
// Singleton client — created once, reused for every event
let client: FluentClient | null = null;

function getClient(): FluentClient {
  if (!client) {
    client = new FluentClient(null, {
      socket: {
        host:    process.env.FLUENT_HOST ?? 'localhost',
        port:    Number(process.env.FLUENT_PORT ?? 24224),
        timeout: 3000,
      },
    });
  }
  return client;
}

export function emitEvent(row: Record<string, unknown>): void {
  const c = getClient();
  c.emit('answerbot.events', row).catch((err) => {
    // One warning log — then silence. A Vector outage never crashes the backend.
    console.warn('[tracking] Fluent emit error (non-fatal):', err);
  });
}
```

`FluentClient.emit('answerbot.events', row)` serialises the row as **msgpack**
and sends it over a persistent TCP connection to Vector on port 24224.
The tag `answerbot.events` is how Vector identifies which source sent the event.

The `.catch` is critical — if Vector is down, the Promise rejects, the warning
logs once, and the backend continues serving chat requests completely unaffected.

---

### Step 10 — Vector: batch and forward to ClickHouse

**File:** [analytics/vector.yaml](../analytics/vector.yaml)

Vector receives the Fluent Forward message on TCP port 24224. It holds events
in memory and flushes when either condition is met:
- 100 events accumulated, OR
- 2 seconds have passed since the last flush

Then it POSTs the batch to ClickHouse HTTP API:
```
POST http://clickhouse:8123
?query=INSERT INTO analytics.answerbot_events FORMAT JSONEachRow
body: { ...row1 }\n{ ...row2 }\n...
```

---

### Step 11 — ClickHouse stores the row

The row lands in `analytics.answerbot_events`:

```sql
SELECT event_timestamp, event_name, user_visitor_id, session_id, props
FROM analytics.answerbot_events
WHERE event_name = 'preset_question_clicked'
ORDER BY event_timestamp DESC
LIMIT 3
FORMAT PrettyCompact;
```

```
┌─event_timestamp──────────────┬─event_name───────────────┬─user_visitor_id──────────────────────┬─session_id─┬─props──────────────────────────────────────────────┐
│ 2026-06-24 07:45:12.483      │ preset_question_clicked   │ d02898e2-8db0-4209-8261-8fb8f9076e66 │            │ {"question_text":"What is GLP-1?","question_index":0} │
└──────────────────────────────┴──────────────────────────┴──────────────────────────────────────┴────────────┴────────────────────────────────────────────────────┘
```

`session_id` is empty — this event fired before the user sent their first
message, so `sessionIdRef.current` was still `null` in React.

---

## 3. Server-side event: chat_turn

**Trigger:** backend finishes streaming a bot response.

This path is shorter — no browser, no Next.js proxy, no beacon. The Express
chat handler calls `trackEvent()` directly and the flow picks up at Step 8 above.

---

### Step 1 — chatRouter emits the event

**File:** [backend/src/chatbot/chatRouter.ts](../backend/src/chatbot/chatRouter.ts)

```typescript
// Inside the OpenRouter stream completion callback:
const endTime = Date.now();

const serverTrackContext = {
  sessionId:  conversation.sessionToken,        // always present — server created it
  pageSlug,
  siteId:     page.siteId ?? undefined,
  rawIp:      String(req.ip ?? ''),
  userAgent:  String(req.headers['user-agent'] ?? ''),  // forwarded from browser by Next.js proxy
};

// Bot turn event
trackEvent({
  eventName: 'chat_turn',
  props: {
    turn_type:        'bot',
    turn_index:       turnIndex,
    response_time_ms: endTime - startTime,
    char_count:       botReply.length,
    model:            modelId,
  },
  context: serverTrackContext,
});

// Conversation completed event
trackEvent({
  eventName: 'chat_completed',
  props: {
    total_user_turns: totalUserTurns,
    total_duration_ms: endTime - conversationStart,
  },
  context: serverTrackContext,
});
```

`trackEvent()` is imported from `trackingService.ts` — same function used by
the beacon path. From here the flow is identical to Steps 8 → 11 above.

---

### What's different vs a client-side event

| | Client-side (`preset_question_clicked`) | Server-side (`chat_turn`) |
|---|---|---|
| Origin | Browser | Express handler |
| Path | UI → events.ts → trackingClient → sendBeacon → Next.js proxy → Express → trackEvent | Express handler → trackEvent directly |
| `user_visitor_id` | Filled — from localStorage | Empty — server has no localStorage |
| `session_id` | May be empty (before first message) | Always filled — server created it |
| `device_type` / `browser` / `os` | Filled — UA parsed from forwarded header | Filled — same UA parsing |
| `ip_address` | Filled — from request IP | Filled — same anonymisation |
| Deferred via setImmediate? | Yes — from beacon handler | Yes — same beacon handler wraps it |

---

## 4. Shared internals used by both paths

These modules are called regardless of whether the event is client or server side.

### trackingService.ts — `trackEvent()`
Single entry point for all event emission. Builds the full ClickHouse row,
anonymises IP, parses UA, formats timestamps. Both the beacon handler and
chatRouter call this function.

### fluentLogger.ts — `emitEvent()`
Singleton FluentClient. Serialises the row as msgpack, sends over TCP to Vector.
Called only by `trackingService.ts`. Fire-and-forget, errors swallowed.

### Vector
Receives all events regardless of source. Batches and writes to ClickHouse.
Has no knowledge of whether an event came from the browser or the server.

### ClickHouse `analytics.answerbot_events`
Single table. All events land here. Server-side events have `user_visitor_id = ''`.
Client-side events may have `session_id = ''`. You join them via whichever field
is present — see the lookup queries in `features/event-tracking.md` section 2.

---

## 5. End-to-end timing

### Client-side event

| Step | Who waits | Approx time |
|---|---|---|
| Chip tap → `requestIdleCallback` queued | Nobody — deferred | 0ms felt by user |
| `buildPayload()` runs (storage reads) | Idle callback | ~1ms |
| `sendBeacon` dispatched | Browser queues it | ~0ms |
| Next.js proxy receives + forwards | Browser waiting for 204 | ~3ms |
| Express sends 204 | Browser done ✓ | ~1ms |
| `setImmediate` → `trackEvent` → Fluent emit | Nobody | ~2ms background |
| Vector batch flush | Nobody | up to 2 seconds |
| ClickHouse write | Nobody | ~10ms |

**User feels: 0ms.** The chip tap and chat send complete before any of this starts.

### Server-side event

| Step | Who waits | Approx time |
|---|---|---|
| `trackEvent()` called in stream callback | Nobody — non-blocking | ~0ms |
| Fluent emit (TCP send) | Nobody — async | ~1ms background |
| Vector batch flush | Nobody | up to 2 seconds |
| ClickHouse write | Nobody | ~10ms |

**User feels: 0ms.** The SSE stream has already completed before `trackEvent` is called.

---

## 6. What happens when something is down

### Vector is down
- `FluentClient.emit()` rejects its Promise
- `.catch()` in `fluentLogger.ts` logs one warning: `[tracking] Fluent emit error (non-fatal):`
- After that: silence — no further logs, no retries, no crash
- Chat, page loads, and all user-facing features work normally
- Events during the outage are lost (not buffered on the backend)

### ClickHouse is down
- Vector cannot write its batch
- Vector retries internally according to its sink retry config
- The backend and frontend have no awareness of this
- Everything works normally

### Next.js proxy is down (only affects client-side events)
- `sendBeacon` to `/api/track/beacon` gets a connection error
- The fallback `fetch(keepalive)` also fails
- The `.catch(() => {})` swallows the error silently
- No user-visible effect

### Backend is down (only affects server-side events)
- `trackEvent()` is never called — the chat request itself failed first
- No tracking-specific handling needed
