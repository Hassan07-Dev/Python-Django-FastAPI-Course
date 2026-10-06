# 05 — Control Flow: if, loops, match-case

> Week 1 · Python Fundamentals

## What you will learn

- How `if` / `elif` / `else` work and why Python uses indentation instead of `{ }`
- The conditional (ternary) expression: `value_a if condition else value_b`
- **Truthiness** — what counts as "true" or "false" in a condition (this trips up every PHP dev)
- `for` loops as *for-each* over iterables (not the C-style counter loop you may expect)
- `range()` in depth — start, stop, step, and the famous off-by-one
- `while` loops, and the loop controls `break` and `continue`
- The unusual but useful `for/else` and `while/else`
- `pass` and the `...` ellipsis placeholder
- `enumerate()` to get index + value, and `zip()` to walk lists in parallel
- Structural pattern matching with `match` / `case` (Python's modern, powerful `switch`)
- Writing readable nested loops

## PHP/Laravel bridge

If you write PHP, you already know `if`, `foreach`, `while`, and `switch` — the *logic* transfers directly. The big adjustments are syntactic and semantic, not conceptual:

- Python has **no curly braces** and **no parentheses around conditions**. Blocks are defined by **indentation** (4 spaces by convention). A colon `:` ends the header line.
- PHP's `foreach ($items as $item)` becomes `for item in items:`. Python's `for` is *always* a for-each — there is no `for ($i = 0; $i < n; $i++)`. To get a counter you use `range()` or `enumerate()`.
- PHP's `switch`/`break` is replaced by `match`/`case` (Python 3.10+), which is far more powerful — it can destructure data, not just compare a scalar.
- Truthiness rules differ. In PHP `"0"` is falsy; in Python the string `"0"` is **truthy** (it's a non-empty string). Memorize Python's rules separately — guessing from PHP will bite you.

---

## Core concept 1 — `if` / `elif` / `else` and indentation

Python uses **indentation** to define a block. There are no braces. The header line ends with a colon, and everything indented under it belongs to that block.

```python
age = 20

if age >= 18:
    print("Adult")          # this line is inside the if
    print("Can vote")       # still inside the if (same indent)
print("Done")               # OUTSIDE the if — runs always
# => Adult
# => Can vote
# => Done
```

Note `elif` — Python's spelling of "else if". You can chain as many as you like.

```python
score = 72

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
else:
    grade = "F"

print(grade)   # => C
```

Conditions do **not** need parentheses. `if (age >= 18):` works but is non-idiomatic; write `if age >= 18:`.

### Combining conditions

Use the **words** `and`, `or`, `not` — not `&&`, `||`, `!`.

```python
temp = 25
sunny = True

if temp > 20 and sunny:
    print("Go outside")     # => Go outside

if not sunny:
    print("Stay in")        # (does not print)
```

Python supports **chained comparisons**, which read like math and have no PHP equivalent:

```python
x = 5
if 0 < x < 10:              # means (0 < x) and (x < 10)
    print("single digit positive")   # => single digit positive
```

### Indentation rules (read this twice)

- Pick **4 spaces** per level. The whole Python community uses 4 spaces.
- **Never mix tabs and spaces** — Python 3 raises `TabError` if you do. Configure your editor to insert spaces when you press Tab.
- The indentation amount must be **consistent** within a block. This is invalid:

```python
if True:
    print("a")
      print("b")   # IndentationError: unexpected indent
```

---

## Core concept 2 — The conditional (ternary) expression

PHP: `$x = $cond ? $a : $b;`
Python flips the order and reads like English: **`a if cond else b`**.

```python
age = 16
status = "adult" if age >= 18 else "minor"
print(status)   # => minor
```

It's an **expression** (it produces a value), so you can use it inline anywhere a value is expected:

```python
n = 7
print(f"{n} is {'even' if n % 2 == 0 else 'odd'}")   # => 7 is odd

prices = [10, -5, 20]
clean = [p if p >= 0 else 0 for p in prices]          # used inside a comprehension
print(clean)   # => [10, 0, 20]
```

Keep it to a single, simple choice. If you need multiple branches, use a real `if`/`elif` block — nesting ternaries hurts readability.

---

## Core concept 3 — Truthiness in conditions

A condition does not need to be a literal `True`/`False`. Python evaluates the **truthiness** of any object. This is one of Python's most loved features — and a common source of bugs for newcomers.

**Falsy values** (everything else is truthy):

| Category | Falsy examples |
| --- | --- |
| Boolean | `False` |
| None | `None` |
| Numbers equal to zero | `0`, `0.0`, `0j`, `Decimal(0)` |
| Empty sequences | `""`, `[]`, `()`, `b""` |
| Empty collections | `{}` (dict), `set()`, `range(0)` |

Everything else — non-empty strings (including `"0"` and `"false"`!), non-zero numbers, non-empty lists/dicts — is **truthy**.

```python
name = ""
if name:
    print("has name")
else:
    print("empty")     # => empty   (empty string is falsy)

items = [1, 2, 3]
if items:              # idiomatic "is this list non-empty?"
    print("not empty") # => not empty

print(bool("0"))       # => True    <-- DIFFERENT FROM PHP!
print(bool(0))         # => False
print(bool(" "))       # => True    (a space is non-empty)
print(bool([]))        # => False
```

### `== None` vs `is None`

To test for `None`, use `is`, not `==`. `is` checks **identity** (same object), which is exactly what you want for the singleton `None`.

```python
value = None
if value is None:          # correct, idiomatic
    print("no value")      # => no value

# Why not `if not value:` ?  Because 0, "", [] are ALSO falsy.
count = 0
if not count:
    print("treated as missing")   # => oops — 0 is a real value, not "missing"
if count is None:
    print("really missing")       # (does not print — count is 0, not None)
```

**Rule of thumb:** use `if x:` for "is this non-empty / non-zero?", and `if x is None:` for "was this explicitly unset?".

---

## Core concept 4 — `for` loops over iterables

Python's `for` is **for-each only**. You loop directly over the elements of any *iterable* (lists, tuples, strings, dicts, sets, files, generators…). There is no C-style `for (init; cond; step)`.

```python
fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
    print(fruit)
# => apple
# => banana
# => cherry
```

Strings are iterable character-by-character:

```python
for ch in "abc":
    print(ch)
# => a
# => b
# => c
```

### Looping over dictionaries

By default you iterate the **keys**. Use `.items()` to get key/value pairs (like PHP's `foreach ($arr as $k => $v)`), or `.values()` for just values.

```python
user = {"name": "Ada", "role": "dev"}

for key in user:                 # keys only
    print(key)
# => name
# => role

for key, value in user.items():  # key + value (note the tuple unpacking)
    print(f"{key} = {value}")
# => name = Ada
# => role = dev
```

---

## Core concept 5 — `range()` in depth

When you genuinely need numbers (a counter, N repetitions), use `range()`. It produces integers **lazily** (it doesn't build a list in memory — it's a memory-efficient sequence object).

Three forms:

```python
range(stop)              # 0, 1, ..., stop-1
range(start, stop)       # start, ..., stop-1
range(start, stop, step) # start, start+step, ... (stops before stop)
```

The **stop is exclusive** — this is the #1 off-by-one to internalize.

```python
for i in range(5):
    print(i, end=" ")    # => 0 1 2 3 4
print()

for i in range(2, 6):
    print(i, end=" ")    # => 2 3 4 5
print()

for i in range(0, 10, 2):
    print(i, end=" ")    # => 0 2 4 6 8
print()

for i in range(5, 0, -1):    # countdown with a negative step
    print(i, end=" ")    # => 5 4 3 2 1
print()
```

`range` is not a list. Materialize it with `list()` if you need to see or store it:

```python
print(range(3))          # => range(0, 3)   (the object, not its contents)
print(list(range(3)))    # => [0, 1, 2]
print(len(range(0, 10, 2)))  # => 5   (range supports len, indexing, membership)
print(7 in range(0, 10, 2))  # => False
```

To loop over a list **by index** (rarely needed — prefer `enumerate`), you can combine `range` and `len`:

```python
colors = ["red", "green", "blue"]
for i in range(len(colors)):
    print(i, colors[i])
# => 0 red
# => 1 green
# => 2 blue
```

> The above is *legal but un-Pythonic*. The idiomatic version uses `enumerate` (see Core concept 8).

---

## Core concept 6 — `while` loops, `break`, `continue`

A `while` loop repeats as long as its condition is truthy — same idea as PHP.

```python
n = 3
while n > 0:
    print(n)
    n -= 1            # Python has no n-- ; use n -= 1
# => 3
# => 2
# => 1
```

`break` exits the nearest enclosing loop immediately. `continue` skips to the next iteration.

```python
# break: stop at the first match
for n in [4, 7, 10, 13]:
    if n % 7 == 0:
        print("found", n)   # => found 7
        break

# continue: skip even numbers
for n in range(1, 8):
    if n % 2 == 0:
        continue
    print(n, end=" ")       # => 1 3 5 7
print()
```

A common safe pattern is `while True` with an explicit `break`:

```python
total = 0
while True:
    total += 10
    if total >= 50:
        break
print(total)   # => 50
```

> Note: Python has **no** `do...while`. Emulate it with `while True` + a `break` at the end, or check the condition after the first iteration.

---

## Core concept 7 — `for/else` and `while/else` (the surprising one)

Python loops can have an `else` clause. The `else` runs **only if the loop finished normally without hitting `break`**. Think of it as "no break ⇒ run the else". It is *not* "if the loop didn't run".

The classic use case is search: did we find it or not?

```python
def find_prime_pair(numbers, target):
    for n in numbers:
        if n == target:
            print(f"Found {target}")
            break
    else:
        # runs only because the for loop never broke
        print(f"{target} not present")

find_prime_pair([2, 3, 5, 7], 5)    # => Found 5
find_prime_pair([2, 3, 5, 7], 4)    # => 4 not present
```

Without `for/else` you'd need a `found = False` flag. The `else` removes that boilerplate. Read it as **"for ... else (= nobreak)"**.

`while/else` works identically — the `else` runs when the condition becomes false naturally (no `break`):

```python
attempts = 0
while attempts < 3:
    print("try", attempts)
    attempts += 1
else:
    print("ran out of attempts")
# => try 0
# => try 1
# => try 2
# => ran out of attempts
```

---

## Core concept 8 — `enumerate()` and `zip()`

### `enumerate()` — index + value, the right way

When you need both the position and the element, use `enumerate()` instead of `range(len(...))`. It yields `(index, value)` pairs.

```python
colors = ["red", "green", "blue"]
for i, color in enumerate(colors):
    print(i, color)
# => 0 red
# => 1 green
# => 2 blue
```

Pass `start=` to begin the count somewhere other than 0 (great for human-facing numbering):

```python
for rank, name in enumerate(["Gold", "Silver", "Bronze"], start=1):
    print(f"{rank}. {name}")
# => 1. Gold
# => 2. Silver
# => 3. Bronze
```

### `zip()` — walk multiple iterables in parallel

`zip()` pairs up elements from several iterables, stopping at the **shortest** one.

```python
names = ["Ada", "Linus", "Grace"]
langs = ["Python", "C", "COBOL"]

for name, lang in zip(names, langs):
    print(f"{name} -> {lang}")
# => Ada -> Python
# => Linus -> C
# => Grace -> COBOL
```

If lengths differ, `zip` silently truncates to the shortest. To catch mismatches as bugs, pass `strict=True` (Python 3.10+):

```python
list(zip([1, 2, 3], ["a", "b"]))                 # => [(1, 'a'), (2, 'b')]
list(zip([1, 2, 3], ["a", "b"], strict=True))    # raises ValueError: zip() argument 2 is shorter
```

`zip` + `dict` is a clean way to build a mapping from two parallel lists:

```python
keys = ["id", "name"]
vals = [1, "Ada"]
record = dict(zip(keys, vals))
print(record)   # => {'id': 1, 'name': 'Ada'}
```

You can even combine `enumerate` and `zip`:

```python
for i, (name, lang) in enumerate(zip(names, langs), start=1):
    print(i, name, lang)
# => 1 Ada Python
# => 2 Linus C
# => 3 Grace COBOL
```

---

## Core concept 9 — `pass` and the `...` ellipsis placeholder

Python requires every block to have a body. When you have nothing to put yet, you need a placeholder; an empty block is a `SyntaxError`.

`pass` means "do nothing — this block is intentionally empty":

```python
for n in range(3):
    pass            # TODO: implement later; loop still runs, does nothing

def not_done_yet():
    pass            # valid empty function

if user_is_admin := True:
    pass            # branch reserved for future logic
```

The `...` literal (the **Ellipsis** object) is often used as a "to be filled in" marker, especially in stubs, abstract methods, and type-stub (`.pyi`) files. Functionally it's similar to `pass` as a body, but it signals intent ("this should be implemented") more clearly.

```python
def fetch_user(user_id: int) -> dict:
    ...             # placeholder body — implement later

class Repository:
    def save(self, item) -> None: ...   # common in protocols / abstract stubs
```

`...` is a real value (`Ellipsis`), so unlike `pass` you can also assign or return it. In day-to-day code, use `pass` for empty blocks and `...` for "unimplemented stub" — both are accepted.

---

## Core concept 10 — Structural pattern matching: `match` / `case`

Introduced in **Python 3.10**, `match`/`case` is much more than a `switch`. It does **structural pattern matching**: it can match literals, *and* destructure sequences and mappings, *and* bind variables, *and* apply guards.

### Literal matching (the switch-like use)

```python
def http_label(status: int) -> str:
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500:
            return "Server Error"
        case _:                 # _ is the wildcard / default
            return "Unknown"

print(http_label(404))   # => Not Found
print(http_label(418))   # => Unknown
```

There is **no fall-through** — each `case` is self-contained, so no `break` is needed (unlike PHP/C `switch`). Combine alternatives with `|` ("or"):

```python
def kind(ch: str) -> str:
    match ch:
        case "a" | "e" | "i" | "o" | "u":
            return "vowel"
        case _:
            return "consonant"

print(kind("e"))   # => vowel
print(kind("z"))   # => consonant
```

### Capture patterns and guards

A bare name in a `case` **captures** the value into that variable. Add an `if` **guard** to refine the match.

```python
def describe(n: int) -> str:
    match n:
        case 0:
            return "zero"
        case x if x < 0:        # guard: only matches negatives, binds x
            return f"negative ({x})"
        case x if x % 2 == 0:
            return f"positive even ({x})"
        case x:                 # catch-all capture
            return f"positive odd ({x})"

print(describe(-4))   # => negative (-4)
print(describe(10))   # => positive even (10)
print(describe(7))    # => positive odd (7)
```

> Gotcha: `case status:` (a bare name) is **not** "match the variable `status`" — it *captures* into a new variable `status` and matches anything. To match against an existing constant, use a **dotted name** like `case HTTPStatus.OK:` or compare with a guard.

### Sequence patterns (destructuring lists/tuples)

This is where `match` shines — you match on *shape*.

```python
def parse_point(point):
    match point:
        case (0, 0):
            return "origin"
        case (x, 0):
            return f"on x-axis at {x}"
        case (0, y):
            return f"on y-axis at {y}"
        case (x, y):
            return f"point at ({x}, {y})"
        case _:
            return "not a 2D point"

print(parse_point((0, 0)))   # => origin
print(parse_point((5, 0)))   # => on x-axis at 5
print(parse_point((3, 4)))   # => point at (3, 4)
```

Use `*rest` to capture "the remaining elements":

```python
def head_tail(items):
    match items:
        case []:
            return "empty"
        case [only]:
            return f"single: {only}"
        case [first, *rest]:
            return f"first={first}, rest={rest}"

print(head_tail([]))            # => empty
print(head_tail([9]))           # => single: 9
print(head_tail([1, 2, 3, 4]))  # => first=1, rest=[2, 3, 4]
```

### Mapping patterns (destructuring dicts)

Match on the presence of keys and bind their values. Extra keys are ignored — it matches as long as the listed keys are present.

```python
def handle(event: dict) -> str:
    match event:
        case {"type": "click", "x": x, "y": y}:
            return f"click at {x},{y}"
        case {"type": "key", "key": k}:
            return f"key {k}"
        case {"type": t}:
            return f"unhandled type: {t}"
        case _:
            return "malformed event"

print(handle({"type": "click", "x": 10, "y": 20}))   # => click at 10,20
print(handle({"type": "key", "key": "Enter"}))       # => key Enter
print(handle({"type": "scroll", "delta": 3}))        # => unhandled type: scroll
```

### Class patterns (bonus — very common with dataclasses)

You can match on object type and pull out attributes by position or name:

```python
from dataclasses import dataclass

@dataclass
class Circle:
    radius: float

@dataclass
class Rectangle:
    width: float
    height: float

def area(shape) -> float:
    match shape:
        case Circle(radius=r):
            return 3.14159 * r * r
        case Rectangle(width=w, height=h):
            return w * h
        case _:
            raise ValueError("unknown shape")

print(area(Circle(2)))           # => 12.56636
print(area(Rectangle(3, 4)))     # => 12
```

This is enormously useful for API request handling, parsing, and state machines — patterns where PHP would force you into a chain of `instanceof`/`if` checks.

---

## Core concept 11 — Nested loops and readability

Loops nest just by indenting one inside another. `break`/`continue` affect only the **innermost** loop they sit in.

```python
for row in range(1, 4):
    for col in range(1, 4):
        print(f"{row}x{col}={row*col}", end="  ")
    print()    # newline after each row
# => 1x1=1  1x2=2  1x3=3
# => 2x1=2  2x2=4  2x3=6
# => 3x1=3  3x2=6  3x3=9
```

### Breaking out of nested loops

Python has no labeled `break`. To exit *all* levels, the cleanest options are: (a) put the loops in a function and `return`, or (b) use a flag, or (c) leverage `for/else`.

```python
def find_pair(grid, target):
    for r, row in enumerate(grid):
        for c, val in enumerate(row):
            if val == target:
                return (r, c)       # return cleanly exits both loops
    return None

grid = [[1, 2], [3, 4]]
print(find_pair(grid, 4))   # => (1, 1)
print(find_pair(grid, 9))   # => None
```

### Flattening with comprehensions

Deeply nested loops that just build a list are often clearer as a comprehension (covered fully in the comprehensions lesson, previewed here):

```python
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [x for row in matrix for x in row]   # read left-to-right as nested loops
print(flat)   # => [1, 2, 3, 4, 5, 6]
```

**Readability rule:** if you're nesting more than two levels, consider extracting an inner loop into a helper function. Deeply nested control flow is the #1 readability killer in backend code.

---

## Common mistakes & gotchas

- **Forgetting the colon** at the end of `if`/`for`/`while`/`def` headers → `SyntaxError`.
- **Mixing tabs and spaces** → `TabError`. Set your editor to "insert spaces", 4 per indent.
- **Using `&&`, `||`, `!`** instead of `and`, `or`, `not` → `SyntaxError`.
- **Off-by-one with `range`**: `range(5)` is `0..4`, not `0..5`. The stop is exclusive.
- **`if x:` when you mean `if x is None:`** — `0`, `""`, `[]`, `0.0` are all falsy, so `if not x:` silently treats a legitimate zero/empty value as "missing".
- **Assuming PHP truthiness**: in Python `"0"` and `"false"` are **truthy** (non-empty strings).
- **Expecting C-style `for(;;)`** — there isn't one; use `range()` or `enumerate()`.
- **Modifying a list while iterating over it** can skip elements or error. Iterate over a copy (`for x in items[:]:`) or build a new list.
- **`match` capture trap**: `case some_const:` does *not* compare to `some_const`; it captures into a new variable and matches everything. Use dotted constants (`Color.RED`) or a guard.
- **Expecting fall-through in `match`** — there is none, and no `break` is needed. Use `|` to share a body across patterns.
- **`zip` silently truncating** to the shortest iterable — pass `strict=True` when lengths must match.
- **Trying to use a labeled `break`** for nested loops — Python has none; refactor into a function with `return`.
- **Mutable default / off-by-one in `while`** loops that never decrement the counter → infinite loop. Always make sure the condition can become false.

## Best practices

- Indent with **4 spaces**; never tabs. Let your editor/formatter (`black`, `ruff format`) enforce it.
- Prefer **for-each** (`for item in items`) over index loops. Reach for `enumerate()` when you need the index, `zip()` when you walk lists in parallel.
- Use truthiness idioms: `if items:` (non-empty), `if not items:` (empty) — but use `is None` / `is not None` for explicit null checks.
- Keep ternaries to a single simple choice; escalate to `if/elif/else` when there are multiple branches.
- Reach for `match`/`case` when dispatching on the **shape** of data (parsed JSON, events, AST nodes, dataclasses). For a simple value-to-value lookup, a plain `dict` is often even cleaner than `match`.
- Use `for/else` to express "search, and handle the not-found case" without a flag variable.
- Always provide a wildcard `case _:` in `match` to handle unexpected input explicitly.
- To break out of nested loops, prefer extracting a function and using `return`.
- Use `pass` for intentionally empty blocks and `...` for "stub to be implemented".
- Avoid mutating a collection while iterating it; iterate a copy or build a new one.

## Interview Q&A

**Q: How does Python delimit code blocks, and what's the practical risk?**
**A:** By indentation, not braces. A header line ends with `:` and the indented lines beneath it form the block. The practical risk is `IndentationError`/`TabError` from inconsistent indentation or mixing tabs and spaces. The fix is to standardize on 4 spaces and use a formatter.

**Q: What values are falsy in Python?**
**A:** `False`, `None`, numeric zero (`0`, `0.0`, `0j`, `Decimal(0)`), and empty containers/sequences: `""`, `[]`, `()`, `{}`, `set()`, `b""`, `range(0)`. Everything else is truthy — including non-empty strings like `"0"` and `"false"`, which differs from PHP.

**Q: Why use `is None` instead of `== None`?**
**A:** `None` is a singleton, so identity (`is`) is the correct, faster, and idiomatic check. Also, `==` can be overridden by a class's `__eq__`, so `== None` could behave unexpectedly; `is None` cannot be fooled.

**Q: When would you use `if x is None` vs `if not x`?**
**A:** Use `if not x` to test for "empty or zero" (empty list, empty string, 0). Use `if x is None` to detect an explicitly missing value. They differ because `0`, `""`, and `[]` are falsy but are *not* `None` — conflating them is a classic bug.

**Q: Explain `range(start, stop, step)` and a common pitfall.**
**A:** It yields integers from `start` (default 0) up to but **not including** `stop`, advancing by `step` (default 1, may be negative). The common pitfall is assuming `stop` is inclusive — `range(5)` gives `0,1,2,3,4`. `range` is a lazy sequence, not a list; wrap in `list()` to materialize it.

**Q: What does the `else` on a `for` or `while` loop do?**
**A:** It runs only if the loop completed **without** executing `break`. It's ideal for search loops: the `else` is the "not found" branch, eliminating a manual `found` flag. It does *not* mean "if the loop body never ran".

**Q: Difference between `break` and `continue`?**
**A:** `break` exits the nearest enclosing loop entirely; `continue` skips the rest of the current iteration and proceeds to the next one. Both affect only the innermost loop they're in.

**Q: How do `enumerate()` and `zip()` differ, and why prefer them over `range(len(...))`?**
**A:** `enumerate(it)` yields `(index, value)` pairs from a single iterable; `zip(a, b, ...)` yields tuples pairing elements across multiple iterables (stopping at the shortest, unless `strict=True`). They're preferred over `range(len(...))` because they're more readable, less error-prone, and work on any iterable, not just indexable sequences.

**Q: How is `match`/`case` more than a `switch`?**
**A:** Besides literal matching, it performs structural pattern matching: it can destructure sequences (`[first, *rest]`), mappings (`{"type": t}`), and class instances (`Circle(radius=r)`), bind captured variables, apply `if` guards, and combine alternatives with `|`. It has no fall-through, so no `break` is needed.

**Q: What's the capture-vs-constant gotcha in `match`?**
**A:** A bare name in a `case` is a **capture** pattern — it matches anything and binds the value to that name, shadowing any same-named variable. To match against a constant you must use a dotted name (e.g. `case HTTPStatus.OK:`) or a literal, or compare inside a guard.

**Q: How do you break out of nested loops in Python?**
**A:** There's no labeled `break`. The cleanest approach is to move the loops into a function and `return` from inside. Alternatives are a boolean flag checked by the outer loop, or restructuring with `for/else`.

**Q: What's the difference between `pass` and `...`?**
**A:** Both can serve as a placeholder body. `pass` is a no-op statement used for intentionally empty blocks. `...` is the `Ellipsis` object (a real value) commonly used to mark unimplemented stubs, abstract methods, and type-stub files; it reads as "to be implemented".

## Practice exercises

Try these before peeking at the solutions.

1. **FizzBuzz.** Print numbers 1 to 20, but print `"Fizz"` for multiples of 3, `"Buzz"` for multiples of 5, and `"FizzBuzz"` for multiples of both.

2. **Truthiness filter.** Given `data = [0, "", "hi", None, [], [1], "0", 42]`, build a list of only the *truthy* items, and a separate list of items that are explicitly `None`.

3. **Parallel scoreboard.** Given `players = ["Ann", "Bob", "Cy"]` and `scores = [88, 92, 75]`, print a 1-indexed leaderboard line like `1. Ann — 88`. Use `enumerate` and `zip`.

4. **Search with `for/else`.** Write a function `first_even(nums)` that returns the first even number, or prints `"no evens"` and returns `None` if there are none. Use a `for/else`.

5. **Event router with `match`.** Write `route(event: dict)` handling these shapes: `{"type": "login", "user": u}` → `"login: {u}"`; `{"type": "logout"}` → `"logout"`; `{"type": "msg", "text": t}` where `t` is non-empty → `"msg: {t}"`; anything else → `"ignored"`. Use a guard for the non-empty text.

6. **Matrix transpose with `zip`.** Given `m = [[1, 2, 3], [4, 5, 6]]`, produce its transpose `[[1, 4], [2, 5], [3, 6]]` in one line using `zip`.

<details>
<summary>Solutions</summary>

```python
# 1. FizzBuzz
for n in range(1, 21):
    if n % 15 == 0:
        print("FizzBuzz")
    elif n % 3 == 0:
        print("Fizz")
    elif n % 5 == 0:
        print("Buzz")
    else:
        print(n)
# (Alternative using string building:)
for n in range(1, 21):
    out = ("Fizz" if n % 3 == 0 else "") + ("Buzz" if n % 5 == 0 else "")
    print(out or n)   # `out or n` falls back to n when out is the empty (falsy) string


# 2. Truthiness filter
data = [0, "", "hi", None, [], [1], "0", 42]
truthy = [x for x in data if x]                 # keep only truthy items
nones = [x for x in data if x is None]
print(truthy)   # => ['hi', [1], '0', 42]   (note: "0" is truthy!)
print(nones)    # => [None]


# 3. Parallel scoreboard
players = ["Ann", "Bob", "Cy"]
scores = [88, 92, 75]
for rank, (name, score) in enumerate(zip(players, scores), start=1):
    print(f"{rank}. {name} — {score}")
# => 1. Ann — 88
# => 2. Bob — 92
# => 3. Cy — 75


# 4. Search with for/else
def first_even(nums):
    for n in nums:
        if n % 2 == 0:
            return n
    else:
        print("no evens")
        return None

print(first_even([1, 3, 6, 9]))   # => 6
print(first_even([1, 3, 5]))      # prints "no evens", => None


# 5. Event router with match
def route(event: dict) -> str:
    match event:
        case {"type": "login", "user": u}:
            return f"login: {u}"
        case {"type": "logout"}:
            return "logout"
        case {"type": "msg", "text": t} if t:    # guard: non-empty text
            return f"msg: {t}"
        case _:
            return "ignored"

print(route({"type": "login", "user": "ada"}))    # => login: ada
print(route({"type": "logout"}))                  # => logout
print(route({"type": "msg", "text": "hi"}))       # => msg: hi
print(route({"type": "msg", "text": ""}))         # => ignored (guard failed)
print(route({"type": "ping"}))                    # => ignored


# 6. Matrix transpose with zip
m = [[1, 2, 3], [4, 5, 6]]
transposed = [list(col) for col in zip(*m)]   # *m unpacks rows as zip args
print(transposed)   # => [[1, 4], [2, 5], [3, 6]]
```

</details>

## Cheat sheet

```python
# --- if / elif / else (colons + indentation, no braces) ---
if cond:
    ...
elif other:
    ...
else:
    ...

# Logical ops: and / or / not  (NOT &&, ||, !)
# Chained comparison: if 0 < x < 10:

# --- Ternary (note the order) ---
val = a if cond else b

# --- Truthiness ---
# Falsy: False, None, 0, 0.0, "", [], (), {}, set(), range(0)
# Truthy: everything else  (incl. "0", "false", " ", [0])
if items:        # non-empty?
if not items:    # empty?
if x is None:    # explicit null check (use `is`, not `==`)

# --- for-each over any iterable ---
for item in iterable: ...
for k, v in d.items(): ...

# --- range(start, stop, step)  (stop EXCLUSIVE) ---
range(5)            # 0 1 2 3 4
range(2, 6)         # 2 3 4 5
range(0, 10, 2)     # 0 2 4 6 8
range(5, 0, -1)     # 5 4 3 2 1

# --- while + loop controls ---
while cond: ...
break       # exit innermost loop
continue    # next iteration
# no do-while -> use `while True:` + break
# no labeled break -> move to a function and `return`

# --- for/else & while/else  (else = "no break happened") ---
for x in xs:
    if hit(x): break
else:
    handle_not_found()

# --- index + value / parallel iteration ---
for i, v in enumerate(xs, start=1): ...
for a, b in zip(xs, ys): ...           # stops at shortest
for a, b in zip(xs, ys, strict=True):  # error if lengths differ

# --- placeholders ---
pass     # do-nothing statement (empty block)
...      # Ellipsis; marks an unimplemented stub

# --- match / case (3.10+, no fall-through) ---
match value:
    case 200:                       # literal
        ...
    case "a" | "e" | "i":           # alternatives
        ...
    case n if n < 0:                # capture + guard
        ...
    case [first, *rest]:            # sequence destructure
        ...
    case {"type": t}:               # mapping destructure
        ...
    case Circle(radius=r):          # class destructure
        ...
    case _:                         # wildcard / default
        ...
```
