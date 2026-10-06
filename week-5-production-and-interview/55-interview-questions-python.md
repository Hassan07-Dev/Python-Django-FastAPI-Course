# 55 — Interview Questions — Python

> Week 5 · Interview Prep

This lesson is a focused **question bank** for core-Python interviews — the language fundamentals an interviewer probes before they trust you with Django or FastAPI. Each answer is written to be *spoken aloud in 20–60 seconds*: correct, concise, and confident. Skim the answers, then cover them and re-answer from memory.

## What you will learn

- 40+ realistic Python interview questions grouped by theme, each with a strong model answer.
- The "why" behind classic gotchas (`is` vs `==`, mutable default args, late-binding closures).
- How to talk about the GIL, threading vs multiprocessing vs asyncio without hand-waving.
- Memory management: reference counting, the cyclic garbage collector, interning.
- OOP internals interviewers love: MRO, `classmethod` vs `staticmethod`, `__slots__`, dataclasses, ABCs.
- A batch of "what does this print?" puzzles — the format that trips up even strong devs.
- A one-page cheat sheet for last-minute review before the call.

## PHP/Laravel bridge

You already know OOP, exceptions, closures, and HTTP — so the *concepts* transfer. What differs is Python's execution model and object semantics. PHP copies arrays by value on assignment; Python binds **names to objects** and never copies on `=`, which is why mutable-default and aliasing bugs feel alien at first. PHP-FPM gives you a fresh process per request with no shared state to worry about; Python long-running servers force you to reason about the GIL, threads, and async. PHP's `==` (loose) vs `===` (strict) maps loosely onto Python's `==` (value) vs `is` (identity) — but Python's `is` is identity, *not* type-strict equality, a distinction interviewers test directly.

---

## Language basics & data types

**Q: What's the difference between `is` and `==`?**
**A:** `==` calls `__eq__` and compares **values**; `is` compares **identity** — whether two names point to the exact same object in memory (same `id()`). Use `==` for value checks and `is` only for singletons like `None`, `True`, `False`. They can disagree: `[1] == [1]` is `True` but `[1] is [1]` is `False` (two distinct list objects).

```python
a = [1, 2, 3]
b = [1, 2, 3]
print(a == b)  # => True  (same value)
print(a is b)  # => False (different objects)
c = a
print(a is c)  # => True  (same object)
```

**Q: Why does `a is b` sometimes return `True` for small integers but not large ones?**
**A:** CPython **caches** (pre-allocates) small integers from -5 to 256 and short interned strings as a performance optimization, so `256 is 256` is `True` but `257 is 257` may be `False`. This is a CPython implementation detail, never a guarantee — which is exactly why you must use `==` for value comparison, never `is`.

```python
x = 256; y = 256
print(x is y)  # => True  (cached)
x = 257; y = 257
print(x is y)  # => False (not cached; may vary by context)
```

**Q: What does it mean that some types are mutable and others immutable? Give examples.**
**A:** A mutable object can change its internal state after creation; an immutable one cannot. **Mutable:** `list`, `dict`, `set`, `bytearray`, most custom objects. **Immutable:** `int`, `float`, `str`, `tuple`, `frozenset`, `bytes`. Immutability matters because only hashable (effectively immutable) objects can be dict keys or set members, and immutables are safe to share across threads.

**Q: List vs tuple — when do you use each?**
**A:** Lists are mutable, variable-length collections for homogeneous, changing data. Tuples are immutable, fixed-size records — ideal for heterogeneous data with positional meaning (like a DB row), as dict keys, and for safely returning multiple values. Tuples are also slightly more memory-efficient and can be hashed.

```python
point = (3, 4)          # fixed record
coords = {(0, 0): "origin"}  # tuple works as dict key
# coords[[0, 0]] -> TypeError: unhashable type: 'list'
```

**Q: How is a Python `dict` implemented, and what's its lookup complexity?**
**A:** A `dict` is a **hash table**. Keys must be hashable; lookup, insert, and delete are **O(1) average** (O(n) worst case under heavy collisions). Since Python 3.7, dicts **preserve insertion order** as a language guarantee. Internally CPython uses a compact "indices + entries" layout that saves memory and gives the ordering for free.

**Q: What values are "falsy" in Python?**
**A:** `None`, `False`, numeric zero (`0`, `0.0`, `0j`, `Decimal(0)`), and **empty** containers/sequences (`""`, `[]`, `{}`, `()`, `set()`, `range(0)`). Everything else is truthy. Custom objects are truthy unless they define `__bool__` returning `False` or `__len__` returning `0`.

```python
print(bool([]))     # => False
print(bool([0]))    # => True  (non-empty list)
print(bool("0"))    # => True  (non-empty string!)
print(bool(0.0))    # => False
```

**Q: How do you safely check for an empty list — `if mylist:` or `if len(mylist) == 0:`?**
**A:** Prefer `if not mylist:` (Pythonic truthiness). It's idiomatic and avoids a redundant `len()` call. But be deliberate when a value might be `None` vs empty: `if mylist is None:` and `if not mylist:` mean different things, and conflating them is a real bug source.

---

## Strings & memory

**Q: Are Python strings mutable? What happens when you "modify" one?**
**A:** Strings are **immutable**. Any operation that "changes" a string actually creates a **new** string object. This is why building a string in a loop with `+=` is O(n²) — use `"".join(list_of_parts)` instead, which is O(n).

```python
s = "abc"
print(id(s))
s += "d"          # creates a NEW string object
print(id(s))      # => different id

# Efficient build:
parts = [str(i) for i in range(1000)]
result = "".join(parts)   # one allocation, O(n)
```

**Q: What is string interning and why does it exist?**
**A:** Interning stores a single shared copy of certain strings so identical literals point to the same object, speeding up equality checks (identity compare first) and saving memory. CPython auto-interns string literals that look like identifiers (letters, digits, underscores). You can force it with `sys.intern()`. Never *rely* on interning for correctness — use `==`, not `is`, for string comparison.

```python
import sys
a = sys.intern("hello world")
b = sys.intern("hello world")
print(a is b)  # => True (forced interning)
```

**Q: f-string vs `.format()` vs `%` — which should you use?**
**A:** Prefer **f-strings** (Python 3.6+): fastest, most readable, evaluated inline. `.format()` is fine when the template is separate from the data (e.g. stored in config). `%`-formatting is legacy. One exception: in logging, pass `%`-style args lazily — `logger.info("user %s", uid)` — so the string is only formatted if the level is enabled.

---

## Functions

**Q: What's the difference between `*args` and `**kwargs`?**
**A:** `*args` collects extra **positional** arguments into a tuple; `**kwargs` collects extra **keyword** arguments into a dict. In a *call*, `*` unpacks an iterable into positional args and `**` unpacks a mapping into keyword args. The canonical order in a signature is: positional, `*args`, keyword-only, `**kwargs`.

```python
def f(a, *args, **kwargs):
    print(a, args, kwargs)

f(1, 2, 3, x=10)        # => 1 (2, 3) {'x': 10}

nums = [2, 3]; opts = {"x": 10}
f(1, *nums, **opts)     # => 1 (2, 3) {'x': 10}  (unpacking at call site)
```

**Q: Explain the mutable default argument pitfall.**
**A:** Default arguments are evaluated **once**, when the function is *defined*, not on each call. So a mutable default like `[]` is **shared across all calls**, accumulating state. The fix is the `None` sentinel pattern.

```python
def bad(item, bucket=[]):      # BUG: one shared list
    bucket.append(item)
    return bucket

print(bad(1))  # => [1]
print(bad(2))  # => [1, 2]  (!) — leaked state

def good(item, bucket=None):   # FIX
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket

print(good(1))  # => [1]
print(good(2))  # => [2]  (fresh each time)
```

**Q: What is a closure?**
**A:** A closure is a nested function that **captures and remembers** variables from its enclosing scope, even after that scope has returned. Python captures variables **by reference**, not by value — the source of the "late binding" gotcha below. Use `nonlocal` to *rebind* an enclosing variable from the inner function.

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count      # rebind enclosing variable
        count += 1
        return count
    return increment

c = make_counter()
print(c(), c(), c())  # => 1 2 3
```

**Q: Why does this loop-closure print the wrong values, and how do you fix it?**
**A:** Closures capture the *variable* `i`, not its value at creation time. By the time the lambdas run, the loop has finished and `i == 2`, so all of them return `2`. Fix by binding the current value as a default argument (evaluated at definition).

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # => [2, 2, 2]  (late binding)

funcs = [lambda i=i: i for i in range(3)]   # bind per-iteration
print([f() for f in funcs])  # => [0, 1, 2]
```

**Q: What is a decorator and how do you write one?**
**A:** A decorator is a callable that takes a function and returns a (usually wrapped) function, used to add behavior — logging, timing, auth, caching — without editing the original. `@deco` above `def f` is sugar for `f = deco(f)`. Always apply `functools.wraps` so the wrapper preserves the original's `__name__`, docstring, and signature metadata.

```python
import functools, time

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.6f}s")
        return result
    return wrapper

@timed
def work(n):
    return sum(range(n))

work(1_000_000)          # => work took 0.0xxxxx s
print(work.__name__)     # => work  (preserved thanks to @wraps)
```

**Q: How do you write a decorator that takes arguments?**
**A:** Add an outer layer: a function that takes the arguments and *returns* the actual decorator. So you have three nested levels: factory → decorator → wrapper.

```python
import functools

def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    print(f"Hi {name}")

greet("Sam")   # prints "Hi Sam" three times
```

---

## Comprehensions & generators

**Q: List comprehension vs generator expression — what's the difference?**
**A:** A list comprehension `[x for x in it]` builds the **entire list in memory eagerly**. A generator expression `(x for x in it)` is **lazy** — it yields items one at a time and holds only the current item, so it's memory-constant. Use a generator for large or streaming data, or when you only iterate once; use a list when you need indexing, length, or multiple passes.

```python
import sys
lst = [x * 2 for x in range(10_000)]   # all in memory
gen = (x * 2 for x in range(10_000))   # lazy
print(sys.getsizeof(lst))  # => ~ tens of thousands of bytes
print(sys.getsizeof(gen))  # => ~ 200 bytes (constant)
```

**Q: What does `yield` do, and how does a generator function differ from a normal one?**
**A:** A function containing `yield` is a **generator function**: calling it returns a generator object without running the body. Each `next()` runs until the next `yield`, returns that value, and **suspends** — preserving local state. It resumes from there on the following `next()`. This enables lazy pipelines and infinite sequences with O(1) memory.

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

for x in countdown(3):
    print(x)   # => 3, 2, 1

g = countdown(2)
print(next(g))  # => 2
print(next(g))  # => 1
# next(g) now raises StopIteration
```

**Q: What does `yield from` do?**
**A:** It **delegates** to a sub-iterator: it yields every item from the iterable (and transparently forwards `send`/`throw` for coroutines). It flattens nested generators cleanly.

```python
def chain(*iterables):
    for it in iterables:
        yield from it

print(list(chain([1, 2], [3, 4])))  # => [1, 2, 3, 4]
```

**Q: Are dict and set comprehensions a thing?**
**A:** Yes. `{k: v for ...}` builds a dict, `{x for ...}` builds a set. Be careful: `{}` is an empty **dict**, not a set — use `set()` for an empty set.

```python
squares = {n: n*n for n in range(4)}   # => {0: 0, 1: 1, 2: 4, 3: 9}
evens = {n for n in range(10) if n % 2 == 0}  # set
```

---

## OOP

**Q: What is `self`, and why must it be explicit?**
**A:** `self` is the instance the method is bound to — Python's equivalent of `$this`. It's explicit because of the "explicit is better than implicit" philosophy: `obj.method()` is really `Class.method(obj)`, so the first parameter receives the instance. The name `self` is convention, not a keyword.

**Q: `@classmethod` vs `@staticmethod` vs instance method?**
**A:** An instance method takes `self` and works on instance state. A `@classmethod` takes `cls` (the class) and works on class state — perfect for **alternative constructors** (factory methods). A `@staticmethod` takes neither; it's just a function namespaced under the class with no access to instance or class.

```python
class User:
    def __init__(self, name):
        self.name = name

    @classmethod
    def from_dict(cls, data):       # alternative constructor
        return cls(data["name"])    # cls() respects subclasses

    @staticmethod
    def is_valid_name(name):        # pure utility
        return bool(name) and name.isalpha()

u = User.from_dict({"name": "Ada"})
print(u.name)                       # => Ada
print(User.is_valid_name("Ada"))   # => True
```

**Q: What is the MRO (Method Resolution Order)?**
**A:** The MRO is the linear order Python searches base classes for an attribute/method, computed with the **C3 linearization** algorithm. It guarantees a consistent order in multiple inheritance and that each class appears once. `super()` follows the MRO, not just the direct parent. Inspect it with `Cls.__mro__` or `Cls.mro()`.

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
print([c.__name__ for c in D.__mro__])
# => ['D', 'B', 'C', 'A', 'object']
```

**Q: What does `super()` actually do?**
**A:** It returns a proxy that dispatches method calls to the **next class in the MRO** of the instance — not necessarily the literal parent. This makes cooperative multiple inheritance work: each `super().__init__()` walks the chain exactly once. In Python 3 you call it argument-free as `super()` inside a method.

**Q: What are dunder (magic) methods? Name a few useful ones.**
**A:** "Dunder" = double-underscore methods that hook into Python's syntax and protocols. `__init__` (construction), `__repr__`/`__str__` (representation), `__eq__`/`__hash__` (equality & hashing), `__len__`, `__getitem__` (indexing/iteration), `__enter__`/`__exit__` (context managers), `__call__` (make an instance callable), `__iter__`/`__next__` (iteration).

```python
class Money:
    def __init__(self, cents): self.cents = cents
    def __repr__(self): return f"Money({self.cents})"
    def __eq__(self, other): return self.cents == other.cents
    def __add__(self, other): return Money(self.cents + other.cents)

print(Money(100) + Money(50))  # => Money(150)
print(Money(100) == Money(100))  # => True
```

**Q: `__str__` vs `__repr__`?**
**A:** `__repr__` is for developers — unambiguous, ideally `eval`-able, shown in the REPL and debuggers. `__str__` is for end users — readable. If you implement only one, implement `__repr__`; `str()` falls back to it.

**Q: What is a `dataclass` and when do you use it?**
**A:** `@dataclass` auto-generates `__init__`, `__repr__`, `__eq__` (and more) from typed class attributes — eliminating boilerplate for data-holding classes. Use `field(default_factory=...)` for mutable defaults, `frozen=True` for immutability (also makes instances hashable), and `slots=True` (3.10+) for memory efficiency.

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True)
class Point:
    x: int
    y: int
    tags: list[str] = field(default_factory=list)  # safe mutable default

p = Point(1, 2)
print(p)            # => Point(x=1, y=2, tags=[])
print(p == Point(1, 2))  # => True
# p.x = 5  -> FrozenInstanceError
```

**Q: What is an abstract base class (ABC)?**
**A:** An ABC defines an interface that subclasses **must** implement. Inherit from `abc.ABC` and mark required methods with `@abstractmethod`; instantiating a subclass that hasn't implemented all of them raises `TypeError`. It's how you enforce a contract at class-definition/instantiation time.

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def get(self, id: int): ...

class InMemoryRepo(Repository):
    def get(self, id): return {"id": id}

r = InMemoryRepo()
print(r.get(1))   # => {'id': 1}
# Repository()    -> TypeError: Can't instantiate abstract class
```

**Q: What is `__slots__` and why use it?**
**A:** `__slots__` declares a fixed set of attributes, telling Python to skip the per-instance `__dict__`. Benefits: **lower memory** per instance (big when you have millions) and slightly faster attribute access. Cost: you can't add new attributes dynamically, and it complicates multiple inheritance.

```python
class Pixel:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y

p = Pixel(1, 2)
# p.z = 3  -> AttributeError: 'Pixel' object has no attribute 'z'
```

---

## Exceptions & context managers

**Q: How does `try/except/else/finally` work?**
**A:** `try` runs the protected code; matching `except` handles a raised exception; `else` runs only if **no** exception occurred; `finally` runs **always** (cleanup), even on `return` or re-raise. Order matters: catch specific exceptions before general ones, and avoid bare `except:` (it swallows `KeyboardInterrupt`/`SystemExit`).

```python
def parse(x):
    try:
        return int(x)
    except ValueError:
        return None
    else:
        print("never reached if exception")
    finally:
        print("always runs")

print(parse("12"))   # => "always runs", then 12
print(parse("abc"))  # => "always runs", then None
```

**Q: What's the difference between `Exception` and `BaseException`?**
**A:** `BaseException` is the root; `Exception` is its subclass and the base for "normal" errors you should catch. Control-flow exceptions — `KeyboardInterrupt`, `SystemExit`, `GeneratorExit` — derive directly from `BaseException`, *not* `Exception`, so `except Exception:` won't accidentally swallow a Ctrl-C. Always catch `Exception`, never bare `except:`.

**Q: What is a context manager and how do you write one?**
**A:** A context manager guarantees setup/teardown around a block via `with`. It implements `__enter__` (returns the resource) and `__exit__` (cleanup, even on exception). The easiest way to write one is `contextlib.contextmanager` with a generator: code before `yield` is setup, after is teardown.

```python
from contextlib import contextmanager

@contextmanager
def managed_resource(name):
    print(f"open {name}")
    try:
        yield name              # value bound to `as`
    finally:
        print(f"close {name}")  # always runs

with managed_resource("db") as r:
    print(f"using {r}")
# => open db / using db / close db
```

**Q: How do you raise a custom exception and chain it?**
**A:** Subclass `Exception`. Use `raise NewError(...) from original` to preserve the cause (shown as "The above exception was the direct cause..."), which keeps the traceback honest.

```python
class ConfigError(Exception): ...

try:
    int("nope")
except ValueError as e:
    raise ConfigError("bad config value") from e
```

---

## Concurrency: GIL, threading, multiprocessing, asyncio

**Q: What is the GIL and why does it exist?**
**A:** The Global Interpreter Lock is a mutex in CPython that allows **only one thread to execute Python bytecode at a time**. It exists because CPython's memory management (reference counting) isn't thread-safe; the GIL makes the interpreter simpler and single-threaded code faster. The consequence: threads can't achieve true parallelism for **CPU-bound** Python code. (Note: Python 3.13 ships an *experimental* free-threaded build that can disable the GIL.)

**Q: Threading vs multiprocessing vs asyncio — when do you use each?**
**A:**

| Model | Parallelism | Best for | Why |
|---|---|---|---|
| `threading` | Concurrent, not parallel (GIL) | **I/O-bound** tasks | GIL is released during blocking I/O, so threads overlap waits |
| `multiprocessing` | True parallel | **CPU-bound** tasks | Separate processes = separate GILs = use all cores |
| `asyncio` | Single-threaded cooperative | **High-concurrency I/O** (thousands of sockets) | One thread, no thread overhead, explicit `await` points |

In short: **CPU-bound → multiprocessing; I/O-bound → asyncio (or threads).**

**Q: If the GIL blocks parallelism, why is threading still useful?**
**A:** Because the GIL is **released during blocking I/O** (network, disk) and inside many C extensions (e.g. NumPy). So while one thread waits on a socket, another runs. For I/O-bound workloads threads give real concurrency; only **CPU-bound pure-Python** work is serialized.

**Q: What is `async`/`await` and how does it differ from threads?**
**A:** `async def` defines a coroutine; `await` suspends it at an I/O point and hands control back to the event loop, which runs other ready coroutines. It's **cooperative** (you choose suspension points) and single-threaded — no locks needed for most shared state, no OS thread overhead. Threads are **preemptive** (the OS switches at any time), so they need locks. asyncio scales to tens of thousands of concurrent connections that threads can't.

```python
import asyncio

async def fetch(name, delay):
    await asyncio.sleep(delay)   # non-blocking; yields to loop
    return f"{name} done"

async def main():
    results = await asyncio.gather(
        fetch("a", 1), fetch("b", 1), fetch("c", 1)
    )
    print(results)   # => ['a done', 'b done', 'c done'] in ~1s, not 3s

asyncio.run(main())
```

**Q: Common async mistake — what happens if you call a blocking function inside a coroutine?**
**A:** It blocks the **entire event loop**, freezing every other coroutine, because there's only one thread. Never call blocking I/O (e.g. `requests.get`, `time.sleep`, sync DB drivers) directly in async code. Use async libraries (`httpx`, `asyncio.sleep`) or offload blocking calls with `await asyncio.to_thread(fn, ...)`.

**Q: What does `asyncio.gather` do vs awaiting tasks in a loop?**
**A:** `gather` schedules coroutines **concurrently** and waits for all — total time ≈ the slowest one. Awaiting them sequentially in a `for` loop runs them **one after another** — total time = the sum. Use `gather` (or `asyncio.TaskGroup` in 3.11+) to overlap independent I/O.

---

## Memory management & garbage collection

**Q: How does Python manage memory / garbage collection?**
**A:** CPython's primary mechanism is **reference counting**: every object tracks how many references point to it; when the count hits zero, it's freed immediately. To handle **reference cycles** (which counting alone can't reclaim), there's a supplemental **generational cyclic garbage collector** (`gc` module). Memory allocation itself is handled by CPython's `pymalloc` arenas.

**Q: What is reference counting and what's its main weakness?**
**A:** Each object holds a count of references to it; assignment increments, going out of scope/`del` decrements; at zero it's deallocated immediately (deterministic, low latency). Its weakness is **reference cycles**: objects that reference each other keep their counts above zero even when unreachable, so they'd leak — which is why the cyclic GC exists.

```python
import sys
a = []
print(sys.getrefcount(a))  # => 2 (the var + the temporary arg to getrefcount)
b = a
print(sys.getrefcount(a))  # => 3
del b
print(sys.getrefcount(a))  # => 2
```

**Q: How does the cyclic garbage collector work?**
**A:** It's **generational**: new objects start in generation 0; survivors are promoted to older generations collected less often (most objects die young — the "generational hypothesis"). It periodically scans tracked container objects to detect groups that reference only each other (unreachable cycles) and reclaims them. You can trigger it manually with `gc.collect()`.

```python
import gc

class Node:
    def __init__(self): self.ref = None

a = Node(); b = Node()
a.ref = b; b.ref = a    # cycle: refcounts never hit 0
del a, b                # unreachable, but cyclic
collected = gc.collect()  # cyclic GC reclaims them
print(collected >= 0)   # => True
```

**Q: What is a weak reference and why use one?**
**A:** A `weakref` lets you reference an object **without** incrementing its refcount, so it doesn't prevent garbage collection. Use it for caches and observer patterns where you don't want the cache to keep objects alive forever (e.g. `weakref.WeakValueDictionary`).

---

## Typing

**Q: Are Python type hints enforced at runtime?**
**A:** **No.** Hints are optional annotations; the interpreter ignores them at runtime (they're stored in `__annotations__`). They're enforced by **static checkers** like `mypy`/`pyright` and used by tools like Pydantic, FastAPI, and IDEs. They improve readability, catch bugs early, and power editor autocomplete — but a wrong type won't raise unless something explicitly validates it.

**Q: How do you type modern collections, optionals, and unions (3.10+)?**
**A:** Use built-in generics and the `|` union syntax. `list[int]`, `dict[str, int]`, `X | None` (replaces `Optional[X]`), `int | str` (replaces `Union`). For "any callable", `Callable[[int], str]`. For structural typing, use `typing.Protocol`.

```python
def greet(names: list[str], prefix: str | None = None) -> str:
    p = prefix or "Hi"
    return ", ".join(f"{p} {n}" for n in names)

print(greet(["Ada", "Sam"]))  # => Hi Ada, Hi Sam
```

---

## Modules & packages

**Q: What is `if __name__ == "__main__":` for?**
**A:** When a file is **run directly**, its `__name__` is `"__main__"`; when **imported**, `__name__` is the module's name. The guard lets a module expose functions for import *and* have script-only entry-point code that doesn't fire on import — essential for testability and avoiding side effects.

```python
def main():
    print("running as script")

if __name__ == "__main__":
    main()
```

**Q: What's the difference between a module and a package?**
**A:** A **module** is a single `.py` file. A **package** is a directory of modules; historically it needed an `__init__.py`, though namespace packages (PEP 420) relax that. Packages give you a hierarchical namespace and let you do `from app.services import users`.

**Q: Absolute vs relative imports?**
**A:** Absolute imports spell out the full path from the project root (`from app.models import User`) — clearest and preferred. Relative imports use leading dots (`from .models import User`, `from ..utils import x`) relative to the current package. Relative imports only work inside packages, not in a top-level script run directly.

---

## "What does this print?" gotchas

**Q: What does this print and why?**
```python
def add(item, lst=[]):
    lst.append(item)
    return lst
print(add(1))
print(add(2))
```
**A:** `[1]` then `[1, 2]`. The default list is created once at definition and shared across calls — the classic mutable-default bug. Fix with `lst=None` and create inside.

**Q: What does this print?**
```python
print(0.1 + 0.2 == 0.3)
```
**A:** `False`. Floats are IEEE-754 binary, and `0.1 + 0.2` is `0.30000000000000004`. Never compare floats with `==`; use `math.isclose(0.1 + 0.2, 0.3)` or `Decimal` for money.

**Q: What does this print?**
```python
a = [1, 2, 3]
b = a
b.append(4)
print(a)
```
**A:** `[1, 2, 3, 4]`. `b = a` binds another name to the **same list** (no copy). Mutating through `b` is visible through `a`. To copy: `b = a.copy()` / `b = a[:]` / `list(a)` (shallow), or `copy.deepcopy(a)` for nested structures.

**Q: What does this print?**
```python
print([x for x in range(3)])
print(x)   # is x defined here?
```
**A:** `[0, 1, 2]`, then a `NameError`. In Python 3, comprehensions have their **own scope**, so the loop variable `x` does not leak into the enclosing scope (unlike a plain `for` loop, which does).

**Q: What does this print?**
```python
print(bool("False"), bool(""), bool("0"))
```
**A:** `True False True`. Any **non-empty** string is truthy regardless of content — `"False"` and `"0"` are non-empty, so both are `True`; only `""` is falsy. A frequent source of bugs when parsing env vars (`os.getenv("DEBUG")` returns the string `"False"`, which is truthy).

**Q: What does this print?**
```python
t = (1, 2, [3, 4])
t[2].append(5)
print(t)
```
**A:** `(1, 2, [3, 4, 5])`. A tuple is immutable in its **references**, but if it holds a mutable object you can still mutate that object in place. The tuple's slots don't change; the list they point to does.

**Q: What does this print?**
```python
print(3 * "ab")
print([0] * 3)
x = [[0]] * 3
x[0].append(1)
print(x)
```
**A:** `ababab`, then `[0, 0, 0]`, then `[[0, 1], [0, 1], [0, 1]]`. `*` repeats. The last is a gotcha: `[[0]] * 3` makes three references to the **same inner list**, so appending via one shows in all. Build independent rows with `[[0] for _ in range(3)]`.

---

## Common mistakes & gotchas

- **Mutable default arguments** (`def f(x=[])`) — shared across calls. Use `None` sentinel.
- **`is` vs `==`** — using `is` for value comparison; works by luck for small ints/interned strings, breaks otherwise.
- **Late-binding closures** in loops — capture with a default arg (`lambda i=i:`).
- **Modifying a list while iterating it** — skips elements or errors; iterate a copy (`for x in lst[:]`) or build a new list.
- **`b = a` is not a copy** for mutable objects — both names share one object.
- **Float `==` comparisons** — use `math.isclose` or `Decimal`.
- **Bare `except:`** — swallows `KeyboardInterrupt`/`SystemExit`; catch `Exception` and be specific.
- **Blocking calls inside `async` code** — freezes the whole event loop.
- **Assuming threads parallelize CPU work** — the GIL serializes them; use multiprocessing.
- **`{}` is an empty dict, not a set** — empty set is `set()`.
- **`+=` string building in a loop** — O(n²); use `"".join`.

## Best practices

- Compare to `None` with `is`/`is not`; compare values with `==`.
- Use truthiness (`if not items:`) but distinguish "empty" from `None` when it matters.
- Reach for `dataclass` / `pydantic` for data containers instead of bare dicts or manual `__init__`.
- Prefer generators for large or streaming data to keep memory flat.
- Always `@functools.wraps` your decorators.
- Use context managers (`with`) for any resource needing cleanup.
- Type-hint public functions; run `mypy`/`pyright` in CI.
- For concurrency: pick the model by workload — asyncio/threads for I/O, multiprocessing for CPU.
- Catch the **narrowest** exception you can handle; chain with `raise ... from ...`.

## Interview Q&A

(The full bank above is the Q&A — here are a few extra meta/synthesis questions interviewers like to close with.)

**Q: When would you choose a generator over returning a list from a function?**
**A:** When the result set is large or unbounded, when the caller may stop early, or when you're building a pipeline. Generators keep memory constant and start producing immediately. Return a list when callers need length, random access, or to iterate multiple times.

**Q: Explain "everything is an object" in Python.**
**A:** Functions, classes, modules, and even types are first-class objects — they have attributes, can be assigned to variables, passed as arguments, and returned. This is why decorators, higher-order functions, and `cls`/factory patterns work naturally. `type` is itself an object whose type is `type`.

**Q: What's duck typing?**
**A:** "If it walks like a duck and quacks like a duck, it's a duck." Python cares about whether an object *supports the operations you use*, not its declared type. You don't need a shared base class — just the right methods. This underpins Python's flexibility and `Protocol`-based structural typing.

**Q: How do you make an object iterable?**
**A:** Implement `__iter__` returning an iterator (often `self` plus `__next__`, or just `yield` inside `__iter__` to make it a generator). Then `for x in obj` works. Implementing only `__getitem__` with integer indices also makes an object iterable via the old sequence protocol.

```python
class Squares:
    def __init__(self, n): self.n = n
    def __iter__(self):
        for i in range(self.n):
            yield i * i

print(list(Squares(4)))  # => [0, 1, 4, 9]
```

**Q: What is monkey patching?**
**A:** Dynamically replacing/adding attributes or methods on a class or module **at runtime**. Useful for testing (patching out a network call) and quick fixes, but risky in production because it hides behavior and breaks expectations. `unittest.mock.patch` is the disciplined way to do it.

## Practice exercises

1. **Predict the output** before running, then verify:
   ```python
   def f(a, b=[]):
       b.append(a)
       return b
   print(f(1)); print(f(2)); print(f(3, []))
   ```
2. Write a decorator `@memoize` that caches a function's return values by its arguments. (Bonus: explain how it differs from `functools.lru_cache`.)
3. Write a generator `fibonacci()` that yields Fibonacci numbers lazily and forever. Use it to print the first 10.
4. Fix this late-binding bug so the list prints `[0, 1, 2]`:
   ```python
   handlers = [lambda: n for n in range(3)]
   print([h() for h in handlers])
   ```
5. Implement a context manager `timer()` (using `contextlib.contextmanager`) that prints how long the `with` block took.
6. Explain the output and fix it to give three independent lists:
   ```python
   grid = [[0] * 2] * 3
   grid[0][0] = 9
   print(grid)
   ```

<details><summary>Solutions</summary>

**1.** Output is `[1]`, `[1, 2]`, `[3]`. The first two share the single default list (mutable-default bug); the third passes a fresh list, so it's independent.

**2.**
```python
import functools

def memoize(func):
    cache = {}
    @functools.wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memoize
def slow_square(n):
    return n * n

print(slow_square(4))  # computed => 16
print(slow_square(4))  # cached    => 16
```
`functools.lru_cache(maxsize=...)` does the same with a bounded size, thread safety, and cache statistics — prefer it in real code.

**3.**
```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

import itertools
print(list(itertools.islice(fibonacci(), 10)))
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

**4.** Bind the loop value as a default argument:
```python
handlers = [lambda n=n: n for n in range(3)]
print([h() for h in handlers])  # => [0, 1, 2]
```

**5.**
```python
from contextlib import contextmanager
import time

@contextmanager
def timer():
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"elapsed: {time.perf_counter() - start:.6f}s")

with timer():
    sum(range(1_000_000))
# => elapsed: 0.0xxxxx s
```

**6.** Output is `[[9, 0], [9, 0], [9, 0]]` — `[[0]*2] * 3` makes three references to **one** inner list. Fix with a comprehension that builds a new row each time:
```python
grid = [[0] * 2 for _ in range(3)]
grid[0][0] = 9
print(grid)  # => [[9, 0], [0, 0], [0, 0]]
```

</details>

## Cheat sheet

| Topic | Key fact |
|---|---|
| `is` vs `==` | `is` = identity (same object); `==` = value. Use `is` only for `None`/`True`/`False` |
| Small-int cache | `-5..256` cached → `is` "works" by accident; never rely on it |
| Mutable | `list`, `dict`, `set`, `bytearray` | 
| Immutable | `int`, `float`, `str`, `tuple`, `frozenset`, `bytes` |
| Falsy | `None`, `False`, `0`, `0.0`, `""`, `[]`, `{}`, `()`, `set()` |
| Mutable default | Use `None` sentinel, not `=[]` / `={}` |
| Closures | Capture by reference (late binding); fix with `lambda x=x:` |
| Decorator | `f = deco(f)`; always `@functools.wraps(func)` |
| `*args`/`**kwargs` | Collect extra positional/keyword; `*`/`**` unpack at call site |
| List vs gen | List = eager, in-memory; generator `()` = lazy, O(1) memory |
| `yield` | Suspends & resumes; returns a generator; `yield from` delegates |
| `classmethod` | Takes `cls`; alternative constructors |
| `staticmethod` | No `self`/`cls`; namespaced utility |
| MRO | C3 linearization; `Cls.__mro__`; `super()` follows it |
| `__slots__` | Drops per-instance `__dict__`; saves memory; no new attrs |
| dataclass | Auto `__init__`/`__repr__`/`__eq__`; `field(default_factory=)`, `frozen=`, `slots=` |
| ABC | `abc.ABC` + `@abstractmethod`; can't instantiate if incomplete |
| try/except | `else` = no error; `finally` = always; never bare `except:` |
| Context mgr | `__enter__`/`__exit__` or `@contextmanager` + `yield` |
| GIL | One thread runs bytecode at a time (CPython); blocks CPU parallelism |
| Concurrency | I/O-bound → asyncio/threads; CPU-bound → multiprocessing |
| async | Coroutines + `await`; single-threaded cooperative; never block the loop |
| GC | Refcounting + generational cyclic collector (`gc.collect()`) for cycles |
| Type hints | Not enforced at runtime; checked by mypy/pyright |
| `__main__` | `__name__ == "__main__"` → run directly, not imported |
| Float `==` | Don't; use `math.isclose` or `Decimal` |
| String build | `"".join(parts)`, not `+=` in a loop |
