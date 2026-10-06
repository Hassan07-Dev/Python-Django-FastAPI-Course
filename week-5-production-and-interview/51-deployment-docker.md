# 51 — Deployment: Docker, Gunicorn & Uvicorn

> Week 5 · Production & Deployment

## What you will learn

- Why Python's built-in `runserver` / `uvicorn --reload` are **dev-only**, and what runs your app in production instead.
- The two serving protocols: **WSGI** (sync, classic Django) and **ASGI** (async, FastAPI / async Django) — and where Gunicorn and Uvicorn fit.
- How to choose the right **worker/process model** and how many workers to run.
- **12-factor** config: keeping secrets and environment-specific settings out of code.
- Serving **static files** in production (`collectstatic`, WhiteNoise, nginx, CDN).
- **Docker fundamentals**: image vs container, layers, and clean **multi-stage Dockerfiles** for both Django and FastAPI.
- `.dockerignore`, running as a **non-root user**, and **health checks**.
- **docker-compose** to wire app + Postgres + Redis together.
- Running **migrations on deploy**, plus a tour of **CI/CD** and the **cloud deployment landscape**.

## PHP/Laravel bridge

In PHP land you rarely "run" your app the way Python does — PHP-FPM spawns a pool of worker processes and **nginx/Apache** forwards each request to a free worker, then the process resets. Python is different: your app is a **long-lived process** that you start and keep running, so you need a dedicated **application server** to manage worker processes. **Gunicorn is the Python equivalent of PHP-FPM** — a process manager that runs N copies of your app and load-balances requests across them. Laravel's `php artisan serve` maps to Django's `runserver` / `uvicorn --reload` (dev only, never production). `.env` + `config()` in Laravel maps almost 1:1 to environment variables read by `django-environ` or Pydantic Settings. And `php artisan migrate` on deploy is exactly `python manage.py migrate`.

## Core concept: the production serving stack

Never expose `manage.py runserver` or `uvicorn --reload` to the internet. They are single-threaded (or lightly threaded), have no process supervision, serve static files inefficiently, and explicitly warn you they are not security-hardened.

A real stack has three layers:

```
Internet
   │  HTTPS (TLS)
   ▼
┌──────────────┐   reverse proxy: terminates TLS, serves static files,
│    nginx     │   gzip, rate-limit, buffers slow clients
└──────┬───────┘
       │  HTTP (localhost / private network)
       ▼
┌──────────────┐   application server: manages worker processes,
│  Gunicorn /  │   restarts crashed workers, load-balances
│   Uvicorn    │
└──────┬───────┘
       │  WSGI / ASGI (in-process function call)
       ▼
┌──────────────┐
│ Django /     │   your code
│ FastAPI app  │
└──────────────┘
```

Each layer does one job well:

| Layer | Responsibility |
|---|---|
| **nginx** (reverse proxy) | TLS/HTTPS, serve static & media files, gzip, buffer slow clients, rate-limiting, route multiple apps |
| **Gunicorn / Uvicorn** (app server) | Spawn & supervise worker processes, restart crashed workers, distribute requests |
| **Django / FastAPI** | Your business logic |

### Why a reverse proxy in front?

Gunicorn's own docs recommend running it behind a proxy. The reason is **slow clients**: if a client uploads slowly over a flaky mobile connection, an app worker is blocked the whole time. nginx **buffers** the full request, then hands the complete request to Gunicorn instantly — so workers stay busy doing real work. nginx also serves static files from disk far faster than Python ever could, and handles TLS so your app code never touches certificates.

## Core concept: WSGI vs ASGI

This is the single most important distinction for choosing your server.

**WSGI** (Web Server Gateway Interface) is the classic synchronous Python web standard. One request occupies one worker until it finishes. Perfect for traditional Django (sync views, ORM calls).

**ASGI** (Asynchronous Server Gateway Interface) is the modern async standard. A single worker can interleave many concurrent requests using `async`/`await`, and it also supports WebSockets and long-lived connections. Required for FastAPI and for async Django.

| | WSGI | ASGI |
|---|---|---|
| Style | Synchronous | Asynchronous (`async`/`await`) |
| Concurrency | One request per worker at a time | Many concurrent requests per worker |
| WebSockets | No | Yes |
| Used by | Classic Django, Flask | FastAPI, async Django, Starlette |
| Server | **Gunicorn** | **Uvicorn** (or Gunicorn + uvicorn workers) |

The servers map cleanly:

- **Gunicorn** = a WSGI server (and a great process *manager*).
- **Uvicorn** = an ASGI server (fast, async). Modern Uvicorn can run multiple workers itself with `uvicorn main:app --workers 4`.
- **Gunicorn with Uvicorn workers** = Gunicorn's battle-tested process management + Uvicorn's ASGI speed. A long-standing way to run FastAPI in production.

> **Modern note (worker class).** The old `uvicorn.workers.UvicornWorker` import is **deprecated as of Uvicorn 0.30**. Install the standalone package (`pip install uvicorn-worker`) and use `uvicorn_worker.UvicornWorker` instead. Equally valid today: skip Gunicorn entirely and run `uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4`, which now manages its own worker processes. The deprecated `tiangolo/uvicorn-gunicorn` base image is no longer recommended.

Every Django project ships both entry points (created by `startproject`):

```python
# myproject/wsgi.py  — used by Gunicorn for classic sync Django
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
application = get_wsgi_application()
```

```python
# myproject/asgi.py  — used by Uvicorn for async Django / WebSockets
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
application = get_asgi_application()
```

FastAPI has no separate file — your `app` object **is** the ASGI application:

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"status": "ok"}
```

### Running each server

```bash
# Django (sync) with Gunicorn — points at the WSGI callable "application" in wsgi.py
gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 3

# FastAPI with plain Uvicorn (simplest, fine for single-process / small apps)
uvicorn main:app --host 0.0.0.0 --port 8000

# FastAPI in production, option A: Uvicorn managing its own workers (simplest today)
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

# FastAPI in production, option B: Gunicorn managing Uvicorn workers
# (requires `pip install uvicorn-worker`; uvicorn_worker.UvicornWorker is the
#  modern class — uvicorn.workers.UvicornWorker is deprecated as of Uvicorn 0.30)
gunicorn main:app \
    --bind 0.0.0.0:8000 \
    --workers 4 \
    --worker-class uvicorn_worker.UvicornWorker

# Async Django with Uvicorn worker class (note: asgi, not wsgi)
gunicorn myproject.asgi:application \
    --bind 0.0.0.0:8000 \
    --workers 4 \
    --worker-class uvicorn_worker.UvicornWorker
```

> The syntax `module:callable` means "import `module`, then use the variable `callable`". `myproject.wsgi:application` = the `application` object in `myproject/wsgi.py`.

## Core concept: the worker/process model & how many workers

Because of the **GIL** (Global Interpreter Lock), a single Python process runs only one thread of bytecode at a time. To use multiple CPU cores you run **multiple worker processes**. Each Gunicorn worker is a separate OS process with its own memory and its own copy of your app.

### How many workers?

The classic Gunicorn formula for **sync (WSGI)** workers:

```
workers = (2 × CPU_cores) + 1
```

A 4-core machine → `(2 × 4) + 1 = 9` workers. The "+1" and "2×" exist because workers spend time blocked on I/O (DB, network), so you want a few more processes than cores to keep the CPU busy.

```bash
# Make it dynamic with an env var, falling back to a sensible default
gunicorn myproject.wsgi:application --workers ${WEB_CONCURRENCY:-3} --bind 0.0.0.0:8000
```

For **async (ASGI)** workers it's different. Each async worker handles many concurrent requests itself, so you usually want roughly **one worker per CPU core** (the I/O concurrency comes from `async`, not from extra processes):

```bash
gunicorn main:app --workers 4 --worker-class uvicorn_worker.UvicornWorker  # ~= core count
# or, without Gunicorn:
uvicorn main:app --workers 4  # ~= core count
```

### Threads vs workers vs async — which concurrency model?

| Workload | Best model | Why |
|---|---|---|
| CPU-bound, sync Django | More **processes** (workers) | Sidesteps the GIL across cores |
| I/O-bound, sync Django | Workers **+ threads** (`--threads`) | Threads release the GIL while waiting on I/O |
| I/O-bound async (FastAPI) | **Uvicorn workers** | One worker juggles thousands of awaited I/O ops |

```bash
# Sync Django doing lots of I/O: a few processes, each with threads
gunicorn myproject.wsgi:application --workers 3 --threads 4 --bind 0.0.0.0:8000
```

> **Memory tip:** each worker is a full copy of your app in RAM. 9 workers each using 150 MB = ~1.4 GB. On a small box, fewer workers may be the right call regardless of the formula. Always measure.

### Useful Gunicorn flags

```bash
# Note: a "\" must be the LAST character on its line, so comments can't trail it.
# What each flag does:
#   --timeout 30             kill+restart a worker stuck >30s
#   --graceful-timeout 30    time to finish in-flight requests on reload
#   --max-requests 1000      restart worker after N requests (curbs memory leaks)
#   --max-requests-jitter    randomize so workers don't all restart together
#   --access-logfile -       "-" = stdout (the 12-factor way)
gunicorn myproject.wsgi:application \
    --bind 0.0.0.0:8000 \
    --workers 3 \
    --timeout 30 \
    --graceful-timeout 30 \
    --max-requests 1000 \
    --max-requests-jitter 100 \
    --access-logfile - \
    --error-logfile -
```

You can also keep config in a file:

```python
# gunicorn.conf.py
import multiprocessing
import os

bind = "0.0.0.0:8000"
workers = int(os.getenv("WEB_CONCURRENCY", multiprocessing.cpu_count() * 2 + 1))
worker_class = os.getenv("GUNICORN_WORKER_CLASS", "sync")  # or "uvicorn_worker.UvicornWorker"
timeout = 30
max_requests = 1000
max_requests_jitter = 100
accesslog = "-"   # stdout
errorlog = "-"    # stderr
```

```bash
gunicorn myproject.wsgi:application -c gunicorn.conf.py
```

## Core concept: 12-factor config & environment variables

The [12-factor app](https://12factor.net) methodology says: **store config in the environment**, not in code. Anything that differs between dev/staging/prod — DB URL, secret key, debug flag, API keys — comes from environment variables. This means one identical image runs everywhere; only the env changes.

Never commit secrets. Locally use a `.env` file (git-ignored); in production inject real env vars via your platform's secret store.

### Django with `django-environ`

```bash
pip install django-environ
```

```python
# settings.py
import environ

env = environ.Env(
    DEBUG=(bool, False),          # (type, default) — default never enables debug
)
environ.Env.read_env()            # reads .env in dev; no-op if absent

SECRET_KEY = env("SECRET_KEY")    # raises if missing — fail loud, not silently insecure
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=[])

# One line parses postgres://user:pass@host:5432/dbname into Django's DATABASES dict
DATABASES = {"default": env.db("DATABASE_URL")}
CACHES = {"default": env.cache("REDIS_URL")}
```

```bash
# .env  (git-ignored — NEVER commit this)
DEBUG=True
SECRET_KEY=dev-only-not-secret
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgres://app:secret@localhost:5432/app
REDIS_URL=redis://localhost:6379/0
```

### FastAPI with Pydantic Settings (v2)

Pydantic v2 moved settings to a separate package. It gives you **typed, validated** config — wrong types fail at startup, not at 3 a.m. in production.

```bash
pip install pydantic-settings
```

```python
# config.py
from functools import lru_cache
from pydantic import PostgresDsn
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    # field names map to env vars (case-insensitive by default)
    database_url: PostgresDsn
    redis_url: str = "redis://localhost:6379/0"
    secret_key: str
    debug: bool = False

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")


@lru_cache  # build once, reuse — cheap dependency injection
def get_settings() -> Settings:
    return Settings()


# usage
settings = get_settings()
print(settings.debug)  # => False (typed bool, not the string "False")
```

> Note the **modern Pydantic v2** style: `SettingsConfigDict` + `model_config`, not the old v1 inner `class Config`. Use `settings.model_dump()` if you need a dict — never the deprecated `.dict()`.

## Core concept: static files in production

In dev, Django/runserver serves CSS/JS for you. In production it refuses to (`DEBUG=False`). You must explicitly collect and serve them.

```bash
# settings.py
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"   # where collectstatic dumps everything

# Run on every deploy: copies all app/admin static files into STATIC_ROOT
python manage.py collectstatic --noinput
```

Three ways to actually serve those files:

| Option | When to use | Notes |
|---|---|---|
| **WhiteNoise** | Simple deploys, PaaS, containers without nginx | Serves static straight from the Python app, with compression + far-future caching. Easiest. |
| **nginx** | You run your own nginx/VM | Fastest; nginx reads files from disk, Python never involved |
| **CDN** (CloudFront, etc.) | High traffic, global users | Offload entirely; often fronts S3 |

WhiteNoise is the pragmatic default for containerized Django:

```bash
pip install whitenoise
```

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",   # right after SecurityMiddleware
    # ... the rest
]

# Compress + hash filenames for cache-busting (Django 5.x STORAGES API)
STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

> FastAPI is typically a **JSON API** with no static files at all — frontends are separate SPAs. If you do serve files, mount `StaticFiles`, but in production let nginx/CDN handle them.

## Core concept: Docker fundamentals

**Image vs container** — the #1 thing to get straight:

- An **image** is a read-only, layered *blueprint* (your code + Python + deps, frozen). Think "class".
- A **container** is a running *instance* of an image. Think "object". You can start many containers from one image.

A **Dockerfile** builds an image. Each instruction creates a **layer**; Docker caches layers, so ordering matters — put rarely-changing steps (installing deps) **before** frequently-changing steps (copying your code), so a code change doesn't reinstall every dependency.

```bash
docker build -t myapp:1.0 .        # build image from Dockerfile in current dir
docker run -p 8000:8000 myapp:1.0  # run a container, map host:container port
docker ps                          # list running containers
docker logs <container_id>         # view stdout/stderr
docker exec -it <id> bash          # shell into a running container
```

### `.dockerignore` — keep junk out of the image

Just like `.gitignore`, but for the build context. Without it, Docker copies your whole directory (including `.git`, `venv`, secrets) into the build — slow and insecure.

```dockerignore
# .dockerignore
.git
.gitignore
__pycache__/
*.pyc
.venv/
venv/
.env
*.sqlite3
.pytest_cache/
.mypy_cache/
node_modules/
*.md
.dockerignore
Dockerfile
```

### Multi-stage build — why?

A **multi-stage** Dockerfile uses one "builder" stage to compile/install dependencies (which needs compilers, dev headers, build tools) and a clean "runtime" stage that copies only the finished artifacts. Result: a much smaller, more secure final image with no build toolchain shipped to production.

### Non-root user — why it matters

By default containers run as **root**. If an attacker breaks out of your app, they're root inside the container (and potentially closer to the host). Create and switch to an unprivileged user.

### A clean multi-stage Dockerfile — Django

```dockerfile
# ---------- Stage 1: builder ----------
FROM python:3.12-slim AS builder

# Python runs better in containers with these set:
# PYTHONDONTWRITEBYTECODE=1 -> don't write .pyc files
# PYTHONUNBUFFERED=1        -> stream logs straight to stdout (no buffering)
# NOTE: Docker does NOT support inline "#" comments on an instruction line,
# and after a "\" line-continuation the "#" text becomes part of the value —
# so keep comments on their own lines, never trailing an ENV continuation.
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Build deps needed to compile wheels (e.g. psycopg). Stay in builder only.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && rm -rf /var/lib/apt/lists/*

# Install into a venv we can copy wholesale to the runtime stage
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy ONLY requirements first → layer caching: deps reinstall only when they change
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# ---------- Stage 2: runtime ----------
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

# Runtime needs only the postgres client lib, not the whole build chain
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && rm -rf /var/lib/apt/lists/*

# Create an unprivileged user
RUN groupadd --system app && useradd --system --gid app --home /app appuser

WORKDIR /app

# Copy the prebuilt venv from the builder stage
COPY --from=builder /opt/venv /opt/venv

# Copy application code, owned by the non-root user
COPY --chown=appuser:app . .

# Collect static at build time (no DB needed for this step)
RUN python manage.py collectstatic --noinput

USER appuser

EXPOSE 8000

# gunicorn.conf.py controls workers/timeouts; see earlier section
CMD ["gunicorn", "myproject.wsgi:application", "-c", "gunicorn.conf.py"]
```

### A clean multi-stage Dockerfile — FastAPI

Nearly identical; the difference is the start command (Gunicorn + Uvicorn workers) and no `collectstatic`.

```dockerfile
# ---------- Stage 1: builder ----------
FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# ---------- Stage 2: runtime ----------
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PATH="/opt/venv/bin:$PATH"

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && rm -rf /var/lib/apt/lists/*

RUN groupadd --system app && useradd --system --gid app --home /app appuser
WORKDIR /app

COPY --from=builder /opt/venv /opt/venv
COPY --chown=appuser:app . .

USER appuser
EXPOSE 8000

# 4 Uvicorn workers managed by Gunicorn (uvicorn_worker.UvicornWorker, modern class)
CMD ["gunicorn", "main:app", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--worker-class", "uvicorn_worker.UvicornWorker"]
# Simpler alternative (no Gunicorn needed): run Uvicorn's own worker manager
# CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

> `requirements.txt` for FastAPI here would include: `fastapi`, `uvicorn[standard]`, `gunicorn`, `uvicorn-worker` (provides the modern `uvicorn_worker.UvicornWorker`; only needed for the Gunicorn variant), `pydantic-settings`, `psycopg[binary]` or `asyncpg`, `sqlalchemy`.

## Core concept: docker-compose — wiring app + Postgres + Redis

`docker-compose` describes a **multi-container** app in one YAML file and starts everything with one command. It creates a private network where services reach each other **by service name** (e.g. the host `db` resolves to the Postgres container).

```yaml
# compose.yaml   (modern filename; "docker-compose.yml" still works)
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data   # persist data across container restarts
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    build: .
    # Run migrations, THEN start the server. Migrations belong in the release step.
    command: >
      sh -c "python manage.py migrate --noinput &&
             gunicorn myproject.wsgi:application -c gunicorn.conf.py"
    environment:
      # Note hostnames "db" and "redis" — the compose service names
      DATABASE_URL: postgres://app:secret@db:5432/app
      REDIS_URL: redis://redis:6379/0
      SECRET_KEY: ${SECRET_KEY}        # pulled from your shell / .env
      DEBUG: "False"
      ALLOWED_HOSTS: "localhost,example.com"
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy     # wait until Postgres passes its healthcheck
      redis:
        condition: service_healthy

volumes:
  pgdata:
```

```bash
docker compose up --build        # build images & start everything (foreground)
docker compose up -d             # detached (background)
docker compose logs -f web       # tail the web service logs
docker compose exec web bash     # shell into the running web container
docker compose down              # stop & remove containers (keeps named volumes)
docker compose down -v           # ...and delete volumes (wipes the DB)
```

> `depends_on` only controls **start order**, not readiness — that's why we add `condition: service_healthy`. Without it, the web container may start before Postgres is accepting connections and crash on the first query.

## Core concept: running migrations on deploy

Migrations must run **once per deploy**, **before** new code serves traffic — not inside every worker (that would race). Patterns, from simple to robust:

```bash
# 1) Simplest: in the container command (fine for single-instance / compose)
sh -c "python manage.py migrate --noinput && gunicorn myproject.wsgi:application -c gunicorn.conf.py"
```

```bash
# 2) An entrypoint script — clearer separation, and waits for the DB
# entrypoint.sh
#!/usr/bin/env sh
set -e
python manage.py migrate --noinput
python manage.py collectstatic --noinput
exec "$@"      # then run whatever CMD was given (gunicorn ...)
```

```dockerfile
# In the Dockerfile:
COPY --chown=appuser:app entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["gunicorn", "myproject.wsgi:application", "-c", "gunicorn.conf.py"]
```

```bash
# 3) Best for multi-instance/PaaS: a dedicated RELEASE step that runs once
#    (Heroku release phase, ECS task, K8s Job) before rolling out new pods.
python manage.py migrate --noinput
```

> FastAPI/SQLAlchemy uses **Alembic**: the deploy equivalent is `alembic upgrade head`. Same rule — run once, before serving.

## Core concept: health checks

Orchestrators (Docker, Kubernetes, load balancers) need a cheap endpoint to ask "are you alive and ready?" If it fails, traffic stops routing to that instance and it gets restarted.

FastAPI:

```python
@app.get("/healthz")
async def healthz():
    return {"status": "ok"}   # liveness: process is up
```

Django (add a tiny view + URL, or use `django-health-check`):

```python
# views.py
from django.http import JsonResponse

def healthz(request):
    return JsonResponse({"status": "ok"})
```

```python
# urls.py
from django.urls import path
from .views import healthz
urlpatterns = [path("healthz", healthz)]
```

Wire it into Docker so the platform knows:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/healthz')" || exit 1
```

> Distinguish **liveness** ("is the process running?" — keep it dependency-free) from **readiness** ("can it serve? is the DB reachable?"). A readiness probe may actually query the DB; a liveness probe should not, or a blip in the DB will kill healthy app processes.

## Core concept: CI/CD pipeline overview

**CI** (Continuous Integration) = on every push, automatically lint + test. **CD** (Continuous Delivery/Deployment) = on merge to main, automatically build the image and ship it.

A typical pipeline:

```
push → lint (ruff) → tests (pytest) → build Docker image → push to registry → deploy → run migrations → health check
```

A minimal GitHub Actions workflow:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: secret
          POSTGRES_DB: app_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 5s --health-timeout 3s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check .
      - run: pytest
        env:
          DATABASE_URL: postgres://app:secret@localhost:5432/app_test
          SECRET_KEY: ci-test-key
```

The deploy job (only on main) would build & push the image and trigger the platform:

```yaml
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t registry.example.com/myapp:${{ github.sha }} .
      - run: docker push registry.example.com/myapp:${{ github.sha }}
      # then: kubectl set image / aws ecs update-service / flyctl deploy / etc.
```

## Core concept: the cloud deployment landscape

You'll get asked "how would you deploy this?" — know the trade-offs:

| Approach | Examples | You manage | Best for |
|---|---|---|---|
| **PaaS** (Platform) | Render, Railway, Fly.io, Heroku, Google App Engine | Just your code + a config file | Small teams, speed, "just deploy it" |
| **Containers / orchestration** | AWS ECS/Fargate, Google Cloud Run, Kubernetes (EKS/GKE) | Image + manifests; platform runs containers | Scaling, microservices, control |
| **VM / IaaS** | AWS EC2, DigitalOcean Droplet, bare metal | Everything: OS, nginx, systemd, patches | Full control, legacy, cost tuning |
| **Serverless functions** | AWS Lambda, Cloud Functions | A function (via Mangum for ASGI) | Spiky/low traffic, event-driven |

Rules of thumb:
- **Default to a PaaS or Cloud Run** unless you have a reason not to — they remove enormous ops toil.
- **Containers** give you a portable artifact that runs identically on your laptop, CI, and prod — this is the modern standard.
- **Managed everything else**: use managed Postgres (RDS, Cloud SQL) and managed Redis. Don't run stateful databases in your app containers in production.

## Common mistakes & gotchas

- **Running `runserver` / `uvicorn --reload` in production.** They are dev tools — no supervision, slow, insecure. Always Gunicorn/Uvicorn.
- **`DEBUG=True` in production.** Leaks stack traces, settings, and SQL to attackers. It must be `False` and read from the environment.
- **Empty or wildcard `ALLOWED_HOSTS`.** With `DEBUG=False`, an empty list rejects everything; `["*"]` is insecure. Set real hostnames.
- **Committing `.env` / secrets.** Git-ignore it; inject real secrets via the platform.
- **Forgetting `collectstatic`** — admin and app static files 404 in production.
- **No `.dockerignore`** — bloated images that may leak `.git`/secrets.
- **Single-stage Dockerfile** shipping compilers and `build-essential` to prod — bigger image, bigger attack surface.
- **Running as root in the container** — unnecessary risk.
- **Too many workers for the RAM** — OOM kills. Each worker is a full app copy; measure memory.
- **Running migrations inside every worker** — race conditions/duplicate work. Run once, in a release/entrypoint step.
- **`depends_on` without `condition: service_healthy`** — the app starts before the DB is ready and crashes.
- **Buffered logs** — forgetting `PYTHONUNBUFFERED=1` means logs appear late or never; in containers, log to **stdout/stderr**, not files.
- **Using plain `uvicorn` for FastAPI in prod and assuming it's multi-process** — a single `uvicorn main:app` is one process; use Gunicorn+Uvicorn workers (or `--workers`).

## Best practices

- One image, many environments: bake **nothing** environment-specific into the image; pass it all via env vars (12-factor).
- Pin versions: pin Python (`3.12-slim`), pin dependencies (`requirements.txt` with `==`, or a lockfile).
- Multi-stage builds + slim base images + non-root user, always.
- Log to **stdout/stderr**; let the platform aggregate. Set `PYTHONUNBUFFERED=1`.
- Put a reverse proxy (nginx) in front for TLS + static + buffering, or use a PaaS that does it for you.
- Use `--max-requests` with jitter to mitigate slow memory leaks.
- Health endpoints + container `HEALTHCHECK`; separate liveness from readiness.
- Migrations in a dedicated release step, run once per deploy.
- Use **managed** databases and caches in production.
- Automate everything in CI/CD: tests gate merges; deploys are push-button (or automatic) and reproducible.

## Interview Q&A

**Q: Why can't you use `python manage.py runserver` (or `uvicorn --reload`) in production?**
**A:** They're development servers: single/lightly threaded, no process supervision or auto-restart of crashed workers, inefficient static serving, and they explicitly state they aren't security-audited. Production needs a real application server (Gunicorn/Uvicorn) that manages a pool of workers.

**Q: What's the difference between WSGI and ASGI?**
**A:** WSGI is the synchronous Python web standard — one request per worker at a time — used by classic Django and Flask. ASGI is the async standard supporting `async`/`await`, concurrent requests per worker, and WebSockets — used by FastAPI and async Django. Gunicorn is a WSGI server; Uvicorn is an ASGI server.

**Q: How do Gunicorn and Uvicorn relate, and how do you run FastAPI in production?**
**A:** Gunicorn is a robust process manager but speaks WSGI; Uvicorn is a fast ASGI server. A classic FastAPI setup is **Gunicorn managing Uvicorn workers**, combining Gunicorn's supervision with Uvicorn's async performance — but note the worker class moved: install `uvicorn-worker` and use `--worker-class uvicorn_worker.UvicornWorker`, since the old `uvicorn.workers.UvicornWorker` is deprecated as of Uvicorn 0.30. Equally accepted today is running Uvicorn directly with `--workers`, which now manages its own worker processes (the old `tiangolo/uvicorn-gunicorn` image is deprecated).

**Q: How many workers should you run?**
**A:** For sync/WSGI, the Gunicorn rule of thumb is `(2 × cores) + 1`. For async/ASGI workers, roughly one per core, since each worker handles many concurrent requests itself. Always validate against available RAM — each worker is a full copy of the app — and measure under real load.

**Q: Why run nginx in front of Gunicorn?**
**A:** nginx terminates TLS, serves static files from disk far faster than Python, gzips responses, and crucially **buffers slow clients** so app workers aren't tied up by slow uploads/downloads. It also enables rate-limiting and routing multiple apps on one host.

**Q: What's the difference between a Docker image and a container?**
**A:** An image is an immutable, layered blueprint (code + runtime + deps) — like a class. A container is a running instance of that image — like an object. One image can spawn many containers.

**Q: What is a multi-stage Docker build and why use it?**
**A:** It uses an early "builder" stage with compilers/dev headers to install dependencies, then a clean "runtime" stage that copies only the finished artifacts (e.g. a venv). The final image is smaller, faster to ship, and has a smaller attack surface because no build toolchain is present.

**Q: Why does layer ordering matter in a Dockerfile?**
**A:** Docker caches each instruction's layer. If you `COPY requirements.txt` and install deps *before* copying your app code, then changing source code reuses the cached dependency layer instead of reinstalling everything — dramatically faster builds.

**Q: Where do migrations run on deploy, and why not in every worker?**
**A:** Migrations run once per deploy, before new code serves traffic — in an entrypoint script or a dedicated release step (Heroku release phase, an ECS task, a K8s Job). Running them in each worker causes races and duplicate work.

**Q: What does 12-factor say about configuration?**
**A:** Store config in the environment, not in code. The same image runs in every environment; only the injected env vars differ. Secrets never live in the repo. In Django use `django-environ`; in FastAPI use Pydantic Settings.

**Q: How do you serve static files in production?**
**A:** Run `collectstatic` to gather them into `STATIC_ROOT`, then serve via WhiteNoise (from the app, simplest for containers), nginx (fastest, from disk), or a CDN (for scale). With `DEBUG=False`, Django won't serve them itself.

**Q: Why run the container as a non-root user, and how?**
**A:** To limit blast radius — if the app is compromised, the attacker isn't root inside the container. Create a system user/group in the Dockerfile (`useradd`/`groupadd`), `chown` the app files to it, then `USER appuser` before the `CMD`.

**Q: What's the difference between liveness and readiness health checks?**
**A:** Liveness answers "is the process alive?" — keep it dependency-free so a transient DB blip doesn't kill healthy processes. Readiness answers "can it serve requests now?" and may check dependencies like the DB. Load balancers use readiness to decide routing; orchestrators use liveness to decide restarts.

## Practice exercises

1. **Run FastAPI the production way.** Given `main.py` with `app = FastAPI()`, write the exact `gunicorn` command to run it with 4 Uvicorn workers bound to all interfaces on port 8000.
2. **Worker math.** You deploy classic sync Django on an 8-core VM with 4 GB RAM, and each worker uses ~250 MB. What does the formula suggest, and would you actually run that many? Explain.
3. **Spot the Dockerfile bugs.** This Dockerfile has three production problems — name them:
   ```dockerfile
   FROM python:3.12
   COPY . .
   RUN pip install -r requirements.txt
   CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
   ```
4. **Compose readiness.** Your `web` service crashes on boot with "connection refused" to Postgres, even though `depends_on: [db]` is set. What's missing and how do you fix it?
5. **12-factor a setting.** Rewrite this hard-coded Django snippet to read from the environment with `django-environ`, defaulting `DEBUG` to `False`:
   ```python
   DEBUG = True
   SECRET_KEY = "super-secret-123"
   ```
6. **Add a health check.** Add a `/healthz` endpoint to a FastAPI app and a Docker `HEALTHCHECK` instruction that hits it.

<details><summary>Solutions</summary>

**1.**
```bash
# Modern worker class (after `pip install uvicorn-worker`):
gunicorn main:app --bind 0.0.0.0:8000 --workers 4 --worker-class uvicorn_worker.UvicornWorker
# Or, without Gunicorn at all:
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

**2.** Formula: `(2 × 8) + 1 = 17` workers. But 17 × 250 MB ≈ 4.25 GB > 4 GB RAM → you'd OOM. In practice you'd run fewer (e.g. 6–8), possibly add `--threads` for I/O-bound work, or scale horizontally with more, smaller instances. The formula is a starting point, not a law — measure memory and latency under load.

**3.** (a) Single-stage on the full `python:3.12` (not `-slim`) ships a large image with build tooling. (b) Runs as **root** (no non-root user created/switched). (c) Uses **`runserver`** — a dev server — in production; should be Gunicorn. Bonus: copies all code before installing deps (poor layer caching) and sets no `PYTHONUNBUFFERED`.

**4.** `depends_on` only controls **start order**, not readiness — the DB container starts but isn't accepting connections yet. Add a healthcheck to `db` and gate `web` on it:
```yaml
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5
  web:
    depends_on:
      db:
        condition: service_healthy
```

**5.**
```python
import environ
env = environ.Env(DEBUG=(bool, False))
environ.Env.read_env()
DEBUG = env("DEBUG")
SECRET_KEY = env("SECRET_KEY")   # raises if unset — fail loud
```

**6.**
```python
# main.py
from fastapi import FastAPI
app = FastAPI()

@app.get("/healthz")
async def healthz():
    return {"status": "ok"}
```
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/healthz')" || exit 1
```

</details>

## Cheat sheet

| Need | Command / setting |
|---|---|
| Django (sync) | `gunicorn myproject.wsgi:application --workers 3 --bind 0.0.0.0:8000` |
| FastAPI (prod) | `gunicorn main:app --workers 4 -k uvicorn_worker.UvicornWorker -b 0.0.0.0:8000` (or `uvicorn main:app --workers 4 --host 0.0.0.0 --port 8000`) |
| Async Django | `gunicorn myproject.asgi:application -k uvicorn_worker.UvicornWorker --workers 4` |
| Sync worker count | `(2 × cores) + 1` |
| Async worker count | `~= core count` |
| Collect static | `python manage.py collectstatic --noinput` |
| Migrate on deploy | `python manage.py migrate --noinput` (Alembic: `alembic upgrade head`) |
| Build / run image | `docker build -t app:1.0 .` · `docker run -p 8000:8000 app:1.0` |
| Compose up/down | `docker compose up --build -d` · `docker compose down` |
| Shell into container | `docker compose exec web bash` |

**Stack:** Internet → nginx (TLS, static, buffering) → Gunicorn/Uvicorn (workers) → Django/FastAPI.

**Protocols:** WSGI = sync (Gunicorn) · ASGI = async + WebSockets (Uvicorn / Gunicorn+UvicornWorker).

**Image vs container:** image = blueprint (class) · container = running instance (object).

**Dockerfile must-haves:** `python:3.12-slim` · multi-stage · `COPY requirements.txt` before code (caching) · non-root `USER` · `PYTHONUNBUFFERED=1` · `.dockerignore` · `HEALTHCHECK`.

**12-factor:** config in env vars · secrets never in git · log to stdout · one image, many environments.

**Deploy order:** build image → push → **migrate (once)** → roll out → health check.

**Cloud:** PaaS (Render/Fly/Railway) = easiest · Containers (Cloud Run/ECS/K8s) = portable & scalable · VM = full control · managed DB/Redis always.
