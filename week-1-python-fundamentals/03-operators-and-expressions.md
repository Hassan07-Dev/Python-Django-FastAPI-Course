# 03 — Operators & Expressions

> Week 1 · Python Fundamentals

## What you will learn

- All arithmetic operators, including the three that surprise PHP devs: `//` (floor division), `%` (modulo), and `**` (power)
- Comparison operators and Python's killer feature: **chained comparisons** like `1 < x < 10`
- Logical `and` / `or` / `not` — and the huge difference from PHP's `&&` / `||`: Python's logical operators **return one of the operands**, not a boolean
- Assignment and augmented assignment (`+=`, `-=`, `//=`, etc.) — and why Python has **no** `++` / `--`
- Identity (`is` / `is not`) vs equality (`==`) — and the iron rule: **always compare to `None` with `is`**
- Membership testing with `in` / `not in`
- Bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`)
- The walrus operator `:=` with a real, practical example
- Operator precedence, at a glance
- **Truthiness**: which values are "falsy" in Python and how that shapes idiomatic code

## PHP/Laravel bridge

Most operators look identical to PHP, but three behaviors will bite you if you assume they match. First, Python's `/` **always** produces a float (`6 / 2` is `3.0`, not `3`); use `//` for integer-style division. Second, `and`/`or` are **value-returning and short-circuiting** like PHP's `&&`/`||`, but Python returns the actual operand (`"a" or "b"` is `"a"`), which makes the idiom `name = supplied or "Anonymous"` work exactly like PHP's `?:` shorthand. Third, Python has **no `++`/`--`** and no `===`; equality is `==` (compares values with type coercion rules far stricter than PHP's loose `==`), and identity is `is` (the closest analog to "same object," not "same value"). Finally, `.` is the property/method accessor — string concatenation is `+`, never `.`.

---

## Core concept: Arithmetic operators

The basics match PHP, but pay attention to division.

```python
print(7 + 3)    # => 10   addition
print(7 - 3)    # => 4    subtraction
print(7 * 3)    # => 21   multiplication
print(7 / 3)    # => 2.3333333333333335   TRUE division -> ALWAYS a float
print(7 // 3)   # => 2    FLOOR division   (integer-like, rounds toward -inf)
print(7 % 3)    # => 1    modulo (remainder)
print(7 ** 3)   # => 343  exponentiation (7 to the power 3)
```

### `/` always returns a float — this is the #1 PHP-dev surprise

In PHP, `6 / 2` gives `3` (an int). In Python 3, `/` is **true division** and the result is *always* a `float`, even when it divides evenly.

```python
print(6 / 2)        # => 3.0   <- note the .0, it's a float
print(type(6 / 2))  # => <class 'float'>
```

If you want integer division (the old PHP-`intdiv` behavior), use `//`:

```python
print(6 // 2)        # => 3
print(type(6 // 2))  # => <class 'int'>
print(7 // 2)        # => 3    (truncated toward negative infinity)
```

### `//` floors toward negative infinity (not toward zero)

This catches people. `//` does **not** simply chop off the decimal; it rounds *down* (toward −∞). For positive numbers that looks the same as truncation, but for negatives it differs:

```python
print(7 // 2)     # =>  3
print(-7 // 2)    # => -4   (NOT -3; floor of -3.5 is -4)
print(7 // -2)    # => -4
```

If you genuinely want truncation toward zero, use `int()` or `math.trunc()`:

```python
import math
print(int(-7 / 2))         # => -3   (chop toward zero)
print(math.trunc(-7 / 2))  # => -3
```

### `%` follows the sign of the divisor

Python's modulo result takes the **sign of the right-hand operand** (the divisor). This differs from PHP/C, where `%` follows the dividend. Beginners debugging negative numbers should know this:

```python
print(7 % 3)     # =>  1
print(-7 % 3)    # =>  2   (PHP would give -1 here)
print(7 % -3)    # => -2
```

The identity `a == (a // b) * b + (a % b)` always holds in Python. A clean trick: `n % 2` is `0` for even and `1` for odd, even for negative `n`, because the result follows the divisor `2` (always non-negative).

### `**` is exponentiation, and `divmod` gives you both at once

```python
print(2 ** 10)        # => 1024
print(2 ** 0.5)       # => 1.4142135623730951   (square root via fractional power)
print(divmod(17, 5))  # => (3, 2)   quotient AND remainder in one call -> (17//5, 17%5)
```

`**` also works as the modular-power shortcut via the three-arg `pow()`:

```python
print(pow(2, 10, 1000))  # => 24   == (2 ** 10) % 1000, but computed efficiently
```

### Numbers don't overflow

Unlike PHP/C ints, Python integers have **arbitrary precision** — they grow as large as your memory allows. No silent overflow, no `PHP_INT_MAX`.

```python
print(2 ** 200)
# => 1606938044258990275541962092341162602522202993782792835301376
```

---

## Core concept: Comparison operators & chained comparisons

The six comparisons return a real `bool`:

```python
print(3 == 3)   # => True
print(3 != 4)   # => True
print(3 < 4)    # => True
print(3 <= 3)   # => True
print(5 > 4)    # => True
print(5 >= 6)   # => False
```

> There is **no `===`** in Python. `==` compares *values*. For "same object," use `is` (covered below). Python's `==` does not do PHP-style loose juggling: `0 == "0"` is `False` (different types, no coercion between str and int).

```python
print(0 == "0")   # => False   (PHP's loose == would say true; Python does not coerce)
print(1 == 1.0)   # => True    (int/float DO compare numerically)
print(1 == True)  # => True    (bool is a subclass of int: True == 1, False == 0)
```

### Chained comparisons — a genuinely lovely Python feature

You can write mathematical range checks directly. In PHP you'd write `1 < $x && $x < 10`; in Python:

```python
x = 5
print(1 < x < 10)        # => True
print(0 <= x <= 100)     # => True
print(1 < x < 10 < 5)    # => False
```

This is **not** `(1 < x) < 10`. Python evaluates each comparison and ANDs them together, and — importantly — each middle operand is **evaluated only once** (matters when it's a function call):

```python
def expensive():
    print("called!")
    return 5

# expensive() runs exactly ONE time, even though it's compared twice:
print(1 < expensive() < 10)
# => called!
# => True
```

Equality chains too:

```python
a = b = c = 1
print(a == b == c)   # => True   (all equal)
```

---

## Core concept: Logical operators (`and` / `or` / `not`)

This is where Python diverges most from PHP, so read carefully.

Python uses the **words** `and`, `or`, `not` (PHP also has these as low-precedence keywords, but you mostly use `&&`/`||`/`!`). Python has **no** symbolic `&&`/`||`/`!` — using `&`/`|` does something completely different (bitwise, see below).

### They short-circuit AND return an operand (not a coerced bool)

`and`/`or` evaluate left to right, stop early, and **return the last value they evaluated** — which may not be `True`/`False` at all.

- `a and b`: if `a` is falsy, return `a`; otherwise return `b`.
- `a or b`: if `a` is truthy, return `a`; otherwise return `b`.

```python
print(0 and 99)        # => 0     (0 is falsy -> stop, return 0)
print(3 and 99)        # => 99    (3 is truthy -> return the second operand)
print("" or "default") # => default   (empty string falsy -> return second)
print("hi" or "default")# => hi   ("hi" truthy -> return first, never evaluates second)
print(not "")          # => True  ('not' DOES always return a real bool)
print(not 5)           # => False
```

### The idiomatic "default value" pattern

This replaces PHP's `$name = $input ?: "Anonymous";`:

```python
def greet(name):
    name = name or "Anonymous"   # if name is "", None, or other falsy -> use default
    return f"Hello, {name}!"

print(greet("Sam"))   # => Hello, Sam!
print(greet(""))      # => Hello, Anonymous!
print(greet(None))    # => Hello, Anonymous!
```

> Caveat (same as PHP's `?:`): `or` triggers on **any** falsy value, including `0` and `""`. If `0` is a *valid* value you want to keep, use an explicit `None` check or the `if x is None` pattern instead — see the conditional expression below.

### Short-circuit as a guard

Because the right side isn't evaluated when the left already decides the result, you can guard against errors:

```python
user = None
# Safe: 'user is not None' is checked first; '.name' is never touched if user is None
print(user is not None and user.name)   # => False   (short-circuits, no AttributeError)
```

### Conditional (ternary) expression

Python's ternary reads like English and is the right tool when `0`/`""` are valid:

```python
count = 0
label = "items" if count != 1 else "item"
print(f"{count} {label}")   # => 0 items

# Keep a valid 0, only default on None:
value = None
result = value if value is not None else "fallback"
print(result)   # => fallback
```

---

## Core concept: Assignment & augmented assignment

Basic assignment binds a name to an object:

```python
x = 10
x = "now a string"   # names are not typed; rebinding to another type is fine
```

Augmented assignment mirrors PHP:

```python
n = 10
n += 3   # n = n + 3  -> 13
n -= 1   # 12
n *= 2   # 24
n /= 5   # 4.8   (note: /= produces a float, just like /)
n //= 2  # 2.0   (floor-divide-assign)
n **= 3  # 8.0   (power-assign)
n %= 5   # 3.0
print(n) # => 3.0
```

String and list augmented assignment:

```python
s = "ab"
s += "cd"          # => "abcd"   (string concatenation; '+' not '.')
s *= 2             # => "abcdabcd"  (repetition!)
print(s)

nums = [1, 2]
nums += [3, 4]     # extends in place -> [1, 2, 3, 4]
print(nums)
```

### There is NO `++` or `--` in Python

```python
i = 5
# i++       # SyntaxError! does not exist
i += 1      # this is how you increment
print(i)    # => 6
```

(Confusingly, `i = ++i` *parses* as "apply unary plus twice," so it runs but does nothing useful — never write it.)

### Multiple & tuple assignment

```python
a, b, c = 1, 2, 3          # unpacking
a, b = b, a                # swap with no temp variable! a=2, b=1
print(a, b)                # => 2 1

x = y = z = 0              # chained assignment, all point to the same 0
first, *rest = [1, 2, 3, 4]  # first=1, rest=[2, 3, 4]
print(first, rest)         # => 1 [2, 3, 4]
```

---

## Core concept: Identity (`is`) vs equality (`==`)

- `==` asks **"are these values equal?"** (calls the object's `__eq__`).
- `is` asks **"are these the exact same object in memory?"** (compares identity / `id()`).

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # => True    (same contents)
print(a is b)   # => False   (two distinct list objects)
print(a is c)   # => True    (c is just another name for the SAME list)
print(id(a) == id(c))  # => True
```

### The iron rule: compare to `None`, `True`, `False` with `is`

`None` is a singleton — there is exactly one `None` in a running program. Always use `is` / `is not`:

```python
value = None
print(value is None)      # => True   <- CORRECT, idiomatic
print(value is not None)  # => False
# print(value == None)    # works, but considered wrong style; linters flag it
```

Why does it matter beyond style? A class could override `__eq__` to return surprising results for `== None`, but nothing can fake *identity*. `is None` is fast, safe, and unambiguous.

### The integer-caching trap (why never use `is` for numbers/strings)

CPython caches small integers (−5 to 256) and some short strings, so `is` *accidentally* works for them — then breaks for larger values. This is an implementation detail, never rely on it:

```python
x = 256
y = 256
print(x is y)   # => True    (cached small int — coincidence!)

x = 257
y = 257
print(x is y)   # => False   (outside cache range — different objects)

print(x == y)   # => True    <- ALWAYS use == for value comparison
```

**Rule of thumb:** use `is`/`is not` only for `None`, `True`, `False`, and other known singletons. Use `==` for everything else.

---

## Core concept: Membership (`in` / `not in`)

`in` checks membership; it works on strings, lists, tuples, sets, dict keys, and any iterable. Far cleaner than PHP's `in_array` / `strpos` / `array_key_exists`.

```python
print(3 in [1, 2, 3])          # => True
print(5 not in [1, 2, 3])      # => True
print("ell" in "hello")        # => True    (substring check, like str_contains)
print("z" in "hello")          # => False

# Dict: 'in' checks KEYS by default
prices = {"apple": 1.2, "pear": 0.9}
print("apple" in prices)       # => True
print(1.2 in prices)           # => False   (that's a value, not a key)
print(1.2 in prices.values())  # => True    (check values explicitly)
```

`in` on a `set` is O(1) (hash lookup) vs O(n) for a list — prefer sets for repeated membership tests over large collections.

```python
allowed = {"GET", "POST", "PUT", "DELETE"}   # set literal
method = "POST"
print(method in allowed)   # => True   (fast hash lookup)
```

---

## Core concept: Bitwise operators

These operate on the binary representation of integers. Same symbols as PHP/C.

| Operator | Name        | Example  | Result |
|----------|-------------|----------|--------|
| `&`      | AND         | `12 & 10`| `8`    |
| `\|`     | OR          | `12 \| 10`| `14`  |
| `^`      | XOR         | `12 ^ 10`| `6`    |
| `~`      | NOT (invert)| `~12`    | `-13`  |
| `<<`     | left shift  | `3 << 2` | `12`   |
| `>>`     | right shift | `12 >> 2`| `3`    |

```python
print(12 & 10)   # => 8    (1100 & 1010 = 1000)
print(12 | 10)   # => 14   (1100 | 1010 = 1110)
print(12 ^ 10)   # => 6    (1100 ^ 1010 = 0110)
print(~12)       # => -13  (~x == -(x+1) in two's complement)
print(3 << 2)    # => 12   (3 * 2**2)  left shift = multiply by powers of 2
print(12 >> 2)   # => 3    (12 // 2**2) right shift = floor-divide by powers of 2
print(format(0b1100, "04b"))  # => 1100   (view the bits)
```

> Watch out: `&` and `|` are **bitwise**, NOT logical. `if a & b:` is a common bug when you meant `if a and b:`. Also, `&`/`|` have *lower* precedence than comparisons, so `x & 1 == 0` parses as `x & (1 == 0)` — parenthesize: `(x & 1) == 0`.

A common real use is flag/permission masks:

```python
READ, WRITE, EXEC = 1, 2, 4          # 0b001, 0b010, 0b100
perms = READ | WRITE                 # => 3, has read+write
print(bool(perms & WRITE))           # => True   (is the WRITE bit set?)
print(bool(perms & EXEC))            # => False
```

---

## Core concept: The walrus operator `:=` (assignment expression)

Added in Python 3.8, `:=` lets you **assign a value AND use it** in the same expression. It solves the "compute once, test, then reuse" pattern without a separate line.

```python
# Without walrus: you call len() conceptually twice or need an extra line
data = [1, 2, 3, 4, 5]
n = len(data)
if n > 3:
    print(f"List is long: {n} items")

# With walrus: assign inside the condition
if (n := len(data)) > 3:
    print(f"List is long: {n} items")   # => List is long: 5 items
```

### The classic practical example: read-until-sentinel loop

```python
# Simulate reading lines until a blank line. Walrus reads + tests in one place.
lines = iter(["alpha", "beta", "", "ignored"])

while (line := next(lines, "")) != "":
    print(f"Got: {line}")
# => Got: alpha
# => Got: beta
```

Another everyday use — avoid recomputing an expensive call inside a comprehension:

```python
def process(x):
    return x * x - 4

# Keep only positive results, without calling process() twice per item:
results = [y for x in range(5) if (y := process(x)) > 0]
print(results)   # => [5, 12]   (for x=3 -> 5, x=4 -> 12; others are <= 0)
```

> Always wrap `:=` in parentheses when mixing with comparisons or in a comprehension; bare `x := 5` at statement level is a `SyntaxError` (use plain `=` there). The walrus exists for *expressions*, not statements.

---

## Core concept: Truthiness & falsy values

Every Python object is either "truthy" or "falsy" in a boolean context (`if`, `while`, `and`, `or`, `not`, `bool()`). This drives a lot of idiomatic code.

### The complete list of falsy values

```python
print(bool(None))      # => False
print(bool(False))     # => False
print(bool(0))         # => False
print(bool(0.0))       # => False
print(bool(0j))        # => False   (complex zero)
print(bool(""))        # => False   (empty string)
print(bool([]))        # => False   (empty list)
print(bool(()))        # => False   (empty tuple)
print(bool({}))        # => False   (empty dict)
print(bool(set()))     # => False   (empty set)
print(bool(range(0)))  # => False   (empty range)
```

**Everything else is truthy** — including `"0"`, `"False"`, `[0]`, `[[]]`, `-1`, and `0.0001`.

```python
print(bool("0"))     # => True    (non-empty string! classic PHP-vs-Python gotcha)
print(bool([0]))     # => True    (non-empty list, even though it contains a 0)
print(bool(-1))      # => True
```

### Idiomatic emptiness checks

Check collections directly — don't compare lengths to zero:

```python
items = []

if not items:                 # idiomatic: empty list is falsy
    print("empty")            # => empty

# Avoid the verbose:
# if len(items) == 0: ...

names = ["Ada", "Linus"]
if names:                     # idiomatic: non-empty -> truthy
    print(f"{len(names)} names")  # => 2 names
```

### Truthiness vs explicit `None` check — pick the right one

These are **not** equivalent. Choose based on whether `0`/`""`/`[]` are legitimate values:

```python
def describe(qty):
    # WRONG if 0 is a valid quantity: 'not qty' is True for qty == 0
    if not qty:
        return "missing"
    return f"have {qty}"

print(describe(0))     # => missing   <- probably a bug! 0 is a real quantity

def describe_fixed(qty):
    if qty is None:    # only None means "not provided"
        return "missing"
    return f"have {qty}"

print(describe_fixed(0))    # => have 0   <- correct
print(describe_fixed(None)) # => missing
```

---

## Core concept: Operator precedence (overview)

When in doubt, **add parentheses**. But here's the order, highest (binds tightest) to lowest:

| Tier | Operators | Notes |
|------|-----------|-------|
| 1 | `()` `[]` `{}` | grouping, calls, subscripts, literals |
| 2 | `x.attr` | attribute access |
| 3 | `**` | power (right-associative: `2**3**2 == 2**(3**2) == 512`) |
| 4 | `+x` `-x` `~x` | unary |
| 5 | `*` `/` `//` `%` `@` | multiply/divide |
| 6 | `+` `-` | add/subtract |
| 7 | `<<` `>>` | bit shifts |
| 8 | `&` | bitwise AND |
| 9 | `^` | bitwise XOR |
| 10 | `\|` | bitwise OR |
| 11 | `==` `!=` `<` `<=` `>` `>=` `is` `is not` `in` `not in` | all comparisons (same level, chainable) |
| 12 | `not` | logical NOT |
| 13 | `and` | logical AND |
| 14 | `or` | logical OR |
| 15 | `x if c else y` | conditional expression |
| 16 | `:=` | walrus (lowest; almost always needs parens) |

Surprising consequences worth memorizing:

```python
print(2 ** 3 ** 2)        # => 512   ** is right-associative -> 2**(3**2)
print(-2 ** 2)            # => -4    ** binds tighter than unary minus -> -(2**2)
print((-2) ** 2)          # => 4
print(2 + 3 * 4)          # => 14    * before +
print(True or False and False)  # => True   'and' before 'or' -> True or (False and False)
print(5 & 1 == 1)         # => 1     comparisons bind TIGHTER than &, so this is 5 & (1 == 1)
# (1 == 1) is True, and True == 1, so 5 & 1 -> 1.  Lesson: PARENTHESIZE bitwise + comparison.
print((5 & 1) == 1)       # => True  what you probably meant
```

---

## Common mistakes & gotchas

- **Expecting `/` to give an int.** `10 / 2` is `5.0`. Use `//` for integer division.
- **`//` truncates toward zero — it doesn't.** It floors toward −∞: `-7 // 2 == -4`.
- **Using `==` to compare with `None`.** Use `is None` / `is not None` (style + correctness).
- **Using `is` to compare numbers or strings.** It "works" for small cached ints/strings, then silently breaks. Use `==`.
- **Writing `i++` or `i--`.** They don't exist; use `i += 1` / `i -= 1`.
- **Using `&` / `|` where you meant `and` / `or`.** `&`/`|` are bitwise, lower-precedence than comparisons, and don't short-circuit. `if x & y:` vs `if x and y:` are different bugs waiting to happen.
- **Forgetting that `or` defaults on any falsy value.** `port = config_port or 8080` silently replaces a legit `0`. Use `config_port if config_port is not None else 8080`.
- **Assuming `0 == "0"` like PHP.** Python does not coerce str/int; that's `False`.
- **Forgetting `bool("0")` is `True`** and `bool("False")` is `True` — any non-empty string is truthy.
- **Bare `x := 5` as a statement.** SyntaxError; walrus is only for expressions. Use plain `=`.
- **`-2 ** 2` returning `-4`.** `**` outranks unary minus. Parenthesize: `(-2) ** 2`.

## Best practices

- **Parenthesize for readers, not just for the parser.** `(x & MASK) == 0` even when precedence happens to be on your side.
- **Use chained comparisons** (`0 <= i < len(arr)`) — they're clearer and evaluate the middle once.
- **Use `is` / `is not` exclusively for singletons** (`None`, `True`, `False`, sentinels).
- **Check emptiness with truthiness** (`if not items:`) — but switch to `if x is None:` whenever `0`/`""`/`[]` are valid values.
- **Prefer `x or default` for the cheap default idiom**, but know its falsy-trap; reach for the conditional expression when precision matters.
- **Use `divmod(a, b)`** when you need both quotient and remainder — it's one call and a tiny bit faster.
- **Reach for the walrus** to avoid recomputation in `while`/comprehensions, but don't overuse it; readability first.
- **Use sets for membership tests** on large or repeatedly-queried collections (O(1) vs O(n)).

## Interview Q&A

**Q:** What's the difference between `/` and `//` in Python 3?
**A:** `/` is true division and **always returns a float** (`6 / 2 == 3.0`). `//` is floor division, returning the largest integer less than or equal to the quotient — it yields an `int` when both operands are ints and *floors toward negative infinity* (`-7 // 2 == -4`), not toward zero.

**Q:** Explain the difference between `==` and `is`.
**A:** `==` compares **values** via the object's `__eq__`. `is` compares **identity** — whether two names refer to the exact same object in memory (same `id()`). Two different lists with identical contents are `==` but not `is`. Use `is` only for singletons like `None`.

**Q:** Why should you write `if x is None` instead of `if x == None`?
**A:** `None` is a singleton, so identity is the precise, fast check. `==` could be overridden by a custom `__eq__` to give a misleading answer, and it's slower; linters (PEP 8) flag `== None`. Identity can't be faked, so `is None` is always correct.

**Q:** What does `"" or "default"` evaluate to, and why?
**A:** `"default"`. Python's `or` short-circuits and **returns an operand**, not a coerced bool: if the left side is falsy (an empty string is falsy), it returns the right side. This powers the `value = x or fallback` default idiom.

**Q:** What's the gotcha with the `x or default` idiom?
**A:** `or` triggers on *any* falsy value — `0`, `0.0`, `""`, `[]`, `None`. So `timeout = cfg or 30` wrongly replaces a legitimate `0`. When `0`/empty are valid, use `cfg if cfg is not None else 30`.

**Q:** How does Python's `%` behave with negative numbers?
**A:** The result takes the **sign of the divisor** (right operand). So `-7 % 3 == 2` (not `-1` like PHP/C). The identity `a == (a // b) * b + (a % b)` always holds, which is why `n % 2` is `0`/`1` even for negative `n`.

**Q:** What is a chained comparison and how is it evaluated?
**A:** Writing `a < b < c`, which Python expands to `(a < b) and (b < c)` — but the middle term `b` is **evaluated only once**. It's both more readable and avoids double-evaluating a function call in the middle.

**Q:** What does the walrus operator `:=` do? Give a use case.
**A:** It's an assignment *expression*: it assigns a value and returns it, so you can bind and use a name in one place. Common uses: `if (n := len(data)) > 10:` to reuse `n`, and `while (line := next(it, "")) != "":` to read-and-test in a loop without duplication.

**Q:** Why is `256 is 256` sometimes `True` but `257 is 257` `False`?
**A:** CPython caches (interns) small integers from −5 to 256, so identical small ints share one object and `is` accidentally returns `True`. Outside that range you get distinct objects. It's an implementation detail — never use `is` for numeric comparison; use `==`.

**Q:** List Python's falsy values.
**A:** `None`, `False`, numeric zeros (`0`, `0.0`, `0j`), and empty containers/sequences: `""`, `[]`, `()`, `{}`, `set()`, `range(0)`. Everything else is truthy — notably `"0"`, `"False"`, and `[0]` are all truthy.

**Q:** Difference between `&`/`|` and `and`/`or`?
**A:** `and`/`or` are **logical**: they short-circuit and return one of the operands based on truthiness. `&`/`|` are **bitwise** integer ops, evaluate both sides, and have lower precedence than comparisons. Using `&` where you meant `and` is a classic bug.

**Q:** Why does `-2 ** 2` equal `-4`?
**A:** `**` has higher precedence than unary minus, so it parses as `-(2 ** 2) = -(4) = -4`. To square a negative, parenthesize: `(-2) ** 2 == 4`. Note `**` is also right-associative: `2 ** 3 ** 2 == 2 ** (3 ** 2) == 512`.

## Practice exercises

1. **Even-or-odd, the right way.** Write a one-line expression that is `True` when an integer `n` is even, and verify it works for `-4`, `-3`, `0`, and `7`.

2. **Safe percentage.** Given `scored` and `total`, compute the integer percentage (floor) without crashing when `total` is `0` — return `0` in that case. Use short-circuiting.

3. **Default config port.** You receive `cfg_port`, which may be `None` (not set) or an integer including a legitimate `0`. Produce the effective port: use `cfg_port` if it was actually provided, else `8080`. Make sure `0` is preserved.

4. **Chained range check.** Write a function `in_range(x)` that returns `True` only when `10 <= x <= 20`, using a chained comparison.

5. **Walrus dedup-read.** Given `data = [3, -1, 4, -2, 5]` and a function `f = lambda x: x * 2 - 6`, build a list of all `f(x)` values that are strictly positive, calling `f` only once per element. Use a comprehension with `:=`.

6. **Permission flags.** With `READ, WRITE, EXEC = 1, 2, 4`, given a `perms` integer, write expressions that answer (a) does it have WRITE? (b) does it have BOTH read and exec? Test with `perms = 5`.

<details>
<summary>Solutions</summary>

```python
# 1. Even-or-odd
def is_even(n):
    return n % 2 == 0

print([is_even(n) for n in (-4, -3, 0, 7)])
# => [True, False, True, False]
# n % 2 follows the sign of the divisor (2, positive), so it's always 0 or 1.


# 2. Safe percentage with short-circuit
def percent(scored, total):
    # 'total and ...' returns 0 (falsy) when total == 0, skipping the division
    return total and (scored * 100 // total) or 0

print(percent(45, 50))   # => 90
print(percent(7, 0))     # => 0   (no ZeroDivisionError)
# More explicit/readable alternative:
def percent2(scored, total):
    if total == 0:
        return 0
    return scored * 100 // total


# 3. Default port that preserves 0
def effective_port(cfg_port):
    return cfg_port if cfg_port is not None else 8080

print(effective_port(0))     # => 0     (preserved!)
print(effective_port(None))  # => 8080
print(effective_port(3000))  # => 3000
# Using 'cfg_port or 8080' would WRONGLY turn 0 into 8080.


# 4. Chained range check
def in_range(x):
    return 10 <= x <= 20

print(in_range(9), in_range(10), in_range(15), in_range(20), in_range(21))
# => False True True True False


# 5. Walrus dedup-read
data = [3, -1, 4, -2, 5]
f = lambda x: x * 2 - 6
result = [y for x in data if (y := f(x)) > 0]
print(result)   # => [2, 4]
# x=3 -> 0 (not >0), x=4 -> 2, x=5 -> 4; negatives give negative results.
# f is called exactly once per element thanks to the walrus binding.


# 6. Permission flags
READ, WRITE, EXEC = 1, 2, 4
perms = 5   # 0b101 -> READ + EXEC

has_write = bool(perms & WRITE)              # (a)
has_read_and_exec = (perms & (READ | EXEC)) == (READ | EXEC)  # (b)
print(has_write)           # => False
print(has_read_and_exec)   # => True
```

</details>

## Cheat sheet

```python
# --- Arithmetic ---
7 / 2     # 3.5   true division -> ALWAYS float
7 // 2    # 3     floor division (floors toward -inf: -7 // 2 == -4)
7 % 2     # 1     modulo (sign follows divisor: -7 % 3 == 2)
2 ** 10   # 1024  power (right-assoc; -2**2 == -4)
divmod(17, 5)  # (3, 2)  -> (quotient, remainder)

# --- Comparison & chaining ---
a == b    # value equality (no === in Python; no PHP-style coercion)
1 < x < 10           # chained; middle evaluated once
a == b == c          # equality chain

# --- Logical: return an OPERAND, short-circuit ---
a and b   # a if a falsy else b
a or b    # a if a truthy else b
not a     # always a real bool
name = supplied or "Anonymous"          # default idiom (beware falsy 0/"")
x if cond else y                        # ternary (use when 0/'' are valid)

# --- Identity vs equality ---
x is None        # YES for singletons
x == value       # YES for everything else
# never: x is 257  (cached-int trap)

# --- Membership ---
3 in [1,2,3]            # True
"ell" in "hello"        # substring
key in some_dict        # checks KEYS
val in d.values()       # checks values

# --- Assignment ---
x += 1   # no ++ / -- in Python!
a, b = b, a              # swap, no temp
first, *rest = seq       # unpack

# --- Bitwise (integers) ---
&  |  ^  ~  <<  >>        # AND OR XOR NOT shift-left shift-right
(x & MASK) == 0          # ALWAYS parenthesize vs comparisons

# --- Walrus (assignment expression) ---
if (n := len(data)) > 3: ...
while (line := next(it, "")) != "": ...

# --- Falsy values (everything else is truthy) ---
None  False  0  0.0  0j  ""  []  ()  {}  set()  range(0)
# truthy surprises: "0", "False", [0], [[]]
if not items: ...        # idiomatic empty check
if x is None: ...        # when 0/'' are valid values

# --- Precedence (tight -> loose) ---
**  >  unary +/-/~  >  * / // %  >  + -  >  shifts  >  &  >  ^  >  |
   >  comparisons/is/in  >  not  >  and  >  or  >  ternary  >  :=
```
