# 54 — System Design for Python Backends

> Week 5 · Production & Interview

System design is where senior backend interviews are won or lost. You will not be asked to write a clever sorting algorithm — you will be handed a vague prompt ("Design a URL shortener") and judged on how you *think*: how you clarify, estimate, structure, and reason about trade-offs. This lesson gives you a repeatable framework and shows how the Python/Django/FastAPI tools you already learned map onto each box you'll draw on the whiteboard.

## What you will learn

- A repeatable 6-step framework for any backend system-design interview
- How to estimate scale with back-of-the-envelope math
- Designing a clean REST API: resource modeling, versioning, status codes, pagination, idempotency
- Layered / clean architecture (views → services → repositories → models) and why business logic must leave your views
- Statelessness and horizontal scaling behind a load balancer
- Caching layers (application, Redis, CDN) and cache invalidation
- Database scaling: read replicas, connection pooling, partitioning/sharding, when to denormalize
- Async processing with queues (Celery) for slow work
- Rate limiting and idempotency keys
- Observability: structured logging, metrics, tracing, health checks
- A worked end-to-end design (URL shortener) and a peek at an e-commerce orders API
- Monolith vs microservices trade-offs

## PHP/Laravel bridge

You already know most of these *concepts* under Laravel names — system design is mostly the same engineering wearing different libraries. Laravel's `Controller → Service → Repository → Eloquent Model` is exactly the layered architecture below; the equivalent in Django is `View → Service → Manager/Repository → Model`, and in FastAPI it's `Router → Service → Repository → SQLAlchemy Model`. Laravel Queues + Horizon map almost 1:1 onto **Celery + Flower**, Laravel's cache façade and `RateLimiter` map onto **Redis**, and `php artisan` migrations map onto Django migrations / Alembic. The interview-specific skill that Laravel didn't teach you is the *whiteboard conversation*: clarifying scope and estimating scale out loud before you touch the data model.

---

## Core concept: a framework for the interview

Interviewers grade your *process*, not a "correct" answer (there isn't one). Drive the conversation through six steps, narrating as you go. Spend the first ~5 minutes on steps 1–2 before drawing anything; juniors who jump straight to boxes-and-arrows lose points.

| Step | What you do | Time budget (45 min) |
|------|-------------|----------------------|
| 1. Clarify requirements | Functional + non-functional. Ask, don't assume. | 5 min |
| 2. Estimate scale | QPS, storage, read/write ratio, bandwidth. | 5 min |
| 3. Define the API | Endpoints, verbs, request/response shapes. | 5 min |
| 4. Data model | Tables/collections, keys, indexes, relationships. | 5 min |
| 5. High-level design | Draw components: LB, app servers, cache, DB, queue. | 15 min |
| 6. Deep dive + trade-offs | Bottlenecks, scaling, failure modes, alternatives. | 10 min |

### Step 1 — Clarify requirements

Split into **functional** (what it does) and **non-functional** (how well it does it). Write them down where the interviewer can see.

```text
Functional (URL shortener):
  - Create a short URL from a long URL
  - Redirect a short URL to the original
  - (Optional) custom aliases, expiry, click analytics

Non-functional:
  - Scale: ~100M new URLs/day? or 1M? -> ASK
  - Read-heavy (redirects) vs write (creates) -> ratio?
  - Latency target: redirect < 100ms (p99)
  - Availability: 99.9%? Consistency: eventual is fine for analytics
```

Good clarifying questions: *Who are the users? Read-heavy or write-heavy? What's the expected scale at launch vs in 2 years? Do we need analytics? Strong or eventual consistency? Any auth/multi-tenancy?* Each answer narrows the design and shows seniority.

### Step 2 — Estimate scale (back-of-the-envelope)

Round aggressively. The goal is an order of magnitude, not precision. Memorize: **86,400 seconds/day ≈ 10^5**.

```text
Assume 100M new short URLs / day.
  Writes/sec  = 100M / 10^5 = 1,000 writes/sec
  Read:write  = 100:1 (redirects dominate) -> 100,000 reads/sec

Storage per URL row ~ 500 bytes (short code, long URL, owner, timestamps)
  Per day  = 100M * 500B = 50 GB/day
  Per year = 50GB * 365  ~ 18 TB/year  -> need partitioning/archival plan

Bandwidth (redirects): 100k reads/sec * ~500B = 50 MB/sec read out
```

Numbers like "100k reads/sec" immediately tell you a single Postgres box won't serve reads — you'll need a cache and read replicas. That realization is the *point* of the estimate.

---

## Core concept: designing a clean REST API

A REST API models **resources** (nouns), acted on by HTTP **verbs**. Keep URLs noun-based and let the verb carry the action.

### Resource modeling and verbs

```text
GET    /api/v1/links            # list (paginated)
POST   /api/v1/links            # create
GET    /api/v1/links/{code}     # retrieve one
PATCH  /api/v1/links/{code}     # partial update
DELETE /api/v1/links/{code}     # delete
GET    /api/v1/links/{code}/clicks   # sub-resource: analytics
```

Rules of thumb: plural nouns (`/links` not `/getLink`), nest sub-resources at most one level deep, and never put verbs in the path (`/links/123/activate` is a pragmatic exception for actions that aren't CRUD).

### Status codes that matter

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Successful GET/PATCH |
| 201 | Created | Successful POST that made a resource (return `Location` header) |
| 202 | Accepted | Async work queued, not done yet |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Malformed/invalid body |
| 401 / 403 | Unauthorized / Forbidden | Not logged in / not allowed |
| 404 | Not Found | Resource missing |
| 409 | Conflict | Duplicate, version conflict, idempotency replay |
| 422 | Unprocessable | Valid JSON but failed semantic validation (common in DRF/FastAPI) |
| 429 | Too Many Requests | Rate limit hit (include `Retry-After`) |
| 500 / 503 | Server error / unavailable | Bug / dependency down |

### Versioning

Version from day one — you cannot un-ship a breaking change to live clients. Most common is **URL path versioning** (`/api/v1/...`): explicit, cache-friendly, trivial to route. Alternatives are a header (`Accept: application/vnd.api+json; version=2`) or a query param (`?version=2`), but path versioning wins on readability in interviews.

### Pagination

Never return an unbounded list. Two strategies:

```jsonc
// Offset pagination — simple, but slow & unstable for deep pages
GET /api/v1/links?limit=20&offset=40
// Problem: OFFSET 100000 still scans 100k rows; new inserts shift pages.

// Cursor (keyset) pagination — scales to any depth, stable
GET /api/v1/links?limit=20&cursor=eyJpZCI6MTQ4Mn0=
{
  "data": [ /* ... */ ],
  "next_cursor": "eyJpZCI6MTUwMn0=",   // opaque, base64(last_seen_id)
  "has_more": true
}
```

Cursor pagination uses `WHERE id > :last_id ORDER BY id LIMIT 20`, which hits an index and stays O(limit) regardless of depth. Prefer it for large or fast-changing datasets.

### Idempotency in the API

A request is **idempotent** if making it twice has the same effect as once. `GET`, `PUT`, `DELETE` are idempotent by HTTP spec; `POST` is **not** — a retried "create order" can double-charge. Fix it with an **idempotency key** (covered in detail below).

Here's a FastAPI sketch tying these together — note the explicit status code, Pydantic v2 models, and cursor response:

```python
# main.py — FastAPI clean REST endpoint
from fastapi import FastAPI, HTTPException, Response, status
from pydantic import BaseModel, HttpUrl, ConfigDict

app = FastAPI(title="Link Shortener", version="1.0")

class LinkCreate(BaseModel):
    long_url: HttpUrl

class LinkOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # Pydantic v2: read ORM objects
    code: str
    long_url: HttpUrl

@app.post("/api/v1/links", response_model=LinkOut, status_code=status.HTTP_201_CREATED)
def create_link(payload: LinkCreate, response: Response):
    code = make_short_code(str(payload.long_url))   # service call
    response.headers["Location"] = f"/api/v1/links/{code}"
    return LinkOut(code=code, long_url=payload.long_url)

@app.get("/api/v1/links/{code}", response_model=LinkOut)
def get_link(code: str):
    link = repo_get(code)                            # repository call
    if link is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Link not found")
    return link  # from_attributes lets Pydantic read the ORM object directly
```

---

## Core concept: layered / clean architecture

The single most common code-quality mistake is **fat views/controllers** — business logic, DB queries, and HTTP handling tangled in one function. Separate concerns into layers so each is testable and swappable.

```text
   HTTP request
       │
 ┌─────▼──────┐   View / Router        — parse request, call service, format response.
 │   View     │                          NO business logic, NO raw queries.
 └─────┬──────┘
 ┌─────▼──────┐   Service               — business rules, orchestration, transactions.
 │  Service   │                          Framework-agnostic; the "brain".
 └─────┬──────┘
 ┌─────▼──────┐   Repository            — all data access lives here. Hides the ORM.
 │ Repository │                          Easy to mock in tests; swap DB without touching service.
 └─────┬──────┘
 ┌─────▼──────┐   Model                 — schema + simple invariants.
 │   Model    │
 └────────────┘
```

**Why bother?** The service layer has no idea whether it's behind Django or FastAPI, so you can unit-test business rules without spinning up an HTTP server or a database. It also stops the classic "logic duplicated between the API view and the admin action" bug.

### Django example (DRF view stays thin)

```python
# repositories.py — all ORM access here
from .models import Link

class LinkRepository:
    def get(self, code: str) -> Link | None:
        return Link.objects.filter(code=code).first()

    def create(self, code: str, long_url: str, owner_id: int) -> Link:
        return Link.objects.create(code=code, long_url=long_url, owner_id=owner_id)

# services.py — business logic, framework-agnostic
import secrets, string
from django.db import IntegrityError

ALPHABET = string.ascii_letters + string.digits  # 62 chars

class LinkService:
    def __init__(self, repo: LinkRepository):
        self.repo = repo

    def shorten(self, long_url: str, owner_id: int) -> Link:
        for _ in range(5):                       # retry on rare code collision
            code = "".join(secrets.choice(ALPHABET) for _ in range(7))
            try:
                return self.repo.create(code, long_url, owner_id)
            except IntegrityError:
                continue
        raise RuntimeError("Could not allocate a unique code")

# views.py — DRF view is thin: HTTP in, HTTP out
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class LinkCreateView(APIView):
    def post(self, request):
        service = LinkService(LinkRepository())
        link = service.shorten(request.data["long_url"], request.user.id)
        return Response({"code": link.code}, status=status.HTTP_201_CREATED)
```

The view knows nothing about how a code is generated; the service knows nothing about HTTP; the repository knows nothing about business rules. Each can be tested in isolation.

---

## Core concept: statelessness & horizontal scaling

A **stateless** app server keeps no client-specific data in local memory between requests — every request carries everything needed (auth token, IDs), and any server can handle any request. This is the prerequisite for **horizontal scaling**: put N identical app servers behind a **load balancer** and add more under load.

```text
            ┌──────────────┐
 clients ──▶│ Load Balancer│  (round-robin / least-conns; TLS termination)
            └──────┬───────┘
        ┌──────────┼──────────┐
   ┌────▼───┐ ┌────▼───┐ ┌────▼───┐    identical, stateless app servers
   │ app #1 │ │ app #2 │ │ app #3 │    (Gunicorn/Uvicorn workers)
   └────┬───┘ └────┬───┘ └────┬───┘
        └──────────┼──────────┘
              shared state lives OUTSIDE the app:
        ┌──────────▼─────────┐  ┌──────────────┐
        │  Redis (sessions,  │  │  PostgreSQL  │
        │  cache, rate limit)│  │   (durable)  │
        └────────────────────┘  └──────────────┘
```

**The cardinal rule:** never store session, cache, or uploaded files on a single app server's local disk/memory — move them to Redis / a shared DB / object storage (S3). If you keep sessions in process memory, "sticky sessions" pin a user to one server and you lose true horizontal scaling and graceful failover.

```python
# Django: keep sessions in Redis, not local memory — so any server can serve any user
# settings.py
CACHES = {"default": {"BACKEND": "django.core.cache.backends.redis.RedisCache",
                      "LOCATION": "redis://redis:6379/1"}}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

JWTs take this further — they're self-contained, so the server stores *no* session at all (at the cost of harder revocation). Either way: state lives outside the app process.

---

## Core concept: caching layers and a CDN

Caching trades freshness for speed and load reduction. Apply it in layers, closest-to-user first:

| Layer | Lives where | Caches | TTL |
|-------|-------------|--------|-----|
| Browser cache | Client | Static assets, GET responses | mins–days |
| CDN (CloudFront, Cloudflare) | Edge POPs | Static files, cacheable GETs | mins–hours |
| Reverse-proxy cache | Nginx/Varnish | Full responses | secs–mins |
| Application cache | Redis/Memcached | Query results, computed values, sessions | secs–hours |
| DB buffer cache | Postgres RAM | Hot pages | n/a |

A **CDN** serves static assets (and cacheable API responses) from edge servers physically near the user, slashing latency and offloading your origin. For the URL shortener, the redirect lookup is the hot path — cache it in Redis.

### The cache-aside pattern (most common)

```python
import json
import redis

r = redis.Redis(host="redis", port=6379, db=0)

def get_long_url(code: str) -> str | None:
    cached = r.get(f"link:{code}")
    if cached is not None:                 # cache HIT
        return cached.decode()
    link = repo_get(code)                  # cache MISS -> hit DB
    if link is None:
        return None
    r.setex(f"link:{code}", 3600, link.long_url)  # populate, 1h TTL
    return link.long_url
```

### Cache invalidation — the hard part

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

When the underlying URL changes or is deleted, the cache goes stale. Strategies: **TTL expiry** (simplest, accept brief staleness), **write-through / delete-on-write** (`r.delete(f"link:{code}")` whenever you update the row), and **versioned keys** (`link:{code}:v3`). Also guard against **cache stampede** (thundering herd): when a hot key expires, thousands of requests miss at once and slam the DB — mitigate with a short lock or "stale-while-revalidate".

---

## Core concept: database scaling

The database is the usual bottleneck. Reach for these in roughly this order:

### 1. Indexing & query tuning first

Before any fancy scaling, make sure your hot queries hit indexes (`EXPLAIN ANALYZE`). A missing index on `links.code` turns every redirect into a full table scan.

```sql
CREATE UNIQUE INDEX idx_links_code ON links (code);  -- O(log n) lookups
```

### 2. Connection pooling

Opening a Postgres connection is expensive (~1–5ms + memory per connection), and Postgres caps total connections (~100s). With many app servers × many workers, you exhaust them fast. A **pooler** (PgBouncer, or SQLAlchemy's built-in pool) reuses a small set of connections.

```python
# SQLAlchemy 2.x: configure the pool explicitly
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://user:pw@db/app",
    pool_size=10,        # persistent connections per process
    max_overflow=20,     # extra burst connections
    pool_pre_ping=True,  # validate before use (drops dead conns)
    pool_recycle=1800,   # recycle every 30 min to dodge server timeouts
)
```

### 3. Read replicas (scale reads)

Replicate writes from a **primary** to read-only **replicas**; route `SELECT`s to replicas, writes to the primary. Perfect for read-heavy systems like our shortener (100:1 read:write). Caveat: **replication lag** means a replica may be milliseconds-to-seconds stale — read-your-own-writes can break, so route a user's reads to the primary right after they write.

```python
# Django: define replicas and a router
# settings.py
DATABASES = {
    "default": {"HOST": "primary-db", "ENGINE": "django.db.backends.postgresql", "NAME": "app"},
    "replica": {"HOST": "replica-db", "ENGINE": "django.db.backends.postgresql", "NAME": "app"},
}
DATABASE_ROUTERS = ["app.routers.PrimaryReplicaRouter"]

# app/routers.py
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):  return "replica"   # reads -> replica
    def db_for_write(self, model, **hints): return "default"   # writes -> primary
    def allow_relation(self, *a, **k):      return True
    def allow_migrate(self, db, app_label, **hints): return db == "default"
```

### 4. Partitioning & sharding (scale writes / data size)

- **Partitioning**: split one table into chunks *within one DB* (e.g. by date range or hash). Postgres native partitioning keeps queries on one node but prunes scans to the relevant partition. Great for time-series like click events.
- **Sharding**: split data across *multiple DB servers* by a shard key (e.g. `hash(code) % N`). Scales writes and storage past one machine, but you lose easy cross-shard joins and transactions — a big complexity jump. Shard only when one box truly can't keep up.

```sql
-- Postgres range partitioning for an ever-growing click-events table
CREATE TABLE click_events (id bigserial, code text, ts timestamptz NOT NULL)
    PARTITION BY RANGE (ts);
CREATE TABLE click_events_2026_06 PARTITION OF click_events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- Old months can be detached/archived cheaply.
```

### 5. When to denormalize

Normalized schemas avoid duplication but require joins. Under heavy read load, **denormalize**: store a redundant copy (e.g. cache `click_count` on the `links` row instead of `COUNT(*)`-ing the events table every redirect). You trade write complexity and consistency risk for fast reads. Denormalize *late*, only for proven hot paths.

---

## Core concept: async processing with queues (Celery)

Anything slow or non-essential to the response should not block the request. Sending email, generating thumbnails, calling a third-party API, aggregating analytics — push it onto a **queue** and return immediately (often `202 Accepted`). **Celery** + a broker (Redis or RabbitMQ) is the Python standard; it's Laravel Queues/Horizon for Python.

```text
 POST /links ──▶ app server ──▶ enqueue task ──▶ [ Redis/RabbitMQ broker ]
        │                                                  │
        └── returns 201 instantly                          ▼
                                                  ┌──────────────────┐
                                                  │  Celery workers  │ (separate
                                                  │  process tasks   │  processes,
                                                  └──────────────────┘  scale independently)
```

```python
# celery_app.py
from celery import Celery

celery_app = Celery("app", broker="redis://redis:6379/0", backend="redis://redis:6379/1")

@celery_app.task(bind=True, max_retries=3, default_retry_delay=10)
def record_click(self, code: str, ip: str, ts: str):
    try:
        analytics_repo.insert_click(code, ip, ts)   # slow / external work
    except TransientError as exc:
        raise self.retry(exc=exc)                   # exponential-ish backoff

# in the request handler — fire and forget, request stays fast
from fastapi import Request
from fastapi.responses import RedirectResponse

def redirect(code: str, request: Request):
    long_url = get_long_url(code)
    record_click.delay(code, request.client.host, now_iso())  # non-blocking
    return RedirectResponse(long_url, status_code=302)
```

Key ideas: tasks should be **idempotent** (they may run more than once on retry), keep payloads small (pass an ID, not a whole object), and use separate queues/priorities for fast vs slow work. For *scheduled* jobs use **Celery Beat** (cron-like).

---

## Core concept: rate limiting & idempotency keys

### Rate limiting

Protect the system from abuse and accidental floods. The classic algorithm is the **token bucket**: each client has a bucket that refills at a fixed rate; each request spends a token; empty bucket → `429 Too Many Requests` with a `Retry-After` header. Store counters in Redis so the limit is shared across all app servers (a per-process limit is useless behind a load balancer).

```python
# Sliding-window-ish limiter in Redis: N requests per window per key
import time, redis
r = redis.Redis()

def allow_request(client_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f"rl:{client_id}:{int(time.time() // window)}"  # bucket per window
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)        # set TTL only on first hit
    return count <= limit            # False -> caller returns 429
```

DRF ships throttling out of the box (`AnonRateThrottle`, `UserRateThrottle`); FastAPI commonly uses `slowapi`. In an interview, mention *where* the limiter lives (edge/API gateway vs app) and that the counter must be centralized.

### Idempotency keys (safe retries for POST)

Clients retry on timeouts. Without protection, a retried `POST /orders` charges twice. The fix: the client sends a unique **`Idempotency-Key`** header; the server records it with the result and replays the stored response on any retry.

```python
# FastAPI: idempotent POST using Redis to remember keys
from fastapi import FastAPI, Header, HTTPException, status
import json, redis

app = FastAPI()
r = redis.Redis()

@app.post("/orders", status_code=status.HTTP_201_CREATED)
def create_order(payload: OrderIn, idempotency_key: str = Header(...)):
    cache_key = f"idem:{idempotency_key}"
    # SET ... NX -> only the first request wins the lock
    if not r.set(cache_key, "processing", nx=True, ex=86400):
        existing = r.get(cache_key)
        if existing in (None, b"processing"):   # still in flight (or just expired)
            raise HTTPException(status.HTTP_409_CONFLICT, "Request in progress")
        return json.loads(existing)          # replay stored result
    order = order_service.create(payload)    # do the real, side-effectful work once
    r.set(cache_key, json.dumps(order.to_dict()), ex=86400)
    return order
```

This makes `POST` *effectively* idempotent without changing its HTTP semantics — essential for payments, orders, and any write that triggers side effects.

---

## Core concept: observability

You cannot operate what you cannot see. The "three pillars" plus health checks:

- **Logging** — *what happened*. Use **structured (JSON) logs** with a `request_id` correlated across services, not free-text `print`. Centralize in ELK/Loki/Datadog.
- **Metrics** — *aggregate trends*. Counters/gauges/histograms (request rate, error rate, p50/p95/p99 latency, queue depth) via Prometheus → Grafana. Watch the **RED** method: Rate, Errors, Duration.
- **Tracing** — *one request across services*. **OpenTelemetry** propagates a trace ID so you can see time spent in app vs DB vs cache vs downstream call.
- **Health checks** — endpoints the load balancer/Kubernetes polls. Separate **liveness** (am I running?) from **readiness** (can I serve traffic? — DB and cache reachable?).

```python
# FastAPI structured logging + health checks
import logging, json, sys
from fastapi import FastAPI, Response, status

logger = logging.getLogger("app")
handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(logging.Formatter('{"level":"%(levelname)s","msg":"%(message)s"}'))
logger.addHandler(handler); logger.setLevel(logging.INFO)

app = FastAPI()

@app.get("/healthz")          # liveness — cheap, no deps; "is the process up?"
def healthz():
    return {"status": "ok"}

@app.get("/readyz")           # readiness — checks dependencies
def readyz(response: Response):
    deps = {"db": check_db(), "cache": check_redis()}
    if not all(deps.values()):
        response.status_code = status.HTTP_503_SERVICE_UNAVAILABLE
    return {"status": "ok" if all(deps.values()) else "degraded", "deps": deps}

@app.middleware("http")       # log every request with a correlation id
async def log_requests(request, call_next):
    import uuid, time
    rid = request.headers.get("x-request-id", str(uuid.uuid4()))
    start = time.perf_counter()
    resp = await call_next(request)
    logger.info(json.dumps({"request_id": rid, "path": request.url.path,
                            "status": resp.status_code,
                            "ms": round((time.perf_counter() - start) * 1000, 1)}))
    resp.headers["x-request-id"] = rid
    return resp
```

---

## Worked example: a URL shortener, end to end

Let's assemble everything into one design — this is exactly what you'd whiteboard.

**1. Requirements.** Create short URL, redirect, optional analytics. ~100M creates/day, 100:1 read:write, redirect p99 < 100ms, 99.9% availability, eventual consistency for analytics.

**2. Scale.** ~1k writes/sec, ~100k reads/sec, ~18 TB/year. Conclusion: redirects must be cache-served; reads need replicas; events table needs partitioning.

**3. API.**

```text
POST /api/v1/links            {long_url}        -> 201 {code, short_url}
GET  /{code}                                    -> 302 Location: <long_url>
GET  /api/v1/links/{code}/stats                 -> 200 {clicks, ...}
```

**4. Data model.**

```sql
CREATE TABLE links (
    code        VARCHAR(10) PRIMARY KEY,   -- base62 short code
    long_url    TEXT NOT NULL,
    owner_id    BIGINT,
    created_at  TIMESTAMPTZ DEFAULT now(),
    click_count BIGINT DEFAULT 0           -- denormalized hot counter
);
CREATE TABLE click_events (                -- partitioned by month (see above)
    id bigserial, code VARCHAR(10), ts TIMESTAMPTZ, ip INET
) PARTITION BY RANGE (ts);
```

**5. Short-code generation.** Two options to discuss:
- *Random base62* (7 chars = 62^7 ≈ 3.5 trillion combos): simple, retry on the rare collision (shown in the service above).
- *Counter + base62 encode* (e.g. from a Redis `INCR` or a DB sequence / Snowflake-style ID): collision-free and shorter codes, but the global counter can become a write hotspot — hand out ranges to each app server to fix that.

**6. High-level design.**

```text
                    ┌─────────┐
 user ─────────────▶│   CDN   │ (static + cacheable redirects at edge)
                    └────┬────┘
                    ┌────▼────────┐
                    │ Load Balancer│
                    └────┬────────┘
        ┌────────────────┼────────────────┐
   ┌────▼────┐      ┌────▼────┐      ┌────▼────┐   stateless app servers
   │  app    │      │  app    │      │  app    │   (Uvicorn/Gunicorn)
   └──┬───┬──┘      └──┬───┬──┘      └──┬───┬──┘
      │   │            │   │            │   │
      │   └────────────┴───┴───────────┘   │
      │            ┌───────▼──────┐         │  redirect: read Redis first
      │            │    Redis     │◀────────┘  (code -> long_url, sessions,
      │            │ cache + rate │            rate-limit counters)
      │            └──────────────┘
      ▼                                         ┌──────────────────┐
 ┌─────────┐     reads     ┌──────────┐         │ Celery workers   │ record_click,
 │ Primary │──replicate──▶ │ Replicas │         │ (analytics, etc) │ aggregate stats
 │   DB    │◀── writes     └──────────┘         └────────▲─────────┘
 └─────────┘                                             │
       └──────────── enqueue click events ──────────────┘ (via Redis/RabbitMQ broker)
```

**Redirect hot path:** LB → app → `GET redis link:{code}` (hit ~99%) → `302`. On miss, read replica, repopulate cache. Click recording is fired to Celery so the redirect stays sub-100ms. Writes (create link) go to the primary; reads to replicas; everything cacheable sits behind the CDN/Redis.

### Sketch: e-commerce orders API (same framework, different shape)

The framework transfers directly. Functional: browse catalog, cart, place order, pay, track. Hard parts and how they map to this lesson:
- **Place order = idempotent POST** with an `Idempotency-Key` so a retry never double-charges.
- **Inventory** needs strong consistency → row-level locking / `SELECT ... FOR UPDATE` on the primary, not a replica.
- **Payment** is a slow external call → enqueue to Celery, return `202`, update order status via webhook.
- **Order history** is read-heavy → replicas + cache.
- **Saga/state machine** for the order lifecycle (`pending → paid → shipped`), since you can't hold one DB transaction across the payment provider.

---

## Core concept: monolith vs microservices

| Dimension | Monolith | Microservices |
|-----------|----------|---------------|
| Deploy | One unit, simple | Many services, CI/CD per service |
| Dev speed (early) | Fast — one repo, easy refactor | Slow — boilerplate, contracts |
| Scaling | Scale the whole app | Scale hot services independently |
| Failure isolation | One bug can take down all | Blast radius contained |
| Transactions | Easy (one DB, ACID) | Hard (distributed, sagas) |
| Debugging | Single stack trace | Cross-service tracing needed |
| Team org | One team / coordinated | Teams own services (Conway's Law) |
| Ops overhead | Low | High (service mesh, discovery) |

**Interview-safe stance:** *Start with a well-structured (modular) monolith.* Most products never need microservices, and premature splitting buys you distributed-systems pain (network failures, eventual consistency, sagas, ops sprawl) before you have the scale or team size to justify it. Extract a microservice only when a clear seam appears — a component with different scaling needs, a different team owning it, or an independent deploy cadence. Saying "I'd begin with a modular monolith and extract services when there's a concrete reason" signals seniority far more than reflexively drawing 12 boxes.

---

## Common mistakes & gotchas

- **Jumping to boxes before clarifying.** Spend the first 5 minutes on requirements and scale. Designing the wrong system perfectly still fails.
- **Fat views/controllers.** Business logic in the view is untestable and gets duplicated. Push it into services.
- **Stateful app servers.** Sessions/cache/uploads in local memory or disk break horizontal scaling and failover. Externalize to Redis/S3.
- **No pagination.** A `GET /links` that returns everything will OOM at scale. Always bound result sets; prefer cursor pagination for big tables.
- **Treating POST as idempotent.** Network retries double-create. Use idempotency keys for any side-effectful write.
- **Reading your own writes off a replica.** Replication lag makes the just-created row "disappear." Route post-write reads to the primary.
- **Cache stampede.** A hot key expiring slams the DB with simultaneous misses. Use locks / staggered TTLs / stale-while-revalidate.
- **Cache invalidation ignored.** Stale data after updates. Delete/version keys on write, or accept and document TTL staleness.
- **Sharding too early.** Sharding kills joins and cross-shard transactions. Exhaust indexing, caching, replicas, and partitioning first.
- **Per-process rate limits / counters.** Useless behind a load balancer; centralize state in Redis.
- **No observability story.** Interviewers ask "how do you know it's down?" Have logs + metrics + tracing + health checks ready.
- **Reflexive microservices.** Distributed complexity with no scale to justify it. Default to a modular monolith.

## Best practices

- Narrate trade-offs out loud — there is no single right answer; reasoning is the score.
- Version the API from day one (`/api/v1`). Use correct status codes and consistent error bodies.
- Keep app servers stateless; push all shared state to Redis / DB / object storage.
- Layer the code: thin views → services (business logic) → repositories (data access) → models.
- Cache the hot read path; pair every cache with a clear invalidation/TTL strategy.
- Optimize the DB in order: index → pool → replicas → partition → (last resort) shard. Denormalize only proven hot paths.
- Move slow/non-critical work to Celery; make tasks idempotent and pass IDs, not objects.
- Protect writes with idempotency keys and the system with centralized rate limiting.
- Ship structured logs with a correlation ID, RED metrics, tracing, and liveness/readiness probes.
- Start with a modular monolith; extract microservices only for a concrete, demonstrated reason.

## Interview Q&A

**Q: Walk me through how you'd approach a system-design question you've never seen.**
**A:** Six steps. (1) Clarify functional and non-functional requirements — ask about users, scale, read/write ratio, consistency, latency. (2) Estimate scale with back-of-the-envelope math (QPS, storage, bandwidth) to expose bottlenecks. (3) Define the API. (4) Define the data model with keys and indexes. (5) Draw the high-level components — load balancer, stateless app servers, cache, DB, queue. (6) Deep-dive bottlenecks and discuss trade-offs and failure modes. I narrate trade-offs throughout because the reasoning is what's graded.

**Q: What makes an app server "stateless," and why does it matter?**
**A:** Stateless means it stores no client-specific data in local memory or disk between requests — each request is self-describing (token, IDs), so any server can handle any request. It matters because it's the prerequisite for horizontal scaling and failover: you can add identical servers behind a load balancer, and one dying doesn't lose user state. Shared state (sessions, cache, uploads) lives in Redis, the DB, or object storage instead.

**Q: Offset vs cursor pagination — which and why?**
**A:** Offset (`LIMIT/OFFSET`) is simple but degrades on deep pages — `OFFSET 100000` still scans 100k rows — and shifts when rows are inserted. Cursor/keyset pagination uses `WHERE id > :last_id ORDER BY id LIMIT n`, hits an index, stays O(limit) at any depth, and is stable under inserts. I default to cursor pagination for large or fast-changing datasets, offset only for small admin tables where deep paging never happens.

**Q: How do you make a POST endpoint safe to retry?**
**A:** With an idempotency key. The client sends a unique `Idempotency-Key` header; the server atomically records it (e.g. Redis `SET key NX`) before doing the work, stores the result, and replays the stored response on any retry with the same key. This prevents double-charges/double-creates from network retries without violating HTTP's POST semantics. Returning `409` for an in-flight duplicate is also common.

**Q: Your redirect endpoint must serve 100k reads/sec at p99 < 100ms. How?**
**A:** Cache the `code → long_url` mapping in Redis and serve redirects straight from cache (~99% hit rate), falling back to a read replica on a miss and repopulating. Put cacheable responses behind a CDN. Keep app servers stateless behind a load balancer to scale out. Move click recording off the request path into Celery so it never adds latency. The DB primary only handles the ~1k writes/sec.

**Q: When do you reach for read replicas, and what's the catch?**
**A:** When the workload is read-heavy and a single DB can't serve the read QPS — route `SELECT`s to replicas and writes to the primary. The catch is replication lag: replicas can be milliseconds-to-seconds stale, so "read your own writes" can fail. Mitigate by routing a user's reads to the primary briefly after they write, or by reading from cache.

**Q: Difference between partitioning and sharding?**
**A:** Partitioning splits one logical table into chunks within a single database (e.g. by date range), so queries prune to relevant partitions but everything's still on one node. Sharding splits data across multiple database servers by a shard key, scaling writes and storage beyond one machine — but you lose easy cross-shard joins and transactions. Partition for manageability; shard only when one box genuinely can't handle the write/storage load.

**Q: When would you denormalize?**
**A:** When a read path is hot and the join/aggregate cost dominates. Example: store a `click_count` column on the `links` row instead of `COUNT(*)`-ing an events table on every redirect. You trade extra write work and a consistency risk for fast reads. Denormalize late and selectively, only for proven hot paths, and keep the normalized source of truth.

**Q: What work belongs on a queue like Celery, and what are the rules?**
**A:** Anything slow or non-essential to the response: emails, image processing, third-party API calls, analytics aggregation. The handler enqueues and returns immediately (often `202`). Rules: tasks must be idempotent (they can run twice on retry), payloads should be small (pass an ID, fetch fresh inside the task), use retries with backoff, and separate queues for fast vs slow work. Celery Beat handles scheduled jobs.

**Q: How do you implement rate limiting across a fleet of servers?**
**A:** Centralize the counter in Redis so the limit is global, not per-process (a per-process limit is meaningless behind a load balancer). Token bucket or fixed/sliding window: increment a per-client key with a TTL; over the limit returns `429` with `Retry-After`. Place it at the edge/API gateway when possible and in the app as a backstop. DRF has built-in throttles; FastAPI commonly uses slowapi.

**Q: What's your observability story?**
**A:** Three pillars plus health checks. Structured JSON logs with a correlated `request_id`; metrics (RED — rate, errors, duration with p95/p99, plus queue depth) via Prometheus/Grafana; distributed tracing with OpenTelemetry to follow one request across services; and separate liveness (`/healthz`) and readiness (`/readyz`, checks DB/cache) probes for the load balancer and orchestrator. This is how you detect, diagnose, and alert on problems.

**Q: Monolith or microservices?**
**A:** Start with a well-structured modular monolith. It's simpler to build, deploy, debug, and keep transactionally consistent, and most products never outgrow it. Microservices buy independent scaling and deployment and failure isolation, but cost you distributed-systems complexity — network failures, eventual consistency, sagas, and ops overhead. I extract a service only for a concrete reason: a component with very different scaling needs, an independent deploy cadence, or a separate owning team.

## Practice exercises

1. **Estimate.** A photo-sharing API expects 50M uploads/day, average photo 2 MB, read:write of 50:1. Compute writes/sec, reads/sec, and yearly storage. What two design conclusions follow?
2. **Clean layers.** Take this fat DRF view and refactor it into view → service → repository:
   ```python
   class OrderView(APIView):
       def post(self, request):
           items = request.data["items"]
           total = sum(i["price"] * i["qty"] for i in items)
           order = Order.objects.create(user=request.user, total=total)
           for i in items:
               OrderItem.objects.create(order=order, sku=i["sku"], qty=i["qty"])
           send_confirmation_email(request.user.email, order.id)  # slow!
           return Response({"id": order.id}, status=201)
   ```
3. **Idempotency.** Add an `Idempotency-Key` mechanism (Redis) to a FastAPI `POST /payments` so a retried request never charges twice. Return `409` if a request with the same key is still in flight.
4. **Cache-aside + invalidation.** Write a `get_profile(user_id)` that reads through Redis (1h TTL), and an `update_profile(user_id, data)` that writes the DB and invalidates the cache.
5. **Design.** Sketch a "Design Twitter timeline" answer through all six framework steps. Address the fan-out-on-write vs fan-out-on-read trade-off for celebrity accounts.

<details><summary>Solutions</summary>

**1. Estimate.**
```text
writes/sec = 50M / 10^5 = 500 uploads/sec
reads/sec  = 500 * 50  = 25,000 reads/sec
storage/day  = 50M * 2MB = 100 TB/day
storage/year = 100TB * 365 ~ 36.5 PB/year
Conclusions: (a) photos belong in object storage (S3) + CDN, NOT the DB —
the relational DB stores only metadata/URLs. (b) 25k reads/sec means
serve images via CDN and cache metadata in Redis; the DB never sees most reads.
```

**2. Clean layers.**
```python
# repositories.py
class OrderRepository:
    def create_order(self, user, total):
        return Order.objects.create(user=user, total=total)
    def add_item(self, order, sku, qty, price):
        return OrderItem.objects.create(order=order, sku=sku, qty=qty, price=price)

# services.py
from django.db import transaction
from .tasks import send_confirmation_email  # Celery task

class OrderService:
    def __init__(self, repo): self.repo = repo
    def place_order(self, user, items):
        total = sum(i["price"] * i["qty"] for i in items)
        with transaction.atomic():                    # all-or-nothing
            order = self.repo.create_order(user, total)
            for i in items:
                self.repo.add_item(order, i["sku"], i["qty"], i["price"])
        send_confirmation_email.delay(user.email, order.id)  # off the request path
        return order

# views.py
class OrderView(APIView):
    def post(self, request):
        order = OrderService(OrderRepository()).place_order(request.user, request.data["items"])
        return Response({"id": order.id}, status=201)
```

**3. Idempotency.**
```python
from fastapi import FastAPI, Header, HTTPException, status
import json, redis
app = FastAPI(); r = redis.Redis()

@app.post("/payments", status_code=201)
def pay(payload: dict, idempotency_key: str = Header(...)):
    k = f"idem:pay:{idempotency_key}"
    if not r.set(k, "processing", nx=True, ex=86400):   # someone got here first
        prev = r.get(k)
        if prev in (None, b"processing"):
            raise HTTPException(status.HTTP_409_CONFLICT, "Payment in progress")
        return json.loads(prev)                          # replay stored result
    result = charge_card(payload)                         # the one real side effect
    r.set(k, json.dumps(result), ex=86400)
    return result
```

**4. Cache-aside + invalidation.**
```python
import json, redis
r = redis.Redis()

def get_profile(user_id: int) -> dict:
    key = f"profile:{user_id}"
    hit = r.get(key)
    if hit:
        return json.loads(hit)
    profile = db_fetch_profile(user_id)        # miss -> DB
    r.setex(key, 3600, json.dumps(profile))    # populate, 1h TTL
    return profile

def update_profile(user_id: int, data: dict) -> None:
    db_update_profile(user_id, data)           # source of truth first
    r.delete(f"profile:{user_id}")             # invalidate; next read repopulates
```

**5. Design Twitter timeline (outline).**
```text
1. Requirements: post tweet, follow users, view home timeline (tweets of those you follow),
   reverse-chronological. Non-functional: read-heavy, timeline p99 < 200ms, eventual consistency OK.
2. Scale: say 500M tweets/day -> ~5k writes/sec; reads dominate -> ~300k timeline reads/sec.
3. API: POST /tweets ; GET /timeline?cursor=... ; POST /users/{id}/follow
4. Data model: tweets(id, user_id, text, ts), follows(follower_id, followee_id),
   timeline cache per user (list of tweet ids in Redis).
5. Design: LB -> stateless app -> Redis timeline cache -> DB (replicas) + Celery for fan-out.
6. Trade-off — fan-out-on-write: on each tweet, push its id into every follower's Redis
   timeline (fast reads, but a celebrity with 50M followers = 50M writes per tweet = hotspot).
   Fan-out-on-read: build the timeline at read time by querying followees (cheap writes,
   expensive reads). HYBRID: fan-out-on-write for normal users; for celebrities, merge their
   recent tweets in at read time. This hybrid is the senior answer.
```

</details>

## Cheat sheet

```text
INTERVIEW STEPS:  Clarify -> Estimate -> API -> Data model -> High-level -> Deep dive/trade-offs
ESTIMATE MATH:    86,400 s/day ~ 10^5 ; QPS = perDay / 10^5 ; storage = rows * bytes

REST:   plural nouns, verbs via HTTP. /api/v1/links  (version from day 1)
STATUS: 201 created(+Location) · 202 queued · 204 deleted · 400 bad · 401/403 auth
        404 missing · 409 conflict/idempotent-replay · 422 validation · 429 rate · 503 down
PAGE:   cursor (keyset) > offset for big/changing tables  (WHERE id > last LIMIT n)
IDEMPOTENT: GET/PUT/DELETE yes, POST no -> Idempotency-Key header + Redis SET NX

LAYERS: View/Router (HTTP) -> Service (logic) -> Repository (data) -> Model
        keep business logic OUT of views; service is framework-agnostic & unit-testable

SCALE OUT: stateless app servers + load balancer; state -> Redis / DB / S3 (never local)
CACHE:  Browser -> CDN -> proxy -> Redis(app) -> DB buffer.  cache-aside; invalidate on write
DB ORDER: index -> connection pool -> read replicas -> partition -> shard (last). denormalize hot reads
          replica lag -> read-your-writes from primary; sharding kills joins/txns
ASYNC:  Celery + Redis/RabbitMQ for slow work; return 202; tasks idempotent, pass IDs
RATE LIMIT: token bucket in Redis (centralized); 429 + Retry-After
OBSERVE: structured logs + request_id · metrics (RED: rate/errors/duration p95/p99)
         · tracing (OpenTelemetry) · /healthz (liveness) /readyz (readiness)

MONO vs MICRO: start modular monolith; split only for real scaling/team/deploy reasons
```
