# 40 — FastAPI: Intro & Setup

> Week 4 · DRF and FastAPI

## What you will learn

- What FastAPI actually is, and the four selling points that made it explode in popularity: **async-first**, **type-hint driven**, **automatic validation + OpenAPI docs**, and **high performance**.
- How FastAPI sits on top of **Starlette** (the web/ASGI toolkit) and **Pydantic v2** (the data/validation layer) — and why that matters.
- Installing `fastapi` and `uvicorn`, and what each piece does.
- Writing the **minimal app**: a `FastAPI` instance, a *path operation decorator*, and returning a `dict` that becomes JSON automatically.
- Running the dev server with `uvicorn --reload`, and the tight edit-save-refresh dev loop.
- The two free, auto-generated interactive docs: **Swagger UI at `/docs`** and **ReDoc at `/redoc`**, plus the raw **OpenAPI schema at `/openapi.json`**.
- How Python **type hints drive everything** — parsing, validation, serialization, and the docs — with zero extra config.
- A clear **mental model vs Django/DRF**: FastAPI is a *micro* API framework (no ORM, no admin, no templates included), more like "Laravel-but-bring-your-own-batteries."
- **When to pick FastAPI** over Django/DRF (and when not to).

## PHP/Laravel bridge

If Django is the "Laravel" of Python — a full-stack, batteries-included framework with an ORM, migrations, an admin panel, templating, and auth all in the box — then **FastAPI is closer to Laravel's bare routing core (or a micro-framework like Lumen/Slim) focused purely on JSON APIs**. You bring your own database layer (SQLAlchemy or an async ORM), your own auth, your own background jobs. What FastAPI gives you for free, that Laravel makes you wire up by hand, is **request validation driven entirely by type declarations** (think Laravel Form Requests, but inferred from your function signature) and **auto-generated, always-in-sync interactive API documentation** (think a Swagger/OpenAPI spec you never have to maintain). One key new concept with no direct PHP-FPM analog: FastAPI is **async-first** and runs on an **ASGI** server (Uvicorn), more like Node.js's event loop than the traditional one-request-per-process PHP model.

## Core concept — What FastAPI is

FastAPI is a modern, high-performance Python web framework for building **HTTP APIs** (overwhelmingly JSON/REST, also great for things like WebSockets). It was first released in 2018 by Sebastián Ramírez and has become the default choice for new Python API services, especially in data/ML-adjacent teams.

Its entire design philosophy is built on one idea: **use standard Python type hints to declare what your API expects and returns, and let the framework do the rest.** From those type hints, FastAPI derives:

1. **Request parsing** — pulling values out of the path, query string, headers, and JSON body.
2. **Validation** — rejecting bad input with a clear, structured `422` error before your code runs.
3. **Serialization** — converting your return values into JSON.
4. **Documentation** — a complete OpenAPI 3.1 schema and two interactive doc UIs, generated automatically and always in sync with the code.

FastAPI is not one monolithic library. It is a thin, brilliant layer that stitches together two excellent existing libraries:

| Layer | Library | Responsibility |
| --- | --- | --- |
| Web / HTTP / routing / ASGI | **Starlette** | The actual ASGI app, routing, requests/responses, middleware, WebSockets, background tasks |
| Data / validation / serialization | **Pydantic v2** | Defining data shapes, validating input, parsing/coercing types, dumping to JSON |
| The glue + docs + DI | **FastAPI** | Path operations, dependency injection, turning type hints into validation + OpenAPI |

Understanding this split is genuinely useful: when you hit a question about request objects or middleware, the answer is often "that's a Starlette thing"; when it's about a model or validation, "that's a Pydantic thing."

### The four selling points

**1. Async-first (ASGI).** Traditional Python web (WSGI) and traditional PHP-FPM handle one request per worker process/thread, blocking while waiting on the database or an external API. FastAPI runs on **ASGI** (Asynchronous Server Gateway Interface) via Uvicorn. You can declare path functions with `async def`, and while one request is `await`-ing I/O (a DB query, an HTTP call), the same worker can serve other requests. This is the same concurrency model as Node.js. (You can also use plain `def` — FastAPI runs those in a threadpool so you don't block the loop. More on this in a later lesson.)

**2. Type-hint driven.** Your function signature *is* your API contract. No separate validation classes required for the basics.

**3. Automatic validation + OpenAPI docs.** Bad input is rejected with a structured error; good input arrives already parsed into the right Python types; and you get Swagger UI + ReDoc for free.

**4. High performance.** Because it's built on Starlette and Uvicorn (which uses `uvloop` and `httptools` under the hood), FastAPI is among the fastest Python frameworks — performance comparable to Node.js and Go for I/O-bound workloads. The validation engine, Pydantic v2, has its core written in Rust, making it dramatically faster than Pydantic v1.

## Core concept — Installation

Make a project folder and a virtual environment first (always work inside a venv in Python — it's like a per-project `vendor/` that also isolates the Python interpreter and tools).

```bash
mkdir fastapi-intro && cd fastapi-intro
python3.12 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
```

Now install. You need two packages: **`fastapi`** (the framework) and **`uvicorn`** (the ASGI server that actually runs it — FastAPI does not include a server, just like Laravel doesn't include PHP-FPM or Nginx).

```bash
pip install "fastapi[standard]"
```

The `[standard]` extra is the modern, recommended install. It pulls in **Uvicorn**, the FastAPI **CLI**, the `httpx`-based test client, Jinja2, and other commonly-needed pieces. If you want a leaner install you can do `pip install fastapi uvicorn[standard]` instead, but `fastapi[standard]` is the documented happy path today.

Check it worked:

```bash
pip show fastapi | grep -i version
# => Version: 0.115.x   (or newer)

python -c "import fastapi; print(fastapi.__version__)"
# => 0.115.x
```

Freeze your dependencies (the Python equivalent of committing `composer.lock`):

```bash
pip freeze > requirements.txt
```

## Core concept — The minimal app

Create a file named `main.py`:

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"message": "Hello, FastAPI!"}
```

That is a complete, runnable web API. Three things are happening:

1. **`app = FastAPI()`** — create the application instance. This single object is your whole app: routes, config, docs metadata, middleware all attach to it. (Roughly analogous to Laravel's `Application` / the router you register routes on.)

2. **`@app.get("/")`** — a **path operation decorator**. `app.get` registers a handler for `GET` requests to the path `/`. There's a decorator per HTTP method: `@app.get`, `@app.post`, `@app.put`, `@app.patch`, `@app.delete`, etc. This is FastAPI's equivalent of `Route::get('/', ...)` in Laravel. The function it decorates is called the **path operation function**.

3. **`return {"message": "..."}`** — you return a plain Python `dict`. FastAPI automatically serializes it to JSON and sets `Content-Type: application/json` and a `200` status. You can also return lists, Pydantic models, strings, ints — FastAPI knows how to turn them into a JSON response. No `json_encode`, no `response()->json()` boilerplate.

### Returning more than a flat dict

Anything JSON-serializable works, and you can mix path parameters in via type-hinted function arguments (we'll go deep on this in the next lesson, but here's a taste so the "type hints drive everything" idea is concrete):

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"message": "Hello, FastAPI!"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    # item_id comes from the URL PATH and is validated/coerced to int.
    # q comes from the QUERY STRING (?q=...) and is optional (default None).
    return {"item_id": item_id, "q": q}
```

Notice what the type hints already bought you, for free:

- `GET /items/42` returns `{"item_id": 42, "q": null}` — and `item_id` is an actual Python `int`, not the string `"42"`.
- `GET /items/42?q=hello` returns `{"item_id": 42, "q": "hello"}`.
- `GET /items/abc` returns a **`422 Unprocessable Entity`** with a precise error message, because `abc` is not a valid `int` — you wrote *zero* validation code.

```json
// Response body for GET /items/abc  (HTTP 422)
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "item_id"],
      "msg": "Input should be a valid integer, unable to parse string as an integer",
      "input": "abc"
    }
  ]
}
```

That is the core magic of FastAPI in one screen: **the type annotation `item_id: int` is simultaneously the parser, the validator, the docs, and your editor's autocomplete.**

## Core concept — Running it with Uvicorn

FastAPI ships a convenience CLI (from the `[standard]` install). The simplest way to run in development:

```bash
fastapi dev main.py
```

This starts Uvicorn with auto-reload on, finds the `app` object in `main.py`, and prints the URLs. Equivalent to calling Uvicorn directly, which is the form you'll see everywhere and should understand:

```bash
uvicorn main:app --reload
```

Breaking down `uvicorn main:app --reload`:

- **`uvicorn`** — the ASGI server program.
- **`main`** — the Python *module* (the file `main.py`, without the `.py`).
- **`app`** — the FastAPI instance *inside* that module (the variable named `app`). The colon separates module from attribute: `module:attribute`.
- **`--reload`** — watch source files and restart on change. **Development only** — never use `--reload` in production (it's slower and spawns a file-watcher process).

You'll see something like:

```text
INFO:     Will watch for changes in these directories: ['/.../fastapi-intro']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [12345] using WatchFiles
INFO:     Started server process [12347]
INFO:     Application startup complete.
```

Now hit it:

```bash
curl http://127.0.0.1:8000/
# => {"message":"Hello, FastAPI!"}

curl http://127.0.0.1:8000/items/42?q=test
# => {"item_id":42,"q":"test"}
```

### Common run options

```bash
# Change host/port (e.g. to expose on your LAN)
uvicorn main:app --reload --host 0.0.0.0 --port 8080

# Production-style (NO reload; multiple worker processes)
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

In real production you typically run Uvicorn workers behind a process manager (often Gunicorn with the Uvicorn worker class, or a container orchestrator) and a reverse proxy like Nginx — conceptually the same "Nginx in front, app server behind" topology you know from PHP-FPM.

### The dev workflow

1. `uvicorn main:app --reload` (or `fastapi dev main.py`) once.
2. Edit `main.py`, hit save.
3. Uvicorn detects the change and restarts the app automatically (watch the terminal log).
4. Refresh `curl`, your HTTP client, or — best of all — the `/docs` page.

That's the loop. No build step, no compile.

## Core concept — The auto-generated interactive docs

This is the feature that makes people fall in love with FastAPI. **With zero extra code**, the framework builds a full **OpenAPI 3.1** description of your API and serves two interactive UIs from it. Start the server and open a browser:

| URL | What it is |
| --- | --- |
| `http://127.0.0.1:8000/docs` | **Swagger UI** — interactive: expand any endpoint, fill in params, click **"Try it out"**, and it fires a real request and shows the response. |
| `http://127.0.0.1:8000/redoc` | **ReDoc** — a clean, three-column, read-only reference document. Great for sharing with API consumers. |
| `http://127.0.0.1:8000/openapi.json` | The raw **OpenAPI schema** (JSON). This is the source of truth both UIs render from; you can feed it to client-code generators, Postman, etc. |

Why this matters so much: in Laravel/DRF you typically maintain API docs separately (or bolt on a package and annotate everything). They drift out of date the moment someone forgets to update an annotation. In FastAPI the docs are **derived from the same type hints that run your validation**, so they literally cannot drift — if the code says `item_id: int`, the docs say `item_id: integer`, guaranteed.

You can customize the metadata that shows up in the docs via the `FastAPI()` constructor:

```python
# main.py
from fastapi import FastAPI

app = FastAPI(
    title="Bookstore API",
    description="A tiny example API for the course.",
    version="1.0.0",
    # You can move or disable the docs endpoints if you want:
    # docs_url="/swagger",   # default "/docs"
    # redoc_url=None,        # set to None to disable ReDoc entirely
    # openapi_url=None,      # set to None to disable the schema (and thus both UIs)
)


@app.get("/")
def read_root():
    return {"message": "Hello, FastAPI!"}
```

A common production pattern is to **disable the docs in production** (set `docs_url=None, redoc_url=None`) or protect them behind auth, while leaving them on in development.

## Core concept — How type hints drive everything

It's worth slowing down on the central idea, because it's what's most different from PHP. In FastAPI, a function parameter's **type annotation** + **where FastAPI infers it comes from** = a fully specified, validated input.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


# A Pydantic model declares the SHAPE of a JSON request body.
# (This is FastAPI's equivalent of a Laravel Form Request /
#  a DRF serializer — but it's just a typed Python class.)
class Item(BaseModel):
    name: str
    price: float
    in_stock: bool = True          # default value => optional field


@app.post("/items")
def create_item(item: Item):
    # `item` arrives already PARSED and VALIDATED into an Item instance.
    # item.price is a real float; if the client sent a non-numeric
    # price, the request was rejected with 422 before we ever ran.
    total = item.price * 1.2
    return {"name": item.name, "price_with_tax": round(total, 2)}
```

From those few lines, FastAPI infers all of this with no further config:

- `item: Item` is a Pydantic model, so FastAPI knows the input is the **JSON request body**.
- The body must have a string `name` and a numeric `price`; `in_stock` is optional and defaults to `True`.
- Sending `{"name": "Pen", "price": 1.5}` works and returns `{"name": "Pen", "price_with_tax": 1.8}`.
- Sending `{"name": "Pen"}` (missing `price`) returns a `422` listing `price` as required.
- The `/docs` page now shows a documented request schema for `Item`, with an editable example body and a "Try it out" button.

```bash
curl -X POST http://127.0.0.1:8000/items \
  -H "Content-Type: application/json" \
  -d '{"name": "Pen", "price": 1.5}'
# => {"name":"Pen","price_with_tax":1.8}
```

The takeaway: **you describe data with ordinary Python types, and validation + parsing + serialization + docs all fall out of that single description.** Pydantic models get their own dedicated lessons later — here just absorb the principle.

## Core concept — Mental model vs Django/DRF (and Laravel)

FastAPI is a **micro API framework**. It deliberately does **not** ship the things a full-stack framework does. This is the single most important expectation to set:

| Feature | Django + DRF | FastAPI | Laravel (for reference) |
| --- | --- | --- | --- |
| HTTP routing | Yes | Yes | Yes |
| Request validation | DRF serializers | Type hints + Pydantic (built in) | Form Requests |
| Auto API docs | Add-on (e.g. drf-spectacular) | **Built in (OpenAPI + Swagger + ReDoc)** | Add-on |
| ORM | Yes (Django ORM) | **No** — bring SQLAlchemy / SQLModel / an async ORM | Yes (Eloquent) |
| Migrations | Yes (built in) | **No** — bring Alembic | Yes |
| Admin panel | Yes (famous) | **No** | No (Nova/Filament add-on) |
| Templating / server-rendered HTML | Yes | Minimal (via Starlette/Jinja2, not its focus) | Yes (Blade) |
| Built-in auth/users | Yes | **No** — bring your own / libraries | Yes |
| Async-first | Partial (improving) | **Yes, native** | No (PHP-FPM model) |
| Best at | Full web apps + APIs, content sites, admin-heavy | **JSON APIs, microservices, ML model serving** | Full web apps + APIs |

So the mental model is: **Django/DRF ≈ Laravel (full stack); FastAPI ≈ a focused, modern, async API core where you assemble the rest.** That assembly cost is real — you'll pick and integrate a DB layer, migrations, and auth yourself — but you gain speed, async, automatic docs, and a small, explicit surface area.

### When to pick FastAPI

**Reach for FastAPI when:**

- You're building a **JSON/REST API or microservice** (no server-rendered HTML pages).
- You want **async** for I/O-heavy work — calling other services, streaming, lots of concurrent connections.
- You want **first-class, automatic OpenAPI docs** and validation with minimal ceremony.
- You're **serving ML models** or doing data/AI work (this is FastAPI's stronghold).
- You value a **small, explicit, composable** stack and don't need an admin/ORM out of the box.

**Prefer Django/DRF when:**

- You need a **full web application**: server-rendered pages, a built-in **admin**, an ORM + migrations, sessions, and auth all integrated from day one.
- The team wants **batteries included** and conventions over assembling pieces.
- The app is **CRUD- and content-heavy** where Django's admin and ORM save enormous time.

Neither is "better" — they optimize for different jobs. For your target backend/REST-API role, expect to know both: **DRF for big integrated apps, FastAPI for lean modern services.**

## Common mistakes & gotchas

- **Forgetting to install a server.** `pip install fastapi` alone gives you no way to run anything. Install `fastapi[standard]` (includes Uvicorn) or add `uvicorn[standard]` explicitly.
- **Wrong `uvicorn` target string.** `uvicorn main:app` means *module* `main`, *variable* `app`. If your file is `app.py` and the instance is `api`, it's `uvicorn app:api`. A mismatch gives `Error loading ASGI app. Could not import module "main".` or `Attribute "app" not found`.
- **Running `uvicorn` from the wrong directory.** The module path is resolved relative to your current working directory / `PYTHONPATH`. Be in the folder that contains `main.py`.
- **Using `--reload` (or many `--workers`) in production.** `--reload` is dev-only. `--workers N` and `--reload` don't combine; pick reload for dev, workers for prod.
- **Forgetting `async`/`await` semantics.** You can use plain `def` and FastAPI handles it. But if you write `async def` and then call a *blocking* library (e.g. a synchronous DB driver) without awaiting, you block the event loop and kill your concurrency. Don't mix blocking calls into `async def` handlers. (Covered in depth later.)
- **Expecting an ORM/admin/migrations.** They aren't there. Plan to add SQLAlchemy/SQLModel + Alembic yourself.
- **Editing files but not seeing changes** — you forgot `--reload`, or the reloader didn't pick up a new file. Watch the terminal for the "Reloading" log line; restart manually if unsure.
- **Trailing-slash surprises.** `@app.get("/items")` and `@app.get("/items/")` are different routes. Be consistent.
- **Confusing FastAPI version with feature set.** FastAPI requires Pydantic v2 for modern behavior; if you see `.dict()`/old `class Config`, that's outdated v1 code — use `model_dump()` and `model_config = ConfigDict(...)`.

## Best practices

- **Always work in a virtual environment** and pin dependencies (`requirements.txt` or a `pyproject.toml`/`uv`/Poetry lockfile).
- Install with **`fastapi[standard]`** for the smooth, documented developer experience.
- Use **`fastapi dev main.py`** or **`uvicorn main:app --reload`** for development; reserve `--workers` (no reload) for production, behind a reverse proxy.
- **Name the instance `app` in `main.py`** — it's the near-universal convention, so tools and tutorials "just work."
- **Set `title`, `description`, and `version`** on the `FastAPI()` instance — it makes your auto-docs professional for free.
- **Lean on type hints everywhere** — they're not optional decoration here; they *are* your validation and docs.
- **Use Pydantic v2 idioms** (`model_dump()`, `ConfigDict`) from day one to avoid relearning.
- In production, **disable or protect `/docs` and `/redoc`** if the API isn't public.
- Keep `main.py` tiny as the app grows — split routes into **routers** (`APIRouter`) later, much like grouping routes in Laravel.

## Interview Q&A

**Q: What is FastAPI, and what are its main selling points?**
**A:** FastAPI is a modern, high-performance Python framework for building HTTP/JSON APIs. Its selling points are: it's **async-first** (runs on ASGI for high concurrency on I/O-bound work), **type-hint driven** (your function signatures define the API contract), provides **automatic request validation and OpenAPI documentation** (Swagger UI and ReDoc generated from those type hints), and offers **high performance** thanks to being built on Starlette and Pydantic v2 (whose core is written in Rust).

**Q: What is FastAPI built on top of, and what does each part do?**
**A:** Two libraries. **Starlette** provides the ASGI application, routing, request/response objects, middleware, WebSockets, and background tasks — the web layer. **Pydantic v2** provides data modeling, validation, type coercion, and JSON serialization — the data layer. FastAPI is the glue that turns Python type hints into validation plus an OpenAPI schema, and adds dependency injection.

**Q: What's the difference between WSGI and ASGI, and why does FastAPI use ASGI?**
**A:** WSGI is the older synchronous Python web server interface — one request blocks a worker until it's done. ASGI is the asynchronous successor; it supports `async`/`await`, so a single worker can handle many concurrent requests by switching to other work while one is waiting on I/O. It also supports long-lived connections like WebSockets. FastAPI uses ASGI (via Uvicorn) to enable that high-concurrency, async-first model.

**Q: Why do you install both `fastapi` and `uvicorn`?**
**A:** `fastapi` is the framework that defines your routes and logic; it doesn't include a web server. `uvicorn` is the ASGI server that actually accepts HTTP connections and runs the app. It's the same separation as Laravel (the framework) needing PHP-FPM/a server to serve it. `pip install "fastapi[standard]"` bundles Uvicorn and the CLI for convenience.

**Q: Walk me through the minimal FastAPI app.**
**A:** Import `FastAPI`, create `app = FastAPI()`, decorate a function with a path operation decorator like `@app.get("/")`, and return a `dict`. FastAPI serializes the dict to JSON automatically with a `200` status. You run it with `uvicorn main:app --reload`, where `main` is the module and `app` is the instance.

**Q: What does `uvicorn main:app --reload` mean exactly?**
**A:** `uvicorn` is the server; `main` is the Python module (`main.py`); `app` is the `FastAPI()` instance variable inside it (the `module:attribute` syntax); `--reload` watches files and restarts on change for development. You omit `--reload` in production and instead use multiple `--workers` behind a reverse proxy.

**Q: How does FastAPI generate its API docs, and where are they?**
**A:** It builds an OpenAPI 3.1 schema automatically from your path operations and type hints, served at `/openapi.json`. From that schema it serves two interactive UIs: **Swagger UI at `/docs`** (interactive — you can fire real requests) and **ReDoc at `/redoc`** (a clean read-only reference). Because the docs come from the same type hints that drive validation, they never drift from the code.

**Q: How do type hints drive behavior in FastAPI?**
**A:** A parameter's type annotation tells FastAPI how to parse, coerce, and validate the input, and FastAPI infers the source (path, query, body, etc.) from how it's declared. For example, `item_id: int` parses the path segment into an int and returns a `422` if it isn't one; a Pydantic-model parameter is read from and validated against the JSON body. The same hints generate the OpenAPI schema and improve editor autocompletion.

**Q: How is FastAPI different from Django/DRF?**
**A:** Django/DRF is full-stack and batteries-included — ORM, migrations, admin, templates, auth all built in (the "Laravel of Python"). FastAPI is a micro API framework: it gives you routing, validation, and auto-docs, but no ORM, admin, migrations, or templating — you bring SQLAlchemy/SQLModel, Alembic, and an auth solution yourself. FastAPI is async-first and excels at lean JSON APIs, microservices, and ML serving; Django/DRF excels at large integrated web apps.

**Q: When would you choose FastAPI over Django/DRF?**
**A:** Choose FastAPI for JSON APIs and microservices, async-heavy I/O workloads, ML model serving, and cases where automatic OpenAPI docs and a small explicit stack matter. Choose Django/DRF when you need a full web app with a built-in admin, ORM, migrations, sessions, and server-rendered pages, or when the team wants conventions over assembling components.

**Q: Can FastAPI handle synchronous code, or must everything be `async`?**
**A:** Both work. You can write path functions with plain `def` and FastAPI runs them in a threadpool so they don't block the event loop; or write `async def` for native async I/O. The pitfall is calling a *blocking* library inside an `async def` handler without awaiting — that blocks the loop and destroys concurrency. Match your handler style to your I/O.

**Q: Why is FastAPI considered fast?**
**A:** It's built on Starlette and runs on Uvicorn (using `uvloop` and `httptools`), giving Node.js/Go-class throughput for I/O-bound work. Its validation/serialization engine, Pydantic v2, has a Rust core (`pydantic-core`), so the data-handling overhead that often dominates request time is minimal.

## Practice exercises

1. **Bootstrap a project.** Create a folder, a Python 3.12 virtual environment, activate it, install `fastapi[standard]`, and confirm the installed FastAPI version from the command line.
2. **Minimal app + run.** Write a `main.py` with a root `GET /` route returning `{"status": "ok"}`. Run it with reload and confirm with `curl` that you get the JSON back.
3. **Type-hint validation.** Add `GET /square/{n}` that takes an integer `n` and returns `{"n": n, "square": n*n}`. Verify that `/square/5` works and that `/square/hello` returns a `422` — *without writing any validation code*.
4. **Explore the docs.** Open `/docs` and `/redoc` in a browser. Use "Try it out" on your `/square/{n}` endpoint. Then open `/openapi.json` and find where your route and its integer parameter are described.
5. **Customize metadata.** Give your app a `title`, `description`, and `version`, then disable ReDoc (`redoc_url=None`) and move Swagger UI to `/swagger`. Confirm `/redoc` now 404s and `/swagger` works.
6. **Body validation (stretch).** Add a Pydantic model `Person` with `name: str` and `age: int`, and a `POST /people` route that accepts it and returns a greeting. Confirm that a missing or non-integer `age` yields a `422`.

<details>
<summary>Solutions</summary>

**1. Bootstrap**

```bash
mkdir fastapi-practice && cd fastapi-practice
python3.12 -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install "fastapi[standard]"
python -c "import fastapi; print(fastapi.__version__)"
# => 0.115.x  (or newer)
```

**2. Minimal app + run**

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def root():
    return {"status": "ok"}
```

```bash
uvicorn main:app --reload
# in another terminal:
curl http://127.0.0.1:8000/
# => {"status":"ok"}
```

**3. Type-hint validation**

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/square/{n}")
def square(n: int):
    return {"n": n, "square": n * n}
```

```bash
curl http://127.0.0.1:8000/square/5
# => {"n":5,"square":25}

curl -i http://127.0.0.1:8000/square/hello
# => HTTP/1.1 422 Unprocessable Entity
#    {"detail":[{"type":"int_parsing","loc":["path","n"], ...}]}
```

**4. Explore the docs** — Start the server and visit:
- `http://127.0.0.1:8000/docs` → expand `GET /square/{n}` → "Try it out" → enter `5` → Execute.
- `http://127.0.0.1:8000/redoc` → read-only reference.
- `http://127.0.0.1:8000/openapi.json` → search for `"/square/{n}"`; under its `parameters` you'll see `"in": "path"`, `"name": "n"`, and `"schema": {"type": "integer"}`.

**5. Customize metadata**

```python
# main.py
from fastapi import FastAPI

app = FastAPI(
    title="Practice API",
    description="Lesson 40 practice.",
    version="0.1.0",
    redoc_url=None,        # disable ReDoc
    docs_url="/swagger",   # move Swagger UI
)


@app.get("/square/{n}")
def square(n: int):
    return {"n": n, "square": n * n}
```

Now `http://127.0.0.1:8000/redoc` returns 404, and the interactive UI lives at `http://127.0.0.1:8000/swagger`.

**6. Body validation**

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Person(BaseModel):
    name: str
    age: int


@app.post("/people")
def greet(person: Person):
    return {"greeting": f"Hello {person.name}, age {person.age}"}
```

```bash
curl -X POST http://127.0.0.1:8000/people \
  -H "Content-Type: application/json" \
  -d '{"name": "Ada", "age": 30}'
# => {"greeting":"Hello Ada, age 30"}

curl -i -X POST http://127.0.0.1:8000/people \
  -H "Content-Type: application/json" \
  -d '{"name": "Ada"}'
# => HTTP/1.1 422 Unprocessable Entity   (age is required)
```

</details>

## Cheat sheet

```bash
# --- Setup ---
python3.12 -m venv .venv && source .venv/bin/activate
pip install "fastapi[standard]"          # framework + uvicorn + CLI
pip freeze > requirements.txt

# --- Run (dev) ---
fastapi dev main.py                       # convenience CLI, reload on
uvicorn main:app --reload                 # explicit; module:instance

# --- Run (prod-ish) ---
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4   # NO --reload
```

```python
# --- Minimal app ---
from fastapi import FastAPI
app = FastAPI(title="My API", version="1.0.0")

@app.get("/")
def root():
    return {"message": "hi"}          # dict -> JSON automatically
```

| Thing | Value |
| --- | --- |
| Install | `pip install "fastapi[standard]"` |
| Run target | `uvicorn <module>:<instance> --reload` |
| Built on | Starlette (web) + Pydantic v2 (data) |
| Server | Uvicorn (ASGI) |
| Swagger UI | `GET /docs` |
| ReDoc | `GET /redoc` |
| OpenAPI schema | `GET /openapi.json` |
| Method decorators | `@app.get/post/put/patch/delete(path)` |
| Path param | `def f(id: int)` → from URL, validated |
| Query param | `def f(q: str \| None = None)` → from `?q=` |
| Body | `def f(item: ItemModel)` → JSON body (Pydantic) |
| Bad input | auto `422` with structured `detail` |
| Disable docs | `FastAPI(docs_url=None, redoc_url=None)` |

**One-line mental model:** *FastAPI = Starlette + Pydantic, where your Python type hints become the parser, validator, serializer, and docs — a lean async API core (bring your own ORM/auth/migrations), unlike full-stack Django/DRF.*
