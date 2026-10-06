# 43 — FastAPI: Dependency Injection

> Week 4 · DRF & FastAPI

## What you will learn

- What `Depends()` is and why dependency injection (DI) matters for reuse, testability, and separation of concerns.
- That a "dependency" is just **any callable that returns a value** — a function, a class, even a lambda.
- Injecting shared logic: pagination params, common query/path params, settings.
- **Sub-dependencies** — dependencies that depend on other dependencies — and how FastAPI **caches** them per request.
- **Classes as dependencies** (and the `Annotated` shortcut).
- Dependencies with `yield` for setup/teardown (the classic DB session lifecycle) and how they handle exceptions.
- **Three levels of scope**: path-operation, router, and global (app-level) dependencies.
- Side-effect-only dependencies that return nothing (auth/permission guards).
- `dependency_overrides` for clean, fast tests.
- How all of this compares to Laravel's service container and automatic resolution.

---

## PHP/Laravel bridge

In Laravel you rarely write `new SomeService(...)` by hand — you typehint a class in a controller method or constructor and the **service container** auto-resolves it, building the whole dependency graph for you. You bind things in a service provider, you swap implementations in tests with `$this->app->instance(...)` or `$this->mock(...)`, and you protect routes with middleware. FastAPI's `Depends()` plays all of these roles at once: it is the resolver (like the container), the place you declare "I need this thing" (like a controller typehint), and the guard layer (like middleware). The big difference: in FastAPI you are usually explicit — you *call* `Depends(get_db)` rather than relying on pure type-based auto-wiring — which makes the dependency graph visible right in the function signature.

---

## Core concept: what `Depends()` actually does

A FastAPI **dependency** is any callable that returns a value. You declare that a path operation (or another dependency) *depends on* it, and FastAPI will:

1. Call the dependency for you when a request comes in.
2. Pass its return value into your function as a parameter.
3. Cache the result for the rest of that single request.

The simplest possible example:

```python
from fastapi import FastAPI, Depends
from typing import Annotated

app = FastAPI()

def get_greeting() -> str:
    return "Hello"

@app.get("/")
def root(greeting: Annotated[str, Depends(get_greeting)]) -> dict:
    return {"message": f"{greeting}, world"}

# GET /  =>  {"message": "Hello, world"}
```

Read `Annotated[str, Depends(get_greeting)]` as: "`greeting` is a `str`, and its value comes from calling `get_greeting`." The `str` is the type for editors/validation; the `Depends(...)` is the *source*.

> **Why `Annotated`?** The modern (and recommended) style puts the dependency inside `Annotated[...]` instead of as a default value (`greeting: str = Depends(get_greeting)`). The old default-value style still works, but `Annotated` is reusable, plays nicely with type checkers, and lets the same parameter declaration be shared across many endpoints. Use `Annotated` for everything in this lesson.

### Why bother? (reuse, testability, separation)

- **Reuse**: write the "parse pagination params" logic once, inject it into 30 endpoints.
- **Testability**: in tests you can *override* a dependency (e.g. swap the real DB for a test DB, or fake the "current user") without touching the endpoint code.
- **Separation of concerns**: the endpoint says *what* it needs ("a DB session", "the current user"), not *how* to build it. Construction logic lives in one place.

---

## Core concept: injecting shared query/path logic

A dependency can declare its own parameters — query params, path params, headers, even other dependencies. FastAPI parses them from the request exactly as if they were on your endpoint. This is the canonical "shared pagination params" pattern:

```python
from fastapi import FastAPI, Depends
from typing import Annotated

app = FastAPI()

def pagination_params(
    skip: int = 0,
    limit: int = 20,
) -> dict:
    # Clamp limit so a client can't ask for a million rows.
    limit = min(limit, 100)
    return {"skip": skip, "limit": limit}

# Define the reusable annotation ONCE...
PaginationDep = Annotated[dict, Depends(pagination_params)]

@app.get("/users")
def list_users(page: PaginationDep) -> dict:
    return {"endpoint": "users", **page}

@app.get("/orders")
def list_orders(page: PaginationDep) -> dict:
    return {"endpoint": "orders", **page}

# GET /users?skip=10&limit=5  => {"endpoint": "users", "skip": 10, "limit": 5}
# GET /orders?limit=9999      => {"endpoint": "orders", "skip": 0, "limit": 100}  (clamped)
```

Notice `skip` and `limit` automatically became **query parameters** on both `/users` and `/orders`, and they show up in the auto-generated OpenAPI docs. You wrote the parsing/validation once. Defining `PaginationDep = Annotated[...]` and reusing it is the idiomatic way to avoid repetition.

### Injecting settings

A common dependency is "give me the app configuration." Combined with `lru_cache`, you build the settings object once and reuse it:

```python
from functools import lru_cache
from fastapi import FastAPI, Depends
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Annotated

class Settings(BaseSettings):
    app_name: str = "Demo API"
    admin_email: str = "admin@example.com"
    model_config = SettingsConfigDict(env_file=".env")

@lru_cache  # build Settings once for the whole process
def get_settings() -> Settings:
    return Settings()

app = FastAPI()
SettingsDep = Annotated[Settings, Depends(get_settings)]

@app.get("/info")
def info(settings: SettingsDep) -> dict:
    return {"app_name": settings.app_name, "admin": settings.admin_email}
```

`@lru_cache` means `get_settings()` runs once and caches its result across *all* requests — perfect for read-only config. (Contrast this with FastAPI's per-request caching, covered next, which is per request.)

---

## Core concept: sub-dependencies and per-request caching

A dependency can itself depend on other dependencies. FastAPI resolves the whole tree. Crucially, **within a single request**, if the same dependency is needed in two places, FastAPI calls it **once** and reuses the value (caching).

```python
from fastapi import FastAPI, Depends, Header
from typing import Annotated

app = FastAPI()

call_count = {"db": 0}

def query_extractor(q: str | None = None) -> str | None:
    return q

def query_or_default(
    q: Annotated[str | None, Depends(query_extractor)],
    last_query: Annotated[str | None, Header()] = None,
) -> str:
    # Sub-dependency: this depends on query_extractor.
    return q or last_query or "default"

@app.get("/search")
def search(
    final_q: Annotated[str, Depends(query_or_default)],
    raw_q: Annotated[str | None, Depends(query_extractor)],  # used AGAIN
) -> dict:
    return {"final": final_q, "raw": raw_q}

# GET /search?q=hello
# => {"final": "hello", "raw": "hello"}
# query_extractor is needed by both query_or_default and the endpoint,
# but FastAPI calls it only ONCE for this request.
```

### Turning caching off

If you genuinely need the dependency to run again (e.g. it produces a fresh value each call), pass `use_cache=False`:

```python
from datetime import datetime, timezone
from fastapi import FastAPI, Depends
from typing import Annotated

app = FastAPI()

def now() -> str:
    # datetime.utcnow() is deprecated since Python 3.12; use a timezone-aware UTC value.
    return datetime.now(timezone.utc).isoformat()

@app.get("/times")
def times(
    a: Annotated[str, Depends(now)],
    b: Annotated[str, Depends(now, use_cache=False)],
) -> dict:
    # `a` and `b` may differ because `b` opts out of caching.
    return {"a": a, "b": b}
```

> **Why cache by default?** It is both a correctness and a performance feature. If three things in a request all need "the current user," you want them to agree on *one* user object and you don't want to hit the DB three times. Caching is keyed on the callable + its arguments, scoped to the single request.

---

## Core concept: classes as dependencies

Any callable works — and a **class is a callable** (calling it runs `__init__` and returns an instance). FastAPI inspects `__init__`'s parameters and treats them as request inputs. This is great when a dependency needs to hold state or expose methods.

```python
from fastapi import FastAPI, Depends
from typing import Annotated

app = FastAPI()

class Paginator:
    def __init__(self, skip: int = 0, limit: int = 20) -> None:
        self.skip = skip
        self.limit = min(limit, 100)

    def slice(self, items: list) -> list:
        return items[self.skip : self.skip + self.limit]

# When the dependency callable IS the type, you don't repeat it:
PaginatorDep = Annotated[Paginator, Depends(Paginator)]
# Shortcut: Depends() with no arg uses the annotated type as the callable:
#   Annotated[Paginator, Depends()]   <-- also valid and equivalent

@app.get("/items")
def list_items(p: PaginatorDep) -> dict:
    data = list(range(1000))
    return {"page": p.slice(data), "skip": p.skip, "limit": p.limit}

# GET /items?skip=5&limit=3  => {"page": [5, 6, 7], "skip": 5, "limit": 3}
```

Because `p` is typed as `Paginator`, your editor gives you autocomplete on `p.slice(...)`, `p.skip`, etc. This is the closest analog to Laravel injecting a fully-built service object into your controller.

---

## Core concept: dependencies with `yield` (setup / teardown)

This is the most important pattern for real apps: a dependency that does setup, **yields** a value, and then runs teardown afterwards — even if the endpoint raised an exception. The textbook case is a database session.

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker, DeclarativeBase, Mapped, mapped_column
from typing import Annotated

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

engine = create_engine("sqlite:///./app.db")
SessionLocal = sessionmaker(bind=engine, autoflush=False)
Base.metadata.create_all(engine)

app = FastAPI()

def get_db():
    db = SessionLocal()            # setup: open a session
    try:
        yield db                   # hand the session to the endpoint
        db.commit()                # commit if the request succeeded
    except Exception:
        db.rollback()              # roll back on any error
        raise
    finally:
        db.close()                 # teardown: ALWAYS close

DbDep = Annotated[Session, Depends(get_db)]

@app.get("/users/{user_id}")
def get_user(user_id: int, db: DbDep) -> dict:
    user = db.get(User, user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return {"id": user.id, "name": user.name}
```

### How `yield` dependencies behave

- Code **before** `yield` runs before the endpoint (setup).
- The yielded value is what gets injected.
- Code **after** `yield` runs **after** the response is sent (teardown). Because of this, the `finally` block is guaranteed to run, so the session always closes.
- If the endpoint raises, the exception propagates back *through* the `yield`, so a `try/except` around the `yield` (as above) can react to it (here: rollback). You then `raise` to let FastAPI turn it into the proper HTTP response.

> **Gotcha — `HTTPException` and `yield`:** if you `raise HTTPException(...)` inside the endpoint, your `except Exception` in `get_db` will catch it (run rollback) and re-raise it; FastAPI still converts it to the right 4xx response. That is the behavior you want. Do **not** swallow it.

A simpler `yield` shape when you only need cleanup (no commit/rollback logic):

```python
def get_resource():
    resource = acquire()
    try:
        yield resource
    finally:
        resource.release()
```

This is exactly Python's context-manager idea (`with`), repackaged as a dependency. If you already have a context manager, you can drive it from a `yield` dependency cleanly.

---

## Core concept: dependencies that return nothing (side-effect guards)

A dependency does not have to return a useful value. It can run a **check** and raise if something is wrong. This is how you do authentication/authorization — the FastAPI equivalent of route middleware.

```python
from fastapi import FastAPI, Depends, Header, HTTPException, status
from typing import Annotated

app = FastAPI()

def verify_api_key(x_api_key: Annotated[str | None, Header()] = None) -> None:
    if x_api_key != "s3cr3t":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or missing API key",
        )
    # returns None — we only care about the side effect (the check)

# Attach it WITHOUT injecting a value, using the `dependencies=[...]` argument:
@app.get("/protected", dependencies=[Depends(verify_api_key)])
def protected() -> dict:
    return {"ok": True}

# GET /protected (no header)               => 401
# GET /protected (X-API-Key: s3cr3t)       => {"ok": True}
```

Using `dependencies=[Depends(...)]` on the decorator (rather than a function parameter) is the right tool when you want the dependency to *run* but you don't need its return value in the handler. It documents the requirement and shows up in OpenAPI's security/parameters too.

### Returning the current user (a value guard)

More often, the auth dependency both checks *and* returns the resolved user, so the handler can use it:

```python
from fastapi import FastAPI, Depends, Header, HTTPException, status
from pydantic import BaseModel
from typing import Annotated

app = FastAPI()

class CurrentUser(BaseModel):
    id: int
    name: str
    is_admin: bool = False

def get_current_user(
    x_token: Annotated[str | None, Header()] = None,
) -> CurrentUser:
    if x_token == "admin-token":
        return CurrentUser(id=1, name="Ada", is_admin=True)
    if x_token == "user-token":
        return CurrentUser(id=2, name="Bob")
    raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Bad token")

CurrentUserDep = Annotated[CurrentUser, Depends(get_current_user)]

# A SUB-DEPENDENCY that builds on get_current_user:
def require_admin(user: CurrentUserDep) -> CurrentUser:
    if not user.is_admin:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Admins only")
    return user

AdminDep = Annotated[CurrentUser, Depends(require_admin)]

@app.get("/me")
def me(user: CurrentUserDep) -> CurrentUser:
    return user

@app.delete("/users/{user_id}")
def delete_user(user_id: int, admin: AdminDep) -> dict:
    return {"deleted": user_id, "by": admin.name}

# DELETE /users/9 (X-Token: user-token)  => 403
# DELETE /users/9 (X-Token: admin-token) => {"deleted": 9, "by": "Ada"}
```

`require_admin` is a sub-dependency: it depends on `get_current_user`. Thanks to caching, even if both run in one request, the user is resolved once.

---

## Core concept: scope — path, router, and global dependencies

You can attach dependencies at three levels. They run from outermost to innermost.

| Level | Where you declare it | Applies to |
|-------|----------------------|------------|
| Path operation | `@app.get("/x", dependencies=[...])` or a function param | One endpoint |
| Router | `APIRouter(dependencies=[...])` | Every route on that router |
| Global / app | `FastAPI(dependencies=[...])` | Every route in the whole app |

```python
from fastapi import FastAPI, APIRouter, Depends, Header, HTTPException
from typing import Annotated

def log_request(user_agent: Annotated[str | None, Header()] = None) -> None:
    print(f"[global] UA={user_agent}")

def require_admin_area(x_role: Annotated[str | None, Header()] = None) -> None:
    if x_role != "admin":
        raise HTTPException(403, "Admin area")

# GLOBAL: runs for every request in the app
app = FastAPI(dependencies=[Depends(log_request)])

# ROUTER: runs for every route on this router
admin = APIRouter(prefix="/admin", dependencies=[Depends(require_admin_area)])

@admin.get("/dashboard")
def dashboard() -> dict:
    return {"area": "admin dashboard"}

@app.get("/public")
def public() -> dict:
    return {"area": "public"}

app.include_router(admin)

# GET /public               => logs UA, returns public
# GET /admin/dashboard       => logs UA, requires X-Role: admin, else 403
```

Use **global** dependencies for cross-cutting concerns (request logging, rate-limit hooks), **router** dependencies to protect a whole feature area (e.g. everything under `/admin`), and **path** dependencies for one-off needs. This mirrors Laravel's global middleware, route-group middleware, and per-route middleware.

---

## Core concept: `dependency_overrides` for testing

This is the payoff of DI. In tests you replace a real dependency with a fake by adding to `app.dependency_overrides`. The endpoint code is untouched.

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.testclient import TestClient
from typing import Annotated

app = FastAPI()

def get_current_user() -> dict:
    # Real version would read a token / hit the DB.
    raise HTTPException(401, "not logged in")

CurrentUserDep = Annotated[dict, Depends(get_current_user)]

@app.get("/me")
def me(user: CurrentUserDep) -> dict:
    return user

# ----- test -----
def fake_user() -> dict:
    return {"id": 42, "name": "Test User"}

def test_me_returns_current_user():
    app.dependency_overrides[get_current_user] = fake_user
    client = TestClient(app)
    resp = client.get("/me")
    assert resp.status_code == 200
    assert resp.json() == {"id": 42, "name": "Test User"}
    app.dependency_overrides.clear()   # always clean up!

test_me_returns_current_user()
print("passed")  # => passed
```

The dictionary key is the **original** dependency callable; the value is the replacement callable. The replacement can also declare its own parameters/sub-dependencies. The canonical use is swapping `get_db` for a SQLite-in-memory test session so tests are fast and isolated. Always `clear()` overrides (or use a pytest fixture with teardown) so they don't leak between tests.

A pytest-flavored version using a fixture:

```python
import pytest
from fastapi.testclient import TestClient

@pytest.fixture
def client():
    app.dependency_overrides[get_current_user] = fake_user
    yield TestClient(app)
    app.dependency_overrides.clear()  # teardown after each test

def test_me(client):
    assert client.get("/me").json()["id"] == 42
```

---

## Comparison to Laravel's service container

| Concept | Laravel | FastAPI |
|---------|---------|---------|
| Resolve & build dependencies | Service container auto-resolves typehints | `Depends(callable)` resolves and calls |
| Declare "I need X" | Typehint in constructor/method | Parameter `Annotated[X, Depends(...)]` |
| Bind / configure construction | `bind` / `singleton` in a service provider | The dependency function itself; `@lru_cache` for singletons |
| Per-request reuse | Container resolves per request; `singleton` for app-wide | Cached per request by default; `@lru_cache` for app-wide |
| Route protection | Middleware (`->middleware('auth')`) | `dependencies=[Depends(guard)]` at path/router/app level |
| Setup/teardown around request | Middleware `before`/`after`, terminable middleware | `yield` dependency (code before/after `yield`) |
| Swap in tests | `$this->mock()`, `app()->instance()` | `app.dependency_overrides[real] = fake` |

The mental shift: Laravel leans on **implicit** type-based auto-wiring; FastAPI is **explicit** — you can see exactly what each endpoint pulls in by reading its signature. Explicitness is a feature here; the dependency graph is never hidden.

---

## Common mistakes & gotchas

- **Calling the dependency yourself.** It is `Depends(get_db)`, not `Depends(get_db())`. You pass the *callable*, FastAPI calls it. `Depends(get_db())` passes a value, not a callable, and breaks.
- **Forgetting `finally` in `yield` deps.** If you only `db.close()` after `yield` without a `try/finally`, an exception in the endpoint can skip cleanup and leak connections.
- **Expecting teardown to alter the response.** Code after `yield` runs *after* the response is sent. You cannot change the response body there, and raising `HTTPException` after `yield` is too late to become a normal error response.
- **Relying on caching when you need freshness.** If a dependency must produce a new value each time within one request, you must pass `use_cache=False`; otherwise you get the same cached value.
- **Leaking overrides between tests.** Forgetting `app.dependency_overrides.clear()` (or a fixture teardown) makes one test's fake bleed into the next. Flaky tests follow.
- **Putting a value-returning dep in `dependencies=[...]`.** The list form ignores return values. If you need the value in your handler, inject it as a parameter instead.
- **Mismatched async/sync.** Both `def` and `async def` dependencies work, but a blocking call (sync DB driver) inside an `async def` dependency blocks the event loop. Use a sync `def` dependency (FastAPI runs it in a threadpool) or a truly async driver.
- **Heavy work in `__init__` of a class dependency.** It runs every request (unless cached). Keep construction cheap; do the heavy lifting in methods or a cached/app-scoped dependency.

---

## Best practices

- Prefer the `Annotated[Type, Depends(...)]` style and define **reusable type aliases** (`DbDep = Annotated[Session, Depends(get_db)]`).
- Use `yield` dependencies for anything needing cleanup (DB sessions, file handles, locks). Wrap with `try/finally`.
- Put **cross-cutting** concerns at the right scope: app-level for logging, router-level for "this whole feature requires auth," path-level for one-offs.
- Keep dependencies **small and single-purpose**, then compose them via sub-dependencies (`require_admin` builds on `get_current_user`).
- Make dependencies **easy to override**: depend on the callable directly so tests can swap it. Avoid hiding it behind import-time globals.
- Use `@lru_cache` for genuinely process-wide singletons (settings); rely on FastAPI's per-request cache for per-request reuse.
- Return rich, typed objects (Pydantic models, ORM instances) from value-returning deps so handlers get autocomplete and validation.

---

## Interview Q&A

**Q: What is a dependency in FastAPI, and how is one declared?**
**A:** A dependency is any callable (function, class, or other callable) that returns a value. You declare it on a parameter with `Annotated[ReturnType, Depends(callable)]`. FastAPI calls it on each request, injects the return value, and caches it for that request.

**Q: Why use dependency injection instead of just calling helper functions inside the handler?**
**A:** Three reasons: reuse (write shared logic like pagination/auth once), testability (override dependencies in tests without touching handlers), and separation of concerns (handlers declare *what* they need; construction lives elsewhere). You also get the parsed parameters in OpenAPI docs automatically.

**Q: What does the per-request caching of dependencies mean, and when would you disable it?**
**A:** If the same dependency is needed multiple times in one request, FastAPI calls it once and reuses the value (keyed on callable + args). This avoids redundant work like re-querying the current user. Disable it with `Depends(dep, use_cache=False)` when you need a fresh value each call.

**Q: Explain a `yield` dependency and its execution order.**
**A:** Code before `yield` is setup, the yielded value is injected, and code after `yield` is teardown that runs after the response is sent. Wrapping in `try/finally` guarantees cleanup. If the endpoint raises, the exception propagates through the `yield`, so an `except` block can react (e.g. `db.rollback()`) before re-raising.

**Q: How do you protect an endpoint with auth using dependencies?**
**A:** Write a dependency that validates credentials and either raises `HTTPException(401/403)` or returns the resolved user. If you don't need the user object, attach it via `dependencies=[Depends(guard)]` on the decorator (or router/app). If you need the user, inject it as a parameter.

**Q: What is the difference between putting a dependency in `dependencies=[...]` versus as a function parameter?**
**A:** A function parameter injects the dependency's return value into the handler. The `dependencies=[...]` list runs the dependency for its side effects (checks) and ignores the return value. Use the list form for guards; use parameters when you need the value.

**Q: What are the three scopes for dependencies?**
**A:** Path-operation (one endpoint), router (every route on an `APIRouter` via its `dependencies=[...]`), and global/app (`FastAPI(dependencies=[...])`, every route). They run outermost to innermost, mirroring Laravel's global, group, and route middleware.

**Q: How are classes used as dependencies?**
**A:** A class is callable, so `Depends(MyClass)` builds an instance per request; FastAPI reads `__init__`'s parameters as request inputs. Because the parameter is typed as the class, you get autocomplete on its attributes/methods. `Annotated[MyClass, Depends()]` (empty `Depends`) uses the annotated type as the callable.

**Q: How do you replace a dependency in tests?**
**A:** `app.dependency_overrides[real_dependency] = fake_dependency`. The key is the original callable, the value the replacement. The replacement may have its own sub-dependencies. Always clear overrides afterward (or use a fixture with teardown) to avoid leakage. The classic use is swapping `get_db` for an in-memory test session.

**Q: A sync dependency vs an async dependency — does it matter?**
**A:** Both are supported. FastAPI runs sync `def` dependencies in a threadpool so they don't block the event loop. An `async def` dependency runs on the loop; doing blocking I/O inside it stalls the whole server, so use an async driver there or make it a sync `def`.

**Q: How does FastAPI DI compare to Laravel's service container?**
**A:** Both build and inject dependencies and support swapping in tests. Laravel auto-wires implicitly from typehints and binds in service providers; FastAPI is explicit — you pass the callable to `Depends(...)`, so the dependency graph is visible in the signature. Laravel middleware ≈ FastAPI guard dependencies; Laravel `singleton` ≈ `@lru_cache`.

**Q: What happens if `get_db`'s teardown raises after the response?**
**A:** Since teardown runs after the response has been sent, it cannot change the response. An exception there is logged/handled by the server but cannot turn into a normal HTTP error for that request. So keep teardown defensive (e.g. close inside `finally`).

---

## Practice exercises

1. **Pagination dependency.** Write a `pagination_params(skip=0, limit=20)` dependency that clamps `limit` to a max of 50, expose it as a reusable `Annotated` alias, and use it in two endpoints (`/posts` and `/comments`) that echo the params back.

2. **Class dependency with a method.** Create a `Filter` class dependency taking optional `q: str | None` and `active: bool = True` query params, with a `.matches(item: dict) -> bool` method. Use it in `/items` to filter a hardcoded list.

3. **DB-session `yield` dependency.** Using SQLAlchemy 2.x and SQLite, write a `get_db` dependency that opens a session, commits on success, rolls back on exception, and always closes. Add a `POST /users` that inserts a row and a `GET /users/{id}` that returns it (404 if missing).

4. **Auth guard + admin sub-dependency.** Build `get_current_user` (reads `X-Token` header) returning a Pydantic `User` with `is_admin`, and `require_admin` that depends on it and raises 403 for non-admins. Protect a `DELETE /posts/{id}` with `require_admin`.

5. **Router-level guard.** Put all admin endpoints under an `APIRouter(prefix="/admin", dependencies=[Depends(require_admin)])` so the guard applies to the whole group. Verify a non-admin token gets 403 on every admin route.

6. **Override in a test.** Write a pytest test for exercise 4 that overrides `get_current_user` with a fake admin user via `app.dependency_overrides`, asserts the delete succeeds, and clears the override in teardown.

<details><summary>Solutions</summary>

```python
# --- Exercise 1: pagination dependency ---
from fastapi import FastAPI, Depends
from typing import Annotated

app = FastAPI()

def pagination_params(skip: int = 0, limit: int = 20) -> dict:
    return {"skip": skip, "limit": min(limit, 50)}

PageDep = Annotated[dict, Depends(pagination_params)]

@app.get("/posts")
def posts(page: PageDep) -> dict:
    return {"resource": "posts", **page}

@app.get("/comments")
def comments(page: PageDep) -> dict:
    return {"resource": "comments", **page}
# GET /posts?limit=999 => {"resource": "posts", "skip": 0, "limit": 50}
```

```python
# --- Exercise 2: class dependency with a method ---
from fastapi import FastAPI, Depends
from typing import Annotated

app = FastAPI()

class Filter:
    def __init__(self, q: str | None = None, active: bool = True) -> None:
        self.q = (q or "").lower()
        self.active = active

    def matches(self, item: dict) -> bool:
        if self.active and not item.get("active", False):
            return False
        if self.q and self.q not in item["name"].lower():
            return False
        return True

FilterDep = Annotated[Filter, Depends(Filter)]
DATA = [
    {"name": "Apple", "active": True},
    {"name": "Banana", "active": False},
    {"name": "Apricot", "active": True},
]

@app.get("/items")
def items(f: FilterDep) -> list[dict]:
    return [i for i in DATA if f.matches(i)]
# GET /items?q=ap => [{"name": "Apple", ...}, {"name": "Apricot", ...}]
```

```python
# --- Exercise 3: DB-session yield dependency ---
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import create_engine
from sqlalchemy.orm import (
    Session, sessionmaker, DeclarativeBase, Mapped, mapped_column,
)
from pydantic import BaseModel
from typing import Annotated

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

engine = create_engine("sqlite:///./ex3.db")
SessionLocal = sessionmaker(bind=engine, autoflush=False)
Base.metadata.create_all(engine)

app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()

DbDep = Annotated[Session, Depends(get_db)]

class UserIn(BaseModel):
    name: str

@app.post("/users")
def create_user(payload: UserIn, db: DbDep) -> dict:
    user = User(name=payload.name)
    db.add(user)
    db.flush()  # populate user.id before commit
    return {"id": user.id, "name": user.name}

@app.get("/users/{user_id}")
def read_user(user_id: int, db: DbDep) -> dict:
    user = db.get(User, user_id)
    if user is None:
        raise HTTPException(404, "User not found")
    return {"id": user.id, "name": user.name}
```

```python
# --- Exercise 4 & 5: auth guard, admin sub-dependency, router guard ---
from fastapi import FastAPI, APIRouter, Depends, Header, HTTPException, status
from pydantic import BaseModel
from typing import Annotated

app = FastAPI()

class User(BaseModel):
    id: int
    name: str
    is_admin: bool = False

def get_current_user(x_token: Annotated[str | None, Header()] = None) -> User:
    if x_token == "admin-token":
        return User(id=1, name="Ada", is_admin=True)
    if x_token == "user-token":
        return User(id=2, name="Bob")
    raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Bad token")

def require_admin(
    user: Annotated[User, Depends(get_current_user)],
) -> User:
    if not user.is_admin:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Admins only")
    return user

# Exercise 4: single endpoint protected by the admin sub-dependency
@app.delete("/posts/{post_id}")
def delete_post(
    post_id: int, admin: Annotated[User, Depends(require_admin)],
) -> dict:
    return {"deleted": post_id, "by": admin.name}

# Exercise 5: whole admin group guarded at the router level
admin_router = APIRouter(
    prefix="/admin", dependencies=[Depends(require_admin)],
)

@admin_router.get("/stats")
def stats() -> dict:
    return {"stats": "ok"}

@admin_router.get("/settings")
def settings() -> dict:
    return {"settings": "ok"}

app.include_router(admin_router)
# GET /admin/stats with X-Token: user-token => 403
# GET /admin/stats with X-Token: admin-token => {"stats": "ok"}
```

```python
# --- Exercise 6: override get_current_user in a test ---
import pytest
from fastapi.testclient import TestClient
# assumes `app`, `get_current_user`, `User` from exercise 4 above

def fake_admin() -> User:
    return User(id=99, name="TestAdmin", is_admin=True)

@pytest.fixture
def client():
    app.dependency_overrides[get_current_user] = fake_admin
    yield TestClient(app)
    app.dependency_overrides.clear()

def test_admin_can_delete(client):
    resp = client.delete("/posts/7")
    assert resp.status_code == 200
    assert resp.json() == {"deleted": 7, "by": "TestAdmin"}
```

</details>

---

## Cheat sheet

```python
from fastapi import FastAPI, APIRouter, Depends, Header, HTTPException
from typing import Annotated

# 1. A dependency is any callable returning a value
def dep(q: int = 0) -> int: return q

# 2. Inject it (modern Annotated style)
def handler(x: Annotated[int, Depends(dep)]): ...

# 3. Reusable alias
Dep = Annotated[int, Depends(dep)]

# 4. Class as dependency  (Annotated[Cls, Depends()] also works)
class C:
    def __init__(self, a: int = 0): self.a = a
CDep = Annotated[C, Depends(C)]

# 5. yield dependency (setup / teardown)
def get_db():
    db = open_session()
    try:
        yield db          # injected value
        db.commit()
    except Exception:
        db.rollback(); raise
    finally:
        db.close()        # always runs after response

# 6. Side-effect guard (no value used)
@app.get("/p", dependencies=[Depends(guard)])
def p(): ...

# 7. Disable per-request caching
Annotated[str, Depends(dep, use_cache=False)]

# 8. Scopes
FastAPI(dependencies=[Depends(g)])                  # global / app
APIRouter(dependencies=[Depends(g)])               # router
@app.get("/x", dependencies=[Depends(g)])          # path operation

# 9. Override in tests
app.dependency_overrides[real_dep] = fake_dep
app.dependency_overrides.clear()
```

| Need | Tool |
|------|------|
| Inject a value | parameter `Annotated[T, Depends(fn)]` |
| Run a check only | `dependencies=[Depends(fn)]` |
| Setup + teardown | `yield` dependency with `try/finally` |
| Process-wide singleton | `@lru_cache` on the dependency |
| Per-request reuse | default caching (free) |
| Force re-run | `use_cache=False` |
| Protect a feature area | router `dependencies=[...]` |
| Protect everything | app `dependencies=[...]` |
| Swap in tests | `app.dependency_overrides[...]` |

**One-liners to remember:** pass the *callable* (`Depends(fn)`, not `Depends(fn())`) · cached per request by default · `yield` teardown runs after the response · key overrides on the *original* callable · explicit graph beats Laravel's implicit auto-wiring.
