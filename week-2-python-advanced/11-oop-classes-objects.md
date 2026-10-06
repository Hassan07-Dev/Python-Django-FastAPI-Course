# 11 — OOP: Classes & Objects

> Week 2 · Python Advanced — Object-Oriented Programming

## What you will learn

- How to **define a class** in Python and how it differs from PHP's `class` syntax
- The `__init__` **constructor** and the mysterious first parameter `self` (Python's `$this`)
- **Instance attributes vs class attributes** — and the notorious *shared mutable class attribute* trap
- **Instance methods** and how to create (instantiate) objects
- Python's **encapsulation conventions**: `_protected`, `__private` name-mangling, and why Python has *no truly private* members
- `@property` — turning methods into computed/validated attributes with **getter / setter / deleter**
- `@classmethod` (alternative constructors, `cls`) and `@staticmethod`
- The `__init__` vs `__new__` distinction (briefly, so it stops being scary)
- A direct map of **PHP class concepts onto Python**

## PHP/Laravel bridge

If you have written Laravel models, controllers, or service classes, you already understand objects: bundles of state (properties) and behavior (methods). The big differences in Python are surface-level but constant: Python has **no `public`/`protected`/`private` keywords** (it uses naming conventions enforced socially, not by the compiler), the constructor is always named `__init__` instead of `__construct`, and the current instance — PHP's implicit `$this` — is an **explicit first parameter** you name `self` on every method. There is no `->` operator; Python uses the dot `.` for both method calls and attribute access, and no `$` sigil on variables. Once you internalize "self is just $this written out loud," 80% of the friction disappears.

---

## Core concept: Defining a class

The minimal class in Python:

```python
class Account:
    pass  # 'pass' is a no-op placeholder; a class body cannot be empty

acc = Account()        # instantiate — note: NO 'new' keyword in Python
print(type(acc))       # => <class '__main__.Account'>
print(isinstance(acc, Account))  # => True
```

Two things jump out at a PHP developer:

1. **No `new` keyword.** You call the class like a function: `Account()`. The class *is* the constructor.
2. **`pass`** is required because Python uses indentation for blocks; an empty block is a syntax error. PHP would use `{}`.

By convention, class names use `PascalCase` (just like PHP), while functions/variables use `snake_case`.

### The PHP/Python class skeleton side by side

```php
<?php
// PHP
class Account {
    public string $owner;
    private float $balance;

    public function __construct(string $owner, float $balance = 0.0) {
        $this->owner = $owner;
        $this->balance = $balance;
    }

    public function deposit(float $amount): void {
        $this->balance += $amount;
    }
}

$acc = new Account("Ada", 100.0);
$acc->deposit(50);
```

```python
# Python — the same idea
class Account:
    def __init__(self, owner: str, balance: float = 0.0) -> None:
        self.owner = owner          # instance attribute
        self._balance = balance     # leading underscore = "protected by convention"

    def deposit(self, amount: float) -> None:
        self._balance += amount

acc = Account("Ada", 100.0)   # no 'new'
acc.deposit(50)               # dot, not arrow
print(acc.owner)              # => Ada
```

Notice you **do not declare attributes at the top of the class** the way PHP 7.4+ typed properties encourage. In Python, instance attributes spring into existence the moment you assign to `self.something` (usually inside `__init__`). The type hints (`str`, `float`) are optional documentation checked by tools like `mypy`, *not* enforced at runtime.

---

## Core concept: `__init__`, `self`, and creating objects

### `__init__` is the constructor (not really "construction")

```python
class Point:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

p = Point(3, 4)
print(p.x, p.y)  # => 3 4
```

`__init__` (two leading + two trailing underscores — pronounced "dunder init", short for *double underscore*) runs automatically right after the object is created, to **initialize** it. Strictly speaking the object already exists by the time `__init__` runs; `__init__` does not *create* anything, it just sets it up (more on `__new__` later). For day-to-day work, treat it exactly like PHP's `__construct`.

### `self` is `$this`, written explicitly

In PHP, `$this` is magically available inside methods. In Python, **the instance is passed as the first argument to every instance method**, and by universal convention it is named `self`:

```python
class Greeter:
    def __init__(self, name: str) -> None:
        self.name = name

    def greet(self) -> str:
        return f"Hello, {self.name}!"   # self.name, like $this->name

g = Greeter("Linus")
print(g.greet())          # => Hello, Linus!

# What actually happens under the hood — these two are equivalent:
print(Greeter.greet(g))   # => Hello, Linus!  (self = g, passed explicitly)
```

That last line is the key insight: `g.greet()` is **syntactic sugar** for `Greeter.greet(g)`. Python "binds" `g` to the `self` parameter for you. This is why every instance method must list `self` first — forgetting it is the single most common beginner error:

```python
class Broken:
    def hello():            # BUG: no self
        return "hi"

Broken().hello()
# => TypeError: Broken.hello() takes 0 positional arguments but 1 was given
#    (Python passed the instance, but the method didn't accept it)
```

`self` is just a convention — you *could* name it `this` or `me` — but **never do**. Every Python developer and every linter expects `self`.

---

## Core concept: Instance attributes vs class attributes

This distinction trips up newcomers and is a favorite interview topic.

- **Instance attribute**: belongs to one specific object, set via `self.x = ...`. Like a normal PHP property.
- **Class attribute**: belongs to the class itself and is **shared by all instances**. Defined directly in the class body. Closest PHP analog is a `public static` property, but with a subtle twist.

```python
class Dog:
    species = "Canis familiaris"   # class attribute — shared by ALL dogs

    def __init__(self, name: str) -> None:
        self.name = name           # instance attribute — unique per dog

rex = Dog("Rex")
fido = Dog("Fido")

print(rex.species)    # => Canis familiaris
print(fido.species)   # => Canis familiaris  (same shared value)
print(rex.name, fido.name)  # => Rex Fido     (different per instance)
```

### How attribute lookup works (the why)

When you read `rex.species`, Python looks on the **instance first**, then falls back to the **class**. When you *write* `rex.species = "X"`, Python creates a **new instance attribute** that *shadows* the class one — it does **not** modify the class attribute:

```python
rex.species = "Good Boy"      # creates an instance attribute on rex only
print(rex.species)            # => Good Boy   (instance shadows class)
print(fido.species)           # => Canis familiaris  (unchanged!)
print(Dog.species)            # => Canis familiaris  (class value intact)

del rex.species               # remove the shadow
print(rex.species)            # => Canis familiaris  (falls back to class again)
```

To change the value *for everyone*, assign through the class: `Dog.species = "..."`.

### The shared-mutable-class-attribute trap

This is the bug that bites every Python beginner exactly once. If a class attribute is a **mutable** object (list, dict, set), every instance shares the *same object*, and mutating it leaks across all instances:

```python
class TodoList:
    items = []   # ⚠️ DANGER: one shared list for ALL instances

    def add(self, task: str) -> None:
        self.items.append(task)   # mutates the SHARED list (no rebinding happens)

a = TodoList()
b = TodoList()
a.add("Buy milk")
print(b.items)   # => ['Buy milk']   😱  b never added anything!
print(a.items is b.items)  # => True  (literally the same list object)
```

Why? `self.items.append(...)` does **not** assign to `self.items`; it mutates the object found via class lookup. Since both instances find the *same* class-level list, both see the change.

**The fix:** initialize mutable per-instance state inside `__init__`:

```python
class TodoList:
    def __init__(self) -> None:
        self.items = []   # ✅ a fresh list per instance

    def add(self, task: str) -> None:
        self.items.append(task)

a = TodoList()
b = TodoList()
a.add("Buy milk")
print(a.items)  # => ['Buy milk']
print(b.items)  # => []   correct — independent lists
```

**Rule of thumb:** class attributes are fine for *immutable* shared constants (strings, numbers, tuples) and config defaults. Anything mutable that should be per-object goes in `__init__`.

| | PHP | Python |
|---|---|---|
| Instance property | `public $name;` then `$this->name` | `self.name = ...` in `__init__` |
| Class/static property | `public static $count = 0;` (`self::$count`) | `count = 0` in class body (`ClassName.count`) |
| Access on instance | `$obj->name` | `obj.name` |
| Access static | `ClassName::$count` | `ClassName.count` or `obj.count` (read-only fallback) |

---

## Core concept: Instance methods

A method is just a function defined inside a class whose first parameter is `self`:

```python
class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def scale(self, factor: float) -> None:
        self.width *= factor
        self.height *= factor   # mutates this instance

r = Rectangle(3, 4)
print(r.area())   # => 12
r.scale(2)
print(r.area())   # => 48
```

Methods can call other methods on the same object via `self.other_method()`:

```python
class Order:
    def __init__(self, subtotal: float) -> None:
        self.subtotal = subtotal

    def tax(self) -> float:
        return self.subtotal * 0.08

    def total(self) -> float:
        return self.subtotal + self.tax()   # self.tax(), like $this->tax()

print(Order(100).total())  # => 108.0
```

---

## Core concept: Encapsulation — Python has no `private`

This is one of the largest cultural gaps for a PHP developer. **Python has no `public`, `protected`, or `private` keywords.** Everything is technically accessible from everywhere. Instead Python uses *naming conventions* and a touch of compiler magic.

### 1. No underscore = public

Plain names are the public API. Use them freely.

```python
class User:
    def __init__(self, email: str) -> None:
        self.email = email   # public

u = User("a@b.com")
print(u.email)   # => a@b.com   — totally fine
```

### 2. Single leading underscore `_x` = "protected" (by convention)

A single underscore is a *signal* to other developers: "this is internal, do not touch from outside." Python does **not** enforce it — it is the equivalent of a polite "Staff Only" sign. Tools, linters, and `from module import *` respect it, but the language will not stop you.

```python
class BankAccount:
    def __init__(self, balance: float) -> None:
        self._balance = balance   # "please treat as internal"

acc = BankAccount(100)
print(acc._balance)   # => 100   — works, but you're breaking the social contract
```

Think of `_x` as PHP's `protected`: a hint about intended scope, intended for the class and its subclasses.

### 3. Double leading underscore `__x` = name mangling (closest to "private")

Two leading underscores (and *at most one* trailing underscore) trigger **name mangling**: Python rewrites `self.__x` to `self._ClassName__x` behind the scenes. This makes accidental access and subclass-name-collisions harder — but it is *still not truly private*.

```python
class Vault:
    def __init__(self) -> None:
        self.__secret = 42        # becomes self._Vault__secret

    def reveal(self) -> int:
        return self.__secret      # works inside the class

v = Vault()
print(v.reveal())        # => 42

# print(v.__secret)      # AttributeError: 'Vault' object has no attribute '__secret'
print(v._Vault__secret)  # => 42   — mangled name still reachable if you insist
```

The real purpose of `__x` is **not** security; it is to avoid *accidental name clashes* in inheritance. If a base class uses `__cache` and a subclass also uses `__cache`, mangling keeps them separate (`_Base__cache` vs `_Sub__cache`).

| Naming | PHP analog | Enforced? | When to use |
|---|---|---|---|
| `name` | `public` | n/a | The public interface |
| `_name` | `protected` (loosely) | No (convention only) | Internal detail; OK for subclasses |
| `__name` | closest to `private` | Soft (name mangling) | Avoid clashes; truly internal |
| `__name__` | magic methods | n/a | Reserved dunders — never invent your own |

> Important: a **double underscore on both ends** (`__init__`, `__str__`) is a *dunder* — a reserved "magic method," **not** mangled and **not** private. Do not create your own dunders.

---

## Core concept: `@property` — computed & validated attributes

In PHP you might write explicit `getBalance()` / `setBalance()` methods, or use magic `__get`/`__set`. Python's idiomatic answer is the `@property` decorator: it lets a **method look like a plain attribute** to callers, so you can start with a simple public attribute and later add computation or validation **without changing the calling code**. This is Python's version of the "uniform access principle."

### Computed (read-only) property

```python
class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius

    @property
    def area(self) -> float:        # accessed as c.area, NOT c.area()
        return 3.14159 * self.radius ** 2

c = Circle(2)
print(c.area)   # => 12.56636   — no parentheses! looks like an attribute
# c.area = 10   # AttributeError: property 'area' of 'Circle' object has no setter (read-only)
```

### Property with validation (getter + setter)

```python
class Temperature:
    def __init__(self, celsius: float) -> None:
        self.celsius = celsius      # goes through the setter below (validates!)

    @property
    def celsius(self) -> float:     # the getter
        return self._celsius

    @celsius.setter
    def celsius(self, value: float) -> None:   # the setter
        if value < -273.15:
            raise ValueError("Below absolute zero is impossible")
        self._celsius = value

    @property
    def fahrenheit(self) -> float:  # a computed, derived property
        return self._celsius * 9 / 5 + 32

    @fahrenheit.setter
    def fahrenheit(self, value: float) -> None:
        self.celsius = (value - 32) * 5 / 9   # reuse celsius setter for validation

t = Temperature(25)
print(t.celsius)      # => 25
print(t.fahrenheit)   # => 77.0
t.fahrenheit = 212    # uses the fahrenheit setter
print(t.celsius)      # => 100.0

# t.celsius = -300    # ValueError: Below absolute zero is impossible
```

Key points:

- The getter and setter **share the same name** (`celsius`). You first define the `@property` (getter), then `@celsius.setter` reusing that name.
- The real data is stored in a *different* attribute, conventionally `_celsius`, so you don't recurse forever. (If the getter returned `self.celsius`, it would call itself endlessly.)
- Callers write `t.celsius = 25` and `t.celsius` — clean attribute syntax — while you get full control behind the scenes.

### Deleter (less common)

```python
class Profile:
    def __init__(self, bio: str) -> None:
        self.bio = bio          # routes through the setter (strips whitespace)

    @property
    def bio(self) -> str:
        return self._bio

    @bio.setter
    def bio(self, value: str) -> None:
        self._bio = value.strip()

    @bio.deleter
    def bio(self) -> None:
        print("Clearing bio")
        self._bio = ""

p = Profile("  hello  ")
print(repr(p.bio))   # => 'hello'   (setter stripped whitespace)
del p.bio            # => Clearing bio
print(repr(p.bio))   # => ''
```

**The big win:** you don't need getters/setters up front (as you might preemptively write in Java/PHP). Start with a public attribute; if you later need validation or computation, convert it to a `@property` and **no caller changes**.

---

## Core concept: `@classmethod` and `@staticmethod`

By default a method receives the *instance* as `self`. The two decorators below change what (if anything) the method receives automatically.

### `@classmethod` — receives the class as `cls`

A class method's first parameter is `cls` (the class itself, not an instance). Its primary use is **alternative constructors** — factory methods that build and return instances in different ways. This mirrors PHP `public static function fromArray(...)` returning `new self(...)`.

```python
from datetime import date

class Person:
    def __init__(self, name: str, birth_year: int) -> None:
        self.name = name
        self.birth_year = birth_year

    @classmethod
    def from_age(cls, name: str, age: int) -> "Person":
        current_year = date.today().year
        return cls(name, current_year - age)   # cls(...) == Person(...), but subclass-aware

    @classmethod
    def from_dict(cls, data: dict) -> "Person":
        return cls(data["name"], data["birth_year"])

p1 = Person("Ada", 1990)
p2 = Person.from_age("Grace", 35)          # alternative constructor
p3 = Person.from_dict({"name": "Linus", "birth_year": 1969})
print(p2.birth_year)   # => (current year - 35), e.g. 1991 in 2026
```

Why `cls(...)` instead of hardcoding `Person(...)`? Because if a subclass calls the factory, `cls` is the subclass, so you get the right type automatically:

```python
class Employee(Person):
    pass

e = Employee.from_age("Margaret", 40)
print(type(e))   # => <class '__main__.Employee'>   — correct subclass!
```

Class methods can also operate on class-level state (e.g., a shared counter or registry).

### `@staticmethod` — receives nothing automatically

A static method takes neither `self` nor `cls`. It is just a plain function that *lives in the class's namespace* for organizational reasons — it relates to the class conceptually but needs no instance or class data. Same idea as a PHP `public static function` that uses no `self::`.

```python
class MathUtils:
    @staticmethod
    def is_even(n: int) -> bool:
        return n % 2 == 0          # no self, no cls

    @staticmethod
    def clamp(value: float, low: float, high: float) -> float:
        return max(low, min(value, high))

print(MathUtils.is_even(4))        # => True
print(MathUtils.clamp(15, 0, 10))  # => 10
```

### Quick decision guide

| Decorator | First param | Can access instance state? | Can access class state? | Typical use |
|---|---|---|---|---|
| (none) — instance method | `self` | Yes | Yes (via `self.__class__`) | Normal behavior on an object |
| `@classmethod` | `cls` | No | Yes | Alternative constructors, class-wide state |
| `@staticmethod` | — | No | No | Utility/helper logically grouped with the class |

```python
class Demo:
    kind = "demo"

    def instance_m(self):  return f"instance, kind={self.kind}"
    @classmethod
    def class_m(cls):      return f"class, kind={cls.kind}"
    @staticmethod
    def static_m():        return "static, knows nothing about the class"

d = Demo()
print(d.instance_m())   # => instance, kind=demo
print(Demo.class_m())   # => class, kind=demo
print(Demo.static_m())  # => static, knows nothing about the class
```

---

## Core concept: `__init__` vs `__new__` (briefly)

Object creation in Python is a **two-step** process:

1. `__new__(cls, ...)` — **allocates and returns** the new (empty) object. This is the *actual* constructor.
2. `__init__(self, ...)` — **initializes** the already-created object. Returns nothing.

You almost never override `__new__`. It matters in a few advanced cases: subclassing immutable types (`int`, `str`, `tuple`), implementing singletons, or caching/interning instances. For 99% of code, just write `__init__`.

```python
class Loud:
    def __new__(cls, *args, **kwargs):
        print("1) __new__ runs first, allocates the object")
        instance = super().__new__(cls)   # create the bare object
        return instance                   # MUST return it, or __init__ won't run

    def __init__(self, label: str) -> None:
        print("2) __init__ runs second, sets it up")
        self.label = label

obj = Loud("x")
# => 1) __new__ runs first, allocates the object
# => 2) __init__ runs second, sets it up
print(obj.label)  # => x
```

Mental model: `__new__` *makes* the object, `__init__` *furnishes* it. PHP has no direct equivalent split — `__construct` does both. Just remember `__init__` for everyday work and that it must **not** `return` a value (returning anything other than `None` raises `TypeError`).

---

## Common mistakes & gotchas

- **Forgetting `self`** in a method signature → `TypeError: ... takes 0 positional arguments but 1 was given`. Every instance method needs `self` first.
- **Calling a `@property` with `()`** → `obj.area()` when it's a property raises `TypeError: 'float' object is not callable`. Properties are accessed like attributes, no parentheses.
- **Mutable class attribute shared across instances** → put lists/dicts/sets in `__init__`, never as bare class attributes, unless you genuinely want shared state.
- **`return` inside `__init__`** with a value → `TypeError`. `__init__` must return `None` (just omit `return`).
- **Infinite recursion in a property** → returning `self.celsius` *inside* the `celsius` getter calls the getter again forever. Store data in `_celsius`.
- **Thinking `__private` is secure** → it's only name mangling (`_Class__private`); a determined caller can still reach it. Use it to avoid clashes, not for security.
- **Using `new`** → there is no `new` keyword. `Account()` instantiates.
- **Mutating vs rebinding a class attr via instance** → `self.x = 5` creates an instance attribute (shadows class); `self.x.append(5)` mutates the shared class object. Subtle but critical.
- **Naming the first param `this`** → it works, but it's non-idiomatic and every linter/teammate will flag it. Always `self` (instances) and `cls` (class methods).
- **Declaring attributes like PHP** → you don't list `public $x;` at the top; attributes appear when assigned in `__init__`. (Dataclasses, a later topic, *do* declare fields — but that's a different mechanism.)

## Best practices

- Initialize **all** instance attributes in `__init__` so the object's shape is obvious in one place (even if some start as `None`).
- Keep the public API attribute-based; reach for `@property` only when you need computation, validation, or to evolve a public attribute without breaking callers.
- Use `_single_underscore` for internal attributes and helper methods; reserve `__double` for genuine name-clash avoidance, which is rare.
- Use `@classmethod` for **alternative constructors** (`from_dict`, `from_json`, `from_age`) and return `cls(...)`, not the hardcoded class name, to stay subclass-friendly.
- Use `@staticmethod` for helpers that logically belong to the class but touch no instance/class state — or consider just a module-level function.
- Prefer immutable class attributes (constants, config defaults) and never bare mutable ones.
- Add type hints; they document intent and unlock `mypy`/IDE checks even though Python doesn't enforce them at runtime.
- Don't write Java/PHP-style boilerplate getters/setters for every field. Start with public attributes; upgrade to `@property` *only when needed*.

## Interview Q&A

**Q: What is `self` in Python and how does it differ from `$this` in PHP?**
**A:** `self` is the reference to the current instance, equivalent to PHP's `$this`. The difference is that Python passes it **explicitly** as the first parameter of every instance method, whereas PHP makes `$this` implicitly available. `obj.method()` is really `Class.method(obj)` — Python binds the instance to `self` for you. The name `self` is convention, not a keyword.

**Q: Does Python have private and protected members?**
**A:** Not as enforced language features. Python uses conventions: a single leading underscore (`_x`) signals "internal/protected" but is not enforced; a double leading underscore (`__x`) triggers **name mangling** to `_ClassName__x`, which discourages accidental access and prevents inheritance name clashes, but is still reachable. There is no truly private member — Python trusts the developer ("we're all consenting adults").

**Q: What's the difference between a class attribute and an instance attribute?**
**A:** A class attribute is defined in the class body and **shared by all instances** (one copy on the class). An instance attribute is set via `self.x = ...` and is **unique per object**. On read, Python checks the instance first, then the class. Writing `obj.x = ...` creates an instance attribute that shadows the class attribute rather than modifying it.

**Q: Explain the shared mutable class attribute pitfall.**
**A:** If a class attribute is a mutable object (e.g., a list), all instances share the *same* object. Calling `self.items.append(...)` mutates that shared object, so changes appear across every instance. The fix is to create the mutable object per instance inside `__init__` (`self.items = []`). The pitfall happens because `append` mutates rather than rebinds, so no instance-level attribute is created.

**Q: When would you use `@classmethod` vs `@staticmethod`?**
**A:** Use `@classmethod` (first arg `cls`) when the method needs the class — most commonly for **alternative constructors** (`from_dict`, `from_age`) that return `cls(...)`, which stays correct for subclasses, or to manage class-level state. Use `@staticmethod` (no `self`/`cls`) for utility functions that are conceptually related to the class but don't use any instance or class data.

**Q: What does `@property` give you, and why prefer it over getter/setter methods?**
**A:** `@property` lets a method be accessed like a plain attribute, enabling computed values and validation while keeping clean `obj.x` syntax. The advantage is the *uniform access principle*: you can start with a public attribute and later add logic by converting it to a property **without changing any calling code**. You add validation via `@x.setter` and computed reads via the getter.

**Q: How do you create a read-only attribute in Python?**
**A:** Define a `@property` getter with no corresponding setter. Reading works (`obj.area`); assigning (`obj.area = 5`) raises `AttributeError: property has no setter`. The backing data, if any, is stored in a private-by-convention attribute like `self._radius`.

**Q: Why store property data in `self._celsius` instead of `self.celsius`?**
**A:** Because `celsius` is the property name. If the getter returned `self.celsius`, it would invoke the getter again, causing infinite recursion. The actual value lives in a separate backing attribute (`_celsius`) so the property can read/write it without calling itself.

**Q: What's the difference between `__init__` and `__new__`?**
**A:** `__new__(cls, ...)` is the true constructor: it allocates and returns the new object. `__init__(self, ...)` runs afterward to initialize that object and returns `None`. You normally only write `__init__`. `__new__` is needed for advanced cases like subclassing immutables (`int`, `str`, `tuple`), singletons, or instance caching.

**Q: Why do alternative constructors use `cls(...)` instead of the class name directly?**
**A:** Using `cls(...)` makes the factory **polymorphic**: when a subclass calls it, `cls` is the subclass, so the right type is returned. Hardcoding `Person(...)` would always produce a `Person` even when called as `Employee.from_age(...)`, breaking inheritance.

**Q: Can you reassign a class attribute through an instance?**
**A:** Assigning `obj.attr = value` does **not** change the class attribute — it creates an instance attribute that shadows it. To change the value for all instances you must assign through the class: `ClassName.attr = value`. Reads on other instances still see the class value until they too shadow it.

**Q: Is the type hint `name: str` enforced at runtime?**
**A:** No. Python type hints are optional annotations used by tools (`mypy`, IDEs) for static checking; they do not cause runtime errors if violated. They are documentation plus tooling support, not runtime validation (unlike Pydantic, which actively validates).

## Practice exercises

1. **Bank account.** Write a `BankAccount` class with `owner` and a backing `_balance`. Add `deposit(amount)` and `withdraw(amount)` instance methods. `withdraw` must raise `ValueError` if the amount exceeds the balance. Expose `balance` as a **read-only** `@property`.

2. **Counter with class state.** Write a `Widget` class that tracks how many widgets have been created using a **class attribute** counter. Each `Widget` gets an auto-incrementing `id`. Add a `@classmethod` `count()` that returns the total created.

3. **Validated property.** Write a `Product` class where `price` is a `@property` that rejects negative values (raise `ValueError`) and rounds to 2 decimals on set. Add a computed read-only property `price_with_tax` (8% tax).

4. **Alternative constructor.** Write a `Date` class storing `year`, `month`, `day`. Add a `@classmethod` `from_string(cls, s)` that parses `"2026-06-19"` into a `Date`. Add a `@staticmethod` `is_leap_year(year)` returning a bool.

5. **Spot the bug.** Fix this class so each `Team` has its own player list:
   ```python
   class Team:
       players = []
       def add(self, name):
           self.players.append(name)
   ```

<details><summary>Solutions</summary>

```python
# 1. Bank account
class BankAccount:
    def __init__(self, owner: str, balance: float = 0.0) -> None:
        self.owner = owner
        self._balance = balance

    @property
    def balance(self) -> float:        # read-only
        return self._balance

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self._balance += amount

    def withdraw(self, amount: float) -> None:
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount

acc = BankAccount("Ada", 100)
acc.deposit(50)
acc.withdraw(30)
print(acc.balance)   # => 120
# acc.balance = 999  # AttributeError: property has no setter


# 2. Counter with class state
class Widget:
    _created = 0                       # class attribute (shared)

    def __init__(self) -> None:
        Widget._created += 1           # increment via the CLASS, not self
        self.id = Widget._created

    @classmethod
    def count(cls) -> int:
        return cls._created

a, b, c = Widget(), Widget(), Widget()
print(a.id, b.id, c.id)   # => 1 2 3
print(Widget.count())     # => 3


# 3. Validated property
class Product:
    def __init__(self, name: str, price: float) -> None:
        self.name = name
        self.price = price            # routed through the setter (validates)

    @property
    def price(self) -> float:
        return self._price

    @price.setter
    def price(self, value: float) -> None:
        if value < 0:
            raise ValueError("Price cannot be negative")
        self._price = round(value, 2)

    @property
    def price_with_tax(self) -> float:
        return round(self._price * 1.08, 2)

p = Product("Book", 19.999)
print(p.price)           # => 20.0
print(p.price_with_tax)  # => 21.6
# p.price = -5           # ValueError: Price cannot be negative


# 4. Alternative constructor + static method
class Date:
    def __init__(self, year: int, month: int, day: int) -> None:
        self.year = year
        self.month = month
        self.day = day

    @classmethod
    def from_string(cls, s: str) -> "Date":
        year, month, day = (int(part) for part in s.split("-"))
        return cls(year, month, day)   # cls(...) -> subclass-friendly

    @staticmethod
    def is_leap_year(year: int) -> bool:
        return year % 4 == 0 and (year % 100 != 0 or year % 400 == 0)

d = Date.from_string("2026-06-19")
print(d.year, d.month, d.day)   # => 2026 6 19
print(Date.is_leap_year(2024))  # => True
print(Date.is_leap_year(2100))  # => False


# 5. Spot the bug — move the mutable into __init__
class Team:
    def __init__(self) -> None:
        self.players = []             # ✅ fresh list per Team

    def add(self, name: str) -> None:
        self.players.append(name)

t1, t2 = Team(), Team()
t1.add("Ada")
print(t1.players)   # => ['Ada']
print(t2.players)   # => []   (independent)
```

</details>

## Cheat sheet

```python
class Thing:
    kind = "gadget"                    # class attribute (shared)

    def __init__(self, name):          # constructor (like __construct)
        self.name = name               # instance attribute (per object)
        self._internal = None          # _ = "protected" (convention)
        self.__mangled = 1             # __ = name-mangled to _Thing__mangled

    def speak(self):                   # instance method (self = $this)
        return f"{self.name} the {self.kind}"

    @property                          # computed/validated attribute
    def label(self):                   # read as t.label (NO parentheses)
        return self.name.upper()

    @label.setter
    def label(self, value):            # write as t.label = "x"
        self.name = value.lower()

    @classmethod                       # alternative constructor; cls = the class
    def create(cls, name):
        return cls(name)               # cls(...) is subclass-safe

    @staticmethod                      # no self/cls; utility grouped on the class
    def help():
        return "I am a Thing"

t = Thing("Rex")                       # NO 'new' keyword
print(t.speak())                       # Rex the gadget
print(t.label)                         # REX  (property, no ())
t.label = "Fido"                       # setter
Thing.create("X")                      # classmethod
Thing.help()                           # staticmethod
```

| Concept | PHP | Python |
|---|---|---|
| Constructor | `__construct` | `__init__` |
| Current instance | `$this` (implicit) | `self` (explicit 1st param) |
| Instantiate | `new Foo()` | `Foo()` |
| Member access | `$obj->x` | `obj.x` |
| Static property | `static $x` / `self::$x` | class-body attr / `Foo.x` |
| Static method | `public static function` | `@staticmethod` |
| Factory method | `static function fromX(): self` | `@classmethod` returning `cls(...)` |
| public / protected / private | keywords (enforced) | `x` / `_x` / `__x` (convention + mangling) |
| Computed getter/setter | `__get` / `__set` or `getX()`/`setX()` | `@property` + `@x.setter` |
| True constructor | (combined in `__construct`) | `__new__` (allocate) then `__init__` (init) |

**Memory hooks:** *self is $this spelled out · no `new` · attributes are born in `__init__` · `_x` is a polite sign, `__x` is mangled not private · property = method that looks like an attribute · classmethod for factories (`cls`), staticmethod for helpers · mutable class attributes are a shared-state trap.*
