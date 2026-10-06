# 12 — OOP: Inheritance & Polymorphism

> Week 2 · Python Advanced

You already write classes every day in Laravel — Eloquent models, controllers, form requests, service classes. Python's object model is similar in spirit but more *explicit* and, in one big way, more *flexible*: Python supports true multiple inheritance, leans heavily on **duck typing**, and gives you mixins and abstract base classes as first-class tools. This lesson connects all of that to what you already know.

## What you will learn

- Single inheritance and how to override methods correctly
- `super()` and **cooperative** initialization (and why you almost never call the parent class by name)
- Multiple inheritance, the **Method Resolution Order (MRO)**, and C3 linearization (`__mro__`)
- **Mixins** as a design tool — Python's answer to Laravel traits
- **Polymorphism and duck typing** — "if it walks like a duck..."
- **Abstract base classes** with `abc.ABC` and `@abstractmethod` — Python's answer to interfaces
- `isinstance()` / `issubclass()` and when (not) to use them
- **Composition over inheritance** — the principle that saves codebases
- A preview of `@dataclass` for data-holding classes
- A clear map between Python and Laravel/PHP traits, interfaces, and abstract classes

## PHP/Laravel bridge

In PHP you have `class Admin extends User`, `interface`, `abstract class`, and `trait`. Python collapses some of these into fewer keywords but adds power. PHP **traits** (`use SoftDeletes;`) map almost exactly onto Python **mixins** — small reusable classes you "mix in." PHP **interfaces** and **abstract classes** map onto Python's **abstract base classes** (`abc.ABC` + `@abstractmethod`). The biggest mental shift: PHP only allows single class inheritance (traits aside), while Python allows true **multiple inheritance**, and PHP's strict type checks are replaced by Python's **duck typing** — Python rarely asks "what type are you?" and instead asks "can you do what I need?"

| PHP / Laravel | Python | Notes |
|---|---|---|
| `class B extends A` | `class B(A):` | Single inheritance, same idea |
| `parent::__construct()` | `super().__init__()` | But Python's `super()` is cooperative (MRO-aware) |
| `interface Foo` | `class Foo(ABC)` with `@abstractmethod` | Python has no `interface` keyword |
| `abstract class Foo` | `class Foo(ABC)` | Same role |
| `trait Bar { ... }` + `use Bar;` | mixin: `class Bar:` then `class X(Bar, Base):` | Traits ≈ mixins |
| `instanceof` | `isinstance()` | |
| `is_subclass_of()` | `issubclass()` | |
| (no multiple inheritance) | `class X(A, B, C):` | Python allows it; MRO resolves conflicts |

---

## Core concept: Single inheritance & method overriding

A subclass inherits attributes and methods from its parent and can **override** them by redefining a method with the same name. No `override` keyword is needed (unlike PHP 8.3+'s optional `#[\Override]`).

```python
class Animal:
    def __init__(self, name: str) -> None:
        self.name = name

    def speak(self) -> str:
        return f"{self.name} makes a sound"

    def describe(self) -> str:
        # Calls speak() on self — so subclasses' overrides are used. This is polymorphism.
        return f"{self.name}: {self.speak()}"


class Dog(Animal):
    def speak(self) -> str:          # overrides Animal.speak
        return f"{self.name} says woof"


class Cat(Animal):
    def speak(self) -> str:
        return f"{self.name} says meow"


print(Dog("Rex").describe())   # => Rex: Rex says woof
print(Cat("Lily").describe())  # => Lily: Lily says meow
```

Notice `describe()` lives only on `Animal`, yet it prints the *subclass's* `speak()`. That is the heart of polymorphism: the method that runs is decided by the actual object at runtime, not by the class where the calling code was written.

### Calling the parent implementation with `super()`

When you override but still want the parent's behavior, call `super()`:

```python
class LoudDog(Dog):
    def speak(self) -> str:
        base = super().speak()     # "Rex says woof"
        return base.upper() + "!!!"


print(LoudDog("Rex").speak())  # => REX SAYS WOOF!!!
```

`super().speak()` is the Python equivalent of PHP's `parent::speak()` — but as you'll see below, `super()` is smarter than `parent::` because it follows the MRO.

---

## Core concept: `super()` and cooperative initialization

The single most common beginner bug is forgetting to initialize the parent in `__init__`. If you define `__init__` in a subclass, you **replace** the parent's `__init__` — Python does not auto-chain constructors (unlike some languages). You must call `super().__init__(...)` yourself.

```python
class User:
    def __init__(self, name: str, email: str) -> None:
        self.name = name
        self.email = email


class Admin(User):
    def __init__(self, name: str, email: str, level: int) -> None:
        super().__init__(name, email)   # initialize the User part FIRST
        self.level = level              # then add Admin-specific state


a = Admin("Sara", "sara@example.com", level=5)
print(a.name, a.email, a.level)  # => Sara sara@example.com 5
```

If you had forgotten `super().__init__(...)`, then `a.name` would raise `AttributeError` because the parent never ran.

### Why `super()` instead of `User.__init__(self, ...)`?

You *could* write `User.__init__(self, name, email)`, and it works for simple single inheritance. But hard-coding the parent's name breaks **cooperative multiple inheritance**: if a class is later mixed into a different hierarchy, `super()` walks the *real* MRO, while a hard-coded name skips classes. Always prefer `super()`.

```python
# Cooperative __init__ across a diamond: every class calls super().__init__()
class Base:
    def __init__(self, **kwargs) -> None:
        # super() here points to `object`; absorb leftover kwargs.
        super().__init__()
        print("Base init")

class A(Base):
    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        print("A init")

class B(Base):
    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        print("B init")

class C(A, B):
    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        print("C init")

C()
# => Base init
# => B init
# => A init
# => C init
```

Each class calls `super().__init__()` exactly once, and Python's MRO guarantees every class in the diamond runs **exactly once** — that's what "cooperative" means. Passing `**kwargs` along is the idiom that lets each class pull out the args it cares about and forward the rest.

---

## Core concept: Multiple inheritance and the MRO

Python lets a class inherit from several parents at once. When two parents define the same method, which one wins? Python answers this deterministically with the **Method Resolution Order**, computed by the **C3 linearization** algorithm. You can inspect it via `__mro__` or `ClassName.mro()`.

```python
class A:
    def who(self) -> str:
        return "A"

class B(A):
    def who(self) -> str:
        return "B"

class C(A):
    def who(self) -> str:
        return "C"

class D(B, C):
    pass

print(D().who())  # => B    (B comes before C in the MRO)

for cls in D.__mro__:
    print(cls.__name__)
# => D
# => B
# => C
# => A
# => object
```

### What C3 linearization guarantees

C3 produces an ordering that respects two rules:

1. **Children before parents** — a subclass always appears before its bases.
2. **Order of declaration** — bases listed left-to-right keep their relative order.

If those rules can't be satisfied (a genuinely inconsistent hierarchy), Python refuses to create the class:

```python
class X: pass
class Y(X): pass

# class Bad(X, Y): pass
# => TypeError: Cannot create a consistent method resolution order (MRO)
#    for bases X, Y
# X cannot come before Y because Y is a subclass of X — contradiction.
```

The practical takeaway: you rarely need to *compute* the MRO by hand, but you must know it exists, that it is deterministic, and that `super()` follows it. When debugging "why did the wrong method run?", print `Cls.__mro__`.

---

## Core concept: Mixins (Python's traits)

A **mixin** is a small class that provides a focused piece of behavior, designed to be combined with other classes — never instantiated on its own. This is exactly how Laravel traits like `SoftDeletes` or `Notifiable` work.

```python
import json
from datetime import datetime, timezone


class JsonSerializableMixin:
    """Adds to_json() to any class that has a __dict__."""
    def to_json(self) -> str:
        return json.dumps(self.__dict__, default=str)


class TimestampMixin:
    """Adds created_at / touch() — like Laravel's timestamps."""
    def __init__(self, *args, **kwargs) -> None:
        super().__init__(*args, **kwargs)   # cooperate with the rest of the MRO
        self.created_at = datetime.now(timezone.utc)

    def touch(self) -> None:
        self.updated_at = datetime.now(timezone.utc)


class Product(TimestampMixin, JsonSerializableMixin):
    def __init__(self, name: str, price: float) -> None:
        super().__init__()          # runs TimestampMixin.__init__ via MRO
        self.name = name
        self.price = price


p = Product("Keyboard", 49.99)
p.touch()
print(p.to_json())
# => {"created_at": "2026-06-19 ...", "name": "Keyboard", "price": 49.99, "updated_at": "2026-06-19 ..."}
```

### Mixin conventions (matching the Laravel mental model)

| Laravel trait | Python mixin |
|---|---|
| `use SoftDeletes;` | `class SoftDeleteMixin:` then list it in bases |
| Trait methods become class methods | Mixin methods become class methods |
| Conflict resolution: `insteadof` / `as` | Conflict resolution: MRO order (left wins) |
| Cannot have constructor state easily | Mixin `__init__` works via cooperative `super()` |

**Naming convention:** suffix mixins with `Mixin` and list them **before** the main base class, because the leftmost class wins method conflicts and you usually want the mixin's version to take precedence.

---

## Core concept: Polymorphism & duck typing

Polymorphism means "many shapes": code calls a method and the right implementation runs based on the object. Python takes this further with **duck typing** — *"If it walks like a duck and quacks like a duck, it's a duck."* You don't check the type; you just call the method and trust the object provides it.

```python
class Duck:
    def quack(self) -> str:
        return "Quack!"

class Person:
    def quack(self) -> str:
        return "I'm imitating a duck!"

def make_it_quack(thing) -> str:
    # No isinstance check, no shared base class required.
    # We only care that `thing` has a quack() method.
    return thing.quack()

print(make_it_quack(Duck()))    # => Quack!
print(make_it_quack(Person()))  # => I'm imitating a duck!
```

`Duck` and `Person` share **no** parent, yet both work. This is profoundly different from PHP, where you would typically require a shared interface (`Quackable`) and a type hint. In Python the "interface" is implicit: it's just the set of methods you happen to call.

### Duck typing in the standard library

You've already used it. `len()` works on anything with `__len__`. A `for` loop works on anything with `__iter__`. `with` works on anything with `__enter__`/`__exit__`. These are **protocols** — informal interfaces defined by behavior, not by inheritance.

```python
class Playlist:
    def __init__(self, songs: list[str]) -> None:
        self._songs = songs

    def __len__(self) -> int:        # makes len() work
        return len(self._songs)

    def __iter__(self):              # makes `for` work
        return iter(self._songs)


pl = Playlist(["a", "b", "c"])
print(len(pl))            # => 3
print(list(pl))          # => ['a', 'b', 'c']
print("b" in (s for s in pl))  # => True
```

> Modern Python can make duck typing *static-type-checkable* using `typing.Protocol`, but that's a later lesson. The runtime behavior above needs no annotations.

---

## Core concept: Abstract base classes (Python's interfaces)

Sometimes you *do* want to enforce a contract: "every subclass MUST implement `area()`." That's what abstract base classes are for. Use `abc.ABC` as a base and decorate required methods with `@abstractmethod`. Python then refuses to instantiate any subclass that hasn't implemented all of them.

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        ...                     # body is irrelevant; subclasses must override

    @abstractmethod
    def perimeter(self) -> float:
        ...

    def describe(self) -> str:  # concrete methods are allowed on an ABC
        return f"area={self.area():.2f}, perimeter={self.perimeter():.2f}"


class Rectangle(Shape):
    def __init__(self, w: float, h: float) -> None:
        self.w, self.h = w, h

    def area(self) -> float:
        return self.w * self.h

    def perimeter(self) -> float:
        return 2 * (self.w + self.h)


r = Rectangle(3, 4)
print(r.describe())   # => area=12.00, perimeter=14.00

# Shape()  -> TypeError: Can't instantiate abstract class Shape
#             without an implementation for abstract methods 'area', 'perimeter'
```

If `Rectangle` had implemented only `area()` and not `perimeter()`, then `Rectangle(3, 4)` would raise the same `TypeError`. This is the closest Python comes to PHP's `interface` enforcement — but it's checked at **instantiation time**, not parse time.

### ABC vs interface vs abstract class — the mapping

- A **pure interface** (PHP `interface`) ≈ an `ABC` where *every* method is `@abstractmethod`.
- An **abstract class** (PHP `abstract class`) ≈ an `ABC` that mixes abstract and concrete methods (like `Shape.describe()` above).
- Python doesn't separate the two concepts with different keywords — it's all `ABC`. The difference is just how many methods you mark abstract.

### Abstract properties

You can also require properties (computed attributes):

```python
from abc import ABC, abstractmethod


class PaymentMethod(ABC):
    @property
    @abstractmethod
    def provider_name(self) -> str:
        ...

    @abstractmethod
    def charge(self, amount_cents: int) -> bool:
        ...


class StripePayment(PaymentMethod):
    @property
    def provider_name(self) -> str:
        return "Stripe"

    def charge(self, amount_cents: int) -> bool:
        print(f"Charging {amount_cents}c via Stripe")
        return True


sp = StripePayment()
print(sp.provider_name)     # => Stripe
sp.charge(1999)             # => Charging 1999c via Stripe
```

---

## Core concept: `isinstance()` and `issubclass()`

These are Python's `instanceof` and `is_subclass_of()`.

```python
class Animal: ...
class Dog(Animal): ...

d = Dog()

print(isinstance(d, Dog))       # => True
print(isinstance(d, Animal))    # => True   (subclass instance IS-A parent)
print(isinstance(d, object))    # => True   (everything is an object)
print(issubclass(Dog, Animal))  # => True
print(issubclass(Animal, Dog))  # => False

# Check against multiple types with a tuple:
print(isinstance(d, (int, Animal)))  # => True
```

### Use them sparingly

Excessive `isinstance` checks are usually a sign you're fighting duck typing. Prefer calling the method you need and letting polymorphism sort it out. Legitimate uses:

- Validating function inputs at a public boundary (e.g., an API layer).
- Branching on a small, closed set of known types.
- Implementing `__eq__` to bail out on unrelated types:

```python
class Money:
    def __init__(self, cents: int) -> None:
        self.cents = cents

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented   # let Python try the reflected op
        return self.cents == other.cents
```

Avoid `type(x) == Dog`; it breaks for subclasses. `isinstance` respects the hierarchy, which is almost always what you want.

---

## Core concept: Composition over inheritance

Inheritance models **"is-a"** (a `Dog` is an `Animal`). Composition models **"has-a"** (a `Car` has an `Engine`). A widely repeated design principle is *"favor composition over inheritance"* because deep inheritance trees become rigid and tangled, while composition stays flexible — you can swap parts at runtime.

### The inheritance trap

```python
# Tempting but brittle: behavior baked into the type hierarchy.
class Logger:
    def log(self, msg: str) -> None:
        print(f"[LOG] {msg}")

class FileLogger(Logger):
    def log(self, msg: str) -> None:
        print(f"[FILE] {msg}")

class OrderServiceConsole(Logger): ...
class OrderServiceFile(FileLogger): ...
# Now you need a class per logging variant. Combinations explode.
```

### The composition fix

```python
class ConsoleLogger:
    def log(self, msg: str) -> None:
        print(f"[LOG] {msg}")

class FileLogger:
    def __init__(self, path: str) -> None:
        self.path = path
    def log(self, msg: str) -> None:
        print(f"[FILE {self.path}] {msg}")


class OrderService:
    def __init__(self, logger) -> None:   # inject the dependency (duck-typed: any .log())
        self.logger = logger

    def place_order(self, item: str) -> None:
        self.logger.log(f"Order placed: {item}")


# Swap behavior without new subclasses — and trivially testable.
OrderService(ConsoleLogger()).place_order("Mouse")
# => [LOG] Order placed: Mouse
OrderService(FileLogger("/tmp/orders.log")).place_order("Mouse")
# => [FILE /tmp/orders.log] Order placed: Mouse
```

This is exactly **dependency injection** — Laravel's service container does the same thing by constructor-injecting dependencies. The `OrderService` doesn't care *what* logger it gets, only that it has a `.log()` method (duck typing again). For tests, you can pass a fake logger that records messages.

**Rule of thumb:** use inheritance when there's a genuine, stable "is-a" relationship and you want to share/override behavior. Use composition when you want to combine independent, swappable capabilities — which is most of the time in business code.

---

## Core concept: `@dataclass` preview for data-holding classes

Many classes exist only to hold data — Laravel DTOs, value objects, API request shapes. Writing `__init__`, `__repr__`, and `__eq__` by hand is tedious boilerplate. Python's `@dataclass` decorator generates all of that for you.

```python
from dataclasses import dataclass, field


@dataclass
class Point:
    x: float
    y: float

    def distance_to(self, other: "Point") -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5


p1 = Point(0, 0)
p2 = Point(3, 4)
print(p1)                 # => Point(x=0, y=0)        (auto __repr__)
print(p1 == Point(0, 0))  # => True                   (auto __eq__ by fields)
print(p1.distance_to(p2)) # => 5.0
```

Dataclasses participate in inheritance just like normal classes, and they support defaults, frozen (immutable) instances, and mutable-default safety via `field`:

```python
@dataclass(frozen=True)        # immutable: assignment raises FrozenInstanceError
class Config:
    debug: bool = False
    hosts: list[str] = field(default_factory=list)  # NEVER use hosts: list = []


c = Config(debug=True)
print(c)                # => Config(debug=True, hosts=[])
# c.debug = False       # => dataclasses.FrozenInstanceError
```

> Why `field(default_factory=list)` and not `= []`? A bare `[]` default would be **shared across all instances** (the classic mutable-default trap). `default_factory` builds a fresh list per instance. We'll go deeper on dataclasses — and how they relate to Pydantic models — in a later lesson; for now, reach for `@dataclass` whenever a class is mostly fields.

---

## Common mistakes & gotchas

- **Forgetting `super().__init__()`** in a subclass `__init__`, so parent attributes are never set → `AttributeError` later. Defining `__init__` *replaces* the parent's; it does not extend it automatically.
- **Hard-coding the parent name** (`Parent.method(self, ...)`) instead of `super()`. It works for simple cases but silently breaks under multiple inheritance because it ignores the MRO.
- **Mutable default arguments / class-level mutable attributes** shared across instances. Use `field(default_factory=...)` in dataclasses, or build the value inside `__init__`.
- **Mixin without cooperative `super()`** — if a mixin defines `__init__` but doesn't call `super().__init__(*args, **kwargs)`, it can cut off the rest of the chain so other classes never initialize.
- **Listing mixins after the base class** when you wanted the mixin to win method conflicts. Leftmost wins; put mixins first.
- **Over-using `isinstance`/`type()` checks** instead of trusting duck typing. And never use `type(x) == SomeClass` for hierarchy checks — it ignores subclasses; use `isinstance`.
- **Trying to instantiate an ABC** (or a subclass that left a method abstract) → `TypeError: Can't instantiate abstract class`. The error fires at instantiation, not at class definition.
- **Diamond inheritance assumptions** — don't assume left-to-right depth-first. The MRO is C3, and the only reliable answer is `Cls.__mro__`.
- **Deep inheritance hierarchies** (4+ levels) for code reuse. This is usually a smell; reach for composition.

## Best practices

- Always call `super().__init__(...)` first when overriding `__init__`, and pass `**kwargs` through in cooperative hierarchies.
- Prefer `super()` over naming the parent class explicitly — it's MRO-correct and refactor-safe.
- Name mixins with a `Mixin` suffix and keep them small and single-purpose, like Laravel traits.
- Use ABCs to document and enforce a contract when "every implementation must do X." Use plain duck typing (or `typing.Protocol`) when you just need behavior.
- **Favor composition over inheritance** for combining capabilities; reserve inheritance for true "is-a" relationships.
- Inject dependencies through the constructor rather than hard-wiring concrete classes — it's testable and swappable.
- Use `@dataclass` for data-centric classes to eliminate boilerplate and get correct `__eq__`/`__repr__` for free.
- When debugging method resolution, print `Cls.__mro__` instead of guessing.
- Keep inheritance trees shallow (1–2 levels is plenty for most app code).

## Interview Q&A

**Q: What's the difference between `super().__init__()` and calling `Parent.__init__(self)` directly?**
**A:** `super()` follows the class's MRO, so in multiple/diamond inheritance it calls the *next* class in line and guarantees each `__init__` runs once when everyone cooperates. Naming the parent directly hard-codes one specific class and skips the MRO, which can cause some bases to be initialized twice or not at all. For single inheritance they behave the same, but `super()` is the safe, refactor-proof default.

**Q: Explain the MRO and C3 linearization.**
**A:** The Method Resolution Order is the deterministic sequence Python searches when resolving a method or attribute on a class. It's computed by the C3 linearization algorithm, which guarantees that subclasses come before their bases and that the left-to-right order of declared bases is preserved. You can inspect it with `Cls.__mro__` or `Cls.mro()`. If a consistent order can't be built, Python raises a `TypeError` at class-creation time.

**Q: How does Python do polymorphism without interfaces?**
**A:** Through duck typing: code calls the methods it needs and any object that provides them works, regardless of its class or inheritance. Built-ins like `len()`, iteration, and context managers all rely on this — they look for `__len__`, `__iter__`, `__enter__`/`__exit__`. You can still enforce contracts explicitly with abstract base classes or check them statically with `typing.Protocol`, but neither is required at runtime.

**Q: What is a mixin, and how is it different from a normal base class?**
**A:** A mixin is a small class providing a focused, reusable slice of behavior that is meant to be combined with other classes via multiple inheritance, never instantiated alone. Unlike a "real" base class, it doesn't represent an "is-a" relationship — it just adds capabilities, like Laravel traits. Convention: name it `...Mixin`, keep it single-purpose, and list it before the main base so its methods win conflicts.

**Q: How do Python mixins compare to PHP/Laravel traits?**
**A:** They serve the same purpose — sharing reusable methods across unrelated classes. The mechanism differs: PHP traits are copied into the class at compile time and resolve conflicts with `insteadof`/`as`, while Python mixins are real classes in the MRO and resolve conflicts by inheritance order. Python mixins can also carry constructor state via cooperative `super().__init__()`, which traits handle less cleanly.

**Q: When would you use an abstract base class instead of duck typing?**
**A:** Use an ABC when you want to *enforce* a contract — guarantee that every subclass implements certain methods, fail fast if it doesn't, and document the expected interface. Duck typing is better when you just need behavior and don't want to force a shared base. ABCs are checked at instantiation time: attempting to instantiate a class with unimplemented `@abstractmethod`s raises `TypeError`.

**Q: What's the difference between `isinstance(x, C)` and `type(x) == C`?**
**A:** `isinstance` respects the inheritance hierarchy — it's `True` for instances of `C` *and* its subclasses, and it accepts a tuple of types. `type(x) == C` is an exact identity check that returns `False` for subclass instances, which usually breaks polymorphism. Prefer `isinstance` unless you specifically need to reject subclasses.

**Q: Why is "composition over inheritance" recommended?**
**A:** Inheritance tightly couples a subclass to its parent's implementation and forces an "is-a" relationship that may not hold as requirements change; deep trees become rigid and combinations explode. Composition ("has-a") lets you assemble behavior from independent, swappable parts injected at runtime, which is more flexible and far easier to test (you can inject fakes/mocks). Use inheritance only for genuine, stable "is-a" relationships.

**Q: Does Python automatically call the parent constructor?**
**A:** No. If a subclass defines `__init__`, it fully replaces the parent's; you must call `super().__init__(...)` yourself to initialize the inherited state. If you don't define `__init__` at all, the subclass inherits the parent's directly.

**Q: What happens with a diamond inheritance like `class D(B, C)` where both `B` and `C` extend `A`?**
**A:** Python builds a single linear MRO — `D, B, C, A, object` — so `A` appears only once. As long as every class calls `super().__init__()` cooperatively (and forwards `**kwargs`), each `__init__` runs exactly once in MRO order. This avoids the classic "diamond problem" of running the shared base twice.

**Q: What does `@dataclass` generate, and why prefer it for DTOs?**
**A:** It auto-generates `__init__`, `__repr__`, and `__eq__` (and optionally ordering and immutability) from the annotated fields, removing boilerplate and giving correct value-based equality. It's ideal for data-holding classes (DTOs, value objects) where the logic is mostly "carry these fields." Use `field(default_factory=...)` for mutable defaults to avoid sharing one list/dict across instances.

**Q: How do you avoid the mutable default argument trap in classes?**
**A:** Never use a mutable literal as a default (`def __init__(self, items=[])` or a class-level `items = []`), because it's created once and shared across all instances/calls. Instead default to `None` and build inside `__init__`, or in a dataclass use `field(default_factory=list)`.

## Practice exercises

1. **Override + super:** Create `Employee` with `__init__(self, name, salary)` and a method `pay_summary()` returning `"{name} earns {salary}"`. Subclass `Manager` that adds a `bonus`, overrides `pay_summary()` to include the bonus, and reuses the parent's string via `super()`.

2. **Abstract contract:** Define an abstract base class `Notifier` with an abstract method `send(self, message: str) -> bool`. Implement `EmailNotifier` and `SmsNotifier`. Show that instantiating `Notifier()` directly raises `TypeError`.

3. **Mixin:** Write a `ReprMixin` whose `__repr__` returns `ClassName(field=value, ...)` built from `self.__dict__`. Mix it into a plain `User` class and verify `repr(User("Sara", 30))` looks right.

4. **MRO inspection:** Build a diamond (`A`, `B(A)`, `C(A)`, `D(B, C)`), give each a `greet()` returning its own name, and print `D().greet()` plus `[c.__name__ for c in D.__mro__]`. Predict the output before running.

5. **Composition + duck typing:** Build a `ReportGenerator` that takes any `formatter` object with a `format(data: dict) -> str` method. Provide `JsonFormatter` and `CsvFormatter` (no shared base class), and show the same generator works with both.

6. **Dataclass:** Convert a hand-written `Coordinate` class (with manual `__init__`, `__repr__`, `__eq__`) into a `@dataclass`, then make a `frozen=True` variant and show that assigning to a field raises `FrozenInstanceError`.

<details><summary>Solutions</summary>

```python
# 1. Override + super
class Employee:
    def __init__(self, name: str, salary: float) -> None:
        self.name = name
        self.salary = salary

    def pay_summary(self) -> str:
        return f"{self.name} earns {self.salary}"


class Manager(Employee):
    def __init__(self, name: str, salary: float, bonus: float) -> None:
        super().__init__(name, salary)
        self.bonus = bonus

    def pay_summary(self) -> str:
        return f"{super().pay_summary()} plus bonus {self.bonus}"


print(Manager("Sara", 5000, 1000).pay_summary())
# => Sara earns 5000 plus bonus 1000
```

```python
# 2. Abstract contract
from abc import ABC, abstractmethod


class Notifier(ABC):
    @abstractmethod
    def send(self, message: str) -> bool:
        ...


class EmailNotifier(Notifier):
    def send(self, message: str) -> bool:
        print(f"EMAIL: {message}")
        return True


class SmsNotifier(Notifier):
    def send(self, message: str) -> bool:
        print(f"SMS: {message}")
        return True


EmailNotifier().send("hi")   # => EMAIL: hi
SmsNotifier().send("yo")     # => SMS: yo

try:
    Notifier()               # type: ignore[abstract]
except TypeError as e:
    print(type(e).__name__)  # => TypeError
```

```python
# 3. Mixin
class ReprMixin:
    def __repr__(self) -> str:
        fields = ", ".join(f"{k}={v!r}" for k, v in self.__dict__.items())
        return f"{type(self).__name__}({fields})"


class User(ReprMixin):
    def __init__(self, name: str, age: int) -> None:
        self.name = name
        self.age = age


print(repr(User("Sara", 30)))   # => User(name='Sara', age=30)
```

```python
# 4. MRO inspection
class A:
    def greet(self) -> str: return "A"
class B(A):
    def greet(self) -> str: return "B"
class C(A):
    def greet(self) -> str: return "C"
class D(B, C):
    pass

print(D().greet())                       # => B
print([c.__name__ for c in D.__mro__])   # => ['D', 'B', 'C', 'A', 'object']
```

```python
# 5. Composition + duck typing
import json


class JsonFormatter:
    def format(self, data: dict) -> str:
        return json.dumps(data)


class CsvFormatter:
    def format(self, data: dict) -> str:
        header = ",".join(data.keys())
        row = ",".join(str(v) for v in data.values())
        return f"{header}\n{row}"


class ReportGenerator:
    def __init__(self, formatter) -> None:   # duck-typed: anything with .format()
        self.formatter = formatter

    def generate(self, data: dict) -> str:
        return self.formatter.format(data)


data = {"name": "Sara", "score": 99}
print(ReportGenerator(JsonFormatter()).generate(data))
# => {"name": "Sara", "score": 99}
print(ReportGenerator(CsvFormatter()).generate(data))
# => name,score
# => Sara,99
```

```python
# 6. Dataclass
from dataclasses import dataclass, FrozenInstanceError


@dataclass
class Coordinate:
    lat: float
    lon: float


print(Coordinate(1.0, 2.0))                 # => Coordinate(lat=1.0, lon=2.0)
print(Coordinate(1.0, 2.0) == Coordinate(1.0, 2.0))  # => True


@dataclass(frozen=True)
class FrozenCoordinate:
    lat: float
    lon: float


fc = FrozenCoordinate(1.0, 2.0)
try:
    fc.lat = 9.0                            # type: ignore[misc]
except FrozenInstanceError as e:
    print(type(e).__name__)                # => FrozenInstanceError
```

</details>

## Cheat sheet

```python
# --- Inheritance & override ---
class Child(Parent):
    def __init__(self, ...):
        super().__init__(...)      # ALWAYS call parent init first
    def method(self):
        return super().method()    # reuse parent, MRO-aware

# --- Multiple inheritance / MRO ---
class D(B, C): ...                  # leftmost wins on conflicts
D.__mro__                          # tuple: search order, ends in object
D.mro()                            # same as a list

# --- Mixins (≈ Laravel traits) ---
class TimestampMixin:              # suffix 'Mixin', list it FIRST in bases
    def __init__(self, *a, **k):
        super().__init__(*a, **k)  # cooperate!
class Model(TimestampMixin, Base): ...

# --- Abstract base class (≈ interface / abstract class) ---
from abc import ABC, abstractmethod
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...
# Shape()  -> TypeError if abstract methods remain

# --- Type checks ---
isinstance(x, Cls)                 # respects subclasses (use this)
isinstance(x, (A, B))              # any of several
issubclass(Sub, Base)
# avoid: type(x) == Cls            # ignores subclasses

# --- Duck typing ---
def use(thing): return thing.quack()   # no type check; just call it

# --- Composition (favor over inheritance) ---
class Service:
    def __init__(self, dep): self.dep = dep   # inject; swap; test easily

# --- Dataclass for data holders ---
from dataclasses import dataclass, field
@dataclass(frozen=True)            # frozen => immutable
class Cfg:
    debug: bool = False
    hosts: list[str] = field(default_factory=list)  # never = []
```

| Concept | Keyword / API | One-liner |
|---|---|---|
| Call parent | `super().__init__()` | MRO-aware, refactor-safe |
| Resolution order | `Cls.__mro__` | Deterministic (C3) |
| Reusable behavior | mixin class | Python's trait |
| Enforce contract | `ABC` + `@abstractmethod` | Python's interface |
| "Is-a" check | `isinstance()` | Respects subclasses |
| "Can-do" check | duck typing | Just call the method |
| Combine capabilities | composition / DI | Favor over inheritance |
| Data holder | `@dataclass` | Auto init/repr/eq |
