# 01 — Getting Started: Install, REPL, venv, pip, Project Layout

> Week 1 · Python fundamentals & tooling

## What you will learn

- How to install Python 3.12 on macOS and verify it (`python3 --version`)
- The difference between `python` and `python3`, and why it matters
- The interactive **REPL**: your scratchpad for trying ideas instantly
- How to run a `.py` script from the terminal
- Why Python uses **indentation instead of braces**, and how comments work
- `print()` for output and `input()` for reading from the user
- **Virtual environments (`venv`)** — Python's answer to per-project dependency isolation (think Composer, but with an extra activation step)
- **pip**: installing packages, `pip list`, `pip freeze`, `requirements.txt`, and `pip install -r`
- A clean **mental map from Composer/PHP to pip/Python**
- A sensible **project folder layout** for a backend/REST service
- Running modules with `python -m`
- `import this` — the Zen of Python, the cultural compass of the language
- Choosing an editor: **VS Code + the Python extension**

## PHP/Laravel bridge

You already know how to build web apps; you just don't know Python's plumbing yet. Here is the headline mapping: **`composer` ≈ `pip`**, **`composer.json` ≈ `requirements.txt`** (or `pyproject.toml`), and **`vendor/` ≈ your virtual environment's `site-packages`**. The big mental shift is that PHP isolates dependencies automatically by putting them in a project-local `vendor/` folder, whereas Python installs globally **by default** — so you must deliberately create and *activate* a virtual environment per project to get the same isolation. There is no `php artisan serve` equivalent yet (that comes with Django/FastAPI), but `python script.py` is your `php script.php`, and the REPL is like `php -a` / `artisan tinker` but used constantly.

---

## Core concept: Installing Python 3.12 on macOS

macOS ships with an old, system-managed Python that you should **never** use for development. Install your own. The cleanest way on a Mac is [Homebrew](https://brew.sh):

```bash
# Install Homebrew first if you don't have it (one-time):
# /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install python@3.12
```

Verify it worked:

```bash
python3 --version
# => Python 3.12.x

python3 -m pip --version
# => pip 24.x from /opt/homebrew/.../pip (python 3.12)
```

> Alternative installers: the official installer from [python.org](https://www.python.org/downloads/), or a version manager like **pyenv** (`brew install pyenv`) if you need to juggle several Python versions — the Python equivalent of PHP's `phpbrew`/`asdf`. Homebrew is the simplest starting point.

### `python` vs `python3` — the classic beginner trap

On macOS and most Linux distros, `python` historically meant **Python 2** (now dead), and `python3` means Python 3. To avoid surprises, **always type `python3` and `pip3`** at the system level until you are inside a virtual environment.

```bash
python --version     # might error "command not found", or be old Python 2 — avoid
python3 --version    # => Python 3.12.x   <-- use this
```

Here's the nice part: **once you activate a virtual environment (below), plain `python` and `pip` point to the right thing automatically.** So `python3` is a system-level habit; inside a venv, `python` is correct and preferred.

---

## Core concept: The REPL (interactive interpreter)

REPL = **R**ead, **E**val, **P**rint, **L**oop. It's an interactive prompt where each line you type is executed immediately and its result printed. If you've used `php -a` or `php artisan tinker`, this is the same idea — but in Python it's a daily-driver tool, not a curiosity.

Start it by running `python3` with no arguments:

```bash
python3
```

```python
>>> 2 + 2
4
>>> name = "Hassan"
>>> name.upper()
'HASSAN'
>>> len("django")
6
>>> import math
>>> math.sqrt(144)
12.0
>>> _          # the underscore holds the LAST result
12.0
>>> exit()     # or press Ctrl-D to quit
```

Notice you did **not** need `echo` or `print` — the REPL auto-prints the value of any expression (the "Print" in REPL). This only happens in the REPL; inside a `.py` file you must call `print()` explicitly.

> Tip: install **IPython** (`pip install ipython`) for a much nicer REPL with tab-completion, syntax highlighting, and history. It's the spiritual cousin of a souped-up `tinker`.

---

## Core concept: Running a `.py` script

Create a file `hello.py`:

```python
# hello.py — your first script
print("Hello from Python 3.12!")
```

Run it from the terminal:

```bash
python3 hello.py
# => Hello from Python 3.12!
```

That's the whole ceremony. No compile step, no `composer dump-autoload`. A `.py` file is just a sequence of statements executed top to bottom — exactly like a plain PHP script run with `php script.php`.

---

## Core concept: Indentation, comments, no braces, no semicolons

This is the single biggest visual shock coming from PHP. **Python has no `{ }` blocks and no `;` line terminators.** Instead, **indentation defines structure**. A colon `:` opens a block, and the consistently-indented lines beneath it *are* the block.

```python
# A line starting with # is a comment (like // in PHP)
age = 20

if age >= 18:            # the colon starts a block
    print("Adult")       # 4-space indent => inside the if
    print("Can vote")    # still inside the if
print("Done")            # back to column 0 => outside the if
# =>
# Adult
# Can vote
# Done
```

Compare to PHP, where whitespace is cosmetic:

```php
// PHP — braces define the block, indentation is just for humans
if ($age >= 18) {
    echo "Adult";
}
echo "Done";
```

Rules to internalize:

- **Use 4 spaces per indent level.** This is the universal convention (PEP 8). Never mix tabs and spaces — Python will refuse to run with an `IndentationError` or `TabError`.
- **No semicolons.** One statement per line. (You *can* write `a = 1; b = 2` on one line, but don't — it's considered ugly.)
- **Comments** use `#` for single lines. There is no `/* ... */` block comment; for multi-line notes you either prefix each line with `#` or use a triple-quoted string (`"""..."""`) which is technically a string but commonly used as a docstring at the top of functions/modules.

```python
"""
This triple-quoted string at the top of a file or function
is a 'docstring' — documentation that tools can read.
"""
def greet(name):
    """Return a greeting. (This docstring describes the function.)"""
    return f"Hi, {name}"
```

Why did Python do this? The language designers wanted code that *looks* the way it *behaves* — you literally cannot have misleading indentation, because the indentation **is** the logic. It eliminates an entire category of "the braces don't match the indentation" bugs.

---

## Core concept: `print()` and `input()`

`print()` writes to standard output. It's a function (note the parentheses — in old Python 2 it was a statement, which is a common gotcha when reading ancient tutorials).

```python
print("Hello")                       # => Hello
print("a", "b", "c")                 # => a b c   (space-separated by default)
print("a", "b", sep="-")             # => a-b
print("no newline", end="")          # suppresses the trailing newline
name = "Sara"
print(f"Welcome, {name}!")           # => Welcome, Sara!   (f-string interpolation)
```

That `f"..."` is an **f-string** — Python's string interpolation, analogous to PHP's `"Welcome, $name!"` double-quoted strings. We'll cover strings in depth later, but you'll use f-strings constantly.

`input()` reads a line of text from the user. **It always returns a string** — convert it if you need a number:

```python
# greet.py
name = input("What's your name? ")    # prompt is printed, waits for Enter
print(f"Hello, {name}!")

age_text = input("Your age? ")        # e.g. user types 30
age = int(age_text)                   # convert str -> int; int("30") => 30
print(f"Next year you'll be {age + 1}.")
```

```bash
python3 greet.py
# What's your name? Hassan
# Your age? 30
# Hello, Hassan!
# Next year you'll be 31.
```

> Gotcha: `input()` returns a `str`. If you do `input("Age? ") + 1` you'll get a `TypeError` because you can't add an int to a string. Always `int(...)` or `float(...)` the result first.

---

## Core concept: Virtual environments with `venv`

Here's the problem virtual environments solve. By default, `pip install something` installs that package **globally**, shared across every project on your machine. Project A needs Django 5.0, project B needs Django 4.2 — global install means they fight. In PHP/Composer you never think about this because Composer always installs into a project-local `vendor/` directory. Python needs you to opt in to that isolation by creating a **virtual environment**.

A venv is just a folder containing its own copy/link of the Python interpreter and its own `site-packages` (the Python `vendor/`). When "activated", your shell's `python` and `pip` point into that folder.

### Create, activate, deactivate

```bash
# 1. Create a venv named ".venv" inside your project folder (one-time)
python3 -m venv .venv

# 2. Activate it (every new terminal session, macOS/Linux zsh/bash)
source .venv/bin/activate

# Your prompt now shows the env name:
# (.venv) hassan@mac project %

# 3. Now `python` and `pip` (no "3") point INTO the venv:
python --version       # => Python 3.12.x
which python           # => /path/to/project/.venv/bin/python

# 4. When done, leave the environment:
deactivate
```

> On Windows the activate command differs (`.venv\Scripts\activate`), but you're on macOS, so `source .venv/bin/activate` is your line.

| Concept | PHP / Composer | Python / venv |
|---|---|---|
| Where deps live | `vendor/` (per project, automatic) | `.venv/lib/.../site-packages` (per project, **manual create + activate**) |
| Manifest of wanted deps | `composer.json` | `requirements.txt` (or `pyproject.toml`) |
| Exact locked versions | `composer.lock` | `requirements.txt` from `pip freeze` (or a lock file with Poetry/uv) |
| Install everything | `composer install` | `pip install -r requirements.txt` |
| Add one package | `composer require x` | `pip install x` |

**Always add `.venv/` to your `.gitignore`** — just like you never commit `vendor/`. You commit the *manifest* (`requirements.txt`), and teammates recreate the environment.

> Modern alternative worth knowing for interviews: **uv** (`pip install uv`) is an extremely fast Rust-based tool that replaces `venv` + `pip` and creates a `.venv` plus a lock file in one step. The classic `venv` + `pip` workflow above is still the universal baseline every Python dev knows, so learn it first.

---

## Core concept: pip basics & `requirements.txt`

With your venv **activated**, `pip` manages packages from [PyPI](https://pypi.org) (the Python Package Index — Python's Packagist).

```bash
pip install requests                 # install a package
pip install "django==5.1.*"          # pin a version (note the quotes for the shell)
pip install --upgrade requests       # upgrade
pip uninstall requests               # remove

pip list                             # show installed packages + versions
# Package    Version
# ---------- -------
# certifi    2024.x
# requests   2.32.x
# ...

pip show requests                    # detailed info about one package
```

### Capturing dependencies: `pip freeze` → `requirements.txt`

`pip freeze` prints every installed package with its exact version, in the `name==version` format. Redirect it into `requirements.txt` to lock your project's dependencies:

```bash
pip freeze > requirements.txt
```

```text
# requirements.txt  (generated by pip freeze)
certifi==2024.8.30
charset-normalizer==3.4.0
django==5.1.4
idna==3.10
requests==2.32.3
urllib3==2.2.3
```

A teammate (or your production server) then recreates the exact environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt      # the "composer install" of Python
```

> Practical tip: `pip freeze` dumps *everything*, including sub-dependencies you didn't ask for directly. For small projects that's fine and is the standard beginner workflow. For larger projects, people split a hand-written `requirements.in` (top-level wants) from a generated lock, or use tools like `pip-tools`, Poetry, or uv. Start with plain `pip freeze`.

---

## Core concept: A sensible project layout

There's no single mandated structure, but here is a clean, conventional layout for a backend/REST project. The `src/` layout below is widely recommended because it prevents accidentally importing your package before it's installed.

```text
my-api/
├── .venv/                 # virtual environment  (gitignored, like vendor/)
├── .gitignore
├── README.md
├── requirements.txt       # your "composer.json + lock"
├── pyproject.toml         # modern project metadata/config (optional but encouraged)
├── src/
│   └── my_api/            # your actual package (note: underscore, not hyphen)
│       ├── __init__.py    # marks this folder as a Python "package"
│       ├── main.py        # entry point
│       └── services/
│           ├── __init__.py
│           └── users.py
└── tests/
    ├── __init__.py
    └── test_users.py
```

A few things to note:

- **`__init__.py`** is an (often empty) file that tells Python "this directory is an importable package." Think of it loosely like an `index.php` marker or PSR-4 namespace registration — it's how `from my_api.services import users` becomes possible.
- **Package names use underscores** (`my_api`), not hyphens, because the name must be a valid Python identifier you can `import`. The *project* folder can be `my-api`, but the importable package inside is `my_api`.
- A minimal **`.gitignore`** for Python:

```text
# .gitignore
.venv/
__pycache__/
*.pyc
.env
.DS_Store
```

`__pycache__/` and `*.pyc` are Python's compiled bytecode caches — auto-generated, never committed.

---

## Core concept: Running modules with `python -m`

The `-m` flag runs a **module** (an installed/importable thing) as a script, instead of running a file by path. This matters because it runs the module with the correct package context and uses the *currently active* interpreter — which is exactly what you want inside a venv.

```bash
python -m venv .venv          # you already used -m: "run the venv module"
python -m pip install requests# the SAFEST way to call pip — guarantees it's THIS python's pip
python -m http.server 8000    # spin up a quick static file server on :8000
python -m json.tool data.json # pretty-print a JSON file
python -m pytest              # run your test suite (once pytest is installed)
```

> Best practice you'll see in real projects: prefer **`python -m pip ...`** over bare `pip`. If you ever have a confusing setup where `pip` points at a different Python than `python`, `python -m pip` removes all ambiguity by using the pip belonging to *that exact* `python`. Later, Django's `python manage.py runserver` is the same `run-a-script-with-context` idea.

---

## Core concept: `import this` — the Zen of Python

Type this into the REPL once. It's a Python rite of passage and genuinely shapes how the community writes code:

```python
>>> import this
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
...
Readability counts.
...
There should be one-- and preferably only one --obvious way to do it.
...
```

The cultural takeaway versus PHP: Python strongly favors **one obvious, readable way** to do things over many clever ones. When you're choosing between two approaches, the more explicit and readable one is almost always the "Pythonic" choice — and interviewers do listen for that word.

---

## Core concept: Choosing an editor (VS Code)

Use **VS Code** with the official **Python extension** (publisher: Microsoft). It gives you IntelliSense, debugging, linting, and — crucially — venv awareness.

1. Install VS Code, then install the **Python** extension (and **Pylance**, which ships with it) from the Extensions panel.
2. Open your project folder.
3. Run **Command Palette → "Python: Select Interpreter"** and pick the one inside `./.venv`. Now VS Code's terminal auto-activates your venv and IntelliSense resolves your installed packages.

> If autocomplete can't find a package you definitely `pip install`-ed, 95% of the time it's because VS Code is pointed at the wrong interpreter — re-run "Python: Select Interpreter" and choose your `.venv`. This is the #1 beginner confusion.

---

## Common mistakes & gotchas

- **Typing `python` and getting Python 2 or "command not found."** Use `python3` at the system level; inside an activated venv, `python` is correct.
- **Forgetting to activate the venv.** Symptom: `pip install` "succeeds" but your script can't import the package, or installs go to the global Python. Look for `(.venv)` in your prompt before running `pip`.
- **Committing `.venv/` to git.** It's huge and machine-specific. Gitignore it; commit `requirements.txt` instead (your `vendor/` vs `composer.json` instinct applies directly).
- **`IndentationError` / mixing tabs and spaces.** Configure your editor to insert 4 spaces for Tab. VS Code + Python extension does this for you.
- **Treating `input()` output as a number.** It's always a `str`; wrap with `int()`/`float()`.
- **Using `print` without parentheses** (Python 2 habit from old tutorials). It's a function: `print("x")`.
- **Naming a package folder with a hyphen** (`my-api`) — not importable. Use underscores for the importable package name.
- **Running `pip install` outside a venv and polluting the global interpreter.** On Homebrew Python you may even hit an `externally-managed-environment` error — that's Python *protecting* you and nudging you to use a venv. Make the venv.
- **Naming your script the same as a standard module** (e.g. `random.py`, `email.py`). Your file shadows the real library and `import random` breaks. Avoid stdlib names.

## Best practices

- One venv per project, named `.venv`, created with `python3 -m venv .venv`, and gitignored.
- Activate first, then `pip install`. Prefer `python -m pip install ...` to avoid ambiguity.
- Keep `requirements.txt` under version control; regenerate with `pip freeze > requirements.txt` after adding deps.
- Use 4-space indentation everywhere; let the editor enforce it.
- Pin versions for anything heading to production (`django==5.1.4`, not bare `django`) for reproducible builds.
- Add a `README.md` documenting setup steps (`python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`).
- Point VS Code at the project's `.venv` via "Python: Select Interpreter."
- Favor readable, explicit code (the Zen) — interviewers reward "Pythonic."

## Interview Q&A

**Q: What is a Python virtual environment and why use one?**
**A:** It's a self-contained directory with its own Python interpreter and `site-packages`, so each project gets isolated dependencies instead of sharing one global install. This prevents version conflicts between projects and makes builds reproducible. It's the conceptual equivalent of Composer's per-project `vendor/`, except Python doesn't isolate automatically — you create the venv with `python3 -m venv .venv` and activate it.

**Q: What's the difference between `python` and `python3`?**
**A:** Historically `python` referred to Python 2 and `python3` to Python 3. Python 2 is end-of-life, so at the system level you use `python3`/`pip3` to be unambiguous. Inside an activated virtual environment, plain `python` and `pip` correctly point to that environment's interpreter, so you use the short names there.

**Q: How do you record and recreate a project's dependencies?**
**A:** `pip freeze > requirements.txt` captures all installed packages with exact `name==version` pins. Anyone can recreate the environment with `python3 -m venv .venv`, activate it, and `pip install -r requirements.txt`. That pairs with version control: commit `requirements.txt`, gitignore `.venv/`.

**Q: How is pip like and unlike Composer?**
**A:** Both install third-party packages from a central registry (PyPI for pip, Packagist for Composer) and read a manifest. The key difference is isolation: Composer installs into a project-local `vendor/` automatically, while pip installs globally unless you've activated a virtual environment first. So pip + venv together equal Composer's default behavior.

**Q: Why does Python use indentation instead of braces?**
**A:** Indentation *is* the block structure — a colon opens a block and the consistently-indented lines beneath form it. This forces the visual layout to match the actual logic, eliminating a whole class of mismatched-brace/misleading-indentation bugs and enforcing readability, which is a core Python value.

**Q: What does `python -m <module>` do, and why prefer `python -m pip`?**
**A:** `-m` runs an importable module as a script using the currently selected interpreter and proper package context, rather than executing a file by path. `python -m pip` guarantees you're using the pip belonging to *that exact* Python, avoiding the classic "pip installed it but python can't import it" mismatch.

**Q: What is the REPL and when do you use it?**
**A:** The REPL (Read-Eval-Print-Loop) is the interactive interpreter you launch by running `python3` with no arguments. You use it to test snippets, inspect objects, and explore APIs interactively — like `php -a` or `artisan tinker`, but used routinely. It auto-prints expression results; scripts need explicit `print()`.

**Q: What does `input()` return, and what's the common bug?**
**A:** It always returns a `str`, even if the user types digits. The common bug is doing arithmetic on it directly, which raises a `TypeError`; you must convert first with `int()` or `float()`.

**Q: What is `__init__.py` for?**
**A:** It marks a directory as an importable Python package, enabling dotted imports like `from my_api.services import users`. It can be empty, or it can run package-initialization code and define what the package exposes.

**Q: Why shouldn't you commit `.venv/` to version control?**
**A:** It's large, machine- and OS-specific, and fully reproducible from `requirements.txt`. You commit the manifest and gitignore the environment — exactly how you'd gitignore `vendor/` and commit `composer.json`/`composer.lock`.

**Q: What is "the Zen of Python"?**
**A:** A set of guiding aphorisms (run `import this`) emphasizing readability, explicitness, and preferring one obvious way to do something. It shapes idiomatic ("Pythonic") code, favoring clarity over cleverness.

**Q: How do you set up a brand-new project from scratch?**
**A:** Create the folder, run `python3 -m venv .venv`, `source .venv/bin/activate`, `pip install` your dependencies, then `pip freeze > requirements.txt`. Add a `.gitignore` (with `.venv/`, `__pycache__/`), a `src/` package layout with `__init__.py`, a `tests/` folder, and a `README.md` documenting the setup commands.

## Practice exercises

1. **Install & verify.** Install Python 3.12 via Homebrew and confirm `python3 --version` reports 3.12.x. Then check which pip you're using with `python3 -m pip --version`.
2. **REPL warm-up.** In the REPL, compute the number of seconds in a week, uppercase the string `"fastapi"`, and find `len("django rest framework")`. Then run `import this` and read the Zen.
3. **Interactive script.** Write `bmi.py` that asks the user for weight (kg) and height (m) via `input()`, computes BMI = weight / height², and prints it rounded to one decimal using an f-string.
4. **Project bootstrap.** Create a folder `my-api/`, set up a `.venv`, activate it, `pip install requests`, and generate a `requirements.txt`. Confirm `requests` appears in `pip list`.
5. **Module runner.** Use `python -m http.server 8000` to serve the current directory, open `http://localhost:8000` in a browser, then stop it with Ctrl-C.
6. **Layout.** Recreate the `src/` project layout from this lesson (with `__init__.py` files and a `tests/` folder) and write a `.gitignore` that ignores the venv and bytecode caches.

<details><summary>Solutions</summary>

**1.**
```bash
brew install python@3.12
python3 --version            # => Python 3.12.x
python3 -m pip --version     # => pip 24.x ... (python 3.12)
```

**2.**
```python
>>> 60 * 60 * 24 * 7
604800
>>> "fastapi".upper()
'FASTAPI'
>>> len("django rest framework")
21
>>> import this        # prints the Zen of Python
```

**3.**
```python
# bmi.py
weight = float(input("Weight in kg? "))
height = float(input("Height in m? "))
bmi = weight / (height ** 2)        # ** is exponentiation in Python
print(f"Your BMI is {round(bmi, 1)}.")
# Weight in kg? 80
# Height in m? 1.8
# => Your BMI is 24.7.
```

**4.**
```bash
mkdir my-api && cd my-api
python3 -m venv .venv
source .venv/bin/activate
python -m pip install requests
pip list                       # requests should be listed
pip freeze > requirements.txt
deactivate
```

**5.**
```bash
python -m http.server 8000
# Serving HTTP on :: port 8000 (http://[::]:8000/) ...
# (visit http://localhost:8000 in a browser, then press Ctrl-C to stop)
```

**6.**
```bash
mkdir -p my-api/src/my_api/services my-api/tests
cd my-api
touch src/my_api/__init__.py src/my_api/main.py
touch src/my_api/services/__init__.py src/my_api/services/users.py
touch tests/__init__.py tests/test_users.py
printf ".venv/\n__pycache__/\n*.pyc\n.env\n.DS_Store\n" > .gitignore
```

</details>

## Cheat sheet

| Task | Command |
|---|---|
| Check version | `python3 --version` |
| Start REPL | `python3` (exit with `exit()` / Ctrl-D) |
| Run a script | `python3 hello.py` |
| Create venv | `python3 -m venv .venv` |
| Activate venv (macOS) | `source .venv/bin/activate` |
| Deactivate venv | `deactivate` |
| Install a package | `pip install requests` (or `python -m pip install ...`) |
| Pin a version | `pip install "django==5.1.*"` |
| List installed | `pip list` |
| Lock dependencies | `pip freeze > requirements.txt` |
| Install from lock | `pip install -r requirements.txt` |
| Run a module | `python -m http.server 8000` |
| Quick static server | `python -m http.server 8000` |
| The Zen | `python3 -c "import this"` |

**Syntax reminders**

```python
# comment with #
x = 10                       # no $, no ;
if x > 5:                    # colon + 4-space indent (no braces)
    print(f"big: {x}")       # f-string interpolation
name = input("Name? ")       # always returns a str
n = int(input("Num? "))      # convert input to a number
```

**Composer → pip quick map:** `composer install` → `pip install -r requirements.txt` · `composer require x` → `pip install x` · `composer.json` → `requirements.txt` · `vendor/` → `.venv` (gitignored).
