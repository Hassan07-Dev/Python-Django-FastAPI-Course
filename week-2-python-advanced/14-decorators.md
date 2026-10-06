# 14 — Decorators

> Week 2 · Python Advanced

Decorators are one of those Python features that look like magic until the moment they click — and then you see them everywhere: in Django views, Flask routes, FastAPI endpoints, DRF permissions, pytest fixtures, caching layers, and retry logic. This lesson builds decorators from the ground up so the `@something` syntax stops being mysterious and becomes a tool you reach for deliberately.

## What you will learn

- Why functions in Python are *objects* you can pass around, return, and store
- Nested functions and **closures** — the foundation every decorator stands on
- The infamous **late-binding closure-in-a-loop** gotcha and how to avoid it
- What a decorator actually *is*, and how `@decorator` desugars to plain function calls
- Writing a simple decorator that wraps another function
- Preserving the wrapped function's identity with `functools.wraps`
- **Decorators that take arguments** (the three-level "decorator factory" pattern)
- **Stacking** multiple decorators and why order matters
- **Class-based decorators** using `__call__`
- Decorating methods (and the `self` subtlety)
- Real-world decorators: timing, logging, caching (`functools.lru_cache`), retry, auth checks
- How Django / Flask / FastAPI lean on decorators for routing and auth

## PHP/Laravel bridge

You have already used "decorator-like" behavior in Laravel, even if it wasn't called that. **Middleware** wraps a request: it runs code before and after your controller, exactly like a decorator wraps a function. PHP 8 **Attributes** (`#[Route('/users')]`, `#[Middleware('auth')]`) look almost identical to Python's `@` syntax — but there's a crucial difference: PHP attributes are *passive metadata* that a framework must read via reflection, whereas a Python decorator is *active code that runs immediately* and replaces the function. If you've ever wrapped a method call in a Laravel **service decorator** or used the `Cache::remember()` pattern, you've done by hand what Python decorators let you do declaratively. Think: middleware + attributes + higher-order functions, all in one tiny `@` symbol.

## Core concept — Functions are objects

Before decorators make any sense, internalize this: in Python a function is a first-class object, just like an `int` or a `list`. You can assign it to a variable, store it in a dict, pass it as an argument, and return it from another function.

```python
def shout(text):
    return text.upper() + "!"

# Assign the function itself (no parentheses!) to a new name.
yell = shout
print(yell("hello"))          # => HELLO!

# It really is an object with attributes.
print(shout.__name__)         # => shout
print(type(shout))            # => <class 'function'>

# Store functions in a data structure.
handlers = {"loud": shout, "quiet": str.lower}
print(handlers["loud"]("hey"))   # => HEY!
```

The single most important distinction for beginners:

| Expression | Meaning |
|------------|---------|
| `shout`   | the function object itself (you can pass it around) |
| `shout()` | *call* the function and get its return value |

A function that takes or returns another function is called a **higher-order function**. Decorators are just a special, sugar-coated case of that.

```python
def apply_twice(func, value):
    """Higher-order: receives a function as an argument."""
    return func(func(value))

print(apply_twice(shout, "hi"))   # => HI!! ... let's verify:
# shout("hi") => "HI!", then shout("HI!") => "HI!!"
```

## Core concept — Nested functions and closures

You can define a function *inside* another function. The inner function can see the outer function's variables, and — this is the key — it **remembers** them even after the outer function has finished running. That captured environment is called a **closure**.

```python
def make_multiplier(factor):
    def multiply(number):
        # `factor` is captured from the enclosing scope.
        return number * factor
    return multiply            # return the inner function (not called!)

double = make_multiplier(2)
triple = make_multiplier(3)

print(double(10))   # => 20
print(triple(10))   # => 30

# The closure kept `factor` alive. Proof:
print(double.__closure__[0].cell_contents)   # => 2
```

`make_multiplier` has already returned by the time you call `double(10)`, yet `factor` is still `2`. The inner function "closed over" that variable — hence *closure*. This capture-and-remember mechanism is exactly how decorators inject extra behavior around a target function.

### Writing to enclosing variables: `nonlocal`

Reading an enclosing variable just works. To *reassign* one, you need the `nonlocal` keyword (the analogue of `global`, but for the enclosing function scope rather than module scope).

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count          # without this, you'd get UnboundLocalError
        count += 1
        return count
    return increment

c = make_counter()
print(c(), c(), c())   # => 1 2 3
```

### The late-binding closure-in-a-loop gotcha

This trips up nearly everyone. Closures capture *variables, not values* — and they look the variable up **at call time**, not at definition time.

```python
# BROKEN: every function prints 2, not 0/1/2
funcs = []
for i in range(3):
    funcs.append(lambda: i)

print([f() for f in funcs])   # => [2, 2, 2]   (surprise!)
```

All three lambdas captured the *same* variable `i`. By the time you call them, the loop has finished and `i` is `2`. The fix is to bind the current value at definition time, most idiomatically via a **default argument** (defaults are evaluated immediately):

```python
# FIXED with a default argument
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)   # `i=i` snapshots the current value

print([f() for f in funcs])   # => [0, 1, 2]

# Also fixed: a factory function creates a fresh scope per iteration
def make(i):
    return lambda: i

funcs = [make(i) for i in range(3)]
print([f() for f in funcs])   # => [0, 1, 2]
```

Keep this in mind: it bites people who build lists of callbacks, event handlers, or — yes — decorators inside loops.

## Core concept — What a decorator actually is

A **decorator is a function that takes a function and returns a (usually wrapped) function.** That's the entire definition. The `@` syntax is pure sugar.

These two snippets are *identical*:

```python
# Using @ syntax
@my_decorator
def greet():
    print("hi")

# Desugared — exactly what Python does for you:
def greet():
    print("hi")
greet = my_decorator(greet)
```

So `@my_decorator` means: "take the function defined right below me, pass it to `my_decorator`, and rebind the original name to whatever `my_decorator` returns." Everything else is detail.

### A simple decorator

Let's write one that announces when a function runs:

```python
def announce(func):
    def wrapper():
        print(f"Calling {func.__name__}...")
        result = func()
        print(f"{func.__name__} finished.")
        return result
    return wrapper

@announce
def make_coffee():
    print("brewing")
    return "espresso"

cup = make_coffee()
# => Calling make_coffee...
# => brewing
# => make_coffee finished.
print(cup)   # => espresso
```

`announce(make_coffee)` returns `wrapper`, and `make_coffee` now *is* `wrapper`. Calling `make_coffee()` actually runs `wrapper()`, which runs the original inside it.

### Handling any arguments with `*args` and `**kwargs`

The `wrapper` above only works for zero-argument functions. To wrap *any* function, accept arbitrary positional and keyword arguments and forward them:

```python
def announce(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}...")
        result = func(*args, **kwargs)    # forward everything
        print(f"{func.__name__} returned {result!r}")
        return result
    return wrapper

@announce
def add(a, b):
    return a + b

print(add(3, 4))
# => Calling add...
# => add returned 7
# => 7
```

`*args` collects positional arguments into a tuple, `**kwargs` collects keyword arguments into a dict; using them again at the call site *unpacks* them back. This `wrapper(*args, **kwargs)` pattern is the standard shape of almost every decorator you'll write.

## Core concept — Preserving metadata with `functools.wraps`

There's a hidden cost above: the original function's identity is lost. After decoration, `add.__name__` is `"wrapper"`, its docstring is gone, and tools that rely on introspection (debuggers, Sphinx docs, FastAPI's signature inspection) get confused.

```python
def announce(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@announce
def add(a, b):
    """Add two numbers."""
    return a + b

print(add.__name__)   # => wrapper      (wrong!)
print(add.__doc__)    # => None         (lost the docstring!)
```

The fix is one line: decorate your `wrapper` with `functools.wraps(func)`. It copies `__name__`, `__doc__`, `__module__`, `__qualname__`, `__dict__`, and stashes the original on `__wrapped__`.

```python
import functools

def announce(func):
    @functools.wraps(func)            # <-- copy metadata from func to wrapper
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@announce
def add(a, b):
    """Add two numbers."""
    return a + b

print(add.__name__)        # => add
print(add.__doc__)         # => Add two numbers.
print(add.__wrapped__)     # => <function add at 0x...>   (the original)
```

**Always use `functools.wraps`** in your decorators. It costs nothing and prevents a class of subtle bugs, especially in web frameworks that inspect function signatures.

## Core concept — Decorators that accept arguments

Often you want to configure a decorator: `@repeat(3)`, `@retry(times=5)`, `@require_role("admin")`. This needs **three** levels of nesting, because `@repeat(3)` is not a decorator — it's a *call that returns a decorator*.

Remember the desugaring. `@repeat(3)` becomes:

```python
greet = repeat(3)(greet)
```

So `repeat(3)` must return a decorator, and that decorator wraps the function. Three layers:

```python
import functools

def repeat(times):                        # 1. factory: takes the config
    def decorator(func):                  # 2. the actual decorator: takes the func
        @functools.wraps(func)
        def wrapper(*args, **kwargs):     # 3. wrapper: takes the call args
            result = None
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    print(f"Hello, {name}!")
    return name

greet("Ada")
# => Hello, Ada!
# => Hello, Ada!
# => Hello, Ada!
```

A mental model for the three levels:

| Level | Name | Receives | Returns |
|-------|------|----------|---------|
| 1 | factory (`repeat`) | the decorator's *arguments* | the decorator |
| 2 | decorator | the *function* being decorated | the wrapper |
| 3 | wrapper | the *call-time arguments* | the result |

If you ever get confused by "how many `def`s do I need," count: a plain decorator is two levels, a parameterized one is three.

## Core concept — Stacking decorators and order

You can apply more than one decorator. They stack **bottom-up at decoration time** and the resulting calls run **top-down (outermost first)** at call time.

```python
import functools

def bold(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return "<b>" + func(*args, **kwargs) + "</b>"
    return wrapper

def italic(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return "<i>" + func(*args, **kwargs) + "</i>"
    return wrapper

@bold
@italic
def greet():
    return "hi"

print(greet())   # => <b><i>hi</i></b>
```

The stack `@bold` over `@italic` desugars to:

```python
greet = bold(italic(greet))
```

So `italic` wraps first (closest to the function), then `bold` wraps that. At call time the *outermost* (`bold`) runs first, delegates inward to `italic`, which delegates to the real `greet`. Order matters: swap them and you get `<i><b>hi</b></i>`. This is exactly why, in DRF or Flask, `@login_required` placed above or below a route decorator changes behavior — the outer one runs first.

## Core concept — Class-based decorators with `__call__`

A decorator just needs to be *callable* and return something callable. A class instance becomes callable if the class defines `__call__`. Class-based decorators are handy when the decorator needs to hold **state** between calls (counters, caches) in a clean, explicit way.

```python
import functools

class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)   # the class-based wraps() equivalent
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call #{self.count} of {self.func.__name__}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hi():
    print("hi")

say_hi()   # => Call #1 of say_hi  /  hi
say_hi()   # => Call #2 of say_hi  /  hi
print(say_hi.count)   # => 2
```

`@CountCalls` desugars to `say_hi = CountCalls(say_hi)`, so `say_hi` is now an *instance*. Calling `say_hi()` invokes `__call__`. Note `functools.update_wrapper` is the function form of `functools.wraps` — use it when you can't use the `@wraps` decorator (e.g., copying onto `self`).

For a *parameterized* class-based decorator, take the args in `__init__` and the function in `__call__`:

```python
class Retry:
    def __init__(self, times):
        self.times = times

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, self.times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as exc:
                    print(f"Attempt {attempt} failed: {exc}")
            raise RuntimeError(f"All {self.times} attempts failed")
        return wrapper

@Retry(times=3)
def flaky():
    raise ValueError("boom")
```

## Core concept — Decorating methods

Decorating a method works the same way, because a method is just a function whose first parameter is `self`. Your `*args` already captures `self` as the first positional argument, so a well-written decorator needs no changes.

```python
import functools

def log_calls(func):
    @functools.wraps(func)
    def wrapper(self, *args, **kwargs):     # `self` made explicit for clarity
        print(f"{type(self).__name__}.{func.__name__} called")
        return func(self, *args, **kwargs)
    return wrapper

class Account:
    def __init__(self, balance):
        self.balance = balance

    @log_calls
    def deposit(self, amount):
        self.balance += amount
        return self.balance

acc = Account(100)
print(acc.deposit(50))
# => Account.deposit called
# => 150
```

Beware the built-in method decorators you'll meet constantly — they are *not* ordinary wrappers and change how the method is accessed:

- `@staticmethod` — no `self`/`cls`; a plain function living in the class namespace.
- `@classmethod` — receives the class as `cls` instead of an instance.
- `@property` — turns a method into a read-only attribute (getter); pair with `@<name>.setter`.

```python
class Circle:
    def __init__(self, radius):
        self.radius = radius

    @property
    def area(self):                  # accessed as c.area, NOT c.area()
        return 3.14159 * self.radius ** 2

    @classmethod
    def unit(cls):                   # alternative constructor
        return cls(radius=1)

    @staticmethod
    def describe():
        return "A round shape."

c = Circle(2)
print(c.area)            # => 12.56636   (no parentheses!)
print(Circle.unit().area)  # => 3.14159
print(Circle.describe())   # => A round shape.
```

If you write a custom decorator *and* combine it with `@classmethod`/`@staticmethod`, put your decorator **below** them (closer to the function), because `classmethod`/`staticmethod` must be the outermost result.

## Core concept — Real-world decorators

### Timing

```python
import functools
import time

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed = time.perf_counter() - start
            print(f"{func.__name__} took {elapsed:.4f}s")
    return wrapper

@timed
def slow_sum(n):
    return sum(range(n))

slow_sum(1_000_000)   # => slow_sum took 0.0123s
```

Using `try/finally` guarantees the timing prints even if the function raises.

### Logging with a configurable level

```python
import functools
import logging

logging.basicConfig(level=logging.INFO)

def log(level=logging.INFO):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            logging.log(level, "Calling %s(%r, %r)", func.__name__, args, kwargs)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@log(level=logging.WARNING)
def charge(card, amount):
    return f"charged {amount} to {card}"

charge("**1234", 9.99)
# => WARNING:root:Calling charge(('**1234', 9.99), {})
```

### Caching with `functools.lru_cache`

Don't hand-roll caching — the standard library has a fast, thread-safe LRU cache decorator. It memoizes results keyed by the arguments (which must be hashable).

```python
import functools

@functools.lru_cache(maxsize=128)
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

print(fib(50))            # => 12586269025  (instant; without cache this is glacial)
print(fib.cache_info())   # => CacheInfo(hits=48, misses=51, maxsize=128, currsize=51)
fib.cache_clear()         # wipe the cache when you need to
```

Use `@functools.cache` (Python 3.9+) for an unbounded version (equivalent to `lru_cache(maxsize=None)`). Caveat: arguments must be hashable, so no `list`/`dict` args — pass tuples instead.

### Retry with backoff

```python
import functools
import time

def retry(times=3, delay=0.5, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    last_exc = exc
                    print(f"[{func.__name__}] attempt {attempt}/{times} failed: {exc}")
                    if attempt < times:
                        time.sleep(delay * attempt)   # linear backoff
            raise last_exc
        return wrapper
    return decorator

@retry(times=3, delay=0.1, exceptions=(ConnectionError,))
def fetch():
    raise ConnectionError("network down")

# fetch()  # would try 3 times, then re-raise ConnectionError
```

### A simple auth check

This is the conceptual core of how web frameworks gate endpoints.

```python
import functools

class AuthError(Exception):
    pass

def require_role(role):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(user, *args, **kwargs):
            if role not in user.get("roles", []):
                raise AuthError(f"'{role}' role required")
            return func(user, *args, **kwargs)
        return wrapper
    return decorator

@require_role("admin")
def delete_everything(user):
    return "all gone"

admin = {"name": "root", "roles": ["admin"]}
guest = {"name": "bob", "roles": ["user"]}

print(delete_everything(admin))   # => all gone
# delete_everything(guest)        # raises AuthError: 'admin' role required
```

## Core concept — How Django / Flask / FastAPI use decorators

This is where it all pays off for a backend role. Frameworks expose huge amounts of behavior through decorators.

**Flask** uses decorators for routing — `@app.route` registers the function in a URL map:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    return {"id": user_id, "name": "Ada"}
```

**FastAPI** does the same with HTTP-method-specific decorators, and reads the function's type-annotated signature (which is exactly why `functools.wraps` and signature preservation matter so much):

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"id": user_id, "name": "Ada"}
```

**Django** uses decorators for view-level concerns. In classic views you'll see auth and HTTP-method guards:

```python
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods

@login_required
@require_http_methods(["POST"])
def create_post(request):
    # only authenticated users, only POST requests reach here
    return ...
```

**Django REST Framework** offers `@api_view` for function-based views and `@action` to add extra routes to a ViewSet:

```python
from rest_framework.decorators import api_view, action
from rest_framework.response import Response
from rest_framework.viewsets import ModelViewSet

@api_view(["GET"])
def ping(request):
    return Response({"status": "ok"})

class OrderViewSet(ModelViewSet):
    @action(detail=True, methods=["post"])
    def cancel(self, request, pk=None):
        return Response({"cancelled": pk})
```

In every case the principle is identical to your hand-written `require_role`: the decorator either **registers** the function with the framework (routing) or **wraps** it with cross-cutting behavior (auth, method checks, throttling). Stacking order matters here too — `@login_required` above the method decorator means the login check runs first.

## Common mistakes & gotchas

- **Forgetting `functools.wraps`.** Your wrapped function loses its name, docstring, and signature — and FastAPI/DRF introspection can break. Always include it.
- **Calling instead of referencing.** `@my_decorator` (no parens) for a plain decorator vs. `@my_decorator(arg)` (with parens) for a parameterized one. Mixing them up gives `TypeError: 'function' object is not callable` or a wrapper that receives the function where it expected config.
- **Wrong number of nesting levels.** Plain decorator = 2 `def`s; parameterized = 3 `def`s. A common bug is writing a parameterized decorator with only two levels.
- **Not forwarding `*args, **kwargs`.** A `wrapper()` that takes no arguments only works on zero-arg functions; everything else raises `TypeError`.
- **The late-binding loop closure.** Building decorators or callbacks in a loop without `x=x` default binding captures the final loop value for all of them.
- **`lru_cache` on methods.** Caching on an instance method keeps `self` alive (memory leak) and requires hashable args; prefer `functools.cached_property` for per-instance caching, or cache at the function level.
- **Caching mutable arguments.** `lru_cache` needs hashable args — passing a `list` or `dict` raises `TypeError: unhashable type`. Convert to tuples/frozensets first.
- **Decorator runs at import time.** The outer decorator code executes when the module is imported (when the `def` is read), not when the function is called. Heavy work in the decorator body slows startup.
- **Order with `classmethod`/`staticmethod`.** These must be the outermost decorator; put your custom one beneath them.

## Best practices

- Reach for `functools.wraps` in *every* decorator you write — no exceptions.
- Prefer standard-library decorators when they exist: `functools.lru_cache` / `functools.cache` for memoization, `functools.cached_property` for lazy per-instance attributes.
- Use `*args, **kwargs` so your decorator works on any function signature.
- Keep decorators focused on a single cross-cutting concern (logging *or* timing *or* auth) — compose by stacking rather than building one mega-decorator.
- Use `try/finally` when the decorator must run cleanup/measurement even on exceptions.
- For parameterized decorators with sensible defaults, allow bare usage by giving keyword args defaults (e.g. `@retry()` works because `times` defaults to 3).
- Reach for class-based decorators when you need persistent state; reach for function-based ones for simple wrapping.
- Document what your decorator does to the signature/return value — callers can't see the wrapping.

## Interview Q&A

**Q: In one sentence, what is a decorator?**
**A:** A callable that takes a function (or class) and returns a replacement — usually one that adds behavior around the original — applied with the `@` syntax, which is just sugar for `func = decorator(func)`.

**Q: What is a closure and how does it relate to decorators?**
**A:** A closure is an inner function that captures and remembers variables from its enclosing scope even after that scope has returned. Decorators rely on closures: the `wrapper` function closes over the original `func` (and any decorator arguments) so it can call it later.

**Q: Why do you need `functools.wraps`?**
**A:** Without it, the wrapper replaces the original's metadata — `__name__`, `__doc__`, `__qualname__`, signature — which breaks introspection, documentation tools, debuggers, and frameworks like FastAPI that read the function's signature. `wraps` copies that metadata onto the wrapper and records the original under `__wrapped__`.

**Q: How does a decorator that takes arguments work?**
**A:** It needs three nesting levels because `@deco(arg)` desugars to `func = deco(arg)(func)`. The outermost function (the "factory") receives the arguments and returns the real decorator, which receives the function and returns the wrapper. So: factory → decorator → wrapper.

**Q: If you stack `@a` over `@b` over a function, what is the call order?**
**A:** They wrap bottom-up: `f = a(b(f))`, so `b` is innermost. At call time the outermost runs first: `a`'s wrapper executes, then delegates to `b`'s wrapper, then to the original. Order matters and changing it can change behavior.

**Q: Explain the late-binding closure gotcha in loops.**
**A:** Closures capture variables by reference, not by value, and resolve them at call time. A `lambda` created in a `for i in range(3)` loop captures the variable `i`, so all of them see the final value `2`. Fix it with a default-argument snapshot (`lambda i=i: i`) or a factory function that creates a fresh scope.

**Q: When would you use a class-based decorator instead of a function-based one?**
**A:** When the decorator needs to maintain state across calls (a call counter, a cache, accumulated metrics) or expose methods/attributes, a class with `__call__` is cleaner and more explicit than nested closures with `nonlocal`.

**Q: How do you decorate a method, and what's the catch?**
**A:** It's the same as decorating a function — `self` simply arrives as the first positional argument, which your `*args` already captures. The catch is the built-in descriptors: `@classmethod`/`@staticmethod` must be the outermost decorator (place your custom one below them), and `@property` changes attribute access semantics.

**Q: What does `functools.lru_cache` do and what are its limitations?**
**A:** It memoizes a function's return values keyed by its arguments, with a least-recently-used eviction policy bounded by `maxsize`. Arguments must be hashable, the function should be pure (no side effects / deterministic), and on methods it can leak memory by keeping `self` alive. It exposes `cache_info()` and `cache_clear()`.

**Q: How do web frameworks use decorators?**
**A:** Two ways. For routing, decorators like `@app.get(...)` (FastAPI), `@app.route(...)` (Flask), or DRF's `@action` *register* the function with the framework's URL map. For cross-cutting concerns, decorators like Django's `@login_required` or `@require_http_methods` *wrap* the view to enforce auth or HTTP-method rules before the body runs.

**Q: Does decorator code run at import time or call time?**
**A:** The decorator *application* (the outer function call that returns the wrapper) runs at import time, when the `def` statement is executed. The *wrapper body* runs each time the decorated function is called. So expensive setup in a decorator slows module import, not the calls.

**Q: How would you write a decorator that works both with and without arguments (e.g., `@deco` and `@deco(x=1)`)?**
**A:** Make the function parameter optional and positional-only-ish: `def deco(func=None, *, x=1)`. If `func` is `None`, you were called with arguments, so return a partial/inner decorator; otherwise you were applied directly, so wrap `func` immediately. It's a known pattern but adds complexity, so use it only when the dual API is genuinely needed.

## Practice exercises

Try each before peeking at the solutions.

1. **`debug`**: Write a decorator that prints the function name, its positional and keyword arguments, and its return value on every call. Preserve metadata.

2. **`repeat(n)`**: Write a parameterized decorator that calls the wrapped function `n` times and returns a list of the results.

3. **`@count_calls`**: Write a class-based decorator that tracks how many times a function has been called and exposes the total as a `.count` attribute.

4. **`@cache_result`**: Write a memoizing decorator from scratch (a dict keyed by args) for a single-positional-argument function — then explain why you'd normally use `functools.lru_cache` instead.

5. **`@require_login`**: Write a decorator for a method on a `Service` class that raises `PermissionError` unless `self.user` is truthy.

6. **Bonus**: Fix this broken loop so each printed value is its own index:
   ```python
   printers = [lambda: i for i in range(3)]
   print([p() for p in printers])   # currently [2, 2, 2]; make it [0, 1, 2]
   ```

<details>
<summary>Solutions</summary>

```python
import functools

# 1. debug
def debug(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"{func.__name__}(args={args}, kwargs={kwargs})")
        result = func(*args, **kwargs)
        print(f"  -> {result!r}")
        return result
    return wrapper

@debug
def add(a, b=0):
    return a + b

add(2, b=3)
# => add(args=(2,), kwargs={'b': 3})
# =>   -> 5


# 2. repeat(n)
def repeat(n):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            return [func(*args, **kwargs) for _ in range(n)]
        return wrapper
    return decorator

@repeat(3)
def roll():
    return 4   # chosen by fair dice roll

print(roll())   # => [4, 4, 4]


# 3. count_calls (class-based)
class count_calls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.func(*args, **kwargs)

@count_calls
def ping():
    return "pong"

ping(); ping()
print(ping.count)   # => 2


# 4. cache_result (DIY memoization)
def cache_result(func):
    store = {}
    @functools.wraps(func)
    def wrapper(arg):
        if arg not in store:
            store[arg] = func(arg)
        return store[arg]
    return wrapper

@cache_result
def square(x):
    print(f"computing {x}")
    return x * x

print(square(4))   # => computing 4 / 16
print(square(4))   # => 16   (no "computing" — served from cache)
# Why use functools.lru_cache instead? It is C-optimized, thread-safe,
# supports multiple/keyword args, bounds memory via maxsize (LRU eviction),
# and gives you cache_info()/cache_clear() for free.


# 5. require_login (method decorator)
def require_login(func):
    @functools.wraps(func)
    def wrapper(self, *args, **kwargs):
        if not self.user:
            raise PermissionError("login required")
        return func(self, *args, **kwargs)
    return wrapper

class Service:
    def __init__(self, user=None):
        self.user = user

    @require_login
    def secret(self):
        return "classified"

print(Service(user="ada").secret())   # => classified
# Service().secret()                  # raises PermissionError


# 6. Bonus: fix late binding with a default-argument snapshot
printers = [lambda i=i: i for i in range(3)]
print([p() for p in printers])   # => [0, 1, 2]
```

</details>

## Cheat sheet

```python
# --- Plain decorator (2 levels) ---
import functools
def deco(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # before
        result = func(*args, **kwargs)
        # after
        return result
    return wrapper

# --- Parameterized decorator (3 levels) ---
def deco(opt):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            return func(*args, **kwargs)
        return wrapper
    return decorator

# --- Class-based (stateful) ---
class Deco:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
    def __call__(self, *args, **kwargs):
        return self.func(*args, **kwargs)
```

| Concept | Key fact |
|---------|----------|
| `@deco` desugars to | `f = deco(f)` |
| `@deco(x)` desugars to | `f = deco(x)(f)` |
| Plain decorator | 2 `def`s (func → wrapper) |
| Parameterized | 3 `def`s (factory → decorator → wrapper) |
| Preserve metadata | `@functools.wraps(func)` (or `update_wrapper` in classes) |
| Forward args | `wrapper(*args, **kwargs)` → `func(*args, **kwargs)` |
| Stacking `@a` over `@b` | `a(b(f))`; `a` runs first at call time |
| Stateful decorator | class with `__call__` |
| Memoize | `@functools.lru_cache(maxsize=N)` / `@functools.cache` |
| Lazy attr | `@functools.cached_property` |
| Late-binding fix | `lambda x=x: x` (snapshot in default arg) |
| Runs at import | decorator application; wrapper body runs per call |

| Stdlib decorator | Use |
|------------------|-----|
| `functools.wraps` | copy metadata onto a wrapper |
| `functools.lru_cache` / `cache` | memoize return values |
| `functools.cached_property` | compute an attribute once per instance |
| `staticmethod` / `classmethod` | method that takes no `self` / takes `cls` |
| `property` | method accessed like an attribute |

| Framework | Decorator | Purpose |
|-----------|-----------|---------|
| Flask | `@app.route(...)` | register URL route |
| FastAPI | `@app.get(...)` / `@app.post(...)` | register typed endpoint |
| Django | `@login_required`, `@require_http_methods` | auth / method guard |
| DRF | `@api_view`, `@action` | function view / extra ViewSet route |
