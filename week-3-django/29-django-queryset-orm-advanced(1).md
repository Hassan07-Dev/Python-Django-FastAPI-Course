# 29 — QuerySets & Advanced ORM

> Week 3 · Django ORM

## What you will learn

- Why a QuerySet is **lazy** and the exact moments a query hits the database
- `filter()`, `exclude()`, `get()` and how chaining builds one SQL statement
- The full field-lookup vocabulary (`exact`, `icontains`, `gte`, `in`, `range`, `isnull`, date lookups, …)
- Complex `OR` / `AND` / `NOT` conditions with `Q` objects
- `F()` expressions for atomic updates and field-to-field comparisons
- Aggregation: `annotate()` vs `aggregate()` with `Count`, `Sum`, `Avg`, `Max`, `Min`
- Reading only the columns you need with `values()` / `values_list()` / `only()` / `defer()`
- `order_by`, `distinct`, `exists`, `count`, slicing, and limits
- The **N+1 problem** and the two cures: `select_related` (joins) vs `prefetch_related` (extra queries)
- Bulk operations (`bulk_create`, `bulk_update`, `.update()`) and `transaction.atomic`
- Inspecting the generated SQL, and when to drop down to `raw()`

## PHP/Laravel bridge

Django's QuerySet is the close cousin of Laravel's **Eloquent query builder**. `Model.objects.filter(...)` is `Model::where(...)`, and just like Eloquent, **nothing runs until you iterate** the result. Django's `select_related`/`prefetch_related` are exactly Eloquent's `with()` eager loading that you reach for to kill the N+1 problem. The big mental shift: in Eloquent you call `->get()` / `->first()` to execute; in Django a QuerySet executes implicitly the moment you loop, slice with a step, call `list()`, or print it. `F()` is Laravel's `DB::raw('price + 1')` / `increment()`, and `Q()` is the nested `where(function ($q) { ... })` closure for `orWhere` groups.

## Core concept: QuerySets are lazy

A QuerySet is a *description* of a database query, not the result. Building and chaining it touches the DB **zero times**. The query is sent only when the QuerySet is **evaluated**.

Assume these models throughout the lesson:

```python
# shop/models.py
from django.db import models


class Author(models.Model):
    name = models.CharField(max_length=120)
    country = models.CharField(max_length=2, default="US")

    def __str__(self):
        return self.name


class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")
    price = models.DecimalField(max_digits=8, decimal_places=2)
    stock = models.PositiveIntegerField(default=0)
    published = models.DateField(null=True, blank=True)
    tags = models.ManyToManyField("Tag", related_name="books", blank=True)

    def __str__(self):
        return self.title


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):
        return self.name
```

Now watch *when* the database is actually contacted:

```python
qs = Book.objects.filter(price__gt=10)   # 0 queries — just a description
qs = qs.exclude(stock=0)                  # still 0 — chaining is free
qs = qs.order_by("title")                 # still 0

for book in qs:                           # ← 1 query fires HERE
    print(book.title)

# Re-iterating uses the cached result, NOT a new query:
len(qs)                                   # 0 extra queries — list is cached
```

A QuerySet is evaluated when you:

| Action | Example |
|---|---|
| Iterate it | `for b in qs:` |
| Call `list()` | `list(qs)` |
| Slice **with a step** | `qs[::2]` |
| Call `len()`, `bool()`, `repr()` | `len(qs)`, `if qs:`, printing in a shell |
| Pickle / cache it | `pickle.dumps(qs)` |

> **Why lazy matters:** you can pass a QuerySet around, add filters in different layers (view, serializer, manager method), and it still compiles to **one** SQL statement. No intermediate round-trips.

The cache is per-QuerySet-object. `Book.objects.all()` written twice is two different objects, so two queries. Reuse the **variable** to reuse the cache.

## Core concept: filter, exclude, get, and chaining

```python
# WHERE price > 10  AND  stock > 0
Book.objects.filter(price__gt=10).filter(stock__gt=0)
# Equivalent single call (same SQL):
Book.objects.filter(price__gt=10, stock__gt=0)

# WHERE NOT (stock = 0)
Book.objects.exclude(stock=0)

# Returns ONE object — or raises
book = Book.objects.get(pk=1)
```

`get()` is strict and that is the point:

- Zero matches → `Book.DoesNotExist`
- More than one → `Book.MultipleObjectsReturned`

```python
from django.shortcuts import get_object_or_404

# In a view, the idiomatic safe version:
book = get_object_or_404(Book, pk=request_pk)  # 404 instead of 500
```

Use `get()` for a unique row you expect to exist; use `filter().first()` when "maybe one" is fine (`first()` returns `None` on no match instead of raising).

## Core concept: field lookups

Lookups are the double-underscore suffix on a field name. They are Django's WHERE-clause operators.

```python
Book.objects.filter(title__exact="Dune")        # title = 'Dune'  (case-sensitive)
Book.objects.filter(title__iexact="dune")        # case-insensitive equals
Book.objects.filter(title__contains="Pyth")      # LIKE '%Pyth%'
Book.objects.filter(title__icontains="pyth")     # case-insensitive LIKE
Book.objects.filter(title__startswith="The ")    # LIKE 'The %'
Book.objects.filter(title__endswith="!")
Book.objects.filter(price__gt=10)                # >
Book.objects.filter(price__gte=10)               # >=
Book.objects.filter(price__lt=50)                # <
Book.objects.filter(price__lte=50)               # <=
Book.objects.filter(stock__in=[0, 1, 2])         # IN (0, 1, 2)
Book.objects.filter(price__range=(10, 50))       # BETWEEN 10 AND 50 (inclusive)
Book.objects.filter(published__isnull=True)      # IS NULL
Book.objects.filter(published__isnull=False)     # IS NOT NULL
```

**Spanning relationships** — follow a ForeignKey with `__`:

```python
# Books whose author is from Pakistan
Book.objects.filter(author__country="PK")
# Two levels deep is fine too: book.author.name contains "King"
Book.objects.filter(author__name__icontains="king")
```

**Date / datetime lookups** decompose a date field:

```python
Book.objects.filter(published__year=2024)
Book.objects.filter(published__month=12)
Book.objects.filter(published__day=25)
Book.objects.filter(published__week_day=2)        # 1=Sunday … 7=Saturday
Book.objects.filter(published__gte="2024-01-01")  # ISO string works
Book.objects.filter(published__date=some_date)    # for DateTimeField, match date part
```

> Default-lookup quirk: `__iexact`/`__icontains` map to case-insensitive matching (`ILIKE` on PostgreSQL, `LOWER(...) LIKE LOWER(...)` elsewhere) and are honest about case folding; `__exact` is the default lookup, so `filter(title="Dune")` and `filter(title__exact="Dune")` are identical.

## Core concept: Q objects for OR / AND / NOT

`filter(a=1, b=2)` only ANDs. For `OR`, grouped logic, or negation inside a filter, use `Q`.

```python
from django.db.models import Q

# price < 10  OR  stock = 0
Book.objects.filter(Q(price__lt=10) | Q(stock=0))

# (cheap OR out-of-stock) AND author from US
Book.objects.filter(
    (Q(price__lt=10) | Q(stock=0)) & Q(author__country="US")
)

# NOT cheap
Book.objects.filter(~Q(price__lt=10))

# Mix Q with keyword args — Q must come FIRST
Book.objects.filter(Q(price__lt=10) | Q(stock=0), author__country="US")
```

Operators: `|` = OR, `&` = AND, `~` = NOT. Wrap each condition in its own `Q(...)`. A common interview trap: keyword arguments must follow positional `Q` arguments, never precede them.

## Core concept: F expressions

An `F()` references a **column value on the database side**, so the operation happens in SQL — no read-modify-write race, and no fetching the row into Python.

```python
from django.db.models import F

# Atomic increment — safe under concurrency.
# WRONG (race condition): book.stock += 1; book.save()
Book.objects.filter(pk=1).update(stock=F("stock") + 1)   # UPDATE ... SET stock = stock + 1

# Bulk price cut, done entirely in the DB in one statement:
Book.objects.update(price=F("price") * 0.9)

# Field-to-field comparison: books priced above 5x their stock count
Book.objects.filter(price__gt=F("stock") * 5)
```

Why this matters: `obj.stock += 1; obj.save()` reads the value into Python, then writes it back. Two concurrent requests both read `10`, both write `11`, and you lost an increment. `F("stock") + 1` lets the database do `stock = stock + 1` atomically.

After an `F()` update on a saved instance, the in-memory object is stale — call `obj.refresh_from_db()` to re-read.

## Core concept: aggregate() vs annotate()

`aggregate()` collapses the whole QuerySet to **one dict of summary values**. `annotate()` adds a computed column **per row/group**.

```python
from django.db.models import Count, Sum, Avg, Max, Min

# aggregate(): a single summary dict over ALL books
Book.objects.aggregate(
    total=Count("id"),
    avg_price=Avg("price"),
    max_price=Max("price"),
    inventory_value=Sum(F("price") * F("stock")),
)
# => {'total': 412, 'avg_price': Decimal('23.7'), 'max_price': Decimal('99.0'),
#     'inventory_value': Decimal('98123.50')}

# annotate(): one extra field PER author (GROUP BY author)
authors = Author.objects.annotate(num_books=Count("books"))
for a in authors:
    print(a.name, a.num_books)   # num_books available on each row

# Filter on an annotation with .filter() (= HAVING in SQL)
prolific = Author.objects.annotate(n=Count("books")).filter(n__gte=5)

# Order by an annotation
Author.objects.annotate(n=Count("books")).order_by("-n")
```

Rule of thumb: **`aggregate` returns a dict; `annotate` returns a QuerySet.** When annotating across a join, beware duplicate counting — use `Count("books", distinct=True)` if multiple joins inflate the count.

## Core concept: values(), values_list(), and skinny reads

By default a QuerySet yields full model instances. When you only need a few columns (e.g. for an API list or a report), fetch dicts or tuples instead — less memory, no model overhead.

```python
# Dicts:
Book.objects.values("id", "title")
# => <QuerySet [{'id': 1, 'title': 'Dune'}, ...]>

# Tuples:
Book.objects.values_list("id", "title")
# => <QuerySet [(1, 'Dune'), ...]>

# A single column as a flat list:
Book.objects.values_list("title", flat=True)
# => <QuerySet ['Dune', 'Python 101', ...]>

# Spanning relations works here too:
Book.objects.values("title", "author__name")
```

`only()` / `defer()` still return **model instances**, but control which columns load. Accessing a deferred field triggers an extra query for it.

```python
Book.objects.only("title")            # load PK + title; other fields lazy-loaded
Book.objects.defer("description")     # load everything EXCEPT description
```

Use `values()` for read-only/serialization; use `only()` when you still need model methods but want to skip heavy columns.

## Core concept: ordering, distinct, exists, count, slicing

```python
Book.objects.order_by("price")          # ASC
Book.objects.order_by("-price")         # DESC
Book.objects.order_by("author__name", "-price")  # multi-key
Book.objects.order_by("?")              # random (slow on big tables)

Book.objects.order_by("price").reverse()  # flips an existing ordering

Book.objects.values("author").distinct()  # SELECT DISTINCT

# Cheap existence check — does NOT pull rows
Book.objects.filter(price__gt=100).exists()   # SELECT 1 ... LIMIT 1 → bool

# Count in the DB (don't len() a huge QuerySet just to count)
Book.objects.filter(stock=0).count()          # SELECT COUNT(*)

# Slicing maps to LIMIT/OFFSET and stays lazy (no step):
Book.objects.all()[:10]      # LIMIT 10
Book.objects.all()[10:20]    # LIMIT 10 OFFSET 10
Book.objects.first()         # ORDER BY pk LIMIT 1
Book.objects.last()
```

Performance tip: `if qs.exists():` beats `if qs:` when you don't also need the rows, and `qs.count()` beats `len(qs)` when you don't need to load them. But if you're about to iterate the rows *anyway*, `len(list(qs))` reuses one fetch.

## Core concept: the N+1 problem

The classic killer. This loop runs **1 query for the books, then 1 query per book** to fetch each author:

```python
# BAD — 1 + N queries
for book in Book.objects.all():
    print(book.author.name)   # each .author hits the DB
```

### select_related — SQL JOIN for FK / OneToOne

For **forward** ForeignKey and OneToOne relations, `select_related` does an SQL `JOIN` and pulls everything in **one** query:

```python
# GOOD — 1 query total (LEFT JOIN authors)
for book in Book.objects.select_related("author"):
    print(book.author.name)

# Follow multiple/nested FKs:
Book.objects.select_related("author", "publisher__owner")
```

### prefetch_related — separate query for M2M / reverse FK

JOINs can't efficiently handle many-to-many or reverse FK (they'd multiply rows). `prefetch_related` runs a **second** query and stitches results in Python:

```python
# 2 queries total: one for books, one for ALL their tags
for book in Book.objects.prefetch_related("tags"):
    print([t.name for t in book.tags.all()])

# Reverse FK: authors with all their books, 2 queries instead of 1+N
Author.objects.prefetch_related("books")
```

Customize the prefetched query with `Prefetch`:

```python
from django.db.models import Prefetch

Author.objects.prefetch_related(
    Prefetch("books", queryset=Book.objects.filter(stock__gt=0).order_by("title"))
)
```

| Use case | Tool | Queries | SQL |
|---|---|---|---|
| Forward FK / OneToOne | `select_related` | 1 | JOIN |
| Reverse FK / ManyToMany | `prefetch_related` | 2 (or N+1 relations) | separate SELECTs |
| Both at once | chain them | — | — |

```python
Book.objects.select_related("author").prefetch_related("tags")
```

## Core concept: bulk operations

Saving in a loop = one INSERT/UPDATE per row. Bulk methods batch them.

```python
# One (or a few) INSERT statements instead of N
Book.objects.bulk_create([
    Book(title="A", author_id=1, price=9),
    Book(title="B", author_id=1, price=12),
], batch_size=500)

# Update many existing rows in batched UPDATEs
books = list(Book.objects.filter(stock=0))
for b in books:
    b.price = 0
Book.objects.bulk_update(books, ["price"], batch_size=500)

# QuerySet .update(): a single UPDATE ... WHERE, no Python objects loaded
Book.objects.filter(stock=0).update(price=0)

# QuerySet .delete(): a single DELETE
Book.objects.filter(stock=0).delete()
```

Caveats: `bulk_create` / `bulk_update` / `.update()` do **not** call `save()` or send `pre_save`/`post_save` signals, and `.update()` does not touch `auto_now` fields. Trade hooks for speed knowingly.

## Core concept: transactions with transaction.atomic

Wrap multiple writes so they all commit or all roll back.

```python
from django.db import transaction

# As a context manager
with transaction.atomic():
    order.status = "paid"
    order.save()
    Book.objects.filter(pk=order.book_id).update(stock=F("stock") - 1)
    # any exception here rolls back BOTH writes

# As a decorator on a view/function
@transaction.atomic
def checkout(request):
    ...
```

Row-level locking for "read then write" safety:

```python
with transaction.atomic():
    book = Book.objects.select_for_update().get(pk=1)  # SELECT ... FOR UPDATE
    book.stock -= 1
    book.save()
```

`select_for_update()` must be inside `atomic()`. Note `ATOMIC_REQUESTS = True` in settings wraps every request in a transaction automatically — handy but blunt.

## Core concept: see the SQL & drop to raw()

```python
# The compiled SQL for a QuerySet (does NOT execute it):
print(Book.objects.filter(price__gt=10).order_by("title").query)
# SELECT "shop_book"."id", ... FROM "shop_book" WHERE "shop_book"."price" > 10 ORDER BY ...

# Every query run this request (needs DEBUG=True):
from django.db import connection
print(connection.queries)   # list of {'sql': ..., 'time': ...}
print(len(connection.queries))

# Count queries in tests:
from django.test.utils import CaptureQueriesContext
with CaptureQueriesContext(connection) as ctx:
    list(Book.objects.select_related("author"))
print(len(ctx))   # assert this is 1
```

When the ORM can't express it (window functions, gnarly CTEs, vendor-specific SQL), use `raw()` — it still maps rows to model instances:

```python
books = Book.objects.raw(
    "SELECT * FROM shop_book WHERE price > %s ORDER BY price",
    [10],                                   # ALWAYS parameterize — no f-strings
)
for b in books:
    print(b.title)

# Fully custom SQL with no model mapping:
with connection.cursor() as cursor:
    cursor.execute("SELECT author_id, COUNT(*) FROM shop_book GROUP BY author_id")
    rows = cursor.fetchall()
```

Rule: reach for the ORM first; drop to `raw()`/cursor only for the 1% it can't do — and **never** interpolate user input into SQL strings.

## Common mistakes & gotchas

- **Triggering evaluation accidentally.** Printing a QuerySet in a shell, `len(qs)`, or `if qs:` all run the query. In templates/loops this can be the hidden N+1.
- **Re-querying instead of reusing the cache.** `Book.objects.all()` written twice = two queries. Assign to a variable and reuse it.
- **Forgetting `select_related`/`prefetch_related`.** Accessing `obj.fk.field` in a loop is the #1 source of slow Django endpoints.
- **Using `select_related` on M2M / reverse FK.** It only works on forward FK / OneToOne; use `prefetch_related` otherwise.
- **`get()` for non-unique rows.** Raises `MultipleObjectsReturned`. Use `filter().first()` if more than one is possible.
- **Read-modify-write races.** `obj.count += 1; obj.save()` loses concurrent updates. Use `F("count") + 1`.
- **Stale instance after `F()`/`.update()`.** The Python object isn't refreshed; call `refresh_from_db()`.
- **Counting annotations across joins.** Multiple joins inflate `Count` — use `distinct=True`.
- **`Q` kwargs ordering.** Keyword args must come *after* positional `Q` objects in `filter()`.
- **`.update()`/`bulk_*` skip signals and `auto_now`.** Don't use them where `save()` side effects are required.
- **Mutable default queries.** Slicing with a step (`qs[::2]`) forces evaluation; plain slices stay lazy.

## Best practices

- Build queries lazily and **execute once** — assign to a variable, then iterate.
- Default to `select_related` for forward FKs and `prefetch_related` for M2M/reverse FKs on any list endpoint.
- Use `.exists()` for presence checks and `.count()` for counts — never `len()`/`bool()` on large sets you won't use.
- Use `values()`/`values_list()` for serialization-only reads; `only()`/`defer()` when you still need model methods.
- Do arithmetic in the DB with `F()`; do multi-write operations inside `transaction.atomic`.
- Encapsulate complex queries in **custom manager methods** so views stay thin and queries are reusable/testable.
- Assert query counts in tests with `assertNumQueries` / `CaptureQueriesContext` to catch N+1 regressions.
- Always parameterize `raw()` SQL; never f-string user input.

## Interview Q&A

**Q: When does a Django QuerySet actually hit the database?**
**A:** Only when it's *evaluated*: iteration, `list()`, slicing with a step, `len()`, `bool()`, `repr()`, or pickling. Building and chaining (`filter`, `exclude`, `order_by`) is lazy and runs zero queries. The result is then cached on that QuerySet object, so re-iterating the same variable doesn't re-query.

**Q: Difference between `filter()`, `get()`, and `exclude()`?**
**A:** `filter()` returns a QuerySet of matching rows (possibly empty). `exclude()` is its negation (`NOT`). `get()` returns exactly one object and raises `DoesNotExist` for zero matches or `MultipleObjectsReturned` for more than one. Use `filter().first()` when "maybe one" is acceptable.

**Q: What is the N+1 problem and how do you fix it in Django?**
**A:** Looping over N objects and accessing a related object on each triggers 1 query for the list plus N queries for the relations. Fix forward FK/OneToOne with `select_related` (a JOIN, 1 query) and M2M/reverse FK with `prefetch_related` (a second query stitched in Python).

**Q: `select_related` vs `prefetch_related`?**
**A:** `select_related` performs an SQL JOIN and works for forward ForeignKey/OneToOne — one query. `prefetch_related` runs a separate query per relation and joins in Python, which is what M2M and reverse FK need (a JOIN would multiply rows). You can chain both.

**Q: When and why use `F()` expressions?**
**A:** To reference a column's value in the database, so the operation runs in SQL. Two big wins: atomic updates (`F("stock") + 1` avoids read-modify-write races) and field-to-field comparisons (`price__gt=F("cost")`) without loading rows into Python.

**Q: `annotate()` vs `aggregate()`?**
**A:** `aggregate()` collapses the whole QuerySet into one dict of summary values (`{'avg': ...}`). `annotate()` adds a computed column per row/group (GROUP BY) and returns a QuerySet, so you can further `filter()` (HAVING) or `order_by()` on it.

**Q: How do you avoid a race condition when incrementing a counter?**
**A:** Don't read it into Python and write it back. Use `Model.objects.filter(pk=pk).update(field=F("field") + 1)`, which compiles to `SET field = field + 1` and is atomic in the database. Call `refresh_from_db()` if you need the new value in memory.

**Q: `values()` vs `only()`?**
**A:** `values()` returns dicts (no model instances, no methods) and is ideal for serialization. `only()` returns real model instances but loads just the named columns; accessing an unfetched field triggers an extra query. Use `values()` for skinny read-only data, `only()` when you still need model behavior.

**Q: How do bulk operations differ from saving in a loop?**
**A:** `bulk_create`/`bulk_update` batch many rows into few statements, and QuerySet `.update()` runs a single `UPDATE ... WHERE` with no Python objects loaded. The trade-off: they skip `save()`, `pre_save`/`post_save` signals, and `auto_now` updates.

**Q: How do you guarantee multiple writes succeed or fail together?**
**A:** Wrap them in `transaction.atomic()` (context manager or decorator). Any unhandled exception rolls back the whole block. For read-then-write safety under concurrency, add `select_for_update()` inside the atomic block to lock the rows.

**Q: How do you see the SQL a QuerySet will run?**
**A:** `print(qs.query)` shows the compiled SQL without executing it. With `DEBUG=True`, `connection.queries` lists every query run with timings, and `CaptureQueriesContext` (or `assertNumQueries` in tests) counts them — the standard way to catch N+1 regressions.

**Q: When would you use `raw()` or a raw cursor?**
**A:** Only when the ORM can't express the query (certain window functions, recursive CTEs, vendor-specific features) or for performance-critical hand-tuned SQL. `raw()` still maps rows to model instances; `connection.cursor()` gives full control. Always parameterize — never interpolate user input.

## Practice exercises

Using the `Author`, `Book`, and `Tag` models above:

1. Return all books priced between 20 and 50 (inclusive) whose author is from `"PK"`, ordered by price descending.
2. In a single query, find books that are either out of stock **or** cheaper than 5, excluding any by author "Anon".
3. Atomically reduce the `stock` of book `pk=10` by 1 inside a transaction, locking the row, and refresh the instance afterward.
4. Produce a list of authors with their book count, keeping only authors with at least 3 books, ordered by count descending.
5. Compute the total inventory value (`price * stock` summed across all books) as a single number.
6. Fetch the 20 most recently published books *and* their authors with **exactly one** SQL query; prove it with a query-count assertion.

<details><summary>Solutions</summary>

```python
from django.db import connection, transaction
from django.db.models import Q, F, Count, Sum
from django.test.utils import CaptureQueriesContext

# 1.
Book.objects.filter(
    price__range=(20, 50), author__country="PK"
).order_by("-price")

# 2.
Book.objects.filter(
    Q(stock=0) | Q(price__lt=5)
).exclude(author__name="Anon")

# 3.
with transaction.atomic():
    book = Book.objects.select_for_update().get(pk=10)
    Book.objects.filter(pk=10).update(stock=F("stock") - 1)
book.refresh_from_db()

# 4.
Author.objects.annotate(n=Count("books")).filter(n__gte=3).order_by("-n")

# 5.
Book.objects.aggregate(total=Sum(F("price") * F("stock")))["total"]

# 6.
with CaptureQueriesContext(connection) as ctx:
    books = list(
        Book.objects.select_related("author").order_by("-published")[:20]
    )
    for b in books:
        _ = b.author.name        # no extra query thanks to select_related
assert len(ctx) == 1, len(ctx)
```

</details>

## Cheat sheet

```python
# --- Lookups ---
field__exact / iexact / contains / icontains
field__startswith / endswith / istartswith
field__gt / gte / lt / lte
field__in=[...] / range=(a, b) / isnull=True
date__year / month / day / week_day
rel__field            # span a ForeignKey

# --- Complex / computed ---
filter(Q(a=1) | Q(b=2) & ~Q(c=3))      # OR / AND / NOT (Q first!)
update(n=F("n") + 1)                    # atomic, DB-side
filter(price__gt=F("cost"))            # field-to-field

# --- Aggregation ---
qs.aggregate(avg=Avg("price"))         # -> dict (whole set)
qs.annotate(n=Count("books"))          # -> QuerySet (per group)
Count(..., distinct=True)              # avoid join inflation

# --- Skinny reads ---
.values("a", "b")  .values_list("a", flat=True)
.only("a")  .defer("big")

# --- Inspect / control ---
.order_by("-x")  .reverse()  .distinct()
.exists()  .count()  qs[:10]  qs[10:20]

# --- Kill N+1 ---
.select_related("fk")          # JOIN, forward FK / O2O, 1 query
.prefetch_related("m2m")       # 2 queries, M2M / reverse FK

# --- Bulk + tx ---
Model.objects.bulk_create([...]) / bulk_update(objs, ["f"])
qs.update(f=...)  qs.delete()
with transaction.atomic(): ...
.select_for_update()           # row lock (inside atomic)

# --- SQL ---
print(qs.query)                # compiled SQL, no exec
connection.queries             # all queries (DEBUG=True)
Model.objects.raw("... %s", [p])   # parameterize!
```

| I want to… | Use |
|---|---|
| One row, must exist | `get()` / `get_object_or_404` |
| Zero-or-one row | `filter().first()` |
| Does any row match? | `.exists()` |
| How many rows? | `.count()` |
| OR / NOT logic | `Q` objects |
| DB-side math / atomic update | `F` expressions |
| Summary over all rows | `aggregate()` |
| Per-group computed column | `annotate()` |
| Forward FK eager load | `select_related` |
| M2M / reverse eager load | `prefetch_related` |
| Many inserts/updates | `bulk_create` / `bulk_update` / `.update()` |
| All-or-nothing writes | `transaction.atomic` |
