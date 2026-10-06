# 47 — Django vs FastAPI — The Comparison

> Week 4 · DRF & FastAPI

You have now built REST APIs in both Django REST Framework (DRF) and FastAPI. This lesson steps back and answers the question every interviewer eventually asks: **"When would you choose Django over FastAPI, and why?"** There is no single "best" framework — there is only the right tool for the project, the team, and the constraints in front of you. By the end of this lesson you will be able to defend a choice with concrete technical reasoning instead of hype.

## What you will learn

- The core **philosophy** of each framework: batteries-included monolith vs. minimal async micro-framework.
- How their **architectures** differ: Django's MVT + DRF layered stack vs. FastAPI's ASGI + Pydantic + dependency-injection model.
- The **sync vs. async** execution model and what it actually means for performance.
- What you get **built-in** (ORM, admin, auth, migrations, templates) vs. what you **bring yourself** (SQLAlchemy, custom auth, Alembic).
- **Validation** compared: DRF serializers vs. Pydantic models.
- **Auto-generated API docs**: drf-spectacular vs. FastAPI's built-in OpenAPI/Swagger.
- **Ecosystem, maturity, community, and learning curve.**
- **Typical use cases** for each, plus team and project factors.
- A **decision checklist** and a side-by-side **comparison table** you can recite in an interview.

## PHP/Laravel bridge

The cleanest mental mapping: **Django is Laravel**, and **FastAPI is closer to Lumen or Slim (with type-safety bolted on)**. Django, like Laravel, is a "batteries-included" full-stack framework — it ships an ORM (Eloquent ↔ Django ORM), migrations (Artisan migrate ↔ Django migrations), an auth system, a templating engine (Blade ↔ Django Templates), and an admin layer. FastAPI, like Lumen/Slim, gives you routing and request handling and lets you assemble the rest yourself (you pick the ORM, the auth, the migration tool). The big twist FastAPI adds — which has no exact Laravel equivalent — is that **type hints drive everything**: request validation, serialization, and interactive docs all fall out of Python type annotations, the way PHP 8 attributes plus a heavy validation library *might* if Laravel were rebuilt around them.

## Core concept — Two philosophies

### Django + DRF: the batteries-included monolith

Django's founding idea is **"the web framework for perfectionists with deadlines."** It bundles everything a typical database-backed web application needs and enforces conventions so that any Django developer can navigate any Django project. DRF sits on top of Django to turn it into a first-class REST API toolkit (serializers, viewsets, routers, browsable API, auth classes).

The trade-off: you adopt the whole ecosystem. The ORM, the settings module, the app structure, the request/response cycle — it's all Django's way. That uniformity is a *feature* for teams and long-lived products.

### FastAPI: the minimal async micro-framework

FastAPI's founding idea is **"high performance, easy to learn, fast to code, based on standard Python type hints."** It is built on **Starlette** (the ASGI toolkit handling routing and the async machinery) and **Pydantic** (data validation). FastAPI itself is small; it deliberately leaves database access, migrations, and auth as your choice. You compose the stack you want.

The trade-off: more decisions and more glue code, but also more freedom and a leaner runtime.

```text
            DJANGO + DRF (full stack)          FASTAPI (composed stack)
          ┌───────────────────────────┐      ┌───────────────────────────┐
  Docs    │ drf-spectacular (add-on)  │      │ OpenAPI/Swagger (built-in)│
  Valid.  │ DRF Serializers           │      │ Pydantic v2 (built-in)    │
  Auth    │ Django auth + DRF auth    │      │ bring-your-own (deps)     │
  ORM     │ Django ORM (built-in)     │      │ SQLAlchemy / your choice  │
  Migrate │ Django migrations         │      │ Alembic / your choice     │
  Routing │ Django URLconf + routers  │      │ Starlette routing         │
  Server  │ WSGI (sync) / ASGI        │      │ ASGI (async-first)        │
          └───────────────────────────┘      └───────────────────────────┘
```

## Core concept — Architecture: MVT+DRF vs ASGI+Pydantic

### Django's layered request flow

A request in a DRF API travels through a well-defined stack: **URLconf → middleware → view/viewset → serializer → ORM → database**, then back out through the serializer to a `Response`. Django follows **MVT (Model-View-Template)**, which is its naming for MVC: the *Model* is the ORM class, the *View* contains the logic, and the *Template* renders HTML (mostly unused in a pure JSON API, where DRF's serializer + renderer take that role).

```python
# Django + DRF: a typical CRUD endpoint
# models.py
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

# serializers.py
from rest_framework import serializers
from .models import Product

class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ["id", "name", "price"]

# views.py
from rest_framework import viewsets
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

# urls.py
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet

router = DefaultRouter()
router.register("products", ProductViewSet)
urlpatterns = router.urls
# => One registration gives you list/create/retrieve/update/delete + browsable API
```

Notice how little code creates a full CRUD resource. Convention does the heavy lifting. The cost is that the framework is "in charge" — you work within its lifecycle.

### FastAPI's ASGI + Pydantic + dependency-injection flow

FastAPI is **ASGI-native** (async from the ground up). A request flows through: **Starlette router → dependencies (DI) → path operation function → Pydantic response model → JSON**. There is no built-in ORM layer; you call your database yourself, often through a dependency.

```python
# FastAPI: the same CRUD idea, composed by hand
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, ConfigDict
from decimal import Decimal

app = FastAPI()

class ProductIn(BaseModel):
    name: str
    price: Decimal

class ProductOut(ProductIn):
    id: int
    model_config = ConfigDict(from_attributes=True)  # Pydantic v2: read from ORM objects

# In-memory store standing in for a database
_db: dict[int, ProductOut] = {}
_next_id = 1

@app.post("/products", response_model=ProductOut, status_code=201)
def create_product(payload: ProductIn) -> ProductOut:
    global _next_id
    product = ProductOut(id=_next_id, **payload.model_dump())
    _db[_next_id] = product
    _next_id += 1
    return product

@app.get("/products/{product_id}", response_model=ProductOut)
def get_product(product_id: int) -> ProductOut:
    if product_id not in _db:
        raise HTTPException(status_code=404, detail="Not found")
    return _db[product_id]
# => Request body validation, response shaping, and OpenAPI docs all come from the type hints
```

You write more lines per endpoint than DRF's viewset, but **everything is explicit**: you can see exactly where validation, persistence, and serialization happen. There's no "magic" lifecycle to learn — just functions and types.

### The big architectural difference: dependency injection

FastAPI's standout feature is its **dependency injection (DI)** system via `Depends`. Reusable concerns — a DB session, the current user, pagination params — are declared as function parameters and resolved automatically. Django/DRF has no comparable first-class DI; it uses class attributes, mixins, and middleware instead.

```python
# FastAPI DI: shared dependencies declared once, injected everywhere
from fastapi import Depends, Header, HTTPException

def get_current_user(x_token: str = Header(...)) -> str:
    if x_token != "secret-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return "alice"

@app.get("/me")
def read_me(user: str = Depends(get_current_user)) -> dict[str, str]:
    return {"user": user}
# => get_current_user runs before the handler; reuse it on any route by adding the param
```

## Core concept — Sync vs async and performance

This is the difference interviewers probe hardest, so be precise.

### What sync vs async actually means

- **Django (traditional) is synchronous / WSGI.** Each request occupies a worker process/thread for its full duration. If a request waits 200 ms on a database or an external HTTP call, that worker is *blocked* and can't serve anyone else during the wait. You scale by running more workers (e.g., Gunicorn processes).
- **FastAPI is asynchronous / ASGI-first.** With `async def` handlers and `await`ed I/O, a single worker can *suspend* a request during its wait and serve other requests in the meantime — using an event loop on one thread. This shines for **I/O-bound** workloads with many concurrent, slow-ish external calls.

```python
# FastAPI: async handler that overlaps I/O instead of blocking
import asyncio
import httpx
from fastapi import FastAPI

app = FastAPI()

@app.get("/aggregate")
async def aggregate() -> dict:
    async with httpx.AsyncClient() as client:
        # Both requests fly concurrently; the worker isn't blocked while waiting
        a, b = await asyncio.gather(
            client.get("https://example.com/a"),
            client.get("https://example.com/b"),
        )
    return {"a": a.status_code, "b": b.status_code}
# => Total latency ≈ the slower of the two calls, not their sum
```

### The honest caveats

1. **Django supports async too (since 3.1+).** You can write `async def` views, async middleware, and `5.x` ships async ORM methods (`aget`, `acreate`, `aupdate`, `adelete`, plus `async for` iteration, etc.). Note there is no `afilter` — `filter()` is lazy and doesn't touch the database, so it needs no async variant; the query runs when you `await` a terminal method or iterate with `async for`. But Django's *core and ecosystem* remain sync-first; many third-party packages and the ORM internals are not fully async, so you can hit sync boundaries. FastAPI is async *all the way down*, which is cleaner.

2. **Async is not "faster" for CPU-bound work.** A tight loop crunching numbers blocks the event loop just like a sync call. For CPU-bound tasks, both frameworks need worker processes or an offloaded task queue (Celery, RQ, or a thread/process pool). Async wins for **I/O-bound concurrency**, not raw computation.

3. **Benchmarks are marketing.** FastAPI tops many "hello world" throughput charts because of ASGI + Starlette + a lean stack. In a real app dominated by database round-trips and serialization, the framework is rarely the bottleneck — your queries and N+1 problems are. Quote raw RPS numbers cautiously in interviews; emphasize *workload shape* instead.

4. **Mixing sync and async incorrectly is dangerous.** In a FastAPI `async def` handler, calling a blocking library (e.g., a sync DB driver or `time.sleep`) blocks the whole event loop and tanks concurrency. Use `def` (FastAPI runs it in a threadpool) or an async-native library instead.

```python
# FastAPI gotcha: blocking call inside async handler freezes the event loop
import time
from fastapi import FastAPI

app = FastAPI()

@app.get("/bad")
async def bad() -> dict:
    time.sleep(2)          # BLOCKS the event loop — every other request stalls
    return {"ok": True}

@app.get("/good")
def good() -> dict:        # plain def -> FastAPI offloads to a threadpool
    time.sleep(2)          # blocking is fine here; it won't freeze the loop
    return {"ok": True}
```

## Core concept — Built-ins vs bring-your-own

### Django gives you a kitchen

| Concern | Django ships it | FastAPI: you choose |
|---|---|---|
| ORM | Django ORM (built-in, integrated) | SQLAlchemy 2.x, SQLModel, Tortoise, or none |
| Migrations | `makemigrations` / `migrate` (auto) | Alembic (manual setup), or your ORM's tool |
| Admin UI | Django Admin (auto CRUD dashboard) | None — build it or use SQLAdmin/Starlette-admin |
| Auth | Sessions, users, groups, permissions | Bring-your-own (OAuth2/JWT via libs + DI) |
| Templates | Django Template Language | Jinja2 (optional) |
| Forms | Django Forms / DRF serializers | Pydantic models |

The Django **admin** deserves special mention: with a few lines you get a production-usable CRUD back office over your models. There is no FastAPI equivalent in the box — this alone justifies Django for internal tools and content-heavy apps.

```python
# Django: a full admin dashboard for a model in 3 lines
from django.contrib import admin
from .models import Product

admin.site.register(Product)
# => Searchable, filterable, paginated CRUD UI at /admin/ — no equivalent ships with FastAPI
```

Django **migrations** are likewise automatic: change a model, run `makemigrations`, and Django diffs the schema and generates the migration for you. With FastAPI you wire up **Alembic** yourself and (typically) write or autogenerate migrations against your SQLAlchemy models.

```bash
# Django — schema changes are framework-managed
python manage.py makemigrations   # diff models -> generate migration file
python manage.py migrate          # apply to the database

# FastAPI + SQLAlchemy — you set up Alembic once, then drive it yourself
alembic revision --autogenerate -m "add product"
alembic upgrade head
```

### FastAPI gives you a toolbox

Nothing is imposed. Want SQLAlchemy 2.x with async sessions? Plug it in via a dependency. Prefer raw SQL or an entirely different datastore? Fine. This freedom is liberating for microservices and unusual architectures, and tedious for a standard CRUD app where Django would have handed you 80% for free.

```python
# FastAPI + SQLAlchemy 2.x: a DB session provided through dependency injection
from collections.abc import Generator
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker
from fastapi import Depends

engine = create_engine("sqlite:///./app.db")
SessionLocal = sessionmaker(bind=engine)

def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db          # injected into handlers; cleaned up after the response
    finally:
        db.close()

# Usage in a handler:
# def list_products(db: Session = Depends(get_db)) -> list[ProductOut]: ...
```

## Core concept — Validation: DRF serializers vs Pydantic

Both frameworks validate input and shape output, but the mechanisms differ in spirit.

- **DRF serializers** are explicit, class-based declarations that double as validators *and* (de)serializers, tightly coupled to the Django ORM via `ModelSerializer`. Field types are DRF's own (`serializers.CharField`, etc.). Validation runs through `is_valid()`.
- **Pydantic v2 models** use plain Python **type hints** as the schema. Validation is automatic on instantiation/parsing, powered by a fast Rust core (`pydantic-core`). The same model defines the request body, the response shape, and the OpenAPI schema.

```python
# DRF serializer: explicit fields, ORM-aware, validated via is_valid()
from rest_framework import serializers

class SignupSerializer(serializers.Serializer):
    email = serializers.EmailField()
    age = serializers.IntegerField(min_value=18)

    def validate_email(self, value: str) -> str:
        if value.endswith("@spam.com"):
            raise serializers.ValidationError("Disposable email not allowed")
        return value

s = SignupSerializer(data={"email": "a@x.com", "age": 25})
s.is_valid(raise_exception=True)
print(s.validated_data)   # => {'email': 'a@x.com', 'age': 25}
```

```python
# Pydantic v2: type hints ARE the schema; validated on construction
from pydantic import BaseModel, EmailStr, Field, field_validator

class Signup(BaseModel):
    email: EmailStr
    age: int = Field(ge=18)

    @field_validator("email")
    @classmethod
    def no_spam(cls, v: str) -> str:
        if v.endswith("@spam.com"):
            raise ValueError("Disposable email not allowed")
        return v

s = Signup(email="a@x.com", age=25)
print(s.model_dump())     # => {'email': 'a@x.com', 'age': 25}
```

**Key contrast:** DRF serializers know about your *database models*; Pydantic models know about *Python types*. DRF's tight ORM coupling means less code for model-backed APIs (`ModelSerializer`); Pydantic's type-first approach means your validation logic is reusable far beyond the web layer and powers the docs for free.

## Core concept — Auto API docs

- **FastAPI**: interactive **Swagger UI** at `/docs` and **ReDoc** at `/redoc` are **built in and on by default**, generated from your type hints and Pydantic models. Zero extra setup. The OpenAPI JSON is at `/openapi.json`.
- **DRF**: the **browsable API** is built in (a clickable HTML interface), but a proper **OpenAPI 3 schema** with Swagger/ReDoc requires the **drf-spectacular** package, installed and configured.

```python
# DRF: OpenAPI docs are an add-on (drf-spectacular)
# settings.py
INSTALLED_APPS = [..., "drf_spectacular"]
REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}
# urls.py
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView
urlpatterns += [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema")),
]
```

```python
# FastAPI: docs are free and automatic — nothing to configure
from fastapi import FastAPI
app = FastAPI(title="My API", version="1.0.0")
# => Visit /docs (Swagger UI) and /redoc (ReDoc) immediately
```

This "docs for free" experience is one of FastAPI's most loved features, and a genuine productivity win for API-first teams.

## Core concept — Ecosystem, maturity, community, learning curve

| Dimension | Django + DRF | FastAPI |
|---|---|---|
| First released | Django 2005, DRF ~2011 | 2018 |
| Maturity | Battle-tested over ~20 years; huge installed base | Younger but extremely popular and stable |
| Community/packages | Enormous: thousands of reusable Django apps | Growing fast; smaller but vibrant |
| Hiring pool | Very large; well-understood patterns | Large and growing, esp. among newer devs |
| Conventions | Strong, opinionated — consistency across projects | Few — architecture is up to you |
| Learning curve | Steeper upfront (more concepts, "the Django way") | Gentle to start; complexity arrives with the stack you add |
| Long-term support | Predictable LTS releases | Fast-moving but mature; depends on Starlette/Pydantic |

**Learning curve nuance to mention in interviews:** FastAPI is easy to *start* with (a 10-line app) but the *full picture* — async correctness, choosing and wiring an ORM, migrations, auth, project structure — pushes complexity onto you. Django is harder to start (more upfront concepts) but then hands you proven answers to those same questions. So "FastAPI is easier" is true only for the first hour.

## Core concept — Typical use cases & team factors

### Reach for Django + DRF when…

- The app is **content-heavy or CRUD-dominant** (CMS, e-commerce, SaaS dashboards, internal tools).
- You need the **admin** out of the box.
- The data model is **relational and central**, and you want a mature, integrated ORM + migrations.
- You value **convention and consistency** across a team and a multi-year codebase.
- You want **rapid development** of a standard web backend with batteries included.
- **Enterprise** requirements: established patterns, large hiring pool, long-term maintenance.

### Reach for FastAPI when…

- You're building **high-performance async APIs** or **microservices** with lots of concurrent I/O (external APIs, queues, websockets).
- You're **serving ML models** or other compute pipelines behind an HTTP layer (its async + type-driven design fits inference services well).
- You want **type-driven development** and automatic docs as first-class citizens.
- The service is **small and focused** and you don't need most of Django's machinery.
- You want **freedom** to compose an unconventional stack.

### Team & project factors (the answer interviewers reward)

- **Team familiarity:** a Django-fluent team ships faster on Django, full stop. Don't pick FastAPI to chase benchmarks if it slows your team down.
- **Timeline:** tight deadline + standard requirements → Django's built-ins win.
- **Service scope:** monolith/back office → Django; lean independent service → FastAPI.
- **Existing infra:** matching the rest of your stack often outweighs framework purity.
- **They can coexist:** a Django monolith for the core product plus a FastAPI microservice for a hot, async-heavy path is a common, legitimate architecture.

## Common mistakes & gotchas

- **Calling async "always faster."** Async helps **I/O-bound concurrency**, not CPU-bound work. Saying "FastAPI is faster" without qualification is a red flag in interviews.
- **Blocking the event loop** in a FastAPI `async def` handler (sync DB driver, `time.sleep`, heavy CPU). Use plain `def` (threadpool) or async-native libraries.
- **Thinking Django can't do async.** It can (3.1+ views, async ORM in 5.x), but the ecosystem is sync-first — be precise about the difference.
- **Underestimating what FastAPI doesn't give you.** No ORM, no migrations, no admin, no auth out of the box — you own those decisions and the glue code.
- **Choosing on benchmarks alone.** Real bottlenecks are queries, N+1s, serialization, and network — rarely the framework's request loop.
- **Ignoring team skills.** The "best" framework your team can't maintain is the wrong framework.
- **Pydantic v1 habits.** Use `model_dump()` (not `.dict()`) and `ConfigDict` / `model_config` (not the old inner `class Config` only). DRF and Pydantic are *different* tools — don't expect serializer semantics from Pydantic or vice versa.

## Best practices

- **Match the tool to the workload and the team**, not to hype. State your reasoning in those terms.
- For **standard relational CRUD / content apps with a deadline**, default to **Django + DRF**.
- For **async-heavy, I/O-bound microservices, ML serving, or type-driven API-first work**, default to **FastAPI**.
- In FastAPI, use **async-native libraries** end-to-end (e.g., async SQLAlchemy, `httpx`) and reserve `def` handlers for genuinely blocking work.
- In Django, lean on the **built-ins** (admin, migrations, auth) rather than reinventing them — that's the whole point.
- **Document with the grain:** drf-spectacular for DRF; rely on FastAPI's built-in OpenAPI.
- Don't be dogmatic — **mixed architectures** (Django core + FastAPI side-services) are a valid, mature pattern.
- Whatever you pick, the **database access patterns** (indexes, query efficiency, avoiding N+1) matter more for performance than the framework choice.

## Interview Q&A

**Q: In one sentence each, how do Django and FastAPI differ in philosophy?**
**A:** Django is a batteries-included, convention-driven full-stack framework that hands you an ORM, admin, auth, and migrations; FastAPI is a minimal, async-first micro-framework built on Starlette and Pydantic that gives you routing and type-driven validation and lets you compose everything else yourself.

**Q: Is FastAPI faster than Django? Be precise.**
**A:** FastAPI generally has higher raw throughput on lightweight/I/O-bound benchmarks because it's ASGI-native (Starlette) with a lean stack, and async lets one worker overlap many concurrent I/O waits. But it's not faster for CPU-bound work, and in real apps the bottleneck is usually database queries and serialization, not the framework. So "faster" depends entirely on the workload shape.

**Q: Can Django do async? What's the catch?**
**A:** Yes — async views and middleware since 3.1, and async ORM methods (`aget`, `acreate`, etc.) in modern versions. The catch is that Django's core and much of its ecosystem are sync-first, so you can hit sync boundaries; FastAPI is async all the way down, which is cleaner for heavily concurrent I/O.

**Q: What does FastAPI NOT give you that Django does?**
**A:** No built-in ORM, no migration tool, no admin UI, no auth/user system, and no enforced project structure. With FastAPI you choose and wire up SQLAlchemy (or another ORM), Alembic for migrations, and your own auth — typically via dependency injection.

**Q: Compare DRF serializers and Pydantic models.**
**A:** DRF serializers are explicit, class-based, ORM-aware validators/serializers validated via `is_valid()`, and `ModelSerializer` generates fields from a Django model. Pydantic v2 uses plain type hints as the schema, validates automatically on construction via a Rust core, and isn't tied to any ORM — the same model powers request validation, response shaping, and the OpenAPI docs.

**Q: How do automatic API docs differ between the two?**
**A:** FastAPI generates interactive Swagger UI (`/docs`) and ReDoc (`/redoc`) from your type hints automatically, with no setup. DRF ships a browsable HTML API by default, but a real OpenAPI 3 schema with Swagger/ReDoc requires installing and configuring drf-spectacular.

**Q: What is FastAPI's dependency injection and why does it matter?**
**A:** `Depends` lets you declare reusable concerns — a DB session, the current user, pagination — as function parameters that FastAPI resolves automatically before the handler runs. It makes shared logic explicit, testable (easy to override in tests), and composable. Django/DRF has no first-class DI equivalent; it uses class attributes, mixins, and middleware.

**Q: When would you pick Django + DRF for a new project?**
**A:** For content-heavy or CRUD-dominant apps (CMS, e-commerce, SaaS dashboards, internal tools), when I need the admin and integrated ORM/migrations out of the box, when the team knows Django, or when a tight deadline favors batteries-included rapid development on a relational data model.

**Q: When would you pick FastAPI?**
**A:** For high-performance async APIs and microservices with heavy concurrent I/O, for ML model serving, when I want type-driven development with automatic docs, and for small focused services where Django's machinery would be overkill and I want freedom to compose the stack.

**Q: What's the biggest performance mistake teams make with FastAPI?**
**A:** Calling blocking code (a sync DB driver, `time.sleep`, or heavy CPU work) inside an `async def` handler, which blocks the single-threaded event loop and destroys concurrency. The fix is to use async-native libraries, or declare the handler as plain `def` so FastAPI runs it in a threadpool, or offload CPU work to a task queue.

**Q: Can you use both in the same architecture?**
**A:** Yes, and it's common: a Django monolith for the core product (with its admin and mature CRUD), plus one or more FastAPI microservices for hot, async-heavy paths like real-time integrations or ML inference. They communicate over HTTP/queues. Choosing a framework per-service rather than per-company is a sign of pragmatic architecture.

**Q: How does the learning curve compare?**
**A:** FastAPI is far easier for the first hour — a tiny typed app with free docs. But its real-world complexity (async correctness, choosing/wiring an ORM, migrations, auth, structure) is pushed onto you. Django has a steeper start (more concepts, "the Django way") but then provides proven answers to all those questions, so it's arguably easier across a full project's lifetime.

## Practice exercises

1. **Decision drill.** For each scenario, name the framework you'd choose and give a one-line justification: (a) an internal admin tool over a relational DB on a two-week deadline; (b) a microservice that fans out to 8 third-party APIs per request and aggregates them; (c) an HTTP layer in front of a PyTorch model; (d) a multi-year e-commerce platform for a Django-fluent team.

2. **Architecture sketch.** In plain text, draw the request flow for a DRF CRUD endpoint and for a FastAPI CRUD endpoint, labeling where validation, persistence, and serialization happen in each.

3. **Spot the bug.** The handler below is `async def` but stalls the whole server under load. Identify why and give two correct fixes.
   ```python
   import time
   from fastapi import FastAPI
   app = FastAPI()

   @app.get("/report")
   async def report() -> dict:
       time.sleep(3)
       return {"status": "done"}
   ```

4. **Built-ins map.** List the five Django built-ins (ORM, migrations, admin, auth, templates) and name the FastAPI-world tool (or "none") you'd reach for to cover each.

5. **Validation translation.** Rewrite this DRF serializer as a Pydantic v2 model (Python 3.12, modern API), preserving the constraints.
   ```python
   from rest_framework import serializers
   class OrderSerializer(serializers.Serializer):
       quantity = serializers.IntegerField(min_value=1, max_value=100)
       coupon = serializers.CharField(required=False, allow_null=True)
   ```

6. **Interview rehearsal.** In 4-6 sentences, answer aloud: "We're starting a greenfield REST API. Walk me through how you'd decide between Django and FastAPI." Hit workload, team, built-ins, and timeline.

<details><summary>Solutions</summary>

**1. Decision drill.**
(a) **Django + DRF** — admin + migrations + ORM give a relational CRUD tool fastest under a deadline.
(b) **FastAPI** — async + `asyncio.gather` overlaps the 8 I/O calls, cutting latency to roughly the slowest call.
(c) **FastAPI** — async, type-driven, lean; ideal HTTP front for ML inference with free docs.
(d) **Django + DRF** — long-lived relational app, team fluency, conventions and ecosystem pay off over years.

**2. Architecture sketch.**
DRF: `URLconf → middleware → viewset → serializer.is_valid() (validation) → Django ORM (persistence) → serializer (serialization) → Response`.
FastAPI: `Starlette router → dependencies (e.g., DB session) → Pydantic parses body (validation) → handler calls DB/ORM (persistence) → response_model / Pydantic (serialization) → JSON`.

**3. Spot the bug.** `time.sleep(3)` is a *blocking* call inside an `async def` handler, so it freezes the single-threaded event loop and stalls every concurrent request for 3 seconds. Fixes: (a) make the handler plain `def` so FastAPI runs it in a threadpool; or (b) use a non-blocking await, e.g. `await asyncio.sleep(3)` (and for real I/O, use async-native libraries like `httpx`/async SQLAlchemy).

**4. Built-ins map.**
- ORM → SQLAlchemy 2.x (or SQLModel/Tortoise).
- Migrations → Alembic.
- Admin → none built-in (SQLAdmin / Starlette-admin as third-party options).
- Auth → bring-your-own (OAuth2/JWT via libraries, wired through `Depends`).
- Templates → Jinja2 (optional).

**5. Validation translation.**
```python
from pydantic import BaseModel, Field

class Order(BaseModel):
    quantity: int = Field(ge=1, le=100)
    coupon: str | None = None   # optional + nullable

o = Order(quantity=5)
print(o.model_dump())   # => {'quantity': 5, 'coupon': None}
```

**6. Interview rehearsal (model answer).**
"First I'd characterize the workload: if it's standard relational CRUD or content-heavy with an admin need, Django + DRF gives me an ORM, migrations, admin, and auth out of the box, which is the fastest path under a deadline. If it's an async-heavy, I/O-bound microservice or an ML-serving layer, FastAPI's ASGI core, type-driven validation, and free OpenAPI docs fit better. Then I weigh the team: a Django-fluent team ships faster on Django regardless of benchmarks. I'd also consider scope — monolith versus lean independent service — and remember the two can coexist. Net: I'd default to Django for batteries-included relational apps and FastAPI for high-concurrency, type-first services, and I'd justify it on workload and team rather than raw RPS."

</details>

## Cheat sheet

| Dimension | Django + DRF | FastAPI |
|---|---|---|
| Philosophy | Batteries-included full-stack | Minimal async micro-framework |
| Built on | Django + DRF | Starlette (ASGI) + Pydantic v2 |
| Concurrency | Sync/WSGI first (async since 3.1) | Async/ASGI first |
| ORM | Django ORM (built-in) | SQLAlchemy / your choice |
| Migrations | `makemigrations`/`migrate` (auto) | Alembic (manual setup) |
| Admin | Django Admin (built-in) | None (third-party only) |
| Auth | Built-in users/sessions/perms | Bring-your-own (via `Depends`) |
| Validation | DRF serializers (`is_valid()`) | Pydantic models (type hints) |
| Docs | Browsable API; OpenAPI via drf-spectacular | Swagger `/docs` + ReDoc (built-in) |
| DI | None (mixins/middleware) | First-class `Depends` |
| Best for | CRUD/content/enterprise, rapid dev | Async I/O APIs, microservices, ML serving |
| Learning curve | Steeper start, pays off long-term | Easy start, complexity added by you |

**One-liners to remember**
- "Django = Laravel; FastAPI ≈ Lumen/Slim + type safety."
- "Async helps I/O concurrency, not CPU work."
- "Never block the event loop in an `async def` handler."
- "Django gives you a kitchen; FastAPI gives you a toolbox."
- "Pick on workload + team, not benchmarks."
- "They can coexist: Django core + FastAPI side-services."
- "Pydantic v2: `model_dump()` and `ConfigDict`, never `.dict()` / v1 `Config`-only."
