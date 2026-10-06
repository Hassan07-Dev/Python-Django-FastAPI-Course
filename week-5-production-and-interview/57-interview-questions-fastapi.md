# 57 — Interview Questions — FastAPI

> Week 5 · Production & Interview Prep

## What you will learn

- A **question bank of 36 realistic FastAPI + Pydantic interview questions** with model answers, grouped by topic.
- How to talk confidently about **FastAPI fundamentals**: ASGI, Starlette + Pydantic, type-hint-driven design, and where the performance comes from.
- How FastAPI **distinguishes path / query / body parameters** purely from your function signature.
- **Pydantic v2** in depth: validation, validators, `model_config` / `from_attributes`, `model_dump`, and the v1 → v2 differences interviewers love to probe.
- **Dependency injection**: `Depends`, `yield` dependencies, sub-dependencies, and `dependency_overrides` for testing.
- **async vs sync** endpoints and how *not* to block the event loop.
- `response_model`, status codes, error handling with `HTTPException`, and **auth** (OAuth2 password flow, JWT, `get_current_user`, scopes).
- **Middleware, CORS, background tasks, lifespan**, and integrating **SQLAlchemy 2.x** (FastAPI has no built-in ORM).
- Structuring large apps with **`APIRouter`**, and **testing** with `TestClient`.
- Several **Django/DRF vs FastAPI** comparison answers so you can frame trade-offs out loud.

## PHP/Laravel bridge

You have already interviewed for Laravel roles, so you know the rhythm: the interviewer probes *why* the framework does something, not just *that* it does. Map your instincts across — FastAPI's **`Depends`** is Laravel's service-container injection and route middleware combined; **Pydantic models** are Laravel **Form Requests + API Resources** fused into one typed object; **`response_model`** is an **API Resource** that also drives the OpenAPI docs; **`HTTPException`** is `abort(404)`; **`APIRouter`** is a route group / controller. The big mental shift to *say out loud* in an interview: FastAPI is **async-first** and **type-hint-driven**, so the framework reads your function signature the way Laravel reads route definitions plus DI plus validation rules — all at once.

---

## Core concept — How to use this lesson

This is a drill file. For each question: cover the answer, say yours out loud, then read the model answer and note what you missed. The strongest interview answers do three things — **define the term, explain the why, and give a one-line concrete example.** Aim for that shape every time.

The questions are grouped:

| # | Section |
|---|---------|
| 1 | Fundamentals & architecture (Q1–Q6) |
| 2 | Parameters: path / query / body (Q7–Q10) |
| 3 | Pydantic v2 (Q11–Q17) |
| 4 | Dependency injection (Q18–Q22) |
| 5 | async vs sync & the event loop (Q23–Q25) |
| 6 | Responses, status codes, errors (Q26–Q28) |
| 7 | Auth: OAuth2 / JWT / scopes (Q29–Q31) |
| 8 | Middleware, CORS, background, lifespan (Q32–Q33) |
| 9 | Databases & SQLAlchemy (Q34) |
| 10 | Structure & testing (Q35) |
| 11 | Django/DRF vs FastAPI (Q36) |

---

## Fundamentals & architecture

**Q: What is FastAPI, and what is it built on?**

**A:** FastAPI is a modern, high-performance Python web framework for building APIs, built on two libraries: **Starlette** (the ASGI toolkit that handles routing, requests/responses, middleware, WebSockets) and **Pydantic** (data validation and serialization via type hints). FastAPI's own contribution is the **dependency-injection system**, automatic **request validation from type hints**, and **automatic OpenAPI/Swagger docs**. In one sentence: *Starlette gives it the web plumbing, Pydantic gives it the data layer, and FastAPI glues them together around Python type hints.*

**Q: What is ASGI and how does it differ from WSGI?**

**A:** **WSGI** (Web Server Gateway Interface) is the old synchronous Python web standard — one request blocks one worker until it finishes (Flask, classic Django). **ASGI** (Asynchronous Server Gateway Interface) is its async successor: it supports `async`/`await`, long-lived connections, WebSockets, and handling many concurrent requests on a single worker via an event loop. FastAPI is ASGI-native, so you run it with an ASGI server like **Uvicorn** (or Uvicorn workers under Gunicorn), not with `gunicorn app:app` in WSGI mode.

```bash
# Run a FastAPI app (development)
uvicorn main:app --reload
# Production: Gunicorn managing Uvicorn workers
gunicorn main:app -k uvicorn.workers.UvicornWorker -w 4
```

**Q: Why is FastAPI considered "fast"? Where does the performance actually come from?**

**A:** Two sources. **(1) Concurrency:** because it's ASGI/async, a single worker can keep serving other requests while one is waiting on I/O (DB, HTTP calls), so throughput on I/O-bound workloads is high. **(2) The underlying libraries are C-accelerated:** Starlette is lean, and **Pydantic v2's core is written in Rust** (`pydantic-core`), making validation/serialization very fast. The honest caveat to mention: "fast" is about I/O-bound concurrency and validation overhead — Python is still Python for CPU-bound work, and raw per-request latency is comparable to other modern frameworks.

**Q: What does "type-hint-driven" mean in FastAPI?**

**A:** You declare types in your function signature, and FastAPI uses them to do real work at runtime: parse and **validate** incoming data, **convert** it to the right Python type, generate the **OpenAPI schema**, and decide whether a parameter comes from the path, query, or body. The type hint isn't just documentation — it's the configuration.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    # item_id is validated & converted to int; q is an optional string
    return {"item_id": item_id, "q": q}
# GET /items/abc  => 422 Unprocessable Entity (not an int)
```

**Q: What do you get "for free" with FastAPI?**

**A:** Automatic request **validation** and clear **422** errors, automatic **serialization** of responses, interactive **API docs** at `/docs` (Swagger UI) and `/redoc`, a machine-readable **OpenAPI** schema at `/openapi.json`, editor **autocompletion** and type-checking thanks to the type hints, and a built-in **dependency-injection** system. In Laravel terms, you get Form-Request validation, API Resources, and auto-generated API docs without bolting on extra packages.

**Q: When would you choose FastAPI over Django/DRF — and when not?**

**A:** Choose **FastAPI** for API-only / microservice work, async-heavy I/O (lots of outbound HTTP, streaming, WebSockets), ML-model serving, and when you want type-driven validation and auto docs with minimal boilerplate. Choose **Django/DRF** when you want **batteries included** — a mature ORM, migrations, admin, auth, and a huge ecosystem — especially for content-heavy or CRUD-heavy products where development speed and conventions matter more than raw async concurrency. The honest framing: *FastAPI is a focused API layer; Django is a full web platform.*

---

## Parameters: path / query / body

**Q: How does FastAPI decide whether a parameter is a path, query, or body parameter?**

**A:** It uses three rules: **(1)** if the name appears in the route path (`/items/{item_id}`), it's a **path** parameter; **(2)** if it's a **singular type** (int, str, bool, etc.) and *not* in the path, it's a **query** parameter; **(3)** if its type is a **Pydantic model** (or you explicitly mark it with `Body(...)`), it's read from the **request body**. Defaults decide required vs optional: no default → required, a default value → optional.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.put("/items/{item_id}")
def update_item(item_id: int, q: str | None = None, item: Item | None = None):
    # item_id -> path, q -> query (optional), item -> body (Pydantic model)
    return {"item_id": item_id, "q": q, "item": item}
```

**Q: How do you add validation/metadata to path and query parameters?**

**A:** Use `Path(...)` and `Query(...)` (modern style: wrap with `Annotated`). They let you set constraints (`ge`, `le`, `min_length`, `max_length`, `pattern`), defaults, titles, and descriptions — all of which flow into the OpenAPI docs.

```python
from typing import Annotated
from fastapi import FastAPI, Path, Query

app = FastAPI()

@app.get("/items/{item_id}")
def read(
    item_id: Annotated[int, Path(ge=1)],
    q: Annotated[str | None, Query(max_length=50)] = None,
):
    return {"item_id": item_id, "q": q}
# GET /items/0  => 422 (item_id must be >= 1)
```

**Q: How do you receive multiple values for one query parameter (e.g. `?tag=a&tag=b`)?**

**A:** Type it as a list and use `Query`:

```python
from typing import Annotated
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
def read(tags: Annotated[list[str], Query()] = []):
    return {"tags": tags}
# GET /items/?tags=a&tags=b  => {"tags": ["a", "b"]}
```

**Q: How do you accept multiple body parameters or embed a single one?**

**A:** Declare several Pydantic models and FastAPI nests them under keys named after the parameters. To force a single model to be nested under its own key, use `Body(embed=True)`.

```python
from typing import Annotated
from fastapi import FastAPI, Body
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    name: str

class Item(BaseModel):
    price: float

@app.post("/orders/")
def create(user: User, item: Item, priority: Annotated[int, Body()] = 1):
    # Expected JSON: {"user": {...}, "item": {...}, "priority": 2}
    return {"user": user, "item": item, "priority": priority}
```

---

## Pydantic v2

**Q: What is Pydantic and what role does it play in FastAPI?**

**A:** Pydantic is a data-validation and parsing library built around type hints. You define a `BaseModel` subclass with typed fields; Pydantic validates and coerces incoming data, raising a clear `ValidationError` on failure, and serializes models back to dicts/JSON. In FastAPI it powers **request body validation**, **response serialization** (via `response_model`), and **settings management** (`pydantic-settings`). Think Laravel Form Request + API Resource + typed DTO in one class.

```python
from pydantic import BaseModel, EmailStr

class UserIn(BaseModel):
    name: str
    age: int
    email: EmailStr

u = UserIn(name="Ada", age="36", email="ada@example.com")  # "36" coerced to int 36
print(u.age)  # => 36
```

**Q: What are the most important differences between Pydantic v1 and v2?**

**A:** Key ones to name:

| Concern | v1 | v2 (modern) |
|---|---|---|
| Serialize to dict | `model.dict()` | `model.model_dump()` |
| Serialize to JSON | `model.json()` | `model.model_dump_json()` |
| Parse from object | `model.from_orm(obj)` | `Model.model_validate(obj)` |
| Config | inner `class Config` | `model_config = ConfigDict(...)` |
| ORM mode | `orm_mode = True` | `from_attributes=True` |
| Field validator | `@validator` | `@field_validator` |
| Model validator | `@root_validator` | `@model_validator` |
| Core | pure Python | Rust core (`pydantic-core`), much faster |

The headline: v2 renamed the public API around `model_*` methods and `ConfigDict`, and rewrote the validation engine in Rust for big speedups. **Avoid quoting v1 APIs in an interview** — saying `.dict()` or `orm_mode` signals you're behind.

**Q: How do you write a field validator in Pydantic v2? And a cross-field (model) validator?**

**A:** Use `@field_validator` for one field and `@model_validator` for logic spanning several fields.

```python
from pydantic import BaseModel, field_validator, model_validator

class Signup(BaseModel):
    username: str
    password: str
    password_confirm: str

    @field_validator("username")
    @classmethod
    def no_spaces(cls, v: str) -> str:
        if " " in v:
            raise ValueError("username cannot contain spaces")
        return v.lower()

    @model_validator(mode="after")
    def passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError("passwords do not match")
        return self

Signup(username="Ada", password="x", password_confirm="x")  # ok, username -> "ada"
```

`mode="before"` runs on raw input before type coercion; `mode="after"` (the common case) runs on the validated model instance.

**Q: What does `from_attributes=True` do, and when do you need it?**

**A:** By default Pydantic validates from a **dict/JSON**. With `from_attributes=True` (the v2 replacement for v1's `orm_mode`), Pydantic can also build a model by reading **attributes off an arbitrary object** — like a SQLAlchemy or Django ORM row — using `Model.model_validate(orm_obj)`. You need it whenever you serialize an ORM object through a Pydantic response model.

```python
from pydantic import BaseModel, ConfigDict

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str

# Given an ORM row `row` with .id and .name attributes:
# UserOut.model_validate(row)  -> reads attributes, not dict keys
```

**Q: What's the difference between `model_dump()` and `model_dump_json()`, and what useful options do they take?**

**A:** `model_dump()` returns a **Python dict**; `model_dump_json()` returns a **JSON string** (with proper handling of types like `datetime`/`Decimal`). Useful kwargs: `exclude_unset=True` (only fields the client actually sent — perfect for PATCH), `exclude_none=True`, `exclude={...}`/`include={...}`, `by_alias=True`, and `mode="json"` on `model_dump()` to get JSON-compatible primitives in a dict.

```python
from pydantic import BaseModel

class Patch(BaseModel):
    name: str | None = None
    age: int | None = None

p = Patch(name="Ada")
print(p.model_dump(exclude_unset=True))  # => {"name": "Ada"}  (age omitted)
```

**Q: How do you set field constraints, defaults, and aliases?**

**A:** Use `Field(...)`. It supports constraints (`gt`, `ge`, `lt`, `le`, `min_length`, `max_length`, `pattern`), `default`/`default_factory`, `alias`, and `description`/`examples` for docs.

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=120)
    price: float = Field(gt=0, description="Price in USD")
    tags: list[str] = Field(default_factory=list)
    sku: str = Field(alias="SKU")

Product.model_validate({"name": "Pen", "price": 2.5, "SKU": "A1"})  # alias on input
```

Use `default_factory=list` for mutable defaults — a bare `= []` is a classic Python footgun (shared mutable default), though Pydantic actually guards against it, so explaining *why* you'd use `default_factory` shows depth.

**Q: How do you manage configuration/secrets in a FastAPI app?**

**A:** Use **`pydantic-settings`** (`BaseSettings`). It reads values from environment variables (and `.env` files), validates and types them, and gives you one typed settings object. Combine with `lru_cache` so it's parsed once and inject it via `Depends`.

```python
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")
    database_url: str
    jwt_secret: str
    debug: bool = False

@lru_cache
def get_settings() -> Settings:
    return Settings()  # reads env once
```

---

## Dependency injection

**Q: What is `Depends` and why is dependency injection central to FastAPI?**

**A:** `Depends` declares that a parameter's value should be produced by **calling another function** (the "dependency"). FastAPI resolves it per request, injects the result, and caches it within the request. It's used for shared logic: DB sessions, the current user, pagination params, settings, permission checks. Benefits: **reuse** (write `get_current_user` once), **separation of concerns** (endpoints stay thin), and **testability** (override dependencies in tests). It's the framework feature that most resembles Laravel's service container + middleware.

```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

def pagination(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

@app.get("/items/")
def list_items(page: Annotated[dict, Depends(pagination)]):
    return page  # GET /items/?skip=20&limit=5 -> {"skip": 20, "limit": 5}
```

**Q: What is a `yield` dependency and what is it for?**

**A:** A dependency that uses `yield` instead of `return` can run **setup before** `yield` and **teardown after** — like a context manager. The classic use is a database session: open it, `yield` it to the endpoint, then close it (even if the endpoint raised). Code after `yield` runs once the response is sent.

```python
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session
# assume SessionLocal is a configured sessionmaker

def get_db():
    db = SessionLocal()
    try:
        yield db          # hand the session to the endpoint
    finally:
        db.close()        # teardown — always runs

DBDep = Annotated[Session, Depends(get_db)]
```

**Q: What are sub-dependencies?**

**A:** A dependency can itself depend on other dependencies, forming a tree. FastAPI resolves the whole graph and, by default, **caches each dependency within a single request** so a shared dependency runs once even if several others need it. Example: `get_current_active_user` depends on `get_current_user`, which depends on `oauth2_scheme` and `get_db`.

```python
from typing import Annotated
from fastapi import Depends

def get_db():
    yield "db"

def get_current_user(db: Annotated[str, Depends(get_db)]):
    return {"user": "ada", "db": db}

def get_active_user(user: Annotated[dict, Depends(get_current_user)]):
    return user  # sub-dependency: get_db -> get_current_user -> get_active_user
```

You can disable caching for a specific dependency with `Depends(fn, use_cache=False)`.

**Q: How do you apply a dependency for its side effect only (e.g. an auth gate), without using its return value?**

**A:** Attach it at the path-operation or router level via the `dependencies=[...]` argument. FastAPI runs it (and raises if it errors) but doesn't inject anything into your function.

```python
from fastapi import Depends, FastAPI, Header, HTTPException

app = FastAPI()

def verify_token(x_token: str = Header()):
    if x_token != "secret":
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/admin", dependencies=[Depends(verify_token)])
def admin():
    return {"ok": True}  # only runs if verify_token passed
```

**Q: How do you override a dependency in tests?**

**A:** Use `app.dependency_overrides`, a dict mapping the original dependency function to a fake. FastAPI swaps it for the duration of the test — perfect for replacing the DB session or the current user without touching real infrastructure. Clear it afterward.

```python
from fastapi.testclient import TestClient
# from app.main import app, get_db

def fake_db():
    yield "test-session"

app.dependency_overrides[get_db] = fake_db
client = TestClient(app)
# ... run requests against the fake ...
app.dependency_overrides.clear()
```

---

## async vs sync & the event loop

**Q: When should an endpoint be `async def` vs plain `def` in FastAPI?**

**A:** Use **`async def`** when the body uses `await` on async I/O (async DB drivers, `httpx.AsyncClient`, async cache clients). Use plain **`def`** when your code is synchronous/blocking (a sync ORM call, a CPU task, a blocking library). The key: FastAPI runs `def` endpoints in a **threadpool** so they don't block the event loop, while `async def` endpoints run **on the event loop itself**. Both are fully supported — pick based on whether your I/O is async-capable.

**Q: What's the danger of calling blocking code inside an `async def` endpoint?**

**A:** It **blocks the event loop** — while that coroutine is stuck (e.g. on a synchronous `requests.get()` or `time.sleep()`), no other request on that worker can make progress, destroying concurrency. The fixes: use an async library and `await` it; OR if you must call blocking code, offload it with `await run_in_threadpool(...)` (Starlette) or `await asyncio.to_thread(...)`; OR just make the endpoint a plain `def` so FastAPI threadpools it for you.

```python
import asyncio, time
from fastapi import FastAPI

app = FastAPI()

@app.get("/bad")
async def bad():
    time.sleep(5)        # BLOCKS the whole event loop for 5s — wrong
    return {"ok": True}

@app.get("/good")
async def good():
    await asyncio.to_thread(time.sleep, 5)  # offloaded to a thread — ok
    return {"ok": True}
```

**Q: Does using `async def` make CPU-bound work faster?**

**A:** No. `async`/`await` helps with **I/O-bound concurrency** (waiting on the network/disk), not CPU-bound work — and a long CPU computation in an `async def` still blocks the event loop. For CPU-bound work, run a plain `def` endpoint (threadpool helps a little against the GIL only for some libraries), or better, offload to a **process pool** or a background **worker/queue** (e.g. Celery/RQ/Dramatiq). Saying this distinguishes you from candidates who think `async` is a magic speed-up.

---

## Responses, status codes, errors

**Q: What is `response_model` and why use it?**

**A:** `response_model` declares the Pydantic model used to **serialize, validate, and filter** the response, and to document it in OpenAPI. Its most important job is **filtering**: fields not on the response model are stripped out — so you can return a full DB object (with a `hashed_password`) but only expose safe fields. It guarantees the contract regardless of what the function returns internally. Pair it with `response_model_exclude_unset=True` for sparse responses.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class UserIn(BaseModel):
    name: str
    password: str

class UserOut(BaseModel):  # no password field
    name: str

@app.post("/users/", response_model=UserOut)
def create(user: UserIn):
    return user  # password is silently filtered out of the response
```

**Q: How do you set status codes, and how do you return a 201 on create?**

**A:** Set the default with the `status_code` argument on the decorator (use the `status` module for readability). For dynamic codes, take a `Response` parameter and set `response.status_code`, or return a `JSONResponse(status_code=...)`.

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/items/", status_code=status.HTTP_201_CREATED)
def create():
    return {"created": True}  # responds with 201
```

**Q: How do you handle errors? What is `HTTPException` and how do you customize error handling?**

**A:** Raise **`HTTPException(status_code=..., detail=...)`** to return a clean error response (it's Laravel's `abort()`). For domain-specific errors, define a custom exception and register an **exception handler** with `@app.exception_handler(...)` to convert it into a response in one place. To customize validation errors, override the handler for `RequestValidationError`.

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

class OutOfStock(Exception):
    def __init__(self, sku: str):
        self.sku = sku

@app.exception_handler(OutOfStock)
def handle_out_of_stock(request: Request, exc: OutOfStock):
    return JSONResponse(status_code=409, content={"detail": f"{exc.sku} out of stock"})

@app.get("/buy/{sku}")
def buy(sku: str):
    if sku == "NONE":
        raise HTTPException(status_code=404, detail="Item not found")
    raise OutOfStock(sku)
```

---

## Auth: OAuth2 / JWT / scopes

**Q: How does authentication typically work in FastAPI? Describe the OAuth2 password flow with JWT.**

**A:** FastAPI ships `OAuth2PasswordBearer` and `OAuth2PasswordRequestForm` from `fastapi.security`. The flow: **(1)** client POSTs username + password to a `/token` endpoint as form data; **(2)** you verify the password (hashed with `pwdlib` — bcrypt/Argon2) and return a signed **JWT access token**; **(3)** the client sends `Authorization: Bearer <token>` on later requests; **(4)** a `get_current_user` dependency extracts the token via `OAuth2PasswordBearer`, decodes/verifies the JWT, and loads the user — raising 401 if invalid. FastAPI doesn't ship a user model or session — *you* own that, which is the main difference from Django's built-in auth.

```python
from datetime import datetime, timedelta, timezone
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
import jwt  # PyJWT

SECRET, ALGO = "change-me", "HS256"
app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_token(sub: str) -> str:
    payload = {"sub": sub, "exp": datetime.now(timezone.utc) + timedelta(minutes=30)}
    return jwt.encode(payload, SECRET, algorithm=ALGO)

@app.post("/token")
def login(form: Annotated[OAuth2PasswordRequestForm, Depends()]):
    if form.username != "ada" or form.password != "secret":  # demo only
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Bad credentials")
    return {"access_token": create_token(form.username), "token_type": "bearer"}

def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]):
    try:
        payload = jwt.decode(token, SECRET, algorithms=[ALGO])
    except jwt.PyJWTError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")
    return {"username": payload["sub"]}

@app.get("/me")
def me(user: Annotated[dict, Depends(get_current_user)]):
    return user
```

**Q: How do you hash and verify passwords?**

**A:** Never store plaintext. Use a slow, salted hash — **bcrypt** or **Argon2**. Hash on signup, verify on login. FastAPI's current docs use **`pwdlib`** (passlib is effectively unmaintained and broke against recent `bcrypt` releases, so prefer `pwdlib` for new code; reach for `passlib` only to read legacy hashes).

```python
from pwdlib import PasswordHash

pwd = PasswordHash.recommended()        # Argon2 by default
hashed = pwd.hash("secret")
print(pwd.verify("secret", hashed))     # => True
print(pwd.verify("wrong", hashed))      # => False
```

If an interviewer specifically asks about bcrypt, mention you can configure it explicitly (`pwdlib` supports bcrypt; the older `passlib.context.CryptContext(schemes=["bcrypt"])` pattern still appears in many codebases but is legacy).

**Q: What are OAuth2 scopes and how does FastAPI support them?**

**A:** Scopes are named permissions attached to a token (`items:read`, `users:admin`). FastAPI supports them via `Security(...)` (a `Depends` variant that also declares required scopes) and the `SecurityScopes` object inside the dependency, which lets you check that the token actually carries the scopes the endpoint requires — raising **401 "Not enough permissions"** otherwise (FastAPI's docs use 401 with a `WWW-Authenticate` header, per the OAuth2 spec). You encode granted scopes into the JWT at login and enforce required scopes per endpoint. Crucially, the token's scopes are **decoded from the JWT** inside the dependency — they are not a request parameter.

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, Security, status
from fastapi.security import OAuth2PasswordBearer, SecurityScopes
import jwt  # PyJWT

SECRET, ALGO = "change-me", "HS256"
app = FastAPI()
# Declare the available scopes so they show up in the docs:
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token", scopes={"items:read": "Read items", "users:admin": "Admin"}
)

def get_current_user(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2_scheme)],
):
    try:
        payload = jwt.decode(token, SECRET, algorithms=[ALGO])
    except jwt.PyJWTError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")
    token_scopes = payload.get("scopes", [])  # granted scopes from the JWT
    for scope in security_scopes.scopes:       # required scopes for this endpoint
        if scope not in token_scopes:
            raise HTTPException(
                status.HTTP_401_UNAUTHORIZED,
                "Not enough permissions",
                headers={"WWW-Authenticate": f'Bearer scope="{security_scopes.scope_str}"'},
            )
    return {"username": payload["sub"], "scopes": token_scopes}

# Endpoint declares required scope via Security(...), wiring a real dependency:
@app.get("/items")
def read_items(user: Annotated[dict, Security(get_current_user, scopes=["items:read"])]):
    return ["ok"]
```

---

## Middleware, CORS, background, lifespan

**Q: How do middleware, CORS, and background tasks work in FastAPI?**

**A:** **Middleware** (Starlette's) wraps every request/response — register with `@app.middleware("http")` or `app.add_middleware(...)` for things like timing, logging, or adding headers. **CORS** is configured with the built-in `CORSMiddleware`, listing allowed origins/methods/headers — essential when a browser frontend on another domain calls your API. **Background tasks** (`BackgroundTasks`) run *after* the response is sent, in the same process — good for fire-and-forget work like sending an email; for heavy or reliable jobs use a real queue (Celery) instead.

```python
import time
from fastapi import BackgroundTasks, FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=["https://app.example.com"],
                   allow_methods=["*"], allow_headers=["*"])

@app.middleware("http")
async def add_timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = str(time.perf_counter() - start)
    return response

def write_log(msg: str):
    with open("log.txt", "a") as f:
        f.write(msg + "\n")

@app.post("/notify")
def notify(bg: BackgroundTasks):
    bg.add_task(write_log, "notification sent")  # runs after response
    return {"queued": True}
```

**Q: What is the `lifespan` and why is it preferred over `@app.on_event`?**

**A:** `lifespan` is an async context manager you pass to `FastAPI(lifespan=...)` to run **startup** code before `yield` (open a DB pool, load an ML model, warm caches) and **shutdown** code after (close connections). It replaces the deprecated `@app.on_event("startup")` / `@app.on_event("shutdown")` decorators, keeps related setup/teardown together, and lets you stash resources on `app.state`.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.pool = "db-pool-opened"   # startup
    yield
    app.state.pool = None               # shutdown / cleanup

app = FastAPI(lifespan=lifespan)
```

---

## Databases & SQLAlchemy

**Q: FastAPI has no built-in ORM — how do you talk to a database, and how do you wire it in cleanly?**

**A:** You bring your own — almost always **SQLAlchemy 2.x** (or SQLModel, which wraps SQLAlchemy + Pydantic), with **Alembic** for migrations. The clean pattern: a `sessionmaker`, a `get_db` **yield dependency** that opens a session per request and closes it in `finally`, and Pydantic models with `from_attributes=True` to serialize ORM rows. For async, use `create_async_engine` + `AsyncSession` and an `async def get_db`.

```python
from typing import Annotated
from fastapi import Depends, FastAPI
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, sessionmaker, DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

engine = create_engine("sqlite:///./app.db")
SessionLocal = sessionmaker(bind=engine, autoflush=False)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

app = FastAPI()

@app.get("/users/{user_id}")
def read_user(user_id: int, db: Annotated[Session, Depends(get_db)]):
    return db.get(User, user_id)  # 2.x style; or db.scalars(select(User))
```

The "no built-in ORM" point is a favorite interview question because it highlights FastAPI's **micro-framework** philosophy versus Django's batteries-included ORM.

---

## Structure & testing

**Q: How do you structure a large FastAPI app, and how do you test it?**

**A:** **Structure:** split routes into modules with **`APIRouter`** (one per resource/domain — `users`, `items`), each with its own `prefix` and `tags`, then `app.include_router(...)` in `main.py`. Keep schemas (Pydantic), models (SQLAlchemy), CRUD/services, and dependencies in separate packages. **Testing:** use `TestClient` (Starlette's, backed by `httpx`) to call the app **in-process** without a running server, and `app.dependency_overrides` to inject test DBs/fakes. For async endpoints/clients, use `httpx.AsyncClient` with `ASGITransport`.

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")
def list_users():
    return [{"id": 1, "name": "Ada"}]

# main.py
from fastapi import FastAPI
from routers import users

app = FastAPI()
app.include_router(users.router)

# test_users.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_list_users():
    r = client.get("/users/")
    assert r.status_code == 200
    assert r.json()[0]["name"] == "Ada"
```

---

## Django/DRF vs FastAPI

**Q: Compare FastAPI with Django/DRF across the dimensions an interviewer cares about.**

**A:** Hit these contrasts:

| Dimension | Django + DRF | FastAPI |
|---|---|---|
| Philosophy | Batteries-included full-stack | Micro-framework, API-focused |
| Concurrency | WSGI sync (ASGI possible, less mature) | ASGI **async-first** |
| ORM | **Built-in** Django ORM + migrations | None — bring SQLAlchemy + Alembic |
| Validation | DRF **Serializers** | **Pydantic** models |
| Output shaping | DRF Serializer / `fields` | `response_model` |
| Auth/admin | Built-in auth, sessions, **admin** | Roll your own (OAuth2 helpers provided) |
| API docs | via `drf-spectacular` (add-on) | **Automatic** OpenAPI / Swagger |
| Validation savings | `serializer.is_valid()` boilerplate | type hints do it |
| Best for | Content/CRUD apps, fast standard build-out | Async APIs, microservices, ML serving |

The mature one-liner: *Django/DRF gives you a whole platform with strong conventions; FastAPI gives you a lean, async, type-driven API layer where you assemble the pieces.* Mention that **DRF Serializer ≈ Pydantic model**, **DRF ViewSet/Router ≈ APIRouter**, and **Django middleware ≈ Starlette middleware** to show you can translate between them.

---

## Common mistakes & gotchas

- **Calling blocking I/O inside `async def`** (`requests`, `time.sleep`, sync ORM) — it freezes the event loop. Use async libs, `asyncio.to_thread`, or a plain `def` endpoint.
- **Thinking `async` speeds up CPU work** — it only helps I/O concurrency.
- **Quoting Pydantic v1 APIs** — `.dict()`, `.json()`, `orm_mode`, `@validator`, `class Config`. Use `model_dump()`, `model_dump_json()`, `from_attributes=True`, `@field_validator`, `ConfigDict`.
- **Forgetting `from_attributes=True`** when serializing ORM objects through a response model — you'll get errors or empty fields.
- **Leaking sensitive fields** because you didn't set a `response_model` (e.g. returning `hashed_password`).
- **Not closing DB sessions** — always use a `yield` dependency with `try/finally`.
- **Mutable default arguments** in Pydantic/functions — prefer `Field(default_factory=list)`.
- **Returning the wrong status code** — POST-create should be **201**, deletes often **204**; set `status_code` explicitly.
- **Forgetting CORS** — browser calls from another origin fail until `CORSMiddleware` is configured.
- **Using `BackgroundTasks` for heavy/critical jobs** — they die with the process and aren't retried; use a real queue.
- **Not clearing `dependency_overrides`** between tests — leaks fakes into later tests.

## Best practices

- **Type everything**; let type hints drive validation, docs, and editor support.
- **Separate input and output schemas** (`UserIn` / `UserOut`) and always set `response_model`.
- **Use `Annotated[...] + Depends/Query/Path`** — the modern, reusable style (`DBDep = Annotated[Session, Depends(get_db)]`).
- **One `get_db` yield dependency**; never open sessions ad hoc in handlers.
- **Centralize auth** in a `get_current_user` dependency; build `get_current_active_user` on top of it.
- **Configure via `pydantic-settings`** + `lru_cache`; never hardcode secrets.
- **Use `lifespan`** for startup/shutdown, not deprecated `on_event`.
- **Split routes with `APIRouter`** by domain; keep `main.py` thin.
- **Test in-process with `TestClient`** and override dependencies for the DB/auth.
- **Run with Uvicorn (Gunicorn + Uvicorn workers in prod)**; put it behind a reverse proxy.

## Interview Q&A

*(The 36 grouped questions above ARE the Q&A bank — this section adds rapid-fire "lightning round" questions interviewers use to close out.)*

**Q: In one sentence, what is FastAPI built on?** **A:** Starlette (ASGI web layer) + Pydantic (validation), glued by a dependency-injection system and driven by type hints.

**Q: How do you return a 201?** **A:** `@app.post("/x", status_code=status.HTTP_201_CREATED)`.

**Q: Pydantic v2 method to serialize a model to a dict?** **A:** `model.model_dump()` (JSON string: `model_dump_json()`).

**Q: v2 equivalent of `orm_mode`?** **A:** `model_config = ConfigDict(from_attributes=True)`.

**Q: Server you run FastAPI with?** **A:** Uvicorn (ASGI); Gunicorn + Uvicorn workers in production.

**Q: How do you replace the DB session in tests?** **A:** `app.dependency_overrides[get_db] = fake_db`.

**Q: Where do path vs query vs body parameters come from?** **A:** Path = in the route URL; singular type not in path = query; Pydantic model / `Body()` = body.

**Q: How do you run code after the response is sent?** **A:** `BackgroundTasks.add_task(...)`.

**Q: How do you avoid blocking the event loop with sync code?** **A:** `await asyncio.to_thread(...)` / `run_in_threadpool`, or make the endpoint a plain `def`.

**Q: Does FastAPI include an ORM?** **A:** No — bring SQLAlchemy (+ Alembic) or SQLModel.

## Practice exercises

Try these before peeking. Speak the answers out loud as if in an interview.

1. **Explain the request lifecycle:** trace a `POST /users` request through FastAPI — from ASGI server to validation to `get_db` dependency to response serialization. Name each component.
2. **Spot the bug:** an endpoint is `async def` and calls `requests.get("https://api...")`. What's wrong and what are *three* ways to fix it?
3. **Migrate v1 → v2:** rewrite this Pydantic v1 model to v2: `class User(BaseModel): name: str; class Config: orm_mode = True` plus a `@validator("name")` that uppercases, and show how you'd serialize it to a dict.
4. **Write a `get_current_user` dependency** (pseudo-OK) that extracts a Bearer token, decodes a JWT, and raises 401 on failure — and explain how you'd test an endpoint that uses it without a real token.
5. **Design schemas:** for a `Product` resource, design separate input and output Pydantic models so the response never leaks an internal `cost_price` field. Wire it with `response_model`.
6. **Compare:** in 4 bullet points, justify choosing FastAPI over DRF for a new async microservice that serves an ML model.

<details><summary>Solutions</summary>

**1.** Uvicorn (ASGI server) receives the request and passes the ASGI scope to Starlette → routing matches `POST /users` → FastAPI resolves dependencies (e.g. `get_db` yields a session) → the JSON body is parsed and **validated against the Pydantic input model** (422 on failure) → your handler runs → the return value is serialized through `response_model` (filtering fields) → response goes back out; the `get_db` dependency's code after `yield` runs (closes the session). Background tasks (if any) run after the response is sent.

**2.** `requests.get` is **synchronous/blocking**, so it blocks the event loop — no other request on that worker progresses until it returns. Fixes: **(a)** use an async client and await it: `async with httpx.AsyncClient() as c: await c.get(...)`; **(b)** offload: `await asyncio.to_thread(requests.get, url)`; **(c)** change the endpoint to plain `def` so FastAPI runs it in a threadpool.

**3.**
```python
from pydantic import BaseModel, ConfigDict, field_validator

class User(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # was orm_mode
    name: str

    @field_validator("name")                          # was @validator
    @classmethod
    def upper(cls, v: str) -> str:
        return v.upper()

u = User(name="ada")
print(u.model_dump())  # => {"name": "ADA"}   (was .dict())
```

**4.**
```python
from typing import Annotated
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
import jwt

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]):
    try:
        payload = jwt.decode(token, "SECRET", algorithms=["HS256"])
    except jwt.PyJWTError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")
    return {"username": payload["sub"]}
```
To test without a real token, override the dependency:
```python
app.dependency_overrides[get_current_user] = lambda: {"username": "tester"}
```
Now any protected endpoint sees a logged-in test user.

**5.**
```python
from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict

class ProductIn(BaseModel):
    name: str
    price: float
    cost_price: float          # internal, supplied on create

class ProductOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    price: float               # NO cost_price -> never leaked

app = FastAPI()

@app.post("/products/", response_model=ProductOut)
def create(p: ProductIn):
    saved = type("Row", (), {"id": 1, "name": p.name,
                             "price": p.price, "cost_price": p.cost_price})()
    return saved               # cost_price filtered out by response_model
```

**6.** (a) **Async-first** — non-blocking I/O for many concurrent inference requests; (b) **Lower latency/overhead** and Pydantic's Rust core for fast request validation; (c) **Automatic OpenAPI docs** for consumers; (d) **Lean footprint** — no Django machinery (admin, ORM, templating) you won't use for a stateless model server, plus easy integration with the Python ML ecosystem.

</details>

## Cheat sheet

```python
# --- App & run ---
from fastapi import FastAPI
app = FastAPI()
# uvicorn main:app --reload   (prod: gunicorn -k uvicorn.workers.UvicornWorker)

# --- Params ---
@app.get("/items/{id}")                      # path: in URL
def f(id: int, q: str | None = None): ...    # query: singular type, not in path
@app.post("/items/")
def g(item: ItemModel): ...                  # body: Pydantic model
# Annotated[int, Path(ge=1)] / Annotated[str, Query(max_length=50)]

# --- Pydantic v2 ---
m.model_dump()           # -> dict     (v1 .dict())
m.model_dump_json()      # -> JSON str (v1 .json())
Model.model_validate(x)  # parse dict/obj (v1 from_orm)
model_config = ConfigDict(from_attributes=True)   # v1 orm_mode
@field_validator("f")    # v1 @validator   |  @model_validator(mode="after")
Field(gt=0, default_factory=list, alias="X")
m.model_dump(exclude_unset=True)   # great for PATCH

# --- Depends ---
def get_db():
    db = SessionLocal()
    try: yield db
    finally: db.close()
db: Annotated[Session, Depends(get_db)]
@app.get("/x", dependencies=[Depends(verify)])     # side-effect only
app.dependency_overrides[get_db] = fake_db         # tests

# --- async ---
async def ep(): await something()      # async I/O on the loop
def ep(): blocking()                   # sync -> threadpooled by FastAPI
await asyncio.to_thread(blocking)      # offload blocking call

# --- Responses / errors ---
@app.post("/x", status_code=status.HTTP_201_CREATED, response_model=Out)
raise HTTPException(status_code=404, detail="Not found")
@app.exception_handler(MyError)        # custom handler

# --- Auth ---
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
form: Annotated[OAuth2PasswordRequestForm, Depends()]
jwt.encode(payload, SECRET, algorithm="HS256") / jwt.decode(...)
pwd.hash(p) / pwd.verify(p, hashed)    # pwdlib (Argon2/bcrypt); passlib is legacy

# --- Infra ---
app.add_middleware(CORSMiddleware, allow_origins=[...])
bg: BackgroundTasks; bg.add_task(fn, arg)          # after response
FastAPI(lifespan=lifespan)             # startup/shutdown (not on_event)
app.include_router(users.router)       # APIRouter(prefix=..., tags=[...])

# --- Test ---
from fastapi.testclient import TestClient
client = TestClient(app); client.get("/x").json()
```

| Remember this | Because |
|---|---|
| Starlette + Pydantic | the two pillars of FastAPI |
| ASGI / Uvicorn | async-first; how you run it |
| `model_dump()` not `.dict()` | Pydantic **v2** |
| `from_attributes=True` | serialize ORM objects |
| yield `get_db` | open/close session per request |
| don't block the loop | sync I/O → `def` endpoint or `to_thread` |
| `response_model` | filters output, drives docs |
| `dependency_overrides` | swap fakes in tests |
| No built-in ORM | bring SQLAlchemy + Alembic |
