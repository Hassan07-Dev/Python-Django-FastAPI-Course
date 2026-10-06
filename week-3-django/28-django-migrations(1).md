# 28 — Migrations

> Week 3 · Django ORM & Schema Management

## What you will learn

- What migrations are and why Django (and Laravel) need them — *version control for your database schema*.
- The two-step workflow: `makemigrations` (detect model changes) vs `migrate` (apply them to the DB).
- The anatomy of a migration file: `dependencies` and `operations`.
- How the **migration graph** links migrations together — within an app and across apps.
- Inspecting state with `showmigrations`, and how to **reverse** (roll back) a migration.
- **Data migrations** with `RunPython` (and why you should always write the reverse function).
- The classic problem: **adding a non-null column to a table that already has rows**, and the multi-step pattern to do it safely.
- `squashmigrations` to collapse a long history into one file.
- The escape hatches: `--fake` and `--fake-initial`, and when they save you.
- Common pitfalls (conflicting migrations from two branches, editing an already-applied migration) and how to recover.

## PHP/Laravel bridge

You already know this concept — Laravel calls it migrations too. In Laravel you *hand-write* a migration class with `up()` and `down()` methods using the `Schema` builder, then run `php artisan migrate`. Django's twist: you almost never hand-write the schema. You edit your **models** (the equivalent of defining your tables in code), and `python manage.py makemigrations` *diffs* your models against the last known schema and **auto-generates** the migration for you. Then `python manage.py migrate` applies it — the direct analogue of `php artisan migrate`. The mental shift is: in Laravel the migration is the source of truth for schema; in Django the **model** is the source of truth and the migration is a generated, replayable record of changes. Rolling back (`migrate app_name 0001`) is Django's `php artisan migrate:rollback`.

| Laravel | Django |
|---|---|
| Hand-write `Schema::create(...)` in `up()` | Edit your model, then `makemigrations` generates it |
| `php artisan make:migration` | `python manage.py makemigrations` |
| `php artisan migrate` | `python manage.py migrate` |
| `php artisan migrate:rollback` | `python manage.py migrate app 0005` (migrate *to* an earlier point) |
| `php artisan migrate:status` | `python manage.py showmigrations` |
| `down()` method | `reverse_code` / auto-reversible operations |
| `migrations` table | `django_migrations` table |

## Core concept: what a migration actually is

A migration is a Python file that describes a **change to your database schema** (create a table, add a column, add an index, etc.) in a database-agnostic way. Django keeps a record of which migrations have run in a table called `django_migrations`. When you run `migrate`, Django compares "what's recorded as applied" with "what migration files exist on disk" and runs whatever is missing, in dependency order.

Why not just run raw `ALTER TABLE` SQL? Because:

- **Version control.** Migrations live in your repo next to your code. A teammate pulls your branch, runs `migrate`, and their DB matches yours. No more "works on my machine because I forgot to tell you about the column I added."
- **Reproducibility.** You can rebuild the entire schema from scratch by replaying migrations from `0001` forward — useful for new dev machines, CI, and fresh deploys.
- **Reversibility.** Most operations can be undone, so you can roll back a bad change.
- **Database portability.** The same migration runs on SQLite (dev), PostgreSQL (prod), MySQL, etc. Django translates to the right SQL dialect.

### The two commands you will type constantly

```bash
# 1. You changed a model. Generate the migration file(s) that capture the change.
python manage.py makemigrations

# 2. Apply any unapplied migrations to the database.
python manage.py migrate
```

`makemigrations` **writes files**; it does **not** touch your database. `migrate` **reads files** and **changes your database**. Keep this distinction crystal clear — it is the single most common point of confusion for beginners.

```bash
# Generate migrations for just one app
python manage.py makemigrations blog

# See the SQL that a migration would run, without running it
python manage.py sqlmigrate blog 0002

# Apply migrations for only one app, up to a specific number
python manage.py migrate blog 0003
```

## Core concept: a worked example

Let's say we have a `blog` app with this model.

```python
# blog/models.py
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self) -> str:
        return self.title
```

Generate the first migration:

```bash
python manage.py makemigrations blog
# => Migrations for 'blog':
# =>   blog/migrations/0001_initial.py
# =>     + Create model Post
```

Apply it:

```bash
python manage.py migrate blog
# => Operations to perform:
# =>   Apply all migrations: blog
# => Running migrations:
# =>   Applying blog.0001_initial... OK
```

Now add a field to the model:

```python
# blog/models.py (changed)
class Post(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    slug = models.SlugField(max_length=220, unique=True, default="")  # <-- new
    created_at = models.DateTimeField(auto_now_add=True)
```

```bash
python manage.py makemigrations blog
# => Migrations for 'blog':
# =>   blog/migrations/0002_post_slug.py
# =>     + Add field slug to post

python manage.py migrate blog
# =>   Applying blog.0002_post_slug... OK
```

That's the entire daily loop: **edit model → makemigrations → migrate**.

## Core concept: anatomy of a migration file

Open `blog/migrations/0001_initial.py` and you'll see something like this.

```python
# blog/migrations/0001_initial.py
from django.db import migrations, models


class Migration(migrations.Migration):

    initial = True

    dependencies = [
        # This migration depends on nothing (it's the first one for this app).
    ]

    operations = [
        migrations.CreateModel(
            name="Post",
            fields=[
                ("id", models.BigAutoField(
                    auto_created=True, primary_key=True,
                    serialize=False, verbose_name="ID",
                )),
                ("title", models.CharField(max_length=200)),
                ("body", models.TextField()),
                ("created_at", models.DateTimeField(auto_now_add=True)),
            ],
        ),
    ]
```

The two parts that matter:

- **`dependencies`** — a list of `(app_label, migration_name)` tuples that must run *before* this one. This is how Django knows the order. `0002` will depend on `0001`, and a migration that adds a foreign key to another app will depend on *that* app's relevant migration.
- **`operations`** — an ordered list of schema (or data) changes. `CreateModel`, `AddField`, `AlterField`, `RemoveField`, `AddIndex`, `RunPython`, `RunSQL`, and many more.

Here's what `0002_post_slug.py` looks like — notice the dependency pointing back to `0001`:

```python
# blog/migrations/0002_post_slug.py
from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ("blog", "0001_initial"),   # must run after the initial migration
    ]

    operations = [
        migrations.AddField(
            model_name="post",
            name="slug",
            field=models.SlugField(default="", max_length=220, unique=True),
        ),
    ]
```

> Tip: run `python manage.py sqlmigrate blog 0001` to see the exact `CREATE TABLE ...` SQL Django will emit. Great for reviewing what a migration *actually* does before trusting it in production.

## Core concept: the migration graph and cross-app dependencies

Django doesn't think of migrations as a simple per-app list. It builds a **directed acyclic graph (DAG)** of every migration across every installed app, linked by `dependencies`. `migrate` does a topological sort of that graph and applies migrations in a valid order.

Within one app, the chain is linear and obvious: `0001 → 0002 → 0003`. The interesting part is **cross-app dependencies**. Suppose `blog.Post` gets an `author` foreign key to `accounts.User`:

```python
# blog/models.py
class Post(models.Model):
    author = models.ForeignKey(
        "accounts.User", on_delete=models.CASCADE, related_name="posts"
    )
    title = models.CharField(max_length=200)
    # ...
```

The generated migration will declare a dependency on the `accounts` app so the `User` table exists *before* the foreign key is created:

```python
class Migration(migrations.Migration):

    dependencies = [
        ("blog", "0002_post_slug"),
        ("accounts", "0001_initial"),   # cross-app: User must exist first
    ]

    operations = [
        migrations.AddField(
            model_name="post",
            name="author",
            field=models.ForeignKey(
                on_delete=django.db.models.deletion.CASCADE,
                related_name="posts",
                to="accounts.user",
            ),
        ),
    ]
```

There's also `run_before` (the inverse of `dependencies`) for forcing a migration to run *before* some other one — rarely needed, but good to know it exists. This graph is why you can have many apps each with their own migration folder and Django still applies everything in a globally consistent order.

## Core concept: inspecting and reversing migrations

### showmigrations — what's applied?

```bash
python manage.py showmigrations
# => accounts
# =>  [X] 0001_initial
# => blog
# =>  [X] 0001_initial
# =>  [X] 0002_post_slug
# =>  [ ] 0003_post_author      <-- the empty brackets mean NOT applied yet
```

`[X]` = applied, `[ ]` = pending. Add `--plan` to see the exact order Django would apply them in:

```bash
python manage.py showmigrations --plan
```

### Reversing (rolling back)

You roll back by migrating *to* an earlier migration. Django runs the **reverse** of every migration between "now" and your target, newest first.

```bash
# Currently at blog 0003. Roll back to the state after 0002:
python manage.py migrate blog 0002
# =>   Unapplying blog.0003_post_author... OK

# Undo EVERYTHING for an app (drop its tables):
python manage.py migrate blog zero
```

Most schema operations are automatically reversible — `AddField` reverses to `RemoveField`, `CreateModel` to `DeleteModel`, and so on. Data migrations are reversible only if *you* wrote a reverse function (covered next). If an operation can't be reversed, Django raises `IrreversibleError`.

> Caution: rolling back a migration that **drops data** (e.g. removing a column) will not magically bring the data back. Reversing recreates the *column*, but the values that were in it are gone. Always back up production before reversing destructive changes.

## Core concept: data migrations with RunPython

Schema migrations change *structure*. **Data migrations** change *data* — backfilling a new column, normalizing values, populating a lookup table. You create an empty migration and add a `RunPython` operation.

```bash
python manage.py makemigrations blog --empty --name backfill_slugs
# => blog/migrations/0004_backfill_slugs.py
```

```python
# blog/migrations/0004_backfill_slugs.py
from django.db import migrations
from django.utils.text import slugify


def populate_slugs(apps, schema_editor):
    # IMPORTANT: use apps.get_model, NOT a direct import of Post.
    Post = apps.get_model("blog", "Post")
    for post in Post.objects.filter(slug=""):
        post.slug = slugify(post.title)
        post.save(update_fields=["slug"])


def clear_slugs(apps, schema_editor):
    Post = apps.get_model("blog", "Post")
    Post.objects.update(slug="")


class Migration(migrations.Migration):

    dependencies = [
        ("blog", "0002_post_slug"),
    ]

    operations = [
        # Pass BOTH directions so the migration is reversible.
        migrations.RunPython(populate_slugs, reverse_code=clear_slugs),
    ]
```

Three rules that trip up everyone:

1. **Always use `apps.get_model("app", "Model")`** instead of `from blog.models import Post`. The `apps` registry gives you the *historical* version of the model as it existed at that point in migration history. Importing the live model can break later when the real model has fields that didn't exist yet (or has been deleted).
2. **Always pass `reverse_code`.** If you can't meaningfully reverse it, pass `migrations.RunPython.noop` so the migration is still reversible (it just does nothing on the way back):
   ```python
   migrations.RunPython(populate_slugs, reverse_code=migrations.RunPython.noop)
   ```
3. **Historical models have no custom methods or signals** wired up. `apps.get_model` returns a bare model with fields and managers only — your `save()` override or `@property` methods won't be there. Write the migration to be self-contained.

For raw SQL when the ORM isn't enough, there's `RunSQL`:

```python
migrations.RunSQL(
    sql="UPDATE blog_post SET slug = lower(slug);",
    reverse_sql=migrations.RunSQL.noop,
)
```

## Core concept: adding a NON-NULL field to a table with existing rows

This is the single most common real-world migration headache, and a favorite interview question. If a table already has rows and you add a column that is `NOT NULL` with no default, the database has no value to put in the existing rows — the migration would fail.

When you run `makemigrations` and Django detects this, it **interactively prompts** you:

```bash
python manage.py makemigrations blog
# => It is impossible to add a non-nullable field 'published_at' to post without
# => specifying a default. This is because the database needs something to populate
# => existing rows.
# => Please select a fix:
# =>  1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
# =>  2) Quit and manually define a default value in models.py.
# => Select an option:
```

You have three good strategies, depending on the situation.

### Strategy A — provide a default

Simplest. Give the field a `default` so existing rows get filled in.

```python
from django.utils import timezone

class Post(models.Model):
    # ...
    published_at = models.DateTimeField(default=timezone.now)
```

The migration applies cleanly; every existing row gets the default value.

### Strategy B — allow null

If "no value" is a legitimate state, just make it nullable. No backfill needed.

```python
published_at = models.DateTimeField(null=True, blank=True)
```

### Strategy C — the safe three-step pattern (add nullable → backfill → enforce NOT NULL)

When you want a column that is ultimately `NOT NULL` but there's no single sensible default, do it in three migrations so you never block on existing data:

```python
# Step 1: add the field as nullable
published_at = models.DateTimeField(null=True, blank=True)
```
```bash
python manage.py makemigrations blog   # 0005_add_published_at_nullable
python manage.py migrate blog
```

```python
# Step 2: a data migration to backfill (RunPython)
def backfill_published_at(apps, schema_editor):
    Post = apps.get_model("blog", "Post")
    Post.objects.filter(published_at__isnull=True).update(
        published_at=models.F("created_at")
    )
```
```bash
python manage.py makemigrations blog --empty --name backfill_published_at
# (paste the RunPython operation in, then:)
python manage.py migrate blog
```

```python
# Step 3: flip it to NOT NULL now that every row has a value
published_at = models.DateTimeField(null=False)
```
```bash
python manage.py makemigrations blog   # 0007_alter_published_at_notnull
python manage.py migrate blog
```

This three-step dance also matters for **zero-downtime deploys**: a single "add NOT NULL with default" can lock a large table while every row is rewritten. Splitting it lets you control the backfill (batched, off-peak) and keeps each migration fast.

## Core concept: squashmigrations

Over months, an app accumulates dozens of migrations. They still replay correctly, but they're slow to run on a fresh DB and noisy to read. `squashmigrations` collapses a *range* of them into a single optimized migration.

```bash
# Squash blog migrations 0001 through 0007 into one
python manage.py squashmigrations blog 0001 0007
# => Will squash the following migrations:
# =>  - 0001_initial
# =>  - 0002_post_slug ...
# => Created new squashed migration blog/migrations/0001_squashed_0007_....py
```

How it works and how to finish the job safely:

- The new file is named `0001_squashed_0007_...` and contains `replaces = [...]` listing the migrations it stands in for. Django treats it as already-applied on any DB where those old migrations ran, and as a fresh single migration on a brand-new DB.
- **Don't delete the old files immediately.** Deploy the squashed migration everywhere first so every environment has recorded the old ones as replaced. *Then*, in a later release, delete the old files and remove the `replaces` attribute (turning the squashed migration into a normal one).
- Data migrations with `RunPython` are **not** automatically optimized away — Django keeps them. Review the squashed output; you may need to hand-edit.

## Core concept: --fake and --fake-initial

These flags record migration state **without running the SQL**. Powerful and dangerous — use them deliberately.

### `--fake`

Marks a migration as applied (or unapplied) in `django_migrations` **without touching the schema**. Use it when the database is *already* in the target state but Django doesn't know it.

```bash
# DB already has the column (you added it manually / restored a dump).
# Tell Django "consider 0005 applied" without re-running it:
python manage.py migrate blog 0005 --fake
```

Common real scenario: you ran a migration on production by hand, or you restored a snapshot that's already at a newer schema, and now `migrate` wants to re-run things that would fail. `--fake` reconciles the bookkeeping.

### `--fake-initial`

Specifically for **initial** migrations against a database whose tables **already exist** (e.g. you're adopting Django on a legacy DB, or you introspected an existing schema). It tells Django: "if the tables this initial migration would create already exist, just mark it applied instead of failing."

```bash
python manage.py migrate --fake-initial
```

> Warning: `--fake` lies to Django about reality. If you fake something whose schema is *not* actually in the target state, every later migration is now built on a false premise and will likely break. Only fake when you are certain the DB already matches.

## Common mistakes & gotchas

- **Forgetting to run `makemigrations` after editing a model.** Your code uses the new field but the DB doesn't have it → `OperationalError: no such column`. Run `makemigrations` *and* `migrate`.
- **Running `makemigrations` but forgetting `migrate`.** The file exists; the DB is unchanged. `showmigrations` shows `[ ]`.
- **Not committing migration files to git.** Migrations are source code. If you `.gitignore` them, teammates and CI can't reproduce your schema. Commit them.
- **Editing a migration that has already been applied** (anywhere — including a colleague's machine or prod). Django tracks migrations by name; changing the *contents* of an applied migration does **not** re-run it, so environments silently diverge. Make a *new* migration instead.
- **Importing the live model in a `RunPython` function** (`from blog.models import Post`). Use `apps.get_model(...)` so you get the historical schema.
- **Conflicting migrations from two branches.** Both branches branch off `0007` and each adds `0008_*`. After merging you have two leaf nodes and Django complains:
  ```bash
  python manage.py migrate
  # => CommandError: Conflicting migrations detected; multiple leaf nodes
  # =>   in the migration graph: (0008_a, 0008_b in blog).
  ```
  Fix it with:
  ```bash
  python manage.py makemigrations --merge
  # => creates 0009_merge_*.py that depends on both 0008_a and 0008_b
  ```
- **Renaming a field and getting a remove+add instead of a rename.** Django sometimes can't tell a rename from a delete-then-create. When prompted "Did you rename post.headline to post.title?" answer **yes** — otherwise you'll *drop the old column and lose its data*.
- **Reversing a destructive migration and expecting data back.** Reversing recreates structure, not deleted values.
- **`auto_now_add` / `auto_now` confusion in defaults.** When prompted for a one-off default on a datetime, picking `timezone.now()` once sets the *same* timestamp on all existing rows — usually fine, but know what you're choosing.

## Best practices

- **One logical change per migration** where practical — easier to review, reason about, and reverse.
- **Always commit migration files** alongside the model change in the same commit/PR.
- **Always write `reverse_code`** for data migrations (use `RunPython.noop` if reversal is a no-op). Future-you wants the rollback.
- **Use `apps.get_model`** in every data migration. Never import live models.
- **Review generated SQL with `sqlmigrate`** before applying anything important in production.
- **Separate schema and data migrations** when possible, so a schema rollback doesn't drag data logic with it.
- **For non-null columns on big tables, use the add-nullable → backfill → enforce pattern** to avoid long locks and failed migrations.
- **Run `makemigrations --check --dry-run` in CI** to fail the build if someone changed a model without generating a migration:
  ```bash
  python manage.py makemigrations --check --dry-run
  # exits non-zero if there are model changes with no migration
  ```
- **Squash periodically** once a release is stable, then clean up old files in a *later* release.
- **Name your migrations** with `--name` so the history is readable (`0004_backfill_slugs` beats `0004_auto_20260619_1042`).

## Interview Q&A

**Q: What is the difference between `makemigrations` and `migrate`?**
**A:** `makemigrations` inspects your models, diffs them against the current migration state, and *writes* migration files describing the changes — it does not touch the database. `migrate` *reads* the migration files and applies the unapplied ones to the database, updating the `django_migrations` table. Generate first, then apply.

**Q: How does Django know which migrations have already run?**
**A:** It records every applied migration as a row (app label + migration name) in the `django_migrations` table. On `migrate`, it compares that table against the files on disk and runs whatever's missing, in dependency order.

**Q: What are the two key parts of a migration file?**
**A:** `dependencies` — a list of `(app, migration)` tuples that must run before this one, which defines ordering — and `operations` — the ordered list of changes to perform (`CreateModel`, `AddField`, `RunPython`, etc.).

**Q: How do you roll back a migration?**
**A:** Migrate *to* an earlier migration: `python manage.py migrate app_label 0005` reverses everything after `0005`. `migrate app_label zero` unapplies all of an app's migrations. Schema ops auto-reverse; data migrations reverse only if you supplied `reverse_code`. Note that reversing recreates structure but does not restore deleted data.

**Q: You need to add a `NOT NULL` column to a table that already has millions of rows. How?**
**A:** The DB needs a value for existing rows. Either provide a `default` (Django backfills all rows), or — for large tables / zero-downtime — use the three-step pattern: (1) add the column nullable, (2) a `RunPython` data migration to backfill in batches, (3) a migration to flip it to `NOT NULL`. This avoids a single long table-rewriting lock.

**Q: Why must you use `apps.get_model()` in a data migration instead of importing the model?**
**A:** Because `apps.get_model` returns the *historical* version of the model — its schema as it existed at that point in migration history. A direct import gives you the *current* model, which may reference columns that don't exist yet at that migration, breaking when you replay history on a fresh DB.

**Q: What does `squashmigrations` do?**
**A:** It collapses a contiguous range of an app's migrations into a single optimized migration that `replaces` them. New databases run the one squashed file; existing databases recognize the old ones as already replaced. You deploy the squashed migration everywhere before deleting the originals in a later release.

**Q: What's the difference between `--fake` and `--fake-initial`?**
**A:** `--fake` records a migration as (un)applied without running its SQL — used when the DB already matches the target state. `--fake-initial` is narrower: when applying an *initial* migration, if the tables it would create already exist, it marks the migration applied instead of erroring — used when adopting Django on a pre-existing schema.

**Q: Two teammates merged branches and now `migrate` says "conflicting migrations / multiple leaf nodes." What happened and how do you fix it?**
**A:** Both branches added a sibling migration (e.g. `0008_a` and `0008_b`) off the same parent, leaving two leaves in the graph. Run `python manage.py makemigrations --merge` to generate a merge migration that depends on both, unifying the graph. Then `migrate`.

**Q: Is it safe to edit a migration that has already been applied?**
**A:** No. Django identifies migrations by name and won't re-run an already-applied one, so editing its contents makes environments silently diverge (your DB has the old version, a teammate's the new). Create a *new* migration for further changes instead.

**Q: How would you backfill or transform existing data as part of a migration?**
**A:** Create an empty migration (`makemigrations --empty`) and add a `RunPython` operation pointing to a function that takes `(apps, schema_editor)`, uses `apps.get_model` to fetch the historical model, and updates rows. Provide a `reverse_code` function (or `RunPython.noop`) so it stays reversible.

**Q: How do migrations differ between Django and Laravel?**
**A:** In Laravel you hand-write the schema in each migration's `up()`/`down()`; the migration is the source of truth. In Django the *model* is the source of truth and `makemigrations` auto-generates the migration by diffing your models. Both record applied migrations in a DB table and support rollback, but Django's auto-generation, cross-app dependency graph, and `apps.get_model` historical models are the big conceptual differences.

## Practice exercises

Try these against a fresh Django project (`django-admin startproject shop && cd shop && python manage.py startapp catalog`, then add `catalog` to `INSTALLED_APPS`). Attempt each before peeking at the solution.

1. **Initial schema.** Create a `Product` model with `name` (CharField, max 120), `price` (DecimalField, 8 digits / 2 decimals), and `created_at` (auto DateTimeField). Generate and apply the initial migration. Then show the generated `CREATE TABLE` SQL.
2. **Add a field with a default.** Add an `in_stock` BooleanField defaulting to `True`. Generate and apply the migration. Inspect the migration file's `operations`.
3. **Data migration.** Add a `slug` SlugField (allow blank for now), then write a data migration that backfills `slug` from `name` using `slugify`, with a working reverse function.
4. **Non-null on existing rows.** You now want `slug` to be `unique=True, NOT NULL`. Using the three-step pattern, get there without breaking on existing rows.
5. **Reverse and inspect.** Roll the `catalog` app back to its `0002` state, confirm with `showmigrations`, then re-apply forward.
6. **Squash.** Squash `catalog` migrations `0001` through your latest into a single migration and verify it still applies on a fresh database (e.g. delete `db.sqlite3` and `migrate`).

<details>
<summary>Solutions</summary>

**1. Initial schema**

```python
# catalog/models.py
from django.db import models


class Product(models.Model):
    name = models.CharField(max_length=120)
    price = models.DecimalField(max_digits=8, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self) -> str:
        return self.name
```
```bash
python manage.py makemigrations catalog   # => 0001_initial
python manage.py migrate catalog
python manage.py sqlmigrate catalog 0001  # prints the CREATE TABLE statement
```

**2. Add a field with a default**

```python
class Product(models.Model):
    # ...
    in_stock = models.BooleanField(default=True)
```
```bash
python manage.py makemigrations catalog   # => 0002_product_in_stock
python manage.py migrate catalog
```
The operation will be `migrations.AddField(model_name="product", name="in_stock", field=models.BooleanField(default=True))`. Because there's a default, existing rows are backfilled with `True` and no prompt appears.

**3. Data migration**

```python
# model: add a blank-allowed slug first
slug = models.SlugField(max_length=140, blank=True, default="")
```
```bash
python manage.py makemigrations catalog                       # => 0003_product_slug
python manage.py makemigrations catalog --empty --name backfill_slugs  # => 0004
```
```python
# catalog/migrations/0004_backfill_slugs.py
from django.db import migrations
from django.utils.text import slugify


def fill(apps, schema_editor):
    Product = apps.get_model("catalog", "Product")
    for p in Product.objects.filter(slug=""):
        p.slug = slugify(p.name)
        p.save(update_fields=["slug"])


def unfill(apps, schema_editor):
    apps.get_model("catalog", "Product").objects.update(slug="")


class Migration(migrations.Migration):
    dependencies = [("catalog", "0003_product_slug")]
    operations = [migrations.RunPython(fill, reverse_code=unfill)]
```
```bash
python manage.py migrate catalog
```

**4. Non-null on existing rows (three-step)**

Step 1 (nullable) and step 2 (backfill) are effectively done in exercise 3 — every row now has a non-empty slug. Step 3 is to enforce uniqueness + NOT NULL:

```python
slug = models.SlugField(max_length=140, unique=True)   # null=False is the default
```
```bash
python manage.py makemigrations catalog   # => 0005_alter_product_slug
python manage.py migrate catalog
```
If two products produced the same slug, the `unique=True` migration will fail — that's the data telling you to dedupe first (e.g. append the id). That failure is exactly why the multi-step approach is safer than a one-shot change.

**5. Reverse and inspect**

```bash
python manage.py migrate catalog 0002
python manage.py showmigrations catalog
# => catalog
# =>  [X] 0001_initial
# =>  [X] 0002_product_in_stock
# =>  [ ] 0003_product_slug
# =>  [ ] 0004_backfill_slugs
# =>  [ ] 0005_alter_product_slug
python manage.py migrate catalog            # re-apply forward to latest
```

**6. Squash**

```bash
python manage.py squashmigrations catalog 0001 0005
# => Created 0001_squashed_0005_....py
# Verify on a clean DB:
rm db.sqlite3
python manage.py migrate
# everything applies from the squashed migration with no errors
```
After confirming every environment has the squashed migration recorded, delete the old `0001`–`0005` files and remove `replaces` in a later release.

</details>

## Cheat sheet

```bash
# Generate / apply
python manage.py makemigrations [app]          # detect model changes -> write files
python manage.py migrate [app] [target]        # apply (or reverse to target)
python manage.py makemigrations app --empty --name x   # blank migration (data migration)

# Inspect
python manage.py showmigrations [app]          # [X] applied / [ ] pending
python manage.py showmigrations --plan         # exact apply order
python manage.py sqlmigrate app 0002           # show the SQL, don't run it
python manage.py makemigrations --check --dry-run   # CI: fail if models changed w/o migration

# Reverse
python manage.py migrate app 0005              # roll back to state after 0005
python manage.py migrate app zero              # unapply ALL of app's migrations

# Conflicts / maintenance
python manage.py makemigrations --merge        # resolve "multiple leaf nodes"
python manage.py squashmigrations app 0001 0009  # collapse a range into one

# Fakes (use with care — they don't run SQL)
python manage.py migrate app 0005 --fake       # mark applied, DB already matches
python manage.py migrate --fake-initial        # initial migration, tables already exist
```

| Concept | Command / API |
|---|---|
| Source of truth | your **model** (Django auto-generates the migration) |
| Where applied state lives | `django_migrations` table |
| Migration file parts | `dependencies` (ordering) + `operations` (changes) |
| Data migration op | `migrations.RunPython(forward, reverse_code=backward)` |
| Historical model in data migration | `apps.get_model("app", "Model")` (never import live) |
| Reversible no-op | `migrations.RunPython.noop` / `migrations.RunSQL.noop` |
| Add NOT NULL to populated table | default, OR null→backfill→enforce (3 steps) |
| Merge branch conflict | `makemigrations --merge` |
| Collapse history | `squashmigrations app start end` (keep `replaces`, delete old files later) |
| Tell Django "already done" | `--fake` / `--fake-initial` |
| Laravel analogue | models→`makemigrations`→`migrate` ≈ `Schema` builder→`artisan migrate` |
