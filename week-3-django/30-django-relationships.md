# 30 — Model Relationships

> Week 3 · Django ORM

## What you will learn

- How to model **many-to-one** relationships with `ForeignKey`, and what every `on_delete` option actually does to your data.
- Controlling the **reverse accessor** with `related_name` and `related_query_name`.
- **One-to-one** relationships (`OneToOneField`) and the classic *profile extension* pattern.
- **Many-to-many** relationships (`ManyToManyField`), the auto-created join table, and how to add/remove/query.
- **Custom `through` models** when the join needs extra fields (e.g. "date enrolled", "role").
- Walking **forward and reverse** across relations from Python.
- Querying across relationships with the **`__` (double-underscore) lookup** syntax.
- **Self-referential** foreign keys (employee → manager, category → parent).
- A mental map from **Eloquent** (`hasOne`/`hasMany`/`belongsTo`/`belongsToMany`/pivot) to Django.

## PHP/Laravel bridge

In Laravel you declare relationships as **methods on the model** (`$user->posts()`), and the foreign-key columns live in migrations written separately. Django flips this: you declare the relationship **as a field on the model class**, and Django generates both the column *and* the migration for you. A Django `ForeignKey` is Eloquent's `belongsTo` + the matching `hasMany` in one line — the "owning" side is where you put the field, and the reverse (`hasMany`) side is auto-generated as a manager. `OneToOneField` ≈ `hasOne`/`belongsTo` with a uniqueness guarantee, `ManyToManyField` ≈ `belongsToMany`, and a Django `through` model is Eloquent's pivot model. The big difference: Django has no "lazy by default and crash on N+1 silently" — you still get N+1, but you fix it explicitly with `select_related`/`prefetch_related` (covered in the queries lesson).

## Core concept: ForeignKey (many-to-one)

A `ForeignKey` says "many of *this* model belong to one of *that* model." It lives on the **many** side. Many books belong to one publisher; many articles belong to one author.

```python
# library/models.py
from django.db import models


class Publisher(models.Model):
    name = models.CharField(max_length=200)
    country = models.CharField(max_length=100)

    def __str__(self):
        return self.name


class Book(models.Model):
    title = models.CharField(max_length=300)
    # The FK lives on the "many" side (many books -> one publisher).
    publisher = models.ForeignKey(
        Publisher,
        on_delete=models.CASCADE,   # required in Django 2.0+
        related_name="books",       # reverse accessor: publisher.books
    )
    published_year = models.PositiveIntegerField()

    def __str__(self):
        return self.title
```

A few beginner questions answered:

- **Why is `on_delete` mandatory?** Because Django refuses to guess what should happen to a `Book` when its `Publisher` is deleted. You must decide. (In MySQL/Laravel this lived in the migration's `->onDelete(...)`; here it lives on the model and Django enforces it at the ORM level, not only in the DB.)
- **What column gets created?** A `publisher_id` integer column on the `book` table. The Python attribute is `publisher` (the object), but the raw column is `publisher_id`. You can set either:

```python
book.publisher = some_publisher   # assign the object
book.publisher_id = 42            # assign the raw id (no extra query)
```

### `on_delete` options — what each one does

When the referenced row (the `Publisher`) is deleted, `on_delete` controls the fate of the rows pointing at it (the `Book`s).

| Option | What happens to the dependent rows | Eloquent / SQL analogue |
|---|---|---|
| `CASCADE` | Deleted too. Delete a publisher → its books vanish. | `->onDelete('cascade')` |
| `PROTECT` | Deletion is **blocked**; raises `ProtectedError`. | `->onDelete('restrict')` |
| `RESTRICT` | Like `PROTECT` but only blocks if no *other* cascade also covers it (rarely needed). | `restrict` |
| `SET_NULL` | FK set to `NULL` (field **must** have `null=True`). | `->onDelete('set null')` |
| `SET_DEFAULT` | FK set to the field's `default` (must define a `default`). | n/a |
| `SET(...)` | FK set to a value/callable you supply. | n/a |
| `DO_NOTHING` | Django does nothing — you take full responsibility (risk of integrity errors unless the DB handles it). | `->onDelete('no action')` |

```python
from django.db import models
from django.contrib.auth.models import User


def get_sentinel_user():
    # Used with SET(): reassign content to a placeholder "deleted" user.
    return User.objects.get_or_create(username="deleted")[0]


class Article(models.Model):
    title = models.CharField(max_length=200)

    # Block deletion of a category that still has articles.
    category = models.ForeignKey(
        "Category", on_delete=models.PROTECT, related_name="articles"
    )

    # If the author is deleted, keep the article but null the link.
    author = models.ForeignKey(
        User, on_delete=models.SET_NULL, null=True, blank=True,
        related_name="articles",
    )

    # If the editor is deleted, reassign to a sentinel account.
    editor = models.ForeignKey(
        User, on_delete=models.SET(get_sentinel_user),
        related_name="edited_articles",
    )
```

Demonstrating `PROTECT`:

```python
from django.db.models.deletion import ProtectedError

try:
    category.delete()
except ProtectedError as exc:
    print("Cannot delete:", exc)
    # => Cannot delete: ("Cannot delete some instances of model 'Category' ...")
```

> **Rule of thumb:** Use `PROTECT` for reference data you never want silently destroyed (categories, currencies). Use `CASCADE` for true ownership where children are meaningless without the parent (order → order lines). Use `SET_NULL` for soft links where the child survives (article keeps existing even if its author is removed).

### `related_name` and `related_query_name`

`related_name` is the attribute used to walk **backwards**, from the "one" side to the "many" side.

```python
publisher = Publisher.objects.get(name="O'Reilly")

# Reverse accessor -> a RelatedManager (like a queryset factory)
publisher.books.all()                       # all books of this publisher
publisher.books.count()                     # => 17
publisher.books.filter(published_year=2024) # chainable like any queryset
```

If you **don't** set `related_name`, Django defaults to `<model>_set` (e.g. `publisher.book_set`). Setting `related_name="books"` reads far better and is strongly recommended.

`related_query_name` is different — it's the name used inside **`filter()` lookups** that cross the relation. It defaults to the model's lowercase name (or to `related_name` if you don't set it explicitly).

```python
class Book(models.Model):
    publisher = models.ForeignKey(
        Publisher,
        on_delete=models.CASCADE,
        related_name="books",            # used for publisher.books
        related_query_name="book",       # used for filter(book__...)
    )

# Find publishers that have at least one 2024 book:
Publisher.objects.filter(book__published_year=2024).distinct()
#                          ^^^^ related_query_name, not related_name
```

> Tip: if you set `related_name` but not `related_query_name`, the query name falls back to `related_name`. Many teams just set `related_name` and use that for both. Set `related_query_name` only when you want the accessor and the filter keyword to differ.

## Core concept: OneToOneField (profile extension)

A `OneToOneField` is a `ForeignKey` with a `unique=True` constraint baked in: each row on one side maps to **at most one** row on the other. The textbook use case is **extending a model you don't want to edit** — most often Django's built-in `User`.

```python
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,   # use this, not a hardcoded import
        on_delete=models.CASCADE,   # delete the user -> delete the profile
        related_name="profile",     # access as user.profile
    )
    bio = models.TextField(blank=True)
    avatar_url = models.URLField(blank=True)
    twitter = models.CharField(max_length=50, blank=True)

    def __str__(self):
        return f"Profile<{self.user.username}>"
```

Using it — note the reverse side is a **single object**, not a manager:

```python
user = User.objects.get(username="ada")

user.profile.bio          # forward-ish: one-to-one reverse is a single object
user.profile.twitter = "@ada"
user.profile.save()

profile = Profile.objects.get(pk=1)
profile.user.email        # forward: jump to the user
```

**Gotcha:** if a `User` has no `Profile` yet, `user.profile` raises `Profile.DoesNotExist` (technically `User.profile.RelatedObjectDoesNotExist`). Guard with `hasattr(user, "profile")` or create the profile via a signal / `get_or_create`.

```python
# Create the missing side safely
Profile.objects.get_or_create(user=user, defaults={"bio": "New here"})
```

Compared to Laravel: this is `hasOne` on `User` and `belongsTo` on `Profile`, with the database guaranteeing the one-to-one via a unique index on `user_id`.

## Core concept: ManyToManyField and the auto join table

Many-to-many: a `Book` has many `Author`s, and an `Author` writes many `Book`s. Put a `ManyToManyField` on **either** model (not both) — Django creates a hidden **join table** automatically.

```python
class Author(models.Model):
    name = models.CharField(max_length=200)

    def __str__(self):
        return self.name


class Book(models.Model):
    title = models.CharField(max_length=300)
    authors = models.ManyToManyField(Author, related_name="books")

    def __str__(self):
        return self.title
```

Django silently creates a table like `library_book_authors` with columns `id`, `book_id`, `author_id` — the equivalent of a Laravel pivot table you'd otherwise hand-write in a migration. You never declare it; the ORM owns it.

Managing the relation (these hit the DB immediately, no `.save()` needed):

```python
book = Book.objects.create(title="Refactoring")
ada = Author.objects.create(name="Ada")
linus = Author.objects.create(name="Linus")

book.authors.add(ada, linus)        # link both
book.authors.remove(linus)          # unlink one
book.authors.set([ada])             # replace the whole set
book.authors.clear()                # remove all links (rows in join table)

book.authors.all()                  # forward: this book's authors
ada.books.all()                     # reverse (related_name): Ada's books
ada.books.count()                   # => 1
```

`add()` is idempotent — adding the same author twice does not create duplicate join rows. `set()` is the convenient "make the relation exactly equal this list" operation (it diffs and applies add/remove for you).

## Core concept: custom `through` models (M2M with extra fields)

The auto join table only stores the two foreign keys. The moment you need **extra data on the relationship itself** — when someone enrolled, what role they have, the price at time of purchase — you need an explicit **`through` model**. This is exactly Eloquent's "pivot model" (`belongsToMany(...)->using(Enrollment::class)->withPivot(...)`).

```python
from django.db import models


class Student(models.Model):
    name = models.CharField(max_length=200)

    def __str__(self):
        return self.name


class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField(
        Student,
        through="Enrollment",          # use our model instead of the auto one
        related_name="courses",
    )

    def __str__(self):
        return self.title


class Enrollment(models.Model):
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    course = models.ForeignKey(Course, on_delete=models.CASCADE)
    # The "extra fields" that justify a through model:
    date_enrolled = models.DateField(auto_now_add=True)
    grade = models.CharField(max_length=2, blank=True)
    is_active = models.BooleanField(default=True)

    class Meta:
        # Prevent the same student enrolling in the same course twice.
        unique_together = ("student", "course")

    def __str__(self):
        return f"{self.student} in {self.course}"
```

**Important rule:** with a custom `through`, calling `add()`, `create()`, or `set()` directly on the M2M only works when Django can fill the extra fields — i.e. they have defaults or you supply them via `through_defaults` (supported since Django 2.2). If a required extra field has no default and you don't supply one, the call raises an error. The most explicit, always-safe approach is to create rows on the `through` model yourself:

```python
ada = Student.objects.create(name="Ada")
algo = Course.objects.create(title="Algorithms")

# Risky: only works if every extra field has a default or is supplied via
# through_defaults; otherwise it errors because `grade` etc. can't be filled.
# algo.students.add(ada)

# Always safe: create the join row explicitly with all the extra fields.
Enrollment.objects.create(student=ada, course=algo, grade="A")
```

Querying through the relation:

```python
# Forward / reverse still work for reading:
algo.students.all()        # all students in this course
ada.courses.all()          # all courses Ada takes

# Reach the extra fields by querying the through model:
Enrollment.objects.filter(course=algo, is_active=True)
Enrollment.objects.filter(student=ada, grade="A")

# Cross the relation with __ lookups (note the lowercase through-model name):
Course.objects.filter(enrollment__grade="A").distinct()
Student.objects.filter(enrollment__course=algo, enrollment__is_active=True)
```

Django 2.2+ also lets you pass `through_defaults` when you *do* use `add()`/`set()`/`create()` and your through model's extra fields have defaults (or you supply them):

```python
# Allowed if extra fields are optional or provided via through_defaults:
algo.students.add(ada, through_defaults={"grade": "B"})
```

## Core concept: accessing forward and reverse relations

Keep this table in your head — it's the most common source of "wait, is it a manager or an object?" confusion.

| Relationship | Forward (where the field is) | Reverse (the other side) | Reverse is a... |
|---|---|---|---|
| `ForeignKey` | `book.publisher` → one object | `publisher.books` (`related_name`) | **Manager** (many) |
| `OneToOneField` | `profile.user` → one object | `user.profile` (`related_name`) | **single object** |
| `ManyToManyField` | `book.authors` → manager | `author.books` (`related_name`) | **Manager** (many) |

The practical consequence: FK and M2M reverse sides give you a `RelatedManager`, so you call `.all()`, `.filter()`, `.count()`, etc. The one-to-one reverse gives you the object directly (and raises `DoesNotExist` if absent).

```python
# Forward FK -> object (may trigger a query)
book.publisher.name

# Reverse FK -> manager -> queryset
for b in publisher.books.filter(published_year__gte=2020):
    print(b.title)

# M2M both directions -> managers
book.authors.all()
author.books.all()

# O2O reverse -> object, not manager (no .all()!)
user.profile.bio
```

## Core concept: querying across relationships with `__`

The double-underscore (`__`) is Django's way of "joining" inside a `filter()`/`exclude()`/`order_by()` without writing SQL joins. Each `__` step walks one relationship; the final segment is a field or a lookup (`__gte`, `__icontains`, `__in`, etc.).

```python
# Books whose publisher is in Ireland:
Book.objects.filter(publisher__country="Ireland")

# Books written by an author named like "ada" (M2M traversal):
Book.objects.filter(authors__name__icontains="ada").distinct()

# Two hops: articles whose author's profile mentions "django":
Article.objects.filter(author__profile__bio__icontains="django")

# Reverse direction works too (uses related_query_name):
# Publishers that have at least one 2024 book:
Publisher.objects.filter(books__published_year=2024).distinct()

# Aggregation across a relation:
from django.db.models import Count
Publisher.objects.annotate(num_books=Count("books")).filter(num_books__gt=5)
```

Two things to remember:

1. **`.distinct()` after M2M / reverse filters.** Joining across a to-many relation can return the same parent row multiple times (once per matching child). Add `.distinct()` to collapse duplicates.
2. **Chained `.filter()` vs. one `.filter()` differ for to-many relations.** `filter(a=1).filter(b=2)` means "rows that have *some* child with a=1 AND *some* (possibly different) child with b=2," while `filter(a=1, b=2)` means "rows that have a *single* child matching both." This trips up everyone once.

```python
# Different meanings on a to-many relation:
Course.objects.filter(enrollment__grade="A").filter(enrollment__is_active=True)
# -> courses with some A enrollment AND some active enrollment (maybe different rows)

Course.objects.filter(enrollment__grade="A", enrollment__is_active=True)
# -> courses with a single enrollment that is both grade A and active
```

## Core concept: self-referential foreign keys

A model can point at itself — for hierarchies (category → parent category) or graphs (employee → manager). Use the string `"self"` as the target.

```python
class Employee(models.Model):
    name = models.CharField(max_length=200)
    manager = models.ForeignKey(
        "self",                      # point at the same model
        on_delete=models.SET_NULL,
        null=True, blank=True,       # the CEO has no manager
        related_name="reports",      # manager.reports -> direct reports
    )

    def __str__(self):
        return self.name
```

```python
ceo = Employee.objects.create(name="Grace")
dev = Employee.objects.create(name="Ada", manager=ceo)

dev.manager.name        # => "Grace"      (forward)
ceo.reports.all()       # => [<Employee: Ada>]  (reverse)

# Cross the relation in queries:
Employee.objects.filter(manager__name="Grace")        # Grace's reports
Employee.objects.filter(manager__isnull=True)         # top-level (no manager)
```

A **self-referential ManyToMany** models symmetric or directed graphs (e.g. "friends", "follows"):

```python
class Person(models.Model):
    name = models.CharField(max_length=100)
    # symmetrical=True (default for self M2M): if A follows B, B follows A.
    # Set symmetrical=False for one-directional "follows".
    follows = models.ManyToManyField("self", symmetrical=False, related_name="followers")

# alice.follows.add(bob)   ->   bob in alice.follows.all(), alice in bob.followers.all()
```

## Core concept: a multi-table example tying it together

Here is a small but realistic schema using every relationship type at once.

```python
from django.conf import settings
from django.db import models


class Publisher(models.Model):
    name = models.CharField(max_length=200)

    def __str__(self):
        return self.name


class Author(models.Model):
    name = models.CharField(max_length=200)

    def __str__(self):
        return self.name


class AuthorProfile(models.Model):                 # OneToOne (profile extension)
    author = models.OneToOneField(
        Author, on_delete=models.CASCADE, related_name="profile"
    )
    website = models.URLField(blank=True)


class Book(models.Model):
    title = models.CharField(max_length=300)
    publisher = models.ForeignKey(                 # many-to-one
        Publisher, on_delete=models.PROTECT, related_name="books"
    )
    authors = models.ManyToManyField(              # many-to-many w/ through
        Author, through="Contribution", related_name="books"
    )

    def __str__(self):
        return self.title


class Contribution(models.Model):                  # the pivot / through model
    book = models.ForeignKey(Book, on_delete=models.CASCADE)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    role = models.CharField(max_length=50, default="author")  # author/editor/translator
    royalty_pct = models.DecimalField(max_digits=5, decimal_places=2, default=0)

    class Meta:
        unique_together = ("book", "author", "role")
```

Exercising it:

```python
oreilly = Publisher.objects.create(name="O'Reilly")
ada = Author.objects.create(name="Ada")
AuthorProfile.objects.create(author=ada, website="https://ada.dev")

book = Book.objects.create(title="Patterns", publisher=oreilly)
Contribution.objects.create(book=book, author=ada, role="author", royalty_pct=12.5)

book.authors.all()                       # M2M forward -> [Ada]
ada.books.all()                          # M2M reverse -> [Patterns]
ada.profile.website                      # O2O reverse -> "https://ada.dev"
oreilly.books.count()                    # FK reverse  -> 1

# Cross-relation queries:
Book.objects.filter(authors__profile__website__icontains="ada").distinct()
Book.objects.filter(contribution__role="editor").distinct()
Author.objects.filter(books__publisher__name="O'Reilly").distinct()
```

## Eloquent → Django cheat-map

| Eloquent | Django | Notes |
|---|---|---|
| `belongsTo(Publisher::class)` | `ForeignKey(Publisher, on_delete=...)` | Field on the "many"/child side |
| `hasMany(Book::class)` | auto reverse of FK (`related_name`) | Generated, not declared |
| `hasOne(Profile::class)` / `belongsTo(User::class)` | `OneToOneField(User, ...)` | Unique constraint built in |
| `belongsToMany(Author::class)` | `ManyToManyField(Author)` | Auto join table |
| pivot table | auto-created `app_model_field` table | Hidden, ORM-managed |
| pivot model + `withPivot()` + `using()` | `ManyToManyField(through="Model")` | Explicit through model |
| `$book->authors()->attach($id)` | `book.authors.add(author)` | |
| `$book->authors()->detach($id)` | `book.authors.remove(author)` | |
| `$book->authors()->sync([...])` | `book.authors.set([...])` | |
| `whereHas('author', fn ...)` | `filter(author__...)` | `__` traversal |

## Common mistakes & gotchas

- **Forgetting `on_delete`.** It is required for `ForeignKey` and `OneToOneField`; omitting it is a `TypeError` at import time.
- **`SET_NULL` without `null=True`.** You must also make the column nullable, or migrations will fail / the DB will reject the NULL.
- **Calling `.add()`/`.set()` on an M2M that has a `through` model.** Historically not allowed; since Django 2.2 it works if you pass `through_defaults` (or the extra fields have defaults). When in doubt, create rows on the through model instead.
- **Treating the O2O reverse like a manager.** `user.profile.all()` is wrong — the reverse of `OneToOneField` is a single object, not a manager.
- **Unhandled `RelatedObjectDoesNotExist`** when a one-to-one partner is missing. Use `hasattr()` or `get_or_create`.
- **Missing `.distinct()`** after filtering across a to-many relation → duplicate parent rows.
- **Confusing `related_name` with `related_query_name`.** `related_name` = the Python attribute for the reverse manager; `related_query_name` = the keyword used inside `filter()`.
- **Hardcoding `from django.contrib.auth.models import User`** in FK/O2O targets. Prefer `settings.AUTH_USER_MODEL` so a custom user model doesn't break you later.
- **N+1 queries.** Accessing `book.publisher` in a loop fires one query per book. Fix with `select_related` (FK/O2O) and `prefetch_related` (M2M/reverse FK) — covered in the queries lesson.
- **`unique_together` forgotten on through models**, allowing accidental duplicate links.

## Best practices

- Always set an explicit, descriptive `related_name` (`"books"`, `"reports"`) instead of relying on `*_set`.
- Choose `on_delete` deliberately per relationship; `PROTECT` for reference data, `CASCADE` for true ownership, `SET_NULL` for soft links.
- Reference the user model via `settings.AUTH_USER_MODEL`, and reference models by string (`"Enrollment"`) to dodge import-order problems.
- Add `unique_together` (or `UniqueConstraint`) on through models to prevent duplicate relationships.
- Use a `through` model the moment the relationship needs its own attributes — don't bolt extra columns onto unrelated tables.
- Add `db_index=True` (FKs are indexed automatically) and consider composite indexes for hot cross-relation queries.
- Remember `.distinct()` when filtering across to-many relations.
- Pair every loop that crosses a relation with `select_related`/`prefetch_related`.

## Interview Q&A

**Q: On which side of a one-to-many relationship does the `ForeignKey` go, and why?**
**A:** On the "many" (child) side. A `Book` belongs to one `Publisher`, so `Book` gets the `ForeignKey`, creating a `publisher_id` column on the book table. The "one" side gets an auto-generated reverse manager (`publisher.books`). Putting it on the many side keeps each foreign key a single scalar column instead of an array.

**Q: Why does Django force you to specify `on_delete`, and what are the main options?**
**A:** To make the deletion policy explicit and enforced at the ORM level rather than guessed. `CASCADE` deletes children, `PROTECT` blocks deletion (`ProtectedError`), `SET_NULL` nulls the FK (needs `null=True`), `SET_DEFAULT`/`SET(...)` set a fallback value, and `DO_NOTHING` leaves it entirely to you (risking integrity errors).

**Q: What is the difference between `related_name` and `related_query_name`?**
**A:** `related_name` is the attribute you use to access the reverse relation in Python (`publisher.books`). `related_query_name` is the keyword used inside ORM lookups crossing that relation (`Publisher.objects.filter(book__...)`). If you set only `related_name`, the query name defaults to it; set `related_query_name` only when you want them to differ.

**Q: When would you use a `OneToOneField` instead of a `ForeignKey`?**
**A:** When each row maps to at most one row on the other side — e.g. extending `User` with a `Profile`, or splitting rarely-used columns into a separate table. It's a `ForeignKey` with `unique=True`, and its reverse accessor returns a single object rather than a manager.

**Q: How does Django implement a `ManyToManyField` under the hood?**
**A:** It creates a hidden join table with two foreign keys (e.g. `book_id`, `author_id`) and a primary key. You never declare that table; the ORM manages it. You manipulate it through `add`, `remove`, `set`, and `clear` on the manager.

**Q: When do you need a custom `through` model, and what changes about how you manage the relation?**
**A:** When the relationship itself carries data (enrolled date, role, price). You declare `ManyToManyField(through="Model")` and an explicit join model with those extra fields. Once a `through` model exists, you can't use `add()`/`create()`/`set()` directly (unless you pass `through_defaults`); you create rows on the through model instead.

**Q: Reverse of a `ForeignKey` vs. reverse of a `OneToOneField` — what's the difference in code?**
**A:** The reverse of a `ForeignKey` is a `RelatedManager` — you call `.all()`, `.filter()`, etc. The reverse of a `OneToOneField` is the related object directly (e.g. `user.profile`), and it raises `RelatedObjectDoesNotExist` if the partner row is missing.

**Q: What does `Book.objects.filter(authors__name="Ada")` do, and why might you need `.distinct()`?**
**A:** It joins through the M2M to the `Author` table and returns books having an author named "Ada." Because the join can match multiple author rows per book, the same book can appear more than once; `.distinct()` collapses those duplicates.

**Q: Explain the difference between chained `.filter()` calls and a single `.filter()` with multiple conditions across a to-many relation.**
**A:** `filter(x=1).filter(y=2)` requires *some* related row with x=1 and *some* (possibly different) related row with y=2. `filter(x=1, y=2)` requires a *single* related row satisfying both. They produce different SQL and different results for to-many relations.

**Q: How do you model an employee/manager hierarchy in Django?**
**A:** A self-referential `ForeignKey("self", on_delete=..., null=True, related_name="reports")`. `employee.manager` walks up, `manager.reports.all()` walks down. Allow `null` so the top of the hierarchy has no manager.

**Q: How do you map Eloquent's `belongsToMany` with pivot fields to Django?**
**A:** `belongsToMany` ≈ `ManyToManyField`; the pivot table is the auto join table; pivot *fields* (`withPivot`) and a pivot *model* (`using()`) map to a Django `through` model carrying the extra columns.

**Q: Why prefer `settings.AUTH_USER_MODEL` over importing `User` directly in relationship fields?**
**A:** Projects sometimes swap in a custom user model. Referencing `settings.AUTH_USER_MODEL` (a string) keeps your FKs/O2Os pointing at whatever user model is configured and avoids import-order and circular-import problems.

## Practice exercises

1. **Blog FK.** Create `Category` and `Post` where each post belongs to one category. Use `on_delete=PROTECT` and `related_name="posts"`. Then write a query for all posts in the category named "Python".
2. **Profile extension.** Add a `Profile` one-to-one to `settings.AUTH_USER_MODEL` with `bio` and `website`. Write code that safely returns a user's bio, returning `""` if no profile exists.
3. **Tags (plain M2M).** Give `Post` a `ManyToManyField(Tag)`. Add two tags to a post, then query all posts that have the tag "django".
4. **Enrollment (through model).** Model `Student`, `Course`, and an `Enrollment` through model with `date_enrolled` and `grade`. Enroll a student with grade "A", then query all courses where that student scored "A".
5. **Org chart (self FK).** Model `Employee` with a self-referential `manager`. Create a 3-level chain and write a query returning everyone who reports (directly) to a given manager.
6. **Cross-relation filter.** Using exercises 1 and 2's models, write one query returning all posts authored by users whose profile `website` contains "github". Make sure duplicates are collapsed.

<details>
<summary>Solutions</summary>

```python
# 1. Blog FK
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=100)


class Post(models.Model):
    title = models.CharField(max_length=200)
    category = models.ForeignKey(
        Category, on_delete=models.PROTECT, related_name="posts"
    )

# Query:
Post.objects.filter(category__name="Python")


# 2. Profile extension
from django.conf import settings


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="profile"
    )
    bio = models.TextField(blank=True)
    website = models.URLField(blank=True)


def safe_bio(user):
    return user.profile.bio if hasattr(user, "profile") else ""


# 3. Tags (plain M2M)
class Tag(models.Model):
    name = models.CharField(max_length=50)


class Post(models.Model):          # extend the earlier Post
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(Tag, related_name="posts")

# Usage:
post = Post.objects.create(title="Hello")
django_tag = Tag.objects.create(name="django")
orm_tag = Tag.objects.create(name="orm")
post.tags.add(django_tag, orm_tag)

Post.objects.filter(tags__name="django").distinct()


# 4. Enrollment (through model)
class Student(models.Model):
    name = models.CharField(max_length=200)


class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField(Student, through="Enrollment", related_name="courses")


class Enrollment(models.Model):
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    course = models.ForeignKey(Course, on_delete=models.CASCADE)
    date_enrolled = models.DateField(auto_now_add=True)
    grade = models.CharField(max_length=2, blank=True)

    class Meta:
        unique_together = ("student", "course")

# Usage:
ada = Student.objects.create(name="Ada")
algo = Course.objects.create(title="Algorithms")
Enrollment.objects.create(student=ada, course=algo, grade="A")

Course.objects.filter(enrollment__student=ada, enrollment__grade="A").distinct()


# 5. Org chart (self FK)
class Employee(models.Model):
    name = models.CharField(max_length=200)
    manager = models.ForeignKey(
        "self", on_delete=models.SET_NULL, null=True, blank=True, related_name="reports"
    )

# Usage:
ceo = Employee.objects.create(name="Grace")
lead = Employee.objects.create(name="Linus", manager=ceo)
dev = Employee.objects.create(name="Ada", manager=lead)

lead.reports.all()                          # direct reports of Linus -> [Ada]
Employee.objects.filter(manager=lead)       # same thing as a query


# 6. Cross-relation filter (two hops + distinct)
Post.objects.filter(
    author__profile__website__icontains="github"
).distinct()
# (assumes Post gained an `author = ForeignKey(settings.AUTH_USER_MODEL, ...)`)
```

</details>

## Cheat sheet

```python
# --- Field declarations ---
ForeignKey(Target, on_delete=models.CASCADE, related_name="children")   # many-to-one
OneToOneField(Target, on_delete=models.CASCADE, related_name="profile") # one-to-one
ManyToManyField(Target, related_name="items")                          # many-to-many (auto join)
ManyToManyField(Target, through="Link", related_name="items")          # m2m + extra fields
ForeignKey("self", on_delete=models.SET_NULL, null=True,
           related_name="reports")                                      # self-referential

# --- on_delete ---
CASCADE | PROTECT | RESTRICT | SET_NULL(+null=True) | SET_DEFAULT(+default) | SET(fn) | DO_NOTHING

# --- Access ---
child.parent                 # FK forward -> object
parent.children.all()        # FK reverse -> manager
profile.user                 # O2O forward -> object
user.profile                 # O2O reverse -> object (DoesNotExist if missing)
book.authors.all()           # M2M forward -> manager
author.books.all()           # M2M reverse -> manager

# --- M2M management (plain) ---
m.items.add(a, b); m.items.remove(a); m.items.set([a]); m.items.clear()
m.items.add(a, through_defaults={"role": "x"})   # only when through model allows it

# --- through model: create join rows directly ---
Link.objects.create(book=b, author=a, role="editor")

# --- queries across relations (__) ---
Model.objects.filter(rel__field=value)
Model.objects.filter(rel__sub__field__icontains="x").distinct()
Parent.objects.annotate(n=Count("children")).filter(n__gt=5)

# related_name -> reverse attribute   |   related_query_name -> filter() keyword
```
