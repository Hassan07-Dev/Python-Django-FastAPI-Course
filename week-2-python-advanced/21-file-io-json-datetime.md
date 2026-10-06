# 21 — File I/O, JSON & datetime

> Week 2 · Python standard library essentials

## What you will learn

- How to open, read, and write files with `open()` and why `with` is non-negotiable
- File modes (`r`/`w`/`a`/`x`), text vs binary, and always specifying `encoding="utf-8"`
- The four ways to read a file and which one to reach for
- `pathlib.Path` — the modern, object-oriented way to build and manipulate paths
- The `json` module: `dumps`/`loads`, `dump`/`load`, pretty-printing, and serializing custom objects (the bread and butter of REST APIs)
- A quick tour of the `csv` module
- The `datetime` family: `date`, `time`, `datetime`, `timedelta`, formatting/parsing, and **timezone-aware** datetimes with `zoneinfo`
- Reading configuration from the environment with `os.environ` / `os.getenv`

## PHP/Laravel bridge

In PHP you reach for `fopen`/`fread`/`fclose`, `file_get_contents`, `json_encode`/`json_decode`, and the `Carbon`/`DateTime` classes. Python's equivalents are cleaner: `open()` plus a `with` block replaces the manual `fopen`/`fclose` pair (it auto-closes like a `try/finally`), `Path.read_text()` is `file_get_contents`, and `json.dumps`/`json.loads` map directly onto `json_encode`/`json_decode` — except Python decodes JSON objects into `dict`s by default, not `stdClass`. Where Laravel gives you `Carbon`, the standard library gives you `datetime` + `zoneinfo` (and the third-party `pendulum` if you miss Carbon's fluent API). And `os.getenv("KEY")` is your `env('KEY')` — though in Python you typically load a `.env` file explicitly with a library like `python-dotenv`.

## Core concept: opening files with `open()`

The built-in `open()` returns a *file object*. Its two most important arguments are the path and the **mode**.

```python
f = open("notes.txt", "r", encoding="utf-8")
content = f.read()
f.close()  # easy to forget — which is exactly the problem
```

This works, but if `read()` raises an exception, `close()` never runs and the file handle leaks. Always prefer the context manager form below.

### File modes

| Mode | Meaning | Creates? | Truncates? | Fails if exists? |
|------|---------|----------|------------|------------------|
| `"r"` | read (default) | no — errors if missing | no | no |
| `"w"` | write | yes | **yes — wipes the file** | no |
| `"a"` | append | yes | no (writes at end) | no |
| `"x"` | exclusive create | yes | n/a | **yes** |
| `"r+"` | read **and** write | no | no | no |

Add a character for the data type:

- `"t"` — text mode (the default). You read/write `str`, and Python decodes/encodes bytes for you.
- `"b"` — binary mode. You read/write `bytes`. Use for images, PDFs, anything non-text. **No `encoding` allowed in binary mode.**

So `"rb"` = read binary, `"wt"` = write text (same as `"w"`), `"ab"` = append binary.

> **Always pass `encoding="utf-8"` in text mode.** If you omit it, Python uses a platform-dependent default (UTF-8 on modern Linux/macOS, but often `cp1252` on Windows). That mismatch is a classic "works on my machine" bug. Be explicit.

### Always use `with`

The `with` statement guarantees the file is closed even if an exception is raised — equivalent to a `try/finally`. This is the idiomatic, correct way to handle files in Python.

```python
with open("notes.txt", "r", encoding="utf-8") as f:
    content = f.read()
# f is automatically closed here, exception or not
print(content)
```

You can open several files in one `with`:

```python
with (
    open("input.txt", "r", encoding="utf-8") as src,
    open("output.txt", "w", encoding="utf-8") as dst,
):
    dst.write(src.read())
```

## Core concept: reading and writing

### Four ways to read

```python
with open("poem.txt", "r", encoding="utf-8") as f:
    whole = f.read()          # -> one big str of the entire file
    # (cursor is now at EOF; further reads return "")

with open("poem.txt", "r", encoding="utf-8") as f:
    first_line = f.readline() # -> "Roses are red\n"  (includes the newline)

with open("poem.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()     # -> ["Roses are red\n", "Violets are blue\n", ...]
```

The **best** way for line-by-line processing is to iterate the file object directly. It is lazy (one line at a time, low memory) and the Pythonic default — reach for this instead of `readlines()` on large files:

```python
with open("big_log.txt", "r", encoding="utf-8") as f:
    for line in f:                 # streams line by line, memory-friendly
        clean = line.rstrip("\n")  # strip the trailing newline
        print(clean)
```

### Writing

```python
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Hello\n")             # write() does NOT add a newline for you
    f.write("World\n")
    f.writelines(["a\n", "b\n"])   # writes an iterable of strings (no newlines added)

# Appending instead of overwriting:
with open("output.txt", "a", encoding="utf-8") as f:
    f.write("appended line\n")
```

`print()` can also write to a file via its `file=` argument — handy because it *does* add newlines:

```python
with open("report.txt", "w", encoding="utf-8") as f:
    print("Line one", file=f)      # newline added automatically
    print("Line two", file=f)
```

### Binary example

```python
# Copy an image byte-for-byte — no encoding involved
with open("logo.png", "rb") as src, open("logo_copy.png", "wb") as dst:
    dst.write(src.read())
```

## Core concept: `pathlib.Path` — the modern path API

Stop concatenating path strings. `pathlib.Path` is the modern, cross-platform, object-oriented way to handle file paths. It works on Windows and POSIX alike and the `/` operator joins path segments.

```python
from pathlib import Path

base = Path("/Users/hassan/project")
config = base / "config" / "settings.json"   # / joins paths, OS-correctly
print(config)                 # => /Users/hassan/project/config/settings.json
print(config.name)            # => settings.json
print(config.stem)            # => settings
print(config.suffix)          # => .json
print(config.parent)          # => /Users/hassan/project/config
print(config.exists())        # => False (probably)
```

Useful constructors and properties:

```python
from pathlib import Path

Path.cwd()           # current working directory
Path.home()          # the user's home directory
Path(__file__)       # the path of the current source file
Path("a/b").is_absolute()   # => False
Path("data").resolve()      # => absolute, symlink-resolved path
```

### Read and write in one line

`Path` has convenience methods that open, read/write, and close for you:

```python
from pathlib import Path

p = Path("greeting.txt")
p.write_text("Hello, world!\n", encoding="utf-8")   # create/overwrite + close
text = p.read_text(encoding="utf-8")                # read whole file + close
print(text)                                         # => Hello, world!

# Binary variants:
data = Path("logo.png").read_bytes()
Path("logo_copy.png").write_bytes(data)
```

### Creating directories and checking existence

```python
from pathlib import Path

out = Path("exports/2026/june")
out.mkdir(parents=True, exist_ok=True)  # like `mkdir -p`; no error if it exists

target = out / "report.txt"
target.write_text("done\n", encoding="utf-8")

print(target.exists())   # => True
print(target.is_file())  # => True
print(out.is_dir())      # => True
target.unlink()          # delete the file (use missing_ok=True to ignore if absent)
```

### Globbing — finding files by pattern

```python
from pathlib import Path

project = Path(".")
for py in project.glob("*.py"):          # non-recursive, this directory only
    print(py)

for py in project.rglob("*.py"):         # recursive — every subdirectory too
    print(py)

# ** also works inside glob():
configs = list(project.glob("**/*.json"))
print(len(configs), "json files found")
```

`pathlib` is the recommended default in modern Python. The older `os.path` (`os.path.join`, `os.path.exists`, ...) still appears everywhere and works fine, but new code should prefer `Path`.

## Core concept: the `json` module

JSON is the lingua franca of REST APIs, so this is the single most important section for a backend developer. The mental model: **`dumps`/`dump` go OUT to JSON; `loads`/`load` come IN from JSON.** The trailing `s` means "string".

| Function | Direction | Works with |
|----------|-----------|------------|
| `json.dumps(obj)` | Python -> JSON | returns a `str` |
| `json.loads(s)`   | JSON -> Python | takes a `str`/`bytes` |
| `json.dump(obj, f)` | Python -> JSON | writes to a file object |
| `json.load(f)`    | JSON -> Python | reads from a file object |

### Strings: `dumps` and `loads`

```python
import json

data = {"name": "Ada", "roles": ["admin", "user"], "active": True, "age": 36}

text = json.dumps(data)
print(text)
# => {"name": "Ada", "roles": ["admin", "user"], "active": true, "age": 36}
print(type(text))   # => <class 'str'>

back = json.loads(text)
print(back["roles"][0])   # => admin
print(type(back))         # => <class 'dict'>
```

Note the type mapping — this is where PHP devs trip up: JSON objects become **`dict`** (not an object), `true`/`false`/`null` become `True`/`False`/`None`, and numbers become `int` or `float`.

| JSON | Python |
|------|--------|
| object `{}` | `dict` |
| array `[]` | `list` |
| string | `str` |
| number (int) | `int` |
| number (real) | `float` |
| `true` / `false` | `True` / `False` |
| `null` | `None` |

### Pretty-printing: `indent` and `sort_keys`

```python
import json

data = {"name": "Ada", "roles": ["admin", "user"], "age": 36}

print(json.dumps(data, indent=2, sort_keys=True))
# => {
# =>   "age": 36,
# =>   "name": "Ada",
# =>   "roles": [
# =>     "admin",
# =>     "user"
# =>   ]
# => }
```

`ensure_ascii=False` keeps non-ASCII characters readable instead of escaping them to `\uXXXX` — useful for names, emoji, and most real-world text:

```python
import json
print(json.dumps({"city": "São Paulo"}))                    # => {"city": "S\u00e3o Paulo"}  (default escapes non-ASCII)
print(json.dumps({"city": "São Paulo"}, ensure_ascii=False)) # => {"city": "São Paulo"}
```

### Files: `dump` and `load`

```python
import json
from pathlib import Path

config = {"debug": False, "max_connections": 20}

# Write JSON to a file
with open("config.json", "w", encoding="utf-8") as f:
    json.dump(config, f, indent=2)

# Read JSON from a file
with open("config.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
print(loaded["max_connections"])   # => 20

# Even shorter with pathlib (round-trip through a string):
Path("config.json").write_text(json.dumps(config, indent=2), encoding="utf-8")
loaded = json.loads(Path("config.json").read_text(encoding="utf-8"))
```

### Serializing custom objects with `default`

By default `json.dumps` only knows how to serialize the built-in JSON types. Throw it a `datetime`, a `Decimal`, or your own class and it raises `TypeError: Object of type ... is not JSON serializable`. The `default=` hook is a function called for any value `json` doesn't recognize; return something serializable from it.

```python
import json
from datetime import datetime, date
from decimal import Decimal

def to_serializable(obj):
    if isinstance(obj, (datetime, date)):
        return obj.isoformat()        # "2026-06-19T14:30:00"
    if isinstance(obj, Decimal):
        return float(obj)
    raise TypeError(f"Type {type(obj).__name__} is not serializable")

payload = {"event": "login", "at": datetime(2026, 6, 19, 14, 30), "fee": Decimal("9.99")}
print(json.dumps(payload, default=to_serializable, indent=2))
# => {
# =>   "event": "login",
# =>   "at": "2026-06-19T14:30:00",
# =>   "fee": 9.99
# => }
```

> In real API work you rarely hand-roll `default` — DRF serializers and Pydantic/FastAPI handle `datetime` and `Decimal` for you. But knowing the hook exists explains *how* those frameworks do it, and it is a common interview question.

### Parsing an API payload

A realistic pattern: a request body arrives as a JSON string (or bytes), you parse it, and you defensively read fields.

```python
import json

raw_body = '{"username": "ada", "tags": ["py", "api"], "limit": 50}'

try:
    body = json.loads(raw_body)
except json.JSONDecodeError as exc:
    # In a real handler you'd return HTTP 400 here
    print(f"Bad JSON: {exc}")
else:
    username = body.get("username")          # .get avoids KeyError -> None if absent
    tags = body.get("tags", [])             # default to empty list
    print(username, tags)                   # => ada ['py', 'api']
```

`json.JSONDecodeError` (a subclass of `ValueError`) is what malformed JSON raises — catch it to return a clean 400 instead of a 500.

## Core concept: the `csv` module (brief)

For comma-separated data, don't split on commas by hand — quoting and escaping will bite you. Use the `csv` module. `DictReader`/`DictWriter` map rows to/from dictionaries, which is usually what you want.

```python
import csv
from pathlib import Path

rows = [
    {"name": "Ada", "age": 36},
    {"name": "Linus", "age": 54},
]

# Write
with open("people.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "age"])
    writer.writeheader()
    writer.writerows(rows)

# Read
with open("people.csv", "r", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])   # values are strings! age is "36", not 36
# => Ada 36
# => Linus 54
```

> Two CSV gotchas: pass `newline=""` to `open()` (the `csv` module handles line endings itself; omitting this can produce blank rows on Windows), and remember every value read back is a **string** — convert types yourself (`int(row["age"])`).

## Core concept: `datetime`

The `datetime` module gives you four core classes:

| Class | Holds | Example |
|-------|-------|---------|
| `date` | year, month, day | `2026-06-19` |
| `time` | hour, minute, second, microsecond | `14:30:00` |
| `datetime` | a date **and** a time | `2026-06-19 14:30:00` |
| `timedelta` | a *duration* / difference | `5 days, 3:00:00` |

```python
from datetime import date, time, datetime, timedelta

d = date(2026, 6, 19)
t = time(14, 30, 0)
dt = datetime(2026, 6, 19, 14, 30, 0)

print(d.year, d.month, d.day)   # => 2026 6 19
print(dt.weekday())             # => 4   (Monday=0 ... Sunday=6; Friday here)
```

### `now()` vs `utcnow()` — and why you should stop using `utcnow()`

```python
from datetime import datetime, timezone

datetime.now()              # local time, NAIVE (no tzinfo attached)
datetime.now(timezone.utc)  # current UTC time, AWARE  <-- prefer this
```

`datetime.utcnow()` is **deprecated as of Python 3.12** because it returns a *naive* datetime that secretly holds UTC values — a footgun that has caused countless bugs. Always use `datetime.now(timezone.utc)` for an aware UTC timestamp.

A **naive** datetime has no timezone info (`.tzinfo is None`); an **aware** one carries a timezone. Aware datetimes are the only safe choice for anything that crosses timezones, gets stored, or gets compared.

### `timedelta` arithmetic

```python
from datetime import datetime, timedelta, timezone

now = datetime(2026, 6, 19, 12, 0, tzinfo=timezone.utc)

tomorrow = now + timedelta(days=1)
in_90_min = now + timedelta(hours=1, minutes=30)
token_expiry = now + timedelta(minutes=15)

print(tomorrow)        # => 2026-06-20 12:00:00+00:00
print(in_90_min)       # => 2026-06-19 13:30:00+00:00

# Subtracting two datetimes yields a timedelta:
delta = datetime(2026, 6, 25, tzinfo=timezone.utc) - now
print(delta.days)            # => 5
print(delta.total_seconds()) # => 466200.0
```

### Formatting and parsing: `strftime` / `strptime`

- `strftime` = **f**ormat: datetime -> string
- `strptime` = **p**arse: string -> datetime

```python
from datetime import datetime

dt = datetime(2026, 6, 19, 14, 30, 45)

# Format a datetime into a custom string
print(dt.strftime("%Y-%m-%d %H:%M:%S"))   # => 2026-06-19 14:30:45
print(dt.strftime("%A, %d %B %Y"))         # => Friday, 19 June 2026
print(dt.strftime("%I:%M %p"))             # => 02:30 PM

# Parse a string back into a datetime (format must match exactly)
parsed = datetime.strptime("2026-06-19 14:30:45", "%Y-%m-%d %H:%M:%S")
print(parsed.year)   # => 2026
```

Common format codes (these come up constantly):

| Code | Means | Example |
|------|-------|---------|
| `%Y` | 4-digit year | `2026` |
| `%y` | 2-digit year | `26` |
| `%m` | month (01-12) | `06` |
| `%d` | day (01-31) | `19` |
| `%H` | hour 24h (00-23) | `14` |
| `%I` | hour 12h (01-12) | `02` |
| `%M` | minute | `30` |
| `%S` | second | `45` |
| `%p` | AM/PM | `PM` |
| `%A` / `%a` | weekday full / abbrev | `Friday` / `Fri` |
| `%B` / `%b` | month full / abbrev | `June` / `Jun` |
| `%z` / `%Z` | UTC offset / tz name | `+0000` / `UTC` |

### ISO 8601 — the format to use in APIs

For machine-to-machine exchange (JSON APIs, databases), use **ISO 8601**, not custom `strftime` patterns. `datetime` has first-class support:

```python
from datetime import datetime, timezone

dt = datetime(2026, 6, 19, 14, 30, tzinfo=timezone.utc)

iso = dt.isoformat()
print(iso)                                  # => 2026-06-19T14:30:00+00:00

# Parse ISO strings (Python 3.11+ even accepts a trailing 'Z'):
back = datetime.fromisoformat("2026-06-19T14:30:00+00:00")
zulu = datetime.fromisoformat("2026-06-19T14:30:00Z")
print(back == zulu)   # => True

# Unix timestamps (seconds since 1970-01-01 UTC):
print(dt.timestamp())                             # => 1781879400.0
print(datetime.fromtimestamp(1781879400, tz=timezone.utc))  # => 2026-06-19 14:30:00+00:00
```

### Timezone-aware datetimes with `zoneinfo`

Since Python 3.9 the standard library ships `zoneinfo`, giving you the full IANA timezone database (`"Europe/London"`, `"America/New_York"`, ...) with no third-party dependency. This is how you correctly convert between zones, including daylight saving time.

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

# An aware UTC instant
utc_now = datetime(2026, 6, 19, 14, 30, tzinfo=timezone.utc)

# Convert it to other zones — same instant, different wall-clock display
london = utc_now.astimezone(ZoneInfo("Europe/London"))
new_york = utc_now.astimezone(ZoneInfo("America/New_York"))

print(london)     # => 2026-06-19 15:30:00+01:00   (BST, summer = UTC+1)
print(new_york)   # => 2026-06-19 10:30:00-04:00   (EDT, summer = UTC-4)

# Attach a zone to a naive datetime (interpret these wall-clock numbers AS Tokyo time):
naive = datetime(2026, 6, 19, 9, 0)
tokyo = naive.replace(tzinfo=ZoneInfo("Asia/Tokyo"))
print(tokyo)                                    # => 2026-06-19 09:00:00+09:00
print(tokyo.astimezone(timezone.utc))           # => 2026-06-19 00:00:00+00:00
```

> The golden rule: **store and compute in UTC; convert to local zones only for display.** Comparing or subtracting a naive and an aware datetime raises `TypeError`, which is the language protecting you from a silent bug.

## Core concept: `os` and environment variables

Configuration (secrets, DB URLs, feature flags) should come from the *environment*, not be hard-coded — this is core to the [Twelve-Factor App](https://12factor.net/config) approach that frameworks like Django and FastAPI assume.

```python
import os

# os.environ is a dict-like mapping of all environment variables
print(os.environ["PATH"])          # KeyError if the var is missing

# os.getenv is the safe accessor — returns None (or a default) instead of raising
db_url = os.getenv("DATABASE_URL")                       # None if unset
debug = os.getenv("DEBUG", "false")                      # default when unset
port = int(os.getenv("PORT", "8000"))                    # env values are ALWAYS strings

# Setting a variable for the current process (and its children):
os.environ["FEATURE_X"] = "1"

# A safe boolean parse — "false"/"0"/"" must read as False:
def env_bool(name: str, default: bool = False) -> bool:
    val = os.getenv(name)
    if val is None:
        return default
    return val.strip().lower() in {"1", "true", "yes", "on"}

print(env_bool("DEBUG"))   # depends on your environment
```

Set them in your shell before running:

```bash
export DATABASE_URL="postgres://localhost/app"
export DEBUG=true
python app.py
```

In real projects you keep these in a `.env` file (never committed) and load them with `python-dotenv`, or let your framework's settings layer (`pydantic-settings` for FastAPI, `django-environ` for Django) read them. The key takeaways: **every env value is a string** (cast it yourself) and **`getenv` won't crash on a missing key** the way `os.environ[...]` will.

## Common mistakes & gotchas

- **Forgetting `with`.** A raw `open()` without a matching `close()` leaks file handles; an exception between them skips the `close()` entirely. Use `with`.
- **Using `"w"` when you meant `"a"`.** `"w"` silently truncates the file to empty before writing. To add to a file, use `"a"`.
- **Omitting `encoding="utf-8"`.** The default is platform-dependent and will corrupt non-ASCII text on some machines. Always specify it in text mode.
- **`write()` doesn't add newlines.** Unlike `print()`, you must include `"\n"` yourself.
- **Confusing `dump`/`dumps` and `load`/`loads`.** The `s` means "string". `dump`/`load` take a *file object*; `dumps`/`loads` take/return a *string*.
- **`json.dumps` on a `datetime`/`Decimal` raises `TypeError`.** Pass a `default=` function, or serialize via your framework's tooling.
- **`datetime.utcnow()` is deprecated and naive.** Use `datetime.now(timezone.utc)` for an aware UTC instant.
- **Mixing naive and aware datetimes.** Subtracting/comparing them raises `TypeError`. Pick aware everywhere and stick to UTC internally.
- **`strptime` format must match the input exactly**, character for character, or it raises `ValueError`.
- **Forgetting env values are strings.** `os.getenv("PORT")` is `"8000"`, not `8000` — cast with `int(...)`.
- **CSV without `newline=""`** can yield blank rows between every record on Windows.

## Best practices

- Always use `with` for files; always pass `encoding="utf-8"` in text mode.
- Prefer `pathlib.Path` over string concatenation and `os.path` for new code.
- Iterate the file object (`for line in f`) for large files instead of `read()`/`readlines()`.
- Use **ISO 8601** (`isoformat()` / `fromisoformat()`) for datetimes in APIs and storage — never locale-dependent custom formats.
- Store and compute in **UTC**; convert to local zones with `zoneinfo` only at the display edge. Keep datetimes timezone-aware.
- Read config from the environment with `os.getenv` and a sensible default; cast types explicitly.
- When parsing untrusted JSON, wrap `json.loads` in `try/except json.JSONDecodeError` and use `.get()` for optional fields.
- Set `ensure_ascii=False` when producing human-facing JSON with non-ASCII text.

## Interview Q&A

**Q: Why use a `with` statement to open files instead of calling `open()` and `close()` yourself?**
**A:** `with` invokes the file object's context-manager protocol, which guarantees `close()` runs when the block exits — even if an exception is raised inside it. It's the equivalent of a `try/finally` and prevents leaked file handles. It's the idiomatic and safest way to handle files in Python.

**Q: What's the difference between `"w"` and `"a"` mode?**
**A:** `"w"` opens for writing and truncates the file to zero length first (you lose existing contents). `"a"` opens for appending — writes always go to the end and existing content is preserved. Both create the file if it doesn't exist.

**Q: Why should you always pass `encoding="utf-8"`?**
**A:** In text mode Python encodes/decodes bytes using a default encoding that is platform-dependent (e.g. `cp1252` on Windows, UTF-8 elsewhere). Relying on it produces non-portable code that corrupts non-ASCII text. Being explicit makes behavior deterministic across machines.

**Q: What's the difference between `json.dump` and `json.dumps`?**
**A:** `dumps` serializes a Python object to a JSON **string** and returns it; `dump` writes the JSON directly to a **file object**. The trailing `s` stands for "string". The same distinction applies to `load` (from a file) vs `loads` (from a string).

**Q: How does Python map JSON types to its own types when you call `json.loads`?**
**A:** JSON objects become `dict`, arrays become `list`, strings become `str`, integers become `int`, reals become `float`, `true`/`false` become `True`/`False`, and `null` becomes `None`. Unlike PHP's `json_decode`, objects become dictionaries by default, not custom objects.

**Q: How do you JSON-serialize a `datetime` or `Decimal`?**
**A:** `json.dumps` can't handle them natively and raises `TypeError`. Pass a `default=` callable that converts unknown types into serializable ones — e.g. return `obj.isoformat()` for datetimes and `float(obj)` for Decimals. In practice, DRF serializers or Pydantic do this for you.

**Q: Why is `datetime.utcnow()` discouraged?**
**A:** It returns a *naive* datetime (no `tzinfo`) that merely *contains* UTC values, which leads to bugs when it's mixed with aware datetimes or interpreted as local time. It's deprecated in Python 3.12. Use `datetime.now(timezone.utc)` to get a timezone-**aware** UTC datetime.

**Q: What's the difference between a naive and an aware datetime?**
**A:** A naive datetime has `tzinfo is None` — it carries no timezone, so its absolute meaning is ambiguous. An aware datetime carries timezone info and represents an unambiguous instant. You can't safely compare or subtract a naive and an aware datetime; doing so raises `TypeError`.

**Q: How do you convert a datetime from one timezone to another?**
**A:** Make sure it's aware, then call `.astimezone(ZoneInfo("Area/City"))`. `zoneinfo` (stdlib since 3.9) provides the full IANA database, so DST is handled correctly. The instant in time doesn't change — only the wall-clock representation.

**Q: What's the difference between `strftime` and `strptime`?**
**A:** `strftime` formats a datetime *to* a string using format codes (`%Y`, `%m`, ...). `strptime` parses a string *into* a datetime using a matching format. Mnemonic: `f` = format (out), `p` = parse (in).

**Q: How should an app read its configuration, and what's the catch with env vars?**
**A:** Read it from environment variables (the Twelve-Factor approach) via `os.getenv("KEY", default)`, which returns `None`/the default instead of raising on a missing key (unlike `os.environ["KEY"]`). The catch: every value is a **string**, so cast non-string config explicitly (`int(...)`, a boolean parser, etc.).

**Q: Why prefer `pathlib` over `os.path`?**
**A:** `pathlib.Path` is object-oriented and cross-platform: the `/` operator joins segments correctly on any OS, and methods like `.exists()`, `.read_text()`, `.glob()`, and `.parent` read clearly. It replaces a grab-bag of `os.path` functions with one cohesive type and is the recommended modern API.

## Practice exercises

1. **Word count.** Write a function `count_words(path)` that opens a UTF-8 text file using `with`, iterates it line by line, and returns the total number of whitespace-separated words. Don't load the whole file with `read()`.

2. **JSON round-trip.** Given `user = {"name": "Ada", "joined": date(2026, 6, 19), "score": Decimal("88.5")}`, write it to `user.json` (pretty-printed, dates as ISO strings, Decimal as float), then read it back and print the name.

3. **Token expiry.** Write `make_token_expiry(minutes=15)` that returns the ISO-8601 string of "now in UTC plus N minutes", and `is_expired(iso_str)` that parses it and returns `True`/`False`. Use aware UTC datetimes.

4. **Timezone scheduler.** Given a meeting at `2026-11-01 09:00` London time, print the equivalent wall-clock time in `America/New_York` and `Asia/Tokyo`. (Bonus: note that Nov 1 is after the UK leaves summer time, so London is UTC+0.)

5. **Config loader.** Write `load_config()` that reads `APP_NAME` (default `"myapp"`), `PORT` (default `8000`, as an `int`), and `DEBUG` (default `False`, parsed from strings like `"true"`/`"1"`) from the environment and returns them as a `dict`.

6. **CSV to JSON.** Read `people.csv` (columns `name,age`) with `csv.DictReader`, convert each `age` to an `int`, and write the list of dicts to `people.json` with `indent=2`.

<details>
<summary>Solutions</summary>

```python
# 1. Word count
def count_words(path):
    total = 0
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            total += len(line.split())   # split() splits on any whitespace
    return total
```

```python
# 2. JSON round-trip
import json
from datetime import date
from decimal import Decimal
from pathlib import Path

def default(obj):
    if isinstance(obj, date):
        return obj.isoformat()
    if isinstance(obj, Decimal):
        return float(obj)
    raise TypeError(f"{type(obj).__name__} not serializable")

user = {"name": "Ada", "joined": date(2026, 6, 19), "score": Decimal("88.5")}
Path("user.json").write_text(json.dumps(user, indent=2, default=default), encoding="utf-8")

loaded = json.loads(Path("user.json").read_text(encoding="utf-8"))
print(loaded["name"])   # => Ada
```

```python
# 3. Token expiry
from datetime import datetime, timedelta, timezone

def make_token_expiry(minutes: int = 15) -> str:
    return (datetime.now(timezone.utc) + timedelta(minutes=minutes)).isoformat()

def is_expired(iso_str: str) -> bool:
    expiry = datetime.fromisoformat(iso_str)
    return datetime.now(timezone.utc) >= expiry

token = make_token_expiry(0)   # expires immediately
print(is_expired(token))       # => True (or True a hair later)
```

```python
# 4. Timezone scheduler
from datetime import datetime
from zoneinfo import ZoneInfo

meeting = datetime(2026, 11, 1, 9, 0, tzinfo=ZoneInfo("Europe/London"))
print(meeting.astimezone(ZoneInfo("America/New_York")))  # => 2026-11-01 04:00:00-05:00 (EST; US DST ends Nov 1)
print(meeting.astimezone(ZoneInfo("Asia/Tokyo")))        # => 2026-11-01 18:00:00+09:00
```

```python
# 5. Config loader
import os

def load_config() -> dict:
    def env_bool(name, default=False):
        val = os.getenv(name)
        if val is None:
            return default
        return val.strip().lower() in {"1", "true", "yes", "on"}

    return {
        "app_name": os.getenv("APP_NAME", "myapp"),
        "port": int(os.getenv("PORT", "8000")),
        "debug": env_bool("DEBUG", False),
    }

print(load_config())
```

```python
# 6. CSV to JSON
import csv
import json
from pathlib import Path

people = []
with open("people.csv", "r", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        row["age"] = int(row["age"])
        people.append(row)

Path("people.json").write_text(json.dumps(people, indent=2), encoding="utf-8")
print(people)
```

</details>

## Cheat sheet

```python
# --- Files ---
with open("f.txt", "r", encoding="utf-8") as f:   # modes: r w a x  +  t/b
    data = f.read()                                # .read() .readline() .readlines()
    for line in f: ...                             # best for large files
with open("f.txt", "w", encoding="utf-8") as f:
    f.write("x\n")                                 # write() adds NO newline

# --- pathlib ---
from pathlib import Path
p = Path("dir") / "file.json"                      # join with /
p.exists(); p.is_file(); p.parent; p.name; p.suffix
p.write_text("hi", encoding="utf-8"); p.read_text(encoding="utf-8")
Path("out").mkdir(parents=True, exist_ok=True)
list(Path(".").rglob("*.py"))                      # recursive glob

# --- json ---  (s = string)
import json
json.dumps(obj, indent=2, sort_keys=True, ensure_ascii=False, default=fn)  # -> str
json.loads(s)                                      # -> dict/list/...
json.dump(obj, fileobj); json.load(fileobj)        # file variants
# except json.JSONDecodeError  for bad input

# --- datetime ---
from datetime import datetime, date, timedelta, timezone
from zoneinfo import ZoneInfo
datetime.now(timezone.utc)                         # aware UTC (NOT utcnow())
dt + timedelta(days=1, minutes=15)                 # arithmetic
dt.strftime("%Y-%m-%d %H:%M")                      # datetime -> str
datetime.strptime("2026-06-19", "%Y-%m-%d")        # str -> datetime
dt.isoformat(); datetime.fromisoformat(s)          # ISO 8601 (APIs!)
dt.astimezone(ZoneInfo("America/New_York"))        # convert zones

# --- os / env ---
import os
os.getenv("KEY", "default")                        # safe, returns str or default
os.environ["KEY"]                                  # raises KeyError if missing
int(os.getenv("PORT", "8000"))                     # cast! env values are strings
```

| Format code | `%Y` | `%m` | `%d` | `%H` | `%M` | `%S` | `%I` | `%p` | `%A` | `%B` | `%z` |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Means | year | month | day | hr24 | min | sec | hr12 | AM/PM | weekday | month name | offset |
