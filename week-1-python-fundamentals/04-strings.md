# 04 — Strings & Formatting

> Week 1 · Python Fundamentals

## What you will learn

- How to write string literals: single, double, and triple quotes — and when each matters
- Why Python strings are **immutable**, and what that means for your code
- **Indexing and slicing** in depth: `start:stop:step`, negative indices, idioms
- **f-strings** as your everyday formatting tool: embedded expressions, format specs, alignment, number formatting, and the `=` debug syntax
- Legacy formatting (`str.format()` and `%`) so you can read old code
- The string method toolbox: `upper`/`lower`, `strip`, `split`/`rsplit`, `join`, `replace`, `find`/`index`, `startswith`/`endswith`, `count`, `zfill`, `center`, and friends
- Membership (`in`) and iterating over characters
- Escape sequences and **raw strings** (`r""`)
- A practical note on Unicode, bytes, and `encode`/`decode`
- A side-by-side map from PHP string functions to Python

## PHP/Laravel bridge

In PHP a string is a mutable byte buffer and you reach for a sprawling set of `str_*` functions (`strtoupper`, `substr`, `explode`, `implode`, `str_replace`, `strpos`). In Python, strings are **objects with methods**, so instead of `strtoupper($s)` you write `s.upper()` — the function lives *on* the value. The biggest mental shift: PHP strings are mutable and byte-oriented, while Python `str` is **immutable** and is a sequence of Unicode *code points* (raw bytes live in a separate `bytes` type). Python's f-strings (`f"Hello {name}"`) are the direct, more powerful analog of PHP's double-quoted `"Hello $name"` interpolation, and `"a" . "b"` concatenation in PHP becomes `"a" + "b"` in Python.

## Core concept: String literals & quotes

A string literal can use single or double quotes — they are **identical** in Python (unlike PHP, where `"$x"` interpolates but `'$x'` does not). Pick whichever avoids escaping.

```python
a = 'hello'
b = "hello"
print(a == b)            # => True

# Choose the quote that avoids backslashes:
quote = "She said \"hi\""     # works, but ugly
quote = 'She said "hi"'       # cleaner
apostrophe = "it's fine"      # double quotes avoid escaping the '
```

There is **no separate char type** in Python. A single character is just a string of length 1.

```python
c = "x"
print(type(c), len(c))   # => <class 'str'> 1
```

### Triple-quoted (multiline) strings

Triple quotes (`'''` or `"""`) span multiple lines and preserve newlines. They are also how docstrings are written.

```python
text = """Line 1
Line 2
Line 3"""
print(text)
# => Line 1
# => Line 2
# => Line 3

# A leading backslash suppresses the very first newline:
sql = """\
SELECT id, name
FROM users
WHERE active = true
"""
print(repr(sql[:6]))     # => 'SELECT'  (no leading newline)
```

### Implicit concatenation & explicit joining

Adjacent string literals are glued together at parse time — handy for breaking long strings across lines without `+`.

```python
msg = (
    "This is one long "
    "logical string built "
    "from several literals."
)
print(msg)
# => This is one long logical string built from several literals.

# Runtime concatenation uses +, repetition uses *
print("ab" + "cd")       # => abcd
print("=" * 10)          # => ==========
```

> **Note:** Use `+` for a couple of pieces, but to join *many* pieces use `"".join(...)` — see the methods section. Repeated `+=` in a loop is O(n²) and a classic beginner trap.

## Core concept: Immutability

Strings cannot be changed in place. Every "modification" returns a **new** string; the original is untouched.

```python
s = "hello"
# s[0] = "H"             # TypeError: 'str' object does not support item assignment

upper = s.upper()
print(upper)             # => HELLO
print(s)                 # => hello   (original unchanged)
```

Why does this matter?

- **Safety & hashability:** because strings never change, they can be dictionary keys and set members.
- **Performance trap:** building a big string with `+=` in a loop creates a new string each iteration. Accumulate in a list and `join` once.

```python
# Bad: O(n^2)
out = ""
for i in range(5):
    out += str(i)

# Good: O(n)
parts = []
for i in range(5):
    parts.append(str(i))
out = "".join(parts)
print(out)               # => 01234
```

This contrasts with PHP, where `$s .= $x` mutates the existing string buffer.

## Core concept: Indexing & slicing

A string is a sequence, so you can index it like an array. Indices are **0-based**; negative indices count from the end (`-1` is the last character).

```python
s = "Python"
#    P  y  t  h  o  n
#    0  1  2  3  4  5
#   -6 -5 -4 -3 -2 -1

print(s[0])     # => P
print(s[5])     # => n
print(s[-1])    # => n   (last)
print(s[-2])    # => o
# print(s[6])   # IndexError: string index out of range
```

### Slicing: `s[start:stop:step]`

Slicing returns a **new substring**. `start` is inclusive, `stop` is **exclusive** (just like `range`). Any part can be omitted.

```python
s = "Python"

print(s[0:2])    # => Py     (indices 0,1 — stop is exclusive)
print(s[2:])     # => thon   (from 2 to end)
print(s[:2])     # => Py     (from start to 2)
print(s[:])      # => Python (full copy)
print(s[-3:])    # => hon    (last 3 chars)
print(s[:-3])    # => Pyt    (everything except last 3)
```

Crucially, slicing **never raises** for out-of-range bounds — it just clamps. This makes slices safer than indexing.

```python
print(s[2:100])  # => thon   (no error, clamped)
print(s[100:])   # => ''     (empty string, no error)
```

### The `step` (third) value

```python
s = "abcdefg"
print(s[::2])    # => aceg   (every 2nd char)
print(s[1::2])   # => bdf    (every 2nd, starting at 1)
print(s[::-1])   # => gfedcba (reverse — the classic idiom)
print(s[5:1:-1]) # => fedc   (walk backwards from 5 down to >1)
```

Reversing with `s[::-1]` is the idiomatic Python reverse (PHP would use `strrev`).

## Core concept: f-strings (your primary formatting tool)

Prefix a literal with `f` and embed any expression in `{...}`. This is the modern, readable default for nearly all formatting.

```python
name = "Ada"
age = 36
print(f"{name} is {age}")            # => Ada is 36

# Any expression works, not just variables:
print(f"{name.upper()} in 2 years is {age + 2}")  # => ADA in 2 years is 38

items = ["a", "b", "c"]
print(f"There are {len(items)} items")            # => There are 3 items
```

### Literal braces

To print a real `{` or `}`, double it.

```python
print(f"{{literal braces}} and a value: {age}")
# => {literal braces} and a value: 36
```

### The `=` debug specifier (Python 3.8+)

Appending `=` prints both the expression text and its value — a fantastic debugging shortcut.

```python
x = 42
y = 8
print(f"{x=}")           # => x=42
print(f"{x + y = }")     # => x + y = 50   (spaces around = are preserved)
```

### Format specifications — `{value:spec}`

After a colon you give a **format spec** that controls width, alignment, precision, and type. General form: `{value:[fill][align][sign][#][0][width][,][.precision][type]}`.

**Number formatting:**

```python
pi = 3.14159265
print(f"{pi:.2f}")       # => 3.14        (2 decimal places)
print(f"{pi:10.2f}")     # =>       3.14  (width 10, right-aligned)
print(f"{pi:.0f}")       # => 3           (no decimals, rounds)

n = 1234567
print(f"{n:,}")          # => 1,234,567   (thousands separator)
print(f"{n:_}")          # => 1_234_567   (underscore separator)

ratio = 0.8472
print(f"{ratio:.1%}")    # => 84.7%       (percentage)

# Integer bases:
print(f"{255:b}")        # => 11111111    (binary)
print(f"{255:o}")        # => 377         (octal)
print(f"{255:x}")        # => ff          (hex, lowercase)
print(f"{255:X}")        # => FF          (hex, uppercase)
print(f"{255:#x}")       # => 0xff        (# adds the 0x prefix)

# Sign control:
print(f"{5:+d}")         # => +5          (always show sign)
print(f"{-5:+d}")        # => -5
```

**Alignment & padding:** `<` left, `>` right, `^` center, `=` pad after sign. A character before the alignment symbol is the fill.

```python
print(f"{'hi':<10}|")    # => hi        |   (left, width 10)
print(f"{'hi':>10}|")    # => |        hi   ... actually: |        hi
print(f"{'hi':>10}|")    # => "        hi|"  (right-aligned)
print(f"{'hi':^10}|")    # => "    hi    |"  (centered)
print(f"{'hi':*^10}|")   # => "****hi****|"  (centered, * fill)
print(f"{42:08d}")       # => 00000042       (zero-padded width 8)
print(f"{42:>08}")       # => 00000042
```

**Dynamic (nested) format values:** you can compute width/precision from variables.

```python
width = 12
prec = 3
val = 3.14159
print(f"{val:{width}.{prec}f}|")   # => "       3.142|"
```

**Conversion flags** `!r` (repr), `!s` (str), `!a` (ascii) come before the colon:

```python
s = "hi\n"
print(f"{s!r}")          # => 'hi\n'   (repr — shows the escape, with quotes)
print(f"{s!s}", end="")  # => hi       (str — actual newline printed)
```

### A formatted table example

```python
rows = [("Alice", 91), ("Bob", 7), ("Charlie", 100)]
for name, score in rows:
    print(f"{name:<10}{score:>5}")
# => Alice        91
# => Bob           7
# => Charlie     100
```

## Core concept: Legacy formatting (read-only awareness)

You will meet two older styles in existing code. Know how to read them; prefer f-strings for new code.

### `str.format()`

```python
print("{} is {}".format("Ada", 36))            # => Ada is 36
print("{0} {1} {0}".format("a", "b"))          # => a b a   (positional)
print("{name} is {age}".format(name="Ada", age=36))  # => Ada is 36
print("{:.2f}".format(3.14159))                # => 3.14    (same spec mini-language as f-strings)
```

`str.format()` is useful when the template is stored separately from the data (e.g., loaded from config) — something f-strings can't do because they evaluate immediately.

### `%`-formatting (printf-style, like PHP's `sprintf`)

```python
print("%s is %d" % ("Ada", 36))     # => Ada is 36
print("%.2f" % 3.14159)             # => 3.14
print("%05d" % 42)                  # => 00042
print("%(name)s" % {"name": "Ada"}) # => Ada   (named)
```

This is the closest match to PHP's `sprintf("%s is %d", ...)`. It still appears in older logging code, but f-strings are clearer and faster.

> **Logging caveat:** the `logging` module uses `%`-style lazy formatting on purpose — write `log.info("user %s", user_id)`, *not* an f-string, so the string is only built if the log level is active.

## Core concept: Common string methods

Methods return new strings (immutability!). Here is the working toolbox.

### Case

```python
s = "Hello World"
print(s.upper())        # => HELLO WORLD
print(s.lower())        # => hello world
print(s.title())        # => Hello World
print(s.capitalize())   # => Hello world  (only first char up, rest down)
print(s.swapcase())     # => hELLO wORLD
print("HELLO".casefold())  # => hello   (aggressive lower; use for caseless compare)
```

### Trimming whitespace

```python
print("  hi  ".strip())     # => "hi"      (both ends)
print("  hi  ".lstrip())    # => "hi  "    (left only)
print("  hi  ".rstrip())    # => "  hi"    (right only)
print("xxhixx".strip("x"))  # => "hi"      (strip given chars, not just spaces)
print("file.txt".removesuffix(".txt"))   # => "file"  (3.9+)
print("v1.2.3".removeprefix("v"))        # => "1.2.3"
```

### Splitting & joining

```python
csv = "a,b,c"
print(csv.split(","))           # => ['a', 'b', 'c']
print("a b  c".split())         # => ['a', 'b', 'c']  (no arg = split on any run of whitespace)
print("a,b,c,d".split(",", 1))  # => ['a', 'b,c,d']   (maxsplit=1)
print("a,b,c,d".rsplit(",", 1)) # => ['a,b,c', 'd']   (split from the right)

print("line1\nline2".splitlines())  # => ['line1', 'line2']

# join is a method ON the separator:
print(",".join(["a", "b", "c"]))    # => a,b,c
print(" ".join(["x", "y"]))         # => x y
# print(",".join([1, 2]))  # TypeError: items must be str — convert first:
print(",".join(str(n) for n in [1, 2, 3]))  # => 1,2,3
```

`split`/`join` are Python's `explode`/`implode`. Note `join`'s argument order is reversed from PHP: the separator owns the method.

### Searching & testing

```python
s = "hello world"
print(s.find("o"))        # => 4    (index of first match, -1 if absent)
print(s.find("o", 5))     # => 7    (start searching at index 5)
print(s.rfind("o"))       # => 7    (last match)
print(s.find("z"))        # => -1   (not found — no error)
# print(s.index("z"))     # ValueError: substring not found (index raises!)

print(s.count("o"))       # => 2    (non-overlapping occurrences)
print("hi" in s)          # => False (membership test — preferred for yes/no)
print("world" in s)       # => True

print(s.startswith("hell"))   # => True
print(s.endswith(".py"))      # => False
print(s.startswith(("a", "h")))  # => True  (tuple = any of these)
```

Use `in` when you only need yes/no, and `find` when you need the position. `find` returns `-1` for "missing"; `index` raises `ValueError`.

### Replacing

```python
print("a-b-c".replace("-", "_"))         # => a_b_c
print("aaa".replace("a", "b", 2))        # => bba   (only first 2)
print("a b c".replace(" ", ""))          # => abc   (remove spaces)
```

### Padding & numeric strings

```python
print("42".zfill(5))        # => 00042   (zero-pad on the left, respects sign)
print("-3".zfill(5))        # => -0003
print("hi".center(10, ".")) # => ....hi....
print("hi".ljust(6, "."))   # => hi....
print("hi".rjust(6, "."))   # => ....hi
```

### Validation predicates (all return bool)

```python
print("12345".isdigit())     # => True
print("abc123".isalnum())    # => True
print("abc".isalpha())       # => True
print("   ".isspace())       # => True
print("Hello".istitle())     # => True
print("hello".islower())     # => True
```

> **Gotcha:** `isdigit()` accepts some non-ASCII digit characters and rejects `-`/`.`/`+`. To safely test "is this a valid int/float" use a `try/except int(...)`, not `isdigit`.

## Core concept: Membership & iteration

```python
s = "abc"
for ch in s:             # strings are iterable, char by char
    print(ch, end=" ")   # => a b c
print()

print("b" in s)          # => True   (substring/char membership)
print("ab" in s)         # => True   (works for substrings too)
print("ba" in s)         # => False

# Enumerate when you need the index:
for i, ch in enumerate("abc"):
    print(i, ch)
# => 0 a
# => 1 b
# => 2 c

# Build a list of chars (rarely needed, but instructive):
print(list("abc"))       # => ['a', 'b', 'c']
```

## Core concept: Escape sequences & raw strings

Backslash starts an escape sequence inside normal string literals.

| Escape | Meaning            |
|--------|--------------------|
| `\n`   | newline            |
| `\t`   | tab                |
| `\\`   | literal backslash  |
| `\'` `\"` | literal quote   |
| `\r`   | carriage return    |
| `\0`   | null character     |
| `\uXXXX` | Unicode by code point |
| `\N{NAME}` | Unicode by name  |

```python
print("a\tb\nc")
# => a    b
# => c
print("C:\\Users\\me")   # => C:\Users\me
print("é")          # => é
print("\N{GREEK SMALL LETTER PI}")  # => π
```

### Raw strings: `r"..."`

A raw string disables escape processing — every backslash is literal. Indispensable for **regex patterns** and **Windows paths**.

```python
import re
pattern = r"\d+\.\d+"          # without r"" you'd need "\\d+\\.\\d+"
print(re.findall(pattern, "v3.14 and 2.71"))  # => ['3.14', '2.71']

path = r"C:\Users\new\test"    # \n and \t are NOT special here
print(path)                    # => C:\Users\new\test
```

> **Edge case:** a raw string still can't *end* with an odd number of backslashes (`r"\"` is a syntax error) because the backslash escapes the closing quote at the tokenizer level. Combine instead: `r"path" + "\\"`.

You can combine prefixes: `rf"..."` (or `fr"..."`) is a raw f-string — common for regex templates with interpolated parts.

```python
digits = 3
print(rf"\d{{{digits}}}")      # => \d{3}   (raw f-string; {{ }} for literal braces)
```

## Core concept: Unicode, bytes & encoding

Python 3 `str` is a sequence of **Unicode code points** — not bytes. When you read/write files or talk over a network, you cross between `str` and `bytes` via **encoding** (str → bytes) and **decoding** (bytes → str). UTF-8 is the near-universal choice.

```python
s = "café"
print(len(s))                 # => 4   (4 code points)

data = s.encode("utf-8")      # str -> bytes
print(data)                   # => b'caf\xc3\xa9'  (the é is 2 bytes in UTF-8)
print(len(data))              # => 5   (5 bytes)

back = data.decode("utf-8")   # bytes -> str
print(back)                   # => café
print(back == s)              # => True

# Bytes literals use a b prefix and behave like a separate type:
raw = b"hello"
print(type(raw))              # => <class 'bytes'>
print(raw[0])                 # => 104  (indexing bytes gives ints, not str!)
```

Key takeaways:
- `len(str)` counts characters; `len(bytes)` counts bytes. They can differ.
- Always specify the encoding explicitly (`encode("utf-8")`, `open(path, encoding="utf-8")`) — never rely on the platform default.
- A `UnicodeDecodeError` means bytes don't match the encoding you claimed. `decode("utf-8", errors="replace")` substitutes `�` instead of crashing.

This is a real shift from PHP, where a "string" is just bytes and you reach for `mb_*` functions to be multibyte-aware. In Python, `str` is already Unicode-aware; bytes are the explicit opt-out.

## PHP → Python string function map

| PHP                          | Python                                  |
|------------------------------|-----------------------------------------|
| `strlen($s)`                 | `len(s)`                                |
| `strtoupper` / `strtolower`  | `s.upper()` / `s.lower()`              |
| `ucfirst`                    | `s.capitalize()`                        |
| `ucwords`                    | `s.title()`                             |
| `trim` / `ltrim` / `rtrim`   | `s.strip()` / `lstrip()` / `rstrip()`  |
| `substr($s, 2, 3)`           | `s[2:5]` (start + length → start:stop)  |
| `strpos`                     | `s.find(...)` (or `.index()` to raise) |
| `str_contains` (PHP 8)       | `sub in s`                              |
| `str_replace`                | `s.replace(old, new)`                   |
| `explode(",", $s)`           | `s.split(",")`                          |
| `implode(",", $arr)`         | `",".join(arr)`                         |
| `str_repeat($s, 3)`          | `s * 3`                                  |
| `str_pad` (left)             | `s.rjust(n, "0")` / `s.zfill(n)`        |
| `sprintf("%s", $x)`          | `f"{x}"` / `"%s" % x`                   |
| `strrev`                     | `s[::-1]`                                |
| `substr_count`               | `s.count(...)`                          |
| `str_starts_with` (PHP 8)    | `s.startswith(...)`                     |

## Common mistakes & gotchas

- **Trying to mutate a string in place.** `s[0] = "X"` raises `TypeError`. Build a new string instead.
- **`+=` in a loop to build big strings.** Quadratic time. Accumulate in a list and `"".join(...)`.
- **Confusing `find` and `index`.** `find` returns `-1` when missing; `index` raises `ValueError`. Don't write `if s.find(x):` — index `0` is falsy! Use `if x in s:` or `if s.find(x) != -1:`.
- **`join` argument order.** It's `separator.join(iterable)`, and every element must already be a `str` — convert numbers first.
- **`split()` with no arg vs `split(" ")`.** No arg collapses runs of whitespace and ignores leading/trailing; `split(" ")` keeps empty strings between consecutive spaces.
- **Slice `stop` is exclusive.** `s[0:2]` gives 2 chars, not 3. Out-of-range slices clamp silently (no error), but out-of-range *indexing* raises `IndexError`.
- **Forgetting `{{`/`}}` for literal braces** in f-strings — a lone `{` is a syntax error.
- **f-strings can't be reused as templates** — they evaluate immediately. For stored templates use `str.format()` or `string.Template`.
- **Using f-strings in `logging` calls.** Prefer lazy `%` args so the message is only formatted when needed (and to avoid logging-injection issues).
- **`isdigit()` is not "is a valid number."** It rejects signs and decimal points and accepts exotic Unicode digits. Validate with `int()`/`float()` in a `try/except`.
- **Mixing `str` and `bytes`.** `"a" + b"b"` raises `TypeError`. Decode bytes first.

## Best practices

- Default to **f-strings** for formatting; they're the most readable and the fastest.
- Use `"".join(parts)` to assemble many fragments; reserve `+` for two or three pieces.
- Prefer `in` for presence checks; use `find`/`index` only when you need the position.
- Use **raw strings** (`r"..."`) for every regex pattern and Windows path.
- Always pass `encoding="utf-8"` explicitly to `open`, `encode`, and `decode`.
- Use format specs (`:.2f`, `:,`, `:>10`) instead of manual rounding/padding logic.
- For caseless comparison use `a.casefold() == b.casefold()`, not `lower()` (handles more scripts correctly).
- Keep templates that come from config in `str.format()`/`string.Template`, not f-strings.
- Use `removeprefix`/`removesuffix` (3.9+) instead of fragile `[len(x):]` slicing.

## Interview Q&A

**Q: Is there a difference between single and double quotes in Python?**
**A:** No semantic difference — both create `str`. Convention is to pick whichever avoids escaping (use `'...'` for strings containing `"`, and vice versa). This differs from PHP, where double quotes interpolate variables and single quotes don't; in Python *all* interpolation requires the `f` prefix regardless of quote style.

**Q: Are Python strings mutable?**
**A:** No. `str` is immutable; any operation that seems to change a string returns a new one. Immutability makes strings hashable (usable as dict keys / set members) and thread-safe to share, but it means building strings with repeated `+=` in a loop is O(n²) — use a list plus `"".join()`.

**Q: How do you reverse a string?**
**A:** `s[::-1]` — a slice with step `-1`. There's no built-in `reverse()` on `str` because it's immutable; the slice produces a new reversed string.

**Q: What's the difference between `find` and `index`?**
**A:** Both return the position of a substring, but `find` returns `-1` when not found while `index` raises `ValueError`. Use `find` when "missing" is normal flow; use `index` (or guard with `in`) when absence is an error.

**Q: Explain `s[start:stop:step]`. Is `stop` inclusive?**
**A:** It slices from `start` (inclusive) up to `stop` (exclusive), taking every `step`-th element. Negative indices count from the end and a negative step walks backwards. Slices clamp out-of-range bounds instead of erroring, unlike direct indexing.

**Q: When would you use `str.format()` over an f-string?**
**A:** When the template and the data are separated — e.g., a format string loaded from configuration or i18n files. f-strings evaluate their expressions at the point of definition, so they can't act as reusable, externally-supplied templates.

**Q: Why shouldn't you use f-strings in logging calls?**
**A:** The `logging` module does lazy `%`-style formatting — `log.info("user %s", uid)` only builds the message if that level is enabled. An f-string is formatted eagerly every time regardless of level, wasting work and potentially interpolating untrusted data into the log record.

**Q: What's the difference between `str` and `bytes`?**
**A:** `str` is a sequence of Unicode code points (text); `bytes` is a sequence of raw 8-bit values. You convert text to bytes with `.encode("utf-8")` and back with `.decode("utf-8")`. Indexing a `bytes` yields an `int`, not a one-character string. You cannot concatenate `str` and `bytes`.

**Q: What does the `=` do in `f"{x=}"`?**
**A:** It's the debug specifier (Python 3.8+). It prints the expression text followed by `=` and its value, e.g. `x=42` — a quick way to log a variable with its name.

**Q: How do you format a number with thousands separators and two decimals?**
**A:** Use a format spec: `f"{value:,.2f}"` gives `1,234,567.89`. The `,` adds grouping, `.2f` fixes two decimal places.

**Q: What's a raw string and when do you need one?**
**A:** `r"..."` disables backslash escape processing, so `r"\d+"` is the four characters backslash-d-plus rather than an escaped sequence. You need it for regular expressions and Windows file paths. Caveat: a raw string can't end in an odd number of backslashes.

**Q: Difference between `split()` and `split(" ")`?**
**A:** `split()` with no argument splits on any run of whitespace and discards leading/trailing/empty segments, returning no empty strings. `split(" ")` splits strictly on single spaces, so consecutive spaces produce empty-string elements. Use the no-arg form for tokenizing arbitrary whitespace.

## Practice exercises

1. **Slug maker:** Given `title = "  Hello, World!  "`, produce `"hello-world"` (trimmed, lowercased, spaces → hyphens, comma and exclamation removed).
2. **Initials:** Given `name = "ada lovelace"`, output `"A.L."` using split, indexing, and an f-string/join.
3. **Aligned report:** Print this table with names left-aligned in a 12-wide column and prices right-aligned in a 10-wide column with 2 decimals and a thousands separator:
   `data = [("Widget", 1999.5), ("Gadget", 12.0), ("Thingamajig", 1234567.89)]`
4. **Reverse words:** Turn `"the quick brown fox"` into `"fox brown quick the"` (reverse word order, not characters).
5. **Mask a card:** Given `card = "4111111122223456"`, output `"************3456"` (keep last 4, mask the rest), using slicing and `*`.
6. **Byte counter:** For `s = "naïve café"`, print the number of characters and the number of UTF-8 bytes, and explain why they differ.

<details><summary>Solutions</summary>

```python
# 1. Slug maker
title = "  Hello, World!  "
slug = title.strip().lower().replace(",", "").replace("!", "").replace(" ", "-")
print(slug)              # => hello-world

# 2. Initials
name = "ada lovelace"
initials = ".".join(word[0].upper() for word in name.split()) + "."
print(initials)          # => A.L.

# 3. Aligned report
data = [("Widget", 1999.5), ("Gadget", 12.0), ("Thingamajig", 1234567.89)]
for item, price in data:
    print(f"{item:<12}{price:>10,.2f}")
# => Widget        1,999.50
# => Gadget           12.00
# => Thingamajig 1,234,567.89

# 4. Reverse words
sentence = "the quick brown fox"
print(" ".join(sentence.split()[::-1]))   # => fox brown quick the

# 5. Mask a card
card = "4111111122223456"
masked = "*" * (len(card) - 4) + card[-4:]
print(masked)            # => ************3456

# 6. Byte counter
s = "naïve café"
chars = len(s)
byts = len(s.encode("utf-8"))
print(f"{chars=} {byts=}")   # => chars=10 byts=12
# 'ï' and 'é' are each 2 bytes in UTF-8, so 10 code points -> 12 bytes.
```

</details>

## Cheat sheet

```python
# Literals
'x'  "x"  '''multi\nline'''  r"\raw"  f"{val}"  b"bytes"  rf"{x}\d"

# Slicing  s[start:stop:step]   (stop exclusive; negatives from end; slices clamp)
s[0]      s[-1]     s[1:4]    s[:3]    s[2:]    s[:]      s[::-1]   s[::2]

# f-string format specs  {value:[fill][<>^][sign][width][,][.prec][type]}
f"{n:,}"       # 1,234,567       thousands
f"{x:.2f}"     # 3.14            fixed 2 dp
f"{x:.1%}"     # 84.7%           percent
f"{n:08d}"     # 00000042        zero-pad width 8
f"{s:>10}"     # right align     (<left  ^center)
f"{s:*^10}"    # *fill* center
f"{255:#x}"    # 0xff            hex with prefix
f"{v:+d}"      # +5              force sign
f"{x=}"        # x=42            debug
f"{s!r}"       # repr conversion

# Methods (all return new strings)
.upper() .lower() .title() .capitalize() .casefold() .swapcase()
.strip() .lstrip() .rstrip()  .removeprefix(p) .removesuffix(s)
.split(sep, max) .rsplit(...) .splitlines()  sep.join(iter)
.replace(old, new, count)
.find(x) .rfind(x) .index(x)   x in s   .count(x)
.startswith(t) .endswith(t)    (t may be a tuple)
.zfill(n) .center(n, c) .ljust(n, c) .rjust(n, c)
.isdigit() .isalpha() .isalnum() .isspace()

# Encoding
s.encode("utf-8")   ->  bytes
b.decode("utf-8")   ->  str
len(str) = code points;  len(bytes) = bytes
```
