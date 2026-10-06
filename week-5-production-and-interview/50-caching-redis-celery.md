# 50 — Caching, Redis & Celery

> Week 5 · Production & Scaling

## What you will learn

- **Why** and **where** to cache, and why the two genuinely hard problems are *cache invalidation* and *naming things* (cache keys).
- The four caching **layers**: per-view, per-object, low-level API, and template fragment.
- Django's **cache framework** and its backends (LocMem for dev, Redis for prod).
- A practical **cache-key + TTL strategy**, plus invalidation patterns and the **cache stampede** problem.
- **Redis** as three things at once: a cache, a message broker, and a data-structure server.
- **Celery** for background and scheduled work: brokers, `@shared_task`, `delay()` / `apply_async()`, workers, the result backend, **retries**, and **Celery Beat** for periodic tasks.
- **Idempotency** and safe task design.
- When FastAPI's `BackgroundTasks` is enough vs. when you actually need Celery.
- How all of this maps onto **Laravel queues / Horizon** and Laravel's cache.

---

## PHP/Laravel bridge

You already know most of this conceptually. Laravel's `Cache::remember('key', 600, fn () => ...)` is exactly the "low-level cache API" you'll meet in Django, and Laravel's cache `tags()` are the invalidation-by-group trick. Laravel **queues** (`dispatch(new SendEmail($user))`, jobs implementing `ShouldQueue`, the `queue:work` worker, and **Horizon** for monitoring Redis queues) map almost one-to-one onto **Celery**: a Celery `@shared_task` is a Laravel Job, `task.delay(...)` is `dispatch(...)`, the Celery worker is `queue:work`, and **Celery Beat** is Laravel's `schedule()` / `schedule:run` cron scheduler. Laravel's `ShouldBeUnique` job marker is the same idea as Celery idempotency. The biggest mental shift: in Python, **Redis is one server doing three jobs** — cache store, queue broker, and result backend — whereas Laravel often hides those behind separate config sections.

---

## Core concept: why and where to cache

Caching trades **freshness for speed**. You store the result of an expensive operation (a DB query, an API call, a rendered template, a heavy computation) so the next request gets it for almost free.

A useful rule of thumb — cache when **all** of these hold:

- The work is **expensive** (slow query, external API, CPU-heavy serialization).
- The result is **read far more often than it changes** (high read:write ratio).
- A **slightly stale** answer is acceptable for a short window.

> Phil Karlton's famous line: *"There are only two hard things in Computer Science: cache invalidation and naming things."* Both bite you in caching specifically — naming = cache keys, invalidation = knowing when stale data must die. Take both seriously.

Where caching lives, from closest to the user to deepest:

| Layer | What it caches | Django tool |
|---|---|---|
| HTTP / CDN | Whole responses at the edge | `Cache-Control` headers (out of scope here) |
| Per-view | A full view's HTML/JSON response | `@cache_page` |
| Template fragment | A chunk of a template | `{% cache %}` tag |
| Per-object / low-level | Arbitrary Python values you choose | `cache.get/set` |
| Query / ORM | DB rows | usually low-level cache around the query |

---

## Core concept: Django cache backends

Configure caching in `settings.py` under `CACHES`. The default alias is `"default"`.

```python
# settings.py — development: in-process, no extra services
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
        "LOCATION": "dev-cache",            # name the in-memory store
        "TIMEOUT": 300,                     # default TTL in seconds (5 min)
    }
}
```

`LocMemCache` lives inside a single Python process. It's perfect for local dev and tests, but **does not share** across processes/servers — if you run 4 Gunicorn workers, you have 4 separate caches. Never rely on it in production.

For production use **Redis**. Django 4.0+ ships a built-in Redis backend (no third-party package needed):

```python
# settings.py — production: shared Redis cache
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",   # DB index 1 for cache
        "TIMEOUT": 300,
        "KEY_PREFIX": "myapp",                     # namespaces all keys
    }
}
```

```bash
# Run Redis locally with Docker — one command, no install
docker run -d --name redis -p 6379:6379 redis:7-alpine
redis-cli ping      # => PONG
```

> Note: a popular alternative backend is `django-redis` (`"BACKEND": "django_redis.cache.RedisCache"`). It predates the built-in one and adds extras like pattern deletion (`cache.delete_pattern("user:*")`) and per-key client access. For most apps the built-in backend is enough; reach for `django-redis` when you need its extras.

### TTL semantics (a common trap)

```python
cache.set("k", "v", timeout=300)   # expires in 300 s
cache.set("k", "v", timeout=None)  # cache FOREVER (until evicted)
cache.set("k", "v", timeout=0)     # expire immediately = effectively "don't cache"
```

`timeout=None` meaning *forever* and `timeout=0` meaning *never* trips up almost everyone. If you want "use the backend default," omit the argument entirely (`DEFAULT_TIMEOUT` sentinel), don't pass `None`.

---

## Core concept: the low-level cache API

This is the workhorse — you decide exactly what to store and under which key. It's the direct analogue of Laravel's `Cache` facade.

```python
from django.core.cache import cache

# Basic get / set
cache.set("answer", 42, timeout=60)
cache.get("answer")              # => 42
cache.get("missing")            # => None
cache.get("missing", "default") # => "default"  (supply a fallback)

# Atomic add: only sets if key is absent (great for locks/once-only flags)
cache.add("once", "yes", timeout=60)   # => True  (was empty)
cache.add("once", "no", timeout=60)    # => False (already present, untouched)

# get_or_set: the "remember" pattern — compute only on a miss
def expensive():
    return sum(range(10_000_000))      # pretend this is slow

total = cache.get_or_set("big_total", expensive, timeout=600)

# Bulk operations save round-trips to Redis
cache.set_many({"a": 1, "b": 2, "c": 3}, timeout=300)
cache.get_many(["a", "b", "x"])        # => {"a": 1, "b": 2}

# Atomic counters (Redis INCR under the hood) — perfect for rate limiting
cache.set("hits", 0)
cache.incr("hits")                     # => 1
cache.incr("hits", 5)                  # => 6

# Deletion
cache.delete("answer")
cache.delete_many(["a", "b"])
cache.clear()                          # nuke everything (careful!)
```

`cache.get_or_set(key, callable, timeout)` is the single most useful method. Compare to Laravel:

```php
// Laravel
$total = Cache::remember('big_total', 600, fn () => expensive());
```

```python
# Django — same idea
total = cache.get_or_set("big_total", expensive, timeout=600)
```

### A realistic service-layer cache (DRF-friendly)

```python
from django.core.cache import cache
from myapp.models import Product

def get_product_detail(product_id: int) -> dict:
    key = f"product:detail:{product_id}:v2"   # v2 = bump to invalidate on schema change
    data = cache.get(key)
    if data is None:                           # cache MISS
        product = Product.objects.select_related("category").get(pk=product_id)
        data = {
            "id": product.id,
            "name": product.name,
            "price": str(product.price),
            "category": product.category.name,
        }
        cache.set(key, data, timeout=600)      # 10-minute TTL
    return data
```

> Why store a **dict**, not the model instance? Caching ORM objects can drag in unexpected related state and bloats the payload. Cache plain, serializable data (dicts, lists, primitives). It pickles smaller and survives schema tweaks better.

---

## Core concept: per-view and template-fragment caching

### Per-view caching

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)                      # cache this view's response for 15 min
def article_list(request):
    ...
```

`cache_page` keys on the **full URL including query string**, and varies by a couple of headers. So `/articles/?page=2` and `/articles/?page=3` are cached separately — good. But beware: it caches *per user only if you make it*. Anything that differs by logged-in user must add `Vary` handling, otherwise user A sees user B's cached page. For per-user or auth'd content, prefer low-level caching with a user-scoped key.

For class-based views, wrap with `method_decorator`:

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.generic import ListView

@method_decorator(cache_page(60 * 15), name="dispatch")
class ArticleListView(ListView):
    model = Article
```

### Template fragment caching

```django
{% load cache %}
{% cache 500 sidebar request.user.id %}   {# TTL 500s, keyed also by user id #}
    {# expensive sidebar rendering here — recomputed only every 500s per user #}
{% endcache %}
```

The extra arguments after the fragment name become part of the cache key — that's how you scope a fragment per-user, per-locale, etc.

---

## Core concept: cache keys, TTLs, and invalidation

### Naming keys (the second hard problem)

A good key is **deterministic, scoped, and versioned**:

```
<entity>:<view>:<id>:<version>      e.g.  product:detail:42:v2
<entity>:list:<filter-hash>         e.g.  product:list:9f3a1c
user:<id>:cart                      e.g.  user:1001:cart
```

For keys built from variable filters, hash the inputs so the key stays short and valid:

```python
import hashlib, json

def list_cache_key(prefix: str, **params) -> str:
    raw = json.dumps(params, sort_keys=True)             # stable ordering
    digest = hashlib.md5(raw.encode()).hexdigest()[:12]  # short, deterministic
    return f"{prefix}:{digest}"

list_cache_key("product:list", category="books", page=2)
# => 'product:list:7b1e3c0a9d42'
```

> Django warns if a key contains spaces or control chars (memcached compatibility) and silently truncates very long keys via the key function. Keep keys short and ASCII.

### Choosing TTLs

| Data | Suggested TTL | Why |
|---|---|---|
| Rarely-changing config / reference lists | hours–forever | cheap to keep, rarely stale |
| Product/catalog detail | 5–15 min | tolerable staleness, high read rate |
| User dashboard aggregates | 30–120 s | freshness matters more |
| Search results | 1–5 min | expensive but changes often |
| Anything financial / security-sensitive | usually **don't cache** | correctness > speed |

A short TTL is itself a cheap invalidation strategy: even if you never explicitly delete a key, it self-heals within the TTL window.

### Invalidation patterns

**1. TTL expiry (passive).** Simplest. Accept bounded staleness; do nothing on write.

**2. Write-through / explicit delete (active).** On every write, delete the affected key(s) with a Django signal:

```python
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from django.core.cache import cache
from myapp.models import Product

@receiver([post_save, post_delete], sender=Product)
def clear_product_cache(sender, instance, **kwargs):
    cache.delete(f"product:detail:{instance.id}:v2")
    # List caches are harder — see "key versioning" below.
```

**3. Key versioning (namespace invalidation).** When you can't enumerate all dependent keys (e.g. dozens of filtered list caches), bump a version number that's baked into every related key. Old keys become unreachable and expire naturally.

```python
def product_list_version() -> int:
    return cache.get_or_set("product:list:ver", 1, timeout=None)

def bump_product_list_version():
    try:
        cache.incr("product:list:ver")      # atomic
    except ValueError:                       # key absent
        cache.set("product:list:ver", 2, timeout=None)

def product_list_key(**params) -> str:
    ver = product_list_version()
    return list_cache_key(f"product:list:v{ver}", **params)
# Call bump_product_list_version() whenever any product changes.
```

This is the manual version of Laravel's cache **tags** — tags let you `Cache::tags(['products'])->flush()`; key-versioning achieves the same "invalidate a whole group" effect on backends without tag support.

### The cache stampede (thundering herd)

When a hot key expires, *every* concurrent request misses at once and they all recompute simultaneously — hammering the DB precisely when you wanted relief.

Mitigations:

- **Lock around recomputation** using `cache.add()` as a mutex — only the first request rebuilds; others briefly serve stale or wait.
- **Probabilistic early expiry** — refresh a bit before the TTL with random jitter so misses spread out.
- **Pre-warm** hot keys with a Celery task before they expire.

```python
def get_with_lock(key, builder, timeout=600, lock_ttl=30):
    val = cache.get(key)
    if val is not None:
        return val
    lock_key = f"lock:{key}"
    if cache.add(lock_key, "1", timeout=lock_ttl):   # only the winner rebuilds
        try:
            val = builder()
            cache.set(key, val, timeout=timeout)
        finally:
            cache.delete(lock_key)
        return val
    # Lost the lock: serve stale if any, else compute as a fallback
    return cache.get(key) or builder()
```

---

## Core concept: Redis — one server, three jobs

Redis is an in-memory key-value store that pulls triple duty in a typical Python stack:

1. **Cache** — what we just used via Django's cache framework.
2. **Message broker** — the queue Celery pushes tasks onto.
3. **Data-structure server** — strings, hashes, lists, sets, sorted sets, with atomic ops; great for counters, leaderboards, rate limiters, ephemeral locks.

Keep them on separate **logical DBs** (Redis has 16 by default, `/0`–`/15`) so a `cache.clear()` can't wipe your task queue:

```python
# Convention: split responsibilities by DB index
REDIS_HOST = "redis://127.0.0.1:6379"
CACHES = {"default": {"BACKEND": "django.core.cache.backends.redis.RedisCache",
                      "LOCATION": f"{REDIS_HOST}/1"}}      # /1 = cache
CELERY_BROKER_URL = f"{REDIS_HOST}/0"                       # /0 = broker
CELERY_RESULT_BACKEND = f"{REDIS_HOST}/2"                   # /2 = results
```

A tiny taste of Redis-as-data-structure (via the `redis` Python client) for an atomic rate limiter:

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, db=3)

def allow_request(user_id: int, limit=100, window=60) -> bool:
    key = f"rl:{user_id}"
    count = r.incr(key)         # atomic increment; creates key at 0 then +1
    if count == 1:
        r.expire(key, window)   # set TTL only on first hit of the window
    return count <= limit
```

---

## Core concept: Celery for background & scheduled work

Some work shouldn't block the HTTP response: sending email, generating PDFs/thumbnails, calling slow third-party APIs, recomputing aggregates. **Celery** runs these out-of-band on separate **worker** processes, with the broker (Redis) holding the queue.

```
Web request → task.delay(args)  →  [ Redis broker queue ]  →  Celery worker runs it
                                                                     │
                                                       (optional) result backend stores return value
```

### 1. App setup

```python
# proj/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "proj.settings")

app = Celery("proj")
# Pull all CELERY_* settings from Django settings, namespaced:
app.config_from_object("django.conf:settings", namespace="CELERY")
# Auto-discover tasks.py in every installed app:
app.autodiscover_tasks()
```

```python
# proj/__init__.py — ensure the Celery app loads when Django starts
from .celery import app as celery_app
__all__ = ("celery_app",)
```

```python
# settings.py — Celery configuration (CELERY_ prefix matches namespace above)
CELERY_BROKER_URL = "redis://127.0.0.1:6379/0"
CELERY_RESULT_BACKEND = "redis://127.0.0.1:6379/2"
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_ACCEPT_CONTENT = ["json"]              # never accept pickle from the wire
CELERY_TIMEZONE = "UTC"
CELERY_TASK_ACKS_LATE = True                  # ack AFTER success (survive worker crash)
CELERY_TASK_REJECT_ON_WORKER_LOST = True
```

> Why `acks_late=True`? By default a task is acknowledged the moment a worker picks it up; if that worker dies mid-task, the message is lost. With `acks_late`, the broker only removes the message after the task finishes, so a crash lets another worker retry it. The trade-off: tasks **must be idempotent**, because a task may run more than once.

### 2. Defining tasks

Use `@shared_task` (not `@app.task`) so tasks aren't bound to a specific app instance — this is the standard for reusable Django apps.

```python
# myapp/tasks.py
from celery import shared_task
from django.core.mail import send_mail

@shared_task
def send_welcome_email(user_id: int) -> str:
    from myapp.models import User           # import inside to avoid app-loading issues
    user = User.objects.get(pk=user_id)
    send_mail("Welcome!", "Thanks for signing up.", "noreply@app.com", [user.email])
    return f"sent:{user.email}"             # stored in the result backend
```

> **Pass IDs, not objects.** Serialize a `user_id` (an int), not a `User` instance. The worker is a different process — it can't share your in-memory object, and by the time it runs the object may be stale. Re-fetch inside the task. This is the same discipline Laravel enforces with its `SerializesModels` trait (which also just stores the key).

### 3. Calling tasks

```python
# In a view / DRF endpoint:
from myapp.tasks import send_welcome_email

send_welcome_email.delay(user.id)           # shorthand: enqueue with args

# apply_async gives full control:
send_welcome_email.apply_async(
    args=[user.id],
    countdown=10,                           # wait 10s before running
    queue="emails",                         # route to a named queue
    expires=3600,                           # drop if not run within 1h
)

# Calling normally (no .delay) runs it SYNCHRONOUSLY in-process — usually a bug:
send_welcome_email(user.id)                 # runs now, blocks the request
```

`.delay(*args, **kwargs)` is just sugar for `.apply_async(args, kwargs)`.

### 4. Running the worker (and Beat)

```bash
# Start a worker (concurrency = number of child processes)
celery -A proj worker --loglevel=info --concurrency=4

# Listen to specific queues only
celery -A proj worker -Q emails,default --loglevel=info

# Start the periodic-task scheduler (Beat) — see below
celery -A proj beat --loglevel=info
```

### 5. Results (when you care about return values)

```python
result = send_welcome_email.delay(user.id)
result.id            # task UUID — store this to poll later
result.ready()       # => False until done
result.get(timeout=5)  # block for the return value — AVOID inside web requests!
result.status        # 'PENDING' | 'STARTED' | 'SUCCESS' | 'FAILURE' | 'RETRY'
```

> Don't call `result.get()` inside a request handler — you've just made your async task synchronous again and can deadlock if the worker pool is busy. For fire-and-forget email, skip the result backend entirely (`ignore_result=True`) to save Redis writes.

### 6. Retries

```python
@shared_task(
    bind=True,                          # gives access to `self`
    autoretry_for=(ConnectionError,),   # auto-retry on these exceptions
    retry_backoff=True,                 # exponential backoff: 1s, 2s, 4s, ...
    retry_backoff_max=600,
    retry_jitter=True,                  # randomize to avoid retry stampede
    max_retries=5,
)
def charge_customer(self, order_id: int):
    from myapp.services import payment_gateway
    try:
        payment_gateway.charge(order_id)
    except RateLimited as exc:
        # Manual retry with explicit delay (overrides autoretry):
        raise self.retry(exc=exc, countdown=30)
```

### 7. Celery Beat — periodic tasks (cron)

Beat is a scheduler process that periodically pushes tasks onto the broker. Two ways to define schedules:

```python
# settings.py — static schedule baked into config
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "purge-expired-sessions": {
        "task": "myapp.tasks.purge_sessions",
        "schedule": crontab(hour=3, minute=0),        # every day at 03:00 UTC
    },
    "refresh-dashboard-cache": {
        "task": "myapp.tasks.warm_dashboard_cache",
        "schedule": 300.0,                            # every 300 seconds
    },
}
```

For schedules editable from the Django admin at runtime, install `django-celery-beat` and use the `DatabaseScheduler`:

```bash
pip install django-celery-beat
# add "django_celery_beat" to INSTALLED_APPS, then migrate
celery -A proj beat -S django_celery_beat.schedulers:DatabaseScheduler
```

This is the analogue of Laravel's `app/Console/Kernel.php` `schedule()` method + a single `* * * * * php artisan schedule:run` cron entry. One Beat process replaces a crontab full of entries.

---

## Core concept: idempotency & task design

A task is **idempotent** when running it twice produces the same end state as running it once. Because workers crash, networks blip, and `acks_late` re-delivers, **assume every task can run more than once.**

Design rules:

- **Pass IDs, re-fetch.** Never trust serialized state to still be current.
- **Guard side effects.** Before charging a card, check whether *this* order is already paid:

```python
from django.db import transaction

@shared_task(bind=True, max_retries=3)
def charge_once(self, order_id: int):
    from myapp.models import Order
    with transaction.atomic():                  # select_for_update needs a transaction
        order = Order.objects.select_for_update().get(pk=order_id)
        if order.status == "paid":              # idempotency guard
            return "already_paid"               # safe no-op on re-run
        charge(order)                           # external side effect
        order.status = "paid"
        order.save(update_fields=["status"])
```

- **Use a dedup key** for "exactly once" intent (Redis `SET key NX` / `cache.add`) so a duplicate enqueue is dropped — the equivalent of Laravel's `ShouldBeUnique`.
- **Keep tasks small and single-purpose.** Chain them (`task_a.s() | task_b.s()`) rather than building one giant task.
- **Set time limits** (`CELERY_TASK_TIME_LIMIT`) so a hung task can't pin a worker forever.

---

## Core concept: BackgroundTasks vs Celery (especially FastAPI)

Not every async job needs Celery. FastAPI ships `BackgroundTasks` for *lightweight, fire-and-forget* work that runs **in the same process, after the response is sent**.

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def write_audit_log(message: str):
    with open("audit.log", "a") as f:
        f.write(message + "\n")

@app.post("/signup")
async def signup(email: str, background_tasks: BackgroundTasks):
    # ... create user ...
    background_tasks.add_task(write_audit_log, f"signup:{email}")
    return {"status": "ok"}      # response returns immediately; log written after
```

| Concern | FastAPI `BackgroundTasks` | Celery |
|---|---|---|
| Runs where | Same web process | Separate worker process(es) |
| Survives a crash/restart | **No** (lost if process dies) | **Yes** (queued in broker) |
| Retries / scheduling | None | Built-in retries + Beat |
| Heavy / long jobs | Blocks the worker process | Designed for it; scale workers |
| Distribution across machines | No | Yes |
| Setup cost | Zero | Broker + worker + config |
| Good for | quick logging, sending one email, cache warm | PDFs, payments, batch jobs, periodic tasks |

**Rule:** use `BackgroundTasks` when losing the job on a crash is acceptable and the work is quick and local. Reach for Celery the moment you need durability, retries, scheduling, heavy compute, or horizontal scaling. (Django has no built-in equivalent of `BackgroundTasks`; for Django, anything beyond trivial almost always means Celery.)

---

## Common mistakes & gotchas

- **`timeout=None` means cache forever, `timeout=0` means don't cache.** Mixing these up silently breaks caching.
- **Relying on `LocMemCache` in production.** Multiple workers = multiple disconnected caches; nothing is actually shared.
- **Caching per-user content with `@cache_page`.** Without proper `Vary`, one user's page leaks to another. Use user-scoped low-level keys instead.
- **Caching ORM model instances** instead of plain dicts — bloated payloads and subtle staleness. Cache serializable data.
- **Forgetting invalidation entirely.** "Why is the price still wrong?" — because nothing deleted or versioned the key on write.
- **Passing model objects to tasks.** Pass the **ID**; re-fetch in the worker.
- **Calling a task without `.delay()`** — it runs synchronously in the request and blocks it.
- **Calling `result.get()` inside a web request** — re-synchronizes the task and can deadlock the worker pool.
- **Non-idempotent tasks with `acks_late`/retries** — double-charged customers, duplicate emails.
- **Letting `cache.clear()` and the broker share a Redis DB** — clearing the cache wipes queued tasks. Separate the DB indices.
- **Cache stampede on a hot key** — all requests miss at once; add a lock or jittered TTL.
- **Storing huge values in Redis** — Redis is RAM; cache small, hot data, not your whole table.

---

## Best practices

- Start with **short TTLs** as your default invalidation; add explicit invalidation only where staleness genuinely hurts.
- Adopt a **consistent key naming scheme** with a `KEY_PREFIX` and a `:vN` version segment so you can invalidate by bumping a version.
- Cache **plain serializable data**, computed close to the source (service layer), not at the view boundary.
- Use **separate Redis logical DBs** for cache / broker / results.
- Make every Celery task **idempotent**; pass IDs, guard side effects, set `acks_late=True` knowingly.
- Configure **retries with exponential backoff + jitter** for anything touching the network.
- Keep tasks **small, single-purpose, and time-limited**; compose with chains/groups.
- Use **named queues** to isolate slow jobs (e.g. `emails`, `reports`) from latency-sensitive ones.
- Monitor Celery (Flower, or Prometheus exporters) — the Horizon equivalent. Set `ignore_result=True` for fire-and-forget tasks.
- Don't cache security- or money-sensitive reads unless you've thought hard about correctness.

---

## Interview Q&A

**Q: What are the two classically "hard problems" in caching, and how do you handle each?**
**A:** Cache invalidation and naming (cache keys). For naming, I use deterministic, scoped, versioned keys (`product:detail:42:v2`) and hash variable filter inputs. For invalidation I combine short TTLs (passive self-healing), explicit deletion via model signals on write, and key-versioning to invalidate whole groups of keys I can't enumerate.

**Q: When should you NOT cache?**
**A:** When data must always be fresh (financial balances, auth/security decisions), when the read:write ratio is low (you'd invalidate as often as you read), when results are tiny/cheap to compute, or when stale data could cause incorrect behavior. Caching trades freshness for speed — if you can't afford the staleness, don't.

**Q: Difference between `@cache_page`, template fragment caching, and the low-level cache API?**
**A:** `@cache_page` caches an entire view response keyed by URL — coarse but easy. Fragment caching (`{% cache %}`) caches a slice of a template, useful when only part of a page is expensive. The low-level API (`cache.get/set/get_or_set`) caches arbitrary values under keys you control — the most flexible and the right tool for per-user or per-object data.

**Q: Why is `LocMemCache` unsuitable for production?**
**A:** It's per-process and in-memory. With multiple Gunicorn/uvicorn workers or multiple servers, each process has its own isolated cache, so hit rates collapse and invalidation in one process doesn't affect others. Use a shared store like Redis in production.

**Q: What is a cache stampede and how do you prevent it?**
**A:** When a hot key expires, all concurrent requests miss simultaneously and recompute at once, hammering the DB. Mitigations: a mutex lock so only the first request rebuilds (others serve stale or wait), probabilistic early expiry with jitter to spread refreshes, and pre-warming hot keys via a scheduled task before they expire.

**Q: Explain Celery's architecture — broker, worker, result backend.**
**A:** The producer (web app) serializes a task message and pushes it to the **broker** (Redis or RabbitMQ), a queue. One or more **worker** processes pull messages and execute the task code. If you need return values, they're stored in the **result backend** (Redis/DB) keyed by task ID. Broker = the queue; result backend = where answers live; they're separate concerns.

**Q: `delay()` vs `apply_async()`?**
**A:** `delay(*args, **kwargs)` is convenience sugar for the common case. `apply_async()` exposes full control: `countdown`/`eta` for delayed execution, `queue` for routing, `expires`, retry policy, priority, and serializer. Use `delay` for simple enqueues, `apply_async` when you need scheduling or routing.

**Q: Why must Celery tasks be idempotent?**
**A:** Workers crash, networks fail, and with `acks_late` or retries a message can be re-delivered and executed more than once. If a task isn't idempotent you risk double charges or duplicate emails. I pass IDs and re-fetch, guard side effects with status checks, and use dedup keys for exactly-once intent.

**Q: Why pass IDs to tasks instead of model instances?**
**A:** The worker is a separate process and can't share your in-memory object; serializing the whole object is wasteful and the data may be stale by execution time. Passing the primary key and re-fetching inside the task guarantees fresh data and small messages — the same reason Laravel's `SerializesModels` stores only the key.

**Q: What does Celery Beat do, and how is it different from a worker?**
**A:** Beat is a scheduler: on a cron/interval it pushes task messages onto the broker. It does **not** execute tasks — workers do. You run exactly one Beat process plus N workers. It replaces a sprawling crontab, like Laravel's `schedule()` + a single `schedule:run` cron entry.

**Q: When is FastAPI's `BackgroundTasks` enough, and when do you need Celery?**
**A:** `BackgroundTasks` runs in the same process after the response and is great for quick fire-and-forget work (a log line, one email) where losing it on a crash is acceptable. It has no durability, retries, scheduling, or distribution. The moment you need any of those — heavy compute, guaranteed execution, periodic jobs, scaling across machines — use Celery.

**Q: How does all this map to Laravel?**
**A:** Django's `cache.get_or_set` ≈ `Cache::remember`; key-versioning ≈ Laravel cache tags. Celery `@shared_task` ≈ a queued Job; `task.delay()` ≈ `dispatch()`; the Celery worker ≈ `queue:work`; Celery Beat ≈ `schedule()` + `schedule:run`; idempotency/dedup keys ≈ `ShouldBeUnique`; Flower ≈ Horizon. The main difference is that Redis often plays cache, broker, and result backend simultaneously.

---

## Practice exercises

1. **Low-level remember.** Write a function `get_user_stats(user_id)` that returns a dict of (precomputed) stats, cached for 120 seconds under a user-scoped, versioned key. On a cache miss it should compute and store; on a hit it should not recompute.

2. **Invalidation by signal.** Add a `post_save` signal for a `Product` model that deletes its detail cache key and bumps a `product:list:ver` counter so all filtered list caches become unreachable.

3. **Stampede-safe builder.** Implement `get_or_build(key, builder, ttl)` that uses `cache.add()` as a lock so only one caller rebuilds a missing hot key; others serve stale if present, else build as a fallback.

4. **A retrying Celery task.** Define `@shared_task` `sync_invoice(invoice_id)` that calls a flaky external API, auto-retries up to 4 times with exponential backoff + jitter on `ConnectionError`, and is idempotent (skips if the invoice is already `synced`).

5. **Periodic warm-up.** Add a Celery Beat entry that runs `warm_dashboard_cache` every 5 minutes, and sketch that task so it pre-computes and stores the dashboard payload to avoid stampedes.

6. **Pick the tool.** For each, say `BackgroundTasks` or `Celery` and why: (a) writing an audit log line after signup; (b) generating a 50-page PDF report; (c) sending a nightly digest email to all users; (d) charging a credit card.

<details><summary>Solutions</summary>

```python
# 1. Low-level remember with a user-scoped, versioned key
from django.core.cache import cache

def get_user_stats(user_id: int) -> dict:
    key = f"user:{user_id}:stats:v1"
    stats = cache.get(key)
    if stats is None:                       # MISS -> compute once
        stats = compute_user_stats(user_id)  # your expensive function
        cache.set(key, stats, timeout=120)
    return stats
# (cache.get_or_set would also work: cache.get_or_set(key, lambda: compute_user_stats(user_id), 120))
```

```python
# 2. Invalidation by signal
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from django.core.cache import cache
from myapp.models import Product

@receiver([post_save, post_delete], sender=Product)
def invalidate_product(sender, instance, **kwargs):
    cache.delete(f"product:detail:{instance.id}:v2")
    try:
        cache.incr("product:list:ver")      # bump -> old list keys unreachable
    except ValueError:
        cache.set("product:list:ver", 2, timeout=None)
```

```python
# 3. Stampede-safe builder
from django.core.cache import cache

def get_or_build(key, builder, ttl=600, lock_ttl=30):
    val = cache.get(key)
    if val is not None:
        return val
    if cache.add(f"lock:{key}", "1", timeout=lock_ttl):   # only winner rebuilds
        try:
            val = builder()
            cache.set(key, val, timeout=ttl)
        finally:
            cache.delete(f"lock:{key}")
        return val
    return cache.get(key) or builder()                    # loser: stale or fallback
```

```python
# 4. Retrying, idempotent Celery task
from celery import shared_task

@shared_task(
    bind=True,
    autoretry_for=(ConnectionError,),
    retry_backoff=True, retry_backoff_max=300, retry_jitter=True,
    max_retries=4,
)
def sync_invoice(self, invoice_id: int):
    from myapp.models import Invoice
    from myapp.services import external_api
    invoice = Invoice.objects.get(pk=invoice_id)
    if invoice.status == "synced":          # idempotency guard
        return "already_synced"
    external_api.push(invoice)              # may raise ConnectionError -> auto-retry
    invoice.status = "synced"
    invoice.save(update_fields=["status"])
    return "synced"
```

```python
# 5. Periodic warm-up
# settings.py
CELERY_BEAT_SCHEDULE = {
    "warm-dashboard": {
        "task": "myapp.tasks.warm_dashboard_cache",
        "schedule": 300.0,                   # every 5 minutes
    },
}

# myapp/tasks.py
from celery import shared_task
from django.core.cache import cache

@shared_task
def warm_dashboard_cache():
    payload = compute_dashboard_payload()    # the expensive aggregation
    cache.set("dashboard:summary:v1", payload, timeout=600)  # TTL > refresh interval
    return "warmed"
```

```text
6. Tool choice
(a) audit log line     -> BackgroundTasks: quick, local, OK to lose on crash.
(b) 50-page PDF        -> Celery: heavy/long, needs its own worker, retryable.
(c) nightly digest     -> Celery + Beat: scheduled, large, must be durable.
(d) charge a card      -> Celery: money = needs retries, durability, idempotency.
```

</details>

---

## Cheat sheet

```python
# --- Django cache (low-level) ---
from django.core.cache import cache
cache.set(key, val, timeout=300)        # timeout None=forever, 0=don't cache
cache.get(key, default=None)
cache.add(key, val, timeout)            # set only if absent (lock/once)
cache.get_or_set(key, callable, 300)    # == Laravel Cache::remember
cache.incr(key, n) / cache.decr(key)    # atomic counters
cache.set_many({...}) / cache.get_many([...]) / cache.delete_many([...])
cache.delete(key) / cache.clear()

# --- Backends (settings.py CACHES) ---
# dev:  django.core.cache.backends.locmem.LocMemCache   (per-process!)
# prod: django.core.cache.backends.redis.RedisCache  LOCATION "redis://host:6379/1"

# --- Per-view / fragment ---
@cache_page(60*15)                       # keyed by full URL
# {% load cache %}{% cache 500 name user.id %}...{% endcache %}
```

```python
# --- Celery task ---
from celery import shared_task
@shared_task(bind=True, autoretry_for=(ConnectionError,),
             retry_backoff=True, retry_jitter=True, max_retries=5)
def my_task(self, obj_id):              # pass IDs, not objects
    ...

my_task.delay(1)                        # enqueue (sugar)
my_task.apply_async(args=[1], countdown=10, queue="emails", expires=3600)
```

```bash
# --- Run ---
celery -A proj worker -l info -Q default,emails --concurrency=4
celery -A proj beat   -l info        # periodic scheduler (one process)
celery -A proj beat   -l info -S django_celery_beat.schedulers:DatabaseScheduler
```

| Concept | Django/Celery | Laravel |
|---|---|---|
| remember pattern | `cache.get_or_set` | `Cache::remember` |
| group invalidation | key versioning / `django-redis` patterns | cache tags |
| background job | `@shared_task` + `.delay()` | Job + `dispatch()` |
| worker | `celery worker` | `queue:work` |
| scheduler | Celery Beat | `schedule()` + `schedule:run` |
| dedup / once | `cache.add` / dedup key | `ShouldBeUnique` |
| monitoring | Flower | Horizon |
| lightweight async | FastAPI `BackgroundTasks` | — |

**Redis DB convention:** `/0` broker · `/1` cache · `/2` results — never share, or `cache.clear()` eats your queue.

**Golden rules:** cache small serializable data with short TTLs and versioned keys; pass IDs to tasks and make them idempotent; use `acks_late` + retries + backoff; one Beat, many workers.
