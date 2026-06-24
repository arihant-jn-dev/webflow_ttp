# Analytics Stack — Fluent · Vector · ClickHouse

This doc covers everything about the local analytics stack:
how the three pieces connect, how to run/stop/check them,
credentials, and common queries.

---

## Table of contents

1. [How the three pieces connect](#1-how-the-three-pieces-connect)
2. [Config files](#2-config-files)
3. [Credentials](#3-credentials)
4. [Running the stack](#4-running-the-stack)
5. [Checking health](#5-checking-health)
6. [Stopping and wiping](#6-stopping-and-wiping)
7. [Querying ClickHouse](#7-querying-clickhouse)
8. [Connecting DBeaver](#8-connecting-dbeaver)
9. [backend env vars](#9-backend-env-vars)
10. [Troubleshooting](#10-troubleshooting)
11. [Production setup (Kubernetes)](#11-production-setup-kubernetes)

---

## 1. How the three pieces connect

```
Your Backend (Node.js — runs natively on your machine)
  │
  │  FluentClient  ──  Fluent Forward protocol (TCP msgpack)
  │  FLUENT_HOST=localhost
  │  FLUENT_PORT=24224
  │
  ▼
┌─────────────────────────────────────────────┐
│  Docker network: answerbot_analytics        │
│                                             │
│  Vector  (container: answerbot_vector)      │
│    • listens on 0.0.0.0:24224 (host port)  │
│    • receives Fluent Forward events         │
│    • batches every 100 events OR 2 seconds  │
│    • sinks to ClickHouse over HTTP :8123    │
│                 │                           │
│                 ▼                           │
│  ClickHouse  (container: answerbot_click…) │
│    • HTTP port 8123  (queries, DBeaver)     │
│    • native port 9000                       │
│    • database: analytics                    │
│    • table: answerbot_events                │
└─────────────────────────────────────────────┘
```

**Key points:**
- Your backend talks to Vector using `localhost:24224` because Docker publishes port 24224 to your host machine.
- Vector talks to ClickHouse using `clickhouse:8123` — the Docker service name, not localhost. They share a Docker network so they can resolve each other by name.
- You (DBeaver, curl) talk to ClickHouse using `localhost:8123` — also published to your host.

---

## 2. Config files

All analytics config lives in the `analytics/` folder at the repo root.

```
analytics/
  docker-compose.analytics.yml   Container definitions for Vector + ClickHouse
  vector.yaml                    Vector source + sink config
  clickhouse-init.sql            DDL — runs once on first ClickHouse boot
```

### docker-compose.analytics.yml — what it does

| Service | Image | Host ports | Purpose |
|---|---|---|---|
| `answerbot_clickhouse` | `clickhouse/clickhouse-server:24.3` | `8123`, `9000` | Event storage |
| `answerbot_vector` | `timberio/vector:0.39.0-alpine` | `24224` | Receives events from backend, writes to ClickHouse |

### vector.yaml — what it does

```yaml
sources:
  fluent_in:
    type: fluent
    address: "0.0.0.0:24224"    # ← listens for backend events

sinks:
  clickhouse_out:
    type: clickhouse
    inputs: [fluent_in]
    endpoint: "http://clickhouse:8123"  # ← writes to ClickHouse
    database: analytics
    table: answerbot_events
    batch:
      max_events: 100       # flush after 100 events...
      timeout_secs: 2       # ...or every 2 seconds, whichever first
```

### clickhouse-init.sql — what it does

Runs automatically the **first time** the ClickHouse container starts.
Creates two databases and the events table in each:

| Database | Purpose |
|---|---|
| `analytics` | Production data (real users) |
| `analytics_stg` | Staging / local dev test data |

Both have identical schema. Vector is pointed at `analytics` by default in local dev.
Change `database: analytics` → `database: analytics_stg` in `vector.yaml` if you want test events to go to the staging table.

---

## 3. Credentials

### ClickHouse

| Field | Value |
|---|---|
| Host | `localhost` |
| HTTP port | `8123` |
| Native port | `9000` |
| User | `default` |
| Password | *(none — blank)* |
| Default database | `analytics` |

No password is set. This is intentional for local dev — ClickHouse ships
with a `default` user and no password out of the box.

### Vector

Vector has no credentials itself. It just writes to ClickHouse using the
`default` user with no password (matches above).

### Backend → Vector (Fluent Forward)

No authentication on the Fluent Forward TCP connection either.
The backend just connects to `localhost:24224` — no username or password.

| Env var | Value |
|---|---|
| `FLUENT_HOST` | `localhost` |
| `FLUENT_PORT` | `24224` |

---

## 4. Running the stack

**Prerequisites:** Docker Desktop must be running.

```bash
# From the repo root
docker compose -f analytics/docker-compose.analytics.yml up -d
```

First boot takes ~30 seconds — ClickHouse initialises its data directories
and runs `clickhouse-init.sql`. Subsequent boots are instant (data persists
in the `clickhouse_data` Docker volume).

**Check both containers are up:**
```bash
docker compose -f analytics/docker-compose.analytics.yml ps
```

Expected output:
```
NAME                    STATUS
answerbot_clickhouse    Up (healthy)
answerbot_vector        Up
```

**Make sure your backend has the right env vars** in `backend/.env`:
```
FLUENT_HOST=localhost
FLUENT_PORT=24224
```

Restart the backend after adding these so it picks them up.

---

## 5. Checking health

### Is ClickHouse alive?
```bash
curl -s "http://localhost:8123/ping"
# Expected: Ok.
```

### Is Vector receiving events?
```bash
# Send a test event directly to Vector (bypasses Next.js and backend)
echo '{"tag":"answerbot.events","eventName":"test"}' | nc -q1 localhost 24224
```

Or send through the normal path (backend must be running):
```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:4000/api/track/beacon \
  -H 'Content-Type: application/json' \
  -d '{"eventName":"page_view","pageSlug":"test"}'
# Expected: 204
```

### Did the event land in ClickHouse?
```bash
curl -s "http://localhost:8123" \
  --data "SELECT event_name, count() FROM analytics.answerbot_events GROUP BY event_name FORMAT PrettyCompact"
```

### Check Vector container logs (useful when events aren't arriving)
```bash
docker logs answerbot_vector --tail 50 --follow
```

### Check ClickHouse container logs
```bash
docker logs answerbot_clickhouse --tail 50
```

---

## 6. Stopping and wiping

### Stop containers (keep data)
```bash
docker compose -f analytics/docker-compose.analytics.yml down
```
Data survives in the `clickhouse_data` Docker volume. Next `up -d` picks up
where you left off.

### Stop and wipe all data
```bash
docker compose -f analytics/docker-compose.analytics.yml down -v
```
`-v` deletes the `clickhouse_data` volume. Next boot re-runs `clickhouse-init.sql`
and starts fresh with empty tables.

### Restart just one service
```bash
docker restart answerbot_vector
docker restart answerbot_clickhouse
```

### Wipe only the events table (keep containers running)
```bash
curl -s "http://localhost:8123" \
  --data "TRUNCATE TABLE analytics.answerbot_events"
```

---

## 7. Querying ClickHouse

All queries go to `http://localhost:8123` via curl or DBeaver.

### Row count by event
```bash
curl -s "http://localhost:8123" \
  --data "SELECT event_name, count() as cnt
          FROM analytics.answerbot_events
          GROUP BY event_name
          ORDER BY cnt DESC
          FORMAT PrettyCompact"
```

### Last 10 events (newest first)
```bash
curl -s "http://localhost:8123" \
  --data "SELECT event_timestamp, event_name, page_slug, session_id, browser, os
          FROM analytics.answerbot_events
          ORDER BY event_timestamp DESC
          LIMIT 10
          FORMAT PrettyCompact"
```

### See full detail of one event
```bash
curl -s "http://localhost:8123" \
  --data "SELECT *
          FROM analytics.answerbot_events
          WHERE event_name = 'chat_session_started'
          LIMIT 1
          FORMAT Vertical"
```

### Chat funnel (sessions → completions)
```bash
curl -s "http://localhost:8123" \
  --data "SELECT
            countIf(event_name='chat_session_started') AS sessions,
            countIf(event_name='chat_completed')       AS completions,
            countIf(event_name='affiliate_click')      AS affiliate_clicks
          FROM analytics.answerbot_events
          FORMAT PrettyCompact"
```

### Average bot response time
```bash
curl -s "http://localhost:8123" \
  --data "SELECT
            JSONExtractString(props, 'model')                AS model,
            avg(JSONExtractInt(props, 'response_time_ms'))   AS avg_ms,
            count()                                          AS turns
          FROM analytics.answerbot_events
          WHERE event_name = 'chat_turn'
            AND JSONExtractString(props, 'turn_type') = 'bot'
          GROUP BY model
          FORMAT PrettyCompact"
```

### Events by browser
```bash
curl -s "http://localhost:8123" \
  --data "SELECT browser, os, count() as cnt
          FROM analytics.answerbot_events
          GROUP BY browser, os
          ORDER BY cnt DESC
          FORMAT PrettyCompact"
```

### Show table schema
```bash
curl -s "http://localhost:8123" \
  --data "DESCRIBE analytics.answerbot_events FORMAT PrettyCompact"
```

---

## 8. Connecting DBeaver

1. Open DBeaver → **Database** → **New Database Connection**
2. Search `ClickHouse` → select it → **Next**
3. Fill in:

| Field | Value |
|---|---|
| Host | `localhost` |
| Port | `8123` |
| Database | `analytics` |
| Username | `default` |
| Password | *(leave blank)* |

4. Click **Test Connection** — should say `Connected`
5. Click **Finish**

You'll see two databases: `analytics` and `analytics_stg`, each with an
`answerbot_events` table. You can run SQL queries directly in DBeaver's SQL editor.

**Driver note:** If DBeaver says the ClickHouse driver is missing, go to
**Database → Driver Manager** → search `ClickHouse` → **Edit** → **Download**.

---

## 9. Backend env vars

These two vars tell the backend's FluentClient where to send events.

```bash
# backend/.env
FLUENT_HOST=localhost    # Vector container's published host
FLUENT_PORT=24224        # Fluent Forward port
```

**When to change `FLUENT_HOST`:**

| Setup | Value |
|---|---|
| Backend runs natively, Vector in Docker (current local dev setup) | `localhost` |
| Backend in Docker compose on same network as Vector | `vector` (Docker service name) |
| Vector on a remote server | the server's IP or hostname |

After changing `.env`, restart the backend — env vars are only read at startup.

---

## 10. Troubleshooting

### Events not appearing in ClickHouse

**Step 1 — Is ClickHouse up?**
```bash
curl -s "http://localhost:8123/ping"   # should return: Ok.
```

**Step 2 — Is Vector up?**
```bash
docker ps | grep vector
docker logs answerbot_vector --tail 20
```

**Step 3 — Is the backend connecting to Vector?**

Check backend logs for this line on first event:
```
[tracking] Fluent emit error (non-fatal): ...ECONNREFUSED...
```
If you see it → Vector is not running or not on port 24224.
If you don't see it → the connection is fine.

**Step 4 — Wait for the batch flush**

Vector batches events: it waits for 100 events OR 2 seconds before writing
to ClickHouse. After firing an event, wait 3 seconds then query.

**Step 5 — Check if the event row exists at all**
```bash
curl -s "http://localhost:8123" \
  --data "SELECT count() FROM analytics.answerbot_events"
```

### ClickHouse init SQL didn't run (table missing)

This happens if the container was created before `clickhouse-init.sql` existed.
The init script only runs on **first boot** (empty volume).

Fix — wipe and restart:
```bash
docker compose -f analytics/docker-compose.analytics.yml down -v
docker compose -f analytics/docker-compose.analytics.yml up -d
```

### Port 24224 already in use
```bash
lsof -i :24224    # find what's using it
```
If another fluentd/fluent-bit is running locally, stop it or change
`FLUENT_PORT` in `backend/.env` and update the port mapping in
`analytics/docker-compose.analytics.yml`.

### Port 8123 already in use
Another ClickHouse instance is running. Either stop it, or change the
host port in `docker-compose.analytics.yml`:
```yaml
ports:
  - "8124:8123"   # use 8124 on host instead
```
Then update queries to use `localhost:8124`.

---

## 11. Production setup (Kubernetes)

The local Docker Compose stack (Vector + ClickHouse containers) is **only for
local development**. In production the architecture is different — Vector runs
as a sidecar inside each pod, and a central Vector Aggregator fans out to
ClickHouse and S3.

```
Browser
  → AWS Load Balancer → Nginx → App Server pod
                                      │
                                 Vector sidecar   ← same pod, localhost:24224
                                      │
                                 Vector Aggregator ← central collector
                                      │
                          ┌───────────┴───────────┐
                       ClickHouse                 S3
```

### What answerbot sends

The backend emits events using the Fluent Forward protocol to `localhost:24224`,
tagged `answerbot.events`. This is the only contract between answerbot and the
infra layer. The sidecar handles everything else.

### What changes in production

**Nothing in the answerbot codebase changes.** The backend always talks to
`localhost:24224` — in local dev that's your Docker Vector container, in
production it's the Vector sidecar running in the same pod. Same host, same
port, same tag.

| Config | Local dev | Production |
|---|---|---|
| `FLUENT_HOST` | `localhost` | `localhost` (sidecar is in the same pod) |
| `FLUENT_PORT` | `24224` | `24224` |
| Vector config | `analytics/vector.yaml` | Managed in infra/k8s repo |
| Docker Compose | `analytics/docker-compose.analytics.yml` | Not used |
| ClickHouse DDL | `analytics/clickhouse-init.sql` | Run once by infra team |

### What to hand off to the infra team

Give them two things:

**1. The ClickHouse DDL** — `analytics/clickhouse-init.sql`
They run this once against the production ClickHouse instance to create the
`analytics` and `analytics_stg` databases and the `answerbot_events` table.

**2. These requirements for the Vector sidecar config:**

```
- Accept Fluent Forward on localhost:24224 inside the pod
- Tag filter: answerbot.events
- Forward to the Vector Aggregator
- Aggregator sinks: ClickHouse (analytics.answerbot_events) + S3 (raw backup)
```

The sidecar's `vector.yaml` in the infra repo should have at minimum:

```yaml
sources:
  fluent_in:
    type: fluent
    address: "0.0.0.0:24224"

sinks:
  to_aggregator:
    type: vector   # or fluent / http depending on aggregator setup
    inputs: [fluent_in]
    address: "<aggregator-host>:<aggregator-port>"
```

The aggregator then writes to ClickHouse and S3 — that config lives entirely
in the infra repo and answerbot has no dependency on it.

### No code changes needed when going to production

The only env var that matters is confirmed to be the same:
```bash
FLUENT_HOST=localhost
FLUENT_PORT=24224
```

Set these in your k8s secret/configmap for the app container and the pipeline
works automatically once the sidecar is running alongside it.
