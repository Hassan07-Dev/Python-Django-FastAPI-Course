# 27 — Models & ORM Basics

> Week 3 · Django ORM

## What you will learn

- How to define a Django **model** by subclassing `models.Model`, and how that one class becomes a database table.
- The most common **field types** (`CharField`, `TextField`, `IntegerField`, `BooleanField`, `DateTimeField`, `EmailField`, `DecimalField`, `SlugField`, …) and the **options** that shape them (`null`, `blank`, `default`, `unique`, `choices`, `db_index`, `max_length`).
- The crucial Python/Django distinction between **`null` and `blank`** (this trips up everyone).
- The implicit **auto `id` primary key** Django gives you for free.
- The **`Meta`** inner class (`ordering`, `db_table`, `verbose_name`, `constraints`) and the **`__str__`** method.
- How **migrations** work: `makemigrations` (write the migration) vs `migrate` (apply it).
- **CRUD** with the ORM: `create`, `save`, `all`, `get`, `filter`, updating, and deleting.
- How to poke at all of this interactively in the **Django shell**.
- A running comparison to **Eloquent models & Laravel migrations** so you can lean on what you already know.

## PHP/Laravel bridge

In Laravel, a database table is described in **two separate places**: a migration file (`Schema::create(...)`) defines the columns, and an Eloquent model (`class User extends Model`) gives you the query/active-record API. Django merges both ideas: your **model class IS the schema**. You declare fields on the class, and Django *generates* the migration for you by diffing your models against the last known state. So one `class Article(models.Model)` plays the role of both `database/migrations/..._create_articles_table.php` **and** `app/Models/Article.php`. Querying is done through a **manager** (`Article.objects`) rather than static calls on the model itself (`Article::all()` in Laravel becomes `Article.objects.all()`), and there is no `$fillable`/`$guarded` mass-assignment layer — that protection lives in serializers/forms, not the model.

## Core concept — A model is a class that maps to a table

A Django model is a plain Python class that inherits from `django.db.models.Model`. Each **class attribute** that is a `Field` instance becomes a **column**. Each **instance** of the class is a **row**.

Assume you've already created a project and an app (covered in the project-setup lesson):

```bash
django-admin startproject config .
python manage.py startapp blog
```

Then register the app in `config/settings.py` so Django knows to look for its models and migrations:

```python
# config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "blog",            # <- your app
]
```

Now the model itself, in `blog/models.py`:

```python
# blog/models.py
from django.db import models


class Article(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self) -> str:
        return self.title
```

That's a complete, working model. Django will create a table named `blog_article` (app label + lowercased class name) with columns `id`, `title`, `body`, `is_published`, `created_at`, `updated_at`.

> **Why a class and not a config file?** Because the class is also your runtime object. The same `Article` you defined the schema with is the object you create, validate, and save. One source of truth.

## Core concept — The implicit `id` primary key

You did **not** declare a primary key above, yet you get one. Django automatically adds:

```python
id = models.BigAutoField(primary_key=True)   # added for you, conceptually
```

It's a 64-bit auto-incrementing integer (`BigAutoField` is the default in modern Django, controlled by `DEFAULT_AUTO_FIELD` in settings). You access it as `article.id` or, generically, `article.pk` (`pk` always points at the primary key, whatever it's called).

If you want a different primary key, declare one explicitly and the auto field is suppressed:

```python
class Product(models.Model):
    sku = models.CharField(max_length=32, primary_key=True)   # no auto id now
    name = models.CharField(max_length=200)
```

> Laravel parallel: this is exactly like `$table->id()` being added by convention — except in Django it's added even when you write nothing, and `pk` is the universal alias.

## Core concept — Common field types

Fields are how you describe both the **database column type** and **validation rules**. Here is a tour of the ones you'll use constantly.

```python
# blog/models.py
from django.db import models


class Author(models.Model):
    # Short, bounded text. max_length is REQUIRED. Maps to VARCHAR.
    name = models.CharField(max_length=120)

    # Unbounded text. Maps to TEXT. No max_length needed.
    bio = models.TextField(blank=True)

    # Validated email string (still a VARCHAR under the hood, default max_length=254).
    email = models.EmailField(unique=True)

    # URL-friendly string, e.g. "my-first-post". Validated to contain only
    # letters, numbers, hyphens, underscores.
    slug = models.SlugField(max_length=140, unique=True)

    # Whole numbers.
    follower_count = models.IntegerField(default=0)
    # PositiveIntegerField rejects negatives; SmallIntegerField/BigIntegerField change range.

    # True/False. Maps to BOOLEAN.
    is_active = models.BooleanField(default=True)

    # Exact decimal — ALWAYS use this for money, never FloatField.
    # max_digits = total digits; decimal_places = digits after the point.
    balance = models.DecimalField(max_digits=10, decimal_places=2, default=0)

    # Date + time.
    created_at = models.DateTimeField(auto_now_add=True)   # set once, on INSERT
    updated_at = models.DateTimeField(auto_now=True)       # overwritten on every save()

    # Date only / time only also exist:
    birthday = models.DateField(null=True, blank=True)

    def __str__(self) -> str:
        return self.name
```

### Field cheat table

| Django field | DB column (Postgres) | Python type you work with | Laravel migration equivalent |
|---|---|---|---|
| `CharField(max_length=n)` | `varchar(n)` | `str` | `$table->string('x', n)` |
| `TextField()` | `text` | `str` | `$table->text('x')` |
| `IntegerField()` | `integer` | `int` | `$table->integer('x')` |
| `BigIntegerField()` | `bigint` | `int` | `$table->bigInteger('x')` |
| `BooleanField()` | `boolean` | `bool` | `$table->boolean('x')` |
| `DecimalField(max_digits, decimal_places)` | `numeric` | `decimal.Decimal` | `$table->decimal('x', m, d)` |
| `FloatField()` | `double precision` | `float` | `$table->float('x')` |
| `DateField()` | `date` | `datetime.date` | `$table->date('x')` |
| `DateTimeField()` | `timestamp` | `datetime.datetime` | `$table->dateTime('x')` |
| `EmailField()` | `varchar(254)` | `str` | `$table->string('email')` |
| `SlugField()` | `varchar(50)` | `str` | `$table->string('slug')` |
| `UUIDField()` | `uuid` | `uuid.UUID` | `$table->uuid('x')` |
| `JSONField()` | `jsonb` | `dict`/`list` | `$table->json('x')` |

### `auto_now` vs `auto_now_add`

These two `DateTimeField` options are Django's answer to Laravel's `created_at` / `updated_at` timestamps.

- `auto_now_add=True` — set **once**, the moment the row is first inserted. Think `created_at`. It is not editable afterward.
- `auto_now=True` — overwritten with "now" **every time you call `.save()`**. Think `updated_at`.

> Gotcha: `auto_now` only fires on `.save()`. A bulk `QuerySet.update(...)` does **not** call `save()`, so it will **not** touch an `auto_now` field. More on that below.

## Core concept — Field options that shape columns and validation

Options are keyword arguments passed to the field. The high-frequency ones:

```python
class Tag(models.Model):
    name = models.CharField(
        max_length=50,
        unique=True,        # adds a UNIQUE constraint
        db_index=True,      # adds a database index (faster lookups/filters)
    )
    description = models.CharField(
        max_length=200,
        blank=True,         # form/validation: value may be empty string
        default="",         # default used when none supplied
    )
    priority = models.IntegerField(
        default=0,
        null=True,          # database: column may store NULL
    )
```

### The single most important distinction: `null` vs `blank`

This is the #1 Django beginner confusion, so let's be crisp:

| Option | Layer it affects | Means |
|---|---|---|
| `null=True` | **Database** | The column may contain SQL `NULL`. |
| `blank=True` | **Validation** (forms, DRF, admin) | The field is allowed to be empty during validation. |

They are independent.

- For **string-based** fields (`CharField`, `TextField`), the Django convention is to use `blank=True` and **avoid `null=True`** — because then "no value" is consistently the empty string `""`, not a confusing mix of `""` and `NULL`. Use `default=""` if you like.
- For **non-string** fields (numbers, dates, FKs) where "no value" must be representable in the DB, use `null=True` (and usually `blank=True` too so forms accept emptiness).

```python
# GOOD: a string that may be omitted
nickname = models.CharField(max_length=50, blank=True)        # stores "" when empty

# GOOD: a date that may be genuinely absent
deleted_at = models.DateTimeField(null=True, blank=True)      # stores NULL when empty

# USUALLY WRONG: nullable string -> two "empty" states (NULL and "")
nickname = models.CharField(max_length=50, null=True, blank=True)
```

### `choices` — enumerated values

`choices` restricts a field to a set of allowed values and gives you a human label for each. The modern idiom uses an enumeration class:

```python
class Article(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"
        ARCHIVED = "archived", "Archived"

    title = models.CharField(max_length=200)
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT,
    )
```

Usage gives you both the stored value and a display helper:

```python
a = Article(status=Article.Status.PUBLISHED)
a.status                     # => 'published'   (what's stored)
a.get_status_display()       # => 'Published'   (the human label; auto-generated method)
```

> Laravel parallel: like casting a column to a PHP `enum` plus an accessor for the label, but built in.

## Core concept — The `Meta` inner class

`Meta` holds table-level metadata that isn't a column. It is an inner class named exactly `Meta`.

```python
class Article(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, default="draft")

    class Meta:
        # Default sort order for ALL queries on this model.
        # "-created_at" means descending; newest first.
        ordering = ["-created_at"]

        # Override the auto-generated table name ("blog_article") if you must.
        db_table = "articles"

        # Human-readable names (used by the admin, error messages, etc.)
        verbose_name = "article"
        verbose_name_plural = "articles"

        # Database-level constraints (preferred over app-level checks).
        constraints = [
            models.UniqueConstraint(
                fields=["slug"], name="uniq_article_slug"
            ),
            models.CheckConstraint(
                condition=~models.Q(title=""),  # Django 5.1+: 'condition' (was 'check')
                name="title_not_empty",
            ),
        ]

        # Multi-column indexes:
        indexes = [
            models.Index(fields=["status", "-created_at"], name="status_created_idx"),
        ]

    def __str__(self) -> str:
        return self.title
```

A few notes:

- `ordering` is convenient but applies to **every** query unless you override it with `.order_by()`. Don't set it casually on big tables — it adds an `ORDER BY` to queries you didn't intend to sort.
- `constraints` enforce rules in the **database**, so they hold even if some code path skips your Python validation. Prefer `UniqueConstraint` over the field-level `unique=True` when you need multi-column uniqueness.
- In Django 5.1+ `CheckConstraint` takes `condition=`; older code used `check=` (now deprecated). Use `condition`.

## Core concept — `__str__`: the readable representation

`__str__` is a standard Python dunder method that returns the string shown when an object is printed. Django uses it heavily — in the admin, in the shell, in error messages. Always define it.

```python
class Author(models.Model):
    name = models.CharField(max_length=120)
    email = models.EmailField()

    def __str__(self) -> str:
        return f"{self.name} <{self.email}>"
```

```python
a = Author(name="Ada", email="ada@example.com")
print(a)          # => Ada <ada@example.com>
repr(a)           # => <Author: Ada <ada@example.com>>   (Django wraps __str__)
```

> Without `__str__`, the shell shows the unhelpful `<Author: Author object (1)>`. This is the rough equivalent of customizing how a model serializes for display in Laravel — but it's a first-class Python method.

## Core concept — Migrations: from model to table

You've described the schema in Python. Nothing exists in the database yet. Two commands bridge that gap.

```bash
# 1) Generate migration files by diffing your models against the last migration state.
python manage.py makemigrations
# Output:
# Migrations for 'blog':
#   blog/migrations/0001_initial.py
#     - Create model Article

# 2) Apply pending migrations to the actual database (runs the SQL).
python manage.py migrate
# Output:
# Operations to perform:
#   Apply all migrations: admin, auth, blog, contenttypes, sessions
# Running migrations:
#   Applying blog.0001_initial... OK
```

The mental model:

| Step | Laravel | Django |
|---|---|---|
| Describe schema | hand-write migration `Schema::create` | **edit the model class** |
| Generate migration | (you write it) | `makemigrations` (auto-generates) |
| Apply migration | `php artisan migrate` | `migrate` |
| Roll back | `php artisan migrate:rollback` | `migrate blog <prev>` (e.g. `migrate blog 0001`, or `migrate blog zero` to undo all) |

A generated migration is just Python you can read and even hand-edit:

```python
# blog/migrations/0001_initial.py (excerpt)
from django.db import migrations, models


class Migration(migrations.Migration):
    initial = True
    dependencies = []

    operations = [
        migrations.CreateModel(
            name="Article",
            fields=[
                ("id", models.BigAutoField(auto_created=True, primary_key=True,
                                           serialize=False, verbose_name="ID")),
                ("title", models.CharField(max_length=200)),
                ("body", models.TextField()),
                ("is_published", models.BooleanField(default=False)),
                ("created_at", models.DateTimeField(auto_now_add=True)),
                ("updated_at", models.DateTimeField(auto_now=True)),
            ],
        ),
    ]
```

Helpful commands while developing:

```bash
python manage.py makemigrations --dry-run    # preview without writing files
python manage.py sqlmigrate blog 0001        # show the raw SQL a migration would run
python manage.py showmigrations              # list migrations and which are applied ([X])
```

> The workflow loop is: **change model → `makemigrations` → review the file → `migrate`**. Commit the migration files to git; they are part of your source, exactly like Laravel migrations.

## Core concept — The Django shell: your REPL playground

Before writing views, experiment in the shell. `manage.py shell` boots Python with your project configured (settings loaded, apps ready), so you can import models and run queries.

```bash
python manage.py shell
```

```python
>>> from blog.models import Article
>>> Article.objects.count()
0
```

> Tip: install `django-extensions` and use `python manage.py shell_plus` to auto-import all your models. Vanilla `shell` requires manual imports.

## Core concept — CRUD with the ORM

Every model has a default **manager** named `objects`. It's your gateway to the table. (In Laravel you'd call static methods on the model; in Django you go through `.objects`.)

### Create

Two ways. `create()` builds and saves in one step; `save()` is the lower-level "persist this instance" call.

```python
# One-shot: build + INSERT, returns the saved object (with its new id populated).
article = Article.objects.create(
    title="Hello Django",
    body="My first post.",
    is_published=True,
)
article.id        # => 1   (assigned by the DB on insert)

# Two-step: construct in memory, then persist.
a2 = Article(title="Draft post", body="WIP")
a2.id             # => None   (not saved yet)
a2.save()         # INSERT
a2.id             # => 2
```

`create()` ≈ Laravel's `Article::create([...])` (minus mass-assignment guarding); the two-step `Article(...)` then `.save()` ≈ `new Article; $a->save();`.

### Read

```python
# All rows -> a QuerySet (lazy; no SQL runs until you iterate/evaluate it).
Article.objects.all()
# => <QuerySet [<Article: Hello Django>, <Article: Draft post>]>

# Filter -> QuerySet of matches. Field lookups use the double-underscore syntax.
Article.objects.filter(is_published=True)
Article.objects.filter(title__icontains="hello")     # WHERE title ILIKE '%hello%'
Article.objects.filter(id__gt=1)                      # WHERE id > 1
Article.objects.exclude(is_published=True)            # logical NOT of filter

# get() -> exactly ONE object. Raises if zero or many.
Article.objects.get(id=1)
Article.objects.get(pk=1)                             # same thing via the pk alias
```

`get()` has two important failure modes — know them cold:

```python
from blog.models import Article

try:
    Article.objects.get(id=999)
except Article.DoesNotExist:          # zero matches
    print("not found")

try:
    Article.objects.get(is_published=True)
except Article.MultipleObjectsReturned:   # more than one match
    print("ambiguous")
```

> Rule of thumb: use `get()` only when you expect exactly one row (typically by `pk` or a `unique` field). Use `filter()` for "zero or more". Compare: Laravel's `find()` returns `null` on miss, but Django's `get()` *raises* — that's a deliberate, explicit design choice.

A few more read helpers you'll reach for:

```python
Article.objects.first()                  # first row by ordering, or None
Article.objects.last()                   # last row, or None
Article.objects.count()                  # SELECT COUNT(*)
Article.objects.exists()                 # bool, efficient existence check
Article.objects.filter(is_published=True).order_by("title")   # ORDER BY title
Article.objects.values("id", "title")    # QuerySet of dicts (only those columns)
Article.objects.values_list("id", flat=True)   # [1, 2, ...] flat list of ids
```

### Update

To change one row: mutate attributes, then `save()`.

```python
article = Article.objects.get(pk=1)
article.title = "Hello Django (edited)"
article.save()                 # UPDATE blog_article SET ... WHERE id = 1
```

To change **many** rows efficiently, use `QuerySet.update()` — a single SQL `UPDATE`, no per-row Python loop:

```python
# Publish every draft in one statement. Returns the number of rows affected.
n = Article.objects.filter(is_published=False).update(is_published=True)
n   # => 3
```

> Important difference: `.update()` runs raw SQL and **does not** call `.save()`, fire model signals, or refresh `auto_now` fields. If you need that per-row behavior (e.g. an `updated_at` to tick), loop and `.save()` instead, or set the timestamp explicitly in the `update()` call.

### Delete

```python
# Delete one object.
article = Article.objects.get(pk=2)
article.delete()
# => (1, {'blog.Article': 1})    # (total deleted, breakdown by model)

# Delete many in one query.
Article.objects.filter(is_published=False).delete()

# Delete everything (careful!).
Article.objects.all().delete()
```

### A full CRUD session in the shell

```python
>>> from blog.models import Article
>>> Article.objects.create(title="A", body="aaa", is_published=True)
<Article: A>
>>> Article.objects.create(title="B", body="bbb")
<Article: B>
>>> Article.objects.count()
2
>>> a = Article.objects.get(title="A")
>>> a.title = "A (updated)"
>>> a.save()
>>> Article.objects.filter(is_published=False).update(is_published=True)
1
>>> Article.objects.values_list("title", "is_published")
<QuerySet [('A (updated)', True), ('B', True)]>
>>> Article.objects.filter(title="B").delete()
(1, {'blog.Article': 1})
>>> Article.objects.count()
1
```

## Common mistakes & gotchas

- **`null=True` on a `CharField`/`TextField`.** Avoid it. Strings should use `blank=True` (+ optional `default=""`) so "empty" is always `""`, never a confusing `NULL`. Reserve `null=True` for non-string columns.
- **Confusing `null` and `blank`.** `null` is database-level (can the column hold SQL NULL?); `blank` is validation-level (can a form/serializer leave it empty?). They are not the same and don't imply each other.
- **Forgetting `makemigrations` after editing a model.** The model change has no effect on the DB until you generate **and** apply a migration. Symptoms: "no such column" errors at runtime.
- **Forgetting to add the app to `INSTALLED_APPS`.** Then `makemigrations` silently finds nothing for your app.
- **Using `get()` when more than one row can match.** It raises `MultipleObjectsReturned`. Use `filter().first()` if you only want "any one".
- **Assuming `get()` returns `None` on a miss** (Laravel `find()` habit). It raises `DoesNotExist`. Catch it or use `filter().first()`.
- **`FloatField` for money.** Floats are binary and lose precision (`0.1 + 0.2 != 0.3`). Use `DecimalField(max_digits=..., decimal_places=...)`, which gives you Python `Decimal`.
- **Expecting `QuerySet.update()` to bump `auto_now`.** It won't — `update()` bypasses `save()`. Loop-and-save or set the field explicitly.
- **Editing an already-applied migration by hand.** If it's applied locally and shared, create a *new* migration instead; rewriting history breaks teammates' DB state.
- **Treating a QuerySet as the data.** QuerySets are lazy. `qs = Article.objects.all()` runs no SQL; iterating, slicing with a step, `len()`, `list()`, or `bool()` triggers it. Re-iterating a stored QuerySet may re-query — wrap in `list()` if you need to reuse it.

## Best practices

- Always define `__str__` on every model — your future self (and the admin) will thank you.
- Prefer `TextChoices`/`IntegerChoices` enums for `choices`; never hardcode magic strings around your codebase.
- Use `DecimalField` for currency and any value needing exact arithmetic.
- Add `db_index=True` to columns you frequently filter or sort on; use `Meta.indexes` for multi-column indexes.
- Push integrity rules into the database with `Meta.constraints` (`UniqueConstraint`, `CheckConstraint`) rather than relying only on Python checks.
- Keep migrations in version control and review the generated file before running `migrate` — it's just Python; read it.
- Set `DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"` (the modern default) so large tables don't overflow 32-bit ids.
- Use `.exists()` for existence checks and `.count()` for counting — don't pull rows into Python just to measure them.
- Use bulk operations (`bulk_create`, `QuerySet.update`, `QuerySet.delete`) when touching many rows; per-row `.save()` in a loop is N queries.

## Interview Q&A

**Q: In Django, where do you define a table's schema — a migration or the model?**
**A:** The model. You declare fields on a class subclassing `models.Model`, and Django auto-generates migration files by diffing your models against the previous migration state. This differs from Laravel, where the migration is the authoritative schema and the Eloquent model is just the query API.

**Q: What's the difference between `null` and `blank`?**
**A:** `null` is a database concept — whether the column may store SQL `NULL`. `blank` is a validation concept — whether a form/serializer/admin will accept an empty value. They're independent. For string fields the convention is `blank=True` without `null=True`, so emptiness is always the empty string.

**Q: Why prefer `DecimalField` over `FloatField` for money?**
**A:** Floats use binary representation and can't exactly represent many decimal fractions, causing rounding drift (`0.1 + 0.2 != 0.3`). `DecimalField` maps to a fixed-precision numeric column and gives you Python's `Decimal`, so arithmetic is exact.

**Q: What's the difference between `auto_now` and `auto_now_add`?**
**A:** `auto_now_add=True` sets the value once at insert (like `created_at`) and never changes it. `auto_now=True` overwrites the value with "now" on every `.save()` (like `updated_at`). Neither fires on a bulk `QuerySet.update()`, since that bypasses `save()`.

**Q: What does `Model.objects` refer to?**
**A:** It's the default **manager**, the entry point for queries on that model. Manager methods (`all`, `filter`, `get`, `create`, etc.) return or operate on `QuerySet`s. It's the Django equivalent of Laravel's static query methods on the model, but accessed via the `objects` attribute.

**Q: How do `get()` and `filter()` differ, including failure behavior?**
**A:** `filter()` returns a `QuerySet` of zero-or-more rows. `get()` returns exactly one object and raises `DoesNotExist` if there are none and `MultipleObjectsReturned` if there are several. Use `get()` only when you expect a unique match; otherwise `filter().first()`.

**Q: What's the difference between `obj.save()` and `QuerySet.update()`?**
**A:** `save()` persists a single in-memory instance, running model logic, signals, and `auto_now`. `QuerySet.update()` issues one SQL `UPDATE` across all matched rows without instantiating them, so it's fast for bulk changes but skips `save()`-time behavior (signals, `auto_now`).

**Q: What is the implicit `id` field, and what is `pk`?**
**A:** If you don't declare a primary key, Django adds a `BigAutoField` named `id` (an auto-incrementing 64-bit integer). `pk` is a universal alias that always points to the primary key column, whatever its name — useful for generic code.

**Q: What goes in a model's `Meta` class?**
**A:** Table-level metadata that isn't a column: default `ordering`, custom `db_table`, `verbose_name`/`verbose_name_plural`, database `constraints` (e.g. `UniqueConstraint`, `CheckConstraint`), and `indexes`. It does not define fields.

**Q: Why are QuerySets described as "lazy"?**
**A:** Building a QuerySet doesn't hit the database. The SQL runs only when the QuerySet is evaluated — iterating it, calling `list()`/`len()`/`bool()`, slicing with a step, or accessing results. This lets you chain `filter().exclude().order_by()` cheaply and execute once.

**Q: How do you restrict a field to a fixed set of values?**
**A:** Use `choices`, ideally backed by a `TextChoices`/`IntegerChoices` enum. The stored value is validated against the set, and Django generates a `get_<field>_display()` method returning the human-readable label.

**Q: What's the command sequence to take a new model to a real table?**
**A:** `python manage.py makemigrations` to generate the migration file from the model, then `python manage.py migrate` to apply it (run the SQL). Confirm with `showmigrations` or inspect the SQL with `sqlmigrate`.

## Practice exercises

Try these in a fresh `blog` app and the Django shell before peeking at the solutions.

1. **Define a model.** Create a `Book` model with: `title` (string, max 200), `slug` (unique, slug), `summary` (long text, optional), `price` (money), `in_stock` (boolean defaulting to `True`), `published_on` (date, optional), `created_at`/`updated_at` timestamps. Add a `__str__` returning the title.
2. **Use choices.** Add a `genre` field limited to `FICTION`, `NONFICTION`, `POETRY` using a `TextChoices` enum, defaulting to `FICTION`.
3. **Meta config.** Give `Book` a default ordering of newest-created first and a `UniqueConstraint` on `slug`.
4. **Migrate.** Generate and apply the migrations. Then print the raw SQL Django would run for the initial migration.
5. **CRUD in the shell.** Create three books, fetch one by slug with `get`, update its price, mark all out-of-stock books as in-stock with a single `update()`, then count the books and delete one.
6. **Lookups.** Write queries returning: all in-stock books ordered by price; books whose title contains "python" case-insensitively; just the `(title, price)` pairs as tuples.

<details>
<summary>Solutions</summary>

**1–3) The model** (`blog/models.py`):

```python
from django.db import models


class Book(models.Model):
    class Genre(models.TextChoices):
        FICTION = "fiction", "Fiction"
        NONFICTION = "nonfiction", "Non-fiction"
        POETRY = "poetry", "Poetry"

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    summary = models.TextField(blank=True)
    price = models.DecimalField(max_digits=8, decimal_places=2, default=0)
    in_stock = models.BooleanField(default=True)
    published_on = models.DateField(null=True, blank=True)
    genre = models.CharField(
        max_length=20, choices=Genre.choices, default=Genre.FICTION
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]
        constraints = [
            models.UniqueConstraint(fields=["slug"], name="uniq_book_slug"),
        ]

    def __str__(self) -> str:
        return self.title
```

**4) Migrate and inspect SQL:**

```bash
python manage.py makemigrations blog
python manage.py migrate
python manage.py sqlmigrate blog 0001
# => prints the CREATE TABLE ... statements Django will run
```

**5) CRUD session** (in `python manage.py shell`):

```python
from decimal import Decimal
from blog.models import Book

Book.objects.create(title="Python 101", slug="python-101",
                    price=Decimal("29.99"))
Book.objects.create(title="Django Deep Dive", slug="django-deep-dive",
                    price=Decimal("39.50"), in_stock=False)
Book.objects.create(title="REST APIs", slug="rest-apis",
                    price=Decimal("19.00"), in_stock=False)

b = Book.objects.get(slug="python-101")     # fetch one by unique field
b.price = Decimal("24.99")
b.save()

Book.objects.filter(in_stock=False).update(in_stock=True)   # bulk update -> 2

Book.objects.count()                         # => 3
Book.objects.get(slug="rest-apis").delete()  # => (1, {'blog.Book': 1})
Book.objects.count()                         # => 2
```

**6) Lookups:**

```python
from blog.models import Book

# All in-stock books, cheapest first.
Book.objects.filter(in_stock=True).order_by("price")

# Title contains "python", case-insensitive.
Book.objects.filter(title__icontains="python")

# (title, price) tuples only.
Book.objects.values_list("title", "price")
# => <QuerySet [('Python 101', Decimal('24.99')), ...]>
```

</details>

## Cheat sheet

```python
# --- Define ---
class Article(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField(blank=True)
    price = models.DecimalField(max_digits=8, decimal_places=2, default=0)
    status = models.CharField(max_length=20, choices=Status.choices, default="draft")
    created_at = models.DateTimeField(auto_now_add=True)   # set on INSERT
    updated_at = models.DateTimeField(auto_now=True)       # set on every save()

    class Meta:
        ordering = ["-created_at"]
        db_table = "articles"
        constraints = [models.UniqueConstraint(fields=["slug"], name="uniq_slug")]

    def __str__(self): return self.title
```

| Need | Code |
|---|---|
| Generate migration | `python manage.py makemigrations` |
| Apply migration | `python manage.py migrate` |
| Show SQL | `python manage.py sqlmigrate app 0001` |
| Open REPL | `python manage.py shell` |
| Create (1 step) | `M.objects.create(**kwargs)` |
| Create (2 step) | `m = M(**kwargs); m.save()` |
| All rows | `M.objects.all()` |
| Filter | `M.objects.filter(field=v)` / `field__icontains="x"` / `field__gt=1` |
| Exclude | `M.objects.exclude(field=v)` |
| One row | `M.objects.get(pk=1)` → raises `DoesNotExist` / `MultipleObjectsReturned` |
| First / exists / count | `.first()` · `.exists()` · `.count()` |
| Update one | `m.field = v; m.save()` |
| Update many | `M.objects.filter(...).update(field=v)` |
| Delete one / many | `m.delete()` · `M.objects.filter(...).delete()` |
| Order | `M.objects.order_by("-field")` |
| Dicts / tuples | `.values("a","b")` · `.values_list("a", flat=True)` |

**`null` vs `blank`:** `null` = DB allows NULL · `blank` = validation allows empty. Strings → `blank=True` (not `null`). Non-strings that can be absent → `null=True, blank=True`.

**Money → `DecimalField`. Never `FloatField`.**

**`get()` raises on miss; `filter().first()` returns `None`.**
