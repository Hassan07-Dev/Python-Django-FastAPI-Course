# 42 — Pydantic: Models & Validation

> Week 4 · DRF & FastAPI

Pydantic is the data-validation engine that powers FastAPI (and a huge slice of modern Python). You declare the *shape* of your data with ordinary Python type hints, and Pydantic parses, validates, coerces, and serializes it for you. If DRF serializers feel like Laravel's Form Requests with a class around them, Pydantic models are the leaner, type-hint-native cousin — and they're the thing you'll touch every single day building FastAPI APIs.

## What you will learn

- Why `BaseModel` is the core of Pydantic v2 and how it turns type hints into validation
- Declaring fields with type hints, defaults, and `Optional` (required vs optional)
- Pydantic's parsing/coercion rules — what it converts silently and what it rejects
- `Field()` for constraints and metadata: `default`, `alias`, `ge`/`le`, `min_length`, `pattern`, `description`, `examples`
- Nested models and lists of models
- Custom validation with `@field_validator` (`mode="before"`/`"after"`) and `@model_validator`
- `@computed_field` for derived, serialized properties
- `model_config = ConfigDict(...)`: `from_attributes`, `populate_by_name`, `str_strip_whitespace`, `extra`
- Serialization (`model_dump`, `model_dump_json`) and parsing (`model_validate`, `model_validate_json`)
- `BaseSettings` from `pydantic-settings` for typed environment configuration
- How all of this maps to DRF serializers and Laravel validation

## PHP/Laravel bridge

In Laravel you validate incoming data with a `FormRequest` (or `$request->validate([...])`) using string rules like `'email' => 'required|email|max:255'`, and you read config through `config()` / `.env`. Pydantic collapses three Laravel concepts into one typed class: the **validation rules** (Form Request), the **DTO / typed object** you pass around your app, and the **API resource / transformer** that serializes output. Instead of stringy rules, you use Python type hints plus `Field(...)` constraints, so your editor and type checker understand the data too. And `pydantic-settings.BaseSettings` is the typed, validated replacement for `config()` + `env()` — your environment variables become a strongly-typed object that fails fast at startup if something is missing or malformed.

## Core concept: `BaseModel` turns type hints into validation

A Pydantic model is just a class that subclasses `BaseModel`. Each class attribute with a type hint becomes a validated field. There's no boilerplate `__init__`, no manual `if not x: raise` — the types *are* the schema.

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    is_active: bool = True   # has a default => optional

# Keyword construction; Pydantic validates everything here.
u = User(id=1, name="Ada")
print(u)             # => id=1 name='Ada' is_active=True
print(u.id, u.name)  # => 1 Ada
print(type(u.id))    # => <class 'int'>
```

The magic moment for a PHP dev: validation happens **at construction time**. The instant `User(...)` returns, you have a guaranteed-valid object. There is no separate "did validation pass?" step like `$validator->fails()` — if it didn't pass, construction raised an exception.

### Validation errors are structured

When data is invalid, Pydantic raises a `ValidationError` containing a list of every problem (not just the first one), each with a location, message, and error type.

```python
from pydantic import BaseModel, ValidationError

class User(BaseModel):
    id: int
    name: str

try:
    User(id="not-a-number", name=123)
except ValidationError as e:
    print(e.error_count())   # => 2  (collects ALL errors, like Laravel's MessageBag)
    for err in e.errors():
        print(err["loc"], err["type"])
    # => ('id',) int_parsing
    # => ('name',) string_type
```

`e.errors()` gives you machine-readable detail; `str(e)` gives a human-readable summary. FastAPI catches this automatically and returns a clean `422 Unprocessable Entity` JSON response — you rarely catch it by hand in a real endpoint.

## Core concept: parsing, coercion, and what gets rejected

Pydantic does not merely *check* types — it **parses** input into the declared type when the conversion is safe and lossless. This is "smart" coercion, and knowing the rules prevents surprises.

```python
from pydantic import BaseModel

class Item(BaseModel):
    qty: int
    price: float
    in_stock: bool

# Strings that cleanly represent the target type are coerced:
item = Item(qty="10", price="19.99", in_stock="yes")
print(item.qty, type(item.qty))         # => 10 <class 'int'>
print(item.price, type(item.price))     # => 19.99 <class 'float'>
print(item.in_stock)                    # => True  ('yes'/'true'/'1'/'on' -> True)
```

But lossy or ambiguous conversions are rejected:

```python
from pydantic import BaseModel, ValidationError

class Item(BaseModel):
    qty: int

try:
    Item(qty="10.5")    # not a clean int
except ValidationError as e:
    print(e.errors()[0]["type"])   # => int_parsing

try:
    Item(qty=10.5)      # float with fractional part -> rejected
except ValidationError as e:
    print(e.errors()[0]["type"])   # => int_from_float
```

> **Beginner question: "Isn't silent coercion dangerous?"** It's the right default for web data, where everything arrives as strings (query params, form fields, headers). If you want zero coercion, Pydantic offers **strict mode** — annotate a field with `Strict` or set `model_config = ConfigDict(strict=True)`, and `"10"` will no longer become `10`.

```python
from pydantic import BaseModel, ValidationError, ConfigDict

class StrictItem(BaseModel):
    model_config = ConfigDict(strict=True)
    qty: int

try:
    StrictItem(qty="10")   # rejected: no coercion in strict mode
except ValidationError as e:
    print(e.errors()[0]["type"])   # => int_type
```

## Core concept: defaults, required vs optional, and `Optional`

A field is **required** unless it has a default. Giving it a default makes it optional.

```python
from typing import Optional
from pydantic import BaseModel

class Profile(BaseModel):
    username: str                 # required (no default)
    bio: str = ""                 # optional, defaults to ""
    age: Optional[int] = None     # optional, may be int OR None
    nickname: str | None = None   # modern syntax (Python 3.10+) — same thing
```

Two subtleties that trip up beginners:

1. **`Optional[int]` does NOT mean "optional field".** It means "the value may be `int` or `None`". Optionality (can it be omitted?) is governed by whether there's a *default*. `Optional[int]` with no default is still **required** — you'd have to pass `None` explicitly.

```python
from typing import Optional
from pydantic import BaseModel, ValidationError

class A(BaseModel):
    x: Optional[int]      # required field that accepts None

try:
    A()                   # missing!
except ValidationError as e:
    print(e.errors()[0]["type"])   # => missing
A(x=None)                 # OK — you must pass it, even if None
```

2. **Never use a mutable default directly** the way you might fear from Python functions — but with Pydantic it's actually safe because it deep-copies, *except* you should still use `Field(default_factory=...)` to be explicit and correct for things like lists/dicts.

```python
from pydantic import BaseModel, Field

class Cart(BaseModel):
    items: list[str] = Field(default_factory=list)   # fresh list per instance
    meta: dict[str, str] = Field(default_factory=dict)
```

## Core concept: `Field()` for constraints and metadata

Type hints describe *kind*; `Field()` describes *rules* and *documentation*. This is where Laravel's `'price' => 'required|numeric|min:0'` lives.

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=120,
                      description="Display name", examples=["Widget"])
    price: float = Field(ge=0, description="Price in USD, must be >= 0")
    discount_pct: int = Field(default=0, ge=0, le=100)
    sku: str = Field(pattern=r"^[A-Z]{3}-\d{4}$", examples=["ABC-1234"])
    tags: list[str] = Field(default_factory=list, max_length=10)

p = Product(name="Widget", price=9.99, sku="ABC-1234")
print(p.discount_pct)   # => 0
```

Common constraint keywords (mapped to Laravel for recall):

| Pydantic `Field(...)`      | Applies to        | Laravel equivalent          |
| -------------------------- | ----------------- | --------------------------- |
| `ge=`, `le=`, `gt=`, `lt=` | numbers           | `min:`, `max:`, `gt:`,`lt:` |
| `min_length=`,`max_length=`| str / list / dict | `min:`, `max:`              |
| `pattern=`                 | str               | `regex:`                    |
| `multiple_of=`             | numbers           | `multiple_of:`             |
| `default=` / `default_factory=` | any          | default value in `validated()` |
| `alias=`                   | any               | input key mapping           |
| `description=`,`examples=` | any               | (feeds OpenAPI docs)        |

### `alias` — mapping external names to Pythonic names

APIs often send `camelCase` or reserved-word keys. `alias` lets the *wire format* differ from your Python attribute name.

```python
from pydantic import BaseModel, Field, ConfigDict

class Order(BaseModel):
    # Accept "orderId" on input, expose .order_id in Python.
    order_id: int = Field(alias="orderId")
    customer: str

o = Order.model_validate({"orderId": 42, "customer": "Ada"})
print(o.order_id)                      # => 42
print(o.model_dump(by_alias=True))     # => {'orderId': 42, 'customer': 'Ada'}
```

By default an aliased field only accepts the alias on input. To accept *either* the field name or the alias, enable `populate_by_name` (see model config below).

## Core concept: nested models and lists of models

Compose models like Lego. A field typed as another `BaseModel` is validated recursively; `list[Model]` validates every element.

```python
from pydantic import BaseModel, Field

class Address(BaseModel):
    street: str
    city: str
    zip_code: str = Field(pattern=r"^\d{5}$")

class LineItem(BaseModel):
    sku: str
    qty: int = Field(ge=1)

class Customer(BaseModel):
    name: str
    shipping: Address                      # nested model
    items: list[LineItem] = Field(default_factory=list)  # list of models

raw = {
    "name": "Ada",
    "shipping": {"street": "1 Main", "city": "Lovelace", "zip_code": "12345"},
    "items": [{"sku": "ABC", "qty": 2}, {"sku": "XYZ", "qty": 1}],
}
c = Customer.model_validate(raw)
print(c.shipping.city)        # => Lovelace
print(c.items[0].qty)         # => 2
print(type(c.items[0]))       # => <class '__main__.LineItem'>  (real objects, not dicts)
```

Notice you pass plain nested dicts and get back fully-typed nested objects. This is exactly DRF's nested serializers, but with far less ceremony.

## Core concept: custom validators

When a constraint can't be expressed with `Field()`, write a validator. Pydantic v2 has two: `@field_validator` (one field) and `@model_validator` (the whole model, e.g. cross-field rules).

### `@field_validator` — `mode="after"` (the default)

An *after* validator runs **after** Pydantic has parsed the value into the target type, so `value` is already the correct Python type. Use it to enforce business rules and to normalize.

```python
from pydantic import BaseModel, field_validator

class Account(BaseModel):
    username: str
    email: str

    @field_validator("username")
    @classmethod
    def username_alnum(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("username must be alphanumeric")
        return v.lower()          # validators can transform & return the value

    @field_validator("email")
    @classmethod
    def email_has_at(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("invalid email")
        return v

a = Account(username="Ada42", email="ada@x.com")
print(a.username)   # => ada42   (normalized to lowercase)
```

Raise `ValueError` (or `AssertionError`) to signal failure — Pydantic wraps it into a `ValidationError`. Don't raise `ValidationError` yourself inside a validator.

### `@field_validator` — `mode="before"`

A *before* validator runs on the **raw input**, before type coercion. Use it to clean/reshape data that wouldn't otherwise parse — e.g. accept a comma-separated string and turn it into a list.

```python
from pydantic import BaseModel, field_validator

class Post(BaseModel):
    tags: list[str]

    @field_validator("tags", mode="before")
    @classmethod
    def split_csv(cls, v):
        if isinstance(v, str):
            return [t.strip() for t in v.split(",") if t.strip()]
        return v

print(Post(tags="python, fastapi , web").tags)   # => ['python', 'fastapi', 'web']
print(Post(tags=["already", "list"]).tags)        # => ['already', 'list']
```

You can target several fields at once: `@field_validator("a", "b")`, or all fields with `@field_validator("*")`.

### `@model_validator` — cross-field rules

For rules that span multiple fields (password confirmation, date ranges), validate the whole model. `mode="after"` gives you the constructed model instance.

```python
from datetime import date
from pydantic import BaseModel, model_validator

class Booking(BaseModel):
    check_in: date
    check_out: date

    @model_validator(mode="after")
    def check_dates(self):
        if self.check_out <= self.check_in:
            raise ValueError("check_out must be after check_in")
        return self          # always return self in an after model_validator

b = Booking(check_in="2026-07-01", check_out="2026-07-05")   # dates parsed from ISO strings
print(b.check_out)   # => 2026-07-05
```

`mode="before"` model validators receive the raw input dict and run before any field validation — handy for restructuring an entire payload.

## Core concept: computed fields

A `@computed_field` is a derived, read-only property that **gets included in serialization** (unlike a plain `@property`, which does not appear in `model_dump()`).

```python
from pydantic import BaseModel, computed_field

class Rectangle(BaseModel):
    width: float
    height: float

    @computed_field
    @property
    def area(self) -> float:
        return self.width * self.height

r = Rectangle(width=3, height=4)
print(r.area)                 # => 12.0
print(r.model_dump())         # => {'width': 3.0, 'height': 4.0, 'area': 12.0}
```

This is the Pydantic answer to Laravel API Resource accessors / computed attributes — derived output without storing it.

## Core concept: model configuration with `ConfigDict`

`model_config = ConfigDict(...)` tunes how a model behaves. The four you'll use constantly:

```python
from pydantic import BaseModel, ConfigDict, Field

class UserOut(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,        # build from ORM objects / any attr-having obj
        populate_by_name=True,       # accept BOTH field name and alias on input
        str_strip_whitespace=True,   # auto-trim all str fields
        extra="forbid",              # reject unknown keys (default is "ignore")
    )
    user_id: int = Field(alias="id")
    name: str
```

### `from_attributes=True` — building models from ORM rows

By default `model_validate` expects a dict. With `from_attributes=True`, you can pass any object and Pydantic reads attributes off it — perfect for turning a SQLAlchemy/Django row into an API response model.

```python
from pydantic import BaseModel, ConfigDict

class ORMRow:                       # stand-in for a SQLAlchemy/Django model row
    def __init__(self):
        self.id = 7
        self.name = "  Ada  "

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True, str_strip_whitespace=True)
    id: int
    name: str

dto = UserOut.model_validate(ORMRow())     # reads .id and .name off the object
print(dto.model_dump())                    # => {'id': 7, 'name': 'Ada'}  (trimmed)
```

### `extra` — what to do with unknown keys

| Value     | Behavior on unknown input keys                    |
| --------- | ------------------------------------------------- |
| `"ignore"`| (default) silently drop them                      |
| `"forbid"`| raise `ValidationError` — strict, recommended for APIs |
| `"allow"` | keep them as extra attributes on the model        |

```python
from pydantic import BaseModel, ConfigDict, ValidationError

class Strict(BaseModel):
    model_config = ConfigDict(extra="forbid")
    name: str

try:
    Strict(name="Ada", hacker="injected")
except ValidationError as e:
    print(e.errors()[0]["type"])   # => extra_forbidden
```

## Core concept: serialization and parsing

These are the verbs you'll type most. Note the v2 names — `.dict()` and `.json()` are deprecated.

| Action                         | v2 method (use these)        | What it does                          |
| ------------------------------ | ---------------------------- | ------------------------------------- |
| dict from a Python obj/dict    | `Model.model_validate(obj)`  | parse + validate, returns model       |
| dict from a JSON **string**    | `Model.model_validate_json(s)` | parse JSON + validate                |
| model -> Python dict           | `m.model_dump()`             | nested models become dicts            |
| model -> JSON string           | `m.model_dump_json()`        | one step, JSON-safe                   |
| JSON Schema (for docs)         | `Model.model_json_schema()`  | OpenAPI-style schema                  |

```python
from datetime import datetime
from pydantic import BaseModel

class Event(BaseModel):
    name: str
    at: datetime
    tags: list[str] = []

e = Event.model_validate({"name": "launch", "at": "2026-06-19T10:00:00", "tags": ["x"]})

print(e.model_dump())
# => {'name': 'launch', 'at': datetime.datetime(2026, 6, 19, 10, 0), 'tags': ['x']}

print(e.model_dump(mode="json"))   # JSON-safe primitives (datetime -> ISO string)
# => {'name': 'launch', 'at': '2026-06-19T10:00:00', 'tags': ['x']}

print(e.model_dump_json())
# => {"name":"launch","at":"2026-06-19T10:00:00","tags":["x"]}
```

Useful `model_dump` options for shaping API output:

```python
m = Event.model_validate({"name": "launch", "at": "2026-06-19T10:00:00"})
m.model_dump(include={"name"})            # => {'name': 'launch'}
m.model_dump(exclude={"tags"})            # drop a field
m.model_dump(exclude_none=True)           # drop None values
m.model_dump(by_alias=True)               # use field aliases as keys
m.model_dump(exclude_defaults=True)       # drop fields still at their default
```

## Core concept: `BaseSettings` for environment config

`pydantic-settings` (a separate install: `pip install pydantic-settings`) gives you typed, validated configuration sourced from environment variables and `.env` files. This replaces Laravel's `config()` + `env()` with something that *fails at startup* if a required var is missing or the wrong type.

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="APP_",        # looks for APP_DEBUG, APP_DB_URL, ...
        extra="ignore",
    )
    debug: bool = False
    db_url: str                                   # required: errors if APP_DB_URL unset
    max_connections: int = Field(default=10, ge=1)
    secret_key: str = Field(min_length=16)

# Reads from environment / .env at instantiation.
settings = Settings()           # type: ignore[call-arg]
print(settings.debug, settings.max_connections)
```

```bash
# .env  (env_prefix="APP_" means each var is prefixed)
APP_DEBUG=true
APP_DB_URL=postgresql://localhost/app
APP_SECRET_KEY=super-secret-value-123
```

Instantiate **once** and reuse (often via a cached function). Because it's a real Pydantic model, you get autocompletion, type checking, constraints (`ge`, `min_length`), and validators on your config — something stringy `env()` can never give you.

```python
from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()           # built once, cached for the process
```

## How this maps to DRF serializers and Laravel validation

| Concept                | Laravel                         | DRF                                | Pydantic v2                          |
| ---------------------- | ------------------------------- | ---------------------------------- | ------------------------------------ |
| Declare rules          | `FormRequest` rules array       | `serializers.Serializer` fields    | `BaseModel` + type hints + `Field()` |
| Run validation         | `$request->validate()`          | `serializer.is_valid()`            | construct / `model_validate()`       |
| Errors                 | `MessageBag` / `$errors`        | `serializer.errors`                | `ValidationError.errors()`           |
| Single-field rule      | custom Rule object              | `validate_<field>()`               | `@field_validator("field")`          |
| Cross-field rule       | `withValidator` closure         | `validate(self, data)`             | `@model_validator(mode="after")`     |
| Output transform       | API Resource                    | `Serializer` (read) / `to_repr`    | `model_dump()` / `@computed_field`   |
| From DB object         | Eloquent -> Resource            | `ModelSerializer(instance)`        | `from_attributes=True`               |
| Config                 | `config()` + `.env`             | `settings.py`                      | `BaseSettings`                       |

The key mental shift: in DRF a serializer is a heavier object that knows about querysets and the DB; Pydantic models are *pure data* and DB-agnostic. In FastAPI you typically pair a Pydantic model (validation/serialization) with a separate ORM layer (SQLAlchemy/Tortoise), wiring them together via `from_attributes=True`.

## Common mistakes & gotchas

- **Using v1 APIs.** `.dict()`, `.json()`, `class Config:`, `@validator`, and `parse_obj()` are v1/deprecated. Use `model_dump()`, `model_dump_json()`, `model_config = ConfigDict(...)`, `@field_validator`, and `model_validate()`.
- **Thinking `Optional[X]` makes a field optional.** It only allows `None`. Add a default (`= None`) to make it actually omittable.
- **Forgetting `@classmethod` under `@field_validator`.** The validator is a classmethod; the decorator order is `@field_validator(...)` then `@classmethod` then `def`.
- **Forgetting to `return` from a validator.** Whatever you return becomes the field's value. Return nothing and the field becomes `None`.
- **Raising `ValidationError` inside a validator.** Raise plain `ValueError`/`AssertionError`; Pydantic packages it for you.
- **Mutable defaults.** Use `Field(default_factory=list)` for lists/dicts, not `= []`.
- **`extra="ignore"` hiding typos.** A client sending `e_mail` instead of `email` is silently dropped. Use `extra="forbid"` on request models to catch this.
- **Expecting `model_dump()` to give JSON-safe types.** Plain `model_dump()` keeps `datetime`, `Decimal`, etc. as Python objects. Use `model_dump(mode="json")` or `model_dump_json()` when you need primitives.
- **Aliases blocking the field name.** With an `alias`, the plain name is rejected on input unless `populate_by_name=True`.
- **Calling `Settings()` everywhere.** Re-reading the environment on every request is wasteful — cache it.

## Best practices

- Define separate models per direction: a `XxxCreate`/`XxxUpdate` (input) and `XxxOut` (output). Don't reuse one model for everything.
- Put `extra="forbid"` on input models; reserve `extra="ignore"` for tolerant integrations.
- Prefer `Field()` constraints over validators when a constraint fits — they're declarative and show up in OpenAPI docs.
- Use `model_validate()` / `model_validate_json()` instead of constructing from untrusted dicts with `**data`.
- Add `description`/`examples` to `Field()` — FastAPI turns them into great interactive docs for free.
- Use `from_attributes=True` on output models to convert ORM rows cleanly.
- Centralize config in one `BaseSettings` class, instantiate once, inject it. Keep secrets in `.env`, never in code.
- Normalize in `mode="after"` validators (e.g. lowercase emails) so the rest of your app gets clean data.
- Use modern `str | None` union syntax (Python 3.10+) over `Optional[str]` for readability — both work.

## Interview Q&A

**Q: What is Pydantic and why is it used in FastAPI?**
**A:** Pydantic is a data-validation and serialization library that uses Python type hints to define, parse, validate, and serialize data. FastAPI uses it for request body validation, response shaping, and auto-generating OpenAPI/JSON Schema docs. Declaring a Pydantic model as an endpoint parameter gives you automatic 422 error responses and editor-level type safety.

**Q: How does Pydantic decide if a field is required or optional?**
**A:** Solely by whether it has a default. No default => required. A default value (including `None` or a `default_factory`) makes it optional. `Optional[int]` only changes the allowed *type* (int or None); without a default it's still a required field that you must pass explicitly.

**Q: Explain coercion vs strict mode.**
**A:** By default Pydantic coerces compatible inputs — `"10"` to `int 10`, `"true"` to `bool True` — which is ideal because web inputs arrive as strings. Lossy conversions (like `"10.5"` to int) are rejected. Strict mode (`ConfigDict(strict=True)` or the `Strict` annotation) disables coercion so types must match exactly.

**Q: Difference between `@field_validator` `mode="before"` and `mode="after"`?**
**A:** A `before` validator runs on the raw input before type coercion — use it to reshape data so it can parse (e.g. split a CSV string into a list). An `after` validator runs once the value is already the correct Python type — use it for business rules and normalization. `after` is the default.

**Q: When do you use `@model_validator` instead of `@field_validator`?**
**A:** When a rule depends on more than one field — password vs confirmation, `check_out` after `check_in`, "at least one contact method present". `@model_validator(mode="after")` receives the whole constructed instance, so you can compare fields; always `return self`.

**Q: What does `from_attributes=True` do?**
**A:** It lets `model_validate()` read values from an object's attributes rather than from dict keys, so you can build a Pydantic model directly from a SQLAlchemy or Django ORM row. It replaces v1's `orm_mode`.

**Q: How do you serialize a Pydantic model, and what changed from v1?**
**A:** Use `model_dump()` for a Python dict and `model_dump_json()` for a JSON string; `model_dump(mode="json")` yields JSON-safe primitives. In v1 these were `.dict()` and `.json()`, now deprecated. Options like `by_alias`, `exclude_none`, `include`/`exclude`, and `exclude_defaults` shape the output.

**Q: What's the difference between a plain `@property` and `@computed_field`?**
**A:** Both compute a value on access, but only `@computed_field` is included in serialization (`model_dump()` and JSON Schema). A plain `@property` is invisible to serialization. Use `@computed_field` for derived output you want in API responses.

**Q: How does `alias` work and how do you accept both names?**
**A:** `Field(alias="orderId")` makes the model read `orderId` from input and (with `by_alias=True`) write it on output, while you reference `.order_id` in Python. By default only the alias is accepted on input; set `populate_by_name=True` to accept the Python field name as well.

**Q: How do you handle unknown/extra keys?**
**A:** Via `ConfigDict(extra=...)`. `"ignore"` (default) drops them, `"forbid"` raises a `ValidationError`, and `"allow"` stores them on the model. For API request models, `"forbid"` is safest because it surfaces client typos.

**Q: What is `BaseSettings` and why prefer it over reading `os.environ`?**
**A:** `BaseSettings` (from `pydantic-settings`) maps environment variables and `.env` entries onto a typed model with defaults, constraints, and validators. It fails fast at startup if a required var is missing or malformed, and gives you autocompletion and type checking — far safer than scattered `os.environ.get()` calls returning untyped strings.

**Q: How do you map a Pydantic concept to DRF?**
**A:** A Pydantic `BaseModel` is roughly a DRF serializer: fields ~ serializer fields, `@field_validator` ~ `validate_<field>`, `@model_validator` ~ `validate()`, `model_dump()` ~ serializer `.data`, and `from_attributes=True` ~ passing a model instance to a `ModelSerializer`. The difference is Pydantic is pure data (DB-agnostic), while DRF serializers are coupled to Django's ORM.

## Practice exercises

1. **Signup model.** Create a `SignupRequest` with `username` (3–20 chars, alphanumeric), `email` (must contain `@`), `password` (min 8 chars), and `password_confirm`. Validate that the two passwords match. Reject unknown keys.

2. **CSV tags.** Create a `Article` model with `title: str` and `tags: list[str]`. Accept `tags` as either a real list or a comma-separated string, trimming whitespace and dropping empties.

3. **ORM to DTO.** Given a plain class `UserRow` with attributes `id`, `full_name`, and `email`, build a `UserOut` Pydantic model that reads from attributes, strips whitespace on strings, exposes `full_name` under the output alias `name`, and adds a computed `email_domain` field.

4. **Nested order.** Model an `Order` containing a list of `OrderLine` (each with `sku` and `qty>=1`) and a computed `total_qty`. Parse it from a dict and print the total.

5. **Typed settings.** Write a `Settings` (`BaseSettings`) reading `APP_`-prefixed vars: `debug: bool=False`, `db_url: str` (required), `pool_size: int` (1–50, default 5). Show what error you'd get if `APP_DB_URL` is missing.

<details><summary>Solutions</summary>

```python
# 1. Signup model
from pydantic import BaseModel, Field, ConfigDict, field_validator, model_validator, ValidationError

class SignupRequest(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)
    username: str = Field(min_length=3, max_length=20)
    email: str
    password: str = Field(min_length=8)
    password_confirm: str

    @field_validator("username")
    @classmethod
    def alnum(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("username must be alphanumeric")
        return v

    @field_validator("email")
    @classmethod
    def has_at(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("invalid email")
        return v

    @model_validator(mode="after")
    def passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError("passwords do not match")
        return self

s = SignupRequest(username="ada42", email="ada@x.com",
                  password="hunter2x", password_confirm="hunter2x")
print(s.username)   # => ada42
```

```python
# 2. CSV tags
from pydantic import BaseModel, field_validator

class Article(BaseModel):
    title: str
    tags: list[str]

    @field_validator("tags", mode="before")
    @classmethod
    def split_csv(cls, v):
        if isinstance(v, str):
            return [t.strip() for t in v.split(",") if t.strip()]
        return v

print(Article(title="Hi", tags="a, b , ,c").tags)   # => ['a', 'b', 'c']
print(Article(title="Hi", tags=["x", "y"]).tags)     # => ['x', 'y']
```

```python
# 3. ORM to DTO
from pydantic import BaseModel, ConfigDict, Field, computed_field

class UserRow:
    def __init__(self):
        self.id = 1
        self.full_name = "  Ada Lovelace  "
        self.email = "ada@math.org"

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True, str_strip_whitespace=True,
                              populate_by_name=True)
    id: int
    name: str = Field(alias="full_name")
    email: str

    @computed_field
    @property
    def email_domain(self) -> str:
        return self.email.split("@")[-1]

dto = UserOut.model_validate(UserRow())
print(dto.model_dump(by_alias=True))
# => {'id': 1, 'full_name': 'Ada Lovelace', 'email': 'ada@math.org', 'email_domain': 'math.org'}
```

```python
# 4. Nested order
from pydantic import BaseModel, Field, computed_field

class OrderLine(BaseModel):
    sku: str
    qty: int = Field(ge=1)

class Order(BaseModel):
    lines: list[OrderLine]

    @computed_field
    @property
    def total_qty(self) -> int:
        return sum(line.qty for line in self.lines)

o = Order.model_validate({"lines": [{"sku": "A", "qty": 2}, {"sku": "B", "qty": 3}]})
print(o.total_qty)   # => 5
```

```python
# 5. Typed settings
from pydantic import Field, ValidationError
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_", env_file=".env", extra="ignore")
    debug: bool = False
    db_url: str
    pool_size: int = Field(default=5, ge=1, le=50)

# If APP_DB_URL is unset:
try:
    Settings()
except ValidationError as e:
    print(e.errors()[0]["type"])   # => missing  (loc = ('db_url',))
```

</details>

## Cheat sheet

```python
from pydantic import (BaseModel, Field, ConfigDict, ValidationError,
                      field_validator, model_validator, computed_field)
from pydantic_settings import BaseSettings, SettingsConfigDict

class M(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,      # build from ORM objects
        populate_by_name=True,     # accept field name AND alias
        str_strip_whitespace=True, # trim str fields
        extra="forbid",            # reject unknown keys ("ignore"/"allow")
        strict=False,              # True => no coercion
    )
    id: int                                  # required
    name: str = Field(min_length=1, max_length=50)
    price: float = Field(ge=0)               # >= 0
    code: str = Field(pattern=r"^[A-Z]{3}$")
    nick: str | None = None                  # optional, nullable
    tags: list[str] = Field(default_factory=list)
    ext_id: int = Field(alias="externalId")

    @field_validator("name", mode="after")   # mode="before" for raw input
    @classmethod
    def norm(cls, v): return v.title()

    @model_validator(mode="after")
    def cross(self): return self             # cross-field rules; return self

    @computed_field
    @property
    def label(self) -> str: return f"{self.code}-{self.id}"
```

| Task                         | Call                                   |
| ---------------------------- | -------------------------------------- |
| Parse + validate (dict/obj)  | `M.model_validate(data)`               |
| Parse + validate (JSON str)  | `M.model_validate_json(s)`             |
| To dict                      | `m.model_dump()`                       |
| To JSON-safe dict            | `m.model_dump(mode="json")`            |
| To JSON string               | `m.model_dump_json()`                  |
| By alias / drop None         | `m.model_dump(by_alias=True, exclude_none=True)` |
| JSON Schema                  | `M.model_json_schema()`                |
| All errors                   | `err.errors()` / `err.error_count()`   |

**Field constraints:** `ge le gt lt` (numbers) · `min_length max_length` (str/list) · `pattern` (regex) · `multiple_of` · `default` / `default_factory` · `alias` · `description` / `examples`

**v1 -> v2 renames:** `.dict()` -> `model_dump()` · `.json()` -> `model_dump_json()` · `parse_obj()` -> `model_validate()` · `class Config` -> `model_config = ConfigDict()` · `@validator` -> `@field_validator` · `orm_mode` -> `from_attributes`
