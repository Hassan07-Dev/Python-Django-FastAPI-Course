# 19 — Functional Tools: functools, itertools, collections

> Week 2 · Python advanced — the standard-library toolkit every backend developer reuses daily

## What you will learn

- A quick recap of `lambda` and where it actually belongs
- `map()` / `filter()` and the honest answer to "why does everyone use comprehensions instead?"
- `functools.reduce` — folding a sequence into one value
- `functools.partial` — pre-filling arguments to make new callables
- `functools.lru_cache` / `functools.cache` — memoization for free
- `functools.wraps` and `functools.total_ordering` (recap, in context)
- Sorting and selecting with `key=`, plus `operator.itemgetter` / `attrgetter`
- The `itertools` heavy hitters: `chain`, `product`, `permutations`, `combinations`, `groupby` (and its sorting trap), `count`, `cycle`, `islice`, `accumulate`
- The `collections` workhorses: `defaultdict`, `Counter`, `OrderedDict` (and why it barely matters now), `deque`, `namedtuple`
- When functional style is Pythonic — and when it is just showing off

## PHP/Laravel bridge

If you have used Laravel Collections (`collect($users)->map(...)->filter(...)->groupBy(...)`), you already understand the *spirit* of this lesson. Python splits that toolbox into three standard-library modules instead of one fluent class: `functools` (function-shaping tools like memoization and `reduce`), `itertools` (lazy iterator combinators — Python's answer to chunking, zipping, and grouping), and `collections` (specialized containers like `Counter`, which is roughly Laravel's `countBy`, and `defaultdict`). The big mental shift: Python prefers **comprehensions and generators** over chained method calls, and most `itertools` functions are **lazy** (they yield on demand like a PHP `Generator`), so you can pipe over millions of rows without loading them all into memory. Where PHP gives you one `Collection` object with 100+ methods, Python gives you small composable functions you wire together.

---

## Core concept: lambda, the one-line function

A `lambda` is an anonymous, single-expression function. It is exactly equivalent to a `def` that `return`s one expression — there is no body, no statements, no `return` keyword.

```python
# These two are equivalent:
def double(x):
    return x * 2

double = lambda x: x * 2   # frowned upon as a *named* lambda — just use def

print(double(21))  # => 42
```

The reason `lambda` exists is to pass a tiny function as an argument *inline*, where naming it would be noise:

```python
pairs = [("apple", 3), ("banana", 1), ("cherry", 2)]
pairs.sort(key=lambda pair: pair[1])   # sort by the count
print(pairs)  # => [('banana', 1), ('cherry', 2), ('apple', 3)]
```

Key rules and limits:

- A `lambda` can only contain **one expression** — no `if/else` statements (the ternary `a if cond else b` is fine, it is an expression), no loops, no assignments.
- PEP 8 says: **never assign a lambda to a variable.** If it needs a name, it needs a `def`. The only legitimate use is as a throwaway argument.

```python
# Bad (linters will warn): E731 do not assign a lambda
clean = lambda s: s.strip().lower()

# Good:
def clean(s):
    return s.strip().lower()
```

---

## Core concept: map(), filter(), and why comprehensions usually win

`map(func, iterable)` applies `func` to every element. `filter(func, iterable)` keeps elements where `func` returns truthy. Both return **lazy iterators** in Python 3 (not lists), so you usually wrap them in `list()` to see results.

```python
nums = [1, 2, 3, 4, 5]

doubled = list(map(lambda n: n * 2, nums))
print(doubled)  # => [2, 4, 6, 8, 10]

evens = list(filter(lambda n: n % 2 == 0, nums))
print(evens)    # => [2, 4]
```

So why does the Python community reach for **comprehensions** instead? Compare:

```python
# map + filter + lambda
result = list(map(lambda n: n * 2, filter(lambda n: n % 2 == 0, nums)))

# the comprehension equivalent — reads left to right, no lambdas
result = [n * 2 for n in nums if n % 2 == 0]
print(result)  # => [4, 8]
```

The comprehension is shorter, faster (no per-element function-call overhead from the `lambda`), and reads in natural order. **Rule of thumb:**

| Situation | Prefer |
|---|---|
| You already have a named function to apply | `map(func, it)` is clean: `map(str.upper, names)` |
| You need a transform *and* a filter | Comprehension |
| You'd have to write `lambda` for `map`/`filter` | Comprehension |
| You want laziness over a huge stream | Generator expression `(x for x in it)` or `map` (both lazy) |

```python
# map shines when the function already has a name — no lambda needed:
names = ["  Ada ", "Linus", " Grace "]
print(list(map(str.strip, names)))  # => ['Ada', 'Linus', 'Grace']
```

Note the **generator expression** — same syntax as a comprehension but with `()` — is lazy like `map`, giving you memory-efficient pipelines:

```python
total = sum(n * n for n in range(1_000_000))  # no intermediate list built
print(total)  # => 333332833333500000
```

---

## Core concept: functools.reduce

`reduce(func, iterable, initializer)` folds a sequence into a single value by repeatedly applying a two-argument function: `func(func(func(init, a), b), c) ...`. It lives in `functools` (it was *removed* from builtins in Python 3 precisely because it is rarely the clearest tool).

```python
from functools import reduce

nums = [1, 2, 3, 4]
total = reduce(lambda acc, n: acc + n, nums, 0)
print(total)  # => 10  (((0+1)+2)+3)+4
```

But for the common cases there is almost always a better, clearer builtin:

```python
print(sum(nums))            # => 10   (not reduce)
print(max(nums))            # => 4    (not reduce)
import math
print(math.prod(nums))      # => 24   (product, since Python 3.8 — not reduce)
```

`reduce` earns its keep only for *non-trivial* accumulation that has no builtin:

```python
from functools import reduce

# Merge a list of dicts (later keys win) — no clean builtin for this
configs = [{"a": 1}, {"b": 2}, {"a": 9}]
merged = reduce(lambda acc, d: {**acc, **d}, configs, {})
print(merged)  # => {'a': 9, 'b': 2}
```

If you find yourself writing `reduce`, pause and ask whether an explicit `for` loop would be clearer. Often it is.

---

## Core concept: functools.partial

`partial(func, *args, **kwargs)` returns a **new callable** with some arguments already filled in. Think of it as "bind these parameters now, supply the rest later." This is the cleanest way to adapt a general function into a specific one without writing a wrapper.

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)

print(square(5))  # => 25
print(cube(2))    # => 8
```

A practical backend example: building a logger that always tags its source, or a configured HTTP call.

```python
from functools import partial

def log(level, message, *, service):
    print(f"[{level}] ({service}) {message}")

# Pre-bind the service so call sites stay clean:
api_log = partial(log, service="payments-api")
api_log("INFO", "charge succeeded")   # => [INFO] (payments-api) charge succeeded
api_log("WARN", "retrying webhook")   # => [WARN] (payments-api) retrying webhook
```

`partial` is also handy when a callback API wants a zero-or-one-argument function but yours needs more:

```python
from functools import partial

def connect(host, port, timeout):
    return f"connecting to {host}:{port} (t={timeout})"

# A scheduler that calls task() with no args — partial supplies them:
task = partial(connect, "db.internal", 5432, timeout=3)
print(task())  # => connecting to db.internal:5432 (t=3)
```

Why prefer `partial` over a `lambda`? `partial` objects carry the original function and bound args as attributes (`.func`, `.args`, `.keywords`), pickle better, and signal intent more clearly than `lambda: connect(...)`.

---

## Core concept: lru_cache and cache — memoization for free

`functools.lru_cache` wraps a function so repeated calls with the **same arguments** return a cached result instead of recomputing. "LRU" = Least Recently Used; when the cache fills (`maxsize`), the oldest-used entries are evicted. `functools.cache` (Python 3.9+) is simply `lru_cache(maxsize=None)` — an unbounded cache.

```python
from functools import lru_cache

@lru_cache(maxsize=None)   # or just @cache
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(50))            # => 12586269025  (instant; without cache this is exponential)
print(fib.cache_info())   # => CacheInfo(hits=48, misses=51, maxsize=None, currsize=51)
fib.cache_clear()         # wipe the cache when you need to
```

The newer, cleaner spelling for "cache everything forever":

```python
from functools import cache

@cache
def slow_lookup(user_id: int) -> str:
    print(f"hitting DB for {user_id}")
    return f"user-{user_id}"

print(slow_lookup(1))  # => hitting DB for 1  /  user-1
print(slow_lookup(1))  # => user-1   (no DB hit — served from cache)
```

Critical caveats — interviewers love these:

- **Arguments must be hashable.** You can cache on `int`, `str`, `tuple`, `frozenset`, but **not** on `list`, `dict`, or `set` arguments — they'll raise `TypeError: unhashable type`.
- The cache lives for the **lifetime of the process** and grows unbounded with `cache` / `maxsize=None`. That is a memory leak risk on functions with many distinct arguments — set a `maxsize`.
- Caching only makes sense for **pure functions** (same input → same output, no side effects you need each time). Do not cache something whose result depends on the current time, DB state, or randomness — you'll serve stale data.
- For an expensive *property*, combine with `@property`, or just use `functools.cached_property` (computes once per instance, stores on the instance):

```python
from functools import cached_property

class Report:
    def __init__(self, rows):
        self.rows = rows

    @cached_property
    def total(self):
        print("computing total...")
        return sum(self.rows)

r = Report([1, 2, 3])
print(r.total)  # => computing total... / 6
print(r.total)  # => 6   (cached on the instance; no recompute)
```

---

## Core concept: wraps and total_ordering (recap in context)

These two belong to `functools` and you will have met them around decorators and dunder methods. A quick refresher because they are exam favorites.

**`functools.wraps`** — when you write a decorator, the inner wrapper function replaces the original, clobbering its `__name__`, `__doc__`, and signature. `@wraps(func)` copies that metadata back, which keeps debuggers, docs, and frameworks (Flask/Django routing, pytest) working correctly.

```python
from functools import wraps

def timed(func):
    @wraps(func)                      # <- preserves func's identity
    def wrapper(*args, **kwargs):
        import time
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper

@timed
def greet(name):
    """Say hello."""
    return f"Hi {name}"

print(greet.__name__)  # => greet   (without @wraps this would print 'wrapper')
print(greet.__doc__)   # => Say hello.
```

**`functools.total_ordering`** — a class decorator that fills in the missing comparison dunders. Define `__eq__` plus **one** of `__lt__`, `__le__`, `__gt__`, `__ge__`, and it generates the rest.

```python
from functools import total_ordering

@total_ordering
class Version:
    def __init__(self, major, minor):
        self.major, self.minor = major, minor

    def __eq__(self, other):
        return (self.major, self.minor) == (other.major, other.minor)

    def __lt__(self, other):
        return (self.major, self.minor) < (other.major, other.minor)

print(Version(2, 1) > Version(2, 0))   # => True   (>= , > , <= all derived)
print(Version(1, 5) <= Version(1, 5))  # => True
```

---

## Core concept: sorted / min / max with key, and operator helpers

`sorted()`, `min()`, and `max()` all accept a `key=` callable that maps each element to the value you want to compare by. This is the single most useful "functional" pattern in day-to-day backend code.

```python
users = [
    {"name": "Ada", "age": 36},
    {"name": "Linus", "age": 21},
    {"name": "Grace", "age": 85},
]

oldest = max(users, key=lambda u: u["age"])
print(oldest["name"])  # => Grace

by_name = sorted(users, key=lambda u: u["name"])
print([u["name"] for u in by_name])  # => ['Ada', 'Grace', 'Linus']
```

Instead of `lambda u: u["age"]`, the `operator` module gives you faster, clearer picker functions:

- `operator.itemgetter("age")` — for dicts/sequences (indexing with `[]`)
- `operator.attrgetter("age")` — for objects (attribute access with `.`)

```python
from operator import itemgetter, attrgetter

by_age = sorted(users, key=itemgetter("age"))
print([u["name"] for u in by_age])  # => ['Linus', 'Ada', 'Grace']

# Multi-key sort: by age, then name — itemgetter takes several keys
rows = [("b", 2), ("a", 2), ("c", 1)]
print(sorted(rows, key=itemgetter(1, 0)))  # => [('c', 1), ('a', 2), ('b', 2)]
```

`attrgetter` is the equivalent for objects and even supports dotted paths:

```python
from operator import attrgetter

class Order:
    def __init__(self, id, total):
        self.id, self.total = id, total

orders = [Order(1, 50), Order(2, 30), Order(3, 90)]
top = max(orders, key=attrgetter("total"))
print(top.id)  # => 3
```

Two more essentials:

- **Stable sort + `reverse=True`**: Python's sort is stable, so equal-key items keep their original order. Use `reverse=True` for descending.
- **Descending on one key, ascending on another**: sort twice (sort is stable) or negate a numeric key: `key=lambda u: (-u["age"], u["name"])`.

```python
print(sorted(users, key=lambda u: (-u["age"], u["name"]))[0]["name"])  # => Grace
```

---

## Core concept: itertools — lazy iterator power tools

`itertools` provides composable, memory-efficient iterators. They are **lazy**: nothing is computed until you iterate. Import what you need.

### chain — flatten / concatenate iterables

```python
from itertools import chain

a, b = [1, 2], [3, 4]
print(list(chain(a, b)))  # => [1, 2, 3, 4]

# chain.from_iterable flattens one level — great for lists of lists:
nested = [[1, 2], [3], [4, 5]]
print(list(chain.from_iterable(nested)))  # => [1, 2, 3, 4, 5]
```

### product, permutations, combinations — combinatorics

```python
from itertools import product, permutations, combinations

# product = nested loops / Cartesian product
print(list(product([1, 2], ["a", "b"])))
# => [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

# permutations = ordered arrangements (order matters)
print(list(permutations([1, 2, 3], 2)))
# => [(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]

# combinations = unordered selections (order does NOT matter)
print(list(combinations([1, 2, 3], 2)))
# => [(1, 2), (1, 3), (2, 3)]
```

### groupby — group consecutive items (MUST sort first!)

This is the classic trap. `itertools.groupby` groups only **consecutive** elements with the same key. If your data is not sorted by that key, you get fragmented groups. **Always `sorted()` by the same key first** unless the data is already ordered.

```python
from itertools import groupby
from operator import itemgetter

people = [
    {"city": "NY", "name": "Ada"},
    {"city": "LA", "name": "Linus"},
    {"city": "NY", "name": "Grace"},  # NY again, but not adjacent
]

# WRONG — not sorted, so 'NY' appears twice as two separate groups:
for city, group in groupby(people, key=itemgetter("city")):
    print(city, [p["name"] for p in group])
# => NY ['Ada']
# => LA ['Linus']
# => NY ['Grace']

# RIGHT — sort by the same key first:
people.sort(key=itemgetter("city"))
for city, group in groupby(people, key=itemgetter("city")):
    print(city, [p["name"] for p in group])
# => LA ['Linus']
# => NY ['Ada', 'Grace']
```

Note also: the `group` is a **one-shot iterator**. Consume it (e.g. `list(group)`) before advancing to the next group, or it'll be exhausted. For arbitrary (non-consecutive) grouping, a `defaultdict(list)` is usually simpler than `groupby` — see below.

### count, cycle, islice — infinite iterators, tamed

`count` and `cycle` are *infinite*, so you must bound them with `islice` (or a `break`), or you'll loop forever.

```python
from itertools import count, cycle, islice

# count(start, step) — infinite counter
for i in islice(count(10, 2), 4):   # take first 4
    print(i, end=" ")               # => 10 12 14 16
print()

# cycle — repeats a sequence forever (round-robin assignment)
servers = cycle(["s1", "s2", "s3"])
print([next(servers) for _ in range(5)])  # => ['s1', 's2', 's3', 's1', 's2']

# islice = lazy slicing for any iterator (you can't do gen[2:5])
print(list(islice(range(100), 2, 8, 2)))  # => [2, 4, 6]  (start, stop, step)
```

`islice` is the iterator equivalent of `seq[start:stop:step]` — essential because plain generators and iterators do not support `[]` slicing.

### accumulate — running totals (and more)

`accumulate(iterable, func=operator.add)` yields the running result. Default is a cumulative sum; pass any binary function for running max, product, etc.

```python
from itertools import accumulate
import operator

print(list(accumulate([1, 2, 3, 4])))                 # => [1, 3, 6, 10]  (running sum)
print(list(accumulate([1, 2, 3, 4], operator.mul)))   # => [1, 2, 6, 24]  (running product)
print(list(accumulate([3, 1, 4, 1, 5], max)))         # => [3, 3, 4, 4, 5]  (running max)
```

---

## Core concept: collections — specialized containers

The `collections` module ships container types that solve problems plain `dict`/`list` solve awkwardly.

### defaultdict — no more KeyError

A `defaultdict(factory)` calls `factory()` to create a default value the first time you touch a missing key. Perfect for grouping and counting without `if key in d` boilerplate.

```python
from collections import defaultdict

# Group names by city — the readable alternative to groupby:
people = [("NY", "Ada"), ("LA", "Linus"), ("NY", "Grace")]
by_city = defaultdict(list)
for city, name in people:
    by_city[city].append(name)   # no KeyError; empty list created on demand

print(dict(by_city))  # => {'NY': ['Ada', 'Grace'], 'LA': ['Linus']}

# Counting with defaultdict(int):
counts = defaultdict(int)
for ch in "mississippi":
    counts[ch] += 1
print(dict(counts))  # => {'m': 1, 'i': 4, 's': 4, 'p': 2}
```

### Counter — counting done right

`Counter` is a `dict` subclass purpose-built for tallying. It is Laravel's `countBy` and then some.

```python
from collections import Counter

votes = ["yes", "no", "yes", "yes", "no", "abstain"]
c = Counter(votes)
print(c)                  # => Counter({'yes': 3, 'no': 2, 'abstain': 1})
print(c["yes"])           # => 3
print(c["missing"])       # => 0   (no KeyError — missing keys count as 0)
print(c.most_common(2))   # => [('yes', 3), ('no', 2)]

# Counters support arithmetic:
print(Counter("aab") + Counter("abc"))  # => Counter({'a': 3, 'b': 2, 'c': 1})

# Count characters / words in one line:
print(Counter("mississippi"))  # => Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
```

### OrderedDict — the nuance (you rarely need it now)

Historically `dict` did not preserve insertion order, so `OrderedDict` existed to guarantee it. **Since Python 3.7, regular `dict` preserves insertion order**, so for ordering alone you don't need `OrderedDict`. It survives for two niche reasons:

```python
from collections import OrderedDict

# 1) Order-sensitive equality: OrderedDict compares order, dict does not.
print({"a": 1, "b": 2} == {"b": 2, "a": 1})                    # => True
print(OrderedDict(a=1, b=2) == OrderedDict(b=2, a=1))          # => False

# 2) move_to_end() and popitem(last=False) — handy for LRU-style structures.
od = OrderedDict(a=1, b=2, c=3)
od.move_to_end("a")
print(list(od))                 # => ['b', 'c', 'a']
print(od.popitem(last=False))   # => ('b', 2)  (pop from the front)
```

### deque — fast queues and stacks

A `deque` ("deck", double-ended queue) gives **O(1)** appends and pops at *both* ends. A plain `list` is O(1) at the right end but **O(n)** at the left (`list.pop(0)` / `insert(0, ...)` shift everything). Use `deque` for queues, stacks, and sliding windows.

```python
from collections import deque

q = deque([1, 2, 3])
q.append(4)         # add right    => deque([1, 2, 3, 4])
q.appendleft(0)     # add left     => deque([0, 1, 2, 3, 4])
print(q.popleft())  # remove left  => 0   (FIFO queue: append + popleft)
print(q.pop())      # remove right => 4   (LIFO stack: append + pop)

# Bounded deque = automatic sliding window / ring buffer:
last3 = deque(maxlen=3)
for x in range(6):
    last3.append(x)        # old items fall off the left automatically
print(last3)               # => deque([3, 4, 5], maxlen=3)

# rotate is O(k):
d = deque([1, 2, 3, 4])
d.rotate(1)
print(d)  # => deque([4, 1, 2, 3])
```

### namedtuple — lightweight, immutable records

`namedtuple` creates a tuple subclass with **named fields** — readable like a class but as light as a tuple. Great for fixed-shape records (DB rows, coordinates, config). For richer needs, `dataclasses` (a later lesson) or `typing.NamedTuple` add type hints.

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
print(p.x, p.y)     # => 3 4   (named access)
print(p[0])         # => 3     (still index-able like a tuple)
print(p)            # => Point(x=3, y=4)   (nice repr for free)

# Immutable: p.x = 5 -> AttributeError. Make a changed copy instead:
p2 = p._replace(y=10)
print(p2)              # => Point(x=3, y=10)
print(p._asdict())     # => {'x': 3, 'y': 4}   (turn into a dict)

# Typed alternative (preferred in modern code):
from typing import NamedTuple
class Coord(NamedTuple):
    x: int
    y: int = 0          # defaults allowed
print(Coord(1))         # => Coord(x=1, y=0)
```

---

## Core concept: when functional style is (and isn't) Pythonic

Python supports functional tools but is **not** a functional-first language like Haskell. The community values readability above cleverness ("Pythonic"). Guidelines:

**Lean functional / built-in tools when:**
- A comprehension or generator expresses the transform clearly.
- You apply an existing named function: `map(str.strip, lines)`.
- You need laziness over a big/infinite stream (`itertools`, generators).
- A specialized container fits exactly (`Counter`, `defaultdict`, `deque`).

**Avoid / step back when:**
- You're nesting `map`/`filter`/`lambda` — a comprehension reads better.
- You reach for `reduce` and a `for` loop or builtin (`sum`, `max`, `math.prod`) would be obvious.
- A `lambda` grows beyond a trivial expression — promote it to a `def`.
- You'd sacrifice clarity for a one-liner. **Explicit is better than implicit** (Zen of Python).

```python
# Clever but cryptic:
result = reduce(lambda a, b: a + b, map(lambda x: x**2, filter(lambda x: x % 2, data)))

# Pythonic — anyone can read it:
result = sum(x**2 for x in data if x % 2)
```

---

## Common mistakes & gotchas

- **`map`/`filter` return iterators, not lists.** `print(map(...))` shows `<map object ...>`; you must `list()` them, and they are **single-use** — iterating twice yields nothing the second time.
- **`groupby` without sorting.** It groups only *consecutive* equal keys. Forgetting to `sorted()` first is the #1 `itertools` bug. Also: each group is a one-shot iterator — `list()` it before moving on.
- **`count` / `cycle` are infinite.** `for x in count(): ...` with no `break`/`islice` hangs forever.
- **`lru_cache` on unhashable args.** Passing a `list`/`dict`/`set` raises `TypeError: unhashable type`. Convert to a `tuple`/`frozenset` first, or don't cache.
- **Caching impure functions.** Caching a function whose output depends on time, DB, or randomness serves stale results. Cache only pure functions.
- **Unbounded `cache` as a memory leak.** `@cache` / `maxsize=None` keeps every distinct call forever. Set a `maxsize` for functions with many argument combinations.
- **Naming a lambda.** `f = lambda x: ...` trips linters (E731). Use `def`.
- **Forgetting `@wraps` in a decorator.** The wrapped function loses its `__name__`/`__doc__`/signature, breaking introspection, docs, and some frameworks.
- **`namedtuple` is immutable.** `p.x = 5` raises `AttributeError`; use `_replace()` to get a modified copy.
- **`list.pop(0)` in a loop.** O(n) each call → O(n²) overall. Use `deque.popleft()` for queues.
- **Mutable default factory confusion.** `defaultdict(list)` takes the factory *callable* (`list`), not a value (`list()` / `[]`). `defaultdict([])` raises `TypeError`.

## Best practices

- Reach for a **comprehension or generator expression** first; use `map`/`filter` only with an existing named function.
- Use `operator.itemgetter` / `attrgetter` over `lambda` in `sorted`/`min`/`max` keys — faster and clearer.
- Prefer the right **specialized container** over hand-rolled logic: `Counter` for tallies, `defaultdict(list)` for grouping, `deque` for queues/stacks.
- Use `@cache` for simple unbounded memoization of pure functions; switch to `@lru_cache(maxsize=N)` when the input space is large.
- Always `@functools.wraps` your decorator wrappers.
- For grouping non-adjacent data, prefer `defaultdict(list)` over sorting + `groupby`.
- Bound infinite iterators (`count`, `cycle`) with `islice` or an explicit termination.
- Use `partial` (not `lambda`) to pre-bind arguments — it's clearer and introspectable.
- Use `math.prod`, `sum`, `min`, `max`, `any`, `all` before considering `reduce`.

## Interview Q&A

**Q: Why are comprehensions usually preferred over `map`/`filter` with `lambda`?**
**A:** They read in natural order, avoid the per-element overhead of calling a `lambda`, and combine transform-and-filter in one expression. `map`/`filter` are still idiomatic when applying an *existing named* function (e.g. `map(str.strip, lines)`), where no lambda is needed.

**Q: What does `map()` return in Python 3, and what's the catch?**
**A:** A lazy `map` iterator, not a list. You wrap it in `list()` to materialize it, and it's single-use — once consumed it yields nothing. The laziness is a feature for large streams.

**Q: When would you actually use `functools.reduce`?**
**A:** Rarely. For sums/products/min/max use `sum`/`math.prod`/`min`/`max`. `reduce` is justified only for custom accumulation with no builtin equivalent (e.g. folding a list of dicts into one), and even then an explicit loop is often clearer.

**Q: Explain `lru_cache` and its main pitfalls.**
**A:** It memoizes a function: same arguments return a cached result. Pitfalls — arguments must be **hashable** (no `list`/`dict`), the cache grows unbounded with `maxsize=None`/`@cache` (memory risk), and caching **impure** functions (time/DB/random dependent) serves stale data. `cache_clear()` resets it; `cache_info()` reports hits/misses.

**Q: Difference between `functools.cache` and `functools.lru_cache`?**
**A:** `cache` (3.9+) is exactly `lru_cache(maxsize=None)` — unbounded, slightly faster (no LRU bookkeeping). Use `lru_cache(maxsize=N)` when you need eviction to cap memory.

**Q: What's the classic `itertools.groupby` mistake?**
**A:** It only groups **consecutive** equal-key items. If the data isn't sorted by the key, the same key appears in multiple groups. Always `sorted()` by the same key first (or use `defaultdict(list)` for arbitrary grouping). Each group is also a one-shot iterator.

**Q: When do you use `deque` instead of `list`?**
**A:** When you push/pop from the **front** (queues) or both ends. `deque` gives O(1) `appendleft`/`popleft`, whereas `list.pop(0)`/`insert(0,...)` are O(n). `deque(maxlen=n)` also gives a free sliding window / ring buffer.

**Q: `partial` vs `lambda` — why prefer `partial`?**
**A:** `partial` clearly signals "pre-bind arguments," exposes `.func`/`.args`/`.keywords` for introspection, and pickles cleanly. A `lambda` is opaque and can't be pickled. Use `partial` to specialize a general function.

**Q: Why do decorators need `functools.wraps`?**
**A:** The wrapper function replaces the original, so without `@wraps(func)` the decorated function reports the wrapper's `__name__`, `__doc__`, and signature — breaking debugging, help text, and frameworks that introspect functions. `@wraps` copies that metadata over.

**Q: What does `functools.total_ordering` do?**
**A:** It's a class decorator: define `__eq__` and one of `__lt__`/`__le__`/`__gt__`/`__ge__`, and it auto-generates the remaining comparison operators — less boilerplate, consistent ordering.

**Q: Counter vs defaultdict(int) for counting?**
**A:** Both work. `Counter` is purpose-built: `most_common()`, arithmetic (`+`, `-`, `&`, `|`), construction straight from an iterable, and missing keys return 0. `defaultdict(int)` is fine for ad-hoc tallies but lacks those conveniences.

**Q: Is `OrderedDict` still needed now that `dict` keeps insertion order?**
**A:** Mostly no, for ordering alone (guaranteed since 3.7). It's still useful for **order-sensitive equality** (`OrderedDict` compares order; `dict` doesn't) and methods like `move_to_end()` / `popitem(last=False)` for LRU-style structures.

## Practice exercises

1. Given `words = ["pear", "apple", "fig", "banana", "kiwi"]`, use `sorted` with a key to order them by length (shortest first); break ties alphabetically. Then get the longest word with `max`.

2. Count word frequency in the sentence `"the cat sat on the mat the cat ran"` and return the two most common words with their counts. Use the right `collections` type.

3. You have `transactions = [("food", 12), ("rent", 800), ("food", 8), ("toys", 30), ("rent", 50)]`. Produce a dict mapping each category to its total amount — first with `defaultdict`, then with `sorted` + `itertools.groupby`.

4. Write a memoized function `paths(n)` that counts the ways to climb `n` stairs taking 1 or 2 steps at a time (`paths(n) = paths(n-1) + paths(n-2)`, `paths(0)=paths(1)=1`). Use `functools.cache` and verify `paths(10) == 89`.

5. Using `itertools`, generate all 2-element **combinations** of `["A", "B", "C", "D"]`, then separately all 2-element **permutations**, and explain the count difference in a comment.

6. Build a `partial` called `to_json` from `json.dumps` that always pretty-prints with `indent=2` and `sort_keys=True`, then dump `{"b": 1, "a": 2}`.

<details>
<summary>Solutions</summary>

```python
# 1) Sort by length, tie-break alphabetically; then longest.
words = ["pear", "apple", "fig", "banana", "kiwi"]
ordered = sorted(words, key=lambda w: (len(w), w))
print(ordered)  # => ['fig', 'kiwi', 'pear', 'apple', 'banana']
print(max(words, key=len))  # => banana


# 2) Word frequency with Counter.
from collections import Counter
sentence = "the cat sat on the mat the cat ran"
counts = Counter(sentence.split())
print(counts.most_common(2))  # => [('the', 3), ('cat', 2)]


# 3) Category totals — two ways.
from collections import defaultdict
from itertools import groupby
from operator import itemgetter

transactions = [("food", 12), ("rent", 800), ("food", 8), ("toys", 30), ("rent", 50)]

# (a) defaultdict — no sorting needed, handles non-adjacent keys:
totals = defaultdict(int)
for category, amount in transactions:
    totals[category] += amount
print(dict(totals))  # => {'food': 20, 'rent': 850, 'toys': 30}

# (b) sorted + groupby — MUST sort by the key first:
totals2 = {}
for category, group in groupby(sorted(transactions, key=itemgetter(0)),
                               key=itemgetter(0)):
    totals2[category] = sum(amount for _, amount in group)
print(totals2)  # => {'food': 20, 'rent': 850, 'toys': 30}


# 4) Memoized stair climbing.
from functools import cache

@cache
def paths(n):
    if n < 2:
        return 1
    return paths(n - 1) + paths(n - 2)

print(paths(10))  # => 89


# 5) Combinations vs permutations.
from itertools import combinations, permutations
items = ["A", "B", "C", "D"]
print(list(combinations(items, 2)))
# => [('A','B'),('A','C'),('A','D'),('B','C'),('B','D'),('C','D')]  -> 6 (order ignored)
print(list(permutations(items, 2)))
# => 12 results -> every ordered pair; (A,B) and (B,A) both counted.
# combinations = n!/(r!(n-r)!) = 6 ; permutations = n!/(n-r)! = 12.


# 6) partial over json.dumps.
import json
from functools import partial

to_json = partial(json.dumps, indent=2, sort_keys=True)
print(to_json({"b": 1, "a": 2}))
# => {
# =>   "a": 2,
# =>   "b": 1
# => }
```

</details>

## Cheat sheet

| Tool | Use it for | Quick example |
|---|---|---|
| `lambda x: ...` | tiny inline function | `sorted(xs, key=lambda x: x[1])` |
| `map(f, it)` | apply named func, lazily | `map(str.strip, lines)` |
| `filter(f, it)` | keep matching, lazily | `filter(None, items)` (drops falsy) |
| comprehension | transform + filter (preferred) | `[x*2 for x in xs if x>0]` |
| `reduce(f, it, init)` | custom fold (last resort) | `reduce(lambda a,d:{**a,**d}, ds, {})` |
| `partial(f, *a, **kw)` | pre-bind arguments | `partial(log, service="api")` |
| `@cache` / `@lru_cache(maxsize=N)` | memoize pure funcs | `@cache def fib(n): ...` |
| `cached_property` | compute-once instance attr | `@cached_property def total(self): ...` |
| `@wraps(func)` | keep metadata in decorators | inside every wrapper |
| `@total_ordering` | derive comparison ops | define `__eq__` + `__lt__` |
| `itemgetter/attrgetter` | fast sort/min/max keys | `sorted(rows, key=itemgetter(1,0))` |

| itertools | Meaning |
|---|---|
| `chain(a, b)` / `chain.from_iterable(nested)` | concatenate / flatten one level |
| `product(a, b)` | Cartesian product (nested loops) |
| `permutations(it, r)` | ordered arrangements |
| `combinations(it, r)` | unordered selections |
| `groupby(sorted_it, key)` | group **consecutive** equal keys (sort first!) |
| `count(start, step)` | infinite counter (bound with `islice`) |
| `cycle(it)` | repeat forever (round-robin) |
| `islice(it, start, stop, step)` | lazy slicing of any iterator |
| `accumulate(it, func)` | running total / max / product |

| collections | Meaning |
|---|---|
| `defaultdict(list/int)` | auto-create missing values; grouping/counting |
| `Counter(it)` | tally; `.most_common(n)`, arithmetic, 0 for missing |
| `OrderedDict` | order-sensitive equality, `move_to_end`, `popitem(last=False)` |
| `deque` | O(1) both-ends; queues/stacks; `maxlen` sliding window |
| `namedtuple("T", [...])` | immutable named record; `_replace`, `_asdict` |

Builtins to prefer over `reduce`: `sum`, `max`, `min`, `math.prod`, `any`, `all`, `sorted`.
