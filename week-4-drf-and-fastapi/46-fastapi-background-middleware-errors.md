# 46 — FastAPI: Background Tasks, Middleware & Structure

> Week 4 · DRF & FastAPI

You can already build endpoints, validate input with Pydantic, and inject dependencies. This lesson is about everything that wraps *around* your endpoints: doing work after you respond, intercepting every request, handling errors gracefully, managing resources for the whole app's lifetime, and organizing all of it so the project doesn't collapse under its own weight at file 30.

## What you will learn

- Running fire-and-forget work after the response with `BackgroundTasks` — and recognizing when you've outgrown it and need Celery/a real queue.
- Writing middleware two ways: the quick `@app.middleware("http")` decorator and the reusable `BaseHTTPMiddleware` class.
- Enabling cross-origin requests correctly with `CORSMiddleware`.
- Centralized error handling with `@app.exception_handler`, `HTTPException`, and overriding `RequestValidationError`.
- The modern `lifespan` context manager for startup/shutdown (DB pools, HTTP clients) — and why `@app.on_event` is dead.
- Splitting a monolithic `main.py` into `APIRouter` modules with prefixes, tags, and per-router dependencies.
- A maintainable project layout: `routers/`, `schemas/`, `models/`, `services/`, `deps/`, plus typed settings.

## PHP/Laravel bridge

A FastAPI `@app.middleware("http")` function is Laravel's HTTP middleware (`app/Http/Middleware/*`) — code that runs before and after every request, perfect for logging, timing, or auth headers. `BackgroundTasks` is the lightweight cousin of Laravel's `dispatch()->afterResponse()`: it runs after the response is sent but **in the same process**, so for anything heavy or retryable you reach for Celery the way you'd reach for Laravel's queue workers. FastAPI's `@app.exception_handler` is Laravel's `App\Exceptions\Handler::render()` — one place to turn exceptions into JSON responses. `APIRouter` is `Route::group(['prefix' => 'api/v1'], ...)` plus controller grouping. And the `lifespan` context manager is your `AppServiceProvider::boot()` / shutdown hooks combined: open the DB pool on boot, close it on shutdown.

## Core concept — BackgroundTasks: fire-and-forget after the response

Some work doesn't need to block the client. A user signs up; you must return `201 Created` fast, but you also want to send a welcome email. Sending email might take 800ms. Don't make the user wait — respond first, send the email after.

`BackgroundTasks` is a special parameter you declare in your path operation. FastAPI injects it. You schedule callables on it; they run **after** the response is delivered, in the same event loop/threadpool.

```python
# app/main.py
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel, EmailStr

app = FastAPI()


class SignupIn(BaseModel):
    email: EmailStr
    name: str


def send_welcome_email(email: str, name: str) -> None:
    # Imagine this hits an SMTP server / email API and takes ~800ms.
    print(f"[email] Sending welcome to {name} <{email}>")  # => runs AFTER response


@app.post("/signup", status_code=201)
async def signup(payload: SignupIn, background_tasks: BackgroundTasks):
    # ... create the user in the DB here ...
    background_tasks.add_task(send_welcome_email, payload.email, payload.name)
    return {"status": "created", "email": payload.email}
    # Client gets the JSON immediately; the email goes out a moment later.
```

Key facts beginners trip on:

- **You add tasks; FastAPI runs them after the response.** The order of `print` proves it: the response is serialized and sent, *then* `send_welcome_email` runs.
- **`add_task(func, *args, **kwargs)`** — you pass the function and its arguments, you do **not** call it. `add_task(send_welcome_email, email)` not `add_task(send_welcome_email(email))`.
- **Sync vs async tasks both work.** A plain `def` task runs in the threadpool; an `async def` task is awaited on the event loop. Same rule as path operations.
- **Dependencies can inject tasks too.** A dependency can take `BackgroundTasks` and add tasks; they all run after the response. Useful for, say, audit logging from a shared dependency.

```python
from fastapi import Depends

def get_audit_logger(background_tasks: BackgroundTasks):
    def log(event: str) -> None:
        background_tasks.add_task(lambda: print(f"[audit] {event}"))
    return log


@app.delete("/items/{item_id}")
async def delete_item(item_id: int, audit=Depends(get_audit_logger)):
    # ... delete ...
    audit(f"item {item_id} deleted")
    return {"deleted": item_id}
```

### When BackgroundTasks is the wrong tool — use Celery instead

`BackgroundTasks` runs **in your web process**. That gives it three hard limits:

| Concern | `BackgroundTasks` | Celery / RQ / Dramatiq |
|---|---|---|
| Where it runs | Same process as the web server | Separate worker process(es)/machines |
| Survives a deploy/crash? | **No** — task is lost if the process dies | Yes — task sits in a broker (Redis/RabbitMQ) |
| Retries on failure | None built in | Configurable retries, backoff |
| Scheduling (run in 1h) | No | Yes (ETA, countdown, beat/cron) |
| Heavy CPU work | Blocks/steals from request handling | Isolated, scale workers independently |
| Visibility/monitoring | None | Flower, result backend, status |

Rule of thumb: **`BackgroundTasks` is for cheap, best-effort, fast work that you don't mind losing** — sending one email, invalidating a cache key, firing an analytics event. The moment you need *durability* (must not be lost), *retries*, *scheduling*, or *heavy/long* work (video transcode, report generation), move to Celery. Conceptually it's identical to choosing Laravel's `afterResponse()` vs. dispatching a job onto a queue worker.

A Celery sketch (not runnable without a broker, shown for orientation):

```python
# app/tasks.py
from celery import Celery

celery_app = Celery("worker", broker="redis://localhost:6379/0",
                    backend="redis://localhost:6379/1")


@celery_app.task(bind=True, max_retries=3)
def generate_report(self, user_id: int):
    try:
        ...  # slow, can fail
    except Exception as exc:
        raise self.retry(exc=exc, countdown=10)


# In a path operation:
@app.post("/reports")
async def make_report(user_id: int):
    generate_report.delay(user_id)   # enqueued; returns instantly
    return {"status": "queued"}
```

## Core concept — Middleware: code around every request

Middleware wraps the entire request/response cycle. It runs *before* your endpoint (you can inspect/modify the request) and *after* (you can inspect/modify the response). FastAPI inherits Starlette's middleware system.

### The quick way: `@app.middleware("http")`

```python
import time
from fastapi import FastAPI, Request

app = FastAPI()


@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)        # <-- runs the rest of the app
    elapsed = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{elapsed:.4f}"
    return response
```

The mental model:

- `request` is the incoming request; you can read headers, path, etc.
- `call_next(request)` calls the **next** thing in the chain (more middleware, then your route). It returns the `Response`. You **must** `await` it and **must** `return` something.
- Code before `await call_next` runs on the way *in*; code after runs on the way *out*.

This is exactly Laravel middleware's `$next($request)` sandwich.

### The reusable way: `BaseHTTPMiddleware`

For middleware you want to configure, reuse across projects, or unit-test, write a class:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp
from fastapi import Request


class RequestIDMiddleware(BaseHTTPMiddleware):
    def __init__(self, app: ASGIApp, header_name: str = "X-Request-ID"):
        super().__init__(app)
        self.header_name = header_name

    async def dispatch(self, request: Request, call_next):
        import uuid
        rid = request.headers.get(self.header_name) or str(uuid.uuid4())
        request.state.request_id = rid          # stash it for handlers to read
        response = await call_next(request)
        response.headers[self.header_name] = rid
        return response


app.add_middleware(RequestIDMiddleware, header_name="X-Request-ID")
```

`request.state` is a scratchpad that lives for the duration of one request — handy for passing a request ID, the authenticated user, or a DB session down to your handlers.

> **Order matters.** Middleware is applied as an onion: the **last** one added is the **outermost** (runs first on the way in, last on the way out). When you have CORS, auth, and logging, think about which must wrap which. CORS usually wants to be near the outside so it still adds headers to error responses.

> **A caveat on `BaseHTTPMiddleware`:** it can interfere with streaming responses and `contextvars` in edge cases. For most apps it's fine; for high-performance streaming or advanced context propagation, a raw ASGI middleware is the escape hatch. Don't reach for that until you actually hit the problem.

### CORS — the one middleware you'll add in almost every API

Browsers block cross-origin requests unless the server opts in with CORS headers. If your React app on `http://localhost:5173` calls your API on `http://localhost:8000`, you need `CORSMiddleware`.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "https://app.example.com"],
    allow_credentials=True,     # allow cookies / Authorization on cross-origin
    allow_methods=["*"],        # or ["GET", "POST", "PUT", "DELETE"]
    allow_headers=["*"],
)
```

Gotchas that bite everyone:

- **`allow_origins=["*"]` + `allow_credentials=True` is invalid** and silently won't work — the browser rejects a wildcard origin when credentials are allowed. List explicit origins if you need credentials.
- Origins are **scheme + host + port** with no trailing slash: `http://localhost:5173`, not `localhost:5173` and not `http://localhost:5173/`.
- CORS is enforced by the **browser**, not your server. `curl` and Postman ignore it — "works in Postman, fails in the browser" is almost always CORS.

## Core concept — Custom exception handling

By default, FastAPI turns `HTTPException` into a JSON error and validation failures into a `422`. You'll want to customize both: consistent error shape, hiding internals, logging.

### Raising `HTTPException`

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

_DB = {1: "Ada", 2: "Linus"}


@app.get("/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in _DB:
        raise HTTPException(
            status_code=404,
            detail="User not found",
            headers={"X-Error": "user-missing"},   # optional response headers
        )
    return {"id": user_id, "name": _DB[user_id]}
# GET /users/99 => 422? No: 404 {"detail": "User not found"}
```

### A custom exception + handler

Define a domain exception and register a handler so business code stays clean (`raise UserNotFound(id)`) while the HTTP translation lives in one place.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()


class UserNotFound(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id


@app.exception_handler(UserNotFound)
async def user_not_found_handler(request: Request, exc: UserNotFound):
    return JSONResponse(
        status_code=404,
        content={
            "error": "user_not_found",
            "message": f"User {exc.user_id} does not exist",
            "request_id": getattr(request.state, "request_id", None),
        },
    )


@app.get("/users/{user_id}")
async def get_user(user_id: int):
    if user_id != 1:
        raise UserNotFound(user_id)
    return {"id": 1, "name": "Ada"}
# GET /users/7 => 404 {"error":"user_not_found","message":"User 7 does not exist", ...}
```

### Overriding validation errors (`RequestValidationError`)

The built-in `422` body is verbose. To reshape it, override the handler for `RequestValidationError`. Import it from `fastapi.exceptions`.

```python
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()


@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    fields = [
        {"field": ".".join(str(p) for p in e["loc"][1:]), "issue": e["msg"]}
        for e in exc.errors()
    ]
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={"error": "validation_failed", "fields": fields},
    )


class ItemIn(BaseModel):
    name: str
    price: float


@app.post("/items")
async def create_item(item: ItemIn):
    return item
# POST /items {"name": "x"} =>
# 422 {"error":"validation_failed","fields":[{"field":"price","issue":"Field required"}]}
```

You can also override the generic `HTTPException` handler (`from fastapi.exceptions import HTTPException` / `from starlette.exceptions import HTTPException`) to give *all* HTTP errors your standard shape. And a catch-all `@app.exception_handler(Exception)` lets you log unexpected errors and return a sanitized `500` instead of leaking a stack trace — just remember to re-raise or log so you don't hide real bugs.

## Core concept — Lifespan: startup and shutdown done right

You often need resources that live for the **whole app**, not per-request: a database connection pool, an HTTP client, a cache connection, an ML model loaded into memory. Open them once at startup, close them at shutdown.

The old way was `@app.on_event("startup")` / `@app.on_event("shutdown")`. **Those are deprecated.** The modern, recommended approach is a single `lifespan` async context manager. Everything before `yield` runs at startup; everything after runs at shutdown.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request


# Pretend these are real clients (asyncpg pool, httpx.AsyncClient, etc.)
class DBPool:
    async def connect(self): print("[db] pool opened")
    async def close(self): print("[db] pool closed")


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ----- startup -----
    pool = DBPool()
    await pool.connect()
    app.state.db_pool = pool          # share it via app.state
    yield                             # <-- app runs here, serving requests
    # ----- shutdown -----
    await pool.close()


app = FastAPI(lifespan=lifespan)


@app.get("/ping")
async def ping(request: Request):
    pool = request.app.state.db_pool   # reach the shared resource
    return {"pool": "ready", "obj": id(pool)}
```

Why this design is better:

- **One scope, one place.** Setup and its matching teardown sit next to each other; you can't forget to close what you opened in the same `with`.
- **Exception-safe.** Code after `yield` runs even if startup raised (handled like a normal context manager), so resources get cleaned up.
- **You can hold state on the stack** (the `pool` local) instead of globals.

Access the resource in endpoints via `request.app.state.db_pool`, or — cleaner — wrap it in a dependency:

```python
from fastapi import Depends, Request

def get_db_pool(request: Request) -> DBPool:
    return request.app.state.db_pool

@app.get("/health")
async def health(pool: DBPool = Depends(get_db_pool)):
    return {"db": "up", "obj": id(pool)}
```

## Core concept — Structuring a real app with APIRouter

A single `main.py` is fine for a demo and a nightmare at scale. `APIRouter` lets you split endpoints into modules that you then `include` into the app — exactly like Laravel splitting routes into `routes/api.php` groups bound to controllers.

### A router module

```python
# app/routers/users.py
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(
    prefix="/users",                 # every path here starts with /users
    tags=["users"],                  # groups them in the /docs UI
    dependencies=[Depends(lambda: None)],   # runs for EVERY route in this router
    responses={404: {"description": "Not found"}},
)


@router.get("")                       # final path: GET /users
async def list_users():
    return [{"id": 1, "name": "Ada"}]


@router.get("/{user_id}")             # final path: GET /users/{user_id}
async def get_user(user_id: int):
    if user_id != 1:
        raise HTTPException(404, "User not found")
    return {"id": user_id, "name": "Ada"}
```

`prefix`, `tags`, `dependencies`, and `responses` set on the router apply to every route in it. **Per-router dependencies** are the killer feature: put `Depends(require_admin)` on an admin router and every endpoint inside is protected — no repetition. (Use `dependencies=[...]` when you need the side effect, like an auth check, but don't need the return value in the handler.)

### Wiring routers into the app

```python
# app/main.py
from fastapi import FastAPI
from app.routers import users, items

app = FastAPI()

app.include_router(users.router)
app.include_router(items.router)

# You can override/extend at include time, e.g. add a version prefix:
# app.include_router(users.router, prefix="/api/v1")
# (this stacks on top of the router's own prefix)
```

### A sensible project layout

```text
app/
├── main.py            # create FastAPI(), lifespan, middleware, include_router()
├── config.py          # Settings (pydantic-settings)
├── deps.py            # shared dependencies (get_db, get_current_user)
├── routers/           # one module per resource — HTTP layer only
│   ├── __init__.py
│   ├── users.py
│   └── items.py
├── schemas/           # Pydantic models = request/response shapes (the "API contract")
│   ├── user.py
│   └── item.py
├── models/            # ORM models (SQLAlchemy tables) — the DB layer
│   ├── user.py
│   └── item.py
├── services/          # business logic — pure-ish functions, no FastAPI imports
│   ├── user_service.py
│   └── item_service.py
└── tests/
```

The discipline that keeps this maintainable:

- **Routers are thin.** They parse input, call a service, shape the response. No business logic, no raw SQL.
- **Services hold logic** and ideally don't import FastAPI at all — that makes them unit-testable and reusable (think: a Laravel Action/Service class, not a controller).
- **`schemas/` (Pydantic) ≠ `models/` (ORM).** Schemas are the *external* contract; models are the *internal* DB representation. Keeping them separate stops a DB column rename from silently changing your API.

## Core concept — Settings management with pydantic-settings

Hardcoding config is the road to leaked secrets. Use `pydantic-settings` (a separate package: `pip install pydantic-settings`) to load typed, validated config from environment variables / a `.env` file.

```python
# app/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    app_name: str = "My API"
    database_url: str                      # required: errors at startup if missing
    debug: bool = False
    cors_origins: list[str] = []           # in env, set as JSON: CORS_ORIGINS=["http://localhost:5173"]


@lru_cache
def get_settings() -> Settings:
    return Settings()                      # cached: read/parsed once per process
```

```python
# usage anywhere, including as a dependency
from fastapi import Depends
from app.config import Settings, get_settings


@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {"app": settings.app_name, "debug": settings.debug}
```

Why `BaseSettings` over `os.environ.get`: it **validates and coerces types** (`DEBUG=1` becomes a real `bool`), fails fast at startup if a required var is missing, gives you autocomplete, and reads `.env` automatically. The `@lru_cache` makes `get_settings()` a singleton you can still override in tests via `app.dependency_overrides`.

## Tying it together

A realistic `main.py` that uses everything above:

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

from app.config import get_settings
from app.routers import users, items


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    app.state.db_pool = await open_pool(settings.database_url)   # your code
    yield
    await app.state.db_pool.close()


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(title=settings.app_name, lifespan=lifespan)

    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    @app.exception_handler(RequestValidationError)
    async def _validation(request: Request, exc: RequestValidationError):
        return JSONResponse(status_code=422,
                            content={"error": "validation_failed",
                                     "fields": exc.errors()})

    app.include_router(users.router, prefix="/api/v1")
    app.include_router(items.router, prefix="/api/v1")
    return app


app = create_app()
```

The `create_app()` factory pattern (build the app inside a function) makes testing easy: each test can build a fresh app with overridden settings/dependencies.

## Common mistakes & gotchas

- **Calling the task instead of passing it:** `add_task(send_email(to))` runs it *now*. Use `add_task(send_email, to)`.
- **Relying on BackgroundTasks for critical work.** If the process restarts mid-task, it's gone. No retries, no durability. Use Celery for anything that must not be lost.
- **Forgetting to `await call_next` / not returning a response** in middleware → hangs or `None` response errors.
- **`allow_origins=["*"]` with `allow_credentials=True`** — browser silently blocks it. Use explicit origins with credentials.
- **Trailing slashes / missing scheme in CORS origins.** `http://localhost:5173`, never `localhost:5173/`.
- **Still using `@app.on_event("startup")`.** Deprecated — use `lifespan`.
- **Putting business logic in routers.** Routers should be thin; logic belongs in `services/`.
- **Mixing Pydantic schemas and ORM models into one class.** Keep `schemas/` and `models/` separate.
- **Catch-all `Exception` handler that swallows errors silently.** Always log; otherwise real bugs vanish into a generic 500.
- **Middleware order surprises.** Last-added is outermost. If CORS headers are missing on error responses, your CORS middleware is too far inside.
- **Reading `request.app.state.x` before lifespan set it.** It only exists after startup; don't touch it at import time.

## Best practices

- Reserve `BackgroundTasks` for cheap, best-effort work; graduate to Celery for durable/retryable/heavy/scheduled jobs.
- Prefer `BaseHTTPMiddleware` (a class) over the decorator when the middleware is configurable or reused; keep middleware fast — it runs on every request.
- Define **domain exceptions** + handlers so business code raises meaningful errors and HTTP mapping lives in one place; return a **consistent error envelope** everywhere.
- Use `lifespan` for all startup/shutdown; share resources via `app.state` and expose them through dependencies, not globals.
- Split by resource with `APIRouter`; use `prefix`, `tags`, and `dependencies=[...]` for cross-cutting concerns like auth.
- Keep routers thin, services logic-only, schemas separate from models.
- Centralize config in `pydantic-settings`; never hardcode secrets; cache with `@lru_cache` and override in tests via `dependency_overrides`.
- Use a `create_app()` factory for testability.

## Interview Q&A

**Q: What are `BackgroundTasks` and when would you *not* use them?**
**A:** They schedule callables to run after the response is sent, in the same web process. They're great for cheap, fire-and-forget work like sending one email or firing an analytics event. You shouldn't use them when the work must be durable (survive a crash/deploy), needs retries, needs scheduling, or is CPU-heavy/long-running — that's a job for Celery or another task queue backed by a broker.

**Q: How does `BackgroundTasks` differ from Celery?**
**A:** `BackgroundTasks` runs in-process with no persistence, no retries, no scheduling — if the process dies, the task is lost. Celery runs tasks in separate worker processes backed by a broker (Redis/RabbitMQ), giving durability, retries with backoff, scheduling, monitoring, and independent scaling. It's the FastAPI analog of Laravel's `afterResponse()` vs. queued jobs.

**Q: Write request-timing middleware. What must `call_next` do?**
**A:** `@app.middleware("http")` on an `async def f(request, call_next)`. You record a start time, `response = await call_next(request)` (you must `await` it — it runs the rest of the chain and returns the `Response`), then add a header and `return response`. Forgetting to await or to return breaks the request.

**Q: When would you use `BaseHTTPMiddleware` instead of the decorator?**
**A:** When the middleware needs configuration (constructor args), is reused across projects, or you want to unit-test it as a class. The decorator is fine for one-off, app-specific middleware. Both ultimately do the same `dispatch(request, call_next)` dance.

**Q: A request works in Postman but fails from the browser with a CORS error. Why?**
**A:** CORS is enforced by the browser, not the server; Postman/curl ignore it. The server isn't returning the right `Access-Control-Allow-*` headers for that origin. Fix it by adding `CORSMiddleware` with the frontend's exact origin (scheme+host+port, no trailing slash). And note `allow_origins=["*"]` won't work together with `allow_credentials=True`.

**Q: How do you customize the error response for validation failures?**
**A:** Register `@app.exception_handler(RequestValidationError)` (imported from `fastapi.exceptions`). In it, read `exc.errors()` and return a `JSONResponse` with your own shape. This overrides FastAPI's default verbose 422 body app-wide.

**Q: How do you handle a custom domain exception?**
**A:** Define a normal `Exception` subclass (e.g. `UserNotFound`), then register `@app.exception_handler(UserNotFound)` returning a `JSONResponse`. Business code just raises `UserNotFound(id)`; the handler maps it to HTTP. This keeps services framework-agnostic and centralizes the HTTP translation.

**Q: Why is `lifespan` preferred over `@app.on_event`?**
**A:** `on_event("startup"/"shutdown")` is deprecated. `lifespan` is a single async context manager: setup before `yield`, teardown after, so paired resources stay together and cleanup is guaranteed even on errors. You attach shared resources (DB pool, HTTP client) to `app.state` and reach them via dependencies.

**Q: What's the difference between `schemas/` and `models/`?**
**A:** `schemas/` holds Pydantic models — the external API contract for request/response validation and serialization. `models/` holds ORM (SQLAlchemy) models — the internal database tables. Keeping them separate means a DB change doesn't accidentally alter your public API, and vice versa.

**Q: How do per-router dependencies work and why use them?**
**A:** `APIRouter(dependencies=[Depends(require_admin)])` runs that dependency for *every* route in the router. You use `dependencies=[...]` (vs. a parameter) when you need the side effect — like an auth/permission check — but not the return value. It removes repetition and makes "the whole admin section is protected" obvious.

**Q: How do you manage configuration in a FastAPI app?**
**A:** With `pydantic-settings` `BaseSettings`: declare typed fields, load from env/`.env`, get validation and type coercion, and fail fast at startup if a required var is missing. Wrap it in `@lru_cache`'d `get_settings()` so it's a singleton, inject it as a dependency, and override it in tests via `app.dependency_overrides`.

**Q: In what order does middleware execute?**
**A:** It's an onion. The last middleware added is the outermost — it runs first on the way in and last on the way out. So if you add logging then CORS, CORS wraps logging. CORS is usually added last/outermost so it can still attach headers to error responses.

## Practice exercises

1. **Fire-and-forget log.** Add a `POST /orders` endpoint that returns `201` immediately and uses `BackgroundTasks` to "write" an audit line (just `print`) after responding. Verify by ordering of output that the response goes first.

2. **Timing middleware.** Add middleware that sets `X-Process-Time` on every response. Then add a second middleware that attaches a random `X-Request-ID`, and confirm both headers appear. Note which one is outermost.

3. **CORS for a SPA.** Configure `CORSMiddleware` to allow `http://localhost:5173` with credentials. Explain in a comment why `["*"]` would break here.

4. **Custom error envelope.** Define a `ResourceNotFound` exception and a handler that returns `{"error": "...", "message": "...", "request_id": ...}` with status 404. Wire `request_id` from a middleware that stashes it on `request.state`.

5. **Validation override.** Override `RequestValidationError` to return `{"error": "validation_failed", "fields": [{"field": ..., "issue": ...}]}`.

6. **Refactor to routers + lifespan.** Take a single-file app and split it into `routers/users.py` with a `/users` prefix and `users` tag, include it under `/api/v1`, and add a `lifespan` that opens/closes a fake DB pool on `app.state`.

<details>
<summary>Solutions</summary>

```python
# Exercise 1 — BackgroundTasks
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def audit(line: str) -> None:
    print(f"[audit] {line}")   # prints AFTER the response is sent

@app.post("/orders", status_code=201)
async def create_order(background_tasks: BackgroundTasks):
    background_tasks.add_task(audit, "order created")
    return {"status": "created"}
```

```python
# Exercise 2 — two middlewares
import time, uuid
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def timing(request: Request, call_next):
    start = time.perf_counter()
    resp = await call_next(request)
    resp.headers["X-Process-Time"] = f"{time.perf_counter() - start:.4f}"
    return resp

@app.middleware("http")
async def request_id(request: Request, call_next):
    rid = str(uuid.uuid4())
    request.state.request_id = rid
    resp = await call_next(request)
    resp.headers["X-Request-ID"] = rid
    return resp
# request_id was added last => it is OUTERMOST (runs first in, last out).
```

```python
# Exercise 3 — CORS for a SPA
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # exact scheme+host+port
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
# "*" would break: the browser refuses a wildcard origin when
# allow_credentials=True, so cookies/Authorization wouldn't be sent.
```

```python
# Exercise 4 — custom error envelope + request_id from middleware
import uuid
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

class ResourceNotFound(Exception):
    def __init__(self, what: str):
        self.what = what

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request.state.request_id = str(uuid.uuid4())
    return await call_next(request)

@app.exception_handler(ResourceNotFound)
async def not_found(request: Request, exc: ResourceNotFound):
    return JSONResponse(
        status_code=404,
        content={
            "error": "not_found",
            "message": f"{exc.what} not found",
            "request_id": getattr(request.state, "request_id", None),
        },
    )

@app.get("/widgets/{wid}")
async def get_widget(wid: int):
    raise ResourceNotFound(f"widget {wid}")
```

```python
# Exercise 5 — validation override
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation(request: Request, exc: RequestValidationError):
    fields = [
        {"field": ".".join(str(p) for p in e["loc"][1:]), "issue": e["msg"]}
        for e in exc.errors()
    ]
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={"error": "validation_failed", "fields": fields},
    )

class Item(BaseModel):
    name: str
    price: float

@app.post("/items")
async def create(item: Item):
    return item
```

```python
# Exercise 6 — routers + lifespan
# app/routers/users.py
from fastapi import APIRouter
router = APIRouter(prefix="/users", tags=["users"])

@router.get("")
async def list_users():
    return [{"id": 1, "name": "Ada"}]

# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.routers import users

class FakePool:
    async def connect(self): print("[db] open")
    async def close(self): print("[db] close")

@asynccontextmanager
async def lifespan(app: FastAPI):
    pool = FakePool()
    await pool.connect()
    app.state.db_pool = pool
    yield
    await pool.close()

app = FastAPI(lifespan=lifespan)
app.include_router(users.router, prefix="/api/v1")  # final: /api/v1/users
```

</details>

## Cheat sheet

```python
# --- BackgroundTasks ---
async def ep(bg: BackgroundTasks):
    bg.add_task(func, arg1, kw=val)   # pass func, DON'T call it
# use Celery instead when: durable / retries / scheduled / heavy

# --- Middleware (decorator) ---
@app.middleware("http")
async def m(request, call_next):
    resp = await call_next(request)   # must await + return
    return resp

# --- Middleware (class) ---
class M(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next): ...
app.add_middleware(M, opt="x")        # last added = outermost

# --- CORS ---
app.add_middleware(CORSMiddleware, allow_origins=[...],
    allow_credentials=True, allow_methods=["*"], allow_headers=["*"])
# "*" + credentials = broken; origins = scheme+host+port, no trailing /

# --- Errors ---
raise HTTPException(404, "msg")
@app.exception_handler(MyError)       # custom domain exception
@app.exception_handler(RequestValidationError)  # override 422

# --- Lifespan (replaces on_event) ---
@asynccontextmanager
async def lifespan(app):
    app.state.x = setup(); yield; teardown()
app = FastAPI(lifespan=lifespan)

# --- Routers ---
router = APIRouter(prefix="/users", tags=["users"],
                   dependencies=[Depends(auth)])
app.include_router(router, prefix="/api/v1")

# --- Settings ---
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")
    database_url: str
@lru_cache
def get_settings(): return Settings()
```

| Need | Reach for |
|---|---|
| Work after response, cheap, OK to lose | `BackgroundTasks` |
| Durable / retryable / scheduled / heavy | Celery |
| Code around every request | middleware |
| Allow browser cross-origin | `CORSMiddleware` |
| Map exception → JSON | `@app.exception_handler` |
| Startup/shutdown resources | `lifespan` |
| Split endpoints into modules | `APIRouter` |
| Typed config from env/.env | `pydantic-settings` |

Layout: `routers/` (thin HTTP) · `services/` (logic) · `schemas/` (Pydantic contract) · `models/` (ORM) · `deps.py` · `config.py` · `main.py`.
