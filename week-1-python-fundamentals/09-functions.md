# 09 — Functions: args, kwargs, scope, lambda

> Week 1 · Python Fundamentals

## What you will learn

- How to define functions with `def`, what `return` does, and why a function with no `return` still hands you back `None`.
- The difference between **positional** and **keyword** arguments, and why Python lets you mix them.
- Default parameter values — and the single nastiest beginner trap in the whole language: the **mutable default argument**.
- `*args` and `**kwargs`: how to accept any number of positional and keyword arguments.
- **Keyword-only** parameters (after `*`) and **positional-only** parameters (before `/`).
- **Argument unpacking** at the call site with `*` and `**`.
- Variable **scope** and the **LEGB rule**, plus `global` and `nonlocal`.
- Functions as **first-class objects**: assign them, pass them, return them.
- **Lambda** expressions — what they are, and where they actually earn their place.
- **Docstrings**, `help()`, and a preview of **type-annotated** signatures.
- **Recursion** and Python's recursion limit.
- **Pure functions** vs side effects, and why the distinction matters for testable backends.

## PHP/Laravel bridge

You already write functions every day in PHP — `function foo($a, $b = 1) { return $a + $b; }`. The shapes are close, but Python adds power PHP only partially has: PHP 8 gave you named arguments (`foo(b: 2)`), but Python has had them since forever and uses them far more idiomatically. Python's `*args`/`**kwargs` are the equivalent of PHP's `...$args` variadic and `func_get_args()`, except `**kwargs` (a dict of named arguments) has no clean PHP analog. Python closures capture the *variable itself* automatically (resolved when the inner function runs, i.e. late binding) — you do **not** write PHP's `use ($x)` clause. And where PHP has `fn($x) => $x * 2` arrow functions, Python has `lambda x: x * 2`. The biggest mental shift: in Python a function is just an object you can pass around as easily as an integer.

## Core concept: `def`, `return`, and implicit `None`

A function is defined with `def`, a name, a parameter list in parentheses, a colon, and an indented body. There are no curly braces — **indentation is the block**.

```python
def greet(name):
    return f"Hello, {name}!"

message = greet("Ada")
print(message)  # => Hello, Ada!
```

`return` hands a value back to the caller and ends the function immediately. If you reach the end of a function without `return`, or write a bare `return` with no value, the function returns `None` — Python's null. This is the equivalent of a PHP function that has no `return` statement (which returns `null`).

```python
def log(msg):
    print(f"[LOG] {msg}")
    # no return statement

result = log("started")  # => [LOG] started
print(result)            # => None
print(type(result))      # => <class 'NoneType'>
```

A common beginner bug: forgetting `return` and wondering why your caller gets `None`.

```python
def add(a, b):
    a + b          # computed, then thrown away — no return!

print(add(2, 3))   # => None   (NOT 5)
```

A function can return multiple values by returning a tuple. Python unpacks it for you at the call site:

```python
def divmod_(a, b):
    return a // b, a % b   # this is a tuple: (quotient, remainder)

q, r = divmod_(17, 5)
print(q, r)  # => 3 2
```

## Core concept: positional vs keyword arguments

When you **call** a function, you can pass arguments by position or by name (keyword).

```python
def make_user(name, role, active):
    return {"name": name, "role": role, "active": active}

# Positional: order matters
make_user("Ada", "admin", True)

# Keyword: order does NOT matter, intent is explicit
make_user(name="Ada", active=True, role="admin")

# Mixed: positional args must come FIRST, then keyword args
make_user("Ada", role="admin", active=True)
```

The rule: **once you use a keyword argument, every argument after it must also be a keyword argument.** This is a syntax error:

```python
# make_user("Ada", role="admin", True)  # SyntaxError: positional argument follows keyword argument
```

Keyword arguments are the Pythonic way to make calls self-documenting. `create_widget(300, 200, True, False)` is a mystery; `create_widget(width=300, height=200, visible=True, resizable=False)` reads itself. (PHP 8 named args do the same job, but in Python this is everyday style, not a special case.)

## Core concept: default parameter values

A parameter can have a default, used when the caller omits it. Parameters **with** defaults must come **after** parameters without them.

```python
def connect(host, port=5432, timeout=30):
    return f"{host}:{port} (timeout={timeout}s)"

print(connect("db.local"))                  # => db.local:5432 (timeout=30s)
print(connect("db.local", 5433))            # => db.local:5433 (timeout=30s)
print(connect("db.local", timeout=5))       # => db.local:5432 (timeout=5s)
```

### The mutable default argument trap

This is the most infamous gotcha in Python, and it shows up in interviews constantly. **Default values are evaluated once, when the function is defined — not on each call.** If the default is a mutable object (a `list`, `dict`, or `set`), every call that omits that argument shares the *same* object.

```python
def append_item(item, bucket=[]):   # DANGER: shared list!
    bucket.append(item)
    return bucket

print(append_item("a"))  # => ['a']
print(append_item("b"))  # => ['a', 'b']   (!!! the 'a' leaked in)
print(append_item("c"))  # => ['a', 'b', 'c']
```

The list was created once at definition time and is reused. The fix is the standard idiom: default to `None`, then create a fresh object inside the function.

```python
def append_item(item, bucket=None):
    if bucket is None:
        bucket = []           # fresh list every call
    bucket.append(item)
    return bucket

print(append_item("a"))  # => ['a']
print(append_item("b"))  # => ['b']   (correct, independent)
```

Immutable defaults (`int`, `str`, `tuple`, `None`, `bool`) are perfectly safe because they can't be mutated in place. The trap only bites with mutable defaults.

## Core concept: `*args` and `**kwargs`

Sometimes you don't know how many arguments you'll receive. `*args` collects extra **positional** arguments into a **tuple**; `**kwargs` collects extra **keyword** arguments into a **dict**.

```python
def report(*args, **kwargs):
    print("positional:", args)    # a tuple
    print("keyword:", kwargs)     # a dict

report(1, 2, 3, name="Ada", role="admin")
# => positional: (1, 2, 3)
# => keyword: {'name': 'Ada', 'role': 'admin'}
```

The names `args` and `kwargs` are convention only; the `*` and `**` are what matter. A realistic example — summing an arbitrary number of values:

```python
def total(*numbers):
    result = 0
    for n in numbers:
        result += n
    return result

print(total(1, 2, 3, 4))  # => 10
print(total())            # => 0
```

You can combine ordinary parameters with `*args` and `**kwargs`. The canonical full ordering is:

```python
def f(positional, *args, keyword_only, **kwargs):
    ...
```

This pattern is everywhere in framework code (decorators, wrappers, middleware) because it lets a wrapper accept *anything* and forward it on.

```python
def logged(*args, **kwargs):
    print(f"called with {args} and {kwargs}")
    return sum(args)

logged(1, 2, 3, label="demo")
# => called with (1, 2, 3) and {'label': 'demo'}
```

## Core concept: keyword-only and positional-only parameters

### Keyword-only (after `*`)

Anything that comes **after** a bare `*` in the parameter list must be passed **by keyword**. This forces callers to be explicit, which prevents bugs from positional mix-ups.

```python
def create_file(path, *, overwrite=False, encoding="utf-8"):
    return f"{path} overwrite={overwrite} encoding={encoding}"

print(create_file("a.txt", overwrite=True))   # => a.txt overwrite=True encoding=utf-8
# create_file("a.txt", True)                   # TypeError: takes 1 positional argument but 2 were given
```

This is widely used in libraries: it means you can reorder or add keyword-only options later without breaking callers who passed them positionally (they couldn't).

### Positional-only (before `/`)

Anything **before** a `/` must be passed **by position** and can never be referred to by name. This is rarer in app code but appears in the standard library and lets you rename a parameter freely without breaking callers.

```python
def power(base, exponent, /):
    return base ** exponent

print(power(2, 10))            # => 1024
# power(base=2, exponent=10)   # TypeError: positional-only arguments passed as keyword
```

You can combine all three regions. Read the signature left to right: positional-only, then normal, then keyword-only.

```python
def func(pos_only, /, normal, *, kw_only):
    return (pos_only, normal, kw_only)

print(func(1, 2, kw_only=3))           # => (1, 2, 3)
print(func(1, normal=2, kw_only=3))    # => (1, 2, 3)
```

## Core concept: argument unpacking at the call site

The `*` and `**` operators also work when **calling** a function. They unpack an iterable into positional arguments, or a dict into keyword arguments. This is the inverse of `*args`/`**kwargs`.

```python
def point(x, y, z):
    return f"({x}, {y}, {z})"

coords = [1, 2, 3]
print(point(*coords))          # => (1, 2, 3)   — unpacks the list

settings = {"x": 10, "y": 20, "z": 30}
print(point(**settings))       # => (10, 20, 30) — unpacks the dict by key

# You can mix literal args with unpacking:
print(point(1, *[2, 3]))       # => (1, 2, 3)
```

This is how forwarding works in practice: a wrapper captures `*args, **kwargs` and re-applies them with `*` and `**`.

```python
def trace(fn, *args, **kwargs):
    print(f"calling {fn.__name__}")
    return fn(*args, **kwargs)   # forward everything along

trace(point, 1, 2, z=3)
# => calling point
# => '(1, 2, 3)'
```

## Core concept: scope and the LEGB rule

When Python encounters a name, it searches scopes in a fixed order — **LEGB**:

| Letter | Scope | What it means |
|--------|-------|---------------|
| **L** | Local | Names assigned inside the current function |
| **E** | Enclosing | Names in any enclosing (outer) function |
| **G** | Global | Names at the top level of the module |
| **B** | Built-in | Python's built-ins (`len`, `print`, `range`, ...) |

```python
x = "global"

def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)   # finds Local first
    inner()
    print(x)       # finds Enclosing-as-Local here

outer()       # => local  then  enclosing
print(x)      # => global
```

A crucial rule: **assigning** to a name inside a function makes it local *for the whole function*, even before that line. Reading is fine, but if you assign anywhere in the function, Python treats the name as local everywhere in it.

```python
counter = 0

def bump():
    print(counter)   # UnboundLocalError! Python sees the assignment below
    counter = counter + 1   # ...so 'counter' is local, and read before assignment

# bump()  # UnboundLocalError: local variable 'counter' referenced before assignment
```

### `global` and `nonlocal`

To **rebind** a global from inside a function, declare it `global`. To rebind a variable from an enclosing function, declare it `nonlocal`.

```python
count = 0

def increment():
    global count
    count += 1        # now legal; modifies the module-level 'count'

increment()
increment()
print(count)  # => 2
```

```python
def make_counter():
    n = 0
    def step():
        nonlocal n     # refers to 'n' in make_counter, not a new local
        n += 1
        return n
    return step

c = make_counter()
print(c(), c(), c())  # => 1 2 3
```

In practice, prefer returning values over reaching for `global` — heavy use of `global` makes code hard to test and reason about (more on that under pure functions). `nonlocal` is the legitimate way to build closures that carry state.

## Core concept: functions are first-class objects

In Python a function is an ordinary object. You can assign it to a variable, store it in a list or dict, pass it as an argument, and return it from another function. This is the foundation for decorators, callbacks, and dependency injection.

```python
def shout(text):
    return text.upper()

# 1. Assign to a variable (no parentheses — you're naming the function, not calling it)
yell = shout
print(yell("hi"))          # => HI

# 2. Store in a data structure (dispatch table — replaces big if/elif chains)
handlers = {"upper": str.upper, "lower": str.lower}
print(handlers["upper"]("Hello"))  # => HELLO

# 3. Pass as an argument
def apply(fn, value):
    return fn(value)
print(apply(shout, "hi"))  # => HI

# 4. Return from a function (a closure)
def multiplier(factor):
    def multiply(n):
        return n * factor   # 'factor' is captured from the enclosing scope
    return multiply

triple = multiplier(3)
print(triple(10))          # => 30
```

Notice closures capture variables automatically — there is no PHP-style `use ($factor)`. The inner function simply reads `factor` from its enclosing scope via the LEGB rule.

## Core concept: lambda expressions

A `lambda` is a small anonymous function written as a single expression. `lambda params: expression` returns the value of `expression` — there is no `return` keyword and no statements, only one expression.

```python
square = lambda x: x * x
print(square(5))   # => 25

# Equivalent to:
def square(x):
    return x * x
```

Lambdas shine as throwaway functions passed to other functions — especially the `key=` argument of `sorted`, `min`, `max`, and `filter`/`map`.

```python
users = [
    {"name": "Ada", "age": 36},
    {"name": "Linus", "age": 25},
    {"name": "Grace", "age": 85},
]

# Sort by age using a lambda as the sort key
by_age = sorted(users, key=lambda u: u["age"])
print([u["name"] for u in by_age])  # => ['Linus', 'Ada', 'Grace']

oldest = max(users, key=lambda u: u["age"])
print(oldest["name"])               # => Grace
```

**When NOT to use a lambda:** if it needs a name, a docstring, multiple statements, or you'd reuse it — write a `def`. A named function is more readable and debuggable (its name shows up in tracebacks; a lambda just shows `<lambda>`). Assigning a lambda to a variable (`square = lambda x: ...`) is considered un-Pythonic — use `def` for that. Lambdas earn their keep only as inline, single-use callables.

## Core concept: docstrings, `help()`, and type-annotation preview

A **docstring** is a string literal as the first statement in a function. It becomes the function's `__doc__` and powers `help()`, IDE tooltips, and tools like Sphinx. This is your equivalent of a PHPDoc block, but it lives inside the function and is available at runtime.

```python
def normalize(text):
    """Strip surrounding whitespace and lowercase the text.

    Args:
        text: The raw input string.

    Returns:
        The cleaned string.
    """
    return text.strip().lower()

print(normalize.__doc__.splitlines()[0])  # => Strip surrounding whitespace and lowercase the text.
help(normalize)  # prints the signature + docstring in the REPL
```

**Type annotations** (covered in depth in a later lesson) let you declare expected types. They are *not* enforced at runtime by Python itself, but tools like `mypy`, `pyright`, and editors use them, and frameworks like FastAPI and Pydantic build real behavior on top of them.

```python
def add(a: int, b: int = 0) -> int:
    return a + b

print(add(2, 3))          # => 5
print(add.__annotations__)  # => {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
```

The `-> int` after the parameters is the **return type**. Get comfortable reading these now — every Django REST Framework serializer and FastAPI endpoint you write later leans on them.

## Core concept: recursion and the recursion limit

A function may call itself. Each call needs a **base case** to stop, or it recurses forever.

```python
def factorial(n):
    if n <= 1:           # base case
        return 1
    return n * factorial(n - 1)   # recursive case

print(factorial(5))  # => 120
```

Python does **not** optimize tail recursion, and each call consumes a stack frame. To prevent a crash, Python caps recursion depth (default ~1000). Exceed it and you get `RecursionError`.

```python
import sys
print(sys.getrecursionlimit())  # => 1000 (typically)

def runaway(n):
    return runaway(n + 1)   # no base case

# runaway(0)  # RecursionError: maximum recursion depth exceeded
```

You *can* raise the limit with `sys.setrecursionlimit(...)`, but for deep iteration prefer a loop — it's faster and won't blow the stack. Recursion is best reserved for naturally recursive structures (trees, nested JSON, directory walks).

## Core concept: pure functions and side effects

A **pure function** depends only on its arguments and returns a value without changing anything outside itself (no mutating globals, no printing, no DB writes, no I/O). The same inputs always give the same output. Pure functions are trivial to test and reason about — exactly what you want in the business-logic core of a backend.

```python
# Pure: output depends only on inputs, nothing external changes
def with_tax(price, rate):
    return round(price * (1 + rate), 2)

print(with_tax(100, 0.2))  # => 120.0  (always, every time)
```

```python
# Impure: mutates external state (a side effect)
cart = []
def add_to_cart(item):
    cart.append(item)   # changes a global — side effect
    return cart
```

Side effects (writing to a database, calling an API, logging) are necessary in real systems — you can't avoid them entirely. The good design pattern is to **push side effects to the edges** and keep your core logic pure. In a Django/FastAPI service this means: compute results with pure functions, then let a thin outer layer do the I/O. It makes unit tests fast and deterministic.

## Common mistakes & gotchas

- **Forgetting `return`** — the function computes a value, throws it away, and the caller silently gets `None`.
- **Mutable default arguments** — `def f(x, items=[])` shares one list across all calls. Always default to `None` and create the object inside.
- **Positional argument after keyword argument** — `f(a=1, 2)` is a `SyntaxError`. Keyword args must come last.
- **`UnboundLocalError`** — assigning to a name anywhere in a function makes it local *throughout*; reading it before assignment fails. Use `global`/`nonlocal` if you truly mean to rebind an outer name.
- **Calling vs referencing** — `my_func` is the function object; `my_func()` calls it. Passing `my_func()` where you meant `my_func` is a frequent callback bug.
- **Overusing `lambda`** — assigning a lambda to a name, or cramming logic into one, hurts readability and produces `<lambda>` in tracebacks. Use `def`.
- **Overusing `global`** — it couples functions to module state and breaks testability. Prefer parameters and return values.
- **Deep recursion** — `RecursionError` on large inputs; convert to a loop when the depth isn't naturally bounded.
- **Confusing `*args`/`**kwargs` at definition vs at the call site** — at `def` they *collect*; at the call they *unpack*.

## Best practices

- Give functions verb-phrase names (`calculate_total`, `fetch_user`) and keep them small and single-purpose.
- Prefer keyword arguments at call sites for booleans and options — `save(commit=False)` beats `save(False)`.
- Use keyword-only parameters (`*`) for optional flags so callers must name them.
- Default mutable parameters to `None`, never to `[]`/`{}`/`set()`.
- Write a docstring for every public function; describe what it does, not how.
- Add type annotations — they document intent and unlock static checkers and frameworks.
- Keep the business-logic core pure; isolate I/O and side effects at the boundaries.
- Reach for `def` over `lambda` unless the function is a genuinely throwaway, single-expression callable.
- Avoid `global`; return values and let the caller decide what to store.

## Interview Q&A

**Q: What does a Python function return if it has no `return` statement?**
**A:** `None`. Reaching the end of the function body, or a bare `return`, both yield `None` (Python's null). This mirrors a PHP function with no return giving `null`.

**Q: Explain the mutable default argument problem and how to fix it.**
**A:** Default values are evaluated once, at function-definition time, not per call. A mutable default like `[]` is therefore shared across every call that omits it, so mutations accumulate between calls. The fix is to default the parameter to `None` and create a fresh object inside the function: `def f(x, items=None): items = [] if items is None else items`.

**Q: What's the difference between `*args` and `**kwargs`?**
**A:** `*args` collects extra *positional* arguments into a tuple; `**kwargs` collects extra *keyword* arguments into a dict. At a call site the same operators *unpack*: `*list` spreads into positional args, `**dict` spreads into keyword args.

**Q: What does the LEGB rule describe?**
**A:** The order Python resolves a name: Local, then Enclosing (outer functions), then Global (module level), then Built-in. The first match wins.

**Q: When do you need `global` vs `nonlocal`?**
**A:** Use `global` to rebind a module-level (global) variable from inside a function. Use `nonlocal` to rebind a variable that lives in an enclosing function (used in closures). Without them, an assignment creates a new local variable instead.

**Q: Why might you get `UnboundLocalError` reading a variable that exists at module level?**
**A:** Because you also *assign* to that name somewhere in the function. Any assignment makes the name local for the entire function, so reading it before the assignment fails. Declare it `global`/`nonlocal` if you meant the outer one.

**Q: What are keyword-only parameters and why use them?**
**A:** Parameters listed after a bare `*` in the signature must be passed by name. They make calls self-documenting (`open(path, *, encoding="utf-8")`) and let you add or reorder options later without breaking callers, since no one could pass them positionally.

**Q: What does it mean that functions are "first-class objects" in Python?**
**A:** Functions are values: you can assign them to variables, store them in lists/dicts, pass them as arguments, and return them. This enables callbacks, dispatch tables, decorators, and closures.

**Q: When should you use a `lambda` instead of `def`?**
**A:** Only for a short, single-expression, throwaway callable passed inline — most commonly as the `key=` argument to `sorted`/`min`/`max` or to `map`/`filter`. If it needs a name, a docstring, multiple statements, or reuse, use `def`.

**Q: Does Python optimize recursion? What's the limit?**
**A:** No — there is no tail-call optimization, and each call uses a stack frame. The default recursion limit is around 1000 (`sys.getrecursionlimit()`); exceeding it raises `RecursionError`. For deep iteration use a loop instead.

**Q: What is a pure function and why does it matter for backend code?**
**A:** A pure function depends only on its inputs and produces no side effects (no I/O, no mutating external state). The same inputs always give the same output, which makes it deterministic and trivial to unit-test. Best practice is to keep core logic pure and push side effects (DB, network, logging) to the edges.

**Q: How are arguments passed in Python — by value or by reference?**
**A:** By "assignment" / object reference: the function receives a reference to the same object. Mutating a mutable argument (e.g. `list.append`) is visible to the caller; rebinding the parameter to a new object is not. There's no implicit copy.

## Practice exercises

1. Write `summarize(*numbers)` that returns a dict with keys `count`, `total`, and `average`. Calling it with no arguments must return `average` of `0` (not raise `ZeroDivisionError`).
2. Fix this buggy function so each call starts with an empty log: `def record(event, log=[]): log.append(event); return log`.
3. Write `make_multiplier(factor)` that returns a function multiplying its argument by `factor`. Demonstrate `double = make_multiplier(2); double(21) == 42`.
4. Write `safe_div(a, b, *, default=None)` with a keyword-only `default` returned when `b == 0`. Show that `safe_div(10, 0, default=0)` returns `0`.
5. Given `people = [{"name": "Sam", "age": 40}, {"name": "Ari", "age": 22}]`, sort by `name` and separately find the youngest person, each using a `lambda`.
6. Write a recursive `flatten(nested)` that turns `[1, [2, [3, 4]], 5]` into `[1, 2, 3, 4, 5]`.

<details><summary>Solutions</summary>

```python
# 1.
def summarize(*numbers):
    count = len(numbers)
    total = sum(numbers)
    average = total / count if count else 0
    return {"count": count, "total": total, "average": average}

print(summarize(2, 4, 6))  # => {'count': 3, 'total': 12, 'average': 4.0}
print(summarize())         # => {'count': 0, 'total': 0, 'average': 0}


# 2.
def record(event, log=None):
    if log is None:
        log = []
    log.append(event)
    return log

print(record("login"))   # => ['login']
print(record("logout"))  # => ['logout']  (independent)


# 3.
def make_multiplier(factor):
    def multiply(n):
        return n * factor
    return multiply

double = make_multiplier(2)
print(double(21))  # => 42


# 4.
def safe_div(a, b, *, default=None):
    if b == 0:
        return default
    return a / b

print(safe_div(10, 2))             # => 5.0
print(safe_div(10, 0, default=0))  # => 0


# 5.
people = [{"name": "Sam", "age": 40}, {"name": "Ari", "age": 22}]
by_name = sorted(people, key=lambda p: p["name"])
print([p["name"] for p in by_name])           # => ['Ari', 'Sam']
youngest = min(people, key=lambda p: p["age"])
print(youngest["name"])                        # => Ari


# 6.
def flatten(nested):
    result = []
    for item in nested:
        if isinstance(item, list):
            result.extend(flatten(item))   # recurse into sublists
        else:
            result.append(item)
    return result

print(flatten([1, [2, [3, 4]], 5]))  # => [1, 2, 3, 4, 5]
```

</details>

## Cheat sheet

```python
def f(a, b=1):          # default value (use None for mutable defaults!)
    return a + b        # no return -> returns None

f(1, 2)                 # positional
f(a=1, b=2)             # keyword
def g(*args, **kwargs)  # collect: args=tuple, kwargs=dict
g(*[1, 2], **{"k": 3})  # unpack at call site

def h(pos, /, normal, *, kw):   # pos-only | normal | kw-only
    ...

# Scope: LEGB = Local -> Enclosing -> Global -> Built-in
global x      # rebind module-level name
nonlocal x    # rebind enclosing-function name

# First-class: assign / store / pass / return functions
sorted(data, key=lambda d: d["age"])   # lambda = single-expression fn

def doc():
    """One-line summary."""   # docstring -> help() / __doc__

def typed(a: int, b: str = "") -> bool:  # annotations (not enforced at runtime)
    ...

import sys; sys.getrecursionlimit()      # ~1000; no tail-call optimization
```

| Concept | Syntax | Notes |
|---------|--------|-------|
| Default arg | `def f(x=1)` | evaluated once at def time |
| Mutable default fix | `def f(x=None): x = x or []` | avoid shared `[]`/`{}` |
| Collect positional | `def f(*args)` | tuple |
| Collect keyword | `def f(**kwargs)` | dict |
| Keyword-only | `def f(*, k)` | must pass by name |
| Positional-only | `def f(p, /)` | must pass by position |
| Unpack args | `f(*lst, **dct)` | spread at call site |
| Rebind global | `global x` | inside function |
| Rebind enclosing | `nonlocal x` | inside closure |
| Anonymous fn | `lambda x: x*2` | single expression only |
