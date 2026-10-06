# 58 — Coding Challenges

> Week 5 · Interview prep & practical problem solving

This lesson is your hands-on dojo. Backend interviews almost always include a live-coding round: a few small problems where the interviewer watches *how* you think, not just whether you arrive at the answer. The goal here is to make idiomatic Python feel automatic — so when you're under pressure you reach for `Counter`, a dict comprehension, or `bisect` instead of writing C-style loops.

I'll narrate each solution the way you'd talk through it aloud in the room: state the approach, write clean code, mention complexity, and call out edge cases.

## What you will learn

- The "interview voice": how to talk through a problem out loud while coding.
- String & array problems: reverse words, anagram check, two-sum, group anagrams, most-frequent element.
- Idiomatic data-structure use: `dict`, `set`, `Counter`, `defaultdict`, `OrderedDict`.
- Comprehensions for transforming and filtering data.
- Sorting with custom keys (`sorted(..., key=...)`, `operator.itemgetter`).
- Recursion done right: factorial, Fibonacci with memoization (`functools.lru_cache`).
- Core algorithms: binary search (`bisect`), FizzBuzz, palindrome check.
- Working with JSON/dict data: flatten nested structures, parse & aggregate.
- A file-processing aggregation task (log parsing).
- Backend-flavored design tasks: a rate limiter, an LRU cache, Pydantic validation, and a custom decorator.

## PHP/Laravel bridge

You've solved these same problems in PHP — the algorithms are language-agnostic. The difference is the toolbox. Where PHP gives you `array_count_values()`, Python gives you `collections.Counter`. Where you'd write `usort($arr, fn($a,$b)=>...)`, Python uses `sorted(arr, key=...)` — note that Python sorts by a *key function* (map each item to a comparable value) rather than a *comparator* (compare two items), which is cleaner and faster. PHP's associative array is split in Python into two distinct types: `dict` (key→value) and `set` (unique membership). And Python's list comprehensions replace most of your `array_map`/`array_filter` chains with one readable expression. Master the toolbox and these problems become short.

## Core concept: how to approach an interview problem

Before any code, say this out loud — it signals senior-level thinking:

1. **Restate the problem** and confirm input/output types. ("So I get a list of ints and a target, and return the two indices?")
2. **Ask about edge cases.** Empty input? Duplicates? Negative numbers? Case sensitivity? Unicode?
3. **State a brute-force baseline** with its complexity, then improve it.
4. **Code the clean version**, narrating as you go.
5. **Test with one normal case and one edge case** by tracing through.
6. **State final time/space complexity.**

Most candidates jump straight to code. Doing steps 1–3 first is what separates a hire from a maybe.

---

## Core concept: string & array manipulation

### Problem 1 — Reverse the words in a sentence

> Given `"the quick brown fox"`, return `"fox brown quick the"`. Word order reversed, not characters.

```python
def reverse_words(sentence: str) -> str:
    # str.split() with no args splits on ANY run of whitespace and
    # drops empties — so "  a   b  " -> ["a", "b"]. That handles
    # messy input for free.
    words = sentence.split()
    return " ".join(reversed(words))

print(reverse_words("the quick brown fox"))   # => fox brown quick the
print(reverse_words("  hello   world  "))     # => world hello
```

A one-liner with slicing is also idiomatic: `" ".join(sentence.split()[::-1])`. The `[::-1]` slice means "step backwards through the whole list" — Python's signature reverse idiom.

**Complexity:** O(n) time, O(n) space (we build a new list and string; strings are immutable in Python so you can't reverse in place).

### Problem 2 — Anagram check

> Are `"listen"` and `"silent"` anagrams (same letters, any order)?

```python
from collections import Counter

def is_anagram(a: str, b: str) -> bool:
    return Counter(a) == Counter(b)

print(is_anagram("listen", "silent"))   # => True
print(is_anagram("hello", "world"))     # => False
```

`Counter` builds a multiset (letter → count) and `==` compares them directly. This is O(n). The sort-based alternative `sorted(a) == sorted(b)` is O(n log n) but a fine answer — mention both and explain the trade-off.

*Interview nuance:* ask whether case and spaces matter. If "Dormitory" / "Dirty Room" should match, normalize first:

```python
def is_anagram_normalized(a: str, b: str) -> bool:
    clean = lambda s: Counter(c.lower() for c in s if c.isalpha())
    return clean(a) == clean(b)

print(is_anagram_normalized("Dormitory", "Dirty Room"))   # => True
```

### Problem 3 — Two-Sum

> Given a list of ints and a target, return the indices of the two numbers that add to the target.

```python
def two_sum(nums: list[int], target: int) -> tuple[int, int] | None:
    seen: dict[int, int] = {}          # value -> index we've already passed
    for i, n in enumerate(nums):
        complement = target - n
        if complement in seen:         # dict lookup is O(1) average
            return (seen[complement], i)
        seen[n] = i
    return None

print(two_sum([2, 7, 11, 15], 9))   # => (0, 1)
print(two_sum([3, 2, 4], 6))        # => (1, 2)
print(two_sum([1, 2], 100))         # => None
```

The naive solution is two nested loops, O(n²). The insight: as we scan, we remember every value we've seen in a dict. For each new number we ask "have I already seen the complement?" — an O(1) lookup. That trades O(n) memory for O(n) time. **This dict-as-lookup trick is the single most common pattern in array interviews.**

### Problem 4 — Group anagrams

> Group words that are anagrams of each other: `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"],["tan","nat"],["bat"]]`.

```python
from collections import defaultdict

def group_anagrams(words: list[str]) -> list[list[str]]:
    groups: dict[tuple[str, ...], list[str]] = defaultdict(list)
    for word in words:
        key = tuple(sorted(word))   # canonical signature; must be hashable
        groups[key].append(word)
    return list(groups.values())

print(group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"]))
# => [['eat', 'tea', 'ate'], ['tan', 'nat'], ['bat']]
```

Two words are anagrams iff their sorted letters match, so the sorted tuple is a perfect group key. We use `tuple` (not `list`) because dict keys must be hashable — lists are mutable and therefore unhashable. `defaultdict(list)` means "if the key is missing, start it as an empty list," so we skip the `if key not in groups` boilerplate.

**Complexity:** O(n · k log k) where n = number of words, k = max word length (the sort dominates per word).

### Problem 5 — Most frequent element

> Return the element that appears most often. Bonus: top-k.

```python
from collections import Counter

def most_frequent(items: list) -> object:
    return Counter(items).most_common(1)[0][0]

def top_k_frequent(items: list, k: int) -> list:
    return [val for val, _count in Counter(items).most_common(k)]

data = ["a", "b", "a", "c", "a", "b"]
print(most_frequent(data))        # => a
print(top_k_frequent(data, 2))    # => ['a', 'b']
```

`Counter.most_common(n)` returns the n highest-count pairs already sorted descending — built for exactly this. `most_common(1)` returns `[('a', 3)]`, so `[0][0]` grabs the value. This is where knowing the stdlib pays off: a from-scratch version is 10 lines.

---

## Core concept: comprehensions for transforming data

List/dict/set comprehensions are Python's most beloved feature. They replace `map`/`filter` loops with a single declarative expression.

```python
nums = [1, 2, 3, 4, 5, 6]

# Transform: square each
squares = [n * n for n in nums]                 # => [1, 4, 9, 16, 25, 36]

# Filter: keep evens
evens = [n for n in nums if n % 2 == 0]         # => [2, 4, 6]

# Transform + filter together
even_squares = [n * n for n in nums if n % 2 == 0]   # => [4, 16, 36]

# Dict comprehension: build a lookup table
square_map = {n: n * n for n in nums}           # => {1: 1, 2: 4, ...}

# Set comprehension: unique remainders
remainders = {n % 3 for n in nums}              # => {0, 1, 2}

# Invert a dict {k: v} -> {v: k}
prices = {"apple": 3, "pear": 2}
by_price = {v: k for k, v in prices.items()}    # => {3: 'apple', 2: 'pear'}

# Flatten a nested list (note the loop order: outer first, then inner)
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [x for row in matrix for x in row]       # => [1, 2, 3, 4, 5, 6]
```

**Rule of thumb:** if a comprehension needs more than one `if`/`for` and a transform, or wraps onto a third line, write a regular loop instead — readability wins. Comprehensions are for clarity, not for code golf.

For huge data, use a **generator expression** (parentheses instead of brackets) to stream lazily without building the whole list in memory:

```python
total = sum(n * n for n in range(1_000_000))   # no intermediate list created
```

---

## Core concept: sorting with custom keys

Python sorting is **key-based**: you supply a function that maps each item to a comparable value. `sorted()` returns a new list; `list.sort()` mutates in place. Both are stable (equal keys keep their original order).

```python
from operator import itemgetter

people = [
    {"name": "Alice", "age": 30},
    {"name": "Bob",   "age": 25},
    {"name": "Cara",  "age": 30},
]

# Sort by a single field
by_age = sorted(people, key=lambda p: p["age"])
# Bob(25), Alice(30), Cara(30) — Alice before Cara because sort is stable

# Multi-key sort: age ascending, then name ascending (tuple comparison)
by_age_then_name = sorted(people, key=lambda p: (p["age"], p["name"]))

# Descending
oldest_first = sorted(people, key=lambda p: p["age"], reverse=True)

# operator.itemgetter is faster than a lambda and reads cleanly
by_name = sorted(people, key=itemgetter("name"))

# Sort strings case-insensitively
words = ["banana", "Apple", "cherry"]
print(sorted(words, key=str.lower))   # => ['Apple', 'banana', 'cherry']

# Classic trick: sort by length, then alphabetically as tiebreaker
print(sorted(["bb", "a", "ccc", "aa"], key=lambda s: (len(s), s)))
# => ['a', 'aa', 'bb', 'ccc']
```

The tuple trick is the key takeaway: returning a tuple `(primary, secondary)` from the key function gives you multi-level sorting for free, because Python compares tuples element by element.

---

## Core concept: recursion & memoization

### Factorial

```python
def factorial(n: int) -> int:
    if n < 0:
        raise ValueError("factorial is undefined for negatives")
    if n in (0, 1):          # base case
        return 1
    return n * factorial(n - 1)   # recursive case

print(factorial(5))   # => 120
```

Always lead with the **base case** so the recursion terminates. In a real codebase you'd often prefer the iterative `math.factorial(n)` or a loop, because Python has no tail-call optimization and a default recursion limit (~1000 frames). Mention that trade-off.

### Fibonacci — and why naive recursion is a trap

```python
# Naive: O(2^n) — recomputes the same subproblems exponentially. Don't ship this.
def fib_naive(n: int) -> int:
    if n < 2:
        return n
    return fib_naive(n - 1) + fib_naive(n - 2)
```

The fix is **memoization** — cache results so each subproblem is computed once. `functools.lru_cache` does this with a decorator:

```python
from functools import lru_cache

@lru_cache(maxsize=None)        # None = unbounded cache
def fib(n: int) -> int:
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

print(fib(50))   # => 12586269025  (instant, vs. minutes for fib_naive)
```

That decorator alone turns O(2ⁿ) into O(n). It's a perfect thing to reach for in an interview because it shows you know the stdlib.

The interviewer-favorite follow-up is "now do it without recursion." The iterative version is O(n) time, O(1) space:

```python
def fib_iter(n: int) -> int:
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b     # tuple unpacking swaps cleanly — no temp var
    return a

print(fib_iter(10))   # => 55
```

Note `a, b = b, a + b`: the right side is fully evaluated first (into a tuple), then unpacked, so you don't need a temporary variable. This is the Pythonic swap.

---

## Core concept: classic algorithms

### Binary search

> Find a target in a *sorted* list; return its index or -1. O(log n).

```python
def binary_search(arr: list[int], target: int) -> int:
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2          # // is integer division
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1

print(binary_search([1, 3, 5, 7, 9, 11], 7))   # => 3
print(binary_search([1, 3, 5, 7, 9, 11], 8))   # => -1
```

The two off-by-one traps everyone hits: the loop must be `while lo <= hi` (not `<`), and the bounds move to `mid + 1` / `mid - 1` (not `mid`) or you loop forever. In production you'd just use the stdlib `bisect` module:

```python
import bisect

arr = [1, 3, 5, 7, 9]
i = bisect.bisect_left(arr, 5)
print(i, arr[i] == 5)              # => 2 True   (found)

# bisect also gives you the insertion point to keep a list sorted:
bisect.insort(arr, 6)
print(arr)                         # => [1, 3, 5, 6, 7, 9]
```

### FizzBuzz

> Print 1..n; multiples of 3 → "Fizz", of 5 → "Buzz", of both → "FizzBuzz". The classic filter for "can this person write a loop at all."

```python
def fizzbuzz(n: int) -> list[str]:
    out = []
    for i in range(1, n + 1):
        if i % 15 == 0:        # 15 = lcm(3,5); check the combined case FIRST
            out.append("FizzBuzz")
        elif i % 3 == 0:
            out.append("Fizz")
        elif i % 5 == 0:
            out.append("Buzz")
        else:
            out.append(str(i))
    return out

print(fizzbuzz(15))
# => ['1', '2', 'Fizz', '4', 'Buzz', 'Fizz', '7', '8', 'Fizz',
#     'Buzz', '11', 'Fizz', '13', '14', 'FizzBuzz']
```

The one bug to avoid: checking `% 3` before `% 15` means 15 prints "Fizz" instead of "FizzBuzz". Test the most specific condition first.

### Palindrome check

```python
def is_palindrome(s: str) -> bool:
    # Normalize: keep only alphanumerics, lowercased.
    cleaned = [c.lower() for c in s if c.isalnum()]
    return cleaned == cleaned[::-1]

print(is_palindrome("A man, a plan, a canal: Panama"))   # => True
print(is_palindrome("hello"))                            # => False
```

`cleaned[::-1]` is the reversed copy; comparing it to the original is the most readable approach. The interviewer may ask for O(1) space — then use a two-pointer walk from both ends inward, which avoids building the reversed copy.

---

## Core concept: working with JSON / nested dict data

Backend work is mostly shuffling JSON, so these come up constantly.

### Parse JSON and aggregate

```python
import json
from collections import defaultdict

raw = """
[
  {"user": "alice", "amount": 30, "currency": "USD"},
  {"user": "bob",   "amount": 20, "currency": "USD"},
  {"user": "alice", "amount": 15, "currency": "USD"}
]
"""

orders = json.loads(raw)   # str -> list[dict]

# Total spend per user
totals: dict[str, int] = defaultdict(int)
for o in orders:
    totals[o["user"]] += o["amount"]

print(dict(totals))   # => {'alice': 45, 'bob': 20}

# Serialize back to JSON (indent for readability)
print(json.dumps(totals, indent=2))
```

`json.loads` parses a string into Python objects; `json.dumps` does the reverse. Remember the mnemonic: the **s** suffix means "string" (`loads`/`dumps` work on strings; `load`/`dump` work on file objects).

### Flatten a nested dictionary

> Turn `{"a": {"b": 1, "c": {"d": 2}}}` into `{"a.b": 1, "a.c.d": 2}` — handy for config files and CSV export.

```python
def flatten(d: dict, parent: str = "", sep: str = ".") -> dict:
    items: dict = {}
    for key, value in d.items():
        new_key = f"{parent}{sep}{key}" if parent else key
        if isinstance(value, dict):
            items.update(flatten(value, new_key, sep))   # recurse
        else:
            items[new_key] = value
    return items

nested = {"a": {"b": 1, "c": {"d": 2}}, "e": 3}
print(flatten(nested))   # => {'a.b': 1, 'a.c.d': 2, 'e': 3}
```

The recursion mirrors the nesting: for each value, if it's another dict we recurse with an extended key prefix; otherwise we've hit a leaf and record it.

---

## Core concept: file processing & aggregation

A very common "realistic" task: parse a log file and produce a summary. Here we count requests per HTTP status code and find the top IP.

```python
from collections import Counter

SAMPLE_LOG = """\
192.168.0.1 GET /home 200
192.168.0.2 GET /login 200
192.168.0.1 POST /login 401
192.168.0.1 GET /home 200
192.168.0.3 GET /missing 404
"""

def analyze_log(text: str) -> dict:
    status_counts: Counter = Counter()
    ip_counts: Counter = Counter()
    for line in text.splitlines():
        if not line.strip():
            continue                       # skip blank lines defensively
        parts = line.split()
        if len(parts) != 4:
            continue                       # skip malformed lines
        ip, _method, _path, status = parts
        status_counts[status] += 1
        ip_counts[ip] += 1
    return {
        "by_status": dict(status_counts),
        "top_ip": ip_counts.most_common(1)[0][0],
        "total_requests": sum(status_counts.values()),
    }

print(analyze_log(SAMPLE_LOG))
# => {'by_status': {'200': 3, '401': 1, '404': 1},
#     'top_ip': '192.168.0.1', 'total_requests': 5}
```

In real life you'd stream the file instead of loading it all:

```python
def analyze_file(path: str) -> Counter:
    counts: Counter = Counter()
    with open(path, encoding="utf-8") as f:   # context manager auto-closes
        for line in f:                        # iterates line-by-line, lazily
            counts[line.split()[-1]] += 1     # count by status (last field)
    return counts
```

Two senior signals here: a `with` block so the file always closes (even on error), and iterating the file object directly so memory stays constant no matter how big the file is. Mentioning "I'd stream a multi-GB log rather than `read()` it all" scores points.

---

## Core concept: backend-flavored design tasks

These are where backend interviews differ from pure algorithm screens. They test class design and knowledge of real-world patterns.

### Design task 1 — A custom decorator (timing)

> Write a decorator that logs how long a function takes. Decorators are huge in Python web frameworks (`@app.get`, `@login_required`), so being fluent matters.

```python
import functools
import time

def timed(func):
    @functools.wraps(func)                 # preserves func.__name__, docstring
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed * 1000:.2f} ms")
        return result
    return wrapper

@timed
def slow_add(a: int, b: int) -> int:
    time.sleep(0.05)
    return a + b

print(slow_add(2, 3))
# => slow_add took ~50.xx ms
# => 5
```

The three things interviewers check: `*args, **kwargs` so the wrapper accepts any signature, `return result` so the wrapped function's return value isn't swallowed, and `@functools.wraps(func)` so metadata (name, docstring) is preserved. Forgetting `@wraps` is the classic mistake — without it, `slow_add.__name__` would wrongly report `"wrapper"`.

### Design task 2 — A rate limiter

> Allow at most `max_calls` per `period` seconds (a sliding window). This is straight from real API gateway work.

```python
import time
from collections import deque

class RateLimiter:
    """Sliding-window rate limiter: max_calls allowed within `period` seconds."""

    def __init__(self, max_calls: int, period: float):
        self.max_calls = max_calls
        self.period = period
        self.calls: deque[float] = deque()    # timestamps of recent calls

    def allow(self) -> bool:
        now = time.monotonic()                # monotonic: immune to clock changes
        # Evict timestamps that have aged out of the window.
        while self.calls and self.calls[0] <= now - self.period:
            self.calls.popleft()              # popleft is O(1) on a deque
        if len(self.calls) < self.max_calls:
            self.calls.append(now)
            return True
        return False

# Demo: 2 calls allowed per second
rl = RateLimiter(max_calls=2, period=1.0)
print(rl.allow())   # => True
print(rl.allow())   # => True
print(rl.allow())   # => False  (window full)
```

Why `deque`? It's a double-ended queue with O(1) `append` and `popleft` — perfect for a FIFO of timestamps. A plain list's `pop(0)` is O(n) because it shifts every element. Why `time.monotonic()`? It never goes backwards (NTP adjustments or DST can move `time.time()`), so it's the correct clock for measuring elapsed durations.

### Design task 3 — An LRU cache

> Build a fixed-capacity cache that evicts the **L**east **R**ecently **U**sed item when full. `OrderedDict` makes this clean because it remembers insertion order and lets you cheaply move a key to the end.

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self.capacity = capacity
        self.store: OrderedDict = OrderedDict()

    def get(self, key):
        if key not in self.store:
            return None
        self.store.move_to_end(key)        # mark as most-recently-used
        return self.store[key]

    def put(self, key, value) -> None:
        if key in self.store:
            self.store.move_to_end(key)
        self.store[key] = value
        if len(self.store) > self.capacity:
            self.store.popitem(last=False)  # evict the OLDEST (front) item

cache = LRUCache(capacity=2)
cache.put("a", 1)
cache.put("b", 2)
print(cache.get("a"))   # => 1  (now 'a' is most-recently-used)
cache.put("c", 3)       # capacity exceeded -> evicts 'b' (least recent)
print(cache.get("b"))   # => None  (evicted)
print(cache.get("c"))   # => 3
```

The two `OrderedDict` superpowers doing the work: `move_to_end(key)` (re-marks a key as freshest in O(1)) and `popitem(last=False)` (removes the oldest item from the front). If the interviewer just wants *a* cache and doesn't require a class, the honest answer is "I'd use `@functools.lru_cache`" — but they usually want to see you implement the eviction logic, so know both.

### Design task 4 — Parse & validate input with Pydantic

> Validate an incoming API payload: coerce types, enforce constraints, reject bad data with clear errors. This is the modern Python way (FastAPI is built on it).

```python
from pydantic import BaseModel, EmailStr, Field, field_validator, ValidationError

class SignupRequest(BaseModel):
    username: str = Field(min_length=3, max_length=20)
    email: EmailStr
    age: int = Field(ge=18, le=120)        # ge = >=, le = <=
    tags: list[str] = []                   # default empty list

    @field_validator("username")
    @classmethod
    def no_spaces(cls, v: str) -> str:
        if " " in v:
            raise ValueError("username must not contain spaces")
        return v.lower()                   # validators can transform too

# Valid input — types are coerced (note age comes in as a string "25")
ok = SignupRequest(username="Alice", email="alice@example.com", age="25")
print(ok.model_dump())
# => {'username': 'alice', 'email': 'alice@example.com', 'age': 25, 'tags': []}

# Invalid input — Pydantic raises one error listing every problem
try:
    SignupRequest(username="al", email="not-an-email", age=15)
except ValidationError as e:
    print(e.error_count(), "errors")       # => 3 errors
```

Pydantic v2 notes worth saying aloud: use `model_dump()` (not the deprecated v1 `.dict()`), `field_validator` decorated with `@classmethod` for custom rules, and `Field(...)` constraints for declarative validation. Pydantic *coerces* compatible types (the string `"25"` became the int `25`) and collects *all* validation errors at once rather than failing on the first — great for API error responses.

---

## Common mistakes & gotchas

- **Mutable default arguments.** `def f(items=[]):` reuses the *same* list across calls — a classic bug. Use `def f(items=None): items = items or []`.
- **Checking membership in a list when you want a set.** `x in big_list` is O(n); `x in big_set` is O(1). For repeated lookups, build a `set` first.
- **`list.pop(0)` in a loop** is O(n) each time → O(n²) overall. Use `collections.deque` and `popleft()`.
- **Forgetting `@functools.wraps`** in a decorator destroys the wrapped function's `__name__`/docstring and breaks introspection.
- **FizzBuzz ordering:** test `% 15` before `% 3`/`% 5`, or 15 prints "Fizz".
- **Binary search off-by-one:** `while lo <= hi` and move bounds to `mid ± 1`, never `mid`.
- **Naive recursive Fibonacci** is O(2ⁿ) — always memoize or iterate.
- **`dict`/`set` keys must be hashable.** You can't use a `list` as a key; convert to `tuple` first (as in group-anagrams).
- **Pydantic v1 muscle memory:** don't call `.dict()` or use a bare inner `class Config` for everything — use `model_dump()` and `ConfigDict`/`Field` in v2.
- **Confusing `sort()` and `sorted()`:** `list.sort()` mutates and returns `None`; `sorted()` returns a new list. Don't write `x = mylist.sort()`.

## Best practices

- **Reach for the stdlib first:** `Counter`, `defaultdict`, `deque`, `OrderedDict`, `bisect`, `functools.lru_cache`, `operator.itemgetter`. Knowing these *is* idiomatic Python.
- **Name things clearly** even in interviews — `complement`, `seen`, `status_counts` beat `x`, `d`, `tmp`. Readable code reads as senior.
- **State complexity unprompted** after each solution. It shows rigor.
- **Handle edge cases explicitly:** empty input, single element, duplicates, negatives, case sensitivity. Ask about them up front.
- **Prefer comprehensions for simple transforms**, regular loops when logic grows.
- **Use generators for large/streaming data** to keep memory flat.
- **Add type hints** (`list[int]`, `dict[str, int]`) — they're modern, document intent, and interviewers notice.
- **Talk continuously.** Silence during live coding is the enemy; narrate your reasoning.

## Interview Q&A

**Q: What's the difference between a list and a tuple, and when would you use each?**
A: A list is mutable and a tuple is immutable. Use a tuple for fixed-size, heterogeneous, or "value" data — and crucially when you need something hashable (dict keys, set members). Tuples are also marginally faster and signal "this won't change." Use a list for ordered, growable collections.

**Q: How does `collections.Counter` help in interviews?**
A: It builds a frequency map in one line (`Counter(iterable)`), supports arithmetic (`c1 + c2`, `c1 - c2`), gives you `most_common(k)` already sorted, and compares as a multiset (`==`). It turns anagram checks, top-k, and frequency problems into two-line solutions.

**Q: Explain `defaultdict` vs a regular dict.**
A: `defaultdict(factory)` auto-creates a missing key by calling the factory (`list`, `int`, `set`), so you skip `if key not in d` checks. `defaultdict(int)` is perfect for counting; `defaultdict(list)` for grouping. A plain dict raises `KeyError` on a missing key (or you use `.get(key, default)`).

**Q: How do you reverse a string or list in Python?**
A: Slicing: `s[::-1]` returns a reversed copy. For lists you can also do `list(reversed(lst))` (an iterator) or `lst.reverse()` (in place). Strings are immutable, so you always get a new string.

**Q: Why is naive recursive Fibonacci slow, and how do you fix it?**
A: It recomputes overlapping subproblems, giving O(2ⁿ) calls. Fix it with memoization — decorate with `@functools.lru_cache(maxsize=None)` to cache results (O(n)), or write the iterative two-variable version for O(n) time and O(1) space.

**Q: When would you choose `deque` over a list?**
A: When you need fast operations at *both* ends. `deque.appendleft()`/`popleft()` are O(1); a list's `insert(0, x)`/`pop(0)` are O(n) because they shift elements. Deques are ideal for queues, sliding windows, and BFS.

**Q: How does Python sorting differ from a comparator-based sort?**
A: Python uses a *key function* — `sorted(data, key=f)` maps each element to a comparable value and sorts by that. It's clearer and faster than pairwise comparators because each key is computed once. Multi-level sorting is done by returning a tuple from the key. Timsort is stable, so equal keys keep their relative order.

**Q: What does the `@functools.wraps` decorator do?**
A: Inside a custom decorator, it copies the wrapped function's metadata (`__name__`, `__doc__`, `__module__`, signature) onto the wrapper. Without it, introspection and tools that rely on function names break — e.g. the function would report its name as `wrapper`.

**Q: How would you implement an LRU cache?**
A: With an `OrderedDict`: on `get`/`put`, `move_to_end(key)` to mark it most-recently-used; when size exceeds capacity, `popitem(last=False)` evicts the oldest. Both operations are O(1). For function results specifically, `@functools.lru_cache` does it for free.

**Q: What's the time complexity of `in` for a list vs a set vs a dict?**
A: O(n) for a list (linear scan), O(1) average for a set and a dict (hash lookup). For repeated membership tests, convert your list to a set first.

**Q: How do you validate and parse incoming request data in modern Python?**
A: With Pydantic v2: define a `BaseModel` with typed fields, use `Field(...)` for constraints (`min_length`, `ge`, `le`), `field_validator` for custom rules, and `EmailStr`/other types for format checks. Pydantic coerces compatible types, raises `ValidationError` listing all failures, and you serialize with `model_dump()`. FastAPI uses this automatically.

**Q: What's wrong with `def add_item(item, items=[])`?**
A: The default list is created once at function definition and shared across all calls, so items accumulate between calls — a notorious bug. Use `items=None` and set `items = items or []` inside. The same applies to dicts and any mutable default.

## Practice exercises

Try each yourself before opening the solutions.

1. **First non-repeating character.** Given a string, return the first character that appears exactly once, or `None`. Example: `"aabbcdd"` → `"c"`.
2. **Merge two sorted lists** into one sorted list without using `sorted()` (mimic the merge step of merge sort).
3. **Word frequency report.** Given a paragraph string, return the top 3 most common words (case-insensitive, ignore punctuation).
4. **Validate a config dict** with Pydantic: a `port` (int, 1–65535), a `host` (non-empty str), and a `debug` flag (bool, default False). Reject invalid input.
5. **Retry decorator.** Write `@retry(times=3)` that re-calls the function on exception up to `times` attempts, then re-raises the last error.
6. **Chunk a list** into sublists of size `n`: `chunk([1,2,3,4,5], 2)` → `[[1,2],[3,4],[5]]`.

<details><summary>Solutions</summary>

```python
# 1. First non-repeating character
from collections import Counter

def first_unique(s: str) -> str | None:
    counts = Counter(s)
    for ch in s:                 # dict preserves insertion order, so this is "first"
        if counts[ch] == 1:
            return ch
    return None

print(first_unique("aabbcdd"))   # => c


# 2. Merge two sorted lists
def merge_sorted(a: list[int], b: list[int]) -> list[int]:
    result, i, j = [], 0, 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            result.append(a[i]); i += 1
        else:
            result.append(b[j]); j += 1
    result.extend(a[i:])         # append whatever remains from either list
    result.extend(b[j:])
    return result

print(merge_sorted([1, 4, 6], [2, 3, 5]))   # => [1, 2, 3, 4, 5, 6]


# 3. Word frequency report
import re
from collections import Counter

def top_words(text: str, k: int = 3) -> list[str]:
    words = re.findall(r"[a-z']+", text.lower())   # tokenize, drop punctuation
    return [w for w, _ in Counter(words).most_common(k)]

print(top_words("The cat sat. The cat ran. The dog sat."))
# => ['the', 'cat', 'sat']


# 4. Validate a config dict with Pydantic
from pydantic import BaseModel, Field, ValidationError

class Config(BaseModel):
    port: int = Field(ge=1, le=65535)
    host: str = Field(min_length=1)
    debug: bool = False

print(Config(port="8080", host="localhost").model_dump())
# => {'port': 8080, 'host': 'localhost', 'debug': False}

try:
    Config(port=99999, host="")
except ValidationError as e:
    print(e.error_count(), "errors")   # => 2 errors


# 5. Retry decorator (decorator factory: takes args, returns a decorator)
import functools

def retry(times: int = 3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as exc:
                    last_exc = exc
                    print(f"attempt {attempt} failed: {exc}")
            raise last_exc          # all attempts exhausted -> re-raise
        return wrapper
    return decorator

calls = {"n": 0}

@retry(times=3)
def flaky():
    calls["n"] += 1
    if calls["n"] < 3:
        raise RuntimeError("boom")
    return "ok"

print(flaky())   # => attempt 1 failed... attempt 2 failed... then 'ok'


# 6. Chunk a list
def chunk(lst: list, n: int) -> list[list]:
    return [lst[i:i + n] for i in range(0, len(lst), n)]

print(chunk([1, 2, 3, 4, 5], 2))   # => [[1, 2], [3, 4], [5]]
```

</details>

## Cheat sheet

| Task | Idiom |
|---|---|
| Reverse list/string | `x[::-1]` |
| Frequency map | `Counter(iterable)` |
| Top-k frequent | `Counter(x).most_common(k)` |
| Group by key | `defaultdict(list)` + append |
| Two-sum / fast lookup | dict of `value -> index`, check complement |
| Fast both-ends queue | `collections.deque` (`append`/`popleft`) |
| Memoize a function | `@functools.lru_cache(maxsize=None)` |
| Sort by field | `sorted(x, key=lambda i: i["f"])` |
| Multi-key sort | `key=lambda i: (i.a, i.b)` |
| Binary search | `bisect.bisect_left(arr, target)` |
| Parse / dump JSON | `json.loads(s)` / `json.dumps(obj)` |
| Read file safely | `with open(path) as f: for line in f:` |
| Validate input | Pydantic `BaseModel` + `model_dump()` |
| Custom decorator skeleton | `@functools.wraps(func)` + `*args, **kwargs` + `return` |
| Empty-default arg | `def f(x=None): x = x or []` |

**Stdlib shortlist to memorize:** `collections.{Counter, defaultdict, deque, OrderedDict}` · `functools.{lru_cache, wraps, reduce}` · `itertools.{chain, groupby, product}` · `operator.itemgetter` · `bisect` · `json` · `re`.

**Closing advice:** in the room, narrate continuously, state complexity after each solution, and reach for the stdlib. A clean 8-line `Counter` solution with a clear explanation beats a clever 30-line one every time.
