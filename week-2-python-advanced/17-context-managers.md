# 17 — Context Managers

> Week 2 · Python Advanced

## What you will learn

- The real problem context managers solve: **deterministic setup and teardown** of resources, even when exceptions fly.
- The `with` statement, and why `open()` is almost always used with it.
- The context manager **protocol** — `__enter__` and `__exit__` — and the subtle but critical meaning of `__exit__`'s return value.
- How to write a **class-based** context manager from scratch.
- The `@contextlib.contextmanager` decorator with `yield` — the easy, modern way.
- Using **multiple** context managers in one `with`, and the parenthesized form (Python 3.10+).
- The `contextlib` toolbox: `suppress`, `closing`, and `ExitStack`.
- Real-world uses: files, locks, **DB transactions/sessions**, and temporarily changing global state.
- A note on **reentrancy** — when a single context manager object can (and cannot) be reused.

## PHP/Laravel bridge

In PHP you usually rely on `try/finally` to release resources (`fclose($handle)` in a `finally`), or you lean on PHP's automatic destructor (`__destruct`) and reference-counting to clean up. Laravel's `DB::transaction(function () { ... })` is the closest cousin to a Python context manager: it wraps your callback, commits on success, and rolls back on a thrown exception. Python generalises that "wrap a block, guarantee cleanup" idea into first-class language syntax — the `with` statement — so *any* resource (files, locks, sessions, even temporary state changes) gets the same begin/commit-or-rollback treatment without you writing a `try/finally` every time. Where Laravel passes a closure to a helper, Python indents a block under `with`; the cleanup is guaranteed by the language, not by GC timing.

## Core concept — The problem: cleanup that always runs

A resource is anything you acquire and must later release: an open file handle, a network socket, a database connection, a lock. The danger is the *release*. If your code throws between acquire and release, the release never happens — you leak file descriptors, hold locks forever, or leave transactions open.

The naive version:

```python
f = open("data.txt", "w")
f.write("hello")
f.close()  # never runs if write() raises
```

If `f.write()` raises, `f.close()` is skipped and the OS file descriptor leaks until garbage collection (which in CPython is *eventually*, but you should never rely on it — and on PyPy or under heavy load it may be much later).

The `try/finally` fix works but is noisy:

```python
f = open("data.txt", "w")
try:
    f.write("hello")
finally:
    f.close()  # runs whether or not write() raised
```

The `with` statement collapses that pattern into one line and makes it impossible to forget the cleanup:

```python
with open("data.txt", "w") as f:
    f.write("hello")
# f is guaranteed closed here, exception or not
```

That is the whole pitch: **`with` guarantees teardown.** This is why you will essentially never see a bare `open()` in good Python code — `open()` returns a context manager, and using it any other way means you are signing up to manage the file descriptor by hand.

## Core concept — The `with` statement, step by step

```python
with EXPR as VAR:
    BODY
```

is (roughly) sugar for:

```python
mgr = EXPR
VAR = mgr.__enter__()          # setup; return value bound to VAR
try:
    BODY
except BaseException as exc:
    if not mgr.__exit__(type(exc), exc, exc.__traceback__):
        raise                  # re-raise unless __exit__ returned truthy
else:
    mgr.__exit__(None, None, None)  # normal path: all three args are None
```

Two things to internalise:

1. `__enter__`'s **return value** is what gets bound to `VAR` after `as`. It is *not* necessarily the manager itself (though it often is). For `open()`, `__enter__` returns the file object.
2. `__exit__` is **always** called — on success it receives `(None, None, None)`; on an exception it receives `(exc_type, exc_value, traceback)`.

The `as VAR` part is optional. `with lock:` is perfectly valid when you do not need the value `__enter__` returns.

## Core concept — The protocol: `__enter__` / `__exit__`

Any object with these two dunder methods is a context manager:

| Method | Signature | Job |
| --- | --- | --- |
| `__enter__` | `def __enter__(self)` | Acquire the resource; return the value bound after `as`. |
| `__exit__` | `def __exit__(self, exc_type, exc_value, traceback)` | Release the resource. Return value controls exception suppression. |

### The `__exit__` return value is the famous gotcha

- Return **falsy** (`None`, `False`, or simply nothing) → any exception that occurred **propagates** normally. This is what you want 99% of the time.
- Return **truthy** (`True`) → the exception is **swallowed** and execution continues after the `with` block.

Returning `True` "by accident" silences errors, which is a real bug source. Only return truthy when you *deliberately* want to suppress a specific exception (and even then, prefer `contextlib.suppress`).

```python
class Demo:
    def __enter__(self):
        print("enter")
        return self
    def __exit__(self, exc_type, exc_value, tb):
        print(f"exit, exc_type={exc_type}")
        return False  # do not suppress

with Demo():
    raise ValueError("boom")
# => enter
# => exit, exc_type=<class 'ValueError'>
# Traceback ... ValueError: boom   (exception still propagates)
```

Change `return False` to `return True` and the `ValueError` is silently swallowed — the program continues past the `with` as if nothing happened.

## Core concept — Writing a class-based context manager

A timer that measures and reports how long a block took:

```python
import time

class Timer:
    def __init__(self, label="block"):
        self.label = label

    def __enter__(self):
        self.start = time.perf_counter()
        return self  # so the caller can read self.elapsed later

    def __exit__(self, exc_type, exc_value, tb):
        self.elapsed = time.perf_counter() - self.start
        print(f"{self.label} took {self.elapsed:.4f}s")
        return False  # never suppress — a slow block that errored should still raise

with Timer("loading") as t:
    total = sum(range(1_000_000))
# => loading took 0.0123s

print(f"elapsed attribute: {t.elapsed:.4f}s")  # still accessible after the block
```

Note `__enter__` returns `self`, so `as t` gives us the `Timer` instance and we can read `t.elapsed` afterward. A managed-resource example returns the *resource* instead:

```python
class ManagedConnection:
    def __init__(self, dsn):
        self.dsn = dsn

    def __enter__(self):
        print(f"connecting to {self.dsn}")
        self.conn = object()  # pretend this is a real connection
        return self.conn      # bind the connection, not the manager

    def __exit__(self, exc_type, exc_value, tb):
        print("closing connection")
        # real code: self.conn.close()
        return False

with ManagedConnection("postgres://localhost/app") as conn:
    print(f"using {conn!r}")
# => connecting to postgres://localhost/app
# => using <object object at 0x...>
# => closing connection
```

## Core concept — The easy way: `@contextlib.contextmanager`

Writing a whole class for simple cases is overkill. The `@contextmanager` decorator lets you write a **generator** where everything **before `yield`** is `__enter__` and everything **after `yield`** is `__exit__`. The value you `yield` is what gets bound after `as`.

```python
from contextlib import contextmanager

@contextmanager
def timer(label="block"):
    import time
    start = time.perf_counter()
    try:
        yield  # control passes to the with-body here
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label} took {elapsed:.4f}s")

with timer("compute"):
    sum(range(1_000_000))
# => compute took 0.0119s
```

**The `try/finally` is not optional.** Without it, if the `with` body raises, the exception propagates *up through the `yield`*, and your teardown code after `yield` never runs. Wrapping the `yield` in `try/finally` is the generator equivalent of "always run cleanup."

```python
@contextmanager
def opened(path, mode="r"):
    f = open(path, mode)
    try:
        yield f          # f is bound after `as`
    finally:
        f.close()        # runs even if the body raises

with opened("data.txt", "w") as f:
    f.write("written via a generator-based CM\n")
```

If you want to *handle* (and optionally suppress) an exception in a generator CM, catch it around the `yield`:

```python
@contextmanager
def ignore_value_errors():
    try:
        yield
    except ValueError as exc:
        print(f"suppressed: {exc}")
        # by NOT re-raising, we suppress — equivalent to __exit__ returning True

with ignore_value_errors():
    raise ValueError("oops")
print("continues normally")
# => suppressed: oops
# => continues normally
```

### Class vs generator — which to use?

| Use a **class** when... | Use `@contextmanager` when... |
| --- | --- |
| You need to store state across methods or expose attributes/methods on the object | The logic is a simple linear setup → yield → teardown |
| The same object is also used outside `with` | It is a one-off helper |
| You want explicit, readable `__enter__`/`__exit__` | You want the least boilerplate |

Reach for `@contextmanager` first; drop to a class when it genuinely needs to be one.

## Core concept — Multiple context managers in one `with`

You can manage several resources at once. They enter left-to-right and exit right-to-left (like nested blocks):

```python
with open("in.txt") as src, open("out.txt", "w") as dst:
    dst.write(src.read())
# dst.__exit__ runs first, then src.__exit__
```

For long lists, Python 3.10+ supports a **parenthesized** form so you can wrap onto multiple lines cleanly:

```python
with (
    open("a.txt") as a,
    open("b.txt") as b,
    open("c.txt", "w") as out,
):
    out.write(a.read())
    out.write(b.read())
```

The comma form is exactly equivalent to nesting:

```python
with A() as a:
    with B() as b:
        ...
```

— so if `B()`'s `__enter__` raises, `a` has already been entered and *will* be cleaned up. Good.

## Core concept — The `contextlib` toolbox

### `suppress` — ignore specific exceptions cleanly

Replaces the `try/except/pass` anti-pattern with intent-revealing code:

```python
from contextlib import suppress
import os

# Old way:
try:
    os.remove("maybe_missing.txt")
except FileNotFoundError:
    pass

# Better:
with suppress(FileNotFoundError):
    os.remove("maybe_missing.txt")
# If the file is missing, the block exits quietly; any OTHER error still propagates.
```

`suppress` only swallows the exception types you name (and their subclasses). It does **not** suppress everything — never use it as a blanket `except:`.

### `closing` — turn a `.close()`-able object into a context manager

For objects that have a `.close()` method but aren't context managers themselves:

```python
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen("https://example.com")) as page:
    data = page.read()
# page.close() is called on the way out, even if read() raises
```

`closing(thing)` simply calls `thing.close()` in its `__exit__`. (Modern `urlopen` is already a CM, but `closing` is invaluable for older or third-party APIs that only offer `.close()`.)

### `ExitStack` — manage a dynamic, variable number of resources

When you don't know at write-time how many resources you'll open (e.g. a list of files), `ExitStack` lets you enter them in a loop and guarantees all of them are cleaned up:

```python
from contextlib import ExitStack

filenames = ["a.txt", "b.txt", "c.txt"]

with ExitStack() as stack:
    files = [stack.enter_context(open(name)) for name in filenames]
    # all files are open here; if open() fails partway, already-opened files still close
    first_lines = [f.readline() for f in files]
# every file is closed here, in reverse order
```

`ExitStack` also supports registering arbitrary cleanup callbacks and **transferring** ownership of the cleanups (via `stack.pop_all()`) — handy for "open these now, but only close them later if something goes wrong":

```python
from contextlib import ExitStack

def open_all_or_none(names):
    with ExitStack() as stack:
        files = [stack.enter_context(open(n)) for n in names]
        # If we reach here, every file opened successfully.
        # Hand the cleanups to the caller instead of running them now:
        stack.pop_all()
        return files
    # On any failure above, the with-block cleans up partial opens automatically.
```

## Core concept — Real-world uses

### Files (the canonical case)

```python
with open("report.csv", "w", encoding="utf-8") as f:
    f.write("id,name\n1,alice\n")
```

Always pass `encoding=` explicitly for text files — relying on the platform default is a portability bug.

### Locks (thread safety)

`threading.Lock` is a context manager: entering acquires, exiting releases — even if the body raises.

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    with lock:            # acquire; auto-release on exit
        counter += 1
# Equivalent to lock.acquire(); try: ...; finally: lock.release()
```

### Database transactions and sessions

This is where context managers shine for backend developers. In Django, `transaction.atomic()` commits on clean exit and **rolls back** if the block raises:

```python
from django.db import transaction
from myapp.models import Account

def transfer(from_id, to_id, amount):
    with transaction.atomic():
        src = Account.objects.select_for_update().get(pk=from_id)
        dst = Account.objects.select_for_update().get(pk=to_id)
        src.balance -= amount
        dst.balance += amount
        src.save()
        dst.save()
    # commit happens here on success; any exception above => full rollback
```

This is the direct analogue of Laravel's `DB::transaction(fn () => ...)`.

In SQLAlchemy 2.x, the `Session` is a context manager, and `session.begin()` gives you transactional commit/rollback semantics:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("sqlite:///app.db")

# Pattern 1: session lifecycle (closes the session on exit)
with Session(engine) as session:
    user = session.get(User, 1)
    user.name = "updated"
    session.commit()

# Pattern 2: session + transaction in one (commits on success, rolls back on error)
with Session(engine) as session, session.begin():
    session.add(User(name="new"))
    # auto-commit on clean exit; auto-rollback if the block raises
```

FastAPI dependencies frequently use a generator-based pattern (`yield`) to hand a DB session to a route and guarantee it's closed afterward. Note the dependency is a **plain generator** — you do **not** decorate it with `@contextmanager`. FastAPI itself drives the generator: it runs the code up to `yield` before the request, injects the yielded value, then resumes the code after `yield` (your cleanup) once the response is sent.

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_session():            # plain generator, NOT @contextmanager
    session = Session(engine)
    try:
        yield session
    finally:
        session.close()

@app.get("/users/{user_id}")
def read_user(user_id: int, session: Session = Depends(get_session)):
    return session.get(User, user_id)
```

If you want to reuse the *same* generator as a normal context manager outside of FastAPI's injection (e.g. in a script or background task), wrap it with `contextlib.contextmanager` separately — but the version injected via `Depends` must stay an undecorated generator.

### Temporarily changing state

Context managers are perfect for "set X, do work, restore X no matter what." Example: temporarily change the working directory.

```python
import os
from contextlib import contextmanager

@contextmanager
def change_dir(path):
    previous = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(previous)   # restored even if the body raises

with change_dir("/tmp"):
    print(os.getcwd())  # => /tmp
print(os.getcwd())      # => back to wherever you were
```

Python's own `decimal` module ships `localcontext()` for exactly this kind of scoped state change, and `unittest.mock.patch` is a context manager that restores patched attributes on exit.

## Core concept — Reentrancy and reuse (the subtle bit)

Most context managers are **single-use**: a `@contextmanager` generator function builds a *fresh* generator each call, so re-entering the **same instance** breaks.

```python
from contextlib import contextmanager

@contextmanager
def cm():
    yield

c = cm()
with c:
    pass
with c:          # RuntimeError: generator didn't yield (already exhausted)
    pass
```

The fix is to call the factory again each time: `with cm():` (a new object per `with`). Some managers are explicitly **reentrant** (you can nest the *same* object), e.g. `threading.RLock` (a *reentrant* lock — the same thread can acquire it multiple times) and `contextlib.suppress` (documented as reentrant). But do not assume reentrancy; the safe default is **one context manager object per `with` block**.

```python
import threading

rlock = threading.RLock()
with rlock:
    with rlock:   # fine — RLock is reentrant for the same thread
        pass
```

## Common mistakes & gotchas

- **Returning `True` from `__exit__` by accident.** This swallows every exception in the block. Default to `return False` (or just don't return anything).
- **Forgetting `try/finally` around `yield`** in a `@contextmanager` generator. If the body raises, your teardown is skipped. Always wrap the `yield`.
- **Using bare `open()` without `with`.** You're now responsible for closing it on every code path, including exceptions.
- **Reusing a single-use context manager object** in two `with` blocks → `RuntimeError`. Create a new one each time.
- **Treating `suppress` like a blanket `except:`.** It only suppresses the exception types you list. Don't list `Exception` to "make errors go away."
- **Doing heavy work inside `__init__` instead of `__enter__`.** Resource acquisition belongs in `__enter__` so it happens when the `with` starts, not when the object is constructed.
- **Assuming `__enter__` returns the manager.** It returns whatever you tell it to; `as VAR` binds *that*. For `open()` it's the file; for a custom manager it might be a connection or `self`.
- **Forgetting that `with a, b:` cleans up `a` even if `b`'s setup fails** — that's correct behaviour, but people sometimes expect (wrongly) that a failure means nothing was acquired.
- **Relying on garbage collection / `__del__` for cleanup.** GC timing is not deterministic; use `with`.

## Best practices

- Always use `with` for files, locks, sockets, and DB sessions/transactions.
- Prefer `@contextmanager` for simple cases; use a class when you need stored state, exposed methods, or reuse outside `with`.
- Put `yield` (or the body between acquire and release) inside `try/finally`.
- Return falsy from `__exit__` unless you have a deliberate, narrow reason to suppress.
- Pass `encoding=` explicitly when opening text files.
- Use `contextlib.suppress` instead of `try/except/pass`, and only for specific exceptions.
- Use `ExitStack` when the number of resources is dynamic or conditional.
- Make each `with` use a fresh manager object unless the type documents reentrancy.
- For backend work, lean on framework-provided managers: Django `transaction.atomic()`, SQLAlchemy `Session` / `session.begin()`.

## Interview Q&A

**Q: What problem do context managers solve?**
**A:** Deterministic setup and teardown of resources. They guarantee that cleanup code (closing a file, releasing a lock, committing/rolling back a transaction) runs when a block exits — whether it exits normally or via an exception — without you writing a `try/finally` by hand each time.

**Q: What two methods make up the context manager protocol, and when is each called?**
**A:** `__enter__` and `__exit__`. `__enter__` runs at the start of the `with`; its return value is bound to the name after `as`. `__exit__(exc_type, exc_value, traceback)` always runs at the end — with all three arguments `None` on a clean exit, or with the exception details if the body raised.

**Q: What does the return value of `__exit__` control?**
**A:** Exception suppression. If `__exit__` returns a truthy value, an exception raised in the block is swallowed and execution continues after the `with`. If it returns falsy (the normal case), the exception propagates. Returning `True` accidentally is a classic bug because it silences errors.

**Q: Why is `open()` almost always used with `with`?**
**A:** Because `open()` returns a context manager whose `__exit__` closes the file. Using `with` guarantees the file descriptor is released on every path, including exceptions. Without `with`, you must call `.close()` yourself in a `finally`, and a missed close leaks descriptors.

**Q: How does `@contextlib.contextmanager` work?**
**A:** It turns a generator into a context manager. Code before `yield` is the `__enter__` phase; the yielded value is bound after `as`; code after `yield` is the `__exit__` phase. You wrap the `yield` in `try/finally` so teardown runs even if the body raises. To suppress an exception, catch it around the `yield` and don't re-raise.

**Q: Why must the `yield` in a generator-based context manager usually be inside `try/finally`?**
**A:** When the `with` body raises, the exception is thrown *into* the generator at the `yield` point. Without `try/finally`, the code after `yield` (your cleanup) never executes. The `finally` guarantees teardown regardless of how the body exits.

**Q: How do multiple context managers in a single `with` behave?**
**A:** They're entered left-to-right and exited right-to-left (equivalent to nesting). If a later manager's `__enter__` fails, the already-entered managers are still cleaned up. Python 3.10+ allows a parenthesized, multi-line form for readability.

**Q: What's the difference between `contextlib.suppress` and a try/except/pass?**
**A:** Functionally similar, but `suppress` is more readable and intent-revealing, and it only swallows the specific exception types you pass (and subclasses). It's a context manager, so `with suppress(FileNotFoundError): os.remove(p)` cleanly ignores just that error while letting others propagate.

**Q: When would you use `ExitStack`?**
**A:** When the number of resources is dynamic or conditional — e.g. opening a list of files whose length you don't know at write time. You call `stack.enter_context(...)` in a loop, and the stack guarantees every entered resource is cleaned up in reverse order, including partial setups that fail midway.

**Q: How do context managers relate to database transactions?**
**A:** They map perfectly: enter begins/holds a transaction, a clean exit commits, and an exception triggers rollback. Django's `with transaction.atomic():` and SQLAlchemy's `with session.begin():` are the canonical examples — analogous to Laravel's `DB::transaction(closure)`.

**Q: What is a reentrant context manager?**
**A:** One whose *same instance* can be safely entered again while already active (e.g. nested), like `threading.RLock` or `contextlib.suppress`. Most context managers — especially `@contextmanager` generators — are single-use; re-entering the same object raises `RuntimeError`. The safe default is one fresh manager per `with`.

**Q: Does `with` create a new scope for the variables defined inside it?**
**A:** No. Unlike a function, `with` does not introduce a new scope — names assigned inside the block (and the `as` target) remain accessible after it. The "scope" `with` controls is the resource's lifetime, not variable visibility.

## Practice exercises

1. Write a class-based context manager `Tag(name)` that prints `<name>` on enter and `</name>` on exit, so that `with Tag("p"): print("hello")` prints the three lines `<p>`, `hello`, `</p>`.

2. Rewrite exercise 1 using `@contextlib.contextmanager` instead of a class.

3. Write a context manager `temp_attr(obj, name, value)` that temporarily sets `obj.name = value` for the duration of the block and restores the original value (or removes the attribute if it didn't exist) on exit — even if the block raises.

4. Using `ExitStack`, write a function `concat_files(paths, out_path)` that opens an arbitrary list of input files plus one output file and writes the concatenation of all inputs to the output. All handles must close even if one `open` fails.

5. Write a context manager `swallow(*exc_types)` (using either style) that suppresses the given exception types — then show why `contextlib.suppress` already does this.

6. Demonstrate the single-use vs reentrant difference: show a `@contextmanager`-based CM raising `RuntimeError` when its single instance is reused, and a `threading.RLock` being safely nested.

<details><summary>Solutions</summary>

```python
# 1. Class-based Tag
class Tag:
    def __init__(self, name):
        self.name = name
    def __enter__(self):
        print(f"<{self.name}>")
        return self
    def __exit__(self, exc_type, exc_value, tb):
        print(f"</{self.name}>")
        return False

with Tag("p"):
    print("hello")
# => <p>
# => hello
# => </p>


# 2. Generator-based Tag
from contextlib import contextmanager

@contextmanager
def tag(name):
    print(f"<{name}>")
    try:
        yield
    finally:
        print(f"</{name}>")

with tag("p"):
    print("hello")


# 3. Temporarily set an attribute, restore on exit
from contextlib import contextmanager

_MISSING = object()

@contextmanager
def temp_attr(obj, name, value):
    original = getattr(obj, name, _MISSING)
    setattr(obj, name, value)
    try:
        yield obj
    finally:
        if original is _MISSING:
            delattr(obj, name)          # didn't exist before; remove it
        else:
            setattr(obj, name, original)  # restore previous value

class Config:
    debug = False

cfg = Config()
with temp_attr(cfg, "debug", True):
    print(cfg.debug)  # => True
print(cfg.debug)      # => False (restored, even if body had raised)


# 4. Concatenate a dynamic list of files with ExitStack
from contextlib import ExitStack

def concat_files(paths, out_path):
    with ExitStack() as stack:
        out = stack.enter_context(open(out_path, "w", encoding="utf-8"))
        for p in paths:
            f = stack.enter_context(open(p, "r", encoding="utf-8"))
            out.write(f.read())
    # all inputs + output closed here, in reverse order


# 5. swallow vs contextlib.suppress
from contextlib import contextmanager, suppress

@contextmanager
def swallow(*exc_types):
    try:
        yield
    except exc_types:    # a tuple of exception classes works directly
        pass             # not re-raising == suppressing

with swallow(ZeroDivisionError):
    1 / 0
print("continued")  # => continued

# contextlib.suppress does exactly this, more robustly:
with suppress(ZeroDivisionError):
    1 / 0
print("continued again")  # => continued again


# 6. Single-use vs reentrant
from contextlib import contextmanager
import threading

@contextmanager
def once():
    yield

c = once()
with c:
    pass
try:
    with c:   # reusing the SAME exhausted generator instance
        pass
except RuntimeError as e:
    print(f"single-use error: {type(e).__name__}")  # => single-use error: RuntimeError

rlock = threading.RLock()
with rlock:
    with rlock:   # same instance, nested — RLock is reentrant
        print("reentrant ok")  # => reentrant ok
```

</details>

## Cheat sheet

```python
# Use a resource safely
with open("f.txt", encoding="utf-8") as f:
    data = f.read()

# Protocol (class-based)
class CM:
    def __enter__(self):  return resource      # bound after `as`
    def __exit__(self, et, ev, tb):  return False  # falsy => don't suppress

# Generator-based (preferred for simple cases)
from contextlib import contextmanager
@contextmanager
def cm():
    setup()
    try:
        yield value          # value bound after `as`
    finally:
        teardown()           # ALWAYS runs

# Multiple managers (3.10+ parenthesized form)
with (open("a") as a, open("b", "w") as b):
    ...

# Dynamic count of resources
from contextlib import ExitStack
with ExitStack() as stack:
    files = [stack.enter_context(open(p)) for p in paths]

# Ignore specific errors
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove(path)

# .close()-only objects
from contextlib import closing
with closing(thing_with_close()) as t:
    ...
```

| Thing | Meaning |
| --- | --- |
| `__enter__` return | Value bound after `as` |
| `__exit__` returns falsy | Exception propagates (normal) |
| `__exit__` returns truthy | Exception suppressed (rare, deliberate) |
| `@contextmanager` before `yield` | Setup / `__enter__` |
| `@contextmanager` after `yield` | Teardown / `__exit__` (wrap in `try/finally`) |
| `with a, b:` | Enter L→R, exit R→L; `a` cleaned up if `b` setup fails |
| `suppress(E)` | Swallow only exception type `E` |
| `closing(x)` | Call `x.close()` on exit |
| `ExitStack` | Manage a variable number of resources |
| Single-use vs `RLock`/`suppress` | New object per `with`, unless documented reentrant |

| Backend manager | Behaviour |
| --- | --- |
| `with transaction.atomic():` (Django) | Commit on success, rollback on exception |
| `with Session(engine) as s:` (SQLAlchemy) | Open + auto-close session |
| `with session.begin():` (SQLAlchemy) | Commit on success, rollback on exception |
| `with lock:` (`threading.Lock`) | Acquire on enter, release on exit |
