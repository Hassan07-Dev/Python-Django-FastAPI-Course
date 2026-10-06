# 07 — Dictionaries & Sets

> Week 1 · Python Fundamentals

Dictionaries are the single most important data structure in Python — JSON bodies, query params, ORM rows, config, function `**kwargs`, even objects under the hood are dicts. Sets are their quieter cousin: unordered collections of unique items, perfect for de-duplication and lightning-fast membership tests. Master both and a huge amount of "real" backend code suddenly reads like plain English.

---

## What you will learn

- Creating dicts four different ways (literal, `dict()`, from pairs, comprehension preview).
- Accessing values safely: `KeyError` vs `.get()` with defaults, plus `in`.
- Adding, updating, and deleting keys.
- The essential dict methods: `keys`, `values`, `items`, `update`, `pop`, `popitem`, `setdefault`, `clear`.
- Iterating dicts correctly (and why insertion order is now guaranteed).
- Nested dicts (the shape of real JSON) and merging dicts with `|` and `{**a, **b}`.
- What makes a value a *valid key* (hashability).
- Sets: creation, uniqueness, `add`/`discard`/`remove`, and set algebra (`|`, `&`, `-`, `^`).
- Subset/superset checks and the immutable `frozenset`.
- **Why** set/dict membership is O(1) while list membership is O(n) — a classic interview point.
- Practical patterns: dedup, fast lookups, counting, grouping.

---

## PHP/Laravel bridge

A Python **dict** is PHP's associative array — but Python *separates* the two concepts PHP fuses into one. PHP's `array` is simultaneously an ordered map *and* a numeric list; Python splits these into `dict` (key → value) and `list` (ordered sequence, Lesson 06). So `['name' => 'Sam', 'age' => 30]` becomes `{"name": "Sam", "age": 30}`. Accessing a missing key throws `KeyError` (like PHP 8's undefined-key warning, but it's a hard exception), and `$arr['x'] ?? 'default'` maps cleanly to `d.get("x", "default")`. There is **no direct PHP equivalent of a Python set** — you'd fake one with `array_unique()` or array keys; Python gives you a real `set` type with mathematical operations built in (`array_intersect` → `&`, `array_diff` → `-`, `array_merge` of unique → `|`).

---

## Core concept — Creating dictionaries

A dict maps **keys** to **values**. Keys are unique; assigning to an existing key overwrites it.

```python
# 1. Literal (most common)
user = {"name": "Sam", "age": 30, "is_admin": False}

# 2. dict() constructor with keyword args — keys must be valid identifiers, no quotes
config = dict(host="localhost", port=5432, debug=True)
# => {'host': 'localhost', 'port': 5432, 'debug': True}

# 3. From a list/iterable of (key, value) pairs
pairs = [("a", 1), ("b", 2), ("c", 3)]
from_pairs = dict(pairs)            # => {'a': 1, 'b': 2, 'c': 3}

# 4. Empty dict, then fill it
scores = {}                          # NOT set() — {} is an empty dict
scores["sam"] = 95

# 5. dict.fromkeys — same default value for many keys
flags = dict.fromkeys(["read", "write", "delete"], False)
# => {'read': False, 'write': False, 'delete': False}

# (Preview) Dict comprehension — Lesson 08 covers these fully
squares = {n: n * n for n in range(1, 4)}   # => {1: 1, 2: 4, 3: 9}
```

> ⚠️ `{}` is an **empty dict**, not an empty set. An empty set is `set()`. This trips up nearly everyone once.

---

## Core concept — Accessing values: `[]` vs `.get()`

```python
user = {"name": "Sam", "age": 30}

# Square brackets: raises KeyError if the key is missing
print(user["name"])      # => Sam
# print(user["email"])   # => KeyError: 'email'   (program crashes)

# .get(): returns None (or your default) instead of raising — safe
print(user.get("email"))            # => None
print(user.get("email", "n/a"))     # => n/a   (second arg = fallback)

# Membership test — fast and idiomatic. Tests KEYS, not values.
print("name" in user)    # => True
print("email" in user)   # => False
print("Sam" in user)     # => False  (Sam is a value, not a key!)
```

**When to use which?**

| Situation | Use | Why |
|-----------|-----|-----|
| Key *must* exist; absence is a bug | `d[key]` | Fail loud and early |
| Key is optional, you have a sensible default | `d.get(key, default)` | No exception, cleaner code |
| You only need to know if it's present | `key in d` | Clear intent, O(1) |

`.get()` is the Pythonic answer to PHP's `$arr['k'] ?? $default`. Reach for it whenever a key may legitimately be absent.

---

## Core concept — Adding, updating, deleting

```python
user = {"name": "Sam"}

# Add or update — same syntax. If the key exists, it's overwritten.
user["age"] = 30            # add    => {'name': 'Sam', 'age': 30}
user["age"] = 31            # update => {'name': 'Sam', 'age': 31}

# update() merges another dict (or kwargs) in place; existing keys overwritten
user.update({"age": 32, "city": "Lahore"})
user.update(is_admin=True)  # keyword form
# => {'name': 'Sam', 'age': 32, 'city': 'Lahore', 'is_admin': True}

# Deleting
del user["city"]            # raises KeyError if "city" is missing
removed = user.pop("age")   # remove AND return the value => 32
gone = user.pop("nope", None)   # safe: default avoids KeyError => None
```

---

## Core concept — The essential dict methods

```python
inv = {"apples": 5, "pears": 2, "plums": 8}

# Views — live, read-only windows onto the dict (they update if the dict changes)
inv.keys()      # => dict_keys(['apples', 'pears', 'plums'])
inv.values()    # => dict_values([5, 2, 8])
inv.items()     # => dict_items([('apples', 5), ('pears', 2), ('plums', 8)])

# Need a real list? Wrap the view:
list(inv.keys())    # => ['apples', 'pears', 'plums']

# pop(key[, default]) — remove & return a value
inv.pop("pears")            # => 2
inv.pop("missing", 0)       # => 0   (no error thanks to default)

# popitem() — remove & return the LAST inserted (key, value) pair (LIFO)
inv.popitem()               # => ('plums', 8)

# setdefault(key, default) — get key if present; else insert it with default and return that
counts = {}
counts.setdefault("a", 0)   # 'a' absent → inserts 'a': 0, returns 0
counts["a"] += 1            # now safe to increment => {'a': 1}

# clear() — empty the dict in place
inv.clear()                 # => {}
```

`setdefault` is the classic one-liner for "get-or-create-then-use" — great for grouping (shown below). For pure counting, `collections.Counter` (Lesson 19) is even cleaner, but `setdefault` is the building block to understand first.

---

## Core concept — Iterating dictionaries

Looping a dict gives you its **keys** by default. Be explicit about what you want.

```python
prices = {"coffee": 3.5, "tea": 2.0, "juice": 4.0}

# Default loop = keys
for name in prices:
    print(name)             # => coffee, tea, juice

# Values only
for price in prices.values():
    print(price)            # => 3.5, 2.0, 4.0

# Keys AND values together — the idiomatic, readable way
for name, price in prices.items():
    print(f"{name}: ${price:.2f}")
# => coffee: $3.50
# => tea: $2.00
# => juice: $4.00
```

### Insertion ordering is guaranteed (Python 3.7+)

Since **Python 3.7** (and as a CPython implementation detail in 3.6), dicts **preserve insertion order**. Iteration yields keys in the order they were first added. Before 3.7 you needed `collections.OrderedDict`; today a plain dict is ordered, so you almost never need `OrderedDict` anymore.

```python
d = {}
d["z"] = 1
d["a"] = 2
d["m"] = 3
print(list(d))   # => ['z', 'a', 'm']   (insertion order, NOT sorted)

# Updating an existing key does NOT change its position
d["z"] = 99
print(list(d))   # => ['z', 'a', 'm']   (z stayed first)
```

> Interview nuance: ordering is a **language guarantee** as of 3.7, not just a CPython quirk. If you need *sorted* output, sort explicitly: `dict(sorted(d.items()))`.

---

## Core concept — Nested dicts (the shape of real JSON)

API request/response bodies are nested dicts and lists. Get comfortable drilling into them — and guarding the parts that might be missing.

```python
order = {
    "id": 1001,
    "customer": {"name": "Sam", "email": "sam@example.com"},
    "items": [
        {"sku": "A1", "qty": 2, "price": 9.99},
        {"sku": "B7", "qty": 1, "price": 19.50},
    ],
    "shipping": {"city": "Lahore", "country": "PK"},
}

print(order["customer"]["name"])         # => Sam
print(order["items"][0]["sku"])          # => A1

# Compute a total by walking the nested list of dicts
total = sum(item["qty"] * item["price"] for item in order["items"])
print(round(total, 2))                   # => 39.48

# Safe deep access — chain .get() so a missing branch yields a default, not a crash
zip_code = order.get("shipping", {}).get("zip", "unknown")
print(zip_code)                          # => unknown
```

The `order.get("shipping", {}).get("zip", "unknown")` pattern is worth memorizing: each `.get(..., {})` returns an empty dict when a branch is missing, so the next `.get()` never explodes with a `KeyError` or `AttributeError`.

---

## Core concept — Merging dicts: `|` and `{**a, **b}`

```python
defaults = {"theme": "light", "lang": "en", "page_size": 20}
overrides = {"theme": "dark", "page_size": 50}

# 1. Merge operator | (Python 3.9+) — returns a NEW dict; right side wins on conflicts
merged = defaults | overrides
# => {'theme': 'dark', 'lang': 'en', 'page_size': 50}

# 2. Dict-unpacking — works on all modern Python, same "right wins" rule
merged2 = {**defaults, **overrides}
# => {'theme': 'dark', 'lang': 'en', 'page_size': 50}

# 3. In-place merge with |= (mutates the left dict) — like update()
settings = dict(defaults)
settings |= overrides       # settings now reflects the overrides

# Unpacking also lets you add/override inline
final = {**defaults, "lang": "ur"}   # tweak one key while copying the rest
```

`defaults` is untouched by `|` and `{**...}` — both build a fresh dict, which is exactly what you want for "config + overrides" without clobbering the original. Remember: **on a key clash, the right-hand dict wins.**

---

## Core concept — Valid keys must be hashable

A dict key (and a set member) must be **hashable**: its hash must never change over its lifetime. In practice that means **immutable** types.

```python
ok = {
    "string": 1,          # str  — hashable
    42: 2,                # int  — hashable
    3.14: 3,              # float — hashable
    (1, 2): 4,            # tuple of hashables — hashable
    True: 5,              # bool — hashable
    None: 6,              # None — hashable
}

# Lists, dicts, and sets are MUTABLE → unhashable → cannot be keys
# bad = {[1, 2]: "x"}    # => TypeError: unhashable type: 'list'
# bad = {{1}: "x"}       # => TypeError: unhashable type: 'set'

# A tuple is hashable ONLY if everything inside it is hashable
# bad = {(1, [2]): "x"}  # => TypeError: unhashable type: 'list'
```

**Why this rule?** Dicts and sets use a **hash table**. They compute `hash(key)` to decide where to store the entry. If a key's hash could change after insertion, the table could never find it again. Immutability guarantees a stable hash. (This is also why tuples can be keys but lists cannot.)

> Gotcha: `True == 1` and `hash(True) == hash(1)` in Python, so `{1: "a", True: "b"}` collapses to a single key `1` with value `"b"`. Rarely matters, but interviewers love it.

---

## Core concept — Sets: unordered collections of unique items

A `set` holds **unique**, **unordered**, **hashable** items. Duplicates vanish automatically.

```python
# Literal (note: braces, but NO key:value pairs)
colors = {"red", "green", "blue", "red"}
print(colors)            # => {'red', 'green', 'blue'}  (dup removed; order not guaranteed)

# From any iterable — the classic dedup trick
nums = set([1, 2, 2, 3, 3, 3])
print(nums)              # => {1, 2, 3}

# EMPTY set — must use set(); {} is an empty dict!
empty = set()
print(type(empty))       # => <class 'set'>
print(type({}))          # => <class 'dict'>

# Mutating
colors.add("yellow")     # add one item (no-op if already present)
colors.discard("pink")   # remove if present — NO error if absent
colors.remove("red")     # remove — raises KeyError if absent
popped = colors.pop()    # remove & return an arbitrary item

# Membership — fast (O(1) average), the whole point of sets
print("blue" in colors)  # => True
```

Sets are **unordered** — never rely on iteration order, and you can't index a set (`colors[0]` is a `TypeError`).

---

## Core concept — Set algebra

This is where sets shine. Each operator returns a **new** set.

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a | b    # union: in a OR b            => {1, 2, 3, 4, 5, 6}
a & b    # intersection: in a AND b    => {3, 4}
a - b    # difference: in a NOT in b   => {1, 2}
b - a    # difference the other way    => {5, 6}
a ^ b    # symmetric difference:        => {1, 2, 5, 6}
         #   in a or b but NOT both

# Each has a method form that accepts any iterable, not just a set:
a.union([5, 6])              # => {1, 2, 3, 4, 5, 6}
a.intersection([3, 4, 99])  # => {3, 4}
a.difference([1])           # => {2, 3, 4}

# In-place variants mutate the left set:
a |= b   # update (union)
a &= b   # intersection_update
a -= b   # difference_update
a ^= b   # symmetric_difference_update
```

### Subset and superset

```python
small = {1, 2}
big = {1, 2, 3, 4}

small <= big    # subset?            => True
small < big     # PROPER subset (≠)  => True
big >= small    # superset?          => True
small.isdisjoint({9, 10})   # no common elements? => True
```

| Operator | Method | Meaning |
|----------|--------|---------|
| `\|` | `union` | items in either set |
| `&` | `intersection` | items in both sets |
| `-` | `difference` | items in left, not right |
| `^` | `symmetric_difference` | items in exactly one set |
| `<=` | `issubset` | left ⊆ right |
| `>=` | `issuperset` | left ⊇ right |

A practical use: comparing requested fields against allowed fields.

```python
requested = {"name", "email", "password_hash"}
allowed = {"name", "email", "age"}

forbidden = requested - allowed     # => {'password_hash'}
if forbidden:
    raise ValueError(f"Not allowed: {forbidden}")
```

---

## Core concept — `frozenset`: the immutable set

A `frozenset` is a set you cannot change after creation. Because it's immutable, it's **hashable** — so it can be a dict key or an element of another set.

```python
fs = frozenset([1, 2, 3])
# fs.add(4)   # => AttributeError: 'frozenset' object has no attribute 'add'

# It supports all the read-only set algebra:
print(fs & {2, 3, 4})   # => frozenset({2, 3})

# The payoff: frozensets can live INSIDE a set, or be DICT KEYS
groups = {frozenset({"a", "b"}), frozenset({"c"})}   # set of sets — legal
permissions = {frozenset({"read", "write"}): "editor"}   # frozenset as a key
```

Think of `frozenset` as "the tuple of sets": same data, locked down, hashable.

---

## Core concept — Why sets/dicts beat lists for lookups (the O(1) story)

This is one of the most common Python interview questions. Checking `x in some_list` scans element by element — **O(n)**, linear. Checking `x in some_set` (or dict) hashes `x` and jumps straight to the bucket — **O(1)** on average, constant time regardless of size.

```python
import timeit

big_list = list(range(1_000_000))
big_set = set(big_list)

target = 999_999   # worst case for the list: it's at the very end

list_time = timeit.timeit(lambda: target in big_list, number=100)
set_time = timeit.timeit(lambda: target in big_set, number=100)

print(f"list: {list_time:.4f}s")   # => list: ~1.5s   (slow — scans a million items)
print(f"set:  {set_time:.6f}s")    # => set:  ~0.000012s (instant — hash lookup)
# The set is thousands of times faster.
```

**Rule of thumb:** if you'll repeatedly ask *"is this in the collection?"*, store it in a `set` (or use a `dict` if you also need an associated value). Building the set is a one-time O(n) cost that pays off on every lookup.

```python
# Anti-pattern: O(n*m) — for each user, scan the whole banned list
banned_list = ["u4", "u8", "u15"]
def is_banned_slow(uid): return uid in banned_list   # O(n) each call

# Better: O(1) per check after a one-time O(n) build
banned = set(banned_list)
def is_banned_fast(uid): return uid in banned        # O(1) each call
```

---

## Core concept — Practical patterns

### Deduplicate while you can sacrifice order

```python
emails = ["a@x.com", "b@x.com", "a@x.com", "c@x.com", "b@x.com"]
unique = list(set(emails))      # order NOT preserved
print(len(unique))              # => 3
```

### Deduplicate **and keep first-seen order** (dict trick)

```python
emails = ["a@x.com", "b@x.com", "a@x.com", "c@x.com"]
unique_ordered = list(dict.fromkeys(emails))
# => ['a@x.com', 'b@x.com', 'c@x.com']   (dedup + insertion order kept)
```

### Count occurrences with `setdefault`

```python
words = "the cat sat on the mat the".split()
counts = {}
for w in words:
    counts[w] = counts.get(w, 0) + 1
print(counts)   # => {'the': 3, 'cat': 1, 'sat': 1, 'on': 1, 'mat': 1}
```

### Group items by a key (build a dict of lists)

```python
people = [
    {"name": "Sam", "dept": "eng"},
    {"name": "Ada", "dept": "eng"},
    {"name": "Lee", "dept": "sales"},
]
by_dept = {}
for p in people:
    by_dept.setdefault(p["dept"], []).append(p["name"])
print(by_dept)   # => {'eng': ['Sam', 'Ada'], 'sales': ['Lee']}
```

### Find common / missing items with set math

```python
db_ids = {1, 2, 3, 4, 5}
request_ids = {3, 4, 5, 6, 7}

print(db_ids & request_ids)   # exist in both      => {3, 4, 5}
print(request_ids - db_ids)   # requested but missing => {6, 7}
```

---

## Common mistakes & gotchas

- **`{}` is an empty dict, not a set.** Use `set()` for an empty set.
- **`d[missing_key]` raises `KeyError`** and crashes. Use `.get(key, default)` when absence is normal.
- **`in` tests dict *keys*, not values.** `"Sam" in user` is `False` if `Sam` is a value. To test values: `"Sam" in user.values()` (but that's O(n)).
- **Lists/dicts/sets can't be dict keys or set members** — `TypeError: unhashable type`. Use a `tuple` or `frozenset` instead.
- **Don't rely on set order.** Sets are unordered; iteration order is arbitrary and may vary.
- **Mutating a dict while iterating it** raises `RuntimeError: dictionary changed size during iteration`. Iterate over a copy: `for k in list(d): ...`, or build a new dict.
- **`dict.keys()`/`.values()`/`.items()` return views, not lists.** They have no index (`d.keys()[0]` fails) and reflect later changes. Wrap in `list()` if you need a snapshot or indexing.
- **`copy` is shallow.** `d2 = d.copy()` (or `dict(d)`) copies the top level only; nested dicts/lists are shared. Use `copy.deepcopy(d)` for a fully independent copy.
- **`True`/`1` and `False`/`0` collide as keys** because they're equal and share a hash — `{1: "a", True: "b"}` keeps only one entry.
- **`pop()` on a dict needs a key; `pop()` on a set takes none** (it removes an arbitrary element). Different APIs.

---

## Best practices

- Prefer `.get(key, default)` over `try/except KeyError` for optional keys — it's clearer and faster for the common case.
- Use `for k, v in d.items()` to iterate keys and values together; it reads better than indexing inside the loop.
- Use a **set** for membership tests and dedup; use a **dict** when each item needs an associated value.
- Convert a list to a set **once** before a loop of `in` checks — don't pay O(n) per check.
- Use `dict.fromkeys(seq)` to dedup while preserving order; use `set(seq)` when order doesn't matter.
- Merge config with `base | overrides` (or `{**base, **overrides}`) — non-destructive and explicit about precedence.
- Reach for `collections.defaultdict` and `collections.Counter` (Lesson 19) once `setdefault`/manual counting feel verbose — but understand the manual versions first.
- Use `frozenset` when you need a set as a dict key, set member, or a guaranteed-immutable constant.
- Don't reinvent `OrderedDict` — a plain dict has been insertion-ordered since 3.7.

---

## Interview Q&A

**Q: What's the difference between `d[key]` and `d.get(key)`?**
**A:** `d[key]` returns the value but raises `KeyError` if the key is absent. `d.get(key)` returns `None` (or a supplied default, `d.get(key, default)`) instead of raising. Use `[]` when the key must exist (absence is a bug) and `.get()` when the key is optional.

**Q: Are Python dictionaries ordered?**
**A:** Yes. As of Python 3.7 it's a language guarantee that dicts preserve insertion order; iteration yields keys in the order they were first inserted. (In CPython 3.6 it was already true as an implementation detail.) Updating an existing key's value keeps its original position. For sorted output you must sort explicitly, e.g. `dict(sorted(d.items()))`.

**Q: Why can a tuple be a dict key but not a list?**
**A:** Dict keys must be **hashable**, which requires a stable hash for the object's lifetime — in practice, immutability. Tuples are immutable, so their hash never changes; lists are mutable, so theirs could, which would corrupt the hash table. (A tuple is only hashable if all its elements are too.)

**Q: Why is membership testing on a set faster than on a list?**
**A:** A list stores items sequentially, so `x in list` scans element by element — O(n). A set uses a hash table: it hashes `x` and jumps directly to the bucket — O(1) on average. For repeated "is it present?" checks, a set is dramatically faster on large data.

**Q: What's the difference between `{}` and `set()`?**
**A:** `{}` creates an **empty dict**, not a set — there's no literal syntax for an empty set, so you must call `set()`. A non-empty set literal uses braces with bare items (`{1, 2}`), while a dict uses `key: value` pairs (`{1: "a"}`).

**Q: How do you merge two dictionaries, and which side wins on conflicts?**
**A:** Use `a | b` (Python 3.9+) or `{**a, **b}` (all modern versions); both return a new dict. On a key collision, the **right-hand** dict wins. For in-place merging use `a.update(b)` or `a |= b`, which mutate `a`.

**Q: When would you use `setdefault`?**
**A:** When you want "get the value if the key exists, otherwise insert a default and return that" in one step — classic for grouping (`d.setdefault(key, []).append(item)`) or initializing counters. `collections.defaultdict` automates the same idea when every missing key needs the same factory.

**Q: What does `popitem()` do?**
**A:** It removes and returns the **last inserted** `(key, value)` pair as a tuple (LIFO order, since 3.7), and raises `KeyError` on an empty dict. It's handy for consuming a dict like a stack.

**Q: What is a `frozenset` and why use one?**
**A:** An immutable set. Because it can't change, it's hashable, so it can be a dict key or an element of another set (a regular set can't). It supports all the read-only set operations but none of the mutating ones (`add`, `discard`, etc.).

**Q: What's the difference between `remove()` and `discard()` on a set?**
**A:** Both delete an element, but `remove(x)` raises `KeyError` if `x` isn't present, while `discard(x)` silently does nothing. Use `discard` when absence is acceptable.

**Q: How do you deduplicate a list while preserving order?**
**A:** `list(dict.fromkeys(seq))`. Plain `list(set(seq))` removes duplicates but loses order, since sets are unordered. The dict trick works because dict keys are unique *and* insertion-ordered.

**Q: What happens if you modify a dict while iterating over it?**
**A:** Python raises `RuntimeError: dictionary changed size during iteration`. Iterate over a snapshot instead — `for k in list(d): ...` — or build a new dict from the old one.

---

## Practice exercises

Try each yourself before opening the solutions.

1. **Safe access.** Given `user = {"name": "Sam", "age": 30}`, write code that prints the user's email, falling back to `"no email on file"` if the key is absent — without using `try/except`.

2. **Word frequency.** Given the string `"red blue red green blue red"`, build a dict mapping each word to how many times it appears. Then print the most frequent word.

3. **Common tags.** Given `post_a = {"python", "web", "api"}` and `post_b = {"api", "django", "web"}`, print: the tags they share, the tags unique to `post_a`, and all tags across both posts.

4. **Merge config.** Given `defaults = {"theme": "light", "page_size": 20, "lang": "en"}` and `user_prefs = {"theme": "dark", "page_size": 50}`, produce a *new* merged dict where user prefs win — without mutating `defaults`. Confirm `defaults` is unchanged.

5. **Group by first letter.** Given `names = ["Sam", "Sara", "Ada", "Aldo", "Ben"]`, build a dict mapping each first letter to a list of names starting with it. Expected: `{"S": ["Sam", "Sara"], "A": ["Ada", "Aldo"], "B": ["Ben"]}`.

6. **Dedup, keep order.** Given `ids = [3, 1, 2, 3, 1, 4]`, produce `[3, 1, 2, 4]` (duplicates removed, first-seen order preserved).

<details><summary>Solutions</summary>

```python
# 1. Safe access
user = {"name": "Sam", "age": 30}
print(user.get("email", "no email on file"))   # => no email on file


# 2. Word frequency
text = "red blue red green blue red"
counts = {}
for word in text.split():
    counts[word] = counts.get(word, 0) + 1
print(counts)                                   # => {'red': 3, 'blue': 2, 'green': 1}
top = max(counts, key=counts.get)               # key with the largest value
print(top)                                      # => red


# 3. Common tags
post_a = {"python", "web", "api"}
post_b = {"api", "django", "web"}
print(post_a & post_b)   # shared        => {'web', 'api'}
print(post_a - post_b)   # only in a     => {'python'}
print(post_a | post_b)   # all tags      => {'python', 'web', 'api', 'django'}


# 4. Merge config (non-destructive)
defaults = {"theme": "light", "page_size": 20, "lang": "en"}
user_prefs = {"theme": "dark", "page_size": 50}
merged = defaults | user_prefs           # or {**defaults, **user_prefs}
print(merged)     # => {'theme': 'dark', 'page_size': 50, 'lang': 'en'}
print(defaults)   # => unchanged: {'theme': 'light', 'page_size': 20, 'lang': 'en'}


# 5. Group by first letter
names = ["Sam", "Sara", "Ada", "Aldo", "Ben"]
groups = {}
for name in names:
    groups.setdefault(name[0], []).append(name)
print(groups)     # => {'S': ['Sam', 'Sara'], 'A': ['Ada', 'Aldo'], 'B': ['Ben']}


# 6. Dedup, keep order
ids = [3, 1, 2, 3, 1, 4]
print(list(dict.fromkeys(ids)))   # => [3, 1, 2, 4]
```

</details>

---

## Cheat sheet

```python
# ---- DICT: create ----
d = {"k": 1}                 # literal
d = dict(k=1)                # kwargs
d = dict([("k", 1)])         # from pairs
d = dict.fromkeys(["a","b"], 0)   # same default
d = {}                       # empty dict (NOT a set)

# ---- DICT: access ----
d["k"]                       # KeyError if missing
d.get("k", default)          # safe, returns default
"k" in d                     # membership (keys), O(1)

# ---- DICT: modify ----
d["k"] = 2                   # add / update
d.update(other, x=1)         # merge in place
d.pop("k", None)             # remove+return (safe with default)
d.popitem()                  # remove+return LAST pair (LIFO)
d.setdefault("k", [])        # get-or-create
del d["k"]                   # delete (KeyError if missing)
d.clear()                    # empty it

# ---- DICT: iterate / view ----
for k in d: ...                  # keys
for v in d.values(): ...         # values
for k, v in d.items(): ...       # both  (idiomatic)
list(d.keys())                   # view -> list

# ---- DICT: merge (new dict; right wins) ----
a | b           # 3.9+
{**a, **b}      # all versions
a |= b          # in place

# ---- SET: create ----
s = {1, 2, 3}                # literal (no empty form)
s = set([1, 1, 2])           # dedup -> {1, 2}
s = set()                    # empty set

# ---- SET: modify ----
s.add(x)                     # add one
s.discard(x)                 # remove if present (no error)
s.remove(x)                  # remove (KeyError if absent)
s.pop()                      # remove+return arbitrary

# ---- SET: algebra (new set) ----
a | b   # union              a & b   # intersection
a - b   # difference         a ^ b   # symmetric difference
a <= b  # subset             a >= b  # superset
a.isdisjoint(b)              # no overlap?

frozenset([1, 2])            # immutable, hashable set

# ---- PATTERNS ----
list(set(seq))               # dedup (order lost)
list(dict.fromkeys(seq))     # dedup (order kept)
x in big_set                 # O(1)  vs  x in big_list  -> O(n)
```

| Need | Reach for | Lookup cost |
|------|-----------|-------------|
| Key → value mapping | `dict` | O(1) |
| Unique items / fast membership | `set` | O(1) |
| Ordered sequence, may repeat | `list` (Lesson 06) | O(n) |
| Immutable set (as key/member) | `frozenset` | O(1) |
| Counting / grouping (advanced) | `Counter` / `defaultdict` (Lesson 19) | O(1) |

**Remember the three "empty" traps:** `{}` = empty dict · `set()` = empty set · `[]` = empty list.
