# 18 — Type Hints & the typing Module

> Week 2 · Python language & ecosystem

Python is dynamically typed: a variable can hold an `int` today and a `str` tomorrow, and the interpreter never complains. Type *hints* (a.k.a. annotations) let you write down the types you *intend* anyway. They don't change how your program runs — but they unlock editor autocomplete, static checkers that catch bugs before you ship, and, most importantly for this course, they are the engine behind FastAPI and Pydantic. If you want to be productive in modern Python backend work, type hints are not optional knowledge.

## What you will learn

- Why type hints exist and what relies on them (IDEs, mypy/pyright, FastAPI, Pydantic)
- How to annotate variables, function parameters, and return types
- The modern built-in generic syntax: `list[int]`, `dict[str, int]`, `tuple[int, ...]`
- `Optional[X]` vs `X | None`, and `Union[A, B]` vs `A | B`
- `Any`, `Callable[..., R]`, and `Literal`
- `TypedDict` for dict-shaped data, `Protocol` for duck typing, `TypeVar`/`Generic` for reusable generics
- Type aliases and `NewType`
- The crucial fact that hints are **not enforced at runtime** by Python itself
- Static checking with `mypy` and `pyright`, plus `from __future__ import annotations` and forward references
- How FastAPI turns your hints into request validation and OpenAPI docs

## PHP/Laravel bridge

You already use type declarations in PHP: `function total(int $qty, float $price): float`, union types like `int|string`, nullable types `?User`, and `declare(strict_types=1)`. Python's type hints are the same idea — *but with one giant difference*: PHP actually enforces its type declarations at runtime (a `TypeError` is thrown if you pass the wrong thing), whereas plain Python **ignores hints entirely at runtime**. Think of Python hints as PHPStan/Psalm-level static analysis that happens to use built-in syntax: the checking is done by a separate tool (mypy/pyright), not the language. The exception is libraries like Pydantic and FastAPI, which *read* your hints and add their own runtime validation on top — that part feels much more like Laravel's Form Request validation and route model binding.

## Core concept: why bother with hints at all?

Annotations buy you four concrete things:

1. **Editor intelligence.** Once a parameter is typed `User`, your IDE autocompletes `.email`, `.is_active`, etc., and flags typos instantly.
2. **Documentation that can't rot.** A signature like `def charge(account: Account, cents: int) -> Receipt` tells the next developer everything without a docstring, and a checker keeps it honest.
3. **Bug-catching before runtime.** Static checkers find `None`-dereferences, wrong argument types, and impossible branches across your whole codebase in seconds.
4. **Framework magic.** FastAPI and Pydantic *parse* your hints to generate validation, serialization, and Swagger docs. No hints, no magic.

```python
# Without hints: what is "items"? A list? Of what? What does it return?
def total(items):
    return sum(i["price"] * i["qty"] for i in items)

# With hints: self-documenting, IDE-assisted, checkable.
def total(items: list[dict[str, float]]) -> float:
    return sum(i["price"] * i["qty"] for i in items)
```

## Core concept: annotating variables, parameters, and return types

The syntax is `name: Type` for variables and parameters, and `-> Type` for return values.

```python
# Variable annotations
age: int = 30
name: str = "Ada"
ratio: float = 0.75
active: bool = True

# You can annotate without assigning (useful for declaring intent):
session_token: str  # no value yet; just a promise about its type

# Function parameters and return type
def greet(name: str, excited: bool = False) -> str:
    suffix = "!" if excited else "."
    return f"Hello, {name}{suffix}"

print(greet("Sam", excited=True))  # => Hello, Sam!

# A function that returns nothing uses -> None
def log(message: str) -> None:
    print(f"[LOG] {message}")
    # implicitly returns None
```

A function with no return value should be annotated `-> None` (not omitted). Leaving the return type off means "unannotated", which is different from "returns None" to a type checker.

## Core concept: built-in generic syntax (the modern way)

A *generic* type is a container parameterised by what it holds: a "list of ints", a "dict from str to User". Since **Python 3.9+** you use the built-in collection types directly with square brackets. The old `from typing import List, Dict` forms are deprecated for new code.

```python
# Modern (3.9+): use the builtins directly
scores: list[int] = [90, 85, 100]
prices: dict[str, float] = {"apple": 0.50, "pear": 0.75}
point: tuple[int, int] = (3, 4)          # exactly two ints
flags: set[str] = {"new", "urgent"}

# A tuple of *unknown length*, all the same type: use Ellipsis (...)
row: tuple[int, ...] = (1, 2, 3, 4)      # zero or more ints

# Nesting works as you'd expect
matrix: list[list[float]] = [[1.0, 2.0], [3.0, 4.0]]
users_by_id: dict[int, dict[str, str]] = {1: {"name": "Ada"}}
```

| Old (`typing`, pre-3.9) | Modern (3.9+)        | Meaning                     |
| ----------------------- | -------------------- | --------------------------- |
| `List[int]`             | `list[int]`          | list of ints                |
| `Dict[str, int]`        | `dict[str, int]`     | dict, str keys, int values  |
| `Tuple[int, str]`       | `tuple[int, str]`    | 2-tuple: an int and a str   |
| `Tuple[int, ...]`       | `tuple[int, ...]`    | tuple of any length of ints |
| `Set[str]`              | `set[str]`           | set of strings              |
| `FrozenSet[int]`        | `frozenset[int]`     | frozen set of ints          |

The key distinction with tuples: `tuple[int, str]` means a fixed-length pair (an int *then* a str), while `tuple[int, ...]` means a homogeneous tuple of arbitrary length. This trips people up constantly.

## Core concept: Optional, Union, and the `|` operator

A value that might be a string *or* `None` is extremely common (think "user not found"). There are several ways to express it; the modern one is the pipe `|`.

```python
# Modern (Python 3.10+): the | union operator
def find_user(uid: int) -> str | None:
    return "Ada" if uid == 1 else None

# A union of several real types
def parse(value: int | str | float) -> str:
    return str(value)
```

`Optional[X]` is **exactly** `X | None` — nothing more. It does *not* mean "this argument is optional/has a default"; it means "the value may be `None`". This naming is the single most confusing thing about Python typing for newcomers.

```python
from typing import Optional, Union

# These three lines mean the EXACT same thing:
a: Optional[str]
b: Union[str, None]
c: str | None

# Optional is just sugar for "... | None". Use whichever your codebase prefers;
# modern code increasingly favors `str | None`.
```

A common gotcha: a parameter with a default of `None` should still be typed as nullable.

```python
# WRONG-ish: hint says str, but the default is None
def connect(host: str = None): ...        # type checker complains

# RIGHT: the type reflects reality
def connect(host: str | None = None) -> None:
    host = host or "localhost"
    print(f"connecting to {host}")
```

Note: `Optional` does *not* imply a default value. `def f(x: int | None)` still *requires* you to pass `x` (you just may pass `None`). To make an argument truly optional you must also give it a default: `def f(x: int | None = None)`.

## Core concept: Any (the escape hatch)

`Any` means "turn off type checking for this value". It is compatible with everything in both directions, which makes it powerful and dangerous — overusing `Any` defeats the entire purpose of typing.

```python
from typing import Any

def debug_dump(value: Any) -> str:
    # value could be literally anything; the checker won't object to any use of it
    return repr(value)

config: dict[str, Any] = {"port": 8000, "debug": True, "name": "api"}
# ^ common, legitimate use: a dict whose values are heterogeneous
```

Reach for `Any` only when you genuinely cannot describe the type (truly dynamic data) or while migrating untyped code. Prefer `object` when you mean "anything, but I'll inspect/narrow it safely" — `object` forces you to check before use, whereas `Any` lets everything slide.

## Core concept: Callable — typing functions as values

When you pass a function around (callbacks, dependency injection, decorators), describe it with `Callable[[ArgTypes], ReturnType]`.

```python
from collections.abc import Callable  # preferred import location (3.9+)

# A function taking two ints and returning an int
def apply(op: Callable[[int, int], int], a: int, b: int) -> int:
    return op(a, b)

print(apply(lambda x, y: x + y, 3, 4))   # => 7

# "I don't care about the arguments, only the return type": use ...
Handler = Callable[..., str]

def run(handler: Handler) -> str:
    return handler()

# A callback that returns nothing
on_done: Callable[[], None]
```

`Callable[..., R]` (with a literal `...`) means "any signature, returning `R`". `Callable[[int, str], R]` pins the exact parameter types.

## Core concept: Literal — exact allowed values

`Literal` constrains a value to a specific set of constants. It is perfect for modes, statuses, and string enums where you want the checker to reject typos.

```python
from typing import Literal

Mode = Literal["r", "w", "a"]

def open_file(path: str, mode: Mode = "r") -> None:
    print(f"opening {path} in {mode}")

open_file("data.txt", "w")     # OK
open_file("data.txt", "x")     # type checker error: "x" not allowed

# Literals can mix types
def set_alignment(side: Literal["left", "right", 0, 1]) -> None: ...
```

FastAPI uses `Literal` to render dropdowns of allowed values in Swagger and to reject invalid input automatically.

## Core concept: TypedDict — typing dict shapes

Sometimes data arrives as a plain `dict` (JSON payloads, config) but always has the *same keys*. `TypedDict` lets you describe that shape so the checker knows `user["email"]` is a `str` and `user["age"]` is an `int`.

```python
from typing import TypedDict, NotRequired

class UserDict(TypedDict):
    id: int
    name: str
    email: str
    nickname: NotRequired[str]   # key may be absent (Python 3.11+)

def send_welcome(user: UserDict) -> None:
    print(f"Welcome {user['name']} <{user['email']}>")

ada: UserDict = {"id": 1, "name": "Ada", "email": "ada@example.com"}
send_welcome(ada)               # OK
# missing "email" or a wrong type would be flagged by the checker
```

`TypedDict` is purely a static-checking tool — at runtime `ada` is an ordinary `dict`. When you control the data and want *runtime* validation and methods, prefer a `dataclass` or a Pydantic model instead. Use `TypedDict` mostly for describing external JSON you receive as raw dicts.

## Core concept: Protocol — structural typing (duck typing, checked)

Normally a checker cares about the *named* class. `Protocol` flips this: it says "I accept anything that *has these attributes/methods*, regardless of its class". This is static duck typing — "if it walks like a duck...". It's like a PHP `interface`, except classes don't have to declare they implement it; they just need the right shape.

```python
from typing import Protocol

class SupportsWrite(Protocol):
    def write(self, data: str) -> int: ...

def emit(sink: SupportsWrite, msg: str) -> None:
    sink.write(msg)

import sys
emit(sys.stdout, "hi\n")        # sys.stdout has .write(str) -> int, so it fits

class MemoryLog:
    def __init__(self) -> None:
        self.buf: list[str] = []
    def write(self, data: str) -> int:
        self.buf.append(data)
        return len(data)

emit(MemoryLog(), "logged")     # also fits — never imported SupportsWrite!
```

`Protocol` is how you write functions that accept "anything file-like" or "anything with a `.id`" without forcing callers into a class hierarchy. Add `@runtime_checkable` if you need `isinstance()` checks against it (it then verifies method *names* exist, not their signatures).

## Core concept: TypeVar and Generic — your own generics

A `TypeVar` is a placeholder for "some type, the same throughout this call". It lets you write functions and classes that preserve type information instead of collapsing to `Any`.

```python
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T:
    return items[0]

n = first([1, 2, 3])        # checker knows n is int
s = first(["a", "b"])       # checker knows s is str
```

Without the `TypeVar`, you'd have to type the parameter `list[Any]` and the return `Any`, losing all the information. With it, the output type follows the input.

Python 3.12 introduced cleaner syntax — no explicit `TypeVar` needed:

```python
# Python 3.12+ native generic syntax
def first[T](items: list[T]) -> T:
    return items[0]

# Generic classes, 3.12+ style
class Box[T]:
    def __init__(self, value: T) -> None:
        self._value = value
    def get(self) -> T:
        return self._value

b = Box(42)        # Box[int]
x = b.get()        # checker knows x is int
```

The pre-3.12 way to write a generic class uses `Generic[T]`:

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

s: Stack[int] = Stack()
s.push(1)
value = s.pop()    # checker knows value is int
```

You can also *bound* a TypeVar (`TypeVar("T", bound=Comparable)`) or restrict it to specific types (`TypeVar("T", int, str)`), but the unbounded form covers most needs.

## Core concept: type aliases and NewType

When a type expression gets long or appears repeatedly, give it a name. A plain alias is just an assignment; Python 3.12 adds a dedicated `type` keyword.

```python
# Simple alias (any version)
UserId = int
ConnectionPool = dict[str, list[tuple[str, int]]]

def get_user(uid: UserId) -> str: ...

# Python 3.12+ explicit alias keyword (clearer, lazily evaluated)
type Vector = list[float]
type JSON = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None

def scale(v: Vector, factor: float) -> Vector:
    return [x * factor for x in v]
```

A plain alias (`UserId = int`) is **interchangeable** with the underlying type — passing a raw `int` where a `UserId` is expected is fine. When you want the checker to treat them as *distinct* (so you can't accidentally pass a `ProductId` where a `UserId` belongs), use `NewType`:

```python
from typing import NewType

UserId = NewType("UserId", int)
ProductId = NewType("ProductId", int)

def fetch_user(uid: UserId) -> str: ...

raw = 42
uid = UserId(42)        # wrap to "tag" it
fetch_user(uid)         # OK
fetch_user(raw)         # type checker error: int is not UserId
fetch_user(ProductId(7))# type checker error: ProductId is not UserId
```

At runtime `UserId(42)` is just `42` (zero overhead) — the distinction exists only for the checker. `NewType` is great for preventing "I passed the customer id where the order id goes" classes of bugs.

## Core concept: hints are NOT enforced at runtime

This is the most important mental adjustment for someone coming from PHP's `strict_types`. Python's interpreter **does not check annotations**. They are stored as metadata and otherwise ignored.

```python
def add(a: int, b: int) -> int:
    return a + b

# Python runs this happily and prints "ab" — NO error from the language:
print(add("a", "b"))    # => ab

# Hints are just accessible metadata:
print(add.__annotations__)  # => {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
```

Nothing about `: int` makes Python reject a string. The protection comes from running a **static checker** during development/CI. If you *want* runtime validation, you use a library (Pydantic) or validate by hand — the type hint alone never does it.

## Core concept: static checking with mypy and pyright

The checkers are separate tools you run over your code. The two dominant ones:

- **mypy** — the original, reference type checker (Python package).
- **pyright** — Microsoft's fast checker written in TypeScript; it powers the **Pylance** extension in VS Code, so most VS Code users are already running it live.

```bash
# Install and run mypy
pip install mypy
mypy app/                       # check a directory
mypy --strict app/              # crank up strictness (recommended for new code)
```

```bash
# Install and run pyright (Node-based)
npm install -g pyright
pyright app/
```

Given this file `prices.py`:

```python
def discount(price: float, pct: float) -> float:
    return price - (price * pct)

discount("10.00", 0.2)   # passing a str where float is expected
```

mypy reports the bug *without running the code*:

```bash
$ mypy prices.py
prices.py:4: error: Argument 1 to "discount" has incompatible type "str"; expected "float"  [arg-type]
Found 1 error in 1 file (checked 1 source file)
```

Configure mypy in `pyproject.toml` so the whole team uses the same rules:

```toml
[tool.mypy]
python_version = "3.12"
strict = true                 # turns on a bundle of strict flags
warn_unused_ignores = true
```

## Core concept: from __future__ import annotations and forward references

Sometimes you must reference a type that isn't defined yet — most commonly a class referring to *itself*, or two classes referring to each other. Writing the bare name fails at runtime because the name doesn't exist when the line executes.

```python
# Problem: Node doesn't exist yet on the line that mentions it
class Node:
    def __init__(self, value: int, next: Node) -> None:  # NameError at class-build time!
        self.value = value
        self.next = next
```

Two fixes:

**1. String (forward reference) — quote the type:**

```python
class Node:
    def __init__(self, value: int, next: "Node | None" = None) -> None:
        self.value = value
        self.next = next
```

**2. `from __future__ import annotations` — make ALL annotations strings automatically:**

```python
from __future__ import annotations   # must be the first import

class Node:
    def __init__(self, value: int, next: Node | None = None) -> None:
        self.value = value          # no quotes needed anywhere now
        self.next = next
```

With that future import, every annotation in the file is stored as a string and never evaluated at runtime ("lazy annotations"). Benefits: no forward-reference errors, slightly faster imports, and you can use modern `X | None` syntax even on older Python versions. Caveat: libraries that *read* annotations at runtime (older Pydantic, some dataclass tricks) may need `typing.get_type_hints()` to resolve them — modern Pydantic v2 and FastAPI handle this correctly, so the future import is safe and commonly recommended.

## Core concept: how FastAPI turns hints into validation + docs

This is the payoff. FastAPI *reads* your function's type hints and uses them to (1) parse and validate incoming requests, (2) convert/serialize responses, and (3) generate interactive OpenAPI/Swagger docs — all automatically.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Literal

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float
    in_stock: bool = True
    tags: list[str] = []

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None) -> dict[str, object]:
    # item_id is parsed from the path AND validated as int — "abc" => 422 error
    # q is an OPTIONAL query string because it's typed `str | None` with a default
    return {"item_id": item_id, "q": q}

@app.post("/items")
def create_item(item: Item, mode: Literal["draft", "live"] = "draft") -> Item:
    # `item: Item` => FastAPI reads the JSON body, validates each field's type,
    # rejects bad input with a 422 + detailed error, and gives you a typed object.
    # `mode` becomes a query param constrained to "draft" or "live".
    return item
```

What each hint does for you:

| Hint in your code            | What FastAPI/Pydantic does                                   |
| ---------------------------- | ----------------------------------------------------------- |
| `item_id: int` (path)        | Parses path segment, coerces to int, 422 if not numeric     |
| `q: str \| None = None`      | Optional query parameter, defaults to `None`                |
| `item: Item` (Pydantic)      | Reads request body JSON, validates every field, types it    |
| `mode: Literal["draft",...]` | Restricts allowed values; shows a dropdown in Swagger       |
| `-> Item` (return)           | Validates/serializes the response, documents the schema     |
| field default `= True`/`[]`  | Marks fields optional with that default in the schema       |

In short: in FastAPI, the type hint *is* the validation rule and *is* the API documentation. The same annotations a static checker reads, FastAPI also reads — but FastAPI additionally enforces them at runtime via Pydantic. This is why everything you learned above is load-bearing, not cosmetic.

## Common mistakes & gotchas

- **Expecting runtime enforcement.** Plain `def f(x: int)` will happily accept a string. Only Pydantic/FastAPI (or manual checks) enforce hints at runtime.
- **Confusing `Optional` with "has a default".** `Optional[int]` / `int | None` means *the value may be None*, not *the argument may be omitted*. Omit-ability comes from giving a default (`= None`).
- **`tuple[int]` vs `tuple[int, ...]`.** The first is a *one-element* tuple of an int; the second is *any-length*. For "a tuple of ints of unknown size" you need the `...`.
- **Using deprecated `typing.List`/`Dict`/`Tuple` in new code.** Use built-in `list[...]`, `dict[...]`, `tuple[...]` (3.9+).
- **`x: int = None`.** The default contradicts the hint; checkers flag it. Use `x: int | None = None`.
- **Mutable default arguments.** `def f(items: list[int] = [])` shares one list across calls (a classic Python bug). Use `= None` and create the list inside, or rely on Pydantic field defaults.
- **Forward-reference NameErrors.** A class referencing itself/another not-yet-defined class needs quotes or `from __future__ import annotations`.
- **Overusing `Any`.** It silently switches off checking. Prefer a precise type, a `Union`, or `object` when you'll narrow it.
- **Thinking `# type: ignore` fixes anything.** It only silences the checker; the underlying issue remains. Use sparingly and add the specific error code.
- **Assuming `List`/`list` matters at runtime.** Subscripting generics (`list[int]`) is for the checker; it does not validate element types when the program runs.

## Best practices

- Type all **public** function signatures (params + return). Internal locals usually don't need annotations — the checker infers them.
- Prefer modern syntax: `list[int]`, `str | None`, `X | Y` over `List`, `Optional`, `Union` in new code.
- Annotate `-> None` explicitly for functions that return nothing.
- Run a checker in CI (`mypy --strict` or `pyright`). Treat type errors like failing tests.
- Reach for `Protocol` to accept "anything shaped like this" instead of forcing a base class; reach for `TypedDict` for external JSON-as-dict.
- Use `NewType` to keep semantically different ids (`UserId` vs `OrderId`) from being mixed.
- Keep complex types readable with aliases (`type JSON = ...`).
- Let FastAPI/Pydantic do runtime validation; let mypy/pyright do static checking. They are complementary, not redundant.
- Import `Callable` from `collections.abc`, not `typing`, in modern code.

## Interview Q&A

**Q: Are Python type hints enforced at runtime?**
**A:** No. The CPython interpreter ignores annotations during execution — `def f(x: int)` will accept any value. Hints are metadata (stored in `__annotations__`) used by static checkers (mypy/pyright) and by libraries that *choose* to read them (Pydantic, FastAPI). Runtime type enforcement only happens when such a library validates the data; the language itself never does.

**Q: What's the difference between `Optional[int]` and `int | None`?**
**A:** They are identical. `Optional[X]` is defined as exactly `X | None`. The `|` form (Python 3.10+) is the modern, preferred spelling. Neither has anything to do with whether the argument has a default value.

**Q: Does `Optional` mean the argument is optional?**
**A:** No — that's the most common misconception. `Optional[X]` means the *value* may be `None`. To make an argument omittable you must give it a default, e.g. `def f(x: int | None = None)`. You can also have a required `Optional`: `def f(x: int | None)` still demands you pass something (possibly `None`).

**Q: What's the difference between `tuple[int, str]` and `tuple[int, ...]`?**
**A:** `tuple[int, str]` is a fixed two-element tuple: an int followed by a str. `tuple[int, ...]` (with literal Ellipsis) is a homogeneous tuple of *any* length whose elements are all ints.

**Q: When would you use `Protocol`?**
**A:** When you want structural (duck) typing: a function accepts any object that has the required methods/attributes, regardless of its class or inheritance. It's like an interface that classes satisfy implicitly — they don't need to declare or import it. Great for "file-like" or "anything with `.id`" parameters.

**Q: `TypedDict` vs `dataclass` vs Pydantic `BaseModel` — when to use which?**
**A:** `TypedDict` describes the shape of a plain `dict` for static checking only (no runtime validation, no methods) — ideal for external JSON you receive as raw dicts. A `dataclass` gives you a real class with attributes and methods but still no runtime type validation. A Pydantic `BaseModel` adds *runtime* parsing/validation and serialization — use it for API boundaries and config.

**Q: What does `Any` do, and why avoid it?**
**A:** `Any` disables type checking for that value — it's compatible with everything both ways. Overusing it silently turns off the safety you added hints for. Prefer a precise type, a union, or `object` (which forces you to narrow before use).

**Q: Why does FastAPI rely so heavily on type hints?**
**A:** FastAPI reads your annotations to derive request validation, response serialization, and OpenAPI/Swagger docs automatically. A path param typed `int` is coerced and validated; a body typed as a Pydantic model is parsed and validated field-by-field with 422 errors on bad input; a `Literal` becomes an enumerated constraint. The hint simultaneously serves as the validation rule and the documentation.

**Q: What's `from __future__ import annotations` for?**
**A:** It makes all annotations in a module evaluate lazily — stored as strings instead of being executed at definition time. This eliminates forward-reference errors (a class can mention itself unquoted), speeds imports slightly, and lets you use newer syntax like `X | None` on older Python. Tools that read annotations at runtime may need `typing.get_type_hints()`; modern Pydantic v2/FastAPI handle it fine.

**Q: What's the difference between a type alias and `NewType`?**
**A:** A plain alias (`UserId = int`) is fully interchangeable with the underlying type — the checker treats `UserId` and `int` as the same. `NewType("UserId", int)` creates a *distinct* type for the checker so you can't accidentally pass a different `int`-based id; at runtime it's a no-op (just the int). Use `NewType` to prevent mixing semantically different values.

**Q: What is a `TypeVar` and why use it?**
**A:** A `TypeVar` is a placeholder representing "some type, consistent within a single call". It lets generic functions/classes preserve type information: `def first[T](xs: list[T]) -> T` returns the *same* type the list holds, so the checker knows `first([1,2])` is an `int`. Without it you'd fall back to `Any` and lose all type info.

**Q: mypy vs pyright — what's the difference?**
**A:** Both are static type checkers. mypy is the original reference implementation (a Python package). pyright is Microsoft's faster checker (written in TypeScript) and powers VS Code's Pylance, so it often runs live as you type. They overlap heavily; pick one for CI. Many teams use pyright in the editor and either tool in CI.

## Practice exercises

1. Write a function `parse_csv_line(line: str) -> list[str]` that splits a comma-separated string into trimmed fields. Add full annotations and a `-> None` helper `print_fields(fields: list[str]) -> None`.

2. Annotate this function correctly so a type checker is happy, given it may return `None` when the key is absent:
   ```python
   def get_setting(config, key):
       return config.get(key)
   ```
   Assume `config` is a dict from `str` to `int`.

3. Define a `TypedDict` named `Product` with `id: int`, `name: str`, `price: float`, and an optional `discount: float`. Write `final_price(p: Product) -> float` that applies the discount if present.

4. Write a `Protocol` named `Identifiable` requiring an attribute `id: int`, then a function `ids_of(items: list[Identifiable]) -> list[int]`. Show it works with two unrelated classes that each have an `id`.

5. Use `NewType` to define `Celsius` and `Fahrenheit` (both based on `float`), then a function `to_fahrenheit(c: Celsius) -> Fahrenheit`. Demonstrate that passing a raw `float` or a `Fahrenheit` where `Celsius` is expected is a type error.

6. Write a generic function using Python 3.12 syntax: `def last[T](items: list[T]) -> T` returning the final element, and show that the checker infers `int` for `last([1, 2, 3])`.

<details>
<summary>Solutions</summary>

```python
# 1.
def parse_csv_line(line: str) -> list[str]:
    return [field.strip() for field in line.split(",")]

def print_fields(fields: list[str]) -> None:
    for f in fields:
        print(f)

print_fields(parse_csv_line("  a , b ,c "))   # => a / b / c on separate lines


# 2.
def get_setting(config: dict[str, int], key: str) -> int | None:
    return config.get(key)   # .get returns None when key is missing


# 3.
from typing import TypedDict, NotRequired

class Product(TypedDict):
    id: int
    name: str
    price: float
    discount: NotRequired[float]   # key may be absent

def final_price(p: Product) -> float:
    return p["price"] * (1 - p.get("discount", 0.0))

print(final_price({"id": 1, "name": "Pen", "price": 2.0, "discount": 0.5}))  # => 1.0
print(final_price({"id": 2, "name": "Cup", "price": 4.0}))                    # => 4.0


# 4.
from typing import Protocol

class Identifiable(Protocol):
    id: int

def ids_of(items: list[Identifiable]) -> list[int]:
    return [item.id for item in items]

class User:
    def __init__(self, id: int) -> None:
        self.id = id

class Order:
    def __init__(self, id: int, total: float) -> None:
        self.id = id
        self.total = total

print(ids_of([User(1), Order(2, 9.99)]))   # => [1, 2] — unrelated classes both fit


# 5.
from typing import NewType

Celsius = NewType("Celsius", float)
Fahrenheit = NewType("Fahrenheit", float)

def to_fahrenheit(c: Celsius) -> Fahrenheit:
    return Fahrenheit(c * 9 / 5 + 32)

temp = Celsius(100.0)
print(to_fahrenheit(temp))        # => 212.0
# to_fahrenheit(100.0)            # type error: float is not Celsius
# to_fahrenheit(Fahrenheit(50.0)) # type error: Fahrenheit is not Celsius


# 6.
def last[T](items: list[T]) -> T:    # Python 3.12+ generic syntax
    return items[-1]

n = last([1, 2, 3])   # checker infers n: int
s = last(["x", "y"])  # checker infers s: str
print(n, s)           # => 3 y
```

</details>

## Cheat sheet

```python
# --- Basics ---
x: int = 5
def f(a: str, b: bool = False) -> str: ...
def g() -> None: ...                     # returns nothing

# --- Built-in generics (3.9+) ---
list[int]            # list of ints
dict[str, float]     # str -> float
tuple[int, str]      # fixed 2-tuple
tuple[int, ...]      # any-length tuple of ints
set[str]; frozenset[int]

# --- Optional / Union ---
str | None           # modern; == Optional[str]
int | str | float    # modern; == Union[int, str, float]

# --- Escape hatch ---
Any                  # disables checking (use sparingly)
object               # anything, but must narrow before use

# --- Callables ---
Callable[[int, int], int]   # (int, int) -> int
Callable[..., str]          # any args -> str

# --- Constants & shapes ---
Literal["r", "w", "a"]      # exact allowed values
class TD(TypedDict): id: int; name: str   # dict shape (static only)

# --- Duck typing & generics ---
class P(Protocol):          # structural typing
    def write(self, s: str) -> int: ...
def first[T](xs: list[T]) -> T: ...        # 3.12 generic fn
class Box[T]: ...                          # 3.12 generic class

# --- Aliases / distinct types ---
type Vector = list[float]                  # 3.12 alias keyword
UserId = NewType("UserId", int)            # distinct from int

# --- Forward references ---
from __future__ import annotations         # lazy annotations (first import)
next: "Node | None"                        # or quote the name
```

| Need                                   | Use                                  |
| -------------------------------------- | ------------------------------------ |
| Value may be None                      | `X \| None` (a.k.a. `Optional[X]`)   |
| One of several types                   | `A \| B` (a.k.a. `Union[A, B]`)      |
| Exact allowed constants                | `Literal[...]`                       |
| Dict with known keys (static only)     | `TypedDict`                          |
| Accept "anything shaped like this"     | `Protocol`                           |
| Reusable generic fn/class              | `TypeVar` / `[T]` (3.12) / `Generic` |
| Distinct id types                      | `NewType`                            |
| Pass a function                        | `Callable[[args], ret]`              |
| Turn off checking                      | `Any`                                |
| Runtime validation                     | Pydantic / FastAPI (not plain hints) |

**Remember:** Python ignores hints at runtime — mypy/pyright check them statically, and FastAPI/Pydantic enforce them at the API boundary.
