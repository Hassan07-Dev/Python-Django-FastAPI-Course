# 16 — Error Handling & Exceptions

> Week 2 · Python Advanced

## What you will learn

- The full `try` / `except` / `else` / `finally` machinery and what each clause is actually for
- The exception class hierarchy: `BaseException`, `Exception`, and the built-ins you meet daily
- How to catch *specific* exceptions, catch *several* types at once, and why a bare `except:` is a trap
- How to grab the exception object with `as e` and read what went wrong
- Raising your own errors with `raise`, re-raising, and exception chaining via `raise ... from ...`
- Designing **custom exception classes** and small hierarchies for your apps and APIs
- The Pythonic **EAFP vs LBYL** philosophy ("ask forgiveness, not permission")
- When `assert` is appropriate and when it will silently betray you
- Guaranteed cleanup with `finally` (and why context managers are usually better)
- Logging exceptions the right way with `logging.exception`
- A side-by-side map from PHP/Laravel `try/catch` to Python

## PHP/Laravel bridge

If you've written `try { ... } catch (\Exception $e) { ... } finally { ... }` in PHP, you already understand 80% of this. Python uses `except` instead of `catch`, has an extra `else` clause PHP lacks, and — crucially — Python's culture *leans into* exceptions for normal control flow far more than PHP does (the EAFP style below). In Laravel, you lean on the framework's exception handler (`App\Exceptions\Handler`) and helpers like `abort(404)` or `throw_if()`; in Django/DRF and FastAPI you'll do the equivalent with `Http404`, DRF's `APIException`, or FastAPI's `HTTPException`. One sharp difference: PHP separates `Error` (engine-level, e.g. `TypeError`) from `Exception`, and you typically catch `\Throwable` to get both — Python instead puts almost everything under one `Exception` base, so catching `Exception` covers the cases you normally care about.

---

## Core concept: the anatomy of `try`

A full statement has four clauses. You rarely use all four at once, but you should know each one's job.

```python
def read_config(path):
    try:
        f = open(path, "r")          # 1. code that might fail
    except FileNotFoundError as e:    # 2. runs ONLY if a matching error is raised
        print(f"No config at {path}: {e}")
        return {}
    else:                             # 3. runs ONLY if try succeeded (no exception)
        data = f.read()
        return {"raw": data}
    finally:                          # 4. ALWAYS runs, success or failure
        # 'f' might not exist if open() failed, so guard it
        if "f" in dir() and not f.closed:
            f.close()

print(read_config("nope.txt"))   # => No config at nope.txt: [Errno 2] No such file...
                                 # => {}
```

Mental model for each clause:

| Clause     | Runs when                                  | Use it for                                        |
| ---------- | ------------------------------------------ | ------------------------------------------------- |
| `try`      | always (it's the protected block)          | the risky operation, kept as small as possible    |
| `except`   | a matching exception was raised in `try`   | handling/recovering from that specific failure    |
| `else`     | `try` finished with **no** exception       | the "happy path" follow-up code                   |
| `finally`  | always — exception or not, even on `return`| cleanup: close files, release locks, etc.         |

### Why `else` exists (the beginner's "wait, why not just put it in `try`?")

Put only the line that can fail inside `try`, and the follow-up in `else`. This keeps the `except` from accidentally catching errors raised by your *success* code.

```python
import json

# BAD: if process() raises ValueError, this except swallows it,
# misreporting it as bad JSON.
def parse_bad(text):
    try:
        data = json.loads(text)
        return process(data)        # a ValueError here gets caught below!
    except ValueError:
        return None

# GOOD: only json.loads can trip the except; process() runs in else.
def parse_good(text):
    try:
        data = json.loads(text)
    except ValueError:              # narrowly about parsing
        return None
    else:
        return process(data)        # its own errors propagate normally

def process(data):
    return data["id"]               # may raise KeyError — and that's fine
```

### `finally` always wins

`finally` runs even when you `return`, `break`, or `continue` out of the block, and even if a new exception is propagating. This is what makes it reliable for cleanup.

```python
def demo():
    try:
        return "from try"
    finally:
        print("finally still runs")   # printed BEFORE the function returns

print(demo())
# => finally still runs
# => from try
```

> Gotcha: a `return` inside `finally` overrides any `return`/exception from the `try`. Don't do it — it silently swallows errors.

---

## Core concept: the exception hierarchy

Every exception is an object whose class sits in a tree. Knowing the tree tells you what a given `except` will and won't catch — because **`except SomeClass` catches that class and all its subclasses.**

```text
BaseException
 ├── SystemExit            # raised by sys.exit() — don't accidentally catch this
 ├── KeyboardInterrupt     # Ctrl-C — don't accidentally catch this either
 ├── GeneratorExit
 └── Exception             # <-- catch THIS (or subclasses) for normal errors
      ├── ArithmeticError
      │    └── ZeroDivisionError
      ├── LookupError
      │    ├── KeyError
      │    └── IndexError
      ├── ValueError
      │    └── UnicodeDecodeError
      ├── TypeError
      ├── OSError
      │    ├── FileNotFoundError
      │    ├── PermissionError
      │    └── TimeoutError
      ├── AttributeError
      ├── RuntimeError
      │    └── RecursionError
      └── StopIteration
```

Two practical consequences:

1. `except Exception` catches almost everything you care about — but **not** `SystemExit` or `KeyboardInterrupt`, because those inherit from `BaseException` directly. That's deliberate: a broad `except Exception` won't stop a user from Ctrl-C-ing your program or break a clean `sys.exit()`.
2. You can catch a whole family by catching a parent. `except LookupError` catches both `KeyError` and `IndexError`.

```python
d = {"a": 1}
try:
    d["missing"]
except LookupError as e:           # KeyError is a LookupError
    print(type(e).__name__, "->", e)   # => KeyError -> 'missing'
```

The most common built-ins, with when you see them:

| Exception           | Typical trigger                                      |
| ------------------- | --------------------------------------------------- |
| `ValueError`        | right type, wrong value: `int("abc")`               |
| `TypeError`         | wrong type: `len(5)`, `"a" + 1`                      |
| `KeyError`          | missing dict key: `d["nope"]`                        |
| `IndexError`        | list index out of range: `[][0]`                     |
| `AttributeError`    | missing attribute: `None.foo`                        |
| `FileNotFoundError` | opening a file that doesn't exist                    |
| `ZeroDivisionError` | `1 / 0`                                              |
| `StopIteration`     | iterator exhausted (usually handled by `for` loops) |

---

## Core concept: specific vs broad, and why bare `except:` is bad

A **bare `except:`** catches *everything*, including `KeyboardInterrupt` and `SystemExit`. It hides bugs, makes Ctrl-C impossible, and tells you nothing about what failed.

```python
# NEVER do this
try:
    risky()
except:                    # catches BaseException — even Ctrl-C
    pass                   # and silently eats it. A debugging nightmare.
```

Catch the narrowest exception that you can actually handle. If `risky()` can raise a `TimeoutError` you know how to retry, catch *that* — and let everything else surface.

```python
import time

def fetch_with_retry(fetch, attempts=3):
    for i in range(attempts):
        try:
            return fetch()
        except TimeoutError:           # only the thing we know how to recover from
            if i == attempts - 1:
                raise                  # out of retries -> re-raise the last one
            time.sleep(2 ** i)         # back off and try again
```

### Catching multiple types

Group types you'd handle the same way into a tuple. Use `as e` to inspect.

```python
def to_int(value):
    try:
        return int(value)
    except (ValueError, TypeError) as e:   # both mean "can't convert"
        print(f"Cannot convert {value!r}: {type(e).__name__}")
        return None

to_int("3")     # => 3
to_int("x")     # => Cannot convert 'x': ValueError -> None
to_int(None)    # => Cannot convert None: TypeError  -> None
```

> The parentheses are required: `except ValueError, TypeError:` is a syntax error in Python 3 (it was old Python 2 syntax meaning something else entirely).

### When broad *is* OK

A top-level catch-all is reasonable at an application boundary — a request handler, a worker loop, a CLI entry point — where the goal is "log it and keep the server alive." Even then, catch `Exception` (not bare `except:`), log it, and usually re-raise or convert to a clean error response.

```python
import logging
log = logging.getLogger(__name__)

def handle_request(req):
    try:
        return do_work(req)
    except Exception:                  # boundary: don't let one bad request crash the worker
        log.exception("Request failed")   # logs message + full traceback
        return {"error": "internal error"}, 500
```

---

## Core concept: raising, re-raising, and chaining

### Raising

`raise` throws an exception. Pass a message to the constructor.

```python
def set_age(age):
    if not isinstance(age, int):
        raise TypeError(f"age must be int, got {type(age).__name__}")
    if age < 0:
        raise ValueError(f"age must be non-negative, got {age}")
    return age

set_age(-1)   # => ValueError: age must be non-negative, got -1
```

Pick the *right* built-in: `ValueError` for bad values, `TypeError` for bad types, `KeyError`/`LookupError` for missing items. Don't reach for a generic `Exception("...")` — specific types let callers catch precisely.

### Re-raising

Inside an `except`, a bare `raise` re-throws the *current* exception with its original traceback intact. Use it to do something (log, clean up) then let it propagate.

```python
def process(item):
    try:
        return transform(item)
    except ValueError:
        log.warning("Skipping malformed item %r", item)
        raise                          # bare raise: re-throw the SAME exception, full traceback preserved
```

> `raise e` (with the variable) also works but can muddy the traceback in older patterns. Prefer a bare `raise` when re-raising the exception you just caught.

### Chaining with `raise ... from ...`

When you catch a low-level error and raise a higher-level one, **chain** them so the traceback shows both. This answers "what was the *root* cause?"

```python
class ConfigError(Exception):
    pass

def load_settings(path):
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError as e:
        raise ConfigError(f"Settings file required: {path}") from e
        # Traceback shows:
        #   FileNotFoundError ...
        #   The above exception was the direct cause of the following exception:
        #   ConfigError: Settings file required: ...
```

- `raise NewError from original` → explicit cause, prints *"direct cause of"*.
- If you raise inside an `except` *without* `from`, Python auto-chains it as the *context*, printing *"During handling of the above exception, another exception occurred"*. That's still informative but reads like an accident.
- `raise NewError from None` → **suppress** the chain when the original is noise (e.g. converting an internal `KeyError` into a clean public error).

```python
def get_user(users, user_id):
    try:
        return users[user_id]
    except KeyError:
        raise LookupError(f"No user {user_id}") from None   # hide the internal KeyError
```

---

## Core concept: custom exceptions and hierarchies

Define your own by subclassing `Exception` (or a more specific built-in). The simplest form is one line.

```python
class PaymentError(Exception):
    """Base for anything that goes wrong taking a payment."""
```

A small hierarchy lets callers catch broadly *or* narrowly. This is the single most useful pattern for libraries and services.

```python
class PaymentError(Exception):
    """Base — catch this to handle ANY payment failure."""

class CardDeclined(PaymentError):
    """The bank said no."""
    def __init__(self, code, message="Card declined"):
        self.code = code                         # carry structured data
        super().__init__(f"{message} (code={code})")

class InsufficientFunds(CardDeclined):
    """A specific kind of decline."""

# Caller can be as specific or as broad as it likes:
def charge(amount):
    raise InsufficientFunds(code="51")

try:
    charge(100)
except InsufficientFunds as e:        # most specific
    print("Top up:", e.code)          # => Top up: 51
except CardDeclined as e:             # broader
    print("Declined:", e.code)
except PaymentError:                  # broadest in our domain
    print("Some payment problem")
```

Two rules of thumb:

- **Always inherit from `Exception`** (directly or transitively), never `BaseException`.
- **Give your package one base exception** (e.g. `PaymentError`) so users can `except PaymentError` to catch everything from your code without catching unrelated bugs.

### How this maps to web frameworks

You'll subclass framework-specific bases so the framework turns your exception into the right HTTP response:

```python
# Django REST Framework — DRF renders this as JSON with the given status
from rest_framework.exceptions import APIException

class ServiceUnavailable(APIException):
    status_code = 503
    default_detail = "Service temporarily unavailable, try again later."
    default_code = "service_unavailable"

# FastAPI — raise HTTPException and FastAPI builds the response
from fastapi import HTTPException

def get_item(item_id: int):
    item = db.get(item_id)
    if item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return item
```

(Django's plain views use `from django.http import Http404; raise Http404("...")`, the analogue of Laravel's `abort(404)`.)

---

## Core concept: EAFP vs LBYL (the Pythonic mindset)

Two philosophies for guarding against errors:

- **LBYL** — *Look Before You Leap*: check preconditions first, then act. Common in PHP (`if (isset($arr['k'])) { ... }`).
- **EAFP** — *Easier to Ask Forgiveness than Permission*: just do it, and handle the exception if it fails. **This is the Pythonic default.**

```python
data = {"name": "Ada"}

# LBYL — check, then access
if "name" in data:
    name = data["name"]
else:
    name = "anonymous"

# EAFP — try, then catch (more idiomatic)
try:
    name = data["name"]
except KeyError:
    name = "anonymous"
```

Why EAFP is often preferred:

1. **No race conditions.** With LBYL, the world can change between the check and the use (a file deleted, a key removed by another thread). EAFP acts atomically.
2. **Less duplicate work.** LBYL often re-does the lookup (`in` then `[]`); EAFP does it once.
3. **It reads as the happy path.** The normal case isn't buried in `if` nesting.

That said, EAFP isn't dogma. For a single dict key, `data.get("name", "anonymous")` beats both. Use LBYL when the check is cheap and the exception would be expensive or noisy, and when failure is *expected and common* (exceptions are cheap to raise but not free, so don't use them for tight inner loops where misses are the norm).

```python
# Best of all here: the dict's own API
name = data.get("name", "anonymous")
```

---

## Core concept: `assert` and why it's not error handling

`assert cond, msg` raises `AssertionError` if `cond` is falsy. It documents an invariant *you believe is always true* — a sanity check for programmers, not a way to validate user input.

```python
def average(nums):
    assert len(nums) > 0, "average() requires a non-empty list"   # internal invariant
    return sum(nums) / len(nums)
```

The killer caveat: **`assert` is stripped when Python runs with `-O` (optimized mode).** In production deploys that use `python -O`, your asserts vanish.

```bash
python -O app.py     # ALL assert statements are removed; their conditions never run
```

So **never** use `assert` for:

- Validating request/user input (use `raise ValueError(...)` or framework validation)
- Security or authorization checks (they'd disappear under `-O`)
- Any check the program's correctness depends on at runtime

Use `assert` for: tests, internal "this can't happen" invariants, and `assert` statements in `pytest`.

---

## Core concept: cleanup with `finally` vs context managers

`finally` guarantees cleanup. But for resources that have a defined "close" step (files, DB connections, locks, network sockets), a **context manager** (`with`) is cleaner and harder to get wrong — it calls cleanup for you, even on exceptions.

```python
# Works, but verbose and easy to mishandle (the guard for 'f', etc.)
f = open("data.txt")
try:
    process(f.read())
finally:
    f.close()

# Idiomatic: 'with' closes the file automatically, even if process() raises
with open("data.txt") as f:
    process(f.read())
# file is guaranteed closed here
```

Multiple resources, and combining `with` + `try`:

```python
import sqlite3

def transfer(db_path, src, dst, amount):
    # NOTE: sqlite3's context manager commits on success / rolls back on error,
    # but it does NOT close the connection — so we close it ourselves in finally.
    conn = sqlite3.connect(db_path)
    try:
        with conn:                            # commits or rolls back the transaction
            conn.execute("UPDATE acct SET bal = bal - ? WHERE id = ?", (amount, src))
            conn.execute("UPDATE acct SET bal = bal + ? WHERE id = ?", (amount, dst))
    except sqlite3.Error:
        log.exception("Transfer failed; transaction will roll back")
        raise
    finally:
        conn.close()                          # the with block won't close it for us
```

Reach for `with` first; use `finally` for cleanup that isn't wrapped by an existing context manager.

---

## Core concept: logging exceptions properly

Printing exceptions throws away the traceback and clutters stdout. Use the `logging` module. Inside an `except` block, **`logging.exception(...)`** logs your message *plus the full traceback* automatically — it's the single most useful logging call you'll make.

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(name)s: %(message)s")
log = logging.getLogger(__name__)

def run(job):
    try:
        do(job)
    except Exception:
        log.exception("Job %s failed", job.id)   # ERROR level + traceback, no need to pass e
        # equivalent to: log.error("...", exc_info=True)
```

Key points:

- `log.exception(msg)` only works *inside* an `except` block (it reads the current exception). It logs at `ERROR` level.
- Outside an except, or for non-error levels, pass `exc_info=True`: `log.warning("retrying", exc_info=True)`.
- Use **lazy `%s` formatting** (`log.error("bad %s", x)`), not f-strings, so the string is only built if that level is enabled.
- **Don't** do `except Exception as e: log.error(e)` — that logs only the message and loses the traceback. Use `log.exception(...)`.

```python
# WRONG — loses the traceback, hard to debug later
except Exception as e:
    log.error(e)

# RIGHT — message + full traceback
except Exception:
    log.exception("operation failed")
```

---

## Common mistakes & gotchas

- **Bare `except:`** — catches `KeyboardInterrupt`/`SystemExit` and hides bugs. Catch `Exception` at most, and only at boundaries.
- **`except Exception: pass`** — the silent swallow. At minimum log it; usually you should re-raise or convert it.
- **Too-broad `try`** — wrapping ten lines so one `except` catches errors from the wrong line. Keep `try` to the risky statement; push the rest into `else`.
- **`log.error(e)` instead of `log.exception(...)`** — drops the traceback.
- **`return` inside `finally`** — silently discards exceptions and other returns. Almost always a bug.
- **Using `assert` for validation** — vanishes under `python -O`. Use `raise`.
- **Forgetting the tuple in multi-catch** — `except (A, B):` not `except A, B:`.
- **Catching then `raise e`** instead of bare `raise` — can reset/clutter the traceback; bare `raise` preserves the original.
- **No chaining** — `raise NewError(...)` inside an except without `from e` hides the root cause (Python still auto-chains as context, but `from e` is clearer; `from None` deliberately hides it).
- **Raising generic `Exception`** — gives callers nothing to catch precisely. Use a specific or custom type.
- **Catching what you can't handle** — if you can't actually recover, don't catch it; let it propagate to a place that can (or to your logger at the boundary).

## Best practices

- Catch the **narrowest** exception type you can meaningfully handle; let the rest propagate.
- Keep the `try` block **small**; use `else` for the success path.
- Define a **base exception per package/app**, with a small subclass hierarchy.
- Prefer **EAFP**, but use `dict.get`, `getattr(obj, name, default)` and similar when the language already offers a clean default.
- Use **`with`** (context managers) for resource cleanup; reserve `finally` for the rest.
- **Chain** with `raise ... from ...` to preserve root causes; use `from None` to hide internal noise.
- Log with **`logging.exception`** inside excepts; never `print()` errors in real services.
- Reserve **`assert`** for invariants and tests, never for input validation or security.
- At app boundaries (HTTP handler, worker), catch `Exception`, **log it**, and return a clean error — don't crash the whole process over one request.
- Make exception **messages actionable**: include the offending value (`f"... got {value!r}"`) and what was expected.

## Interview Q&A

**Q: What's the difference between `Exception` and `BaseException`, and why does it matter?**
**A:** `BaseException` is the root of all exceptions. `Exception` is its subclass and the base for ordinary errors. A few exceptions — `SystemExit`, `KeyboardInterrupt`, `GeneratorExit` — inherit from `BaseException` directly, *not* `Exception`. So `except Exception` deliberately does **not** catch Ctrl-C or `sys.exit()`, which is what you want: a broad handler shouldn't trap a user trying to quit. Custom exceptions should always subclass `Exception`, never `BaseException`.

**Q: Why is a bare `except:` considered bad?**
**A:** It catches `BaseException`, including `KeyboardInterrupt` and `SystemExit`, so it can make a program impossible to interrupt or exit cleanly. It also hides every bug — typos, `AttributeError`s, anything — behind a single handler, making debugging miserable. Catch the specific type you can handle, or `Exception` at most, and always log.

**Q: What does the `else` clause in a `try` statement do, and when would you use it?**
**A:** `else` runs only if the `try` block completed with **no** exception. You use it to separate the "this might fail" code (in `try`) from the "do this on success" code (in `else`), so your `except` doesn't accidentally catch errors raised by the success path. It makes the `try` block minimal and the intent clear.

**Q: Explain EAFP vs LBYL. Which does Python prefer and why?**
**A:** LBYL ("look before you leap") checks preconditions before acting (`if key in d`). EAFP ("easier to ask forgiveness than permission") just does the operation and handles the exception (`try: d[key] except KeyError`). Python prefers EAFP because it avoids race conditions between check and use, avoids duplicate lookups, and reads as the happy path. LBYL still makes sense when failure is common and exceptions would be noisy, or when a clean default API exists (`dict.get`).

**Q: What does `raise ... from ...` do?**
**A:** It chains exceptions, recording an explicit cause. `raise NewError(...) from original` makes the traceback show the original as the "direct cause" of the new one, preserving the root cause when you translate a low-level error into a domain-level one. `from None` suppresses chaining when the internal exception is just noise. Raising inside an `except` without `from` still auto-chains as "context."

**Q: When re-raising inside an `except`, what's the difference between `raise` and `raise e`?**
**A:** Bare `raise` re-throws the currently-handled exception with its original traceback fully preserved — that's the idiomatic way to re-raise. `raise e` re-raises the same object but can produce a less clean traceback in some patterns and is more verbose. Use bare `raise` when you just want to log/cleanup and let it propagate.

**Q: Why shouldn't you use `assert` to validate user input?**
**A:** Because `assert` statements are removed entirely when Python runs with the `-O` (optimize) flag. In a production deploy using `-O`, every `assert`-based check silently disappears — including any security or validation logic. Use `assert` only for internal invariants and tests; use `raise ValueError(...)` (or framework validation) for input.

**Q: How do you log an exception with its traceback?**
**A:** Inside an `except` block, call `logging.exception("message")` (or `log.error("message", exc_info=True)`). It logs at ERROR level and automatically attaches the full traceback. Avoid `log.error(e)` — that records only the message text and throws the traceback away, which makes post-mortem debugging much harder.

**Q: How do you catch multiple exception types in one handler?**
**A:** Group them in a tuple: `except (ValueError, TypeError) as e:`. The parentheses are required. You'd do this when several distinct exceptions should be handled the same way. Alternatively, catch a shared parent class (e.g. `LookupError` covers both `KeyError` and `IndexError`).

**Q: Does `finally` run if the `try` block executes a `return`?**
**A:** Yes. `finally` runs no matter how the `try`/`except` block is exited — normal completion, an exception, `return`, `break`, or `continue`. The `finally` body executes before control actually leaves. (Caveat: a `return` *inside* `finally` will override the original return/exception, which is a bug to avoid.)

**Q: How should you design custom exceptions for a library or service?**
**A:** Subclass `Exception`, define a single base exception for your package (e.g. `PaymentError`), and derive specific exceptions from it (`CardDeclined`, `InsufficientFunds`). This lets callers catch broadly (`except PaymentError`) or narrowly. Attach structured data (like an error code) as attributes, and write actionable messages. In web frameworks, subclass the framework's base (DRF's `APIException`, FastAPI's `HTTPException`) so it maps to the right HTTP response.

**Q: How does Python error handling compare to PHP/Laravel?**
**A:** Conceptually identical: `try`/`except` vs `try`/`catch`, `raise` vs `throw`, `finally` in both. Differences: Python adds an `else` clause; Python's culture uses exceptions for normal control flow (EAFP) more than PHP does; PHP splits `Error` and `Exception` under `Throwable` (so you catch `\Throwable` for both), whereas Python puts nearly everything under `Exception`. Laravel centralizes handling in `App\Exceptions\Handler`; Django/DRF and FastAPI map exceptions to HTTP responses via `Http404`, `APIException`, and `HTTPException`.

## Practice exercises

1. **Safe division.** Write `safe_divide(a, b)` that returns the quotient, but returns `None` and logs (with `logging.exception`) if `b` is `0`. Catch only `ZeroDivisionError`.

2. **Config loader with chaining.** Write `load_json_config(path)` that reads and parses a JSON file. If the file is missing, raise a custom `ConfigError` chained from the original `FileNotFoundError`. If the JSON is malformed, raise `ConfigError` chained from the `JSONDecodeError`. Define `ConfigError` yourself.

3. **EAFP rewrite.** Given this LBYL code, rewrite it in EAFP style:
   ```python
   def get_first_char(s):
       if isinstance(s, str) and len(s) > 0:
           return s[0]
       return None
   ```

4. **Custom hierarchy.** Design exceptions for a tiny auth module: a base `AuthError`, plus `InvalidCredentials` and `TokenExpired` subclasses. Write `authenticate(user, pw)` that raises `InvalidCredentials`, and show a caller that handles `TokenExpired` and `InvalidCredentials` differently but can also fall back to catching `AuthError`.

5. **Retry with backoff.** Write `fetch(url_fn, attempts=3)` that calls `url_fn()`, retries on `TimeoutError` with exponential backoff, and re-raises the final error if all attempts fail. Other exceptions should propagate immediately.

6. **Spot the bug.** Explain what's wrong with each snippet and fix it:
   ```python
   # (a)
   try:
       value = compute()
   except:
       value = 0

   # (b)
   def average(nums):
       assert nums, "need numbers"
       return sum(nums) / len(nums)   # used to validate API input

   # (c)
   try:
       data = parse(raw)
   except Exception as e:
       log.error(e)
   ```

<details><summary>Solutions</summary>

```python
import json
import logging

logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)


# 1. Safe division
def safe_divide(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        log.exception("Division by zero: %s / %s", a, b)   # ERROR + traceback
        return None

print(safe_divide(10, 2))   # => 5.0
print(safe_divide(10, 0))   # => None  (and an ERROR log with traceback)


# 2. Config loader with chaining
class ConfigError(Exception):
    """Raised when configuration can't be loaded."""

def load_json_config(path):
    try:
        with open(path) as f:
            text = f.read()
    except FileNotFoundError as e:
        raise ConfigError(f"Config file not found: {path}") from e
    try:
        return json.loads(text)
    except json.JSONDecodeError as e:
        raise ConfigError(f"Config file is not valid JSON: {path}") from e


# 3. EAFP rewrite
def get_first_char(s):
    try:
        return s[0]                       # works for any indexable, fails cleanly otherwise
    except (TypeError, IndexError):       # not indexable / empty
        return None

print(get_first_char("hi"))   # => h
print(get_first_char(""))     # => None
print(get_first_char(None))   # => None


# 4. Custom auth hierarchy
class AuthError(Exception):
    """Base for authentication problems."""

class InvalidCredentials(AuthError):
    pass

class TokenExpired(AuthError):
    pass

def authenticate(user, pw):
    if (user, pw) != ("ada", "secret"):
        raise InvalidCredentials(f"Bad credentials for {user!r}")
    return {"user": user, "token": "abc123"}

def login(user, pw):
    try:
        return authenticate(user, pw)
    except TokenExpired:
        return "Please log in again"
    except InvalidCredentials:
        return "Wrong username or password"
    except AuthError:                     # any other auth problem
        return "Authentication failed"

print(login("ada", "nope"))   # => Wrong username or password


# 5. Retry with backoff
import time

def fetch(url_fn, attempts=3):
    for i in range(attempts):
        try:
            return url_fn()
        except TimeoutError:
            if i == attempts - 1:
                raise                     # exhausted retries: re-raise the last timeout
            time.sleep(2 ** i)            # 1s, 2s, 4s, ...
        # any other exception is NOT caught -> propagates immediately


# 6. Spot the bug
# (a) Bare `except:` catches KeyboardInterrupt/SystemExit and hides bugs.
#     Fix: catch the specific error you expect.
try:
    value = compute()
except ValueError:        # or whatever compute() legitimately raises
    value = 0

# (b) `assert` is stripped under `python -O`, so input validation vanishes.
#     Fix: raise a real exception.
def average(nums):
    if not nums:
        raise ValueError("need at least one number")
    return sum(nums) / len(nums)

# (c) `log.error(e)` drops the traceback.
#     Fix: use log.exception() inside the except block.
try:
    data = parse(raw)
except Exception:
    log.exception("Failed to parse input")
    raise                 # (re-raise or handle — don't silently continue)
```

</details>

## Cheat sheet

```python
# Full skeleton
try:
    risky()
except SpecificError as e:     # narrow first
    handle(e)
except (TypeError, ValueError) as e:   # multiple types -> tuple
    handle(e)
except Exception:              # broad, only at boundaries; log it
    log.exception("failed")
    raise
else:
    on_success()              # ran only if try had no exception
finally:
    cleanup()                 # always runs

# Raising / re-raising / chaining
raise ValueError("bad value: got -1")     # raise with a clear message
raise                                      # re-raise current exc (preserve traceback)
raise NewError("...") from original        # chain: show root cause
raise NewError("...") from None            # suppress the chain

# Custom exceptions
class AppError(Exception): ...             # one base per package
class NotFound(AppError): ...              # subclass for specifics

# Logging (inside except)
log.exception("msg %s", arg)               # ERROR + traceback
log.error("msg", exc_info=True)            # same, any custom message/level

# EAFP over LBYL
val = d.get("k", default)                  # cleanest for dicts
try: val = d["k"]                          # EAFP when no clean default
except KeyError: val = default

# assert = invariants/tests ONLY (stripped by python -O)
assert invariant, "should never happen"
```

| Need to...                          | Use                                   |
| ----------------------------------- | ------------------------------------- |
| Run cleanup no matter what          | `finally:` (or better, `with`)        |
| Run code only on success            | `else:`                               |
| Catch several types the same way    | `except (A, B):`                      |
| Inspect the error object            | `except E as e:`                      |
| Re-raise, keep traceback            | bare `raise`                          |
| Translate error, keep root cause    | `raise New() from e`                  |
| Hide the internal cause             | `raise New() from None`               |
| Log error + traceback               | `log.exception("...")`                |
| Validate input                      | `raise ValueError(...)` (never `assert`) |
| Default for a missing dict key      | `d.get(k, default)`                   |

| PHP / Laravel                    | Python                                  |
| -------------------------------- | --------------------------------------- |
| `try { } catch (\Ex $e) { }`     | `try: ... except Ex as e: ...`          |
| `catch (\Throwable $e)`          | `except Exception:` (most cases)        |
| `finally { }`                    | `finally:`                              |
| `throw new Ex("...")`            | `raise Ex("...")`                       |
| `throw_if($c, Ex::class)`        | `if c: raise Ex(...)`                   |
| `abort(404)`                     | `raise Http404()` / `HTTPException(404)`|
| `App\Exceptions\Handler`         | DRF `APIException` / FastAPI handlers   |
