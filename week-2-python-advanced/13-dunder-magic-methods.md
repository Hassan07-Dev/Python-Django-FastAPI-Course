# 13 — Dunder / Magic Methods & Dataclasses

> Week 2 · Python advanced (the object model)

## What you will learn

- What "dunder" (double-underscore) methods are, and how Python calls them *implicitly* on your behalf.
- The big four: `__init__`, `__repr__`, `__str__`, and how `repr` vs `str` differ.
- Value semantics: `__eq__` + `__hash__` (and the iron rule: override them **together**).
- Ordering with `__lt__`, `__le__`, … and the `@functools.total_ordering` shortcut.
- Truthiness and size: `__len__`, `__bool__`, `__contains__`.
- The container protocol: `__getitem__`, `__setitem__`, `__delitem__`.
- A preview of iteration (`__iter__` / `__next__`) and context managers (`__enter__` / `__exit__`).
- Callable objects with `__call__`.
- Operator overloading: `__add__`, `__radd__`, `__mul__`, `__eq__`, etc.
- `@dataclass` in depth — the modern way to kill boilerplate (`fields`, defaults, `default_factory`, `frozen`, `order`, `__post_init__`).
- `__slots__` for memory and attribute discipline.

## PHP/Laravel bridge

PHP also has "magic methods" — you already know `__construct`, `__toString`, `__get`, `__set`, `__call`, `__invoke`, `ArrayAccess` (`offsetGet`/`offsetSet`), and `Countable::count()`. Python's dunder methods are the exact same idea but far more pervasive and consistent: nearly every operator and built-in (`len()`, `==`, `+`, `in`, `for`, `with`) dispatches to a dunder method, so you can make your own classes feel like native types. Where PHP relies on interfaces (`ArrayAccess`, `Countable`, `Stringable`), Python uses *duck typing* — implement the right dunder and it just works, no interface declaration needed. And where you'd reach for Laravel's `Data` objects or a hand-written DTO with a constructor and getters, Python gives you `@dataclass`: one decorator that generates `__init__`, `__repr__`, `__eq__`, and more for free — think of it as a built-in Spatie Laravel-Data.

## Core concept: what is a dunder method?

A **dunder** ("double underscore") method is a method whose name starts and ends with `__`, like `__init__` or `__len__`. They are also called **magic methods** or **special methods**. You almost never call them directly. Instead, Python's syntax and built-in functions call them *for you*:

| You write        | Python actually calls        |
|------------------|------------------------------|
| `obj = Foo(1)`   | `Foo.__init__(obj, 1)`       |
| `len(obj)`       | `obj.__len__()`              |
| `str(obj)`       | `obj.__str__()`              |
| `repr(obj)`      | `obj.__repr__()`             |
| `a + b`          | `a.__add__(b)`               |
| `a == b`         | `a.__eq__(b)`                |
| `x in obj`       | `obj.__contains__(x)`        |
| `obj[k]`         | `obj.__getitem__(k)`         |
| `for x in obj:`  | `obj.__iter__()` then `__next__()` |
| `with obj:`      | `obj.__enter__()` / `__exit__()` |
| `obj()`          | `obj.__call__()`             |

This is what people mean when they say "everything in Python is an object" — the operators are just sugar over method calls. Implement the protocol (the right set of dunders) and your object plugs straight into the language.

```python
class Distance:
    def __init__(self, meters: float) -> None:
        self.meters = meters

d = Distance(100)
print(d.meters)  # => 100
```

`__init__` is the initializer — the closest analog to PHP's `__construct`. (Technically `__new__` *creates* the instance and `__init__` *initializes* it, but you rarely touch `__new__`.)

## Core concept: `__repr__` vs `__str__`

These two trip up every newcomer. Both produce a string, but they have different audiences.

- **`__repr__`** is for *developers*. It should be unambiguous and, ideally, look like valid Python that could recreate the object. It is what you see in the REPL, in a debugger, and inside containers (`[obj1, obj2]`).
- **`__str__`** is for *end users*. It should be readable/pretty. It is what `print()` and `str()` and f-strings use.

```python
class User:
    def __init__(self, name: str, age: int) -> None:
        self.name = name
        self.age = age

    def __repr__(self) -> str:
        return f"User(name={self.name!r}, age={self.age!r})"

    def __str__(self) -> str:
        return f"{self.name} ({self.age})"

u = User("Ada", 36)
print(u)            # => Ada (36)          -> uses __str__
print(str(u))       # => Ada (36)          -> __str__
print(repr(u))      # => User(name='Ada', age=36)  -> __repr__
print([u])          # => [User(name='Ada', age=36)] -> containers use __repr__
print(f"{u}")       # => Ada (36)          -> f-string uses __str__
print(f"{u!r}")     # => User(name='Ada', age=36)   -> !r forces __repr__
```

Key fallback rule: **if you define only `__repr__`, Python uses it for `str()` too.** If you define only `__str__`, `repr()` falls back to the ugly default `<User object at 0x...>`. So if you implement only one, **implement `__repr__`** — it covers more cases.

The `!r` conversion inside an f-string means "use repr of this value." Using `{self.name!r}` instead of `{self.name}` is why the output shows `'Ada'` with quotes — exactly what you want in a repr.

## Core concept: equality (`__eq__`) and hashing (`__hash__`)

By default, `==` on custom objects compares *identity* (same object in memory), just like PHP's `===` on objects. To compare by *value*, override `__eq__`.

```python
class Money:
    def __init__(self, amount: int, currency: str) -> None:
        self.amount = amount
        self.currency = currency

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented  # let Python try other.__eq__ / fall back
        return self.amount == other.amount and self.currency == other.currency

a = Money(100, "USD")
b = Money(100, "USD")
print(a == b)   # => True   (value equality)
print(a is b)   # => False  (different objects)
```

Returning the special `NotImplemented` (not `NotImplementedError`!) when the types don't match is the polite move: it tells Python "I don't know how to compare these," so it can try the reflected operation or fall back to identity.

### The iron rule: override `__eq__` and `__hash__` together

Here is the gotcha that bites everyone. The moment you define `__eq__`, Python **sets `__hash__` to `None`**, making your instances *unhashable* — you can't put them in a `set` or use them as `dict` keys.

```python
a = Money(100, "USD")
s = {a}   # => TypeError: unhashable type: 'Money'
```

Why? Because hashing and equality have a contract: **objects that are equal must have the same hash.** If you let equality change but kept the old identity-based hash, a `set`/`dict` could store two "equal" objects without noticing they're equal — breaking lookups. Python protects you by disabling hashing until you opt back in.

Fix it by defining `__hash__` over the *same fields* you used in `__eq__`:

```python
class Money:
    def __init__(self, amount: int, currency: str) -> None:
        self.amount = amount
        self.currency = currency

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __hash__(self) -> int:
        return hash((self.amount, self.currency))  # hash a tuple of the key fields

a = Money(100, "USD")
b = Money(100, "USD")
print({a, b})        # => {Money(...)}  -> deduplicated, only one element
print(len({a, b}))   # => 1
```

The standard trick: build a tuple of the identifying fields and hash *that*. Tuples are hashable if their contents are.

**Rule of thumb:** only objects that are logically *immutable* should be hashable. If `amount` can be mutated after creation, its hash would change and break any set/dict it lives in. This is exactly why `frozen` dataclasses (below) are the clean way to get hashable value objects.

## Core concept: ordering with rich comparisons + `total_ordering`

To make objects sortable / comparable with `<`, `<=`, `>`, `>=`, implement the **rich comparison** dunders:

| Operator | Dunder      |
|----------|-------------|
| `<`      | `__lt__`    |
| `<=`     | `__le__`    |
| `>`      | `__gt__`    |
| `>=`     | `__ge__`    |
| `==`     | `__eq__`    |
| `!=`     | `__ne__`    |

Writing all six is tedious. `functools.total_ordering` fills in the rest from just `__eq__` plus **one** of `__lt__`/`__le__`/`__gt__`/`__ge__`:

```python
from functools import total_ordering

@total_ordering
class Version:
    def __init__(self, major: int, minor: int) -> None:
        self.major = major
        self.minor = minor

    def _key(self) -> tuple[int, int]:
        return (self.major, self.minor)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Version):
            return NotImplemented
        return self._key() == other._key()

    def __lt__(self, other: "Version") -> bool:
        if not isinstance(other, Version):
            return NotImplemented
        return self._key() < other._key()

    def __repr__(self) -> str:
        return f"Version({self.major}, {self.minor})"

versions = [Version(1, 4), Version(1, 2), Version(2, 0)]
print(sorted(versions))
# => [Version(1, 2), Version(1, 4), Version(2, 0)]
print(Version(1, 2) <= Version(1, 4))  # => True  (synthesized by total_ordering)
```

Comparing a *tuple* of fields is the idiomatic Python way to get lexicographic ordering for free — `(major, minor)` sorts by major first, then minor, exactly like a version number should.

## Core concept: `__len__`, `__bool__`, `__contains__`

These make your object behave like a built-in collection.

- **`__len__`** powers `len(obj)`. It must return a non-negative `int`.
- **`__bool__`** powers truthiness (`if obj:`). If you don't define it, Python falls back to `__len__` (zero length = falsy). If neither exists, every instance is truthy.
- **`__contains__`** powers the `in` operator.

```python
class Playlist:
    def __init__(self, songs: list[str]) -> None:
        self.songs = songs

    def __len__(self) -> int:
        return len(self.songs)

    def __bool__(self) -> bool:
        return len(self.songs) > 0

    def __contains__(self, song: str) -> bool:
        return song in self.songs

p = Playlist(["Take Five", "So What"])
print(len(p))                 # => 2
print(bool(p))                # => True
print("So What" in p)         # => True
print("Imagine" in p)         # => False

empty = Playlist([])
if not empty:                 # uses __bool__ (or __len__ if __bool__ absent)
    print("Playlist is empty")  # => Playlist is empty
```

Without `__contains__`, Python would *still* support `in` by iterating (if the object is iterable) — but an explicit `__contains__` is faster and clearer.

## Core concept: the container protocol (`__getitem__`, `__setitem__`, `__delitem__`)

These give you `obj[key]` square-bracket access — the equivalent of PHP's `ArrayAccess` (`offsetGet` / `offsetSet` / `offsetUnset`).

```python
class Headers:
    """A case-insensitive HTTP header bag."""
    def __init__(self) -> None:
        self._data: dict[str, str] = {}

    def __getitem__(self, key: str) -> str:
        return self._data[key.lower()]

    def __setitem__(self, key: str, value: str) -> None:
        self._data[key.lower()] = value

    def __delitem__(self, key: str) -> None:
        del self._data[key.lower()]

    def __contains__(self, key: str) -> bool:
        return key.lower() in self._data

    def __repr__(self) -> str:
        return f"Headers({self._data!r})"

h = Headers()
h["Content-Type"] = "application/json"   # __setitem__
print(h["content-type"])                  # => application/json   (__getitem__, case-insensitive)
print("CONTENT-TYPE" in h)                # => True               (__contains__)
del h["Content-Type"]                     # __delitem__
print("content-type" in h)                # => False
```

Bonus: `__getitem__` also enables slicing — your method receives a `slice` object for `obj[1:3]`. And historically, defining `__getitem__` alone makes an object iterable even without `__iter__` (Python calls it with `0, 1, 2, …` until `IndexError`). Prefer `__iter__` for real iteration, though.

## Core concept: iteration preview (`__iter__` / `__next__`)

`for x in obj:` works if `obj.__iter__()` returns an **iterator** — an object with a `__next__()` method that yields the next value and raises `StopIteration` when exhausted. (Full coverage lives in the iterators/generators lesson; here's the shape.)

```python
class CountDown:
    def __init__(self, start: int) -> None:
        self.start = start

    def __iter__(self):
        # The cleanest way: be a generator. `yield` makes this return an iterator.
        n = self.start
        while n > 0:
            yield n
            n -= 1

for n in CountDown(3):
    print(n)   # => 3, then 2, then 1
print(list(CountDown(3)))  # => [3, 2, 1]
```

The manual (non-generator) form spells out the protocol explicitly:

```python
class CountDownManual:
    def __init__(self, start: int) -> None:
        self.current = start

    def __iter__(self) -> "CountDownManual":
        return self          # the object is its own iterator

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration   # signals "done" to the for-loop
        value = self.current
        self.current -= 1
        return value

print(list(CountDownManual(3)))  # => [3, 2, 1]
```

Returning `self` from `__iter__` makes this single-use (it's exhausted after one loop). The generator version is almost always nicer.

## Core concept: callable objects (`__call__`)

Define `__call__` and instances become callable like functions — exactly PHP's `__invoke`. Useful for stateful "functions," configurable strategies, decorators-as-classes, and DI-style services.

```python
class Multiplier:
    def __init__(self, factor: int) -> None:
        self.factor = factor

    def __call__(self, x: int) -> int:
        return x * self.factor

double = Multiplier(2)
triple = Multiplier(3)
print(double(10))           # => 20   -> calls __call__
print(triple(10))           # => 30
print(callable(double))     # => True
print([double(n) for n in range(4)])  # => [0, 2, 4, 6]
```

A callable object beats a closure when you need to *inspect* or *reconfigure* the state later (`double.factor = 5`) or give it a clean `__repr__`.

## Core concept: operator overloading (`__add__`, `__radd__`, `__mul__`, …)

Arithmetic and bitwise operators all map to dunders. Implement them to make math-like value types (vectors, money, matrices) read naturally.

| Operator | Dunder        | Reflected (right-hand) |
|----------|---------------|------------------------|
| `+`      | `__add__`     | `__radd__`             |
| `-`      | `__sub__`     | `__rsub__`             |
| `*`      | `__mul__`     | `__rmul__`             |
| `/`      | `__truediv__` | `__rtruediv__`         |
| `//`     | `__floordiv__`| `__rfloordiv__`        |
| `%`      | `__mod__`     | `__rmod__`             |
| `**`     | `__pow__`     | `__rpow__`             |
| `+=`     | `__iadd__`    | —                      |

```python
class Vector:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __add__(self, other: "Vector") -> "Vector":
        if not isinstance(other, Vector):
            return NotImplemented
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar: float) -> "Vector":
        return Vector(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar: float) -> "Vector":
        # Called for `3 * v` when int.__mul__(3, v) returns NotImplemented.
        return self.__mul__(scalar)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector):
            return NotImplemented
        return (self.x, self.y) == (other.x, other.y)

    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"

v = Vector(1, 2) + Vector(3, 4)
print(v)            # => Vector(4, 6)
print(v * 2)        # => Vector(8, 12)   -> __mul__
print(3 * Vector(1, 1))  # => Vector(3, 3)  -> __rmul__ (left operand is int, doesn't know Vector)
```

**Why reflected (`__r*`) methods exist:** in `3 * v`, Python first asks `int.__mul__(3, v)`. The `int` has no idea what a `Vector` is, so it returns `NotImplemented`. Python then tries the *reflected* method on the right operand: `v.__rmul__(3)`. Without `__rmul__`, `3 * v` would raise `TypeError` even though `v * 3` works.

## Core concept: context managers preview (`__enter__` / `__exit__`)

The `with` statement guarantees setup/teardown even if an exception fires — like Laravel's `DB::transaction()` closure or a `try/finally`. (Deep dive in the context-managers lesson.)

```python
class Timer:
    def __enter__(self) -> "Timer":
        import time
        self.start = time.perf_counter()
        return self            # bound to the `as` variable

    def __exit__(self, exc_type, exc_value, traceback) -> bool:
        import time
        self.elapsed = time.perf_counter() - self.start
        print(f"Took {self.elapsed:.4f}s")
        return False           # False = don't suppress exceptions; re-raise them

with Timer() as t:
    total = sum(range(1_000_000))
# => Took 0.01XYs   (printed by __exit__, runs even if the block raised)
```

`__exit__` receives the exception info (all `None` on a clean exit). Returning a truthy value *suppresses* the exception — return `False`/`None` to let it propagate.

## Core concept: `@dataclass` — the modern way to kill boilerplate

Writing `__init__`, `__repr__`, and `__eq__` by hand for every data-holding class is tedious. `@dataclass` (from the standard library, Python 3.7+) generates them from your type-annotated fields. This is the Pythonic DTO / value object, and the spiritual cousin of Spatie's Laravel-Data.

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p = Point(1, 2)
print(p)              # => Point(x=1, y=2)   -> generated __repr__
print(p == Point(1, 2))  # => True            -> generated __eq__ (value equality!)
print(p.x, p.y)       # => 1 2
```

That decorator generated `__init__(self, x, y)`, `__repr__`, and `__eq__` automatically. The annotations (`x: int`) are **required** — they're how dataclass discovers the fields.

### Defaults and the `default_factory` trap

```python
from dataclasses import dataclass, field

@dataclass
class Config:
    host: str = "localhost"           # simple default
    port: int = 8000
    tags: list[str] = field(default_factory=list)   # MUST use default_factory for mutables

c1 = Config()
c2 = Config()
c1.tags.append("prod")
print(c1.tags)   # => ['prod']
print(c2.tags)   # => []    -> each instance gets its OWN list
print(c1)        # => Config(host='localhost', port=8000, tags=['prod'])
```

**Why `default_factory`?** A bare mutable default like `tags: list = []` is the classic Python footgun — the *same list object* would be shared across every instance (the "mutable default argument" bug). Dataclasses **forbid** it outright:

```python
@dataclass
class Bad:
    tags: list = []     # => ValueError: mutable default <class 'list'> ... use default_factory
```

`field(default_factory=list)` tells the generated `__init__` to call `list()` fresh for each new instance. Use it for any `list`, `dict`, `set`, or other mutable default.

Fields with defaults must come *after* fields without them — same rule as function parameters.

### `frozen` (immutable + hashable) and `order` (sortable)

```python
from dataclasses import dataclass

@dataclass(frozen=True, order=True)
class Money:
    amount: int
    currency: str

m = Money(100, "USD")
print(m == Money(100, "USD"))   # => True
print({m, Money(100, "USD")})   # => {Money(amount=100, currency='USD')}  -> frozen => hashable!
print(Money(50, "USD") < Money(100, "USD"))  # => True  -> order= generated __lt__ etc.

m.amount = 200   # => dataclasses.FrozenInstanceError: cannot assign to field 'amount'
```

- **`frozen=True`** makes instances read-only (assignment raises `FrozenInstanceError`) and — crucially — auto-generates `__hash__`, so the object is usable in sets/dict keys. This is the clean replacement for hand-writing `__eq__` + `__hash__`. Perfect for value objects.
- **`order=True`** generates `__lt__`, `__le__`, `__gt__`, `__ge__` by comparing fields *in definition order* as a tuple — so put your most-significant field first.

A handy table of the most-used `@dataclass` parameters:

| Parameter        | Default | What it does                                                |
|------------------|---------|-------------------------------------------------------------|
| `init`           | `True`  | Generate `__init__`                                          |
| `repr`           | `True`  | Generate `__repr__`                                          |
| `eq`             | `True`  | Generate `__eq__`                                            |
| `order`          | `False` | Generate `__lt__`/`__le__`/`__gt__`/`__ge__`                 |
| `frozen`         | `False` | Make instances immutable + hashable                         |
| `slots`          | `False` | Generate `__slots__` (3.10+) — less memory, no `__dict__`   |
| `kw_only`        | `False` | Make all fields keyword-only (3.10+)                        |

### `__post_init__` for validation / derived fields

The generated `__init__` just assigns fields. To validate or compute derived values, define `__post_init__`, which runs *after* `__init__`.

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)   # not a constructor arg; computed below

    def __post_init__(self) -> None:
        if self.width <= 0 or self.height <= 0:
            raise ValueError("dimensions must be positive")
        self.area = self.width * self.height

r = Rectangle(3, 4)
print(r)            # => Rectangle(width=3, height=4, area=12)
print(r.area)       # => 12
Rectangle(-1, 5)    # => ValueError: dimensions must be positive
```

`field(init=False)` keeps `area` out of the constructor signature; `__post_init__` fills it in. (For a frozen dataclass you'd assign derived fields via `object.__setattr__(self, "area", ...)` since normal assignment is blocked.)

### Useful dataclass helpers

```python
from dataclasses import dataclass, asdict, astuple, replace, fields

@dataclass
class User:
    name: str
    age: int

u = User("Ada", 36)
print(asdict(u))                 # => {'name': 'Ada', 'age': 36}
print(astuple(u))                # => ('Ada', 36)
print(replace(u, age=37))        # => User(name='Ada', age=37)  -> copy with a change (great for frozen)
print([f.name for f in fields(u)])  # => ['name', 'age']
```

`replace()` is the immutable-friendly "update" — it returns a new instance instead of mutating, perfect for `frozen` objects.

> Note: `@dataclass` is plain-Python data plumbing. **Pydantic** (covered later) looks similar but adds runtime *validation and parsing* (great for API request bodies). Rule of thumb: `@dataclass` for internal value objects; Pydantic when data crosses a trust boundary (HTTP, files, env).

## Core concept: `__slots__` for memory and attribute discipline

By default each instance stores its attributes in a per-instance `__dict__`, which is flexible but memory-hungry. `__slots__` replaces that dict with a fixed, compact layout — less memory, slightly faster attribute access, and it *forbids* adding undeclared attributes (catching typos).

```python
class PointSlots:
    __slots__ = ("x", "y")        # fixed set of allowed attributes; no per-instance __dict__

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

p = PointSlots(1, 2)
print(p.x, p.y)    # => 1 2
p.z = 3            # => AttributeError: 'PointSlots' object has no attribute 'z'
# print(p.__dict__)  # => AttributeError: no __dict__ at all
```

For thousands or millions of small objects (e.g. parsing rows), `__slots__` can cut memory dramatically. With dataclasses you get it via `@dataclass(slots=True)`:

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Pixel:
    r: int
    g: int
    b: int

px = Pixel(255, 0, 0)
print(px)          # => Pixel(r=255, g=0, b=0)
# px.alpha = 128   # => AttributeError -> slots block undeclared attrs
```

Trade-offs: no dynamic attributes, and multiple inheritance with slots needs care. Use `__slots__` when you have many instances and a stable set of fields — not on every class.

## Common mistakes & gotchas

- **Defining `__eq__` without `__hash__`.** This silently makes instances unhashable (sets/dict keys break). Override both, or use `@dataclass(frozen=True)` / `@dataclass(eq=True)` and let it manage hashing.
- **Confusing `__str__` and `__repr__`.** If you implement only one, implement `__repr__` — it's the universal fallback. `__str__`-only leaves `repr()` ugly.
- **Returning `NotImplementedError` instead of `NotImplemented`.** The first is an *exception class*; the second is a *singleton value* that operator dunders must return when they can't handle the other type. They are not interchangeable.
- **Mutable dataclass default (`tags: list = []`).** A `ValueError` at class-definition time. Always use `field(default_factory=list)`.
- **Field-order error in dataclasses.** A field with a default cannot precede one without a default (`TypeError`). Reorder, give defaults, or use `kw_only=True`.
- **Making a mutable object hashable.** If a hashed field changes after the object is put in a set/dict, lookups silently fail. Only hash immutable (ideally `frozen`) objects.
- **Forgetting `__radd__`/`__rmul__`** for math types — `v + 3` works but `3 + v` raises `TypeError` until you add the reflected method.
- **`__len__` returning a non-int or negative number** — raises `TypeError`/`ValueError`. It must be a non-negative `int`.
- **Calling dunders directly** (`obj.__len__()`). Works, but un-Pythonic — use `len(obj)`. The built-in also does extra checks the direct call skips.
- **Heavy logic in `__repr__`** that can itself raise — a broken `__repr__` makes debugging miserable because the debugger can't display the object.

## Best practices

- Always give value classes a clear `__repr__` — future-you debugging at 2am will thank you.
- Reach for `@dataclass` before hand-writing `__init__`/`__repr__`/`__eq__`. Less code, fewer bugs.
- Use `@dataclass(frozen=True)` for value objects you want to compare, hash, and pass around safely.
- Build `__eq__`/`__hash__`/ordering off a *tuple of the identifying fields* — concise and consistent.
- Return `NotImplemented` (the value) from comparison/operator dunders when the other operand's type is unsupported.
- Add `__slots__` (or `slots=True`) only when profiling shows many instances dominate memory — don't pre-optimize.
- Keep dunders cheap and side-effect-free where possible; especially `__repr__`, `__eq__`, `__hash__`.
- Prefer generators (`yield`) over hand-rolled `__iter__`/`__next__` for iteration.
- Use Pydantic (not bare dataclasses) when validating/parsing untrusted external input.

## Interview Q&A

**Q: What's the difference between `__str__` and `__repr__`, and which should you implement if you only implement one?**
**A:** `__str__` is the human-readable form used by `print()`, `str()`, and f-strings; `__repr__` is the unambiguous developer form used in the REPL, debuggers, and inside containers, ideally looking like code that recreates the object. If you implement only one, implement `__repr__` — `str()` falls back to `__repr__`, but `repr()` does not fall back to `__str__`.

**Q: Why does defining `__eq__` break hashing, and how do you fix it?**
**A:** When you define `__eq__`, Python sets `__hash__` to `None`, making instances unhashable, because the hash/equality contract requires equal objects to share a hash and Python can't guarantee that automatically. Fix it by also defining `__hash__` over the same fields (commonly `hash((self.a, self.b))`), or use `@dataclass(frozen=True)` which generates both correctly.

**Q: What does returning `NotImplemented` from `__eq__` accomplish, and how is it different from `NotImplementedError`?**
**A:** `NotImplemented` is a singleton value signaling "I can't compare with this type"; Python then tries the reflected operation on the other operand or falls back to identity. `NotImplementedError` is an exception class you `raise` from unfinished methods — returning it from a dunder would be a bug.

**Q: What is `functools.total_ordering` and what does it require?**
**A:** It's a class decorator that fills in the missing rich-comparison methods (`__le__`, `__gt__`, `__ge__`) given `__eq__` plus one of `__lt__`/`__le__`/`__gt__`/`__ge__`. It saves boilerplate; the trade-off is slightly slower comparisons than hand-written ones.

**Q: Why must you use `field(default_factory=list)` for a list default in a dataclass?**
**A:** A bare mutable default would be a single shared object across all instances (the mutable-default footgun), so dataclasses raise a `ValueError` for it. `default_factory` is a zero-arg callable invoked per instance, giving each its own fresh list/dict/set.

**Q: What does `@dataclass(frozen=True)` give you, and when would you use it?**
**A:** It makes instances immutable (attribute assignment raises `FrozenInstanceError`) and auto-generates `__hash__`, so instances are hashable and safe as dict keys / set members. Use it for value objects — money, coordinates, config snapshots — where identity is value-based and you want safe sharing.

**Q: How does `3 * my_vector` work when `my_vector * 3` is what you defined?**
**A:** Python first tries `int.__mul__(3, my_vector)`, which returns `NotImplemented` because `int` doesn't know about `Vector`; Python then tries the reflected method `my_vector.__rmul__(3)`. So you implement `__rmul__` to support the int-on-the-left case.

**Q: What does `__call__` do?**
**A:** It makes instances callable like functions — `obj(...)` invokes `obj.__call__(...)`. It's useful for stateful or configurable "functions," class-based decorators, and DI-managed service objects; `callable(obj)` returns `True` for such instances.

**Q: When would you use `__slots__`, and what's the cost?**
**A:** Use it when you have very many instances of a class with a fixed attribute set, to cut memory and slightly speed attribute access by removing the per-instance `__dict__`. The cost: you can't add undeclared attributes dynamically, there's no `__dict__`, and slots interact awkwardly with multiple inheritance.

**Q: How do `__len__` and `__bool__` interact?**
**A:** `if obj:` calls `__bool__`; if `__bool__` is absent, Python falls back to `__len__` (zero length is falsy); if both are absent, the instance is always truthy. So defining `__len__` alone already gives sensible truthiness for collection-like types.

**Q: What's the difference between a dataclass and a Pydantic model?**
**A:** A dataclass is pure data plumbing — it generates `__init__`/`__repr__`/`__eq__` but does no runtime validation or type coercion. Pydantic v2 validates and coerces data against the annotations at construction (via `model_validate`/`model_dump`), which is what you want at trust boundaries like HTTP request bodies; use dataclasses for internal value objects.

**Q: What does `__post_init__` do in a dataclass?**
**A:** It runs immediately after the generated `__init__` finishes assigning fields, letting you validate inputs or compute derived fields (often paired with `field(init=False)`). For frozen dataclasses you must set derived fields via `object.__setattr__` because direct assignment is blocked.

## Practice exercises

1. **Temperature value object.** Write a `Temperature` class storing `celsius: float`. Implement `__repr__`, `__eq__`, and `__hash__` so two temperatures with the same celsius are equal and usable in a `set`. Add a `__lt__` and decorate with `@total_ordering` so a list of temperatures can be `sorted()`.

2. **Convert it to a dataclass.** Rewrite exercise 1 as a single `@dataclass(frozen=True, order=True)`. Confirm equality, hashing (put two equal ones in a set), and sorting all work with far less code.

3. **Money with operator overloading.** Build a `Money` class (`amount: int` cents, `currency: str`). Implement `__add__` that raises `ValueError` when currencies differ, plus `__eq__`, `__repr__`, and a friendly `__str__` like `$1.50`. Make `2 * Money(...)` work via `__mul__` and `__rmul__`.

4. **A dict-like store.** Implement `CaseInsensitiveDict` supporting `d["A"] = 1`, `d["a"]`, `del d["A"]`, `"A" in d`, and `len(d)` using `__getitem__`, `__setitem__`, `__delitem__`, `__contains__`, and `__len__`.

5. **Callable counter.** Write a `Counter` class whose instances, when called, return how many times they've been called so far. `c = Counter(); c(); c(); c()` should print `1 2 3`.

6. **Validated dataclass.** Make a `@dataclass` `Order` with `items: list[str]` (use `default_factory`), `discount: float = 0.0`, and a derived `count` field via `__post_init__` / `field(init=False)`. Raise `ValueError` if `discount` is outside `0.0–1.0`.

<details>
<summary>Solutions</summary>

```python
# 1. Temperature value object (hand-written dunders)
from functools import total_ordering

@total_ordering
class Temperature:
    def __init__(self, celsius: float) -> None:
        self.celsius = celsius

    def __repr__(self) -> str:
        return f"Temperature({self.celsius})"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Temperature):
            return NotImplemented
        return self.celsius == other.celsius

    def __hash__(self) -> int:
        return hash(self.celsius)

    def __lt__(self, other: "Temperature") -> bool:
        if not isinstance(other, Temperature):
            return NotImplemented
        return self.celsius < other.celsius

print(Temperature(20) == Temperature(20))   # => True
print(len({Temperature(20), Temperature(20)}))  # => 1
print(sorted([Temperature(30), Temperature(10), Temperature(20)]))
# => [Temperature(10), Temperature(20), Temperature(30)]
```

```python
# 2. Same thing as a dataclass
from dataclasses import dataclass

@dataclass(frozen=True, order=True)
class Temperature:
    celsius: float

print(Temperature(20) == Temperature(20))        # => True
print(len({Temperature(20), Temperature(20)}))   # => 1  (frozen => hashable)
print(sorted([Temperature(30), Temperature(10)]))  # => [Temperature(celsius=10), Temperature(celsius=30)]
```

```python
# 3. Money with operator overloading
class Money:
    def __init__(self, amount: int, currency: str) -> None:
        self.amount = amount      # cents
        self.currency = currency

    def __add__(self, other: "Money") -> "Money":
        if not isinstance(other, Money):
            return NotImplemented
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return Money(self.amount + other.amount, self.currency)

    def __mul__(self, n: int) -> "Money":
        return Money(self.amount * n, self.currency)

    def __rmul__(self, n: int) -> "Money":
        return self.__mul__(n)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __repr__(self) -> str:
        return f"Money({self.amount}, {self.currency!r})"

    def __str__(self) -> str:
        return f"${self.amount / 100:.2f}"

print(Money(100, "USD") + Money(50, "USD"))   # => Money(150, 'USD')
print(str(2 * Money(75, "USD")))              # => $1.50
print(Money(100, "USD") == Money(100, "USD")) # => True
```

```python
# 4. Case-insensitive dict-like store
class CaseInsensitiveDict:
    def __init__(self) -> None:
        self._data: dict[str, object] = {}

    def __getitem__(self, key: str):
        return self._data[key.lower()]

    def __setitem__(self, key: str, value: object) -> None:
        self._data[key.lower()] = value

    def __delitem__(self, key: str) -> None:
        del self._data[key.lower()]

    def __contains__(self, key: str) -> bool:
        return key.lower() in self._data

    def __len__(self) -> int:
        return len(self._data)

d = CaseInsensitiveDict()
d["Accept"] = "json"
print(d["accept"])     # => json
print("ACCEPT" in d)   # => True
print(len(d))          # => 1
del d["accept"]
print(len(d))          # => 0
```

```python
# 5. Callable counter
class Counter:
    def __init__(self) -> None:
        self.count = 0

    def __call__(self) -> int:
        self.count += 1
        return self.count

c = Counter()
print(c(), c(), c())   # => 1 2 3
```

```python
# 6. Validated dataclass
from dataclasses import dataclass, field

@dataclass
class Order:
    items: list[str] = field(default_factory=list)
    discount: float = 0.0
    count: int = field(init=False)

    def __post_init__(self) -> None:
        if not 0.0 <= self.discount <= 1.0:
            raise ValueError("discount must be between 0.0 and 1.0")
        self.count = len(self.items)

o = Order(items=["a", "b", "c"], discount=0.2)
print(o)            # => Order(items=['a', 'b', 'c'], discount=0.2, count=3)
print(o.count)      # => 3
# Order(discount=1.5)  # => ValueError: discount must be between 0.0 and 1.0
```

</details>

## Cheat sheet

```text
CONSTRUCTION / DISPLAY
  __init__(self, ...)     initializer (like __construct)
  __repr__(self)          dev string; REPL, debugger, containers; IMPLEMENT THIS
  __str__(self)           user string; print(), str(), f"{x}"
  f"{x!r}"                force repr inside f-string

EQUALITY / HASHING  (override together!)
  __eq__(self, other)     value equality; return NotImplemented for unknown types
  __hash__(self)          hash((self.a, self.b)); needed for set/dict keys
  rule: define __eq__  => must define __hash__ (or instances become unhashable)

ORDERING
  __lt__ __le__ __gt__ __ge__   rich comparisons
  @functools.total_ordering     derive the rest from __eq__ + one of the above
  idiom: compare a tuple of fields -> (self.a, self.b) < (other.a, other.b)

COLLECTION-LIKE
  __len__       -> len(obj)        (non-negative int)
  __bool__      -> if obj:         (falls back to __len__)
  __contains__  -> x in obj
  __getitem__   -> obj[k]   (also slicing)
  __setitem__   -> obj[k] = v
  __delitem__   -> del obj[k]
  __iter__/__next__ -> for x in obj   (prefer a generator with yield)

BEHAVIOR
  __call__      -> obj(...)         callable instances (like __invoke)
  __enter__/__exit__ -> with obj:   __exit__ returns True to suppress exception

OPERATORS
  __add__/__radd__  __sub__/__rsub__  __mul__/__rmul__  __truediv__/__rtruediv__
  reflected __r*__ handles `3 * v` when left operand can't (returns NotImplemented)

DATACLASSES
  @dataclass                         generates __init__/__repr__/__eq__
  x: int = 5                         simple default
  field(default_factory=list)        mutable default (NEVER `= []`)
  field(init=False)                  field not in constructor (set in __post_init__)
  @dataclass(frozen=True)            immutable + hashable (value objects)
  @dataclass(order=True)             generates __lt__ etc. (fields in order)
  @dataclass(slots=True)             __slots__ (less memory, no __dict__)
  @dataclass(kw_only=True)           all fields keyword-only
  __post_init__(self)                validation / derived fields
  asdict(o) astuple(o) replace(o, **) fields(o)   helpers

MEMORY
  __slots__ = ("x", "y")   fixed attrs, no per-instance __dict__, blocks typos
  vs Pydantic: dataclass = plumbing; Pydantic = runtime validation at boundaries
```
