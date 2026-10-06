# 10 — Modules, Packages & Imports

> Week 1 · Python fundamentals

## What you will learn

- What a **module** is (spoiler: it's just a `.py` file) and what actually happens the first time you import one.
- The three import forms: `import x`, `from x import y`, and `import x as alias` — and when to use each.
- How Python *finds* a module: the **module search path** (`sys.path`) and where it comes from.
- **Packages**, the role of `__init__.py`, and how a folder becomes importable.
- **Absolute vs relative imports** and why one of them keeps breaking your scripts.
- The `if __name__ == "__main__":` idiom — what it means and why every serious file has it.
- Running code two ways: `python script.py` vs `python -m package.module`, and why they behave differently.
- The dreaded **circular import** and concrete ways to dodge it.
- A whirlwind tour of the **standard library** (`os`, `sys`, `math`, `random`, `datetime`, `json`, `pathlib`, `collections`).
- Installing third-party packages with **pip** and importing them.
- How to **structure a small project** into clean packages.

## PHP/Laravel bridge

In PHP you `require`/`include` files, but in practice Composer + PSR-4 autoloading means you rarely write `require` by hand — you just `use App\Services\PaymentService;` and the autoloader maps the namespace to a file path. Python has **no autoloader**: you import explicitly, and an import literally *runs* the target file top to bottom the first time. Think of a Python module as one PHP file, a Python package as a namespaced directory (the `__init__.py` is loosely like a directory that PSR-4 knows about), and `pip` as `composer` (with `requirements.txt`/`pyproject.toml` playing the role of `composer.json`). The big mental shift: a `use` statement in PHP is a pure compile-time alias with no side effects, whereas a Python `import` executes code.

## Core concept: a module is just a `.py` file

Any file ending in `.py` is a module. Its name is the filename without the extension. Create `greetings.py`:

```python
# greetings.py
GREETING = "Hello"

def greet(name):
    return f"{GREETING}, {name}!"

print("greetings.py is being imported/run")  # a side effect — watch this
```

Now use it from another file in the same folder:

```python
# app.py
import greetings

print(greetings.greet("Sara"))   # => Hello, Sara!
print(greetings.GREETING)        # => Hello
```

Run it:

```bash
python app.py
# => greetings.py is being imported/run
# => Hello, Sara!
# => Hello
```

Notice the `print` inside `greetings.py` fired. **Importing a module executes its top-level code**, exactly once per program run. After that first execution, Python caches the module object in `sys.modules`, so a second `import greetings` anywhere in your program does *not* re-run the file — it just hands back the cached module.

```python
# proof.py
import greetings   # runs greetings.py, prints the side-effect line
import greetings   # cached — does NOT print again
print("done")
# => greetings.py is being imported/run
# => done
```

This caching is why import-time side effects (like that `print`, or opening a DB connection at module top level) are sneaky: they happen the moment *anyone* imports the file, and only once. Keep heavy work *inside functions*, not at module top level.

## Core concept: the three import forms

### `import module`

```python
import math

print(math.sqrt(16))   # => 4.0
print(math.pi)         # => 3.141592653589793
```

You always prefix names with the module: `math.sqrt`. This keeps the namespace clean and makes the origin obvious — a reader instantly knows `math.sqrt` comes from `math`.

### `from module import name`

```python
from math import sqrt, pi

print(sqrt(16))   # => 4.0
print(pi)         # => 3.141592653589793
```

This binds `sqrt` and `pi` directly into your namespace — no prefix needed. Convenient, but if two modules both export `loads`, the second import silently shadows the first. Be deliberate.

### `import module as alias` (and `from ... import ... as`)

```python
import datetime as dt
from collections import OrderedDict as ODict

now = dt.datetime.now()
d = ODict()
```

Aliases are great for long names or community conventions (`import numpy as np`, `import pandas as pd`).

### Avoid `from module import *`

```python
from math import *   # imports everything — DON'T do this in real code
print(sqrt(9))       # => 3.0
```

Star imports dump an unknown set of names into your namespace, clobber existing ones, and make it impossible to tell where a name came from. Linters flag it. The one acceptable place is an interactive REPL session.

| Form | You write | Pollutes namespace? | When to use |
|------|-----------|--------------------|-------------|
| `import math` | `math.sqrt(x)` | No | Default. Clear origin. |
| `from math import sqrt` | `sqrt(x)` | A little | A few hot names used often |
| `import numpy as np` | `np.array(...)` | No | Long names / conventions |
| `from math import *` | `sqrt(x)` | Heavily | Almost never |

## Core concept: how Python finds a module (`sys.path`)

When you write `import foo`, Python searches a list of directories in order and uses the **first** match. That list is `sys.path`:

```python
import sys
for p in sys.path:
    print(p)
# => ''                         (or the script's directory)
# => /usr/lib/python3.12
# => /usr/lib/python3.12/lib-dynload
# => .../site-packages         (where pip installs things)
```

`sys.path` is built from, roughly in order:

1. The directory of the script you ran (or the current directory in the REPL).
2. The `PYTHONPATH` environment variable, if set.
3. The standard library directories.
4. The `site-packages` directory (where `pip install` puts third-party packages).

Two practical consequences:

- **Naming a file after a stdlib module breaks everything.** If you create `random.py` in your project and then `import random`, *your* file wins because the script directory comes first. You'll get bizarre `AttributeError`s. Same trap with `json.py`, `email.py`, `string.py`, etc.
- You can inspect where a module actually came from with its `__file__`:

```python
import json
print(json.__file__)   # => /usr/lib/python3.12/json/__init__.py
```

You *can* mutate `sys.path` at runtime (`sys.path.append(...)`), but treat that as a last resort — proper packaging is cleaner.

## Core concept: packages and `__init__.py`

A **package** is a directory containing modules. Historically it needed an `__init__.py` file (which can be empty) to mark it as a package; the `__init__.py` runs when the package is first imported.

```
myproject/
├── app.py
└── store/
    ├── __init__.py
    ├── products.py
    └── orders.py
```

```python
# store/products.py
def list_products():
    return ["pen", "notebook"]
```

```python
# store/orders.py
def create_order(items):
    return {"items": items, "status": "new"}
```

```python
# app.py
import store.products
from store.orders import create_order

print(store.products.list_products())     # => ['pen', 'notebook']
print(create_order(["pen"]))              # => {'items': ['pen'], 'status': 'new'}
```

Dots are folders: `store.products` means "the `products` module inside the `store` package."

### What `__init__.py` is for

The `__init__.py` runs once when the package is imported and is the package's public face. A common pattern is to re-export the names you want users to reach easily:

```python
# store/__init__.py
from .products import list_products
from .orders import create_order

__all__ = ["list_products", "create_order"]   # what 'from store import *' exposes
```

Now callers can do `from store import list_products` instead of `from store.products import list_products`. That `.products` is a **relative import** — more on that next.

> Note on "namespace packages": since Python 3.3 a folder without `__init__.py` can still be imported as a (namespace) package. It works, but for ordinary application code keep adding `__init__.py` — it's explicit, lets you re-export, and avoids surprising edge cases.

## Core concept: absolute vs relative imports

**Absolute imports** spell out the full path from the project root (the top of `sys.path`):

```python
# store/orders.py
from store.products import list_products   # absolute — full path from root
```

**Relative imports** use leading dots to mean "relative to *this* package":

```python
# store/orders.py
from .products import list_products    # . = current package (store)
from . import products                 # import the sibling module itself
# from ..billing import invoice        # .. = parent package (one level up)
```

One dot = current package, two dots = parent package, and so on. Relative imports keep a package self-contained: rename the top-level folder and the internal `.products` references still work.

**The catch that bites every beginner:** relative imports only work when the file is run *as part of a package*, not as a top-level script. If `orders.py` uses `from .products import ...` and you run `python store/orders.py` directly, you get:

```
ImportError: attempted relative import with no known parent package
```

Why? Run directly, `orders.py` becomes the top-level module named `__main__` with no package context, so `.` refers to nothing. The fix is to run it as a module (next section) or use absolute imports in files you intend to run directly.

| | Absolute | Relative |
|---|----------|----------|
| Looks like | `from store.products import x` | `from .products import x` |
| Resolves from | top of `sys.path` | current package |
| Survives renaming top folder | No | Yes (internal refs) |
| Works with `python file.py`? | Often | No — needs package context |
| Recommended for | app entry points, clarity | imports *within* a package |

PEP 8 mildly prefers absolute imports for readability; relative imports are fine and idiomatic *inside* a package. Pick one style per package and stay consistent.

## Core concept: `if __name__ == "__main__":`

Every module has a built-in variable `__name__`. Its value depends on **how the file was loaded**:

- If the file is **run directly** (`python greetings.py`), `__name__ == "__main__"`.
- If the file is **imported** (`import greetings`), `__name__ == "greetings"` (the module's name).

This lets a file behave one way when run and another when imported:

```python
# fib.py
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a

def _demo():
    for i in range(8):
        print(fib(i), end=" ")
    print()

if __name__ == "__main__":
    _demo()   # only runs when you execute: python fib.py
```

```bash
python fib.py
# => 0 1 1 2 3 5 8 13
```

```python
import fib
print(fib.fib(10))   # => 55   (the demo did NOT run on import)
```

**Why it matters:** without that guard, importing `fib` would also execute `_demo()` and spam your output (or worse, run a server, send an email, wipe a table...). The idiom cleanly separates "library code to be imported" from "script behavior when run directly." It's the rough equivalent of guarding `php artisan`-style entry logic so it doesn't fire when the class is merely autoloaded. Put your CLI/demo/quick-test code under this guard.

## Core concept: running scripts vs modules (`python -m`)

There are two ways to launch Python code, and they set up `sys.path` and `__name__` differently.

**As a script** — `python path/to/file.py`:
- `sys.path[0]` becomes the *directory containing that file*.
- The file's `__name__` is `"__main__"`.
- Relative imports inside it fail (no package context).

**As a module** — `python -m package.module`:
- `sys.path[0]` becomes the *current working directory*.
- Python imports the module through the normal package machinery, so **relative imports work**.
- The module's `__name__` is still `"__main__"`, so the guard still fires.

Demonstration with the package from earlier. Put a runnable block in `orders.py`:

```python
# store/orders.py
from .products import list_products   # relative import

def create_order(items):
    return {"items": items, "status": "new"}

if __name__ == "__main__":
    print("Available:", list_products())
    print(create_order(["pen"]))
```

```bash
# From the myproject/ directory (the one ABOVE store/):

python store/orders.py
# => ImportError: attempted relative import with no known parent package

python -m store.orders
# => Available: ['pen', 'notebook']
# => {'items': ['pen'], 'status': 'new'}
```

Note the syntax: `-m` takes a **dotted module path**, not a file path — `store.orders`, never `store/orders.py`. This is also exactly how you run installed tools: `python -m pip install ...`, `python -m http.server`, `python -m venv .venv`, `python -m pytest`. Running them through `-m` guarantees you use the interpreter you intend.

## Core concept: circular imports

A circular import is when module A imports B while B imports A. Because importing runs the file top to bottom, one of them ends up reaching for a name the other hasn't defined *yet*.

```python
# a.py
from b import helper_b

def helper_a():
    return "A"

print(helper_b())
```

```python
# b.py
from a import helper_a   # a.py isn't finished running yet!

def helper_b():
    return "B"
```

```bash
python a.py
# => ImportError: cannot import name 'helper_a' from partially initialized module 'a'
#    (most likely due to a circular import)
```

The phrase **"partially initialized module"** is your tell: it's a circular import.

How to fix, roughly best to worst:

1. **Refactor to break the cycle.** Usually the two modules share something that belongs in a third module (`common.py`). Move it there; both import from `common`, the cycle disappears. This is the real fix.
2. **Import the module, not the name.** `import b` then call `b.helper_b()` later. Module-level `import b` only binds the module object; you don't need its contents until call time, by which point both modules are fully loaded.
3. **Import inside the function** (a "deferred"/local import). The import runs when the function is *called*, long after both modules finished loading:

   ```python
   # b.py
   def helper_b():
       from a import helper_a   # deferred — runs at call time, not import time
       return "B+" + helper_a()
   ```

4. **Type-only imports**: if you only need the name for type hints, guard it with `TYPE_CHECKING` so it never runs at import time:

   ```python
   from __future__ import annotations
   from typing import TYPE_CHECKING
   if TYPE_CHECKING:
       from a import HelperA   # only seen by type checkers, not at runtime
   ```

You'll meet circular imports constantly in Django (models importing each other) and FastAPI projects — knowing the "partially initialized" signature saves hours.

## Core concept: a quick tour of the standard library

Python's "batteries included" stdlib means a huge amount works with zero installs. The essentials for a backend dev:

### `os` and `sys` — operating system & interpreter

```python
import os, sys

print(os.getcwd())                     # current working directory
print(os.environ.get("HOME"))          # read an env var safely (None if missing)
print(os.path.join("a", "b", "c.txt")) # => a/b/c.txt (OS-correct separator)

print(sys.argv)        # command-line args: ['script.py', 'arg1', ...]
print(sys.version)     # interpreter version string
sys.exit(0)            # exit with status code (0 = success)
```

### `math` and `random`

```python
import math, random

print(math.ceil(4.1))      # => 5
print(math.floor(4.9))     # => 4
print(math.gcd(12, 18))    # => 6

random.seed(42)            # deterministic for tests
print(random.randint(1, 6))            # => an int in [1, 6]
print(random.choice(["a", "b", "c"]))  # => one element
print(random.sample(range(10), 3))     # => 3 unique values
```

### `datetime` — dates and times

```python
from datetime import datetime, date, timedelta, timezone

now = datetime.now(timezone.utc)          # timezone-aware "now" (preferred)
print(now.isoformat())                    # => 2026-06-19T...+00:00
print(date.today())                       # => 2026-06-19
print(now + timedelta(days=7))            # one week later
parsed = datetime.strptime("2026-06-19", "%Y-%m-%d")
print(parsed.strftime("%d/%m/%Y"))        # => 19/06/2026
```

Prefer timezone-aware datetimes (`timezone.utc`) for anything stored or sent over an API. Naive datetimes cause subtle bugs.

### `json` — serialize/parse JSON

```python
import json

payload = {"id": 1, "tags": ["a", "b"], "active": True}
text = json.dumps(payload)                # dict -> JSON string
print(text)                               # => {"id": 1, "tags": ["a", "b"], "active": true}

back = json.loads(text)                   # JSON string -> dict
print(back["tags"])                       # => ['a', 'b']

print(json.dumps(payload, indent=2))      # pretty-printed
```

Mnemonic: `dumps`/`loads` end in **s** = **s**tring; `dump`/`load` (no s) work on **files**.

### `pathlib` — modern filesystem paths

```python
from pathlib import Path

p = Path("data") / "users.json"   # the / operator joins paths
print(p)                          # => data/users.json
print(p.suffix)                   # => .json
print(p.name)                     # => users.json
print(p.exists())                 # => True/False

Path("logs").mkdir(exist_ok=True)         # create dir, no error if it exists
Path("logs/app.log").write_text("hi\n")   # write a file
print(Path("logs/app.log").read_text())   # => hi
```

`pathlib` is the modern replacement for most `os.path` string-juggling — object-oriented, cross-platform, and far more readable.

### `collections` — specialized containers

```python
from collections import Counter, defaultdict, namedtuple

print(Counter("mississippi"))             # => Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})

groups = defaultdict(list)                # missing key auto-creates an empty list
groups["fruit"].append("apple")
print(dict(groups))                       # => {'fruit': ['apple']}

Point = namedtuple("Point", ["x", "y"])   # lightweight immutable record
pt = Point(1, 2)
print(pt.x, pt.y)                         # => 1 2
```

`Counter` (tally things), `defaultdict` (auto-initializing dict), and `namedtuple` (tiny immutable struct) come up in interviews constantly.

## Common mistakes & gotchas

- **Shadowing stdlib names.** Naming a file `random.py`, `json.py`, `email.py`, `queue.py`, `test.py` etc. hijacks the real import because your project dir is first on `sys.path`. Symptom: `module 'random' has no attribute 'randint'`. Rename your file.
- **Running a package file directly and hitting relative-import errors.** `python store/orders.py` fails on `from .products import x`. Use `python -m store.orders` from the project root.
- **Expecting `import` to re-run a changed file.** Once imported, a module is cached in `sys.modules`. In a long-running REPL your edits won't take effect on re-import; restart, or `importlib.reload(module)` for the rare case.
- **Heavy work at module top level.** A DB connection or HTTP call at the top of a module runs the instant *anyone* imports it. Put such work in functions or behind `if __name__ == "__main__":`.
- **`from module import *`.** Pollutes the namespace and hides where names come from. Avoid outside the REPL.
- **Circular imports.** The "partially initialized module" error. Extract shared code to a third module, or defer the import.
- **Mutable default arguments leaking across calls** — not strictly imports, but the same "runs once at definition time" trap; never write `def f(items=[]):`.
- **Forgetting that `-m` wants dots, not slashes.** It's `python -m package.module`, never `python -m package/module.py`.
- **Assuming `pip install x` always gives you `import x`.** The install name and import name often differ (e.g. install `beautifulsoup4`, import `bs4`; install `Pillow`, import `PIL`; install `python-dateutil`, import `dateutil`).

## Best practices

- Default to `import module` and qualified access (`module.thing`); reach for `from module import thing` only for a few frequently used names.
- Put a `if __name__ == "__main__":` guard on every file that has any "run it" behavior; keep top-level code side-effect-free otherwise.
- Use **absolute imports** for entry points and clarity; **relative imports** inside a package for self-containment. Be consistent within a package.
- Group imports in PEP 8 order: standard library, then third-party, then your own (local) — separated by blank lines.
- Always work inside a **virtual environment** (`python -m venv .venv`) so project dependencies stay isolated, then pin them in `requirements.txt`.
- Prefer `pathlib` over `os.path` and timezone-aware `datetime`s over naive ones.
- Keep packages small and cohesive; if two modules import each other, that's a smell — extract the shared piece.
- Run tools and your own modules with `python -m ...` to guarantee the right interpreter/environment.

## Core concept: pip and virtual environments

`pip` is Python's package installer (your `composer`). It pulls from **PyPI**, the Python Package Index. Always install into a **virtual environment** so each project's dependencies are isolated.

```bash
# 1. Create an isolated environment in the project folder
python -m venv .venv

# 2. Activate it
source .venv/bin/activate        # macOS / Linux
# .venv\Scripts\activate         # Windows (PowerShell)

# 3. Install packages (goes into THIS env's site-packages only)
python -m pip install requests

# 4. Freeze exact versions for reproducibility (like composer.lock)
python -m pip freeze > requirements.txt

# 5. On another machine: recreate the environment
python -m pip install -r requirements.txt
```

A `requirements.txt` looks like:

```
requests==2.32.3
pydantic==2.9.2
```

Then import and use the package exactly like a stdlib one:

```python
import requests

resp = requests.get("https://httpbin.org/json", timeout=10)
print(resp.status_code)       # => 200
print(resp.json()["slideshow"]["title"])
```

Tip: prefer `python -m pip ...` over a bare `pip` — it guarantees you install into the interpreter you're actually running, avoiding the classic "I installed it but Python can't find it" confusion. Modern projects increasingly declare dependencies in `pyproject.toml` and use tools like `uv` or `poetry`, but `pip` + `venv` is the universal baseline you must know.

## Core concept: structuring a small project into packages

Here's a realistic layout for a tiny backend-ish app:

```
shop/
├── pyproject.toml          # or requirements.txt — project metadata & deps
├── README.md
├── main.py                 # entry point you run
└── shop/                   # the package (same name as project is common)
    ├── __init__.py
    ├── models.py
    ├── services/
    │   ├── __init__.py
    │   ├── pricing.py
    │   └── inventory.py
    └── utils/
        ├── __init__.py
        └── formatting.py
```

```python
# shop/utils/formatting.py
def money(cents: int) -> str:
    return f"${cents / 100:.2f}"
```

```python
# shop/services/pricing.py
from shop.utils.formatting import money   # absolute import from the package root

def label_price(cents: int) -> str:
    return f"Price: {money(cents)}"
```

```python
# shop/__init__.py
from .services.pricing import label_price   # re-export for a clean public API

__all__ = ["label_price"]
```

```python
# main.py  (sits above the shop/ package, at the project root)
from shop import label_price

if __name__ == "__main__":
    print(label_price(1999))   # => Price: $19.99
```

```bash
python main.py
# => Price: $19.99
```

Why this works: you run `main.py` from the project root, so the root is on `sys.path`, so `import shop` (and its absolute sub-imports like `shop.utils.formatting`) all resolve. Inside the package, the modules use absolute imports rooted at `shop`, and `__init__.py` curates the public surface. This is the same shape you'll see in Django apps and FastAPI projects, just bigger.

## Interview Q&A

**Q: What exactly is a module in Python, and what happens when you import one?**
**A:** A module is any `.py` file; its name is the filename without the extension. Importing it executes the file top to bottom **once**, creates a module object, and caches it in `sys.modules`. Subsequent imports return the cached object without re-running the file. That's why import-time side effects fire exactly once and why you keep heavy work inside functions.

**Q: Difference between `import os` and `from os import path`?**
**A:** `import os` binds the module; you access members as `os.path`, keeping the origin explicit. `from os import path` binds `path` directly into your namespace so you use it unqualified, at the cost of possible name collisions. Use the first by default; the second sparingly for hot names.

**Q: What does `if __name__ == "__main__":` do and why use it?**
**A:** `__name__` is `"__main__"` when the file is run directly and the module's own name when it's imported. The guard runs the indented block only on direct execution, so a file can act as both an importable library and a runnable script without its script logic firing on import.

**Q: How does Python decide where to find a module?**
**A:** It searches `sys.path` in order and uses the first match. `sys.path` starts with the script's directory (or CWD), then `PYTHONPATH`, then the standard library, then `site-packages`. Because the script directory is first, naming a file after a stdlib module shadows the real one.

**Q: What's the role of `__init__.py`?**
**A:** It marks a directory as a package and runs once when the package is imported. It's the package's public face — commonly used to re-export selected names (so callers do `from pkg import thing`) and to define `__all__`. Since 3.3, namespace packages can omit it, but explicit `__init__.py` is preferred for application code.

**Q: Absolute vs relative imports — when do you use each?**
**A:** Absolute (`from shop.utils import money`) spells the full path from the project root and is clear and robust for entry points. Relative (`from .utils import money`) is resolved against the current package and keeps a package self-contained when renamed/moved. Use relative inside a package, absolute at the top level — and consistently. Relative imports fail if you run the file directly instead of via `python -m`.

**Q: Why does `python store/orders.py` fail on a relative import but `python -m store.orders` works?**
**A:** Running a file directly makes it the top-level `__main__` module with no package context, so `.` resolves to nothing. `-m` imports it through the package machinery with proper package context, so relative imports resolve. `-m` takes a dotted module path, not a file path.

**Q: What is a circular import and how do you fix it?**
**A:** Two modules import each other, so during import one reaches for a name the other hasn't defined yet — the error mentions a "partially initialized module." Fixes: extract the shared code into a third module (best), import the module rather than a specific name and use it later, defer the import inside a function, or use `TYPE_CHECKING` for type-only imports.

**Q: Does the install name always equal the import name?**
**A:** No. You `pip install beautifulsoup4` but `import bs4`; install `Pillow` but `import PIL`; install `python-dateutil` but `import dateutil`. Check a package's docs for its import name.

**Q: Why use a virtual environment, and how do you create one?**
**A:** A venv isolates a project's dependencies (and versions) from the system Python and other projects, preventing conflicts. Create with `python -m venv .venv`, activate it, install into it with `pip`, and pin with `pip freeze > requirements.txt` for reproducibility — conceptually like Composer's per-project `vendor/` plus `composer.lock`.

**Q: What's wrong with `from module import *`?**
**A:** It imports an unknown set of names, can silently overwrite existing names, and makes it impossible to tell where a name originated, hurting readability and tooling. Acceptable only in an interactive REPL. In code, import explicitly.

**Q: After editing a module, why doesn't a re-`import` in the same REPL session pick up changes?**
**A:** Because the module is cached in `sys.modules`; `import` returns the cached object instead of re-running the file. Restart the interpreter, or for the rare interactive case use `importlib.reload(module)`.

## Practice exercises

1. **Two-file basics.** Create `mathutils.py` with a function `area_of_circle(r)` (use `math.pi`) and a module-level `print("loaded mathutils")`. Create `run.py` that imports it and prints the area for `r=2`. Confirm the "loaded" line prints exactly once even if you `import mathutils` twice.

2. **The `__main__` guard.** Add an `if __name__ == "__main__":` block to `mathutils.py` that prints a quick self-test. Show that running `python mathutils.py` prints the self-test, but importing it from `run.py` does not.

3. **Build a package.** Create a package `geometry/` with `__init__.py`, `circle.py` (function `area(r)`), and `rectangle.py` (function `area(w, h)`). Re-export both from `__init__.py` (e.g. as `circle_area` and `rect_area`) and call them from a top-level `main.py`.

4. **Trigger and fix a circular import.** Make `a.py` and `b.py` import a name from each other so it fails with "partially initialized module." Then fix it two different ways: (a) extract the shared function into `common.py`, and (b) defer one import inside a function.

5. **Stdlib mini-tool.** Write `report.py` that builds a dict with the current UTC timestamp (`datetime`), a random 4-digit code (`random`), and the current working directory (`pathlib`/`os`), then writes it as pretty JSON to `report.json`. Read it back and print one field.

6. **(Stretch) pip + import-name gap.** In a fresh virtualenv, `pip install requests`, write a script that GETs `https://httpbin.org/uuid` and prints the returned uuid, then `pip freeze > requirements.txt`. Note any package whose install name differs from its import name.

<details><summary>Solutions</summary>

**1 & 2 — `mathutils.py` and `run.py`:**

```python
# mathutils.py
import math

print("loaded mathutils")   # import-time side effect

def area_of_circle(r):
    return math.pi * r ** 2

if __name__ == "__main__":           # exercise 2
    print("self-test:", area_of_circle(1))   # => self-test: 3.14159...
```

```python
# run.py
import mathutils
import mathutils   # cached — "loaded mathutils" prints only once total

print(mathutils.area_of_circle(2))   # => 12.566370614359172
```

```bash
python run.py
# => loaded mathutils
# => 12.566370614359172

python mathutils.py
# => loaded mathutils
# => self-test: 3.141592653589793
```

**3 — the `geometry` package:**

```python
# geometry/circle.py
import math
def area(r):
    return math.pi * r ** 2
```

```python
# geometry/rectangle.py
def area(w, h):
    return w * h
```

```python
# geometry/__init__.py
from .circle import area as circle_area
from .rectangle import area as rect_area
__all__ = ["circle_area", "rect_area"]
```

```python
# main.py
from geometry import circle_area, rect_area
print(circle_area(2))    # => 12.566370614359172
print(rect_area(3, 4))   # => 12
```

**4 — circular import, broken then fixed:**

```python
# BROKEN
# a.py
from b import b_val
A_VAL = 1
# b.py
from a import A_VAL   # a.py not finished yet -> partially initialized error
b_val = 2
```

```python
# FIX (a): shared module
# common.py
A_VAL = 1
B_VAL = 2
# a.py
from common import A_VAL, B_VAL
# b.py
from common import A_VAL, B_VAL   # no cycle
```

```python
# FIX (b): deferred import
# b.py
def use_a():
    from a import A_VAL   # runs at call time, after both modules load
    return A_VAL + 1
```

**5 — stdlib mini-tool:**

```python
# report.py
import json
import random
from datetime import datetime, timezone
from pathlib import Path

data = {
    "generated_at": datetime.now(timezone.utc).isoformat(),
    "code": random.randint(1000, 9999),
    "cwd": str(Path.cwd()),
}

out = Path("report.json")
out.write_text(json.dumps(data, indent=2))

loaded = json.loads(out.read_text())
print("code:", loaded["code"])   # => code: <some 4-digit number>
```

**6 — pip + import-name gap:**

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install requests
```

```python
# fetch_uuid.py
import requests
resp = requests.get("https://httpbin.org/uuid", timeout=10)
print(resp.json()["uuid"])   # => e.g. 5f...-...-...
```

```bash
python fetch_uuid.py
python -m pip freeze > requirements.txt
```

Install-vs-import gaps to remember: `beautifulsoup4` -> `bs4`, `Pillow` -> `PIL`, `python-dateutil` -> `dateutil`, `PyYAML` -> `yaml`.

</details>

## Cheat sheet

```python
# --- import forms ---
import math                 # math.sqrt(x)
from math import sqrt, pi   # sqrt(x)
import numpy as np          # alias
from math import *          # AVOID (REPL only)

# --- the __main__ guard ---
if __name__ == "__main__":
    main()                  # runs only on `python file.py`, not on import

# --- relative imports (inside a package) ---
from .sibling import thing     # current package
from ..parent_pkg import thing # parent package

# --- finding modules ---
import sys; print(sys.path)    # search order; [0] = script dir / CWD
import json; print(json.__file__)   # where a module actually lives
```

```bash
# run a script vs a module
python path/to/file.py          # file = __main__, no package context
python -m package.module        # dotted path; relative imports work

# common -m tools
python -m venv .venv            # create virtual env
python -m pip install requests  # install into THIS interpreter
python -m pytest                # run tests
python -m http.server 8000      # quick static server
```

| Concept | Key fact |
|---------|----------|
| Module | A `.py` file; runs once on first import, then cached in `sys.modules` |
| Package | A directory; `__init__.py` marks it and curates its public API |
| `__name__` | `"__main__"` when run directly, the module name when imported |
| Absolute import | `from pkg.sub import x` — from `sys.path` root |
| Relative import | `from .sub import x` — needs package context (`python -m`) |
| Circular import | "partially initialized module" → extract shared code or defer import |
| pip + venv | `python -m venv .venv`, activate, `pip install`, `pip freeze > requirements.txt` |
| Install ≠ import | `beautifulsoup4`→`bs4`, `Pillow`→`PIL`, `PyYAML`→`yaml` |

| Stdlib module | Use it for |
|---------------|-----------|
| `os` / `sys` | env vars, paths, CWD; argv, exit, version |
| `math` / `random` | numeric helpers; randomness (seed for tests) |
| `datetime` | dates/times; prefer timezone-aware UTC |
| `json` | `dumps`/`loads` (strings), `dump`/`load` (files) |
| `pathlib` | object-oriented paths; `Path("a") / "b"` |
| `collections` | `Counter`, `defaultdict`, `namedtuple` |
