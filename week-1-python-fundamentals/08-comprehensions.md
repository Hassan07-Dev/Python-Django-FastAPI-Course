# 08 — Comprehensions & Generator Expressions

> Week 1 · Python Fundamentals

## What you will learn

- The mental model for a **list comprehension** as a fused `map` + `filter`.
- How to add a **filter** (`if` at the end) versus a **conditional expression** (`if/else` in the output part) — and why these live in different places.
- **Nested loops** inside one comprehension vs **nested comprehensions** (e.g. building and flattening matrices).
- **Dict comprehensions** and **set comprehensions**.
- **Generator expressions**: lazy, memory-efficient iteration, and exactly how they differ from list comprehensions.
- When a comprehension *hurts* readability and you should fall back to a plain loop.
- Performance: why comprehensions are usually faster than an equivalent `for`-loop-with-`append`.
- The handful of comprehension transformations that show up again and again in interviews.

## PHP/Laravel bridge

In Laravel you almost never write a manual `foreach` to transform a collection — you reach for `collect($items)->map(...)->filter(...)->values()`. A Python **list comprehension** is the language-level equivalent of that fluent chain, except it is *syntax*, not a method call: `[transform(x) for x in items if keep(x)]` reads like `->map(transform)->filter(keep)`. The big difference: Laravel collections are **eager** (each `map`/`filter` builds a new array immediately), whereas Python also gives you **generator expressions** — the lazy cousin, closer to a PHP `Generator` / `yield` or `LazyCollection`, that computes values one at a time and never materializes the whole list. Where Laravel has `mapWithKeys` for associative arrays, Python has the **dict comprehension**. Keep that mapping in your head and the syntax below will feel familiar.

## Core concept: the list comprehension mental model

A list comprehension builds a new list by looping over an iterable. The canonical form:

```python
result = [expression for item in iterable]
```

Read it right-to-left from the `for`: "for each `item` in `iterable`, evaluate `expression`, collect the results into a list."

Compare the loop you would otherwise write:

```python
# The explicit loop
squares = []
for n in range(5):
    squares.append(n * n)
print(squares)  # => [0, 1, 4, 9, 16]

# The comprehension — same result, one line
squares = [n * n for n in range(5)]
print(squares)  # => [0, 1, 4, 9, 16]
```

The three moving parts, named:

| Part | In `[n * n for n in range(5)]` | Laravel analogue |
|------|-------------------------------|------------------|
| **output expression** | `n * n` | the closure body in `->map(...)` |
| **loop variable** | `n` | `$n` inside the closure |
| **iterable** | `range(5)` | the collection you call `->map()` on |

**Why use it?** Three reasons: it is shorter, it signals intent ("I am building a list by transforming another iterable"), and it is *faster* than the `append` loop (more on that later). The loop variable is scoped to the comprehension — `n` does **not** leak into the surrounding namespace in Python 3 (it did in Python 2; do not let an old tutorial confuse you).

## Core concept: adding a filter (the trailing `if`)

Append `if condition` to keep only some items. This is your `->filter(...)`:

```python
# Keep even numbers only
evens = [n for n in range(10) if n % 2 == 0]
print(evens)  # => [0, 2, 4, 6, 8]

# Transform AND filter in one pass: squares of odd numbers
odd_squares = [n * n for n in range(10) if n % 2 == 1]
print(odd_squares)  # => [1, 9, 25, 49, 81]
```

The trailing `if` is a **filter**: when the condition is false the item is *skipped entirely* — nothing is added to the list. You can stack multiple filters; they act like a logical AND:

```python
# Numbers divisible by both 2 and 3
nums = [n for n in range(30) if n % 2 == 0 if n % 3 == 0]
print(nums)  # => [0, 6, 12, 18, 24]

# Equivalent, often clearer, with a single combined condition
nums = [n for n in range(30) if n % 2 == 0 and n % 3 == 0]
print(nums)  # => [0, 6, 12, 18, 24]
```

## Core concept: conditional expressions in the output (`if/else`)

A trailing `if` *removes* items. But sometimes you want to keep every item and choose *what value to produce*. For that you put a **conditional expression** (Python's ternary) in the **output** position, before the `for`:

```python
# Label every number — no items dropped
labels = ["even" if n % 2 == 0 else "odd" for n in range(5)]
print(labels)  # => ['even', 'odd', 'even', 'odd', 'even']
```

This is the single most common point of confusion. Memorize the rule:

| You want to... | Put the `if`... | Form |
|----------------|-----------------|------|
| **drop** some items (filter) | **after** the `for` | `[x for x in xs if cond]` |
| **choose a value** for every item | **before** the `for` (as a ternary) | `[a if cond else b for x in xs]` |

The reason is grammatical: `a if cond else b` is a complete *expression* that evaluates to a single value, so it sits in the output slot. A bare trailing `if cond` is a *filter clause* and has no `else`. In fact, `if/else` is **required** to be complete in the output position — you cannot write `[x if cond for x in xs]` (syntax error), because the output expression must always produce a value.

You can combine both: filter first, then choose a value among the survivors:

```python
prices = [10, -5, 25, 0, -3, 40]
# Drop non-positive prices, then tag the rest as 'cheap' or 'pricey'
tags = ["cheap" if p < 20 else "pricey" for p in prices if p > 0]
print(tags)  # => ['cheap', 'pricey', 'pricey']
```

Order of clauses left-to-right: `output_expression` · `for` · `if-filter`.

## Core concept: nested loops in one comprehension

You can chain multiple `for` clauses. They nest **left-to-right, outermost first** — the same order you would write the equivalent nested `for` loops top-to-bottom:

```python
# Cartesian product of suits and ranks
suits = ["H", "D"]
ranks = ["A", "K", "Q"]
deck = [f"{rank}{suit}" for suit in suits for rank in ranks]
print(deck)
# => ['AH', 'KH', 'QH', 'AD', 'KD', 'QD']

# Read it as:
# for suit in suits:        <- first for = outer loop
#     for rank in ranks:    <- second for = inner loop
#         deck.append(...)
```

A later `for` can reference variables from an earlier one — this is how you flatten a list of lists:

```python
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [value for row in matrix for value in row]
print(flat)  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

Again, read the `for` clauses in the order they appear: "for each `row` in `matrix`, for each `value` in `row`, collect `value`."

## Core concept: nested comprehensions (building structures)

A nested *comprehension* is different from nested *loops*: here the **output expression is itself a comprehension**. Use this to build matrices and other nested structures:

```python
# Build a 3x4 grid of zeros
grid = [[0 for _ in range(4)] for _ in range(3)]
print(grid)  # => [[0, 0, 0, 0], [0, 0, 0, 0], [0, 0, 0, 0]]
```

> Gotcha: this is the *correct* way to build a grid of mutable rows. The tempting `[[0] * 4] * 3` creates three references to the **same** inner list, so `grid[0][0] = 9` would change all three rows. The comprehension builds a fresh inner list each iteration.

Transposing a matrix is the classic nested-comprehension interview question:

```python
matrix = [[1, 2, 3],
          [4, 5, 6]]
# Swap rows and columns
transposed = [[row[i] for row in matrix] for i in range(len(matrix[0]))]
print(transposed)  # => [[1, 4], [2, 5], [3, 6]]
```

Read the **inner** comprehension first (`[row[i] for row in matrix]` builds one output column), then the outer one (`for i in range(...)` repeats it per column index). When you nest, the inner comprehension's `for` is *physically inside* the brackets; contrast that with the flatten example above where both `for`s sit at the same level inside one set of brackets.

## Core concept: dict comprehensions

Same idea, but produce a dictionary. Syntax uses braces and a `key: value` output:

```python
# {key: value for item in iterable}
squares = {n: n * n for n in range(5)}
print(squares)  # => {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

Extremely common real-world uses:

```python
# Build a lookup table (id -> object) — like Laravel's ->keyBy('id')
users = [{"id": 1, "name": "Ada"}, {"id": 2, "name": "Linus"}]
by_id = {u["id"]: u["name"] for u in users}
print(by_id)  # => {1: 'Ada', 2: 'Linus'}

# Invert a dict (swap keys and values)
codes = {"US": 1, "UK": 44, "PK": 92}
by_code = {v: k for k, v in codes.items()}
print(by_code)  # => {1: 'US', 44: 'UK', 92: 'PK'}

# Filter a dict: keep only entries above a threshold
scores = {"alice": 90, "bob": 55, "carol": 78}
passed = {name: score for name, score in scores.items() if score >= 60}
print(passed)  # => {'alice': 90, 'carol': 78}
```

Note `.items()` yields `(key, value)` tuples, which you unpack into two loop variables. If later keys collide, the last one wins — just like assigning to the same dict key twice.

## Core concept: set comprehensions

Braces but only an expression (no colon) gives a **set** — automatically deduplicated:

```python
words = ["Apple", "banana", "apple", "BANANA", "cherry"]
# Distinct lowercase words
distinct = {w.lower() for w in words}
print(distinct)  # => {'apple', 'banana', 'cherry'}  (order not guaranteed)

# Distinct lengths
lengths = {len(w) for w in words}
print(lengths)  # => {5, 6}
```

Quick disambiguation of the brace forms:

| Syntax | Produces |
|--------|----------|
| `{k: v for ...}` | dict (has a colon) |
| `{x for ...}` | set (no colon) |
| `[x for ...]` | list |
| `(x for ...)` | generator (not a tuple!) — see next |

There is no "tuple comprehension"; parentheses make a generator expression instead.

## Core concept: generator expressions (lazy & memory-efficient)

Swap the square brackets for parentheses and you get a **generator expression**. It does *not* build a list. It produces a generator object that yields values **one at a time, on demand** (lazy evaluation):

```python
gen = (n * n for n in range(5))
print(gen)             # => <generator object <genexpr> at 0x...>
print(next(gen))       # => 0
print(next(gen))       # => 1
print(list(gen))       # => [4, 9, 16]   (consumes the rest)
```

Key properties that make generators worth knowing:

**1. Memory.** A list comprehension over a billion items allocates a billion-element list. A generator holds essentially nothing — it computes each value as you ask for it.

```python
import sys
list_comp = [n for n in range(10_000)]
gen_exp   = (n for n in range(10_000))
print(sys.getsizeof(list_comp))  # => ~85176 bytes (the whole list)
print(sys.getsizeof(gen_exp))    # => ~200 bytes (just the generator)
```

**2. They are single-use / exhaustible.** Once iterated, a generator is empty. A list can be iterated again and again:

```python
gen = (n for n in range(3))
print(list(gen))  # => [0, 1, 2]
print(list(gen))  # => []   <- already exhausted!
```

**3. Pass directly to consuming functions — drop the extra parentheses.** When a generator expression is the *sole* argument to a function, you do not need a second pair of parens. This is the idiomatic way to aggregate without building a throwaway list:

```python
# sum/any/all/max/min consume lazily — no intermediate list created
total = sum(n * n for n in range(1000))
print(total)  # => 332833500

any_negative = any(n < 0 for n in [3, -1, 4])
print(any_negative)  # => True   (and `any` short-circuits: stops at the first True)

# `all` short-circuits at the first False
all_positive = all(n > 0 for n in [3, 1, 4])
print(all_positive)  # => True
```

Because `any`/`all` short-circuit and the generator is lazy, `any(expensive(x) for x in huge)` can stop after the first match without evaluating the rest — something a list comprehension cannot do, since it must build the entire list first.

**When to choose which:**

| Use a **list comprehension** when... | Use a **generator expression** when... |
|--------------------------------------|----------------------------------------|
| you need the result more than once | you iterate exactly once |
| you need indexing / `len()` / slicing | you only feed it to `sum`/`any`/`all`/`for`/etc. |
| the data is small | the data is large or unbounded (streaming) |
| you need a real `list` (e.g. JSON serialization) | you want to short-circuit or save memory |

## Core concept: performance vs explicit loops

Comprehensions are generally **faster** than an equivalent `for` + `append` loop, because the iteration and list-building run in optimized C internally rather than executing a Python-level `list.append` lookup and call on every step.

```python
import timeit

loop = """
result = []
for n in range(1000):
    result.append(n * 2)
"""
comp = "result = [n * 2 for n in range(1000)]"

print(timeit.timeit(loop, number=10_000))  # => ~0.45s (illustrative)
print(timeit.timeit(comp, number=10_000))  # => ~0.30s (illustrative — faster)
```

Numbers vary by machine and Python version, but the comprehension consistently wins for this transform-and-collect pattern. **Caveat:** the speedup is about list construction, not magic — if the work *per item* is expensive (a network call, heavy computation), the loop body dominates and the comprehension vs. loop choice barely matters. And do not contort logic into an unreadable comprehension purely for a few microseconds; readability usually wins.

## Core concept: when a comprehension hurts readability

Comprehensions are for *building a collection from a transform/filter*. They are the wrong tool when:

- **You only want side effects** (printing, calling an API, mutating). Never write `[print(x) for x in xs]` — it builds a junk list of `None`s purely for the side effect. Use a real `for` loop.
- **There are multiple/complex conditions or try/except.** Comprehensions cannot contain statements (`try`, `break`, `continue`, assignments-as-statements). If you need them, use a loop.
- **It spans several lines and needs mental re-parsing.** If a reader has to stop and decode it, a plain loop with a descriptive variable name is clearer.

```python
# BAD: comprehension purely for side effects
[print(user["name"]) for user in users]   # builds [None, None] for nothing

# GOOD: a loop says "I'm doing something to each item"
for user in users:
    print(user["name"])

# BAD: deeply nested, hard to read
result = [f(x, y, z) for x in xs if p(x) for y in ys if q(x, y) for z in zs if r(z)]

# GOOD: the same logic as a loop is easier to follow and debug
result = []
for x in xs:
    if not p(x):
        continue
    for y in ys:
        if not q(x, y):
            continue
        for z in zs:
            if r(z):
                result.append(f(x, y, z))
```

Rule of thumb: **two `for` clauses or one filter is fine; three levels of nesting or a tangle of conditions means write the loop.**

## Core concept: the walrus operator in comprehensions (3.8+)

Sometimes you compute an expensive value, want to filter on it, *and* keep it. The walrus operator `:=` lets you assign inside the comprehension so you compute once:

```python
def process(x):
    return x * x - 4   # pretend this is expensive

data = [1, 2, 3, 4, 5]
# Keep results that are positive — without calling process() twice
results = [y for x in data if (y := process(x)) > 0]
print(results)  # => [5, 12, 21]   (process(1)=-3, process(2)=0 dropped; rest kept)
```

Without the walrus you would either call `process(x)` twice (in the filter and the output) or fall back to a loop. Use it sparingly — it is powerful but easy to overuse.

## Common mistakes & gotchas

- **Putting the filter `if` in the wrong place.** `[x for x in xs if cond]` filters; `[a if cond else b for x in xs]` chooses a value. A trailing `if` must have **no** `else`; an output `if` must have an `else`.
- **Expecting a "tuple comprehension."** `(x for x in xs)` is a **generator**, not a tuple. Use `tuple(x for x in xs)` if you really want a tuple.
- **Reusing an exhausted generator.** After you iterate a generator once it is empty. If you need the values twice, build a list.
- **`[[0] * n] * m` for a grid.** All rows share one list object. Use `[[0] * n for _ in range(m)]`.
- **Comprehensions for side effects.** `[do(x) for x in xs]` allocates a useless list. Use a `for` loop.
- **Leaking variables (myth).** In Python 3 the loop variable does **not** escape the comprehension's scope — do not rely on it being defined afterward; it is not.
- **Misreading nested `for` order.** `for a in A for b in B` means A is the outer loop, B the inner — left-to-right, same as written-out nested loops.
- **Calling an expensive function twice** in the filter and the output. Use a helper variable via the walrus operator, or a loop.
- **Order of clauses.** The output expression always comes first, then `for`, then `if`. `[x if cond for x in xs]` (filter without else, in output position) is a `SyntaxError`.

## Best practices

- Prefer a comprehension over a manual `append` loop for the simple **transform/filter into a new collection** pattern — it is faster and clearer.
- Use a **generator expression** when feeding `sum`, `any`, `all`, `max`, `min`, `join`, or any single-pass consumer — you avoid building a throwaway list. Drop the redundant parentheses when it is the only argument.
- Reach for **dict / set comprehensions** for lookup tables, inversions, and dedup instead of building an empty collection and filling it in a loop.
- Keep comprehensions **single-line-readable**. One `for`, optionally one `if`, is the sweet spot. Past that, write a loop.
- Never use a comprehension purely for **side effects**.
- Use `_` as the loop variable when you do not use the value: `[0 for _ in range(n)]`.
- When in doubt about list vs generator: if you iterate once and the data could be large, generator; if you need it multiple times or need `len`/indexing, list.

## Interview Q&A

**Q: What is a list comprehension and why is it preferred over a `for` loop with `append`?**
**A:** It is a concise expression `[expr for item in iterable if cond]` that builds a new list by transforming and/or filtering an existing iterable. It is preferred because it is more declarative (it states *what* list you want), avoids the boilerplate of initializing and appending, and is typically faster since the list construction runs in optimized C rather than executing a Python-level `append` call per element.

**Q: What is the difference between putting `if` before the `for` versus after it?**
**A:** After the `for` it is a **filter** — `[x for x in xs if cond]` skips items where the condition is false. Before the `for` it is a **conditional expression (ternary)** that selects the output value for every item — `[a if cond else b for x in xs]` keeps all items but produces `a` or `b`. The trailing filter has no `else`; the output ternary requires one.

**Q: How does a generator expression differ from a list comprehension?**
**A:** Syntax: parentheses instead of brackets. Behavior: a generator is **lazy** — it yields values one at a time on demand instead of building the whole list up front. This makes it memory-efficient for large or infinite sequences and allows short-circuiting (e.g. with `any`/`all`). The trade-offs: a generator is single-use (exhausted after one iteration) and supports neither `len()` nor indexing.

**Q: When would you use a generator expression instead of a list comprehension?**
**A:** When you iterate the data exactly once and either the dataset is large (saving memory) or you can benefit from lazy/short-circuit evaluation — for example `sum(x*x for x in big)`, `any(is_bad(x) for x in items)`, or streaming over a huge file. Use a list when you need the data more than once, need indexing/`len()`, or must hand a concrete list to something like JSON serialization.

**Q: Is `(x for x in range(3))` a tuple?**
**A:** No — it is a **generator expression**. There is no tuple comprehension. To get a tuple, wrap a generator in the constructor: `tuple(x for x in range(3))`.

**Q: How do you flatten a list of lists with a comprehension?**
**A:** `[item for sublist in nested for item in sublist]`. The `for` clauses read left-to-right as outer-then-inner loops: for each `sublist` in `nested`, for each `item` in `sublist`, collect `item`.

**Q: How do you build a dictionary from two parallel lists, or invert an existing dict?**
**A:** From parallel lists: `{k: v for k, v in zip(keys, values)}`. To invert: `{v: k for k, v in d.items()}` (assuming values are unique and hashable). Both are dict comprehensions; `.items()` yields `(key, value)` tuples you unpack into two loop variables.

**Q: Why is `[[0] * 3] * 2` dangerous for building a matrix?**
**A:** The outer `* 2` copies the **reference** to the same inner list twice, so all rows are the same object — mutating one row mutates them all. Use `[[0] * 3 for _ in range(2)]`, which creates a fresh inner list each iteration.

**Q: Does the loop variable in a comprehension leak into the enclosing scope?**
**A:** Not in Python 3 — the comprehension has its own scope, so the loop variable is not defined afterward. (It did leak in Python 2; comprehensions were changed to have their own scope in Python 3.)

**Q: When should you NOT use a comprehension?**
**A:** When you only want side effects (printing, mutating, API calls) — use a `for` loop instead of building a throwaway list. Also when you need statements like `try`/`break`/`continue`, or when nesting/conditions make it unreadable. Clarity beats cleverness.

**Q: How can a generator expression short-circuit, and why can't a list comprehension?**
**A:** Because a generator is lazy, a consumer like `any()`/`all()` can stop pulling values as soon as the answer is determined, so later elements are never computed. A list comprehension eagerly builds the entire list first, so every element is evaluated regardless — there is nothing to short-circuit.

**Q: What does the walrus operator do inside a comprehension?**
**A:** `:=` assigns a value to a name as part of an expression, letting you compute something once and reuse it in both the filter and the output — e.g. `[y for x in xs if (y := f(x)) > 0]` avoids calling `f(x)` twice.

## Practice exercises

Try each before opening the solutions.

1. **Filter + transform:** Given `nums = range(1, 21)`, build a list of the **cubes** of the numbers that are multiples of 3. Expected: `[27, 216, 729, 1728, 3375, 5832]`.

2. **Conditional output:** Given `temps = [12, 35, 28, 5, 40, 19]`, produce a list labelling each as `"hot"` if `>= 30` else `"mild"`. Do not drop any items.

3. **Dict comprehension / inversion:** Given `roles = {"alice": "admin", "bob": "user", "carol": "admin"}`, build a dict mapping each role to a list of the users who have it. (Hint: a comprehension over the distinct roles, with a nested comprehension as the value.)

4. **Flatten + dedup:** Given `pages = [[1, 2, 2], [3, 1], [4, 4, 5]]`, produce the set of all distinct numbers across every page using a single set comprehension.

5. **Generator + aggregate:** Without building an intermediate list, compute the sum of squares of the even numbers from 1 to 100 using a generator expression passed to `sum`.

6. **Matrix transpose:** Given `m = [[1, 2, 3], [4, 5, 6]]`, transpose it to `[[1, 4], [2, 5], [3, 6]]` using a nested comprehension.

<details><summary>Solutions</summary>

```python
# 1. Filter + transform
nums = range(1, 21)
result = [n ** 3 for n in nums if n % 3 == 0]
print(result)  # => [27, 216, 729, 1728, 3375, 5832]

# 2. Conditional output (ternary in the output position, no items dropped)
temps = [12, 35, 28, 5, 40, 19]
labels = ["hot" if t >= 30 else "mild" for t in temps]
print(labels)  # => ['mild', 'hot', 'mild', 'mild', 'hot', 'mild']

# 3. Role -> [users]
roles = {"alice": "admin", "bob": "user", "carol": "admin"}
by_role = {
    role: [user for user, r in roles.items() if r == role]
    for role in set(roles.values())
}
print(by_role)  # => {'admin': ['alice', 'carol'], 'user': ['bob']}  (role order not guaranteed)

# 4. Flatten + dedup with a set comprehension (two for clauses, one level of brackets)
pages = [[1, 2, 2], [3, 1], [4, 4, 5]]
distinct = {n for page in pages for n in page}
print(distinct)  # => {1, 2, 3, 4, 5}

# 5. Generator expression fed straight to sum() — no intermediate list
total = sum(n * n for n in range(1, 101) if n % 2 == 0)
print(total)  # => 171700

# 6. Transpose with a nested comprehension
m = [[1, 2, 3], [4, 5, 6]]
transposed = [[row[i] for row in m] for i in range(len(m[0]))]
print(transposed)  # => [[1, 4], [2, 5], [3, 6]]
```

</details>

## Cheat sheet

```python
# LIST: transform every item
[x * 2 for x in xs]

# LIST + FILTER: drop items where cond is False  (if AFTER for)
[x for x in xs if cond]

# LIST + TERNARY: choose value per item, drop nothing  (if/else BEFORE for)
[a if cond else b for x in xs]

# NESTED LOOPS (flatten): left-to-right = outer-to-inner
[v for row in matrix for v in row]

# NESTED COMPREHENSION (build a grid / matrix)
[[0 for _ in range(cols)] for _ in range(rows)]

# DICT comprehension (note the colon)
{k: v for k, v in pairs}
{v: k for k, v in d.items()}          # invert

# SET comprehension (braces, no colon) — auto-dedup
{x.lower() for x in words}

# GENERATOR expression (parentheses) — lazy, single-use, memory-light
(x * x for x in xs)
sum(x * x for x in xs)                # drop extra parens as sole arg
any(p(x) for x in xs)                 # short-circuits

# WALRUS (3.8+): compute once, filter and keep
[y for x in xs if (y := f(x)) > 0]
```

| Brackets | Result |
|----------|--------|
| `[...]` | list |
| `{k: v ...}` | dict |
| `{x ...}` | set |
| `(...)` | generator (NOT tuple) |

**Decision rules:** filter → `if` after `for`; pick a value → `if/else` before `for`; iterate once / big data → generator; need it twice or need `len`/indexing → list; side effects only → plain `for` loop.
