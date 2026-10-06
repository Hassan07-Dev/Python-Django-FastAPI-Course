# 44 — FastAPI: Async & Databases (SQLAlchemy)

> Week 4 · DRF & FastAPI

## What you will learn

- Why FastAPI ships with **no ORM**, and why SQLAlchemy 2.x is the default pairing
- The difference between **sync** and **async** SQLAlchemy engines and sessions
- How to declare models with the modern **2.0 declarative style** (`Mapped`, `mapped_column`)
- How to build an **engine** and a **session factory** once, and reuse them
- How to inject a **DB session per request** with a `yield` dependency
- The core **CRUD** patterns: `add` / `commit` / `refresh`, `select()`, `scalars`, `get`
- **Relationships** in a nutshell (one-to-many with `relationship()`)
- **Alembic** for migrations: `init`, `autogenerate`, `upgrade`
- When `async def` + async session actually helps, and when sync is fine
- The **SQLModel** alternative (Pydantic + SQLAlchemy in one class)
- How to **avoid blocking the event loop**
- A side-by-side comparison to the **Django ORM** workflow you saw earlier

## PHP/Laravel bridge

In Laravel, the framework hands you Eloquent — an ORM, migrations (`php artisan make:migration`), and a query builder, all bundled and wired to your `.env` database config. FastAPI is the opposite philosophy: it is a thin, fast HTTP layer and deliberately ships **no database layer at all**. You bring your own. The community-standard choice is **SQLAlchemy** for the ORM/Core and **Alembic** (SQLAlchemy's sibling project) for migrations — think "Eloquent + Doctrine migrations, but as separate libraries you assemble yourself." The big new idea with no direct Eloquent analog is the **async** path: a `User::find(1)` in Laravel is always blocking, whereas SQLAlchemy gives you a choice between a classic blocking session and an `await session.get(User, 1)` that yields control to the event loop. We will cover both and tell you which to reach for.

## Core concept: why no built-in ORM, and the SQLAlchemy mental model

FastAPI's job is request parsing, validation (via Pydantic), dependency injection, and serialization. Persistence is out of scope on purpose — you might use SQLAlchemy, Tortoise ORM, raw `asyncpg`, MongoDB with Beanie, or nothing. SQLAlchemy is the most common choice because it is mature, supports both sync and async, and works with every major SQL database.

SQLAlchemy has two layers:

- **Core** — a SQL expression language (`select()`, `insert()`, table metadata). Lower level.
- **ORM** — maps Python classes to tables, gives you a `Session` (unit of work), identity map, relationships, and lazy/eager loading.

In 2.x, both layers share one unified API. You write `select(User)` whether you are using Core or ORM. That is a big simplification over the 1.x era — ignore any old tutorial that uses `session.query(User)` (the legacy 1.x query API; still works but discouraged).

```bash
# Install the pieces. asyncpg is the async Postgres driver; psycopg the sync one.
pip install "sqlalchemy>=2.0" "fastapi" "uvicorn[standard]"
pip install asyncpg          # async Postgres driver
pip install "psycopg[binary]"  # sync Postgres driver (psycopg 3)
pip install alembic          # migrations
# SQLite needs no extra driver for sync; for async install aiosqlite:
pip install aiosqlite
```

## Core concept: declaring models the 2.0 way (`Mapped`, `mapped_column`)

The modern declarative style uses Python type hints to define columns. `Mapped[int]` tells SQLAlchemy the Python/SQL type; `mapped_column(...)` carries the column options (primary key, nullable, defaults). This is far more type-checker-friendly than the old `Column(Integer)` style — your IDE knows `user.id` is an `int`.

```python
# models.py
from datetime import datetime
from sqlalchemy import String, ForeignKey, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    """All models inherit from this. Holds the shared MetaData."""
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    # Mapped[str] => NOT NULL. Mapped[str | None] => nullable.
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(100))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    # One user has many posts. back_populates wires the two sides together.
    posts: Mapped[list["Post"]] = relationship(back_populates="author")


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    body: Mapped[str]                      # str with no length => TEXT-ish
    published: Mapped[bool] = mapped_column(default=False)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    author: Mapped["User"] = relationship(back_populates="posts")
```

Key reading rules:

| Annotation | Meaning |
|---|---|
| `Mapped[int]` | NOT NULL column of int type |
| `Mapped[str \| None]` | nullable column |
| `mapped_column(primary_key=True)` | PK (auto-increment for int) |
| `mapped_column(default=...)` | Python-side default (set on insert by SQLAlchemy) |
| `mapped_column(server_default=func.now())` | DB-side default (in the `CREATE TABLE` DDL) |
| `relationship(back_populates=...)` | navigation to related rows, not a real column |

`default` vs `server_default` is a common interview trap: `default` is applied by SQLAlchemy in Python before the INSERT; `server_default` is rendered into the schema and applied by the database itself.

## Core concept: the engine and the session factory

The **engine** is the connection pool + dialect — create exactly one, at startup, and share it. The **session factory** is a callable that mints `Session` objects bound to that engine. A `Session` is your unit of work: it tracks new/dirty/deleted objects and flushes them on `commit()`.

### Sync version

```python
# db.py  (sync)
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

# echo=True logs every SQL statement — great while learning, off in prod.
engine = create_engine(
    "postgresql+psycopg://user:pass@localhost/mydb",
    echo=True,
    pool_pre_ping=True,   # checks a connection is alive before using it
)

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)
```

### Async version

```python
# db.py  (async)
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    async_sessionmaker,
    AsyncSession,
)

# Note the +asyncpg driver in the URL — this MUST be an async driver.
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    echo=True,
    pool_pre_ping=True,
)

AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    expire_on_commit=False,   # see "gotchas" — keeps attrs usable after commit
    class_=AsyncSession,
)
```

The URL prefix decides everything: `postgresql+psycopg://` is sync, `postgresql+asyncpg://` is async. For SQLite it is `sqlite:///./app.db` (sync) vs `sqlite+aiosqlite:///./app.db` (async). You cannot use an async driver with `create_engine` or a sync driver with `create_async_engine`.

## Core concept: a DB session per request via a `yield` dependency

This is the idiomatic FastAPI pattern. A dependency that `yield`s gives you setup-before / teardown-after semantics — like a context manager scoped to one request. FastAPI runs the code up to `yield`, injects the value, runs your endpoint, then runs the code after `yield` (even if the endpoint raised).

### Sync dependency

```python
# deps.py  (sync)
from collections.abc import Generator
from sqlalchemy.orm import Session
from db import SessionLocal


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Async dependency

```python
# deps.py  (async)
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from db import AsyncSessionLocal


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
    # async with closes (and rolls back if not committed) automatically.
```

You then declare `db: Session = Depends(get_db)` (or `AsyncSession`) in each endpoint. One session = one request = one transaction boundary. This mirrors Laravel's per-request DB connection handled by the framework — except here you wire it yourself, which makes it explicit and testable (override the dependency in tests).

## Core concept: CRUD operations

### Sync CRUD

```python
# main.py  (sync flavour)
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.orm import Session
from pydantic import BaseModel, EmailStr, ConfigDict

from db import engine
from deps import get_db
from models import Base, User


# Modern FastAPI uses a lifespan context manager instead of the deprecated
# @app.on_event("startup"). Code before `yield` runs at startup, code after
# `yield` runs at shutdown. (create_all is for demos only; prefer Alembic.)
@asynccontextmanager
async def lifespan(app: FastAPI):
    Base.metadata.create_all(engine)
    yield


app = FastAPI(lifespan=lifespan)


# ---- Pydantic schemas (request/response shapes) ----
class UserCreate(BaseModel):
    email: EmailStr
    name: str


class UserOut(BaseModel):
    id: int
    email: EmailStr
    name: str
    # Pydantic v2: read attributes off ORM objects, not just dicts.
    model_config = ConfigDict(from_attributes=True)


# ---- CREATE ----
@app.post("/users", response_model=UserOut, status_code=201)
def create_user(payload: UserCreate, db: Session = Depends(get_db)) -> User:
    user = User(email=payload.email, name=payload.name)
    db.add(user)            # stage the insert
    db.commit()             # flush + COMMIT
    db.refresh(user)        # reload to populate id / server defaults
    return user


# ---- READ one ----
@app.get("/users/{user_id}", response_model=UserOut)
def get_user(user_id: int, db: Session = Depends(get_db)) -> User:
    # session.get() is the fast path for primary-key lookups (uses identity map).
    user = db.get(User, user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user


# ---- READ many (with a filter) ----
@app.get("/users", response_model=list[UserOut])
def list_users(db: Session = Depends(get_db)) -> list[User]:
    stmt = select(User).order_by(User.id).limit(50)
    # scalars() unwraps Row -> the User object; all() materializes the list.
    return list(db.scalars(stmt).all())


# ---- UPDATE ----
@app.patch("/users/{user_id}", response_model=UserOut)
def update_user(user_id: int, payload: UserCreate, db: Session = Depends(get_db)) -> User:
    user = db.get(User, user_id)
    if user is None:
        raise HTTPException(404, "User not found")
    user.email = payload.email     # mutate the tracked object...
    user.name = payload.name
    db.commit()                    # ...and SQLAlchemy emits the UPDATE
    db.refresh(user)
    return user


# ---- DELETE ----
@app.delete("/users/{user_id}", status_code=204)
def delete_user(user_id: int, db: Session = Depends(get_db)) -> None:
    user = db.get(User, user_id)
    if user is None:
        raise HTTPException(404, "User not found")
    db.delete(user)
    db.commit()
```

### Async CRUD

The only differences: `await` the I/O calls, and use `db.execute(...)` / `await db.scalars(...)`.

```python
# main_async.py
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from deps import get_db          # the async get_db
from models import User

app = FastAPI()


@app.post("/users", status_code=201)
async def create_user(email: str, name: str, db: AsyncSession = Depends(get_db)):
    user = User(email=email, name=name)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return {"id": user.id, "email": user.email}


@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)     # awaited PK lookup
    if user is None:
        raise HTTPException(404, "User not found")
    return {"id": user.id, "email": user.email}


@app.get("/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.scalars(select(User).order_by(User.id))
    users = result.all()
    return [{"id": u.id, "email": u.email} for u in users]
```

Notice the API is nearly identical — `add` does not do I/O so it is not awaited, but `commit`, `refresh`, `get`, `execute`, and `scalars` all hit the database and so are awaited in the async version.

## Core concept: relationships, briefly

With the `relationship()` declared on the model, you can navigate object graphs. In **sync** code, a not-yet-loaded relationship triggers a lazy SELECT on access. In **async** code, lazy loading is **not allowed by default** (it would need to do hidden I/O outside an `await`), so you must eager-load.

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload

# Eager-load posts in one extra query (avoids the N+1 problem).
stmt = select(User).options(selectinload(User.posts)).where(User.id == 1)

# sync:
user = db.scalars(stmt).first()
print([p.title for p in user.posts])     # already loaded, no extra query

# async (selectinload is mandatory here, or accessing .posts raises):
user = (await db.scalars(stmt)).first()
print([p.title for p in user.posts])
```

`selectinload` is your everyday choice for one-to-many; `joinedload` is better for many-to-one / one-to-one. This is the SQLAlchemy answer to Django's `prefetch_related` / `select_related`.

## Core concept: migrations with Alembic

`Base.metadata.create_all()` is fine for a demo or tests, but it cannot alter an existing table. For real projects use **Alembic** — it diffs your models against the live schema and generates migration scripts.

```bash
# 1. Scaffold the migrations/ folder and alembic.ini
alembic init migrations
```

Edit two things in the scaffold:

```python
# migrations/env.py  — point Alembic at your models' metadata
from models import Base          # your DeclarativeBase subclass
target_metadata = Base.metadata  # replace the existing `target_metadata = None`
```

```ini
# alembic.ini  — set your DB URL (use the SYNC driver here; Alembic's
# default env.py runs synchronously). For async projects, either keep a
# sync URL just for migrations, or use the async template (alembic init -t async).
sqlalchemy.url = postgresql+psycopg://user:pass@localhost/mydb
```

Then the everyday loop:

```bash
# 2. Autogenerate a migration from model changes
alembic revision --autogenerate -m "create users and posts"

# 3. Apply migrations (run all pending up to head)
alembic upgrade head

# Useful extras:
alembic downgrade -1        # roll back one revision
alembic current            # show current DB revision
alembic history            # list all revisions
```

Always **review** the autogenerated script before running it — Alembic detects added/removed columns and tables well, but is weaker on renames (it sees a drop + add) and some type changes. This is the equivalent of Laravel's `make:migration` + `migrate`, except the script is auto-drafted from your models rather than hand-written.

## Core concept: when to go async vs sync, and not blocking the loop

FastAPI runs on an event loop (ASGI). An `async def` endpoint runs **on that loop**. If you do blocking work inside it — a sync DB call, `time.sleep()`, `requests.get()`, heavy CPU — you freeze the loop and stall *every* concurrent request, not just yours.

The decision table:

| Situation | Endpoint | Session |
|---|---|---|
| Mostly I/O-bound, high concurrency, async driver available | `async def` | `AsyncSession` |
| Existing sync codebase, simpler, "good enough" throughput | `def` (sync) | `Session` |
| Heavy CPU work | `def` (FastAPI offloads it to a threadpool) | either |

The crucial safety valve: a **sync `def` endpoint is run in a threadpool** by FastAPI, so blocking calls there do not freeze the loop. That is why mixing a sync `Session` with a sync `def` endpoint is perfectly safe and a totally legitimate choice. The danger is only the combination *async endpoint + blocking call*.

```python
import anyio

@app.get("/report")
async def report(db: AsyncSession = Depends(get_db)):
    # DB I/O: fine, it's awaited (non-blocking).
    rows = (await db.scalars(select(User))).all()

    # CPU-heavy or a sync-only library? Push it off the loop:
    summary = await anyio.to_thread.run_sync(expensive_pure_python, rows)
    return summary
```

Rule of thumb for the interview: **"async def must `await` all its I/O; if you can't, either make the endpoint a plain `def` (threadpool) or offload the blocking bit with `anyio.to_thread.run_sync` / `run_in_threadpool`."**

## Core concept: the SQLModel alternative (briefly)

[SQLModel](https://sqlmodel.tiangolo.com/) is by FastAPI's author. It merges your SQLAlchemy model and your Pydantic schema into **one class**, which removes the duplicate "ORM model + Pydantic schema" boilerplate. Under the hood it *is* SQLAlchemy + Pydantic.

```python
from sqlmodel import SQLModel, Field, create_engine, Session, select


class Hero(SQLModel, table=True):           # table=True => a real table
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    secret_name: str
    age: int | None = None


engine = create_engine("sqlite:///./heroes.db")
SQLModel.metadata.create_all(engine)

with Session(engine) as session:
    session.add(Hero(name="Deadpond", secret_name="Dive Wilson"))
    session.commit()
    heroes = session.exec(select(Hero)).all()   # note .exec, not .execute
```

Trade-offs: SQLModel is more concise and beginner-friendly, but it lags behind on supporting the newest SQLAlchemy/Pydantic features and some advanced relationship/typing cases get awkward. For an interview, know that it exists, that it is "Pydantic + SQLAlchemy fused," and that plain SQLAlchemy 2.x is the safer default for complex apps.

## Core concept: comparison to the Django ORM workflow

| Step | Django | FastAPI + SQLAlchemy |
|---|---|---|
| ORM included? | Yes, built in | No — you add SQLAlchemy |
| Define a model | `class User(models.Model)` | `class User(Base)` with `Mapped[...]` |
| DB config | `DATABASES` in `settings.py` | engine URL you create |
| Make a migration | `manage.py makemigrations` | `alembic revision --autogenerate` |
| Apply migration | `manage.py migrate` | `alembic upgrade head` |
| Query all | `User.objects.all()` | `db.scalars(select(User)).all()` |
| Get by PK | `User.objects.get(pk=1)` | `db.get(User, 1)` |
| Filter | `User.objects.filter(name="x")` | `select(User).where(User.name == "x")` |
| Create | `User.objects.create(...)` | `db.add(u); db.commit()` |
| Session/unit of work | implicit per request | explicit `Session`, your `get_db` dep |
| Async | `async`-capable ORM methods (`aget`, `acreate`) | `AsyncSession` + `await` |

The mental shift from Django: there is no global `Model.objects` manager. You always have an explicit **session** that you obtained for this request, and you build **statements** (`select()`) that you execute against it. More verbose, but more explicit about transactions and connections.

## Common mistakes & gotchas

- **Wrong driver in the URL.** `create_async_engine("postgresql+psycopg://...")` fails — async needs `+asyncpg`. Sync needs `+psycopg`/`+psycopg2`. SQLite async needs `+aiosqlite`.
- **Blocking the event loop.** Calling a sync `Session`, `requests`, or `time.sleep()` inside an `async def` endpoint stalls all requests. Use the async session, or make the endpoint sync `def`.
- **`MissingGreenlet` / lazy-load errors in async.** Accessing an un-loaded `relationship` (or a `server_default` attribute) on an async session after commit triggers hidden I/O. Fix with `selectinload`/`joinedload` and `expire_on_commit=False`.
- **Forgetting `expire_on_commit=False`.** By default SQLAlchemy expires all attributes after commit; the next attribute access re-queries — which is impossible to do lazily in async and wasteful in sync. Set it `False` on the session factory when you return objects after commit.
- **Forgetting `refresh()` after insert.** Without `db.refresh(user)`, server-side defaults and the new `id` may not be populated on the object you return.
- **Using the legacy `session.query(User)` API.** Works, but it is the 1.x style. Use `select()` + `scalars()` for 2.0.
- **Forgetting `from_attributes=True`** (`ConfigDict(from_attributes=True)`) on response models — Pydantic then can't read attributes off ORM objects and `response_model` validation fails.
- **One engine per request.** Create the engine **once** at module/startup scope. Creating an engine per request destroys connection pooling.
- **Pydantic v1 habits.** Use `model_dump()` not `.dict()`, and `ConfigDict` not an inner `class Config`. (`orm_mode=True` is now `from_attributes=True`.)
- **Mixing sync and async sessions in one engine.** An async engine yields async sessions only, and vice versa — pick one stack per app (or keep them strictly separate).

## Best practices

- Keep **one** engine and **one** session factory in a `db.py`; import them everywhere.
- Always provide the session through a **`yield` dependency** so cleanup is automatic and the dependency is overridable in tests.
- Use **Alembic** from day one; reserve `create_all()` for tests and quick spikes.
- Default to **`selectinload`/`joinedload`** to dodge N+1; never rely on lazy loading in async code.
- Separate **Pydantic schemas** (API shape) from **SQLAlchemy models** (DB shape) unless you deliberately choose SQLModel.
- Set `expire_on_commit=False`, `pool_pre_ping=True`, and turn `echo` off in production.
- Wrap multi-step writes in a single `commit()` so they share one transaction; let the `yield` dependency roll back on exceptions.
- Choose **sync** unless you have a concrete I/O-concurrency need — sync `def` endpoints run in a threadpool and are simpler to reason about.
- In tests, override `get_db` with a session bound to a transactional test database and roll back after each test.

## Interview Q&A

**Q: Why does FastAPI not include an ORM like Django or Laravel?**
**A:** By design it is a minimal ASGI web/validation layer. Persistence is left to the developer so you can pick the right tool (SQLAlchemy, Tortoise, raw asyncpg, a NoSQL ODM, or none). The de-facto standard pairing is SQLAlchemy for the ORM and Alembic for migrations.

**Q: What is the difference between the engine and the session in SQLAlchemy?**
**A:** The engine is the connection pool plus dialect — create one and share it for the whole app's lifetime. A session is a short-lived unit of work that tracks object changes and defines a transaction boundary; you create one per request via a `yield` dependency and close it afterward.

**Q: When should you use an async session and `async def` endpoints?**
**A:** When the workload is I/O-bound with meaningful concurrency and you have an async driver (e.g. asyncpg). Then awaited DB calls let the event loop serve other requests during I/O. For simpler apps or existing sync code, a sync `Session` with sync `def` endpoints is fine because FastAPI runs sync endpoints in a threadpool.

**Q: What happens if you make a blocking call inside an `async def` endpoint?**
**A:** It blocks the single event loop thread, stalling every concurrent request until it returns. Fixes: use the async session and `await` the I/O, switch the endpoint to a plain `def` (threadpool), or offload the blocking work with `anyio.to_thread.run_sync` / `run_in_threadpool`.

**Q: How do you provide a database session per request?**
**A:** With a dependency that `yield`s the session: create it before `yield`, hand it to the endpoint, and close/rollback after `yield` in a `finally` (sync) or via `async with` (async). Declare it as `Depends(get_db)`. It is also trivially overridable in tests.

**Q: What does `db.refresh(obj)` do and why call it after insert?**
**A:** It re-reads the row from the database into the object, populating values the DB generated — the auto-increment primary key and any `server_default` columns — so the object you return is complete.

**Q: Explain `default` vs `server_default` on `mapped_column`.**
**A:** `default` is a Python-side value SQLAlchemy applies during INSERT; it does not appear in the table DDL. `server_default` is rendered into the `CREATE TABLE` and applied by the database engine, so it also affects rows inserted outside your app.

**Q: Why might accessing a relationship raise an error with an async session?**
**A:** Lazy loading would require synchronous, hidden I/O, which is illegal on an async session. You must eager-load the relationship with `selectinload`/`joinedload`, or fetch it explicitly. Sync sessions allow lazy loading because they can block.

**Q: How does the migration workflow compare to Django/Laravel?**
**A:** Alembic plays the role of Django migrations / Laravel's migration tooling. `alembic revision --autogenerate` diffs your models against the DB to draft a script (like `makemigrations`), and `alembic upgrade head` applies it (like `migrate`). You should review autogenerated scripts since renames and some type changes are detected imperfectly.

**Q: What is SQLModel and when would you use it?**
**A:** SQLModel, by FastAPI's author, fuses a SQLAlchemy model and a Pydantic model into one class via `table=True`, cutting duplicate boilerplate. It is great for small/medium apps and learning. For complex apps wanting the newest SQLAlchemy/Pydantic features, plain SQLAlchemy 2.x is the safer choice.

**Q: Why set `expire_on_commit=False`?**
**A:** By default SQLAlchemy expires loaded attributes after `commit()`, so the next access re-queries. That breaks async code (it would need lazy I/O) and is wasteful when you return objects right after committing. Setting it `False` keeps the in-memory attribute values usable.

**Q: How do you query in 2.0 style versus 1.x?**
**A:** 2.0 uses construct-and-execute: `db.scalars(select(User).where(User.id == 1)).first()`. The 1.x `db.query(User).filter(...).first()` still works but is the legacy API; prefer `select()` + `execute`/`scalars`.

## Practice exercises

1. **Set up the stack.** Create `db.py` with a sync SQLite engine (`sqlite:///./app.db`) and a `SessionLocal` factory, plus a `get_db` `yield` dependency. Define a `Task` model (`id`, `title`, `done: bool default False`) using `Mapped`/`mapped_column`.

2. **CRUD endpoints.** Build POST `/tasks`, GET `/tasks`, GET `/tasks/{id}` (404 if missing), PATCH `/tasks/{id}` to toggle `done`, DELETE `/tasks/{id}` (204). Use Pydantic schemas with `ConfigDict(from_attributes=True)` and `response_model`.

3. **Go async.** Convert exercise 2 to an async stack: `create_async_engine` with `sqlite+aiosqlite:///./app.db`, `async_sessionmaker`, an async `get_db`, and `async def` endpoints that `await` their DB calls.

4. **Add a relationship.** Add a `Project` model with a one-to-many `tasks` relationship. Write a GET `/projects/{id}` that returns the project with its tasks eager-loaded via `selectinload` (mandatory in the async version).

5. **Alembic.** Initialize Alembic, point `target_metadata` at your `Base.metadata`, autogenerate a migration for the `Task` (and `Project`) tables, and run `alembic upgrade head`. Then add a `priority: int` column to `Task` and generate + apply a second migration.

6. **Don't block the loop.** Add an async endpoint `/slow-report` that runs a CPU-bound pure-Python function (e.g. summing a big range) without freezing the loop, using `anyio.to_thread.run_sync`.

<details><summary>Solutions</summary>

```python
# --- Exercise 1: db.py + Task model (sync) ---
from collections.abc import Generator
from sqlalchemy import create_engine, String
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, sessionmaker, Session, relationship,
)

engine = create_engine("sqlite:///./app.db", echo=False)
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


class Task(Base):
    __tablename__ = "tasks"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    done: Mapped[bool] = mapped_column(default=False)
    priority: Mapped[int] = mapped_column(default=0)          # added in ex.5
    project_id: Mapped[int | None] = mapped_column(           # added in ex.4
        nullable=True
    )


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
# --- Exercise 2: CRUD endpoints (sync) ---
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.orm import Session
from pydantic import BaseModel, ConfigDict

app = FastAPI()
Base.metadata.create_all(engine)   # demo only; replace with Alembic (ex.5)


class TaskCreate(BaseModel):
    title: str


class TaskOut(BaseModel):
    id: int
    title: str
    done: bool
    model_config = ConfigDict(from_attributes=True)


@app.post("/tasks", response_model=TaskOut, status_code=201)
def create_task(payload: TaskCreate, db: Session = Depends(get_db)):
    task = Task(title=payload.title)
    db.add(task)
    db.commit()
    db.refresh(task)
    return task


@app.get("/tasks", response_model=list[TaskOut])
def list_tasks(db: Session = Depends(get_db)):
    return list(db.scalars(select(Task).order_by(Task.id)).all())


@app.get("/tasks/{task_id}", response_model=TaskOut)
def get_task(task_id: int, db: Session = Depends(get_db)):
    task = db.get(Task, task_id)
    if task is None:
        raise HTTPException(404, "Task not found")
    return task


@app.patch("/tasks/{task_id}", response_model=TaskOut)
def toggle_task(task_id: int, db: Session = Depends(get_db)):
    task = db.get(Task, task_id)
    if task is None:
        raise HTTPException(404, "Task not found")
    task.done = not task.done
    db.commit()
    db.refresh(task)
    return task


@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int, db: Session = Depends(get_db)):
    task = db.get(Task, task_id)
    if task is None:
        raise HTTPException(404, "Task not found")
    db.delete(task)
    db.commit()
```

```python
# --- Exercise 3: async stack ---
from collections.abc import AsyncGenerator
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import (
    create_async_engine, async_sessionmaker, AsyncSession,
)

async_engine = create_async_engine("sqlite+aiosqlite:///./app.db")
AsyncSessionLocal = async_sessionmaker(bind=async_engine, expire_on_commit=False)


async def get_async_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session


app = FastAPI()


@app.post("/tasks", status_code=201)
async def create_task(title: str, db: AsyncSession = Depends(get_async_db)):
    task = Task(title=title)
    db.add(task)
    await db.commit()
    await db.refresh(task)
    return {"id": task.id, "title": task.title}


@app.get("/tasks/{task_id}")
async def get_task(task_id: int, db: AsyncSession = Depends(get_async_db)):
    task = await db.get(Task, task_id)
    if task is None:
        raise HTTPException(404, "Task not found")
    return {"id": task.id, "title": task.title, "done": task.done}


@app.get("/tasks")
async def list_tasks(db: AsyncSession = Depends(get_async_db)):
    result = await db.scalars(select(Task).order_by(Task.id))
    return [{"id": t.id, "title": t.title} for t in result.all()]
```

```python
# --- Exercise 4: relationship + eager load (async) ---
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship, selectinload


class Project(Base):
    __tablename__ = "projects"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    tasks: Mapped[list["Task"]] = relationship(back_populates="project")


# Add to Task:
#   project_id: Mapped[int | None] = mapped_column(ForeignKey("projects.id"))
#   project: Mapped["Project | None"] = relationship(back_populates="tasks")


@app.get("/projects/{project_id}")
async def get_project(project_id: int, db: AsyncSession = Depends(get_async_db)):
    stmt = (
        select(Project)
        .options(selectinload(Project.tasks))   # eager-load: required in async
        .where(Project.id == project_id)
    )
    project = (await db.scalars(stmt)).first()
    if project is None:
        raise HTTPException(404, "Project not found")
    return {
        "id": project.id,
        "name": project.name,
        "tasks": [{"id": t.id, "title": t.title} for t in project.tasks],
    }
```

```bash
# --- Exercise 5: Alembic ---
alembic init migrations
# In migrations/env.py:   from db import Base;  target_metadata = Base.metadata
# In alembic.ini:         sqlalchemy.url = sqlite:///./app.db   (sync URL)

alembic revision --autogenerate -m "create tasks and projects"
alembic upgrade head

# After adding `priority: Mapped[int] = mapped_column(default=0)` to Task:
alembic revision --autogenerate -m "add priority to tasks"
alembic upgrade head
```

```python
# --- Exercise 6: don't block the loop ---
import anyio
from fastapi import FastAPI

app = FastAPI()


def heavy_sum(n: int) -> int:
    return sum(i * i for i in range(n))     # CPU-bound, pure Python


@app.get("/slow-report")
async def slow_report():
    # Offload to a worker thread so the event loop stays responsive.
    total = await anyio.to_thread.run_sync(heavy_sum, 10_000_000)
    return {"total": total}
```

</details>

## Cheat sheet

```python
# Engine + session factory (create ONCE)
engine = create_engine("postgresql+psycopg://u:p@h/db")            # sync
engine = create_async_engine("postgresql+asyncpg://u:p@h/db")      # async
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)
AsyncSessionLocal = async_sessionmaker(bind=engine, expire_on_commit=False)

# Model (2.0 declarative)
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    name: Mapped[str | None]                       # nullable

# Per-request session dependency
def get_db():                                      # sync
    db = SessionLocal()
    try: yield db
    finally: db.close()

async def get_db():                                # async
    async with AsyncSessionLocal() as s: yield s

# CRUD quick map
db.add(obj); db.commit(); db.refresh(obj)          # create
db.get(User, pk)                                   # read by PK (await in async)
db.scalars(select(User).where(...)).all()          # read many
obj.attr = x; db.commit()                          # update
db.delete(obj); db.commit()                        # delete

# Eager load (avoid N+1; required in async)
select(User).options(selectinload(User.posts))     # one-to-many
select(Post).options(joinedload(Post.author))      # many-to-one
```

| Driver / URL | Use |
|---|---|
| `postgresql+psycopg://` | sync Postgres |
| `postgresql+asyncpg://` | async Postgres |
| `sqlite:///./app.db` | sync SQLite |
| `sqlite+aiosqlite:///./app.db` | async SQLite |

| Alembic command | Does |
|---|---|
| `alembic init migrations` | scaffold |
| `alembic revision --autogenerate -m "msg"` | draft migration from models |
| `alembic upgrade head` | apply all pending |
| `alembic downgrade -1` | roll back one |
| `alembic current` / `history` | inspect state |

| Pydantic v2 (not v1) | Use |
|---|---|
| `model_dump()` | not `.dict()` |
| `model_validate(obj)` | not `.from_orm()` |
| `ConfigDict(from_attributes=True)` | not `class Config: orm_mode = True` |

**Async safety rule:** an `async def` endpoint must `await` all I/O; never call a blocking sync DB/HTTP/sleep inside it. If you must, use a plain `def` endpoint (threadpool) or `anyio.to_thread.run_sync(...)`.
