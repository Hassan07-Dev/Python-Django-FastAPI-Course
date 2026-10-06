# 06 — Lists & Tuples

> Week 1 · Python fundamentals

## What you will learn

- How to create, index, and slice **lists** — Python's go-to ordered, mutable sequence.
- The full toolbox of list methods: `append`, `extend`, `insert`, `remove`, `pop`, `index`, `count`, `sort`, `reverse`, `clear`, `copy`.
- The crucial difference between `sorted()` (returns a new list) and `.sort()` (mutates in place), plus sorting with `key` and `reverse`.
- List concatenation (`+`), repetition (`*`), and nested lists.
- The **aliasing trap** that bites every Python beginner, and how shallow vs. deep copy (`copy` module) saves you.
- **Tuples**: immutable sequences, the single-element gotcha, packing/unpacking, starred unpacking, and returning multiple values.
- `namedtuple` from `collections` for readable, lightweight records.
- When to reach for a list vs. a tuple, how to convert between them, and the performance trade-offs.

## PHP/Laravel bridge

In PHP, the humble `array` is a chameleon: it is simultaneously a list (`[1, 2, 3]`) and a map (`['a' => 1]`). Python splits these into two distinct types — **lists** (this lesson) and **dicts** (next lesson). So a Python `list` is like a PHP **sequential/indexed array** (`[1, 2, 3]` with implicit `0,1,2` keys). The big difference: Python lists are objects with methods you call (`my_list.append(4)`) rather than standalone functions (`array_push($arr, 4)`). A Python **tuple** has no direct PHP equivalent — think of it as a "frozen", read-only list, useful where you'd otherwise pass a fixed-shape array you promise not to mutate. And where Laravel hands you Collections (`collect([1,2,3])->map(...)`), Python leans on built-ins (`map`, list comprehensions) plus these native types.

## Core concept: Creating and indexing lists

A list is an **ordered, mutable** sequence. You write it with square brackets, and it can hold values of any type — even mixed types — though in practice you usually keep them homogeneous.

```python
empty = []                      # empty list
nums = [10, 20, 30, 40, 50]     # list of ints
mixed = [1, "two", 3.0, True]   # legal, but usually a code smell
also_a_list = list("abc")       # the list() constructor: => ['a', 'b', 'c']

print(len(nums))                # => 5
print(type(nums))               # => <class 'list'>
```

### Indexing

Indices start at `0`. Negative indices count from the end (`-1` is the last item). This negative indexing is something PHP doesn't give you natively.

```python
nums = [10, 20, 30, 40, 50]

print(nums[0])    # => 10  (first)
print(nums[2])    # => 30
print(nums[-1])   # => 50  (last)
print(nums[-2])   # => 40  (second to last)

# Out-of-range access raises, it does not return null:
# print(nums[99]) # => IndexError: list index out of range
```

> **Beginner question: "Why an error instead of `null`?"** Python prefers to fail loudly. An `IndexError` tells you immediately that your assumption about the list's length was wrong, instead of silently propagating a `None` deep into your code.

### Slicing

Slicing extracts a sub-list with `list[start:stop:step]`. The `start` is **inclusive**, the `stop` is **exclusive** (like `range`). Slicing always returns a **new list**.

```python
nums = [10, 20, 30, 40, 50]

print(nums[1:4])    # => [20, 30, 40]   (indices 1,2,3 — stop 4 excluded)
print(nums[:3])     # => [10, 20, 30]   (start defaults to 0)
print(nums[2:])     # => [30, 40, 50]   (stop defaults to len)
print(nums[:])      # => [10, 20, 30, 40, 50]  (a shallow COPY of the whole list)
print(nums[::2])    # => [10, 30, 50]   (every 2nd element)
print(nums[::-1])   # => [50, 40, 30, 20, 10]  (reversed copy — a famous idiom)
print(nums[-2:])    # => [40, 50]       (last two)
```

Out-of-range slices are forgiving (unlike indexing): `nums[1:999]` just returns what exists.

```python
print(nums[1:999])  # => [20, 30, 40, 50]  -- no error
```

### Slice assignment (mutating with a slice)

Because lists are mutable, you can assign **to** a slice to splice the list in place.

```python
nums = [10, 20, 30, 40, 50]
nums[1:3] = [99]          # replace indices 1,2 with a single value
print(nums)               # => [10, 99, 40, 50]

nums[1:1] = [200, 300]    # insert without removing anything (empty slice)
print(nums)               # => [10, 200, 300, 99, 40, 50]
```

## Core concept: Mutability

Lists are **mutable** — you can change them after creation without making a new object. This is the central fact that distinguishes them from tuples and strings.

```python
colors = ["red", "green", "blue"]
colors[0] = "crimson"      # change an element in place
print(colors)              # => ['crimson', 'green', 'blue']

# Compare: strings are IMMUTABLE
word = "hello"
# word[0] = "H"            # => TypeError: 'str' object does not support item assignment
```

`id()` (Python's "object identity", roughly a memory address) proves the list object stays the same after mutation:

```python
colors = ["red", "green", "blue"]
before = id(colors)
colors.append("yellow")
print(id(colors) == before)   # => True  (same object, just modified)
```

## Core concept: List methods

These are methods you call **on** the list object. Note a recurring Python design choice: methods that mutate the list (like `append`, `sort`, `reverse`) return `None`, not the list. This prevents accidental chaining on a mutated object.

### Adding items: `append`, `extend`, `insert`

```python
items = [1, 2, 3]

items.append(4)            # add ONE item to the end
print(items)               # => [1, 2, 3, 4]

items.append([5, 6])       # appends the LIST as a single element (nested!)
print(items)               # => [1, 2, 3, 4, [5, 6]]
items.pop()                # undo that for the next example

items.extend([5, 6])       # add EACH item of the iterable
print(items)               # => [1, 2, 3, 4, 5, 6]

items.insert(0, 0)         # insert at index 0
print(items)               # => [0, 1, 2, 3, 4, 5, 6]
```

> **`append` vs `extend` is a classic interview trap.** `append(x)` adds `x` as a single element. `extend(iterable)` iterates and adds each element. `[1,2].append([3,4])` → `[1,2,[3,4]]`; `[1,2].extend([3,4])` → `[1,2,3,4]`. Note `extend` works with *any* iterable (`"ab"` → `'a','b'`), while `+=` on a list behaves like `extend`.

### Removing items: `remove`, `pop`, `clear`

```python
items = [10, 20, 30, 20, 40]

items.remove(20)           # removes the FIRST matching VALUE
print(items)               # => [10, 30, 20, 40]
# items.remove(999)        # => ValueError if not found

last = items.pop()         # removes & RETURNS the last item
print(last, items)         # => 40 [10, 30, 20]

second = items.pop(1)      # pop by INDEX, returns it
print(second, items)       # => 30 [10, 20]

items.clear()              # remove everything
print(items)               # => []
```

There is also the `del` statement, which removes by index or slice (and can't be chained or return a value):

```python
nums = [10, 20, 30, 40]
del nums[0]                # => [20, 30, 40]
del nums[1:]               # => [20]
print(nums)                # => [20]
```

| Goal | Use |
|------|-----|
| Remove by **value** | `remove(value)` (raises `ValueError` if absent) |
| Remove by **index**, keep the value | `pop(index)` |
| Remove last, keep the value | `pop()` |
| Remove by index/slice, discard | `del list[i]` |
| Remove everything | `clear()` |

### Searching: `index`, `count`, `in`

```python
letters = ['a', 'b', 'c', 'b', 'a']

print(letters.index('b'))      # => 1   (first match)
print(letters.index('b', 2))   # => 3   (search starting at index 2)
print(letters.count('a'))      # => 2
print('c' in letters)          # => True   (membership test — very idiomatic)
print('z' not in letters)      # => True
# letters.index('z')           # => ValueError if not present
```

> Use `in` to *check existence* (it never raises). Use `index()` only when you actually need the position, and guard it or wrap in `try/except` because it raises on a miss.

### Sorting: `.sort()` vs `sorted()`

This distinction is asked in interviews constantly.

- `list.sort()` — sorts the list **in place** and returns `None`.
- `sorted(iterable)` — returns a **new sorted list**, leaves the original untouched, and works on *any* iterable (tuples, sets, dict keys...).

```python
nums = [3, 1, 4, 1, 5, 9, 2]

new_list = sorted(nums)        # new list, original unchanged
print(new_list)                # => [1, 1, 2, 3, 4, 5, 9]
print(nums)                    # => [3, 1, 4, 1, 5, 9, 2]  (untouched)

nums.sort()                    # mutates in place, returns None
print(nums)                    # => [1, 1, 2, 3, 4, 5, 9]

result = nums.sort()           # COMMON BUG:
print(result)                  # => None  (sort() does not return the list!)
```

Both accept `key` (a function deciding the sort value) and `reverse`:

```python
words = ["banana", "Apple", "cherry", "fig"]

print(sorted(words))                       # => ['Apple', 'banana', 'cherry', 'fig']
                                           #    (uppercase sorts before lowercase by ASCII!)
print(sorted(words, key=str.lower))        # => ['Apple', 'banana', 'cherry', 'fig']
                                           #    case-insensitive
print(sorted(words, key=len))              # => ['fig', 'Apple', 'banana', 'cherry']
print(sorted(words, key=len, reverse=True))# => ['banana', 'cherry', 'Apple', 'fig']

# Sort a list of tuples/records by a chosen field:
people = [("Alice", 30), ("Bob", 25), ("Cara", 30)]
print(sorted(people, key=lambda p: p[1]))               # by age
# => [('Bob', 25), ('Alice', 30), ('Cara', 30)]
print(sorted(people, key=lambda p: (-p[1], p[0])))      # age desc, then name asc
# => [('Alice', 30), ('Cara', 30), ('Bob', 25)]
```

> **Stability:** Python's sort is *stable* — equal elements keep their original relative order. That's why you can sort by a secondary key first, then the primary key, to get multi-level ordering. The `(-p[1], p[0])` tuple trick sorts by multiple criteria in one pass.

### `reverse`, `copy`

```python
nums = [1, 2, 3]
nums.reverse()             # in place, returns None
print(nums)                # => [3, 2, 1]

shallow = nums.copy()      # a (shallow) copy — same as nums[:] or list(nums)
shallow.append(99)
print(nums)                # => [3, 2, 1]   (original safe for top-level changes)
print(shallow)             # => [3, 2, 1, 99]
```

## Core concept: Concatenation, repetition, and nesting

```python
a = [1, 2]
b = [3, 4]

print(a + b)        # => [1, 2, 3, 4]   (NEW list via concatenation)
print(a * 3)        # => [1, 2, 1, 2, 1, 2]   (repetition)
print([0] * 5)      # => [0, 0, 0, 0, 0]   (handy for fixed-size init)

a += b              # in-place extend (mutates a)
print(a)            # => [1, 2, 3, 4]
```

### Nested lists (lists of lists)

```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
]
print(matrix[1])        # => [4, 5, 6]   (the second row)
print(matrix[1][2])     # => 6           (row 1, column 2)

# Iterate rows and columns:
for row in matrix:
    for value in row:
        print(value, end=" ")
# => 1 2 3 4 5 6 7 8 9
```

> **The repetition trap with nested lists.** Never build a grid with `[[0] * 3] * 3`. The outer `* 3` copies the *same inner list reference* three times, so editing one row edits all of them:

```python
bad = [[0] * 3] * 3
bad[0][0] = 99
print(bad)          # => [[99, 0, 0], [99, 0, 0], [99, 0, 0]]  -- YIKES

good = [[0] * 3 for _ in range(3)]   # comprehension makes a fresh inner list each time
good[0][0] = 99
print(good)         # => [[99, 0, 0], [0, 0, 0], [0, 0, 0]]  -- correct
```

## Core concept: The aliasing trap & shallow vs. deep copy

This is the single most important concept in this lesson — and a guaranteed interview question. In Python, a variable holds a **reference** to an object, not the object itself. Assigning one list to another variable does **not** copy it; both names point at the same object (an *alias*).

```python
original = [1, 2, 3]
alias = original          # NOT a copy — same object, two names

alias.append(4)
print(original)           # => [1, 2, 3, 4]   (surprise! original changed)
print(original is alias)  # => True           (same object identity)
```

> **Laravel contrast:** PHP arrays are *copy-on-assign* by value, so `$b = $a; $b[] = 4;` leaves `$a` unchanged. Python is the opposite — assignment shares the reference. This trips up nearly every PHP dev moving to Python.

### Shallow copy — copies the top level only

A shallow copy makes a new outer list, but the *elements inside* are still shared references.

```python
import copy

original = [[1, 2], [3, 4]]

shallow = original.copy()          # also: original[:], list(original), copy.copy(original)
shallow.append([5, 6])             # top-level change is isolated
print(original)                    # => [[1, 2], [3, 4]]   (safe)

shallow[0][0] = 99                 # but the INNER list is shared!
print(original)                    # => [[99, 2], [3, 4]]  -- leaked through
```

### Deep copy — copies everything, recursively

`copy.deepcopy()` recursively duplicates every nested object, so the copy is fully independent.

```python
import copy

original = [[1, 2], [3, 4]]
deep = copy.deepcopy(original)

deep[0][0] = 99
print(original)   # => [[1, 2], [3, 4]]   (untouched — fully independent)
print(deep)       # => [[99, 2], [3, 4]]
```

| Technique | New outer? | New inner objects? | Cost |
|-----------|-----------|--------------------|------|
| `b = a` (alias) | No | No | Free, but shares everything |
| `a.copy()` / `a[:]` / `list(a)` | Yes | No (shared) | Cheap |
| `copy.deepcopy(a)` | Yes | Yes (recursive) | Expensive |

> **Rule of thumb:** for a flat list of immutables (ints, strings), a shallow copy is all you ever need. Reach for `deepcopy` only when the list contains mutable objects (other lists/dicts) that must not be shared.

## Core concept: Tuples & immutability

A tuple is an **ordered, immutable** sequence. Written with parentheses (or often no brackets at all). Once created, you cannot add, remove, or change elements.

```python
point = (3, 4)
print(point[0])        # => 3   (same indexing/slicing as lists)
print(point[-1])       # => 4
print(len(point))      # => 2

# point[0] = 99        # => TypeError: 'tuple' object does not support item assignment

# Tuples support read-only operations:
print(point + (5,))    # => (3, 4, 5)   (concatenation makes a NEW tuple)
print(point * 2)       # => (3, 4, 3, 4)
print(point.count(3))  # => 1
print(point.index(4))  # => 1
print(3 in point)      # => True
```

Tuples have only two methods — `count` and `index` — precisely *because* they can't be mutated.

### The single-element tuple gotcha

This is one of the most famous Python footguns and a frequent interview question. It's the **trailing comma**, not the parentheses, that makes a tuple.

```python
not_a_tuple = (5)      # just the integer 5 in parentheses!
print(type(not_a_tuple))   # => <class 'int'>

a_tuple = (5,)         # the comma makes it a tuple
print(type(a_tuple))       # => <class 'tuple'>

# Even without parentheses, the comma is what counts:
also_tuple = 5,
print(type(also_tuple))    # => <class 'tuple'>

empty = ()             # empty tuple (no comma needed here)
print(type(empty))         # => <class 'tuple'>
```

> **The hidden bug this causes:** `return (value)` returns the value, but `return value,` (or a stray comma anywhere) silently returns a one-element tuple. Watch for accidental trailing commas in `print("done"),` style code.

### "Immutable" has an asterisk

A tuple's *structure* is fixed, but if it contains a mutable object, that object can still change. The tuple guarantees its slots point at the same objects — not that those objects are frozen.

```python
data = (1, [2, 3])
# data[1] = [9]        # => TypeError (can't reassign the slot)
data[1].append(4)      # but the list it points to IS mutable
print(data)            # => (1, [2, 3, 4])
```

This is also why a tuple is only **hashable** (usable as a dict key / set member) if *all* its elements are hashable:

```python
print(hash((1, 2)))            # works fine
# hash((1, [2, 3]))            # => TypeError: unhashable type: 'list'
```

## Core concept: Packing, unpacking & multiple return values

### Packing and unpacking

```python
# Packing: bare commas build a tuple
coords = 10, 20, 30
print(coords)              # => (10, 20, 30)

# Unpacking: spread a sequence across variables
x, y, z = coords
print(x, y, z)             # => 10 20 30

# Count must match, or you get a ValueError:
# a, b = (1, 2, 3)         # => ValueError: too many values to unpack

# The classic swap — no temp variable needed:
a, b = 1, 2
a, b = b, a
print(a, b)                # => 2 1
```

### Starred unpacking (the `*` catch-all)

`*name` absorbs "the rest" into a list. Only one starred target is allowed.

```python
first, *middle, last = [1, 2, 3, 4, 5]
print(first)    # => 1
print(middle)   # => [2, 3, 4]   (always a LIST, even when from a tuple)
print(last)     # => 5

head, *tail = [10, 20, 30]
print(head, tail)   # => 10 [20, 30]

*init, final = (1, 2, 3)
print(init, final)  # => [1, 2] 3

# Use _ for values you intend to ignore:
name, _, age = ("Alice", "ignored", 30)
print(name, age)    # => Alice 30
```

### Returning multiple values

Python functions "return multiple values" by returning a tuple — and the caller unpacks it. This is far cleaner than PHP, where you'd return an array and index into it.

```python
def min_max(numbers):
    return min(numbers), max(numbers)   # packs into a tuple

low, high = min_max([4, 1, 7, 3])       # unpacks at the call site
print(low, high)        # => 1 7

# Or keep the tuple whole:
result = min_max([4, 1, 7, 3])
print(result)           # => (1, 7)
print(result[0])        # => 1
```

This pattern also powers `enumerate` and `dict.items()`:

```python
for i, color in enumerate(["red", "green"], start=1):
    print(i, color)
# => 1 red
# => 2 green
```

## Core concept: namedtuple — readable records

A plain tuple is positional: `person[0]` tells you nothing about what index `0` means. `collections.namedtuple` gives you a lightweight, immutable record where fields have names — perfect for returning structured data without a full class.

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])

p = Point(3, 4)
print(p.x, p.y)        # => 3 4   (access by name!)
print(p[0])            # => 3     (still indexable like a tuple)
print(p)               # => Point(x=3, y=4)   (nice repr)

# It IS a tuple, so it unpacks and is immutable:
x, y = p
print(x, y)            # => 3 4
# p.x = 99             # => AttributeError: can't set attribute

# Make a modified copy with _replace (returns a NEW namedtuple):
moved = p._replace(x=10)
print(moved)           # => Point(x=10, y=4)
print(p)               # => Point(x=3, y=4)   (original untouched)

# Convert to a dict:
print(p._asdict())     # => {'x': 3, 'y': 4}
```

> For more elaborate records you'll later prefer `@dataclass` or Pydantic models (covered in the OOP and FastAPI weeks). But `namedtuple` remains the lightest way to add field names to tuple-shaped data. The newer `typing.NamedTuple` class syntax is a typed equivalent worth knowing too.

## Core concept: list vs. tuple — when to use which

| | `list` | `tuple` |
|---|--------|---------|
| Mutable? | Yes | No |
| Syntax | `[1, 2, 3]` | `(1, 2, 3)` |
| Use as dict key / set member? | No (unhashable) | Yes (if all elements hashable) |
| Typical use | Homogeneous, variable-length collection you'll modify | Fixed, heterogeneous record; "this won't change" |
| Methods | Many (append, sort, ...) | Only `count`, `index` |
| Memory / speed | Slightly larger, slightly slower to create | Smaller, faster to create; can be cached |

**Mental model:** use a **list** for "a bunch of the same thing" (a list of users, of prices) where the count varies and you'll add/remove. Use a **tuple** for "one thing with several parts" (a coordinate, an RGB color, a database row) where the shape is fixed and meaning is positional. Immutability also signals intent ("don't change this") and lets the value be a dict key.

### Converting between them

```python
nums_list = [1, 2, 3]
nums_tuple = tuple(nums_list)      # => (1, 2, 3)
back_to_list = list(nums_tuple)    # => [1, 2, 3]

# tuple()/list() accept ANY iterable:
print(list("abc"))                 # => ['a', 'b', 'c']
print(tuple(range(3)))             # => (0, 1, 2)
print(list({1, 2, 3}))             # => [1, 2, 3]  (from a set)

# Common trick: dedupe a list while throwing away order via set, then back to list:
print(list(set([1, 1, 2, 3, 3]))) # => [1, 2, 3]  (order not guaranteed!)
```

### Performance notes

- Tuples are **faster to construct** and use **less memory** than equivalent lists, because their fixed size lets CPython optimize allocation. For large numbers of small fixed records, tuples add up.
- Indexing (`x[i]`), `len()`, and iteration are O(1)/O(n) and essentially the same speed for both.
- `append` is **amortized O(1)** (lists over-allocate to avoid copying on every add). `insert(0, x)` and `pop(0)` are **O(n)** because every other element shifts — if you need fast front operations, use `collections.deque` instead.
- Membership testing (`x in my_list`) is **O(n)**. If you do many lookups, convert to a `set` (O(1) average) — covered in the sets lesson.

```python
import sys
print(sys.getsizeof([1, 2, 3]))    # => e.g. 88  (bytes; exact number varies)
print(sys.getsizeof((1, 2, 3)))    # => e.g. 64  (tuple is smaller)
```

## Common mistakes & gotchas

- **Aliasing instead of copying:** `b = a` shares the same list. Use `a.copy()`, `a[:]`, or `list(a)` for a shallow copy, and `copy.deepcopy(a)` when nested mutables must be independent.
- **Expecting `.sort()`/`.append()`/`.reverse()` to return the list.** They mutate in place and return `None`. `x = my_list.sort()` sets `x` to `None`.
- **`append` vs `extend`:** `append` adds one element (possibly a whole list as one item); `extend` adds each element of an iterable.
- **The `[[0]*3]*3` nested-list trap:** the inner list is shared. Use a comprehension: `[[0]*3 for _ in range(3)]`.
- **Single-element tuple without a comma:** `(5)` is an `int`; you need `(5,)`. Conversely, a stray trailing comma turns a value into a one-element tuple by accident.
- **`list.index(x)` / `list.remove(x)` raise** `ValueError` when `x` is absent. Check with `if x in list:` first, or use `try/except`.
- **Mutating a list while iterating over it** skips or repeats elements. Iterate over a copy (`for x in items[:]`) or build a new list instead.
- **Using `+` in a loop to build a big list** is O(n^2) (each `+` copies). Use `append` in a loop, or a list comprehension.
- **Thinking tuples are deeply immutable:** a tuple containing a list can still have that list mutated; such a tuple is also unhashable.

## Best practices

- Prefer **list comprehensions** over manual `append` loops for building lists from other iterables — they're faster and more readable (covered next week, but start noticing them).
- Use **tuples for fixed-shape records** and **lists for variable collections**; let the type signal mutability intent.
- Reach for **`namedtuple`** (or a dataclass) the moment positional tuple access (`row[3]`) becomes unclear.
- Use `sorted()` when you need to keep the original; use `.sort()` when in-place mutation is intended and you don't need the return value.
- For multi-key sorting, pass a `key` returning a tuple: `key=lambda r: (r.last, r.first)`.
- Use `in` for existence checks; reserve `index()` for when you genuinely need a position.
- Default to **shallow copy**; use `deepcopy` only when nesting actually requires it (it's slow).
- For heavy front-of-list insertion/removal, use `collections.deque`, not a list.

## Interview Q&A

**Q: What's the difference between a list and a tuple, and when would you use each?**
**A:** A list is mutable and written with `[]`; a tuple is immutable and written with `()`. Use a list for a variable-length collection of similar items you'll modify (a list of users). Use a tuple for a fixed-shape, often heterogeneous record (a coordinate, a DB row) where you want to signal "this won't change." Tuples are also hashable (if their contents are), so only tuples can be dict keys or set members.

**Q: What's the difference between `list.sort()` and `sorted(list)`?**
**A:** `list.sort()` sorts in place and returns `None`; it only works on lists. `sorted(iterable)` returns a new sorted list and leaves the input untouched, and it accepts any iterable. Both support `key` and `reverse`. A common bug is `x = my_list.sort()`, which sets `x` to `None`.

**Q: Explain the difference between `append` and `extend`.**
**A:** `append(x)` adds `x` as a single element — `[1,2].append([3,4])` gives `[1,2,[3,4]]`. `extend(iterable)` iterates and adds each element — `[1,2].extend([3,4])` gives `[1,2,3,4]`. `extend` accepts any iterable and behaves like `+=`.

**Q: What is the aliasing trap, and how do you avoid it?**
**A:** Assignment in Python copies the reference, not the object: `b = a` makes `b` and `a` point at the *same* list, so mutating one affects the other. To get an independent copy, use `a.copy()`, `a[:]`, or `list(a)` for a shallow copy, and `copy.deepcopy(a)` when the list contains nested mutable objects.

**Q: Shallow copy vs. deep copy?**
**A:** A shallow copy (`a.copy()`, `a[:]`, `copy.copy(a)`) creates a new outer list but shares the inner element objects — mutating a nested list leaks into both. A deep copy (`copy.deepcopy(a)`) recursively duplicates everything, so the copy is fully independent. Deep copy is correct but slower, so use it only when nested mutables must not be shared.

**Q: How do you create a single-element tuple, and why is `(5)` not one?**
**A:** Parentheses are just grouping; the **comma** makes a tuple. So `(5)` is the integer `5`, while `(5,)` (or `5,`) is a one-element tuple. The empty tuple `()` is the one case that needs no comma.

**Q: Are tuples truly immutable?**
**A:** The tuple's structure is immutable — you can't reassign, add, or remove slots. But if a slot holds a mutable object (like a list), that object can still be mutated. Such a tuple is also unhashable, because hashability requires all contents to be hashable.

**Q: What is tuple unpacking, including the starred form?**
**A:** Unpacking assigns sequence elements to multiple variables: `a, b = (1, 2)`. The counts must match. Starred unpacking lets one variable absorb the rest as a list: `first, *rest = [1,2,3]` gives `first=1, rest=[2,3]`. It also enables the no-temp swap `a, b = b, a` and clean multiple return values.

**Q: How do Python functions return multiple values?**
**A:** They return a tuple (often via implicit packing: `return low, high`), and the caller unpacks it: `low, high = f()`. There's no special multi-return syntax — it's just tuple packing/unpacking, which is why `enumerate()` and `dict.items()` give you tuples you can unpack in a loop.

**Q: What is a `namedtuple` and why use it?**
**A:** `collections.namedtuple` creates a tuple subclass with named fields, so you access data as `p.x` instead of `p[0]` while keeping tuple behavior (immutable, indexable, unpackable, with a clean repr). It's ideal for small immutable records where a full class is overkill. `_replace()` returns a modified copy and `_asdict()` converts it to a dict.

**Q: Why is `[[0]*3]*3` dangerous for building a 2D grid?**
**A:** The outer `*3` repeats the *same inner list reference* three times, so editing one row edits all rows. Build it with a comprehension instead — `[[0]*3 for _ in range(3)]` — which creates a fresh inner list per row.

**Q: What's the time complexity of common list operations?**
**A:** Indexing, `len`, and `append` (amortized) are O(1). `insert(0, x)`, `pop(0)`, `remove`, and `x in list` are O(n) because they shift or scan elements. For fast front operations use `collections.deque`; for fast membership use a `set`.

## Practice exercises

1. **Slicing drill.** Given `data = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]`, produce (a) the last three elements, (b) every other element starting at index 1, (c) a reversed copy, all using slicing only.

2. **Top scorers.** Given `scores = [("Ann", 88), ("Bo", 95), ("Cy", 88), ("Di", 72)]`, return the names sorted by score descending, breaking ties alphabetically by name.

3. **Safe copy.** Write a function `independent_grid(grid)` that returns a copy of a 2D list such that mutating the copy never affects the original. Demonstrate it does *not* suffer the shallow-copy leak.

4. **Min/max/avg.** Write a function `stats(numbers)` that returns the minimum, maximum, and average as a tuple, then unpack the result at the call site.

5. **Dedupe preserving order.** Write `dedupe(items)` that removes duplicates from a list while preserving first-seen order (so `[3,1,3,2,1]` → `[3,1,2]`). (Hint: a `set` for "seen" plus a result list.)

6. **Named record.** Define a `namedtuple` called `Book` with fields `title`, `author`, `year`. Create one, print it, then create a modified copy with a new `year` using `_replace`, proving the original is unchanged.

<details>
<summary>Solutions</summary>

```python
# 1. Slicing drill
data = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
print(data[-3:])       # => [7, 8, 9]
print(data[1::2])      # => [1, 3, 5, 7, 9]
print(data[::-1])      # => [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]


# 2. Top scorers (score desc, then name asc)
scores = [("Ann", 88), ("Bo", 95), ("Cy", 88), ("Di", 72)]
ranked = sorted(scores, key=lambda s: (-s[1], s[0]))
print([name for name, _ in ranked])
# => ['Bo', 'Ann', 'Cy', 'Di']


# 3. Safe copy (deep copy avoids the shallow leak)
import copy

def independent_grid(grid):
    return copy.deepcopy(grid)

original = [[1, 2], [3, 4]]
copied = independent_grid(original)
copied[0][0] = 99
print(original)   # => [[1, 2], [3, 4]]   (untouched)
print(copied)     # => [[99, 2], [3, 4]]


# 4. Min/max/avg
def stats(numbers):
    return min(numbers), max(numbers), sum(numbers) / len(numbers)

lo, hi, avg = stats([4, 1, 7, 3])
print(lo, hi, avg)   # => 1 7 3.75


# 5. Dedupe preserving order
def dedupe(items):
    seen = set()
    result = []
    for item in items:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

print(dedupe([3, 1, 3, 2, 1]))   # => [3, 1, 2]


# 6. Named record
from collections import namedtuple

Book = namedtuple("Book", ["title", "author", "year"])
b = Book("Dune", "Herbert", 1965)
print(b)                          # => Book(title='Dune', author='Herbert', year=1965)

reissue = b._replace(year=2021)
print(reissue)                    # => Book(title='Dune', author='Herbert', year=2021)
print(b)                          # => Book(title='Dune', author='Herbert', year=1965)
```

</details>

## Cheat sheet

```python
# CREATE
xs = [1, 2, 3]          # list (mutable)
t  = (1, 2, 3)          # tuple (immutable)
one = (5,)              # single-element tuple — COMMA matters
empty_t = ()            # empty tuple

# INDEX / SLICE  (start inclusive, stop exclusive)
xs[0]; xs[-1]; xs[1:3]; xs[:2]; xs[::2]; xs[::-1]   # reversed copy

# ADD / REMOVE (lists)
xs.append(4)            # one item at end
xs.extend([5, 6])       # each item of iterable  (== xs += [5,6])
xs.insert(0, 99)        # at index
xs.remove(99)           # first matching VALUE (ValueError if absent)
v = xs.pop()            # last, returns it
v = xs.pop(0)           # by index, returns it
del xs[0]               # by index/slice
xs.clear()              # empty it

# SEARCH
xs.index(2)             # first position (ValueError if absent)
xs.count(2)             # occurrences
2 in xs                 # membership (never raises) — preferred

# SORT / REVERSE
xs.sort()                       # in place, returns None
xs.sort(key=len, reverse=True)
new = sorted(xs, key=str.lower) # new list, any iterable
xs.reverse()                    # in place

# COPY
b = a.copy()           # shallow (also a[:], list(a))
import copy
b = copy.deepcopy(a)   # deep (independent nested objects)
b = a                  # ALIAS — NOT a copy!

# CONCAT / REPEAT
[1,2] + [3,4]          # => [1, 2, 3, 4]
[0] * 3                # => [0, 0, 0]
[[0]*3 for _ in range(3)]   # safe 2D grid (NOT [[0]*3]*3)

# PACK / UNPACK
a, b = 1, 2
a, b = b, a                    # swap
first, *rest = [1, 2, 3]       # rest => [2, 3]
def mm(n): return min(n), max(n)
lo, hi = mm([3, 1, 2])

# NAMEDTUPLE
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4); p.x; p._replace(x=9); p._asdict()

# CONVERT
list((1,2)); tuple([1,2]); list("abc"); list(range(3))
```

| Operation | List | Tuple |
|-----------|------|-------|
| Mutable | Yes | No |
| Literal | `[ ]` | `( )` |
| Hashable (dict key) | No | Yes (if contents are) |
| Methods | many | `count`, `index` |
| Copy independence | aliasing trap! use `.copy()` / `deepcopy` | n/a (immutable) |
