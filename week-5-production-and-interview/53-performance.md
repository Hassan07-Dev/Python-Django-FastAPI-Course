# 53 — Performance Optimization

> Week 5 · Production & Interview

## What you will learn

- Why **"measure first, optimize second"** is the only sane workflow — and the tools to measure (`cProfile`, `django-debug-toolbar`, query counting).
- The famous **N+1 query problem**, how to spot it, and how to kill it with `select_related` / `prefetch_related`.
- Fetching *less* data: `only`, `defer`, `values`, `values_list`, `count`, `exists`.
- **Database indexing**, reading `EXPLAIN` / `EXPLAIN ANALYZE`, and finding slow queries.
- **Pagination** instead of loading the world; **bulk operations** instead of per-row saves.
- **Caching** strategically (per-view, low-level, computed values) and invalidation.
- **Connection pooling**, async for I/O-bound concurrency, and offloading heavy work to **Celery**.
- DRF **serialization cost**, **gunicorn/uvicorn worker tuning**, CPU- vs I/O-bound trade-offs.
- **CDN / static** optimization and **load testing** with Locust / k6.
- A reusable **bottleneck checklist** and how to *talk* about scaling in interviews.

## PHP/Laravel bridge

If you've used Laravel, almost all of this transfers — only the names change. Laravel's `Model::with('posts')` eager loading is Django's `prefetch_related('posts')`; the dreaded N+1 you fixed with `with()` is the exact same problem here. `DB::enableQueryLog()` / Laravel Debugbar is `django-debug-toolbar`. `Cache::remember()` is Django's `cache.get_or_set()`. Laravel queues (Horizon) map onto **Celery**; `php artisan queue:work` is `celery -A app worker`. PHP-FPM worker pools are conceptually `gunicorn`/`uvicorn` workers. Even `EXPLAIN` is identical — it's a database feature, not a framework one. So you already have the mental model; this lesson teaches you the Python spelling and a few Python-specific gotchas (the GIL, sync vs async).

---

## Core concept 1 — Measure before you optimize

The single most expensive mistake junior engineers make is *guessing*. You rewrite a "slow" function for two days, ship it, and the real bottleneck was a missing index that took 30 seconds to add. **Donald Knuth's rule still holds: premature optimization is the root of all evil.** The workflow is always:

1. **Reproduce** the slowness with a realistic data size.
2. **Measure** to find the *actual* hot spot (profiler, query log, timers).
3. **Fix the biggest thing**, then measure again.
4. Stop when it's fast enough. (You are optimizing for a budget, not infinity.)

### Quick-and-dirty timing

For a single block, `time.perf_counter()` beats `time.time()` (it's monotonic and high-resolution).

```python
import time

def slow_thing():
    return sum(i * i for i in range(2_000_000))

start = time.perf_counter()
result = slow_thing()
elapsed = time.perf_counter() - start
print(f"took {elapsed:.4f}s")   # => took 0.1100s (varies by machine)
```

A reusable context manager so you stop copy-pasting timers:

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(label: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.4f}s")

with timer("report build"):
    slow_thing()
# => report build: 0.1100s
```

### Profiling with cProfile

A timer tells you *that* something is slow; a profiler tells you *which function* is slow. `cProfile` ships with the standard library — no install needed.

```bash
# Profile a whole script and sort by cumulative time
python -m cProfile -s cumtime my_script.py
```

In code, profile just the suspicious region and print a clean top-10:

```python
import cProfile
import pstats
import io

def expensive():
    total = 0
    for i in range(100_000):
        total += sum(range(i % 100))
    return total

profiler = cProfile.Profile()
profiler.enable()
expensive()
profiler.disable()

s = io.StringIO()
stats = pstats.Stats(profiler, stream=s).sort_stats("cumulative")
stats.print_stats(10)   # top 10 by cumulative time
print(s.getvalue())
# => columns: ncalls  tottime  percall  cumtime  percall  filename:lineno(function)
```

Read the columns like this:

| Column | Meaning |
| --- | --- |
| `ncalls` | how many times the function was called (a huge number here often *is* the bug) |
| `tottime` | time **in this function only**, excluding sub-calls |
| `cumtime` | time in this function **including** everything it calls |
| `percall` | time per call |

> **Tip:** Sort by `tottime` to find a single slow function; sort by `cumtime` to find an expensive *call tree*. For line-by-line detail, install `line_profiler` (`@profile` decorator + `kernprof -l -v script.py`). For visual flame graphs, use `py-spy` (`py-spy record -o profile.svg -- python app.py`) — and `py-spy` can attach to a *running production process* without restarting it.

### Counting queries in Django

In web apps the bottleneck is almost always the **database**, not Python. `django-debug-toolbar` shows you every SQL query, its duration, and duplicates, right in the browser.

```bash
pip install django-debug-toolbar
```

```python
# settings.py (dev only!)
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE = ["debug_toolbar.middleware.DebugToolbarMiddleware", *MIDDLEWARE]
INTERNAL_IPS = ["127.0.0.1"]
```

For tests and scripts, assert the query count directly so a future change can't silently re-introduce an N+1:

```python
from django.test.utils import CaptureQueriesContext
from django.db import connection

with CaptureQueriesContext(connection) as ctx:
    list(Author.objects.all())          # run your code
print(len(ctx.captured_queries))        # => 1
for q in ctx.captured_queries:
    print(q["time"], q["sql"][:80])

# In a Django TestCase, use the built-in assertion:
# with self.assertNumQueries(1):
#     list(Author.objects.all())
```

---

## Core concept 2 — The N+1 query problem

This is the #1 performance bug in ORM-backed apps, and it's *the* question you'll be asked in interviews. It happens when you fetch a list (1 query) and then touch a related object for each row (N more queries).

```python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")
```

```python
# THE BUG: 1 query for books + 1 query per book to get its author = N+1 queries
books = Book.objects.all()              # 1 query
for book in books:
    print(book.title, book.author.name) # each .author hits the DB again!
# 1 + 100 = 101 queries for 100 books
```

### Fix 1: `select_related` — for ForeignKey / OneToOne (SQL JOIN)

`select_related` follows "to-one" relationships by doing a **SQL JOIN**, pulling everything in one query.

```python
books = Book.objects.select_related("author")   # single JOINed query
for book in books:
    print(book.title, book.author.name)         # no extra queries
# => 1 query total

# You can span multiple hops with __ :
Book.objects.select_related("author__publisher")
```

### Fix 2: `prefetch_related` — for ManyToMany / reverse FK (separate query + Python join)

`select_related` can't JOIN "to-many" relationships (it would multiply rows). `prefetch_related` runs a **second query** with `WHERE id IN (...)` and stitches the results together in Python.

```python
# Reverse FK: each author has many books
authors = Author.objects.prefetch_related("books")  # 2 queries total
for author in authors:
    for book in author.books.all():   # served from the prefetch cache, no new query
        print(author.name, book.title)
# => 2 queries no matter how many authors
```

`Prefetch` gives you control over the inner query (filter, order, or use a different manager):

```python
from django.db.models import Prefetch

recent = Prefetch(
    "books",
    queryset=Book.objects.filter(title__startswith="A").order_by("title"),
    to_attr="a_books",          # stored on author.a_books (a plain list)
)
authors = Author.objects.prefetch_related(recent)
for a in authors:
    print(a.name, [b.title for b in a.a_books])
```

> **Rule of thumb:** to-**one** → `select_related` (JOIN). to-**many** → `prefetch_related` (IN query). You can chain both: `Book.objects.select_related("author").prefetch_related("tags")`.

### The FastAPI / SQLAlchemy equivalent

Same bug, same fix, different spelling. SQLAlchemy 2.x uses loader options.

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload, joinedload

# joinedload  ≈ select_related (a JOIN, best for to-one)
stmt = select(Book).options(joinedload(Book.author))

# selectinload ≈ prefetch_related (a second IN query, best for to-many / collections)
stmt = select(Author).options(selectinload(Author.books))

rows = session.execute(stmt).scalars().unique().all()
```

---

## Core concept 3 — Fetch less data

The fastest query is the one that returns the fewest bytes. Don't `SELECT *` a 40-column table to read one name.

```python
# only(): load just these columns now; others lazy-load on access (careful — that can re-N+1)
Book.objects.only("id", "title")

# defer(): load everything EXCEPT these (good for big TextField/BLOB columns)
Book.objects.defer("full_text")

# values()/values_list(): skip model instantiation entirely — return dicts/tuples.
# Model objects are expensive to build; for read-only data this is much cheaper.
list(Book.objects.values("id", "title"))            # => [{'id': 1, 'title': 'Dune'}, ...]
list(Book.objects.values_list("title", flat=True))  # => ['Dune', 'Hyperion', ...]

# Don't load rows just to count or check existence:
Book.objects.filter(author_id=5).count()   # SELECT COUNT(*) — not len(list(...))
Book.objects.filter(author_id=5).exists()  # SELECT 1 ... LIMIT 1 — not "if queryset:"
```

```python
# Aggregate in the DB, not in Python:
from django.db.models import Count, Avg

Author.objects.annotate(num_books=Count("books")).values("name", "num_books")
Book.objects.aggregate(avg_pages=Avg("pages"))   # => {'avg_pages': 312.5}
```

> **Gotcha with `only`/`defer`:** accessing a *deferred* field later triggers a fresh query per object — that's a new N+1. Use them only when you're sure you won't touch the excluded fields.

---

## Core concept 4 — Indexing and EXPLAIN

An index is a sorted lookup structure (usually a B-tree). Without one, the DB does a **full table scan**; with one, it jumps straight to the rows. Index columns you **filter, join, or order by** frequently.

```python
class Order(models.Model):
    customer = models.ForeignKey("Customer", on_delete=models.CASCADE)  # FKs are auto-indexed
    status = models.CharField(max_length=20, db_index=True)             # single-column index
    created_at = models.DateTimeField()

    class Meta:
        indexes = [
            # Composite index — order matters! Good for: WHERE status=? ORDER BY created_at
            models.Index(fields=["status", "created_at"]),
            # Partial index — only index the rows you actually query (Postgres)
            models.Index(
                fields=["created_at"],
                name="pending_created_idx",
                condition=models.Q(status="pending"),
            ),
        ]
```

### Reading EXPLAIN

`EXPLAIN` shows the query *plan*; `EXPLAIN ANALYZE` actually *runs* it and reports real timing and row counts. This is database-level (works the same in psql), but Django exposes it on a queryset:

```python
print(Order.objects.filter(status="pending").explain())
# Postgres without index => "Seq Scan on order  ... rows=100000"   <-- BAD (full scan)
# Postgres with index    => "Index Scan using ..._idx ... rows=42" <-- GOOD

# Real timings:
print(Order.objects.filter(status="pending").explain(analyze=True))
```

```sql
-- Straight SQL, the way you'd run it in psql:
EXPLAIN ANALYZE
SELECT * FROM "order" WHERE status = 'pending' ORDER BY created_at DESC LIMIT 20;
```

What to look for in a Postgres plan:

| You see | Meaning |
| --- | --- |
| `Seq Scan` on a big table | no usable index — usually the bug |
| `Index Scan` / `Index Only Scan` | index is being used (Index Only = doesn't even touch the table) |
| `Bitmap Heap Scan` | index used for many matching rows |
| big gap between *estimated* and *actual* rows | stale statistics — run `ANALYZE` |

> **Index trade-off (interview gold):** indexes speed up reads but **slow down writes** (every INSERT/UPDATE must update the index) and consume disk. Don't index everything — index what your real queries need. To find slow queries in production, enable Postgres `log_min_duration_statement = 200` (log anything over 200ms) or use the `pg_stat_statements` extension.

---

## Core concept 5 — Pagination

Never `Book.objects.all()` into a response. If the table grows to a million rows, you'll OOM the server and time out the client. Always page.

```python
from django.core.paginator import Paginator

qs = Book.objects.order_by("id")     # ORDER BY is REQUIRED for stable paging
paginator = Paginator(qs, per_page=20)
page = paginator.get_page(2)         # clamps out-of-range safely
print(page.object_list, page.has_next())
```

DRF gives you pagination for free:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```

```python
# FastAPI: classic LIMIT/OFFSET
from fastapi import FastAPI, Query
app = FastAPI()

@app.get("/books")
def list_books(limit: int = Query(20, le=100), offset: int = 0):
    return session.execute(select(Book).limit(limit).offset(offset)).scalars().all()
```

> **Cursor / keyset pagination > OFFSET at scale.** `OFFSET 100000` still makes the DB scan and discard 100k rows. Keyset paging (`WHERE id > :last_id ORDER BY id LIMIT 20`) is O(page size) regardless of depth. DRF ships `CursorPagination` for exactly this.

---

## Core concept 6 — Bulk operations vs per-row saves

Each `.save()` is a round trip to the database. A thousand saves = a thousand round trips. Batch them.

```python
# SLOW: 1000 INSERT statements, 1000 round trips
for i in range(1000):
    Book.objects.create(title=f"Book {i}", author_id=1)

# FAST: a handful of multi-row INSERTs
books = [Book(title=f"Book {i}", author_id=1) for i in range(1000)]
Book.objects.bulk_create(books, batch_size=500)   # 2 queries

# Update many rows at once:
Book.objects.filter(author_id=1).update(featured=True)            # single UPDATE
# or change different objects and flush together:
Book.objects.bulk_update(books, ["title"], batch_size=500)
```

```python
# get_or_create / update_or_create avoid a manual SELECT-then-INSERT race:
author, created = Author.objects.get_or_create(name="Frank Herbert")
```

> **Caveat:** `bulk_create` / `bulk_update` skip `save()` overrides and `pre_save`/`post_save` signals (and `bulk_create` may not populate `pk` on some databases for older versions — modern Postgres does). If you depend on signals, you trade them for speed consciously. Also wrap large batch jobs in `transaction.atomic()` so they commit once, not per statement.

---

## Core concept 7 — Caching strategically

Caching is "remember the answer so you don't recompute it." It's the highest-leverage optimization — and the source of the second-hardest problem in computer science: invalidation. Cache things that are **expensive to compute** and **read far more than written**.

```python
# settings.py — use Redis in production
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
    }
}
```

```python
from django.core.cache import cache

# Low-level cache: get-or-set a computed value
def get_dashboard_stats():
    key = "dashboard:stats"
    stats = cache.get(key)
    if stats is None:                       # cache miss
        stats = expensive_aggregation()     # the slow thing
        cache.set(key, stats, timeout=300)  # cache for 5 minutes
    return stats

# Cleaner one-liner (computes & stores only on miss):
stats = cache.get_or_set("dashboard:stats", expensive_aggregation, timeout=300)
```

```python
# Per-view caching (whole HTTP response):
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)   # 15 minutes
def book_list(request):
    ...
```

**Invalidation** — delete or overwrite the key when the underlying data changes:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Book)
def bust_book_cache(sender, instance, **kwargs):
    cache.delete("dashboard:stats")
```

```python
# FastAPI: a tiny in-process LRU for pure, deterministic functions
from functools import lru_cache

@lru_cache(maxsize=1024)
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)
# For shared/cross-process caching in FastAPI, use Redis (redis-py / aiocache).
```

> **Caching layers, cheapest to most flexible:** `functools.lru_cache` (per-process, free) → Redis/Memcached (shared across workers) → HTTP/CDN caching (don't even hit your app). **Always set a TTL** unless you have rock-solid invalidation — a stale cache that never expires is a bug waiting to happen.

---

## Core concept 8 — Connection pooling

Opening a new DB connection per request is expensive (TCP + auth handshake). Reuse connections.

```python
# Django: persistent connections — keep each connection alive for 60s instead of
# closing it after every request.
DATABASES = {
    "default": {
        # ...
        "CONN_MAX_AGE": 60,
        "CONN_HEALTH_CHECKS": True,   # verify the connection is alive before reuse
    }
}
# Django 5.1+ also supports a real pool: "OPTIONS": {"pool": True} (with psycopg 3).
# At larger scale, put PgBouncer in front of Postgres for transaction-level pooling.
```

```python
# SQLAlchemy: a connection pool is built in. Size it to your worker concurrency.
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://user:pass@localhost/db",
    pool_size=10,        # persistent connections
    max_overflow=5,      # temporary extras under burst
    pool_pre_ping=True,  # check liveness before handing out a connection
    pool_recycle=1800,   # recycle connections every 30 min (avoids stale ones)
)
```

> **Watch the math:** total DB connections ≈ `workers × pool_size`. Four gunicorn workers × pool_size 10 = 40 connections — easily past a small Postgres `max_connections`. This is exactly why **PgBouncer** exists.

---

## Core concept 9 — Async for I/O, Celery for heavy work

Python has the **GIL** (Global Interpreter Lock): only one thread runs Python bytecode at a time. This shapes every scaling decision.

| Workload | Bottleneck | Best tool |
| --- | --- | --- |
| **I/O-bound** (DB, HTTP calls, files) | waiting on the network/disk | **async** (`asyncio`) or threads — the GIL is released during the wait |
| **CPU-bound** (image resize, ML, crypto) | the CPU itself | **processes** (`ProcessPoolExecutor`) or offload to **Celery** — threads/async don't help, the GIL blocks them |

### Async for concurrent I/O

If a request makes three independent HTTP calls sequentially, you wait for the sum. Run them concurrently and you wait for the *max*.

```python
import asyncio
import httpx

async def fetch(client, url):
    r = await client.get(url)
    return r.status_code

async def main():
    urls = ["https://httpbin.org/delay/1"] * 3
    async with httpx.AsyncClient() as client:
        # gather runs all three concurrently: ~1s total, not ~3s
        results = await asyncio.gather(*(fetch(client, u) for u in urls))
    print(results)   # => [200, 200, 200]

asyncio.run(main())
```

```python
# FastAPI is async-native. Use async def for I/O endpoints; run blocking calls in a
# threadpool so they don't stall the event loop.
from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool

app = FastAPI()

@app.get("/data")
async def get_data():
    async with httpx.AsyncClient() as client:
        r = await client.get("https://api.example.com/data")
    return r.json()

@app.get("/legacy")
async def legacy():
    # blocking_lib.call() would freeze the event loop — offload it:
    return await run_in_threadpool(blocking_lib.call)
```

> **Golden rule of async:** never call a *blocking* function (e.g. `requests.get`, `time.sleep`, heavy CPU) directly inside an `async def` — it freezes the entire event loop and every other request stalls. Use `await asyncio.sleep`, async libraries (`httpx`, async DB drivers), or `run_in_threadpool`.

### Offload heavy/slow work to Celery

Don't make a user wait 30 seconds for a PDF or an email batch during their HTTP request. Hand it to a background worker and return immediately.

```python
# tasks.py
from celery import Celery

app = Celery("proj", broker="redis://localhost:6379/0", backend="redis://localhost:6379/0")

@app.task
def send_report(user_id: int) -> str:
    build_and_email_report(user_id)   # slow: runs OFF the request thread
    return "sent"
```

```python
# In a view: enqueue and return instantly
def request_report(request):
    send_report.delay(request.user.id)          # non-blocking, returns a task id
    return JsonResponse({"status": "queued"})
```

```bash
celery -A tasks worker --loglevel=info        # run the worker process
celery -A tasks beat                          # for scheduled/periodic tasks
```

> Use Celery for: emails, report generation, image/video processing, third-party API calls that can be retried, scheduled jobs. The web request stays fast; the heavy lifting happens elsewhere and can scale independently.

---

## Core concept 10 — DRF serialization cost & worker tuning

### Serialization is not free

DRF's `ModelSerializer` is convenient but does a lot of per-field reflection. On large lists it can rival the DB query in cost.

```python
from rest_framework import serializers

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "author_id"]   # name exactly what you need, not "__all__"
```

Speed tips: list only the fields you return; never re-N+1 inside a `SerializerMethodField`; for read-heavy endpoints consider returning `.values()` dicts directly, or use a lighter serializer (`drf-flex-fields`) or even a Pydantic/`orjson` path. **Combine with the ORM:** the serializer can't fix an N+1 — set up `select_related`/`prefetch_related` on the view's queryset.

### Gunicorn / Uvicorn worker tuning

This is the production lever interviewers love. Match the worker model to the workload.

```bash
# WSGI (Django sync): start ~ (2 × CPU cores) + 1 worker processes
gunicorn myproject.wsgi:application --workers 5 --bind 0.0.0.0:8000 --timeout 30

# I/O-bound sync app: gevent/eventlet workers handle many concurrent connections
gunicorn myproject.wsgi:application --workers 4 --worker-class gevent --worker-connections 1000

# ASGI (FastAPI / async Django): gunicorn managing uvicorn workers.
# Modern uvicorn (>=0.30) moved the worker class to the separate `uvicorn-worker`
# package (pip install uvicorn-worker); the old uvicorn.workers.UvicornWorker path
# still works but is deprecated.
gunicorn myapp:app --workers 4 --worker-class uvicorn_worker.UvicornWorker

# Pure uvicorn (no gunicorn needed):
uvicorn myapp:app --workers 4 --host 0.0.0.0 --port 8000
```

| Setting | Guideline | Why |
| --- | --- | --- |
| sync workers | `(2 × cores) + 1` | each worker handles one request at a time; the GIL means more than ~cores of pure-Python workers won't add CPU throughput |
| async/gevent | fewer workers, high connections | one worker juggles thousands of waiting I/O sockets |
| `--timeout` | kill stuck workers | a hung worker shouldn't pin a slot forever |
| `--max-requests` | recycle workers periodically | mitigates slow memory leaks |

> **The core trade-off:** more workers = more memory (each is a full Python process) and more DB connections. CPU-bound? Scale **processes/cores**. I/O-bound? Scale **async concurrency**. Don't crank workers to 50 and starve the database.

---

## Core concept 11 — CDN, static files, and load testing

### Static / CDN

Your Python app should never serve CSS, JS, or images in production — that's wasted worker time. Push them to a CDN / object store and let it cache at the edge.

```bash
pip install whitenoise   # serve compressed, hashed static files efficiently
```

```python
# settings.py
MIDDLEWARE = ["whitenoise.middleware.WhiteNoiseMiddleware", *MIDDLEWARE]
STORAGES = {
    "staticfiles": {"BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"},
}
# In production, front this with a CDN (CloudFront/Cloudflare) or serve from S3.
```

CDN wins: edge caching near users, gzip/brotli compression, far-future cache headers on hashed filenames (`app.a1b2c3.js`), and offloading bandwidth from your origin.

### Load testing

You can't tune what you haven't load-tested. Find the breaking point *before* your users do.

```python
# locustfile.py  ->  run: locust -f locustfile.py --host http://localhost:8000
from locust import HttpUser, task, between

class ApiUser(HttpUser):
    wait_time = between(1, 3)

    @task(3)               # weighted: runs 3x as often as the next task
    def list_books(self):
        self.client.get("/api/books/?limit=20")

    @task(1)
    def book_detail(self):
        self.client.get("/api/books/1/")
```

```javascript
// k6 script -> run: k6 run script.js   (k6 is written in Go, scripts in JS)
import http from "k6/http";
import { check } from "k6";

export const options = {
  stages: [
    { duration: "30s", target: 50 },   // ramp up to 50 virtual users
    { duration: "1m",  target: 50 },   // hold
    { duration: "10s", target: 0 },    // ramp down
  ],
};

export default function () {
  const res = http.get("http://localhost:8000/api/books/");
  check(res, { "status 200": (r) => r.status === 200 });
}
```

> Watch **p95/p99 latency** (not just the average — averages hide tail pain), **requests/sec (throughput)**, and **error rate**. The goal is to find where latency hockey-sticks and errors begin, then fix that bottleneck and re-test.

---

## Common mistakes & gotchas

- **Optimizing without measuring.** You'll fix the wrong thing. Profile / count queries first, every time.
- **N+1 everywhere**, especially hidden inside DRF serializers and template loops. Always check the query count on list endpoints.
- **`if queryset:` or `len(qs)`** to test for rows — loads every row into memory. Use `.exists()` and `.count()`.
- **`only()` / `defer()` then touching the excluded field** — silently re-introduces an N+1.
- **Looping `.save()`** instead of `bulk_create` / `bulk_update` / a single `.update()`.
- **Caching with no TTL and no invalidation** — serves stale data forever.
- **Blocking calls inside `async def`** (`requests`, `time.sleep`, CPU loops) — freezes the whole event loop.
- **Throwing async at a CPU-bound problem** — the GIL means it won't help; use processes/Celery.
- **Cranking gunicorn workers sky-high** — exhausts RAM and DB connections; throughput drops.
- **`OFFSET` on deep pages** — the DB still scans all skipped rows; use keyset/cursor pagination.
- **Indexing every column** — slows writes and wastes disk; index what you actually query.
- **Serving static files through Python** in production instead of a CDN/WhiteNoise.

## Best practices

- Establish a baseline metric, change **one** thing, measure again. Keep a number you're optimizing toward.
- Add `assertNumQueries` to tests on list/detail endpoints so regressions can't slip in.
- Default every list endpoint to **paginated** and a sane max page size.
- Reach for `select_related`/`prefetch_related` the moment a view touches a relation.
- Push work to the database (`annotate`, `aggregate`, `update`) instead of computing in Python.
- Cache at the **cheapest effective layer**; always set a TTL; bust on write.
- Keep request handlers fast (< ~200ms); offload anything slow to **Celery**.
- Use **async** for I/O-bound concurrency, **processes/Celery** for CPU-bound work.
- Tune workers to the workload and the box; cap DB connections (PgBouncer if needed).
- **Load test before launch** and watch **p95/p99**, not averages.

## Interview Q&A

**Q: What is the N+1 query problem and how do you fix it in Django?**
**A:** You run one query to fetch a list, then trigger one extra query per row by accessing a related object — 1 + N total. Fix it with eager loading: `select_related` for to-one relations (ForeignKey/OneToOne, via a SQL JOIN) and `prefetch_related` for to-many relations (reverse FK/ManyToMany, via a second `IN` query joined in Python). You can verify the fix by counting queries with `assertNumQueries` or django-debug-toolbar.

**Q: When do you use `select_related` vs `prefetch_related`?**
**A:** `select_related` for to-**one** relationships — it does a JOIN and returns everything in one query. `prefetch_related` for to-**many** — a JOIN there would multiply rows, so it issues a separate `WHERE id IN (...)` query and stitches results in Python. They compose: `Book.objects.select_related('author').prefetch_related('tags')`.

**Q: How would you find out *why* an endpoint is slow?**
**A:** Measure, don't guess. In a web app I start with django-debug-toolbar or query counting — the DB is the usual culprit (N+1, missing index, fetching too much). For CPU-bound Python I use `cProfile` (sort by `cumtime`/`tottime`) or `py-spy` to attach to a live process. I check `EXPLAIN ANALYZE` on suspect queries. Then I fix the single biggest contributor and re-measure.

**Q: What does a database index do, and what's the downside?**
**A:** It's a sorted structure (usually a B-tree) that lets the DB find rows without scanning the whole table — turning a Seq Scan into an Index Scan. The downside: every write must also update the index, so indexes slow INSERT/UPDATE/DELETE and use disk. So you index columns you filter/join/order by, not every column.

**Q: Explain Python's GIL and how it affects scaling.**
**A:** The Global Interpreter Lock allows only one thread to execute Python bytecode at a time, so multithreading doesn't speed up CPU-bound work. For CPU-bound work you use multiple processes (or Celery). For I/O-bound work the GIL is released while waiting on the network/disk, so async or threads *do* give concurrency. This is why workload classification drives the tooling choice.

**Q: I/O-bound vs CPU-bound — how does that change your approach?**
**A:** I/O-bound (DB, HTTP, files) is dominated by waiting, so I use async/`asyncio.gather` or thread workers to overlap the waits — one process handles many in-flight requests. CPU-bound (encoding, ML, crypto) is dominated by computation, so I scale across cores with processes or offload to Celery; async won't help because of the GIL.

**Q: When do you reach for Celery?**
**A:** When work is too slow or heavy to do inside the request/response cycle — sending emails, generating reports/PDFs, image/video processing, retryable third-party calls, scheduled jobs. The view enqueues a task with `.delay()` and returns immediately; a separate worker pool does the work and can scale independently of the web tier.

**Q: How do you cache effectively and avoid serving stale data?**
**A:** Cache things expensive to compute and read far more than written, at the cheapest layer that works (`lru_cache` → Redis → CDN). Always set a TTL as a safety net, and actively invalidate on write — e.g. delete the key in a `post_save` signal. Choose a stable, namespaced key scheme. The hard part is invalidation, so I keep cached data scoped and TTLs short unless invalidation is bulletproof.

**Q: How many gunicorn workers should you run?**
**A:** A common starting point for sync workers is `(2 × CPU cores) + 1`, then load-test and adjust. For I/O-bound apps, gevent/async workers with high connection counts beat raw worker count. The constraints are memory (each worker is a full process) and DB connections (`workers × pool_size` must stay under `max_connections` — PgBouncer helps).

**Q: What's wrong with `OFFSET`-based pagination at scale?**
**A:** `OFFSET 100000` makes the database generate and discard 100k rows before returning your page, so deep pages get linearly slower. Keyset/cursor pagination (`WHERE id > :last_seen ORDER BY id LIMIT n`) is O(page size) regardless of depth because it seeks via an index. DRF's `CursorPagination` implements this.

**Q: How do you reduce the data a query returns?**
**A:** Select only needed columns with `only()`/`values()`/`values_list()`, skip big columns with `defer()`, use `count()`/`exists()` instead of materializing rows, and aggregate in the database with `annotate`/`aggregate` instead of in Python. For read-only endpoints, `values()` dicts skip the cost of building model instances entirely.

**Q: How do you talk about scaling a slow API in an interview?**
**A:** I work top-down: (1) measure to find the real bottleneck; (2) optimize the DB — indexes, kill N+1s, paginate, fetch less; (3) cache expensive read-heavy results; (4) move slow work off the request path into Celery; (5) scale horizontally with more workers/instances behind a load balancer, watching DB connection limits; (6) front static/cacheable content with a CDN; (7) load-test to confirm and watch p95/p99. The theme is *measure, fix the biggest thing, repeat.*

## Practice exercises

1. **Spot the N+1.** Given `Order` with a ForeignKey to `Customer` and a reverse relation `items`, write a view that lists each order's customer name and item count using exactly **2 queries total**. Prove it with `assertNumQueries`.

2. **Profile it.** Write a function that builds a 50k-element list of squares using a `for` loop with `.append()`, profile it with `cProfile`, then rewrite it as a list comprehension and compare `tottime`.

3. **Index decision.** You have `SELECT * FROM event WHERE tenant_id = ? AND created_at > ? ORDER BY created_at DESC`. Which index would you add, and in what column order? Justify using how composite indexes are read.

4. **Cache a computed value.** Implement `get_top_authors()` that returns the 5 authors with the most books, cached in Redis for 10 minutes, and invalidated whenever a `Book` is saved.

5. **Pick the concurrency model.** For each task, say async, threads, or processes/Celery and why: (a) call 20 external REST APIs, (b) resize 500 uploaded images, (c) generate a 40-page PDF on user request.

6. **Load test.** Write a Locust file that hits a paginated list endpoint and a detail endpoint with a 3:1 weighting, then describe which metric tells you the server is overloaded.

<details>
<summary>Solutions</summary>

**1. No N+1**
```python
from django.db.models import Count

def order_list(request):
    orders = (
        Order.objects
        .select_related("customer")      # to-one  -> JOIN (query 1)
        .prefetch_related("items")       # to-many -> IN query (query 2)
    )
    data = [
        {"id": o.id, "customer": o.customer.name, "items": len(o.items.all())}
        for o in orders
    ]
    return JsonResponse({"orders": data})

# Test:
# with self.assertNumQueries(2):
#     order_list(request)
# (Note: len(o.items.all()) is served from the prefetch cache — 0 extra queries.
#  DON'T use o.items.count() here: .count() ignores the prefetch cache and fires a
#  fresh SELECT COUNT(*) per order, re-introducing the N+1. To count in the DB
#  instead, annotate the queryset with Count("items") and read o.items__count.)
```

**2. Profiling**
```python
import cProfile, pstats

def with_loop():
    out = []
    for i in range(50_000):
        out.append(i * i)
    return out

def with_comprehension():
    return [i * i for i in range(50_000)]

cProfile.run("with_loop()", sort="tottime")
cProfile.run("with_comprehension()", sort="tottime")
# The comprehension shows lower tottime: it avoids 50k bound-method .append lookups
# and runs the loop in C rather than Python bytecode.
```

**3. Index decision**
```python
# Composite index on (tenant_id, created_at) — equality column FIRST, range/sort SECOND.
models.Index(fields=["tenant_id", "-created_at"])
# Why: a B-tree composite index is read left-to-right. tenant_id (=) narrows to one
# tenant, then created_at is already ordered within that tenant, so the DB satisfies
# both the WHERE range and the ORDER BY from the same index — no separate sort step.
```

**4. Cached computed value**
```python
from django.core.cache import cache
from django.db.models import Count
from django.db.models.signals import post_save
from django.dispatch import receiver

def get_top_authors():
    def compute():
        return list(
            Author.objects.annotate(n=Count("books")).order_by("-n").values("name", "n")[:5]
        )
    return cache.get_or_set("authors:top5", compute, timeout=600)  # 10 minutes

@receiver(post_save, sender=Book)
def bust_top_authors(sender, **kwargs):
    cache.delete("authors:top5")
```

**5. Concurrency model**
- (a) 20 external REST APIs → **async** (`asyncio.gather` with `httpx`). I/O-bound; overlap the waits in one process.
- (b) Resize 500 images → **processes / Celery**. CPU-bound; the GIL blocks threads/async, so spread across cores.
- (c) Generate a 40-page PDF on request → **Celery**. Too slow for the request cycle; enqueue, return "queued", deliver when done.

**6. Load test**
```python
from locust import HttpUser, task, between

class User(HttpUser):
    wait_time = between(1, 2)

    @task(3)
    def list_endpoint(self):
        self.client.get("/api/books/?limit=20")

    @task(1)
    def detail_endpoint(self):
        self.client.get("/api/books/1/")
# Overload signal: p95/p99 latency climbs sharply (hockey-sticks) and/or error rate
# rises while throughput (req/s) plateaus or drops — that's the saturation point.
```

</details>

## Cheat sheet

```text
WORKFLOW:   measure -> fix biggest thing -> measure again -> stop when fast enough
```

| Problem | Tool / Fix |
| --- | --- |
| Find slow Python | `cProfile` (`-s cumtime`), `py-spy`, `line_profiler` |
| Count SQL queries | django-debug-toolbar, `assertNumQueries`, `CaptureQueriesContext` |
| N+1 (to-one) | `select_related("fk")` — JOIN |
| N+1 (to-many) | `prefetch_related("rev")` — IN query (`joinedload`/`selectinload` in SQLAlchemy) |
| Fetch less | `only` / `defer` / `values` / `values_list` / `count` / `exists` |
| Aggregate | `annotate(Count(...))`, `aggregate(Avg(...))` |
| Slow query | add `Index`, read `EXPLAIN ANALYZE` (Seq Scan = bad) |
| Big result set | paginate; keyset/cursor for deep pages |
| Many writes | `bulk_create` / `bulk_update` / `.update()` in `transaction.atomic()` |
| Repeated compute | `cache.get_or_set(key, fn, timeout)`; bust on `post_save` |
| Conn overhead | `CONN_MAX_AGE`, SQLAlchemy pool, PgBouncer |
| I/O concurrency | `async` + `asyncio.gather`; `run_in_threadpool` for blocking calls |
| CPU / slow work | processes / **Celery** (`.delay()`) |
| Worker count | sync `(2×cores)+1`; async = fewer workers, high connections |
| Static files | WhiteNoise + CDN, hashed filenames, brotli/gzip |
| Load test | Locust / k6 — watch **p95/p99**, throughput, error rate |

```text
GIL:   1 thread runs Python at a time.
       I/O-bound  -> async / threads (GIL released while waiting)
       CPU-bound  -> processes / Celery (threads & async won't help)
```
