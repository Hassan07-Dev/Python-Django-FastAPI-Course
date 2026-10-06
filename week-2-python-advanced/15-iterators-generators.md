# 15 — Iterators & Generators

> Week 2 · Python core / advanced language features

## What you will learn

- The precise difference between an **iterable** and an **iterator** (and why it matters).
- The **iterator protocol**: `__iter__`, `__next__`, and the role of `StopIteration`.
- What a `for` loop *actually* does under the hood.
- How to build a **custom iterator class**.
- **Generator functions** with `yield` — how Python suspends and resumes their state.
- **Generator expressions** (the lazy cousin of list comprehensions).
- `yield from` for **delegation** to sub-generators.
- **Laziness & memory efficiency** — processing huge or even infinite sequences without blowing up RAM.
- The advanced generator methods `send`, `throw`, `close` (brief but practical).
- Real-world generators: reading large files line by line, building **data pipelines**.
- The **`itertools`** toolbox (`count`, `cycle`, `islice`, `chain`, `groupby`) and how it composes with your own generators.

## PHP/Laravel bridge

PHP already has generators — `yield` works almost identically, and `function gen() { yield 1; yield 2; }` returns a `Generator` object you `foreach` over. PHP's `Iterator` interface (`current()`, `next()`, `key()`, `valid()`, `rewind()`) is Python's iterator protocol with different method names; Python's `IteratorAggregate` equivalent is just implementing `__iter__`. Laravel's `LazyCollection` (built on generators) is the spiritual twin of Python generators + `itertools`: process a million-row CSV or a chunked Eloquent query without loading everything into memory. The big mental shift: in Python, iteration is *everywhere* and *uniform* — `for`, comprehensions, `sum()`, `max()`, unpacking, and the `*` splat all speak the same protocol, so a generator you write plugs into all of them for free.

## Core concept: iterable vs iterator

These two words sound interchangeable but are not. Getting this distinction crisp is the foundation for everything else.

- An **iterable** is anything you *can* loop over. It knows how to *produce* an iterator. Formally: it implements `__iter__()`, which returns an iterator. Lists, tuples, dicts, sets, strings, files, ranges — all iterable.
- An **iterator** is the object that *does* the actual walking. It implements `__next__()` (hand me the next item) and also `__iter__()` (return itself). It holds the *position/state* of the iteration.

Think of an iterable as a **book** and an iterator as a **bookmark**. The book doesn't know where you are; the bookmark does. You can have several bookmarks in the same book at once.

```python
nums = [10, 20, 30]          # the iterable (the "book")

it = iter(nums)              # ask the iterable for an iterator (a "bookmark")
print(type(it))             # => <class 'list_iterator'>

print(next(it))             # => 10
print(next(it))             # => 20
print(next(it))             # => 30
print(next(it))             # raises StopIteration
```

`iter(x)` calls `x.__iter__()`; `next(x)` calls `x.__next__()`. When the iterator is exhausted it raises `StopIteration` — that is the official "I'm done" signal, not an error you should fear.

### Why an iterator is also iterable (it returns itself)

A key rule: **every iterator must also be an iterable**, and its `__iter__` must return `self`. This lets you use an iterator directly in a `for` loop. But beware the consequence — iterators are *one-shot*:

```python
nums = [1, 2, 3]
it = iter(nums)

print(list(it))   # => [1, 2, 3]   consumes the iterator
print(list(it))   # => []          already exhausted, nothing left!

# A list, by contrast, is restartable because each iter() call makes a FRESH iterator:
print(list(nums)) # => [1, 2, 3]
print(list(nums)) # => [1, 2, 3]   works again
```

This is the single most common source of generator/iterator bugs: you iterate once, then try again and get nothing.

## Core concept: how `for` works under the hood

A `for` loop is **syntactic sugar**. When you write this:

```python
for x in [10, 20, 30]:
    print(x)
```

Python effectively does this:

```python
_iterator = iter([10, 20, 30])     # 1. get an iterator from the iterable
while True:
    try:
        x = next(_iterator)        # 2. pull the next value
    except StopIteration:          # 3. exhausted -> stop the loop cleanly
        break
    print(x)                       # 4. run the loop body
```

That's the whole magic. Anything that satisfies the protocol — your own class, a generator, a file object — drops straight into `for`, `list()`, `sum()`, `min()`, `", ".join(...)`, `*unpacking`, comprehensions, and more, with zero special-casing.

## Core concept: building a custom iterator class

Let's implement the protocol by hand to demystify it. We'll build a `Countdown` that counts from `n` down to `1`.

```python
class Countdown:
    def __init__(self, start: int) -> None:
        self.start = start

    def __iter__(self):
        # Return a FRESH iterator each time so the iterable is reusable.
        return CountdownIterator(self.start)


class CountdownIterator:
    def __init__(self, current: int) -> None:
        self.current = current

    def __iter__(self):
        return self          # an iterator returns itself

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration
        value = self.current
        self.current -= 1
        return value


cd = Countdown(3)
print(list(cd))   # => [3, 2, 1]
print(list(cd))   # => [3, 2, 1]   reusable, because __iter__ makes a new iterator

for n in Countdown(5):
    print(n, end=" ")   # => 5 4 3 2 1
print()
```

Notice the deliberate split into two classes. If you fuse them — put `__next__` on `Countdown` itself and `return self` from `__iter__` — the object becomes a one-shot iterator (like the `it` example above) and can't be re-looped. Keeping the *iterable* and its *iterator* separate is the idiomatic way to get a reusable collection.

That's a lot of boilerplate for a countdown. Generators let us write the same thing in three lines.

## Core concept: generator functions with `yield`

A **generator function** looks like a normal function but contains at least one `yield`. Calling it does **not** run the body — it instantly returns a **generator object** (which is an iterator). The body runs lazily, one chunk at a time, each time you call `next()`.

```python
def countdown(start: int):
    while start > 0:
        yield start          # hand a value to the caller AND pause here
        start -= 1           # resumes here on the next next() call

gen = countdown(3)
print(type(gen))   # => <class 'generator'>

print(next(gen))   # => 3
print(next(gen))   # => 2
print(next(gen))   # => 1
print(next(gen))   # raises StopIteration

# Or just loop — the for handles StopIteration for you:
print(list(countdown(5)))   # => [5, 4, 3, 2, 1]
```

This single function replaces both classes from the previous section, and it's reusable (each call returns a brand-new generator).

### How state is suspended and resumed

This is the part that feels like magic but isn't. When execution hits `yield value`:

1. `value` is handed back to whoever called `next()`.
2. The function is **frozen** — its local variables, instruction pointer, and call stack are all preserved in the generator object.
3. Control returns to the caller, who can do other work.
4. The next `next()` call **thaws** the function and resumes execution on the line *right after* the `yield`, with all locals intact.

A `print`-trace makes the pause/resume visible:

```python
def chatty():
    print("  -> starting")
    yield 1
    print("  -> resumed after first yield")
    yield 2
    print("  -> resumed after second yield")

g = chatty()
print("created generator (nothing printed yet)")
print("got:", next(g))   # prints "-> starting", then "got: 1"
print("got:", next(g))   # prints "-> resumed after first yield", then "got: 2"
# next(g) here would print "-> resumed after second yield" then raise StopIteration
```

Output:

```text
created generator (nothing printed yet)
  -> starting
got: 1
  -> resumed after first yield
got: 2
```

The body only advances when you ask for the next value. That on-demand execution is the heart of **laziness**.

### `return` inside a generator

A bare `return` (or falling off the end) ends the generator and raises `StopIteration`. A `return value` attaches that value to the exception (`StopIteration.value`) — mostly useful with `yield from`:

```python
def with_result():
    yield 1
    yield 2
    return "done"          # becomes StopIteration.value, NOT yielded

g = with_result()
print(next(g))   # => 1
print(next(g))   # => 2
try:
    next(g)
except StopIteration as e:
    print(e.value)   # => done
```

## Core concept: generator expressions

A **generator expression** is to a generator what a list comprehension is to a list. Same syntax — but with **parentheses** instead of brackets — and it's **lazy**: it produces values one at a time instead of building the whole list in memory.

```python
nums = range(1, 6)

squares_list = [n * n for n in nums]   # list comprehension -> builds a list now
squares_gen  = (n * n for n in nums)   # generator expression -> lazy, builds nothing yet

print(squares_list)        # => [1, 4, 9, 16, 25]
print(squares_gen)         # => <generator object <genexpr> at 0x...>
print(list(squares_gen))   # => [1, 4, 9, 16, 25]   (now it runs)
```

The payoff is memory. Compare summing the squares of ten million numbers:

```python
# List comprehension: materializes ~10M ints in a list first. Wasteful.
total = sum([n * n for n in range(10_000_000)])

# Generator expression: streams one value at a time. Tiny, constant memory.
total = sum(n * n for n in range(10_000_000))
```

When a generator expression is the *sole* argument to a function, you can drop the extra parentheses — `sum(n * n for n in range(10))` is fine. Use generator expressions by default when you're going to consume the result immediately (sum, max, any, all, join, a `for` loop). Use a list comprehension only when you genuinely need a list you can index, re-iterate, or keep around.

```python
words = ["api", "db", "cache"]
# any/all short-circuit and never build a list:
print(any(len(w) > 4 for w in words))   # => True  ("cache" has length 5)
print(", ".join(w.upper() for w in words))  # => API, DB, CACHE
```

## Core concept: `yield from` for delegation

When one generator needs to yield *all* the values of another iterable, the naive way is a loop:

```python
def chain_manual(a, b):
    for x in a:
        yield x
    for x in b:
        yield x
```

`yield from` says "delegate to this sub-iterable and yield everything it produces":

```python
def chain_two(a, b):
    yield from a
    yield from b

print(list(chain_two([1, 2], [3, 4])))   # => [1, 2, 3, 4]
```

It shines for **recursion**, like flattening a nested structure:

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)   # delegate to the recursive call
        else:
            yield item

data = [1, [2, 3, [4, [5]]], 6]
print(list(flatten(data)))   # => [1, 2, 3, 4, 5, 6]
```

Beyond being shorter, `yield from` also transparently forwards `send`/`throw` to the sub-generator and captures its `return` value — things the manual `for` loop does not do. For everyday delegation just remember: `yield from iterable` re-yields each item.

## Core concept: laziness & memory efficiency

Because generators compute values on demand, they can represent sequences that are **too big for memory** — or even **infinite**. The trick is to pair an unbounded producer with a *consumer that stops*.

```python
def naturals():
    n = 1
    while True:        # infinite! never builds a collection
        yield n
        n += 1

gen = naturals()
print(next(gen), next(gen), next(gen))   # => 1 2 3

# Take just the first 5 using itertools.islice (more on itertools below):
from itertools import islice
print(list(islice(naturals(), 5)))       # => [1, 2, 3, 4, 5]
```

If you tried to build naturals as a list you'd hang forever and run out of RAM. The generator stays flat — it only ever holds `n` and remembers where it paused. This is the difference between **eager** (compute everything now) and **lazy** (compute the next thing only when asked) evaluation.

A quick mental model for memory: a list of one million dicts holds a million dicts at once; a generator yielding those dicts one at a time holds roughly **one** at a time. For ETL, log processing, and streaming APIs, that's the difference between a 4 GB process and a 40 MB one.

## Core concept: practical generators

### Reading a large file line by line

File objects are *already* lazy iterators over their lines — iterating a file does not load it all into memory. This is the canonical real-world generator.

```python
# Memory-friendly: reads ONE line at a time, even for a 50 GB file.
def read_lines(path: str):
    with open(path, encoding="utf-8") as f:
        for line in f:              # the file object yields lines lazily
            yield line.rstrip("\n")

# Contrast with the eager, memory-hungry version you should avoid for big files:
# lines = open(path).read().splitlines()   # loads the WHOLE file into RAM
```

### Building a pipeline (generators composing generators)

Each stage is a generator that takes an iterable and yields a transformed iterable. Because every stage is lazy, data flows through one item at a time — like Unix pipes (`cat file | grep ERROR | cut ...`).

```python
def read_lines(lines):
    for line in lines:
        yield line.rstrip("\n")

def only_errors(lines):
    for line in lines:
        if "ERROR" in line:
            yield line

def extract_message(lines):
    for line in lines:
        # "2026-06-19 ERROR something broke" -> "something broke"
        yield line.split("ERROR", 1)[1].strip()

raw = [
    "2026-06-19 INFO  started\n",
    "2026-06-19 ERROR disk full\n",
    "2026-06-19 INFO  retry\n",
    "2026-06-19 ERROR timeout\n",
]

# Wire the stages together. Nothing runs until we iterate the final generator.
pipeline = extract_message(only_errors(read_lines(raw)))
for msg in pipeline:
    print(msg)
# => disk full
# => timeout
```

No intermediate lists are built between stages. A 10 GB log file flows through this with near-constant memory. This composability — generators feeding generators — is the pattern interviewers love to see.

## Core concept: `send`, `throw`, `close` (brief)

Generators are technically **coroutines**: you can push data *in*, not just pull data *out*.

- **`gen.send(value)`** resumes the generator and makes the paused `yield` expression *evaluate to* `value`. (You must `next()`/`send(None)` once first to reach the initial `yield`.)
- **`gen.throw(ExcType)`** raises an exception *at the point of the yield*, so the generator can catch and react to it.
- **`gen.close()`** raises `GeneratorExit` inside the generator, letting it run cleanup (often in a `finally`).

```python
def accumulator():
    total = 0
    try:
        while True:
            x = yield total       # value sent in becomes x; we yield the running total
            total += x
    finally:
        print(f"  (closing; final total was {total})")

acc = accumulator()
print(next(acc))        # => 0   prime it; runs up to the first yield
print(acc.send(10))     # => 10
print(acc.send(5))      # => 15
acc.close()             # prints "(closing; final total was 15)"
```

In modern code you'll rarely hand-write coroutines this way — `async`/`await` (a later lesson) is the high-level replacement. But knowing `send`/`throw`/`close` exist, and that `close` triggers `finally` for cleanup, is fair game in interviews.

## Core concept: the `itertools` toolbox

`itertools` is a standard-library module of fast, memory-efficient building blocks that all return **lazy iterators**. They compose beautifully with your own generators. Five to know cold:

| Function | What it does | Lazy/infinite? |
|---|---|---|
| `count(start, step)` | Counts upward forever | Infinite |
| `cycle(iterable)` | Repeats an iterable forever | Infinite |
| `islice(it, stop)` / `islice(it, start, stop, step)` | Slices an iterator (you can't `it[1:5]`) | Lazy |
| `chain(*iterables)` | Concatenates iterables into one stream | Lazy |
| `groupby(it, key)` | Groups **consecutive** items by a key | Lazy |

```python
from itertools import count, cycle, islice, chain, groupby

# count: infinite counter (often used for IDs / enumerate-like steps)
print(list(islice(count(10, 2), 4)))      # => [10, 12, 14, 16]

# cycle: round-robin assignment without manual index math
servers = cycle(["s1", "s2", "s3"])
print([next(servers) for _ in range(5)])  # => ['s1', 's2', 's3', 's1', 's2']

# islice: safely take a window from ANY iterator (incl. infinite ones)
print(list(islice(count(), 2, 8, 2)))     # => [2, 4, 6]

# chain: flatten several iterables into a single stream (lazily)
print(list(chain([1, 2], (3, 4), "ab")))  # => [1, 2, 3, 4, 'a', 'b']
```

### `groupby` — the gotcha that trips everyone

`groupby` only groups **adjacent** equal keys, exactly like SQL `GROUP BY` *without* an `ORDER BY`. **Sort first** if you want global grouping. Each group's values are themselves a lazy iterator, so materialize them (e.g. with `list`) before moving on.

```python
from itertools import groupby

people = [
    {"team": "api", "name": "Ada"},
    {"team": "api", "name": "Lin"},
    {"team": "web", "name": "Sam"},
    {"team": "api", "name": "Bob"},   # out of order!
]

key = lambda p: p["team"]

# WRONG: unsorted input -> 'api' appears twice because groups are by ADJACENCY
for team, members in groupby(people, key=key):
    print(team, [m["name"] for m in members])
# => api ['Ada', 'Lin']
# => web ['Sam']
# => api ['Bob']        <-- split!

print("---")

# RIGHT: sort by the same key first
for team, members in groupby(sorted(people, key=key), key=key):
    print(team, [m["name"] for m in members])
# => api ['Ada', 'Lin', 'Bob']
# => web ['Sam']
```

### Composing itertools with your own generators

Everything is just an iterator, so it all snaps together:

```python
from itertools import islice, chain

def naturals():
    n = 1
    while True:
        yield n
        n += 1

# first 3 naturals, then a fixed tail — all lazy until list() consumes it
print(list(chain(islice(naturals(), 3), ["end"])))   # => [1, 2, 3, 'end']
```

## Common mistakes & gotchas

- **Reusing an exhausted iterator/generator.** A generator is one-shot. `g = (x for x in data); list(g)` then `list(g)` gives `[]` the second time. Re-create the generator, or use a list if you need to iterate twice.
- **Confusing `[...]` with `(...)`.** `[x for x in it]` builds a list immediately; `(x for x in it)` is a lazy generator. Don't write a list comprehension when you only need to stream once — it wastes memory.
- **`len()` on a generator.** Generators have no length (`len(g)` raises `TypeError`) because they don't know how many items remain. Convert to a list first, or count with `sum(1 for _ in g)` (which *consumes* it).
- **Indexing/slicing a generator.** `g[0]` and `g[1:5]` fail — generators aren't sequences. Use `next(g)` or `itertools.islice`.
- **`groupby` without sorting.** Produces fragmented groups. Sort by the same key first.
- **Consuming `groupby` sub-iterators late.** The grouped values are lazy and become invalid once you advance the outer `groupby`. Convert each group to a `list` immediately if you need it later.
- **Returning a value expecting it to be yielded.** `return x` in a generator does *not* yield `x`; it ends iteration and stashes `x` in `StopIteration.value`.
- **Forgetting to prime a `send`-based generator.** You must advance to the first `yield` (with `next()` or `send(None)`) before sending real values, or you get `TypeError`.
- **Holding references that defeat laziness.** Wrapping a generator in `list(...)` "to be safe" loads everything into memory — the very thing you used a generator to avoid.

## Best practices

- Prefer **generator expressions** over list comprehensions when you'll consume the result once (`sum`, `any`, `all`, `max`, `join`, a single `for`).
- Reach for **generator functions** when iteration logic is non-trivial (state machines, recursion, pipelines) — they're far cleaner than custom iterator classes.
- Use **`yield from`** to delegate to sub-iterables instead of manual re-yield loops.
- Build **pipelines** as a chain of small, single-purpose generators; each stays lazy and testable in isolation.
- Lean on **`itertools`** before writing your own looping logic — it's C-fast, lazy, and battle-tested. `islice` is your safe slicer for any iterator.
- For big files, **iterate the file object directly** (`for line in f`); never `f.read()` the whole thing unless it's small.
- Keep an **iterable** (reusable, fresh iterator each `__iter__`) distinct from an **iterator** (one-shot, returns `self`). When writing a custom collection, make `__iter__` return a new iterator.
- Add type hints with `Iterator[T]` / `Iterable[T]` / `Generator[Y, S, R]` from `collections.abc` (or `typing`) so callers know what they get.

## Interview Q&A

**Q: What is the difference between an iterable and an iterator?**
**A:** An iterable implements `__iter__()` and can produce an iterator (e.g. list, dict, str, file). An iterator implements `__next__()` (and `__iter__` returning `self`) and holds the iteration state. You get an iterator *from* an iterable via `iter()`. Crucially, iterators are one-shot and stateful; many iterables are reusable because each `iter()` call yields a fresh iterator.

**Q: What exactly does a `for` loop do under the hood?**
**A:** It calls `iter()` on the iterable to get an iterator, then repeatedly calls `next()` on it, binding each result to the loop variable and running the body, until `next()` raises `StopIteration`, which the loop catches to terminate cleanly.

**Q: What is `StopIteration` and who raises it?**
**A:** It's the exception an iterator raises from `__next__` to signal exhaustion. The `for` loop (and `list()`, `sum()`, etc.) catch it automatically to end iteration. It's a control-flow signal, not an error condition.

**Q: How does a generator function differ from a normal function?**
**A:** A function containing `yield` is a generator function. Calling it doesn't execute the body — it returns a generator object (an iterator). The body runs incrementally: each `next()` runs until the next `yield`, which produces a value and suspends execution, preserving all local state until the next `next()`.

**Q: Generator expression vs list comprehension — when do you use which?**
**A:** Same syntax, parentheses vs brackets. A list comprehension builds the whole list eagerly in memory; a generator expression is lazy and yields one item at a time. Use a generator expression when consuming once (especially with `sum`/`any`/`all`/`join`) for constant memory; use a list comprehension when you need a reusable, indexable list.

**Q: How do generators save memory?**
**A:** They compute values on demand instead of materializing the whole sequence. At any moment a generator holds only its current local state, not all items. This lets you process huge files or infinite sequences in roughly constant memory.

**Q: What does `yield from` do?**
**A:** It delegates to a sub-iterable, yielding all of its items in turn — shorthand for a re-yield loop. It also forwards `send`/`throw` to the sub-generator and captures the sub-generator's `return` value, which a manual loop wouldn't.

**Q: Can you iterate the same generator twice?**
**A:** No. A generator is a one-shot iterator; once exhausted it yields nothing on subsequent iteration. To iterate again, recreate the generator (call the function again) or materialize it into a list if you need to reuse the data.

**Q: What do `send`, `throw`, and `close` do?**
**A:** `send(value)` resumes the generator with `value` as the result of the paused `yield` (enabling two-way communication / coroutines). `throw(exc)` raises an exception at the yield point so the generator can handle it. `close()` raises `GeneratorExit` inside the generator to trigger cleanup (e.g. a `finally` block).

**Q: Why must you sort before `itertools.groupby`?**
**A:** `groupby` groups only *consecutive* items sharing a key — like SQL `GROUP BY` without `ORDER BY`. Unsorted input produces fragmented, repeated groups. Sort by the same key function first to get correct global grouping.

**Q: How would you process a 100 GB log file in Python without running out of memory?**
**A:** Iterate the file object line by line (`for line in f`), which is lazy, and pass each line through a pipeline of generators (filter, transform) that also stream one item at a time. Avoid `f.read()` or list comprehensions that materialize everything. Use `itertools.islice` if you only need a window.

**Q: Why does a custom iterable usually split into two classes?**
**A:** So it's reusable. The iterable's `__iter__` returns a *new* iterator object each time, so multiple independent loops can run over the same data. If the iterable were its own iterator (`__iter__` returns `self`), it would be one-shot and couldn't be re-looped.

## Practice exercises

1. **Fibonacci generator.** Write a generator `fibonacci()` that yields the Fibonacci sequence indefinitely (1, 1, 2, 3, 5, 8, ...). Use `itertools.islice` to print the first 10 values.

2. **Custom reusable iterable.** Build a class `Squares(n)` that, when iterated, yields the squares `1, 4, 9, ..., n²`. Make it reusable (looping twice should both work). Verify with two consecutive `for` loops.

3. **Chunker.** Write a generator `chunked(iterable, size)` that yields successive lists of length `size` (the last chunk may be shorter). Example: `list(chunked(range(7), 3))` → `[[0,1,2],[3,4,5],[6]]`. (Hint: `itertools.islice` over `iter(iterable)`.)

4. **Log pipeline.** Given a list of log lines, build a generator pipeline that (a) strips whitespace, (b) keeps only lines containing `"WARN"` or `"ERROR"`, and (c) yields `(level, message)` tuples. Keep every stage lazy.

5. **Group by first letter.** Given `["apple", "avocado", "banana", "blueberry", "cherry"]`, use `itertools.groupby` to produce a dict mapping each first letter to the list of words. Remember to sort first.

6. **(Stretch) Running average coroutine.** Write a `send`-based generator `running_average()` that, each time you `send(x)`, yields the running mean of all values sent so far. Prime it, send a few numbers, and confirm the means.

<details>
<summary>Solutions</summary>

```python
# 1. Fibonacci generator
from itertools import islice

def fibonacci():
    a, b = 1, 1
    while True:
        yield a
        a, b = b, a + b

print(list(islice(fibonacci(), 10)))
# => [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]


# 2. Custom reusable iterable (two-class pattern)
class Squares:
    def __init__(self, n: int) -> None:
        self.n = n

    def __iter__(self):
        return SquaresIterator(self.n)

class SquaresIterator:
    def __init__(self, n: int) -> None:
        self.n = n
        self.i = 1

    def __iter__(self):
        return self

    def __next__(self) -> int:
        if self.i > self.n:
            raise StopIteration
        value = self.i * self.i
        self.i += 1
        return value

sq = Squares(4)
print(list(sq))   # => [1, 4, 9, 16]
print(list(sq))   # => [1, 4, 9, 16]   reusable

# (A generator-based version is shorter and also reusable since each call is fresh:)
def squares(n):
    for i in range(1, n + 1):
        yield i * i
print(list(squares(4)))   # => [1, 4, 9, 16]


# 3. Chunker
from itertools import islice

def chunked(iterable, size: int):
    it = iter(iterable)
    while True:
        chunk = list(islice(it, size))
        if not chunk:
            return
        yield chunk

print(list(chunked(range(7), 3)))   # => [[0, 1, 2], [3, 4, 5], [6]]


# 4. Log pipeline
def strip_lines(lines):
    for line in lines:
        yield line.strip()

def only_warn_error(lines):
    for line in lines:
        if "WARN" in line or "ERROR" in line:
            yield line

def to_tuples(lines):
    for line in lines:
        level, _, message = line.partition(" ")
        yield (level, message)

logs = [
    "  INFO started  ",
    "WARN low disk space",
    "DEBUG noise",
    "ERROR connection refused",
]

pipeline = to_tuples(only_warn_error(strip_lines(logs)))
print(list(pipeline))
# => [('WARN', 'low disk space'), ('ERROR', 'connection refused')]


# 5. Group by first letter
from itertools import groupby

words = ["apple", "avocado", "banana", "blueberry", "cherry"]
key = lambda w: w[0]

grouped = {
    letter: list(group)
    for letter, group in groupby(sorted(words, key=key), key=key)
}
print(grouped)
# => {'a': ['apple', 'avocado'], 'b': ['banana', 'blueberry'], 'c': ['cherry']}


# 6. Running average coroutine
def running_average():
    total = 0.0
    count = 0
    average = None
    while True:
        x = yield average
        total += x
        count += 1
        average = total / count

avg = running_average()
next(avg)                  # prime it (yields None)
print(avg.send(10))        # => 10.0
print(avg.send(20))        # => 15.0
print(avg.send(30))        # => 20.0
```

</details>

## Cheat sheet

```python
# --- Protocol ---
iter(x)        # -> x.__iter__()   get an iterator from an iterable
next(it)       # -> it.__next__()  get next value (raises StopIteration when done)
next(it, dflt) # return dflt instead of raising when exhausted

# --- for loop is just ---
it = iter(obj)
while True:
    try: x = next(it)
    except StopIteration: break
    ...  # body

# --- Generator function ---
def gen():
    yield 1            # pause + emit; resume after this line on next next()
    yield from [2, 3]  # delegate: re-yield each item
    return "result"    # -> StopIteration.value (NOT yielded)

# --- Generator expression (lazy) vs list comp (eager) ---
(x*x for x in it)   # lazy, one-shot, parentheses
[x*x for x in it]   # eager list, reusable, brackets
sum(x*x for x in it)  # parens optional when sole arg

# --- Coroutine methods ---
g.send(v)   # resume; paused `yield` evaluates to v (prime with next() first)
g.throw(E)  # raise E at the yield point
g.close()   # raise GeneratorExit -> runs finally/cleanup

# --- itertools highlights (all lazy) ---
from itertools import count, cycle, islice, chain, groupby
count(10, 2)                 # 10, 12, 14, ...        (infinite)
cycle("ab")                  # a, b, a, b, ...        (infinite)
islice(it, start, stop, step)# slice any iterator     (lazy)
chain(a, b, c)               # concatenate iterables  (lazy)
groupby(sorted(x, key=k), key=k)  # group CONSECUTIVE -> sort first!
```

| Concept | Iterable | Iterator | Generator |
|---|---|---|---|
| Implements | `__iter__` | `__iter__` + `__next__` | both (auto) |
| Reusable? | usually yes | no (one-shot) | no (one-shot) |
| Holds state? | no | yes (position) | yes (frozen frame) |
| Memory | varies | tiny | tiny / constant |
| Example | `list`, `str`, file | `iter(list)` | `(x for x in ...)`, `def f(): yield` |

**Golden rules:** iterables make iterators; iterators are one-shot; generators are the easy way to write iterators; prefer lazy (generator) when consuming once; sort before `groupby`.
