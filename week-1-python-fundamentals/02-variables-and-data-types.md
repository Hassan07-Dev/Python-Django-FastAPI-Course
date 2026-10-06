# 02 — Variables & Data Types

> Week 1 · Python Fundamentals

## What you will learn

- How Python's **dynamic typing** works, and how it differs from PHP's type juggling.
- The rules for **variable assignment**, naming, and the `snake_case` / `UPPER_CASE` conventions.
- The **core built-in types**: `int`, `float`, `complex`, `bool`, `str`, and `NoneType`.
- **Multiple assignment** and the famous one-line swap.
- The introspection trio: `type()`, `isinstance()`, and `id()`.
- **Mutability vs immutability** — and why it bites people in interviews and production.
- **Numeric literals** (underscores, hex/bin/oct), arithmetic, and the `/` vs `//` division gotcha.
- **Explicit type conversion** with `int()`, `str()`, `float()`, `bool()`.
- A preview of **truthiness**.
- The big idea that **everything in Python is an object**.

## PHP/Laravel bridge

In PHP you already write `$total = 5;` with no type declaration, so dynamic typing will feel familiar. The key differences: Python has **no `$` sigil** and **no statement-ending `;` required**; you just write `total = 5`. PHP loves *implicit* coercion (`"5" + 3 == 8`, `"abc" == 0` in old versions) — Python deliberately refuses most of that (`"5" + 3` is a `TypeError`), which trades convenience for far fewer surprise bugs. PHP's `define('PI', 3.14)` or `const` gives you true constants; Python has **no real constants**, only an `ALL_CAPS` naming convention that says "please don't reassign this." And where PHP has `null`, Python has `None` — a real singleton object, not just an absence.

## Core concept: dynamic typing (and the type-juggling difference)

A variable in Python is just a **name bound to an object**. The *object* carries the type, not the name. You never declare a type, and the same name can point at different types over its life:

```python
x = 42          # x now refers to an int object
x = "hello"     # totally legal: x now refers to a str object
x = [1, 2, 3]   # now a list
print(x)        # => [1, 2, 3]
```

This is **dynamic typing**: types are checked at runtime, attached to values. Contrast with static typing (Java, Go) where the *variable* has a fixed type.

But Python is also **strongly typed** — and this is where it diverges sharply from PHP. Python will not silently coerce unrelated types in an operation:

```python
print("5" + 3)
# => TypeError: can only concatenate str (not "int") to str
```

PHP would happily juggle that. In Python you must convert explicitly:

```python
print("5" + str(3))   # => "53"   (string concatenation)
print(int("5") + 3)   # => 8      (integer addition)
```

| Aspect | PHP | Python |
|---|---|---|
| Type on the variable? | No (dynamic) | No (dynamic) |
| Implicit cross-type coercion | Yes, aggressively | No (strong typing) |
| `"5" + 3` | `8` | `TypeError` |
| `null` / absence | `null` | `None` |
| Variable sigil | `$x` | `x` |

> **Mental model:** "dynamically typed" = you don't declare types. "Strongly typed" = Python won't guess what you meant across types. Python is both.

## Core concept: assignment, naming rules, and conventions

Assignment uses a single `=`. No `var`, no `let`, no `$`:

```python
user_count = 10
api_base_url = "https://api.example.com"
is_active = True
```

**Naming rules (enforced by the interpreter):**

- Must start with a letter or underscore (`_`), followed by letters, digits, or underscores.
- Case-sensitive: `name` and `Name` are different.
- Cannot be a reserved keyword (`class`, `def`, `return`, `None`, `True`, `if`, ...).

```python
2cool = 1      # SyntaxError: cannot start with a digit
class = "x"    # SyntaxError: 'class' is a keyword
my_var = 1     # fine
_private = 1   # fine (leading underscore = "internal" by convention)
```

**Naming conventions (PEP 8 — not enforced, but everyone follows them):**

| Thing | Convention | Example |
|---|---|---|
| Variables, functions | `snake_case` | `total_price`, `get_user()` |
| "Constants" | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `PI` |
| Classes | `PascalCase` | `UserAccount` |
| "Internal" / private | leading `_` | `_cache`, `_helper()` |

Coming from Laravel where camelCase methods (`$user->firstName`) are everywhere, retrain your fingers: idiomatic Python is `snake_case` for variables and functions.

### There are no real constants

```python
MAX_CONNECTIONS = 100   # convention says "don't reassign me"
MAX_CONNECTIONS = 200   # Python lets you do this anyway — no error
```

Python will *not* stop you. The `UPPER_CASE` name is a social contract among developers, like a sign that says "wet paint." (`typing.Final` and `enum.Enum` exist to signal intent more strongly, but even `Final` is only checked by type-checkers like `mypy`, not at runtime.)

## Core concept: the built-in scalar types

### `int` — integers with arbitrary precision

Python integers have **no fixed size and never overflow**. This is a genuinely big deal coming from PHP/C where ints are 64-bit:

```python
big = 2 ** 200
print(big)
# => 1606938044258990275541962092341162602522202993782792835301376
print(type(big))   # => <class 'int'>
```

No `BigInteger` library needed — the language handles it natively.

### `float` — double-precision floating point

```python
price = 19.99
ratio = 1.0
print(type(price))   # => <class 'float'>
```

Floats are IEEE-754 doubles, so the usual binary-fraction caveats apply:

```python
print(0.1 + 0.2)            # => 0.30000000000000004
print(0.1 + 0.2 == 0.3)     # => False
```

For money, use `decimal.Decimal` instead of `float` (more on that later in the course).

### `complex` — built-in complex numbers

Rarely used in web work, but it's a core type. The imaginary part uses `j`:

```python
z = 2 + 3j
print(z.real)        # => 2.0
print(z.imag)        # => 3.0
print(type(z))       # => <class 'complex'>
```

### `bool` — `True` and `False` (note the capital letters)

```python
is_admin = True
has_paid = False
```

A subtle but interview-worthy fact: in Python, **`bool` is a subclass of `int`**. `True` equals `1` and `False` equals `0`:

```python
print(True + True)          # => 2
print(isinstance(True, int))  # => True
print(True == 1)            # => True
```

### `str` — text (immutable Unicode)

Strings are Unicode by default and use single or double quotes interchangeably:

```python
name = "Ada"
greeting = 'Hello'
print(type(name))   # => <class 'str'>
```

Strings are **immutable** — you cannot change a character in place (covered below and in the strings lesson).

### `NoneType` — the `None` singleton

`None` represents "no value" — Python's answer to PHP's `null`. There is exactly **one** `None` object in the entire program:

```python
result = None
print(type(result))       # => <class 'NoneType'>
print(result is None)     # => True  (the idiomatic check)
```

Use `is None` / `is not None`, **not** `== None`, to test for it (it's faster and unambiguous because `None` is a singleton).

| Type | Example literal | `type()` result | Mutable? |
|---|---|---|---|
| `int` | `42`, `2**200` | `<class 'int'>` | No |
| `float` | `3.14`, `1.0` | `<class 'float'>` | No |
| `complex` | `2 + 3j` | `<class 'complex'>` | No |
| `bool` | `True`, `False` | `<class 'bool'>` | No |
| `str` | `"hi"` | `<class 'str'>` | No |
| `NoneType` | `None` | `<class 'NoneType'>` | n/a |

## Core concept: multiple assignment and swapping

Python lets you bind several names at once. This is **tuple unpacking** under the hood.

```python
# Assign different values to several names
x, y, z = 1, 2, 3
print(x, y, z)   # => 1 2 3

# Assign the SAME value to several names (chained)
a = b = c = 0
print(a, b, c)   # => 0 0 0
```

The elegant swap — no temp variable, the thing PHP people always notice first:

```python
a, b = 10, 20
a, b = b, a
print(a, b)      # => 20 10
```

In PHP you'd typically need `[$a, $b] = [$b, $a];` (or a temp). Python's version reads cleanly because the **right-hand side is fully evaluated first** (into a tuple), then unpacked into the names on the left.

You can also use a starred name to grab "the rest":

```python
first, *rest = [1, 2, 3, 4]
print(first)   # => 1
print(rest)    # => [2, 3, 4]
```

## Core concept: introspection with `type()`, `isinstance()`, and `id()`

### `type(obj)` — what exactly is this?

```python
print(type(42))        # => <class 'int'>
print(type(3.14))      # => <class 'float'>
print(type("hi"))      # => <class 'str'>
print(type(None))      # => <class 'NoneType'>
```

### `isinstance(obj, cls)` — is this an instance of (or subclass of)?

Prefer `isinstance()` over `type(x) == ...` for type checks, because it respects inheritance:

```python
print(isinstance(42, int))          # => True
print(isinstance(True, int))        # => True   (bool subclasses int!)
print(isinstance(3.14, (int, float)))  # => True  (tuple = "any of these")
```

`type(x) == int` would return `False` for a subclass, which is usually not what you want.

### `id(obj)` — the object's identity

`id()` returns a unique integer identifying the object for its lifetime (in CPython, its memory address). It's how `is` works under the hood:

```python
a = [1, 2, 3]
b = a              # same object, b is just another name for it
c = [1, 2, 3]      # a different object that happens to be equal

print(id(a) == id(b))   # => True   (same object)
print(a is b)           # => True
print(id(a) == id(c))   # => False  (different objects)
print(a == c)           # => True   (but equal in value)
```

> **Key distinction:** `==` compares **values** (equality); `is` compares **identity** (same object in memory). Use `is` only for singletons like `None`, `True`, `False`.

### Small-integer caching (a famous gotcha)

CPython pre-caches small integers (roughly `-5` to `256`) and interns short strings, so identity for those can surprise you:

```python
x = 256
y = 256
print(x is y)     # => True   (cached — same object)

m = 257
n = 257
print(m is n)     # => False  (NOT cached — usually two objects)
print(m == n)     # => True   (always — values are equal)
```

This is exactly why you **never** use `is` to compare numbers or strings for equality. Use `==`.

## Core concept: mutability vs immutability

Some objects can be changed in place (**mutable**); others cannot (**immutable**). For an immutable object, any "change" actually creates a **new object**.

| Immutable | Mutable |
|---|---|
| `int`, `float`, `complex`, `bool` | `list` |
| `str` | `dict` |
| `tuple` | `set` |
| `frozenset` | `bytearray` |
| `bytes` | (instances of most custom classes) |

Strings are immutable:

```python
s = "hello"
s[0] = "H"
# => TypeError: 'str' object does not support item assignment

s = s.upper()      # this works — it builds a NEW string and rebinds s
print(s)           # => HELLO
```

Lists are mutable — they change in place:

```python
nums = [1, 2, 3]
print(id(nums))    # e.g. => 140234...
nums.append(4)
print(nums)        # => [1, 2, 3, 4]
print(id(nums))    # same id — it was mutated, not replaced
```

### Why mutability matters: shared references

Because variables are **names bound to objects**, two names can point at the *same* mutable object. Mutating through one name is visible through the other:

```python
a = [1, 2, 3]
b = a            # b and a refer to the SAME list
b.append(99)
print(a)         # => [1, 2, 3, 99]   <- surprise! a changed too
```

With immutable objects there's no such surprise, because you can't mutate them — reassignment just rebinds the name:

```python
x = 5
y = x
y = y + 1        # creates a new int 6, rebinds y; x is untouched
print(x, y)      # => 5 6
```

This distinction underlies a huge class of bugs (and a classic interview question): **mutable default arguments**, copying lists vs aliasing them, and passing mutable objects into functions. Keep it front of mind.

## Core concept: numeric literals and arithmetic

### Readable and alternative-base literals

```python
million = 1_000_000        # underscores are visual separators, ignored by Python
print(million)             # => 1000000

hexval = 0xFF              # hexadecimal => 255
binval = 0b1010            # binary      => 10
octval = 0o17             # octal       => 15
print(hexval, binval, octval)   # => 255 10 15

sci = 1.5e3               # scientific notation => 1500.0
print(sci)                # => 1500.0
```

### Operators

```python
print(7 + 2)     # => 9    addition
print(7 - 2)     # => 5    subtraction
print(7 * 2)     # => 14   multiplication
print(7 ** 2)    # => 49   exponentiation (PHP: ** as well, or pow())
print(7 % 2)     # => 1    modulo
```

### The `/` vs `//` division gotcha

This is the single most common surprise for newcomers. In Python 3, **`/` always produces a `float`**, even when the result is whole. Use `//` for floor (integer) division:

```python
print(7 / 2)     # => 3.5    "true" division, always a float
print(7 // 2)    # => 3      floor division (rounds toward -infinity)
print(8 / 2)     # => 4.0    still a float! note the .0
print(8 // 2)    # => 4      int
print(-7 // 2)   # => -4     floors toward negative infinity, not toward zero
print(7 % 2)     # => 1
print(7.0 // 2)  # => 3.0    float in, float out
```

> PHP's `/` gives an int when evenly divisible and a float otherwise (`7/2 == 3.5`, `8/2 == 4`). Python's `/` is consistent: **always float**. Reach for `//` whenever you want an integer count.

`divmod()` gives you quotient and remainder together:

```python
print(divmod(7, 2))   # => (3, 1)
```

## Core concept: explicit type conversion (casting)

Because Python won't coerce for you, you convert deliberately using the type's constructor.

```python
print(int("42"))      # => 42     str -> int
print(int(3.9))       # => 3      float -> int (TRUNCATES toward zero, no rounding!)
print(int(-3.9))      # => -3
print(float("3.14"))  # => 3.14   str -> float
print(float(5))       # => 5.0
print(str(42))        # => "42"   anything -> its string form
print(str(3.14))      # => "3.14"
```

`int()` accepts a base for parsing strings:

```python
print(int("FF", 16))   # => 255   parse hex string
print(int("1010", 2))  # => 10    parse binary string
```

Invalid conversions raise a clear error (no silent `0` like old PHP):

```python
print(int("abc"))
# => ValueError: invalid literal for int() with base 10: 'abc'
```

### `bool()` and truthiness preview

`bool(x)` tells you how `x` behaves in a condition. Most values are "truthy"; a specific set is "falsy":

```python
# Falsy values:
print(bool(0))        # => False
print(bool(0.0))      # => False
print(bool(""))       # => False   empty string
print(bool([]))       # => False   empty list
print(bool({}))       # => False   empty dict
print(bool(None))     # => False

# Truthy values (basically everything else):
print(bool(42))       # => True
print(bool(-1))       # => True   (any non-zero number, even negative)
print(bool("0"))      # => True   <- a non-empty STRING, even "0"!
print(bool(" "))      # => True   a space is non-empty
print(bool([0]))      # => True   non-empty list
```

That `bool("0") == True` line trips up every PHP developer: in PHP, the string `"0"` is falsy. **In Python, any non-empty string is truthy.** We cover truthiness in full in its own lesson; for now, memorize the falsy set: `0`, `0.0`, `""`, `[]`, `{}`, `()`, `set()`, `None`, `False`.

## Core concept: everything is an object

In Python, *everything* is an object — integers, strings, functions, classes, even types themselves. Every object has a type, an identity (`id()`), and attributes/methods you can call.

```python
print((255).bit_length()) # => 8            ints have methods, e.g. bit_length
print("hello".upper())    # => 'HELLO'      strings have methods
print(hex(5))             # => '0x5'        the hex() builtin formats an int

# Even functions are objects you can pass around:
def shout(s):
    return s.upper()

f = shout                 # bind another name to the function object
print(f("hi"))            # => HI
print(type(shout))        # => <class 'function'>
```

This is why method-on-literal syntax works (`"x".strip()`) and why you can store functions in variables, lists, and dicts. It also explains the naming/binding model: a variable is just a label stuck onto an object, and many labels can stick to the same object.

## Common mistakes & gotchas

- **Using `==` to test for `None`.** Use `is None` / `is not None` — `None` is a singleton.
- **Using `is` to compare numbers or strings.** Use `==`. `is` only checks identity; small-int and string caching make `is` *accidentally* work sometimes, which is worse than never working.
- **Expecting `/` to give an integer.** `7 / 2 == 3.5` and `8 / 2 == 4.0` (a float!). Use `//` for integer division.
- **Forgetting `int()` truncates, not rounds.** `int(3.9) == 3`, `int(-3.9) == -3`. Use `round()` if you want rounding.
- **Assuming `b = a` copies a list.** It creates another *name for the same object*. Mutating `b` mutates `a`. Use `a.copy()` or `list(a)` for a shallow copy.
- **Trying to mutate a string.** `s[0] = "X"` raises `TypeError`; strings are immutable. Build a new string instead.
- **Expecting PHP-style coercion.** `"5" + 3` is a `TypeError`, not `8`. Convert explicitly.
- **Thinking `"0"` is falsy.** It isn't — only the *empty* string `""` is falsy in Python.
- **Lowercase `true`/`false`/`null`.** Python uses `True`, `False`, `None` (capitalized). Lowercase is a `NameError`.
- **Reassigning an `UPPER_CASE` "constant."** Python won't stop you, but you'll confuse every reader.

## Best practices

- Follow **PEP 8**: `snake_case` for variables/functions, `UPPER_SNAKE_CASE` for constants, `PascalCase` for classes.
- Give variables **descriptive names** (`user_count`, not `uc` or `x`) — Python prizes readability.
- Use **`is`/`is not` only** for `None`, `True`, `False`; use `==` for everything else.
- Use `isinstance(x, T)` rather than `type(x) == T` so subclasses are handled correctly.
- Use **underscores in big numeric literals** (`10_000_000`) for readability.
- Reach for **`//`** when you intend integer division; reach for **`decimal.Decimal`** for money, never `float`.
- Be deliberate about **mutability**: copy a list/dict before mutating if the caller still needs the original.
- Prefer **explicit conversion** at boundaries (parsing request data, env vars) and validate it — `int(request.GET["page"])` can raise; handle it.

## Interview Q&A

**Q: Is Python statically or dynamically typed? Strongly or weakly typed?**
**A:** Dynamically typed (types are attached to values and checked at runtime; you don't declare variable types) and strongly typed (it won't implicitly coerce unrelated types — `"5" + 3` raises `TypeError`). The combination is the key point.

**Q: What's the difference between `==` and `is`?**
**A:** `==` compares values (equality, via `__eq__`); `is` compares identity (whether two names point to the exact same object, comparing `id()`). Use `is` only for singletons like `None`. Example: two equal lists are `==` but not `is`.

**Q: Why does `0.1 + 0.2 == 0.3` return `False`?**
**A:** `float` is IEEE-754 double precision; `0.1` and `0.2` can't be represented exactly in binary, so their sum is `0.30000000000000004`. Compare with a tolerance (`math.isclose`) or use `decimal.Decimal` for exact decimal math (e.g., money).

**Q: What's the difference between `/` and `//`?**
**A:** `/` is true division and always returns a `float` (`8 / 2 == 4.0`). `//` is floor division, returning the result rounded toward negative infinity (`7 // 2 == 3`, `-7 // 2 == -4`). `//` returns an `int` if both operands are ints, a `float` if either is a float.

**Q: Does Python have constants?**
**A:** Not real ones. The convention is `UPPER_SNAKE_CASE` to signal "don't reassign," but the interpreter won't enforce it. `typing.Final` documents intent and is checked by static type-checkers like `mypy`, not at runtime; `enum.Enum` is the closest to genuinely immutable named values.

**Q: What does "everything is an object" mean in Python?**
**A:** Every value — ints, strings, functions, classes, modules, even types — is an object with a type, an identity (`id()`), and attributes/methods. That's why `"abc".upper()` works and why you can assign a function to a variable or store it in a list.

**Q: Explain mutability with an example of how it causes a bug.**
**A:** Mutable objects (`list`, `dict`, `set`) can change in place; immutable ones (`int`, `str`, `tuple`) cannot. Because `b = a` binds `b` to the *same* list, `b.append(x)` also changes `a`. The fix is to copy: `b = a.copy()`. The classic trap is a mutable default argument (`def f(items=[])`), which is shared across calls.

**Q: Why is `bool` considered a subclass of `int`?**
**A:** Historically `bool` was added on top of `int`, so `True == 1` and `False == 0`, and `isinstance(True, int)` is `True`. You can even do `True + True == 2`. Practically, it means a `bool` works anywhere an `int` is expected.

**Q: How big can a Python `int` be?**
**A:** Arbitrarily big — Python integers have unlimited precision and never overflow (limited only by available memory). `2 ** 1000` just works, no special library needed.

**Q: Why should you use `is None` instead of `== None`?**
**A:** `None` is a singleton, so identity is the correct, unambiguous test, and `is` can't be fooled by a custom `__eq__`. It's also marginally faster and is the idiom PEP 8 recommends.

**Q: What is `id()` and when would you use it?**
**A:** `id(obj)` returns a unique integer identifying an object for its lifetime (in CPython, the memory address). It backs the `is` operator. You rarely call it directly, but it's useful for understanding aliasing and explaining why `a is b` is `True` after `b = a`.

**Q: Why might `a is b` be `True` for `a = 256; b = 256` but `False` for `257`?**
**A:** CPython caches small integers (about `-5` to `256`), so both names point to the same cached object. `257` is outside that range, so two separate objects are usually created. This is an implementation detail and exactly why you must use `==`, not `is`, for numeric equality.

## Practice exercises

1. **Type tour.** Create one variable of each core type (`int`, `float`, `complex`, `bool`, `str`, `None`) and print each value alongside its `type()`.
2. **Swap and unpack.** Given `a = 1`, `b = 2`, `c = 3`, rotate the values so `a` becomes `b`'s value, `b` becomes `c`'s, and `c` becomes `a`'s — in a single line. Then use starred unpacking to split `[10, 20, 30, 40, 50]` into `head` (first), `tail` (last), and `middle` (everything between).
3. **Division drill.** For `a = 17`, `b = 5`, print `a / b`, `a // b`, `a % b`, and `divmod(a, b)`. Explain in a comment why the first is a float and the second is an int.
4. **Aliasing trap.** Write code that demonstrates the shared-reference bug: make `list2 = list1`, append to `list2`, and show that `list1` changed. Then fix it so `list1` is left untouched.
5. **Casting and validation.** Read a value that "came from a request" (`raw = "0042"`), convert it to an `int`, and print it. Then attempt `int("12.5")`, catch the resulting error with `try/except`, and print a friendly message.
6. **Truthiness check.** Print `bool(...)` for each of: `0`, `""`, `"0"`, `[]`, `[0]`, `None`, `" "`. Predict each result before running.

<details>
<summary>Solutions</summary>

```python
# 1. Type tour
i = 42
f = 3.14
c = 2 + 3j
b = True
s = "hello"
n = None
for value in (i, f, c, b, s, n):
    print(value, "->", type(value))
# => 42 -> <class 'int'>
# => 3.14 -> <class 'float'>
# => (2+3j) -> <class 'complex'>
# => True -> <class 'bool'>
# => hello -> <class 'str'>
# => None -> <class 'NoneType'>
```

```python
# 2. Swap and unpack
a, b, c = 1, 2, 3
a, b, c = b, c, a          # rotate left in one line
print(a, b, c)             # => 2 3 1

head, *middle, tail = [10, 20, 30, 40, 50]
print(head)                # => 10
print(middle)              # => [20, 30, 40]
print(tail)                # => 50
```

```python
# 3. Division drill
a, b = 17, 5
print(a / b)        # => 3.4   '/' is true division -> always float
print(a // b)       # => 3     '//' floor division -> int here (both ints)
print(a % b)        # => 2     remainder
print(divmod(a, b)) # => (3, 2)  quotient and remainder together
```

```python
# 4. Aliasing trap
list1 = [1, 2, 3]
list2 = list1          # same object!
list2.append(99)
print(list1)           # => [1, 2, 3, 99]  (oops, list1 changed)

# Fix: make a copy
list1 = [1, 2, 3]
list2 = list1.copy()   # or list(list1), or list1[:]
list2.append(99)
print(list1)           # => [1, 2, 3]      (untouched)
print(list2)           # => [1, 2, 3, 99]
```

```python
# 5. Casting and validation
raw = "0042"
print(int(raw))        # => 42  (leading zeros are fine for base-10 strings)

try:
    value = int("12.5")     # int() won't parse a decimal string
except ValueError as e:
    print("Could not convert:", e)
# => Could not convert: invalid literal for int() with base 10: '12.5'

# If you actually want 12 from "12.5", go through float first:
print(int(float("12.5")))   # => 12
```

```python
# 6. Truthiness check
for v in (0, "", "0", [], [0], None, " "):
    print(repr(v), "->", bool(v))
# => 0 -> False
# => '' -> False
# => '0' -> True    <- non-empty string is truthy!
# => [] -> False
# => [0] -> True    <- non-empty list is truthy
# => None -> False
# => ' ' -> True    <- a space is a non-empty string
```

</details>

## Cheat sheet

```python
# --- Assignment ---
x = 10                      # no $, no ;, no type
a, b = 1, 2                 # multiple assignment
a, b = b, a                 # swap, no temp
p = q = r = 0               # chained
first, *rest = [1, 2, 3]    # starred unpacking

# --- Core types ---
i = 42            # int   (arbitrary precision, never overflows)
f = 3.14          # float (IEEE-754 double)
z = 2 + 3j        # complex
flag = True       # bool  (subclass of int: True == 1)
s = "text"        # str   (immutable Unicode)
nothing = None    # NoneType (singleton)

# --- Introspection ---
type(x)              # the exact class
isinstance(x, int)   # True for x and its subclasses (preferred check)
id(x)                # identity / address; backs `is`
x is None            # identity test (use for None/True/False)
x == y               # value/equality test (use for everything else)

# --- Numeric literals & ops ---
1_000_000   0xFF   0b1010   0o17   1.5e3
7 / 2   == 3.5     # true division -> ALWAYS float
7 // 2  == 3       # floor division -> int (here)
7 % 2   == 1       # modulo
7 ** 2  == 49      # power
divmod(7, 2) == (3, 1)

# --- Casting ---
int("42")        # 42     (int("abc") -> ValueError)
int(3.9)         # 3      (truncates, not rounds)
int("FF", 16)    # 255    (parse with base)
float("3.14")    # 3.14
str(42)          # "42"
bool(x)          # truthiness

# --- Falsy values (everything else is truthy) ---
0, 0.0, "", [], {}, (), set(), None, False

# --- Conventions ---
snake_case          # variables, functions
UPPER_SNAKE_CASE    # "constants" (not enforced!)
PascalCase          # classes
_leading_underscore # "internal" by convention
```

**Remember the four traps:** `/` is always float (use `//`), `is` ≠ `==` (use `==` for values), `b = a` aliases a mutable object (copy it), and `"0"` is truthy in Python.
