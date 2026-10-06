# 22 — Pythonic Idioms & PEP 8

> Week 2 · Python language & ecosystem

## What you will learn

- What people mean when they say code is "Pythonic," and how the **Zen of Python** guides those judgments.
- The **PEP 8** rules that actually matter in interviews and code review: naming, indentation, line length, import ordering, whitespace.
- The core **idioms** that separate a Python beginner from someone who looks senior: truthiness checks, `enumerate`/`zip`, unpacking, `dict.get`/`setdefault`, comprehensions, context managers, EAFP, `str.join`, and f-strings.
- The **anti-patterns** to never write (and what to write instead).
- The modern **tooling** that enforces all of this for you: `black`, `ruff`, `isort`, `mypy`.
- A **review-ready checklist** you can run through before opening a pull request.

## PHP/Laravel bridge

In PHP, the community converged on **PSR-1/PSR-12** for style and **PSR-4** for autoloading, and you lean on `php-cs-fixer` or Laravel Pint to auto-format. Python's equivalent is **PEP 8** (style) plus **PEP 20** (the Zen — a philosophy doc with no PHP analogue) plus formatters like **black** and linters like **ruff**. The big mental shift: PHP is happy with `camelCase` methods and braces; Python uses `snake_case` for functions/variables and **significant whitespace** instead of braces, so indentation is not cosmetic — it *is* the syntax. And where Laravel rewards clever fluent chains, idiomatic Python rewards plain, obvious code: "There should be one — and preferably only one — obvious way to do it."

## Core concept: what does "Pythonic" mean?

"Pythonic" code uses the language the way its designers intended: it is readable, leans on built-in idioms instead of reinventing them, and prefers clarity over cleverness. The same algorithm can be written in a C-style way or a Pythonic way, and both run — but only one reads naturally to a Python developer.

```python
# Non-Pythonic (C/PHP habits): manual index, manual length, manual append
names = ["alice", "bob", "carol"]
result = []
for i in range(len(names)):
    result.append(names[i].upper())
# result => ['ALICE', 'BOB', 'CAROL']

# Pythonic: iterate directly, build with a comprehension
result = [name.upper() for name in names]
# result => ['ALICE', 'BOB', 'CAROL']
```

Both produce the same list. The second is shorter, has no off-by-one risk, and signals to a reviewer "I know Python." That difference is exactly what interviewers probe for.

### The Zen of Python (PEP 20)

Python ships its own design philosophy. Run it:

```python
import this
# Prints "The Zen of Python, by Tim Peters" followed by 19 aphorisms.
```

You do not need to memorize all 19, but these guide nearly every style decision:

| Aphorism | What it means in practice |
| --- | --- |
| Beautiful is better than ugly. | Format consistently; let `black` decide. |
| Explicit is better than implicit. | No magic; prefer clear names and obvious flow. |
| Simple is better than complex. | Don't reach for a metaclass when a function works. |
| Flat is better than nested. | Use guard clauses / early returns to avoid deep `if` pyramids. |
| Readability counts. | Code is read far more than written. |
| Errors should never pass silently. | Don't write a bare `except: pass`. |
| There should be one obvious way to do it. | Idioms exist so everyone writes the same thing. |
| Now is better than never. | Ship; iterate. |

## Core concept: PEP 8 essentials

PEP 8 is the official style guide. Memorize the parts below — they come up in reviews constantly. In real projects you won't apply most of these by hand because `black` and `ruff` do it, but you must be able to *read* them and explain them.

### Naming conventions

| Thing | Convention | Example |
| --- | --- | --- |
| Module / package | `lowercase`, short, `snake_case` if needed | `user_service.py` |
| Class | `PascalCase` (a.k.a. CapWords) | `class OrderRepository:` |
| Exception | `PascalCase`, usually ending in `Error` | `class PaymentError(Exception):` |
| Function / method | `snake_case` | `def get_active_users():` |
| Variable | `snake_case` | `total_price = 0` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRIES = 3` |
| "Internal" name | single leading underscore | `_cache`, `_build_url()` |
| Name-mangled (avoid clash in subclass) | double leading underscore | `self.__token` |
| Type variable | `PascalCase`, short | `T`, `KeyT` |

```python
# PEP 8 names in context
MAX_RETRIES = 3                       # constant

class PaymentGateway:                 # PascalCase class
    def charge_card(self, amount):    # snake_case method
        retry_count = 0               # snake_case local
        self._last_status = None      # _internal attribute
        return amount
```

A subtle one PHP devs miss: there is **no `camelCase` for functions** in Python. `def getActiveUsers()` is technically legal but flags you instantly as a non-native.

### Indentation, line length, blank lines

- **4 spaces** per indentation level. Never tabs. Never mix.
- **Line length:** PEP 8 says 79 characters; `black` defaults to **88**, which is the modern de-facto standard. Pick one project-wide and let the formatter enforce it.
- **Two blank lines** between top-level functions and classes; **one blank line** between methods inside a class.

```python
import json


def load_config(path):
    with open(path) as f:
        return json.load(f)


class Server:
    def start(self):
        ...

    def stop(self):
        ...
```

### Imports ordering

Imports go at the **top of the file**, one per line, grouped into three blocks separated by a blank line, in this order:

1. Standard library
2. Third-party packages
3. Local / first-party modules

```python
# 1. standard library
import os
from datetime import datetime

# 2. third-party
import httpx
from fastapi import FastAPI

# 3. local
from app.config import settings
from app.services import user_service
```

`isort` (and `ruff`) automate this exact grouping and sorting, so you rarely order them by hand — but you should recognize the structure.

### Whitespace rules that trip people up

```python
# YES                              # NO
x = 1                              x=1
def f(a, b):                       def f(a,b):
items[1]                           items [1]
d["key"]                           d ["key"]
f(a, b=2)                          f(a, b = 2)   # no spaces around = for kwargs/defaults
total = a + b                      total=a+b
x = -1                             x = - 1
```

Rule of thumb: spaces around binary operators and after commas; **no** spaces around `=` when it's a keyword argument or default value; **no** space before `(` `[` or before a `:` in a slice.

## Core concept: the idioms that matter

This is the heart of the lesson. Each idiom below is something interviewers expect you to reach for automatically.

### Truthiness instead of explicit comparisons

Empty collections, `0`, `""`, and `None` are all falsy. Use that.

```python
items = []

# Pythonic
if not items:
    print("empty")

# Non-Pythonic
if len(items) == 0:
    print("empty")
if items == []:
    print("empty")
```

But be careful — *comparison to `None` is the exception*: use `is`, not truthiness, when the distinction between "empty" and "missing" matters.

```python
def fetch(limit=None):
    # WRONG: 0 is falsy, so `if not limit` treats limit=0 like "no limit"
    if limit is None:
        limit = 100
    return limit

# Always compare to None / True / False with `is`
if value is None:
    ...
if found is True:   # rarely needed; usually just `if found:`
    ...
```

### `enumerate` instead of `range(len(...))`

When you need the index *and* the value, use `enumerate`. It can even start at a custom number.

```python
colors = ["red", "green", "blue"]

for index, color in enumerate(colors):
    print(index, color)
# => 0 red / 1 green / 2 blue

for line_no, color in enumerate(colors, start=1):
    print(line_no, color)
# => 1 red / 2 green / 3 blue
```

### `zip` to iterate multiple sequences together

```python
names = ["alice", "bob"]
scores = [90, 85]

for name, score in zip(names, scores):
    print(f"{name}: {score}")
# => alice: 90 / bob: 85

# Build a dict from two parallel lists
lookup = dict(zip(names, scores))
# => {'alice': 90, 'bob': 85}

# Python 3.10+: strict=True raises if lengths differ (catches bugs)
for n, s in zip(names, scores, strict=True):
    ...
```

### Unpacking (tuple, starred, dict)

```python
# Multiple assignment / swap without a temp variable
a, b = 1, 2
a, b = b, a            # a => 2, b => 1

# Starred unpacking grabs "the rest"
first, *rest = [1, 2, 3, 4]      # first => 1, rest => [2, 3, 4]
head, *middle, tail = [1, 2, 3, 4]   # head=1, middle=[2,3], tail=4

# Unpacking in a loop
pairs = [("a", 1), ("b", 2)]
for key, value in pairs:
    print(key, value)

# Dict merge / spread (Python 3.9+ has |; ** works everywhere)
defaults = {"page": 1, "size": 20}
overrides = {"size": 50}
params = {**defaults, **overrides}   # => {'page': 1, 'size': 50}
params = defaults | overrides        # same result, 3.9+
```

### `dict.get` and `dict.setdefault`

Avoid `KeyError` and the clumsy "check then assign" dance.

```python
config = {"host": "localhost"}

# get with a default instead of KeyError or `if "port" in config`
port = config.get("port", 8000)      # => 8000

# setdefault: read-or-insert in one call — great for grouping
words = ["apple", "ant", "bee"]
groups = {}
for word in words:
    groups.setdefault(word[0], []).append(word)
# groups => {'a': ['apple', 'ant'], 'b': ['bee']}
```

For the grouping case specifically, the standard library gives you something even cleaner:

```python
from collections import defaultdict

groups = defaultdict(list)
for word in words:
    groups[word[0]].append(word)
# groups => {'a': ['apple', 'ant'], 'b': ['bee']}
```

### Comprehensions over manual loops

List, dict, and set comprehensions replace the build-empty-then-append pattern.

```python
nums = [1, 2, 3, 4, 5]

squares = [n * n for n in nums]                  # list comp
even_squares = [n * n for n in nums if n % 2 == 0]  # with filter
square_map = {n: n * n for n in nums}            # dict comp
unique_lengths = {len(w) for w in ["a", "bb", "cc"]}  # set comp => {1, 2}

# Generator expression: lazy, memory-friendly — note () not []
total = sum(n * n for n in nums)                 # no intermediate list built
```

Keep comprehensions to **one level of logic**. If you need nested loops *and* conditions *and* transformations, a plain `for` loop is more readable — "Simple is better than complex."

### Context managers with `with`

A `with` block guarantees cleanup (closing files, releasing locks, committing/rolling back transactions) even if an exception is raised — like a `try/finally` you don't have to write.

```python
# Files: closed automatically, even on error
with open("data.txt", "w") as f:
    f.write("hello")
# f is closed here, guaranteed

# Multiple managers in one statement
with open("in.txt") as src, open("out.txt", "w") as dst:
    dst.write(src.read())

# Write your own with contextlib
from contextlib import contextmanager

@contextmanager
def timer(label):
    import time
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.4f}s")

with timer("work"):
    sum(range(1_000_000))
# => work: 0.0123s
```

### EAFP vs LBYL

Python prefers **EAFP** — "Easier to Ask Forgiveness than Permission": just try the operation and catch the exception. The opposite, **LBYL** ("Look Before You Leap"), checks preconditions first and is prone to race conditions and duplicated logic.

```python
data = {"name": "Ada"}

# EAFP (Pythonic): attempt, handle failure
try:
    name = data["name"]
except KeyError:
    name = "unknown"

# LBYL (less Pythonic here): check first
if "name" in data:
    name = data["name"]
else:
    name = "unknown"
```

EAFP shines when the "happy path" is the common case and checks would be redundant or racy (e.g. a file existing when you open it). For a simple dict default, `data.get("name", "unknown")` beats both.

### `str.join` instead of `+=` in a loop

Strings are **immutable**, so `s += piece` in a loop creates a new string each iteration — quadratic time and garbage. Build a list (or use a generator) and `join` once.

```python
parts = ["GET", "/users", "HTTP/1.1"]

# Pythonic: one allocation
line = " ".join(parts)        # => 'GET /users HTTP/1.1'

# Anti-pattern: repeated concatenation in a loop
line = ""
for p in parts:
    line += p + " "           # O(n^2), trailing space bug too

# join works with a generator for transforms
csv_row = ",".join(str(n) for n in [1, 2, 3])   # => '1,2,3'
```

### f-strings for formatting

f-strings (PEP 498, Python 3.6+) are the modern, fast, readable way to format. Prefer them over `%`-formatting and `str.format()`.

```python
name, score = "Ada", 0.9576

f"{name} scored {score:.1%}"        # => 'Ada scored 95.8%'
f"{name!r}"                         # => "'Ada'"  (repr)
f"{score:.2f}"                      # => '0.96'
f"{1_000_000:,}"                    # => '1,000,000'

# 3.8+ self-documenting debug form: prints "score=0.9576"
f"{score=}"                         # => 'score=0.9576'

# Multi-line and expressions inside braces
items = [1, 2, 3]
f"There are {len(items)} items, summing to {sum(items)}"
# => 'There are 3 items, summing to 6'
```

Never use an f-string for SQL — that's an injection hole. Use parameterized queries (covered in the ORM lessons).

## Common mistakes & gotchas

- **Mutable default arguments.** `def add(item, bucket=[])` shares one list across all calls. Use `None` as the sentinel.
  ```python
  def add(item, bucket=None):
      if bucket is None:
          bucket = []
      bucket.append(item)
      return bucket
  ```
- **Comparing to `None`/`True`/`False` with `==`.** Use `is` / `is not`. `if x == None` works but is wrong style and can be fooled by overloaded `__eq__`.
- **Bare `except:`.** It swallows `KeyboardInterrupt` and `SystemExit` and hides bugs. Catch specific exceptions: `except (KeyError, ValueError):`.
- **`except Exception: pass`** silently. At minimum log it — "Errors should never pass silently."
- **`len(x) == 0` / `len(x) > 0`.** Use `if not x:` / `if x:`.
- **`range(len(seq))` to index.** Use `enumerate` (and `zip` for parallel lists).
- **Building strings with `+=` in a loop.** Use `"".join(...)`.
- **camelCase functions/variables.** Python is `snake_case`; `PascalCase` is for classes only.
- **`from module import *`.** Pollutes the namespace and breaks linters/readers. Import names explicitly.
- **Mixing tabs and spaces.** A `TabError` waiting to happen; configure your editor for 4 spaces.
- **Wildcard truthiness on numbers.** `if not count:` treats `0` as "missing" — use `is None` when zero is a valid value.
- **Manual `f.close()` / `lock.release()`** instead of `with`. Easy to leak on an exception path.

## Best practices

- Let the **tools** own formatting and import order so reviews focus on logic, not style. Run `black` + `ruff` in CI and a pre-commit hook.
- Write **guard clauses** (early `return`/`raise`) to keep functions flat instead of deeply nested.
- Prefer **built-ins and the standard library** (`enumerate`, `zip`, `any`, `all`, `sum`, `sorted`, `collections`, `itertools`) before writing your own loop.
- Keep comprehensions **simple and single-purpose**; fall back to a loop when logic grows.
- Use **type hints** on public functions; they document intent and let `mypy` catch bugs.
- Choose **descriptive names** over comments. `active_users` beats `au` plus a comment.
- Reach for **EAFP** when the happy path dominates; use `.get()` for simple dict defaults.
- Use **`with`** for every resource that needs cleanup.
- Pin one **line length** (88 is standard with black) and one Python version per project.

## Core concept: the tooling

You will be expected to know these by name and what each does. They compose: format first, then lint, then type-check.

| Tool | Job | One-liner |
| --- | --- | --- |
| **black** | Opinionated **formatter** — rewrites your code to a canonical style. | `black .` |
| **isort** | Sorts and groups **imports** (stdlib / third-party / local). | `isort .` |
| **ruff** | Extremely fast **linter** (and now formatter + import sorter); a Rust drop-in for flake8/isort/pyflakes/pyupgrade. | `ruff check . --fix` |
| **flake8** | The classic **linter** (pyflakes + pycodestyle + mccabe). Largely superseded by ruff. | `flake8 .` |
| **mypy** | Static **type checker** — verifies your type hints are consistent. | `mypy app/` |

```bash
# Install
pip install black ruff isort mypy

# Typical workflow
black .                 # auto-format the whole project
ruff check . --fix      # lint and auto-fix what it safely can
isort .                 # (or let ruff do it) sort imports
mypy app/               # type-check the package
```

Modern teams increasingly use **ruff for everything** — it lints, sorts imports, and even formats (`ruff format`), replacing flake8 + isort and overlapping with black. Configure it in `pyproject.toml`:

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]  # pycodestyle, pyflakes, isort, pyupgrade, bugbear

[tool.black]
line-length = 88

[tool.mypy]
python_version = "3.12"
strict = true
```

A **pre-commit** hook runs these automatically before every commit, so unformatted code can never land:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

## Interview Q&A

**Q: What does "Pythonic" mean?**
**A:** Writing code that follows Python's idioms and philosophy — readable, explicit, leaning on built-ins (`enumerate`, comprehensions, `with`, unpacking) rather than translating patterns from C/Java/PHP. The Zen of Python (`import this`) captures the values: simple over complex, explicit over implicit, readability counts, one obvious way to do it.

**Q: What is PEP 8 and is it enforced by the interpreter?**
**A:** PEP 8 is the official **style guide** for Python code — naming, indentation, line length, imports, whitespace. It is *not* enforced by the interpreter (your code runs regardless), but it's enforced socially via code review and mechanically via tools like `ruff`/`flake8` and formatters like `black`. The only whitespace the interpreter *does* enforce is indentation, since it delimits blocks.

**Q: What are the naming conventions?**
**A:** `snake_case` for functions, methods, and variables; `PascalCase` (CapWords) for classes and exceptions; `UPPER_SNAKE_CASE` for constants; `lowercase`/`snake_case` for modules and packages. A single leading underscore signals "internal"; a double leading underscore triggers name mangling. There is no `camelCase` for functions in idiomatic Python.

**Q: Why use `enumerate` instead of `range(len(seq))`?**
**A:** It's more readable, eliminates off-by-one and manual-indexing bugs, gives you the value directly instead of `seq[i]`, and supports a `start=` argument. `range(len(...))` is a C habit that interviewers flag immediately.

**Q: Explain EAFP versus LBYL.**
**A:** LBYL ("Look Before You Leap") checks preconditions before acting; EAFP ("Easier to Ask Forgiveness than Permission") just tries the operation and catches the exception. Python favors EAFP because it's often cleaner and avoids race conditions (e.g. a file getting deleted between the check and the open). LBYL can be fine for simple, non-racy checks.

**Q: Why is `s += x` in a loop a problem, and what's the fix?**
**A:** Strings are immutable, so each `+=` allocates a brand-new string and copies everything — O(n²) overall. The fix is to collect pieces in a list and call `"".join(parts)` once, which is a single allocation. For two strings, `+` is fine.

**Q: What's wrong with a mutable default argument?**
**A:** Default values are evaluated **once**, at function-definition time, so a `def f(x, acc=[])` shares the *same* list across every call — state leaks between invocations. Use `None` as a sentinel and create the mutable object inside the function body.

**Q: How should you compare against `None`?**
**A:** With the identity operator: `if x is None:` / `if x is not None:`. `None` is a singleton, so identity is correct and fast, and it can't be fooled by a custom `__eq__`. Don't use `if not x` when `0`, `""`, or `[]` are valid values, because they're falsy too.

**Q: When should you NOT use a comprehension?**
**A:** When it gets hard to read — multiple nested loops combined with conditions and transformations, or any logic with side effects. Comprehensions are for building a collection from an expression; if there's branching, accumulation, or `try/except` involved, a plain loop is clearer. "Simple is better than complex."

**Q: What's the difference between `black`, `ruff`, `isort`, and `mypy`?**
**A:** `black` is an opinionated auto-formatter; `isort` sorts/groups imports; `ruff` is a very fast linter (and now also a formatter and import-sorter) that replaces flake8 + isort + several plugins; `mypy` is a static type checker for your type hints. Formatting/linting fix *style and likely bugs*; mypy catches *type* errors before runtime.

**Q: What is `with` and what problem does it solve?**
**A:** `with` invokes a context manager, which guarantees setup and teardown around a block — most commonly closing files, releasing locks, or committing/rolling back transactions — even if the block raises. It replaces error-prone manual `try/finally` cleanup. You can author your own with a class implementing `__enter__`/`__exit__` or with `@contextlib.contextmanager`.

**Q: How do f-strings compare to `%` and `.format()`?**
**A:** f-strings (3.6+) embed expressions directly in the literal, are the fastest of the three, and are the most readable. They support format specs (`:.2f`), conversions (`!r`), thousands separators (`:,`), and the self-documenting `f"{x=}"` debug form. Prefer them — but never build SQL with them; use parameterized queries.

## Practice exercises

Try each before opening the solutions.

1. **De-C-ify a loop.** Rewrite this Pythonically:
   ```python
   names = ["ann", "ben", "cy"]
   out = []
   for i in range(len(names)):
       out.append(str(i) + ": " + names[i].upper())
   ```

2. **Fix the default-argument bug.** This function accumulates across calls — fix it so each call with no `tags` starts fresh:
   ```python
   def tag(item, tags=[]):
       tags.append(item)
       return tags
   ```

3. **Word grouping.** Given `words = ["apple", "avocado", "banana", "cherry", "blueberry"]`, build a dict mapping each first letter to the list of words starting with it. Use the idiomatic tool.

4. **Safe config read.** Given `cfg = {"host": "db", "port": 0}`, write code that reads `port` defaulting to `5432` *only when the key is absent* — note that `0` is a valid port and must be preserved.

5. **Lint it in your head.** List every PEP 8 / idiom violation in this snippet, then rewrite it:
   ```python
   import os,sys
   def GetUsers( db ):
     u=[]
     for i in range(len(db)):
       if db[i]['active']==True:
         u.append(db[i])
     return u
   ```

6. **Build a string the right way.** Given `fields = ["id", "name", "email"]`, produce the SQL-ish fragment `"id, name, email"` without using `+=`.

<details>
<summary>Solutions</summary>

**1.**
```python
names = ["ann", "ben", "cy"]
out = [f"{i}: {name.upper()}" for i, name in enumerate(names)]
# => ['0: ANN', '1: BEN', '2: CY']
```

**2.**
```python
def tag(item, tags=None):
    if tags is None:
        tags = []
    tags.append(item)
    return tags
# Each call without `tags` now starts with a fresh list.
```

**3.**
```python
from collections import defaultdict

words = ["apple", "avocado", "banana", "cherry", "blueberry"]
groups = defaultdict(list)
for word in words:
    groups[word[0]].append(word)
# dict(groups) =>
# {'a': ['apple', 'avocado'], 'b': ['banana', 'blueberry'], 'c': ['cherry']}

# setdefault alternative (no import):
# groups = {}
# for word in words:
#     groups.setdefault(word[0], []).append(word)
```

**4.**
```python
cfg = {"host": "db", "port": 0}
port = cfg.get("port", 5432)   # => 0, because the key exists
# `cfg.get("port") or 5432` would WRONGLY return 5432, since 0 is falsy.
```

**5.** Violations: imports not on separate lines (`import os,sys`), and both `os` and `sys` are unused; `GetUsers` should be `snake_case`; spaces inside `( db )`; `u` is non-descriptive; missing spaces around `=`; 2-space indent should be 4; `range(len(...))` should be iteration/comprehension; `== True` should be plain truthiness. Rewrite (drop the unused imports entirely):
```python
def get_active_users(db):
    return [user for user in db if user["active"]]
```

**6.**
```python
fields = ["id", "name", "email"]
fragment = ", ".join(fields)   # => 'id, name, email'
```

</details>

## Cheat sheet

```python
# Truthiness
if not items: ...            # empty list/str/dict/None/0
if x is None: ...            # identity for None/True/False

# Loops
for i, v in enumerate(seq, start=1): ...
for a, b in zip(xs, ys, strict=True): ...

# Unpacking
a, b = b, a
first, *rest = seq
merged = {**d1, **d2}        # or d1 | d2 (3.9+)

# Dicts
val = d.get("k", default)
d.setdefault("k", []).append(x)
from collections import defaultdict   # for grouping

# Comprehensions
[f(x) for x in xs if cond]           # list
{k: v for k, v in pairs}             # dict
sum(x*x for x in xs)                 # generator (lazy)

# Resources & errors
with open(p) as f: ...               # auto-close
try: v = d["k"]                      # EAFP
except KeyError: v = default

# Strings
" ".join(parts)                      # not += in a loop
f"{name} = {value:.2f}"              # f-string
f"{value=}"                          # debug form (3.8+)
```

| Topic | Rule |
| --- | --- |
| Indent | 4 spaces, never tabs |
| Line length | 88 (black) / 79 (PEP 8) |
| Functions/vars | `snake_case` |
| Classes/exceptions | `PascalCase` |
| Constants | `UPPER_SNAKE_CASE` |
| Imports | stdlib / third-party / local, blank-line separated |
| Blank lines | 2 between top-level defs, 1 between methods |
| `==` for None? | No — use `is` |

| Tool | Role | Command |
| --- | --- | --- |
| black | formatter | `black .` |
| ruff | linter (+format/imports) | `ruff check . --fix` |
| isort | import sorter | `isort .` |
| mypy | type checker | `mypy app/` |

**Review-ready checklist (run before every PR):**

- [ ] `black .` and `ruff check . --fix` pass with no diff.
- [ ] `mypy` passes; public functions have type hints.
- [ ] No `range(len(...))`, no `+=` string building in loops, no bare `except:`.
- [ ] No mutable default arguments; `None` sentinel used instead.
- [ ] `is None` (not `== None`); truthiness used for empties (but not where `0`/`""` are valid).
- [ ] Resources use `with`; dict access uses `.get`/`setdefault` where appropriate.
- [ ] Names follow PEP 8; imports grouped/sorted; no `import *`.
- [ ] Comprehensions stay simple; complex logic uses a plain loop.
