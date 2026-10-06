# 41 — FastAPI: Path, Query & Body Params

> Week 4 · DRF and FastAPI

## What you will learn

- How FastAPI reads values out of the **URL path**, the **query string**, and the **request body** — purely from your function signature and type hints.
- Why a type hint like `item_id: int` gives you free **conversion and validation** (and a clean 422 error when the client sends garbage).
- How **defaults** turn a parameter into an optional query param, and how `None` / `Optional` works.
- How to declare a request body with a **Pydantic v2 model** and get automatic JSON parsing + validation.
- How FastAPI **disambiguates** path vs query vs body when all three appear in one function.
- Adding constraints and metadata with `Path()`, `Query()`, and `Body()` (`gt`, `le`, `min_length`, `max_length`, `pattern`, `examples`).
- Multiple body params and the `embed` trick.
- Shaping output with `response_model`, setting `status_code`, and raising errors with `HTTPException`.
- A quick look at **header** and **cookie** params.

## PHP/Laravel bridge

In Laravel you pull request data from three places by hand: route parameters via `Route::get('/items/{id}', ...)` and the `$id` argument (or `$request->route('id')`), query/body values via `$request->input('q')` or `$request->query('q')`, and you validate separately with `$request->validate([...])` or a FormRequest class. FastAPI collapses all of that into **one function signature**: where a parameter appears (path, query, or body) is inferred from your route string and its type, and validation is the type hint itself. Think of it as Laravel's route model binding + FormRequest validation + automatic JSON deserialization, all driven by Python type hints and Pydantic. The payoff is the same contract that powers Laravel's `$request` is now also your auto-generated OpenAPI docs at `/docs`.

Throughout this lesson, run the dev server with:

```bash
pip install "fastapi[standard]"   # pulls in uvicorn + the CLI
fastapi dev main.py               # hot-reload server on http://127.0.0.1:8000
# Interactive docs: http://127.0.0.1:8000/docs
```

## Core concept — Path parameters

A **path parameter** is a piece of the URL itself, declared with `{braces}` in the route. Bind it by giving the function a parameter with the **same name**.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int):
    # item_id arrives from the URL already converted to an int
    return {"item_id": item_id, "type": type(item_id).__name__}

# GET /items/42  =>  {"item_id": 42, "type": "int"}
# GET /items/abc => 422 Unprocessable Entity (not an int)
```

The magic is the type hint `item_id: int`. FastAPI:

1. **Parses** the raw string `"42"` from the URL.
2. **Converts** it to a real Python `int`.
3. **Validates** it — if conversion fails, the client never reaches your code; they get a structured `422` instead:

```json
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

> Why this matters: in Laravel you'd cast and validate `$id` manually. Here the hint *is* the validation, and it's documented automatically.

### Order matters with fixed paths

Routes are matched top to bottom. A fixed path like `/users/me` must be declared **before** the variable path `/users/{user_id}`, or `me` would be swallowed as a `user_id`.

```python
@app.get("/users/me")
def read_current_user():
    return {"user": "the current user"}

@app.get("/users/{user_id}")
def read_user(user_id: str):
    return {"user_id": user_id}

# GET /users/me  -> read_current_user  (because it's declared first)
# GET /users/123 -> read_user
```

### Enums for a fixed set of values

When a path segment should be one of a known set, use an `Enum`. FastAPI validates membership and documents the choices.

```python
from enum import Enum

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
def get_model(model_name: ModelName):
    # model_name is an enum member; compare against members or .value
    if model_name is ModelName.alexnet:
        return {"model_name": model_name, "message": "Deep Learning FTW!"}
    return {"model_name": model_name, "value": model_name.value}

# GET /models/resnet  => {"model_name": "resnet", "value": "resnet"}
# GET /models/foo     => 422 (not a valid enum value)
```

### Path containing a path (the `:path` converter)

If a parameter must capture slashes (e.g. a file path), use the `:path` converter:

```python
@app.get("/files/{file_path:path}")
def read_file(file_path: str):
    return {"file_path": file_path}

# GET /files/home/user/notes.txt => {"file_path": "home/user/notes.txt"}
```

## Core concept — Query parameters

Any function parameter that is **not** part of the path is interpreted as a **query parameter** (the `?key=value` part of the URL) — *unless* it's a Pydantic model or otherwise marked as a body, which we'll cover below.

The rule for required vs optional is simple and it's the same rule as normal Python functions:

| Declaration | Behaviour |
|---|---|
| `q: str` (no default) | **Required** query param; missing => 422 |
| `q: str = "hello"` | **Optional**, defaults to `"hello"` |
| `q: str \| None = None` | **Optional**, defaults to `None` (no value provided) |

```python
@app.get("/products/")
def list_products(skip: int = 0, limit: int = 10):
    # /products/?skip=20&limit=5  => skip=20, limit=5
    # /products/                  => skip=0,  limit=10  (defaults)
    return {"skip": skip, "limit": limit}
```

### Optional vs required, and `None`

```python
@app.get("/search/")
def search(q: str | None = None, required_tag: str = ...):
    # q is optional (defaults to None)
    # required_tag has no real default (...) so it's REQUIRED
    result = {"required_tag": required_tag}
    if q:
        result["q"] = q
    return result

# /search/?required_tag=books             => {"required_tag": "books"}
# /search/?required_tag=books&q=python    => {"required_tag": "books", "q": "python"}
# /search/                                 => 422 (required_tag missing)
```

`...` (the literal `Ellipsis` object) means "no default — this is required." It's mostly useful inside `Query(...)`/`Path(...)`, shown later; for plain params, just omit the default to make it required.

### Booleans get parsed loosely

Query strings are always text, but FastAPI converts to your declared type. Booleans accept several truthy spellings:

```python
@app.get("/flags/")
def flags(active: bool = False):
    return {"active": active}

# ?active=true  ?active=1  ?active=on  ?active=yes  => active is True
# ?active=false ?active=0  ?active=off          => active is False
```

### Lists in the query string

To accept the same key multiple times (`?tag=a&tag=b`), declare a `list` and use `Query` so FastAPI knows it's a multi-value query param, not a body:

```python
from typing import Annotated
from fastapi import Query

@app.get("/posts/")
def posts(tags: Annotated[list[str], Query()] = []):
    return {"tags": tags}

# /posts/?tags=python&tags=fastapi => {"tags": ["python", "fastapi"]}
# /posts/                          => {"tags": []}
```

## Core concept — Request body with Pydantic models

For `POST`, `PUT`, and `PATCH`, the data usually arrives as a **JSON body**. Declare a parameter typed as a **Pydantic model** and FastAPI will read the body, parse the JSON, validate it against the model, and hand you a fully-typed object.

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str
    description: str | None = None          # optional field
    price: float = Field(gt=0)              # must be > 0
    tax: float | None = None

@app.post("/items/")
def create_item(item: Item):
    # item is a validated Item instance, not a dict
    total = item.price + (item.tax or 0)
    return {"name": item.name, "total_with_tax": total}
```

Send it:

```bash
curl -X POST http://127.0.0.1:8000/items/ \
  -H "Content-Type: application/json" \
  -d '{"name": "Keyboard", "price": 49.99, "tax": 4.5}'
# => {"name": "Keyboard", "total_with_tax": 54.49}

# Missing required "name" or a negative price => 422 with a precise error list.
```

What you get for free:

- The raw bytes are read and JSON-decoded.
- Each field is validated and coerced to its declared type (`"49.99"` -> `49.99`).
- Unknown vs missing fields are reported clearly.
- The model schema is published in the OpenAPI docs and the `/docs` "Try it out" form.

> Pydantic v2 note: convert a model to a dict with `item.model_dump()` (not the deprecated `.dict()`), and to JSON with `item.model_dump_json()`. Configure models with `model_config = ConfigDict(...)`, not an inner `class Config`.

## Core concept — Combining path + query + body

Here's the part that surprises newcomers: you can put all three in one signature, and FastAPI figures out where each value comes from using these rules:

1. If the parameter name **matches a `{placeholder}`** in the route path → it's a **path parameter**.
2. Else if it's a **singular/scalar type** (`int`, `str`, `float`, `bool`, ...) → it's a **query parameter**.
3. Else if it's a **Pydantic model** (or explicitly marked with `Body()`) → it comes from the **request body**.

```python
@app.put("/items/{item_id}")
def update_item(
    item_id: int,                  # in path  -> path param
    item: Item,                    # Pydantic model -> request body
    q: str | None = None,          # scalar with default -> query param
):
    result = {"item_id": item_id, **item.model_dump()}
    if q:
        result["q"] = q
    return result

# PUT /items/7?q=urgent  with JSON body {"name":"Mouse","price":19.99}
# => {"item_id": 7, "name": "Mouse", "description": null,
#     "price": 19.99, "tax": null, "q": "urgent"}
```

No ordering rules to memorize, no `$request->...` plumbing — the *types* do the routing.

## Core concept — `Path()`, `Query()`, `Body()` for metadata & constraints

The plain hints are great, but eventually you need **constraints** (a minimum, a max length, a regex) and **metadata** (title, description, examples for the docs). That's what `Path()`, `Query()`, and `Body()` provide. The modern, recommended way to attach them is with `Annotated`, so the type stays the type and the metadata rides alongside.

```python
from typing import Annotated
from fastapi import FastAPI, Path, Query

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(
    item_id: Annotated[int, Path(title="The item ID", ge=1)],
    q: Annotated[
        str | None,
        Query(
            min_length=3,
            max_length=50,
            pattern="^[a-z]+$",          # 'pattern' in Pydantic v2 (was 'regex')
            description="Lowercase search term",
            examples=["widget"],
        ),
    ] = None,
):
    out = {"item_id": item_id}
    if q:
        out["q"] = q
    return out

# GET /items/0           => 422 (item_id must be >= 1)
# GET /items/5?q=ab      => 422 (q too short, min_length=3)
# GET /items/5?q=Widget  => 422 (pattern requires lowercase)
# GET /items/5?q=widget  => {"item_id": 5, "q": "widget"}
```

### The numeric & string constraints you'll actually use

| Constraint | Meaning | Applies to |
|---|---|---|
| `gt`, `ge` | greater than / greater-or-equal | numbers |
| `lt`, `le` | less than / less-or-equal | numbers |
| `min_length`, `max_length` | string length bounds | strings |
| `pattern` | regex the value must match | strings |
| `multiple_of` | numeric multiple | numbers |

```python
@app.get("/products/")
def list_products(
    limit: Annotated[int, Query(gt=0, le=100)] = 10,   # 1..100
    page: Annotated[int, Query(ge=1)] = 1,             # >= 1
):
    return {"limit": limit, "page": page}

# /products/?limit=0   => 422 (must be > 0)
# /products/?limit=500 => 422 (must be <= 100)
```

### Why `Annotated`?

You may see the older style `q: str = Query(default=None, ...)`. It works, but `Annotated` is now the recommended form because:

- The default value lives in the normal place (`= None`), so the function is still a normal, callable Python function.
- The same `Annotated[...]` type can be reused / aliased across endpoints.
- It avoids the confusing "the default *is* a `Query` object" pattern.

```python
# Old (still valid):    q: str | None = Query(default=None, max_length=50)
# Modern (preferred):   q: Annotated[str | None, Query(max_length=50)] = None
```

A common reusable alias:

```python
CommonQ = Annotated[str | None, Query(max_length=50, description="Search term")]

@app.get("/a/")
def a(q: CommonQ = None): return {"q": q}

@app.get("/b/")
def b(q: CommonQ = None): return {"q": q}
```

### Required with `Query`

Omit the default (or pass `...`) to keep a `Query`/`Path` param required:

```python
@app.get("/strict/")
def strict(q: Annotated[str, Query(min_length=3)]):   # no default => required
    return {"q": q}
```

## Core concept — Multiple body params & `embed`

When you declare **two** Pydantic models, FastAPI expects a JSON object with **one key per model**, using the parameter names:

```python
class Item(BaseModel):
    name: str
    price: float

class User(BaseModel):
    username: str
    full_name: str | None = None

@app.put("/orders/{order_id}")
def update_order(order_id: int, item: Item, user: User):
    return {"order_id": order_id, "item": item, "user": user}
```

Expected body:

```json
{
  "item": {"name": "Pen", "price": 1.5},
  "user": {"username": "neo", "full_name": "Thomas Anderson"}
}
```

### Singular values in the body with `Body()`

A scalar would normally be a query param. To force it into the body alongside models, mark it with `Body()`:

```python
from fastapi import Body

@app.put("/orders/{order_id}")
def update_order(
    order_id: int,
    item: Item,
    importance: Annotated[int, Body(gt=0)],   # now read from the JSON body
):
    return {"order_id": order_id, "item": item, "importance": importance}
```

Body now looks like:

```json
{ "item": {"name": "Pen", "price": 1.5}, "importance": 3 }
```

### `embed` for a single model

With a single model param, FastAPI expects the model's fields at the **top level** of the body. If you instead want it nested under the parameter name, use `embed=True`:

```python
@app.put("/items/{item_id}")
def update_item(item_id: int, item: Annotated[Item, Body(embed=True)]):
    return {"item_id": item_id, "item": item}

# WITHOUT embed, body is:  {"name": "Pen", "price": 1.5}
# WITH embed=True, body is: {"item": {"name": "Pen", "price": 1.5}}
```

## Core concept — `response_model`, `status_code`, and `HTTPException`

### Shaping the response with `response_model`

`response_model` declares the schema of what you return. FastAPI **filters** the output to that model — extra fields are dropped — which is perfect for hiding sensitive data like passwords.

```python
from pydantic import BaseModel, EmailStr

class UserIn(BaseModel):
    username: str
    password: str            # accepted as input
    email: EmailStr

class UserOut(BaseModel):
    username: str
    email: EmailStr          # NOTE: no password

@app.post("/users/", response_model=UserOut)
def create_user(user: UserIn):
    # We return the full object, but FastAPI strips it down to UserOut.
    return user

# POST {"username":"neo","password":"secret","email":"neo@example.com"}
# => {"username":"neo","email":"neo@example.com"}   (password never leaks)
```

Handy extras:

- `response_model_exclude_none=True` — omit fields whose value is `None`.
- `response_model_exclude_unset=True` — omit fields the caller never set (great for `PATCH`).

```python
@app.get("/items/{item_id}", response_model=Item, response_model_exclude_none=True)
def get_item(item_id: int):
    return Item(name="Pen", price=1.5)   # description/tax are None -> omitted
```

### Setting the status code

Default success status is `200`. For creation, return `201`:

```python
from fastapi import status

@app.post("/users/", response_model=UserOut, status_code=status.HTTP_201_CREATED)
def create_user(user: UserIn):
    return user
# Response status: 201 Created
```

Use the `status` constants instead of magic numbers — they read clearly and the docs label them.

### Returning errors with `HTTPException`

To return a non-2xx because of business logic (not found, forbidden, ...), **raise** `HTTPException`. FastAPI turns it into the right JSON response.

```python
from fastapi import HTTPException, status

FAKE_DB = {1: {"name": "Pen"}}

@app.get("/items/{item_id}")
def read_item(item_id: int):
    if item_id not in FAKE_DB:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found",
            headers={"X-Error": "ItemMissing"},   # optional custom headers
        )
    return FAKE_DB[item_id]

# GET /items/99 => 404 {"detail": "Item not found"}
```

`detail` can be a string or any JSON-serializable structure (dict/list) when you need richer error bodies.

## Core concept — Header and cookie params (brief)

Headers and cookies are declared just like query params, but with `Header()` / `Cookie()`. Header names are case-insensitive, and underscores in the parameter name are auto-converted to hyphens (`user_agent` -> `User-Agent`).

```python
from fastapi import Header, Cookie

@app.get("/whoami/")
def whoami(
    user_agent: Annotated[str | None, Header()] = None,   # reads "User-Agent"
    x_token: Annotated[str | None, Header()] = None,      # reads "X-Token"
    session_id: Annotated[str | None, Cookie()] = None,   # reads the "session_id" cookie
):
    return {"user_agent": user_agent, "x_token": x_token, "session_id": session_id}
```

To keep the literal header name (disable underscore→hyphen conversion), pass `Header(convert_underscores=False)`.

## Common mistakes & gotchas

- **Path param name mismatch.** The function arg must match the `{name}` in the route exactly. `@app.get("/items/{item_id}")` with `def f(id: int)` will treat `id` as a query param and 404/422.
- **Route ordering.** Declare fixed routes (`/users/me`) before variable ones (`/users/{user_id}`), or the variable route shadows them.
- **Forgetting `Query()` on list query params.** `tags: list[str] = []` without `Query()` makes FastAPI look for a body, not `?tags=a&tags=b`. Use `Annotated[list[str], Query()]`.
- **Using `regex=` instead of `pattern=`.** In Pydantic v2 / current FastAPI it's `pattern=`. `regex=` is removed.
- **Pydantic v1 habits.** `.dict()` → `model_dump()`, `.json()` → `model_dump_json()`, inner `class Config` → `model_config = ConfigDict(...)`. Don't mix v1 and v2 patterns.
- **Mutable default of `[]`.** It's fine for FastAPI query params (FastAPI handles it per-request), but never reuse a plain mutable default in ordinary Python helper functions — it's a classic shared-state bug.
- **`return`ing an error instead of raising.** Returning a dict like `{"error": "not found"}` still sends `200`. Use `raise HTTPException(...)` so the status code is correct.
- **Expecting `response_model` to validate input.** It validates/filters the *response*, not the request body. Input validation is the parameter's own type/model.
- **Body for GET requests.** Don't put request bodies on `GET`; many clients/proxies ignore them. Use query params for `GET`.
- **`status_code` vs `HTTPException`.** `status_code` on the decorator is the *success* code; error codes come from `HTTPException`.

## Best practices

- Prefer `Annotated[...]` for `Path`/`Query`/`Body`/`Header`/`Cookie` metadata — it's the current recommended style and keeps defaults in their natural place.
- Use **separate input and output models** (`UserIn` / `UserOut`) and a `response_model` so you never leak sensitive fields.
- Add `examples=` and `description=` to params and `Field(...)` to models — it makes `/docs` genuinely useful for consumers.
- Use the `status` constants (`status.HTTP_201_CREATED`) rather than raw numbers.
- Validate as early and declaratively as possible — push constraints into `Path()/Query()/Field()` rather than `if` checks in the body.
- Keep path params for **identity** (which resource) and query params for **modifiers** (filter/sort/paginate). Bodies are for the resource representation.
- Type everything. The richer your hints, the better your validation and your auto-generated OpenAPI docs.

## Interview Q&A

**Q: How does FastAPI decide whether a function parameter is a path, query, or body parameter?**
**A:** Three rules, in order: (1) if the name matches a `{placeholder}` in the route path, it's a path param; (2) otherwise if it's a singular/scalar type (int, str, float, bool, etc.), it's a query param; (3) otherwise if it's a Pydantic model — or is explicitly wrapped in `Body()` — it comes from the request body. The type hints and the route string do the routing, not the argument order.

**Q: What makes a query parameter required vs optional?**
**A:** The presence of a default. No default (`q: str`) makes it required; any default (`q: str = "x"` or `q: str | None = None`) makes it optional. With `Query()`, omitting the default — or passing `...` — keeps it required. This mirrors normal Python function defaults.

**Q: A client sends `item_id=abc` to a route declared as `item_id: int`. What happens?**
**A:** Your handler never runs. FastAPI tries to coerce `"abc"` to `int`, fails, and returns a `422 Unprocessable Entity` with a structured `detail` list pointing to `loc: ["path","item_id"]` and an explanatory message. Validation happens before your code.

**Q: How do you add constraints like "greater than 0" or "max length 50" to params?**
**A:** Wrap the type in `Annotated` with `Path()`/`Query()`/`Body()` and pass the constraint: `Annotated[int, Query(gt=0, le=100)]` or `Annotated[str, Query(max_length=50, pattern="^[a-z]+$")]`. Numeric constraints are `gt/ge/lt/le/multiple_of`; string ones are `min_length/max_length/pattern`.

**Q: What's the difference between the old `q: str = Query(...)` style and `Annotated[str, Query(...)]`?**
**A:** Functionally similar, but `Annotated` is the recommended modern form: the default value stays in the normal position (`= None`), the function remains a clean callable, and the annotated type can be reused/aliased. The old style stores metadata *as* the default, which is more confusing.

**Q: How do you accept multiple Pydantic models in one request body?**
**A:** Declare multiple model params; FastAPI wraps them into a single JSON object keyed by the parameter names, e.g. `{"item": {...}, "user": {...}}`. To force a scalar into that body, mark it `Annotated[int, Body()]`. With a single model you can use `Body(embed=True)` to nest it under its parameter name.

**Q: What does `response_model` do, and why is it a security feature?**
**A:** It declares and enforces the response schema. FastAPI serializes your return value through that model, dropping any field not in it — so even if you return a full DB object containing a password hash, a `UserOut` without that field guarantees it's filtered out. It also documents the response in OpenAPI and supports `exclude_none` / `exclude_unset`.

**Q: How do you return a 404 (or other error status)?**
**A:** `raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Item not found")`. Raising — not returning — ensures the correct status code; a returned dict would still be a `200`. You can also pass custom `headers` and a structured `detail`.

**Q: How do you set the success status code to 201 for a create endpoint?**
**A:** Pass `status_code=status.HTTP_201_CREATED` to the route decorator: `@app.post("/users/", response_model=UserOut, status_code=status.HTTP_201_CREATED)`. The `status_code` argument controls the *success* response code; errors still come from `HTTPException`.

**Q: How do you read a header or cookie?**
**A:** Declare a param with `Header()` or `Cookie()`: `user_agent: Annotated[str | None, Header()] = None` reads the `User-Agent` header (underscores auto-convert to hyphens; disable with `convert_underscores=False`). Cookies work the same with `Cookie()`.

**Q: Why does `tags: list[str] = []` not work for `?tags=a&tags=b` and how do you fix it?**
**A:** Without explicit marking, a non-scalar like `list` is interpreted as a body. Wrap it: `tags: Annotated[list[str], Query()] = []` tells FastAPI it's a repeated query param. Now `?tags=a&tags=b` yields `["a","b"]`.

**Q: What's the difference between validating the request body and `response_model`?**
**A:** The request body is validated by the parameter's own Pydantic model/type when the request comes in. `response_model` validates and filters what you send back out. They are independent — one guards input, the other guards output.

## Practice exercises

Try these before peeking. Run with `fastapi dev main.py` and test at `/docs`.

1. **Pagination params.** Write `GET /books/` with query params `skip` (default 0, must be `>= 0`) and `limit` (default 10, range 1–100). Return the received values. Trigger a 422 by sending `limit=0`.

2. **Search with constraints.** Write `GET /search/` with an optional query param `q` that, *if provided*, must be 3–30 characters and match `^[a-zA-Z0-9 ]+$`. Return `{"q": q}` or `{"q": None}`.

3. **Create with response filtering.** Define `AccountIn(username, password, email)` and `AccountOut(username, email)`. Write `POST /accounts/` that returns the created account using `response_model=AccountOut` and `status_code=201`. Verify the password is not in the response.

4. **Combined params.** Write `PUT /books/{book_id}` taking a path `book_id: int (>=1)`, a body model `Book(title: str, pages: int (gt=0))`, and an optional query `notify: bool = False`. Return everything combined.

5. **404 handling.** Given `BOOKS = {1: "Dune", 2: "Neuromancer"}`, write `GET /books/{book_id}` that returns the title or raises a 404 with `detail="Book not found"`.

6. **Multiple body params.** Write `POST /reviews/` taking two models `Book(title, pages)` and `Reviewer(name)` plus a body scalar `rating: int (1..5)`. Return all three.

<details>
<summary>Solutions</summary>

```python
from typing import Annotated
from enum import Enum
from fastapi import FastAPI, Path, Query, Body, HTTPException, status
from pydantic import BaseModel, Field, EmailStr

app = FastAPI()

# 1. Pagination
@app.get("/books/")
def list_books(
    skip: Annotated[int, Query(ge=0)] = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
):
    return {"skip": skip, "limit": limit}
# /books/?limit=0 => 422

# 2. Search with constraints
@app.get("/search/")
def search(
    q: Annotated[
        str | None,
        Query(min_length=3, max_length=30, pattern="^[a-zA-Z0-9 ]+$"),
    ] = None,
):
    return {"q": q}

# 3. Create with response filtering
class AccountIn(BaseModel):
    username: str
    password: str
    email: EmailStr

class AccountOut(BaseModel):
    username: str
    email: EmailStr

@app.post("/accounts/", response_model=AccountOut, status_code=status.HTTP_201_CREATED)
def create_account(account: AccountIn):
    # In real code: hash the password and persist here.
    return account   # password is filtered out by AccountOut

# 4. Combined params
class Book(BaseModel):
    title: str
    pages: int = Field(gt=0)

@app.put("/books/{book_id}")
def update_book(
    book_id: Annotated[int, Path(ge=1)],
    book: Book,
    notify: bool = False,
):
    return {"book_id": book_id, **book.model_dump(), "notify": notify}

# 5. 404 handling
BOOKS = {1: "Dune", 2: "Neuromancer"}

@app.get("/library/{book_id}")
def get_book(book_id: int):
    if book_id not in BOOKS:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Book not found",
        )
    return {"book_id": book_id, "title": BOOKS[book_id]}

# 6. Multiple body params
class Reviewer(BaseModel):
    name: str

@app.post("/reviews/")
def create_review(
    book: Book,
    reviewer: Reviewer,
    rating: Annotated[int, Body(ge=1, le=5)],
):
    return {"book": book, "reviewer": reviewer, "rating": rating}
# Expected body:
# {"book": {"title":"Dune","pages":412},
#  "reviewer": {"name":"Paul"},
#  "rating": 5}
```

</details>

## Cheat sheet

```python
from typing import Annotated
from fastapi import FastAPI, Path, Query, Body, Header, Cookie, HTTPException, status
from pydantic import BaseModel, Field

app = FastAPI()

# PATH  — name must match {placeholder}; type = auto convert+validate
@app.get("/items/{item_id}")
def f(item_id: Annotated[int, Path(ge=1)]): ...

# QUERY — scalar params; default => optional, no default => required
@app.get("/q/")
def g(
    required: str,                                   # required query
    optional: str | None = None,                     # optional
    limit: Annotated[int, Query(gt=0, le=100)] = 10, # constrained
    tags: Annotated[list[str], Query()] = [],        # ?tags=a&tags=b
): ...

# BODY  — Pydantic model => request body
class Item(BaseModel):
    name: str
    price: float = Field(gt=0)
    description: str | None = None

@app.post("/items/", status_code=status.HTTP_201_CREATED)
def h(item: Item): ...

# COMBINED — path + body + query in one signature
@app.put("/items/{item_id}")
def u(item_id: int, item: Item, q: str | None = None): ...

# MULTIPLE BODY + scalar body + embed
def m(item: Item, importance: Annotated[int, Body(gt=0)]): ...   # {"item":{...},"importance":3}
def e(item: Annotated[Item, Body(embed=True)]): ...              # {"item":{...}}
```

| Need | Tool | Example |
|---|---|---|
| URL segment | `Path()` | `Annotated[int, Path(ge=1)]` |
| `?key=val` | `Query()` | `Annotated[str, Query(min_length=3)]` |
| JSON body | Pydantic model / `Body()` | `item: Item` |
| Required | no default / `...` | `q: str` |
| Optional | default value | `q: str \| None = None` |
| Numeric bounds | `gt ge lt le multiple_of` | `Query(gt=0, le=100)` |
| String bounds | `min_length max_length pattern` | `Query(pattern="^[a-z]+$")` |
| Filter output | `response_model` | `response_model=UserOut` |
| Drop None/unset | `response_model_exclude_none/unset` | `=True` |
| Success status | `status_code` | `status.HTTP_201_CREATED` |
| Error response | `raise HTTPException` | `HTTPException(404, "Not found")` |
| Header / cookie | `Header() / Cookie()` | `Annotated[str, Header()]` |

**Validation cheat:** type hint = validation. Bad type/constraint => automatic `422` with a `detail` list (`type`, `loc`, `msg`, `input`). You never write the parsing.

**Pydantic v2 reminders:** `model_dump()` not `.dict()`, `model_dump_json()` not `.json()`, `model_config = ConfigDict(...)` not inner `class Config`, `pattern=` not `regex=`.
