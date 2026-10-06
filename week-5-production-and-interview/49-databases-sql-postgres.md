# 49 — Databases & SQL (PostgreSQL)

> Week 5 · Production & Interview Prep

## What you will learn

- The relational model in plain terms: tables, rows, keys, and how relationships are wired with foreign keys.
- The core SQL your ORM hides but interviewers expect on a whiteboard: `SELECT`/`WHERE`/`ORDER BY`/`LIMIT`, `INSERT`/`UPDATE`/`DELETE`.
- Every `JOIN` type (INNER, LEFT, RIGHT, FULL) with a mental model for each.
- Aggregation with `GROUP BY`, `HAVING`, and the difference between `WHERE` and `HAVING`.
- Subqueries (scalar, `IN`, correlated) and when a JOIN is better.
- Indexes: B-tree, composite, covering — what to index, and the real cost of over-indexing.
- Reading query plans with `EXPLAIN` / `EXPLAIN ANALYZE`.
- Transactions and ACID, isolation levels, the read anomalies they prevent, and deadlocks.
- Normalization (1NF–3NF) vs. denormalization trade-offs.
- PostgreSQL specifics that come up constantly: `psycopg` (v3), `JSONB`, and sequences.
- Connecting from Django and SQLAlchemy 2.x; ORM vs. raw SQL; and when NoSQL is the right call.

## PHP/Laravel bridge

You already know relational databases — Laravel's Eloquent sits on top of MySQL/PostgreSQL exactly the way Django's ORM and SQLAlchemy do in Python. Eloquent's `User::where('active', true)->get()` and the query builder `DB::table('users')->join(...)` are direct analogues to Django's `User.objects.filter(active=True)` and SQLAlchemy's `select(User)`. Laravel migrations map to Django migrations / Alembic. The SQL itself is identical across ecosystems — `JOIN`, `GROUP BY`, indexes, and transactions don't change when you switch from PHP to Python. The biggest practical shift: the Python world (and most interviewers) leans toward **PostgreSQL** rather than MySQL, so you should know Postgres quirks like `JSONB`, sequences (instead of `AUTO_INCREMENT`), and stricter `GROUP BY` rules.

## Core concept — The relational model

A **relational database** stores data in **tables** (relations). Each table has **columns** (attributes, with a type) and **rows** (records/tuples). The model is built on a few hard rules:

- **Primary key (PK):** a column (or set of columns) that uniquely identifies each row. Never null, never duplicated.
- **Foreign key (FK):** a column that references the primary key of another table — this is how relationships are enforced.
- **Unique constraint:** like a PK but you can have several, and one nullable value is allowed.

Relationships come in three shapes:

| Relationship | Example | How it's modeled |
|---|---|---|
| One-to-many | One author has many books | FK `author_id` on the `books` table |
| Many-to-many | Books ↔ tags | A **join table** `book_tags(book_id, tag_id)` |
| One-to-one | User ↔ profile | FK with a `UNIQUE` constraint |

Let's create a small schema we'll use throughout the lesson. Run this in `psql` or any client.

```sql
CREATE TABLE authors (
    id          SERIAL PRIMARY KEY,           -- auto-incrementing integer
    name        TEXT NOT NULL,
    country     TEXT
);

CREATE TABLE books (
    id          SERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    author_id   INTEGER NOT NULL REFERENCES authors(id) ON DELETE CASCADE,
    price       NUMERIC(8, 2) NOT NULL,       -- 8 digits, 2 after the decimal
    published   DATE,
    genre       TEXT
);

-- Many-to-many: a book can have many tags, a tag many books
CREATE TABLE tags (
    id    SERIAL PRIMARY KEY,
    name  TEXT UNIQUE NOT NULL
);

CREATE TABLE book_tags (
    book_id  INTEGER REFERENCES books(id) ON DELETE CASCADE,
    tag_id   INTEGER REFERENCES tags(id)  ON DELETE CASCADE,
    PRIMARY KEY (book_id, tag_id)             -- composite key prevents duplicate pairs
);
```

A few things a PHP dev should note: `SERIAL` is Postgres shorthand that creates an integer column backed by a **sequence** (more later) — it's the Postgres equivalent of MySQL's `AUTO_INCREMENT`. `ON DELETE CASCADE` means deleting an author auto-deletes their books — referential integrity enforced by the database, not your app code. Use `NUMERIC`/`DECIMAL` for money, never `FLOAT` (floating point can't represent `0.10` exactly).

Some seed data so our queries return something:

```sql
INSERT INTO authors (name, country) VALUES
    ('Ursula K. Le Guin', 'USA'),
    ('Haruki Murakami', 'Japan'),
    ('Chimamanda Ngozi Adichie', 'Nigeria');

INSERT INTO books (title, author_id, price, published, genre) VALUES
    ('The Left Hand of Darkness', 1, 14.99, '1969-03-01', 'scifi'),
    ('A Wizard of Earthsea',      1,  9.99, '1968-09-01', 'fantasy'),
    ('Kafka on the Shore',        2, 16.50, '2002-09-12', 'fiction'),
    ('Norwegian Wood',            2, 12.00, '1987-08-04', 'fiction'),
    ('Americanah',                3, 15.75, '2013-05-14', 'fiction');
```

## Core concept — Reading data: SELECT, WHERE, ORDER BY, LIMIT

`SELECT` is the workhorse. The logical pieces, in the order you usually think about them:

```sql
SELECT title, price                     -- which columns to return
FROM books                              -- from where
WHERE genre = 'fiction' AND price < 16  -- filter rows
ORDER BY price DESC                     -- sort
LIMIT 10 OFFSET 0;                      -- page: 10 rows, skip 0
```

Important: SQL **executes** in a different order than it's written. The engine runs `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`. This explains a classic beginner trap: a column alias defined in `SELECT` cannot be used in `WHERE` (because `WHERE` runs first), but **can** be used in `ORDER BY`.

```sql
-- Common predicates
SELECT * FROM books WHERE price BETWEEN 10 AND 15;       -- inclusive range
SELECT * FROM books WHERE genre IN ('scifi', 'fantasy'); -- set membership
SELECT * FROM books WHERE title ILIKE '%wood%';          -- case-insensitive match (Postgres ILIKE)
SELECT * FROM books WHERE published IS NULL;             -- NULL needs IS, never = NULL
```

The `NULL` rule trips everyone: `NULL` means "unknown", so `price = NULL` is never true — it's `NULL` (which is not `TRUE`). Always use `IS NULL` / `IS NOT NULL`.

`LIMIT` + `OFFSET` is how pagination usually starts, but `OFFSET 100000` makes the database scan and discard 100k rows. For deep pagination prefer **keyset (cursor) pagination**:

```sql
-- Page after the last id you saw — fast even at page 10,000
SELECT * FROM books WHERE id > 42 ORDER BY id LIMIT 10;
```

## Core concept — Writing data: INSERT, UPDATE, DELETE

```sql
-- INSERT and get the generated id back (Postgres RETURNING is gold)
INSERT INTO authors (name, country)
VALUES ('Octavia Butler', 'USA')
RETURNING id;          -- => 4

-- UPDATE — the WHERE clause is not optional in your head; forgetting it updates EVERY row
UPDATE books SET price = price * 1.10 WHERE genre = 'fiction';

-- DELETE
DELETE FROM books WHERE published < '1970-01-01';

-- UPSERT: insert, or update if the unique key already exists ("ON CONFLICT")
INSERT INTO tags (name) VALUES ('classic')
ON CONFLICT (name) DO NOTHING;
```

`RETURNING` is a Postgres feature MySQL historically lacked — it lets one statement insert/update and hand back the affected rows, avoiding a second round-trip. `ON CONFLICT` is Postgres's upsert (Laravel calls it `upsert()`).

## Core concept — JOINs (the interview favorite)

A JOIN combines rows from two tables based on a related column. Picture two overlapping circles (a Venn diagram); the JOIN type decides which parts you keep.

| JOIN | Keeps | Mental model |
|---|---|---|
| `INNER JOIN` | Rows matching in **both** tables | The overlap only |
| `LEFT JOIN` | All left rows + matches from right (NULLs when no match) | Everything on the left |
| `RIGHT JOIN` | All right rows + matches from left | Everything on the right |
| `FULL JOIN` | All rows from both, matched where possible | The whole Venn diagram |

```sql
-- INNER: only books that have an author (the normal case)
SELECT b.title, a.name AS author
FROM books b
INNER JOIN authors a ON a.id = b.author_id;

-- LEFT: every author, even those with no books (books columns are NULL)
SELECT a.name, b.title
FROM authors a
LEFT JOIN books b ON b.author_id = a.id;

-- Find the "orphans" — authors with zero books — using LEFT JOIN + IS NULL
SELECT a.name
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
WHERE b.id IS NULL;
```

`RIGHT JOIN` is just a `LEFT JOIN` with the tables flipped, so it's rare in practice — most people standardize on `LEFT`. `FULL JOIN` shows rows that exist on either side with no match on the other; useful for reconciling two data sources.

Many-to-many needs **two** joins through the join table:

```sql
SELECT b.title, t.name AS tag
FROM books b
JOIN book_tags bt ON bt.book_id = b.id
JOIN tags t       ON t.id = bt.tag_id;
```

> Note on `JOIN` placement vs `WHERE`: for an INNER JOIN it doesn't matter whether a condition is in `ON` or `WHERE`. For a LEFT JOIN it matters a lot — a condition on the right table in `WHERE` quietly turns it back into an INNER JOIN (because `NULL` rows get filtered out). Put right-table filters in the `ON` clause.

## Core concept — Aggregation: GROUP BY, aggregates, HAVING

Aggregate functions collapse many rows into one value: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. `GROUP BY` says "compute the aggregate once per group".

```sql
-- How many books and the average price, per author
SELECT a.name,
       COUNT(b.id)        AS book_count,
       ROUND(AVG(b.price), 2) AS avg_price
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
GROUP BY a.id, a.name           -- group by the PK; include name so we can SELECT it
ORDER BY book_count DESC;
```

`HAVING` filters **groups** after aggregation; `WHERE` filters **rows** before. You cannot put `COUNT(...) > 1` in `WHERE` because the count doesn't exist yet at that stage.

```sql
-- Only authors with more than one book, and only count books over $10
SELECT a.name, COUNT(b.id) AS pricey_books
FROM authors a
JOIN books b ON b.author_id = a.id
WHERE b.price > 10          -- filter rows first
GROUP BY a.id, a.name
HAVING COUNT(b.id) > 1;     -- then filter the resulting groups
```

PostgreSQL is stricter than MySQL here: every non-aggregated column in `SELECT` must appear in `GROUP BY` (or be functionally dependent on the grouped PK). MySQL historically let you select random columns; Postgres rejects it. This strictness is correct — it forces unambiguous queries.

`COUNT(*)` counts rows; `COUNT(column)` counts non-NULL values; `COUNT(DISTINCT column)` counts unique non-NULLs. Knowing the difference is a common interview check.

## Core concept — Subqueries

A subquery is a query nested inside another. Three flavors:

```sql
-- 1. Scalar subquery: returns a single value
SELECT title, price,
       (SELECT AVG(price) FROM books) AS overall_avg
FROM books;

-- 2. IN subquery: filter by a set produced by another query
SELECT name FROM authors
WHERE id IN (SELECT author_id FROM books WHERE price > 15);

-- 3. Correlated subquery: the inner query references the outer row (runs per row)
SELECT b.title, b.price
FROM books b
WHERE b.price > (
    SELECT AVG(price) FROM books WHERE author_id = b.author_id  -- per-author average
);
```

Correlated subqueries are powerful but can be slow (conceptually they re-run for every outer row), so the planner — or you — often rewrites them as JOINs or **window functions**:

```sql
-- Same "above own author's average" using a window function — usually faster and clearer
SELECT title, price FROM (
    SELECT title, price,
           AVG(price) OVER (PARTITION BY author_id) AS author_avg
    FROM books
) s
WHERE price > author_avg;
```

A practical rule: use a subquery when it reads clearly; reach for a JOIN when you need columns from both tables or when performance matters.

## Core concept — Indexes

An **index** is a separate data structure that lets the database find rows without scanning the whole table — like the index at the back of a book versus reading every page. Postgres's default index is a **B-tree** (balanced tree), great for equality (`=`) and range (`<`, `>`, `BETWEEN`, `ORDER BY`) lookups.

```sql
-- Speed up lookups/filters on author_id (FKs are NOT auto-indexed in Postgres!)
CREATE INDEX idx_books_author_id ON books(author_id);

-- Speed up filtering and sorting by published date
CREATE INDEX idx_books_published ON books(published);
```

**Composite (multi-column) indexes** follow the **left-prefix rule**: an index on `(author_id, published)` helps queries filtering on `author_id`, or `author_id AND published`, but *not* one filtering on `published` alone.

```sql
CREATE INDEX idx_books_author_pub ON books(author_id, published);
-- Helps: WHERE author_id = 1
-- Helps: WHERE author_id = 1 AND published > '2000-01-01'
-- Does NOT help: WHERE published > '2000-01-01'   (author_id is the leading column)
```

A **covering index** includes every column a query needs, so Postgres answers it from the index alone (an "index-only scan") without touching the table. The `INCLUDE` clause adds payload columns:

```sql
CREATE INDEX idx_books_author_covering ON books(author_id) INCLUDE (title, price);
-- SELECT title, price FROM books WHERE author_id = 1;  -> index-only scan
```

**What to index:**
- Primary keys (automatic) and columns you `JOIN` on (especially foreign keys).
- Columns frequently used in `WHERE`, `ORDER BY`, or `GROUP BY`.
- High-selectivity columns (many distinct values, like email) benefit most; a boolean with two values barely helps.

**The cost of over-indexing:** every index must be updated on every `INSERT`/`UPDATE`/`DELETE`, so writes slow down and disk usage grows. Indexes you never query are pure overhead. Rule of thumb: index for your real query patterns, then remove unused indexes (`pg_stat_user_indexes` shows usage).

## Core concept — EXPLAIN and EXPLAIN ANALYZE

`EXPLAIN` shows the planner's **chosen plan and estimated cost** without running the query. `EXPLAIN ANALYZE` actually runs it and shows **real timings and row counts** — the gold standard for diagnosing slow queries.

```sql
EXPLAIN ANALYZE
SELECT * FROM books WHERE author_id = 1;
```

Reading the output, you mostly look for:

- **Seq Scan** (sequential scan = reads the whole table). Fine for small tables; a red flag on big ones with a selective filter — usually means a missing index.
- **Index Scan / Index Only Scan** — the planner used an index. Good.
- **Estimated vs. actual rows** — if they diverge wildly, your table statistics are stale; run `ANALYZE books;` (or `VACUUM ANALYZE`).
- **cost=...** numbers are arbitrary units for *comparison*, not milliseconds. The `actual time=` in `EXPLAIN ANALYZE` is real milliseconds.

```text
Index Scan using idx_books_author_id on books  (cost=0.15..8.30 rows=2 width=72)
                                                (actual time=0.018..0.020 rows=2 loops=1)
  Index Cond: (author_id = 1)
Planning Time: 0.10 ms
Execution Time: 0.04 ms
```

A practical workflow: find the slow query, run `EXPLAIN ANALYZE`, look for a Seq Scan on a large table or a huge row estimate, add the right index, and confirm the plan switches to an Index Scan.

## Core concept — Transactions and ACID

A **transaction** groups statements into an all-or-nothing unit. Classic example: transferring money — debit one account and credit another must both happen or neither.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;     -- both stick. Use ROLLBACK to undo everything since BEGIN.
```

**ACID** is the set of guarantees:

| Letter | Property | Meaning |
|---|---|---|
| **A** | Atomicity | All statements commit or none do. |
| **C** | Consistency | Constraints (FKs, checks, unique) hold before and after. |
| **I** | Isolation | Concurrent transactions don't step on each other (degree set by isolation level). |
| **D** | Durability | Once committed, it survives a crash (written to disk/WAL). |

### Isolation levels and the anomalies they prevent

When transactions run concurrently, weaker isolation allows certain **read anomalies**:

| Anomaly | What happens |
|---|---|
| **Dirty read** | You read another transaction's *uncommitted* changes (which may roll back). |
| **Non-repeatable read** | You read a row twice in one transaction and get different values (another txn committed an UPDATE in between). |
| **Phantom read** | You run the same query twice and get *different rows* (another txn committed an INSERT/DELETE matching your filter). |

The SQL standard isolation levels, from weakest to strongest:

| Level | Dirty read | Non-repeatable | Phantom |
|---|---|---|---|
| Read Uncommitted | possible* | possible | possible |
| Read Committed | no | possible | possible |
| Repeatable Read | no | no | possible* |
| Serializable | no | no | no |

\* PostgreSQL specifics: Postgres never allows dirty reads, so **Read Uncommitted behaves like Read Committed**. Its default is **Read Committed**. Postgres's **Repeatable Read** is implemented with snapshots (MVCC) and actually prevents phantoms too; **Serializable** adds full serialization checks and may abort a transaction with a serialization error you must retry.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... your reads/writes ...
COMMIT;   -- may raise "could not serialize access" -> catch it and retry the whole txn
```

### Deadlocks

A **deadlock** is when transaction A holds a lock B wants, and B holds a lock A wants — neither can proceed. Postgres detects this and kills one transaction with a deadlock error. Prevent them by **acquiring locks in a consistent order** everywhere (e.g., always lock the lower account id first), keeping transactions short, and using `SELECT ... FOR UPDATE` deliberately.

```sql
-- Pessimistic lock: reserve these rows so no one else can modify them until COMMIT
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;  -- consistent order
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

## Core concept — Normalization vs. denormalization

**Normalization** organizes data to eliminate redundancy and update anomalies. The first three normal forms:

- **1NF:** atomic values (no comma-separated lists in a column), each row unique. One value per cell.
- **2NF:** 1NF + every non-key column depends on the *whole* primary key (matters for composite keys — no partial dependency).
- **3NF:** 2NF + no non-key column depends on another non-key column (no *transitive* dependency).

Example of a 3NF violation: storing `author_country` directly on the `books` table. `country` depends on `author_id` (a non-key column), not on the book — so an author moving countries means updating many rows inconsistently. The fix is exactly our schema: `country` lives on `authors`, books reference the author.

**Denormalization** deliberately reintroduces redundancy for read performance — e.g., storing a cached `book_count` on `authors` so you don't `COUNT` on every read. Trade-off: faster reads, but you must keep the duplicate in sync (triggers, application code, or scheduled jobs), risking drift. Default to normalized; denormalize only when profiling proves a read bottleneck.

| | Normalized | Denormalized |
|---|---|---|
| Redundancy | Minimal | Intentional duplication |
| Writes | Simple, consistent | Must update copies in sync |
| Reads | More JOINs | Fewer JOINs, faster |
| Use when | Default (OLTP) | Proven read hotspot, reporting |

## Core concept — PostgreSQL specifics

### Sequences (the engine behind SERIAL)

`SERIAL` creates a hidden **sequence** object — a thread-safe counter the database uses to generate ids. The modern, SQL-standard way is `GENERATED ... AS IDENTITY`:

```sql
CREATE TABLE events (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- preferred over BIGSERIAL
    name TEXT NOT NULL
);

SELECT nextval('authors_id_seq');   -- manually advance a sequence
SELECT currval('authors_id_seq');   -- last value this session generated
```

Two gotchas: sequences are **not** rolled back by a transaction (so a failed insert still "uses up" an id — gaps are normal and fine), and they are not gap-free counters. Never assume ids are contiguous.

### JSONB

`JSONB` stores JSON in a decomposed binary format you can index and query — perfect when part of your data is genuinely schemaless. Prefer `JSONB` over `JSON` (the `JSON` type stores raw text and is slower to query).

```sql
CREATE TABLE products (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    attrs JSONB NOT NULL DEFAULT '{}'
);

INSERT INTO products (name, attrs) VALUES
    ('Laptop', '{"ram_gb": 16, "tags": ["sale", "new"]}');

-- -> gets a JSON field as JSON; ->> gets it as text
SELECT name, attrs ->> 'ram_gb' AS ram FROM products;

-- @> is the "contains" operator; great with a GIN index
SELECT name FROM products WHERE attrs @> '{"ram_gb": 16}';

-- GIN index makes JSONB containment queries fast
CREATE INDEX idx_products_attrs ON products USING GIN (attrs);
```

Use JSONB for sparse/variable attributes or external payloads — but if a field is queried/filtered heavily and exists on every row, a real column with a real type and constraint is still better.

## Core concept — Connecting from Python

### psycopg (v3) — the raw driver

`psycopg` is the standard PostgreSQL driver for Python (v3 is the current major; the older `psycopg2` is still common in legacy code). Always use **parameterized queries** — never f-string your values in, or you open the door to SQL injection.

```python
import psycopg

# Context managers auto-commit on success and close cleanly.
with psycopg.connect("postgresql://user:pass@localhost:5432/library") as conn:
    with conn.cursor() as cur:
        # %s placeholders + a tuple of params -> the driver escapes safely
        cur.execute(
            "SELECT title, price FROM books WHERE author_id = %s AND price > %s",
            (1, 10.0),
        )
        for title, price in cur.fetchall():
            print(title, price)
        # => The Left Hand of Darkness 14.99

        cur.execute(
            "INSERT INTO authors (name, country) VALUES (%s, %s) RETURNING id",
            ("Octavia Butler", "USA"),
        )
        new_id = cur.fetchone()[0]
        print("inserted", new_id)   # => inserted 4
    # leaving the conn `with` block commits the transaction
```

> NEVER do this: `cur.execute(f"... WHERE name = '{name}'")`. If `name` is `'; DROP TABLE authors; --` you've handed an attacker your database. The `%s` placeholder is **not** Python string formatting — it's the driver's safe parameter binding.

### Django — settings + ORM

Django ships first-class PostgreSQL support. Configure it once, then mostly use the ORM:

```python
# settings.py
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "library",
        "USER": "user",
        "PASSWORD": "pass",
        "HOST": "localhost",
        "PORT": "5432",
        "CONN_MAX_AGE": 60,   # reuse connections (basic pooling)
    }
}
```

```python
from django.db import transaction
from myapp.models import Author, Book

# ORM query -> Django generates the SQL JOIN for you
books = Book.objects.filter(author__country="USA").select_related("author")

# Transactions: everything in the block commits together, or rolls back on exception
with transaction.atomic():
    author = Author.objects.create(name="Octavia Butler", country="USA")
    Book.objects.create(title="Kindred", author=author, price=13.0)

# When the ORM can't express it, drop to raw SQL (still parameterized!)
from django.db import connection
with connection.cursor() as cur:
    cur.execute("SELECT genre, COUNT(*) FROM books GROUP BY genre")
    rows = cur.fetchall()
```

`select_related` (for FK/one-to-one) and `prefetch_related` (for many-to-many/reverse FK) exist to defeat the **N+1 query problem** — the equivalent of Laravel's eager loading `with('author')`.

### SQLAlchemy 2.x — Core and ORM

SQLAlchemy is the dominant Python toolkit and pairs with FastAPI. The 2.x style uses `select()` and `session.execute()`:

```python
from sqlalchemy import create_engine, select, func, ForeignKey
from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column, relationship

engine = create_engine("postgresql+psycopg://user:pass@localhost/library", echo=False)

class Base(DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    books: Mapped[list["Book"]] = relationship(back_populates="author")

class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped["Author"] = relationship(back_populates="books")

with Session(engine) as session:
    # SELECT title, ... FROM books JOIN authors ... WHERE authors.name = ...
    stmt = select(Book).join(Author).where(Author.name == "Haruki Murakami")
    for book in session.scalars(stmt):
        print(book.title)

    # Aggregation via func.*
    stmt = select(Book.author_id, func.count()).group_by(Book.author_id)
    print(session.execute(stmt).all())
```

Note the connection string `postgresql+psycopg://` tells SQLAlchemy to use the psycopg v3 driver under the hood.

## Core concept — ORM vs. raw SQL, and when NoSQL

**ORM vs. raw SQL** — they're not enemies; pick per query.

| Use the ORM when | Use raw SQL when |
|---|---|
| CRUD and typical filters/joins | Complex reporting, window functions, CTEs the ORM mangles |
| You want safety + portability | You need a specific plan / DB-specific feature |
| Team productivity matters | Bulk operations where the ORM is too slow |

The ORM's biggest risks are the **N+1 problem** (a loop that lazily fires one query per row — always eager-load) and queries you *think* are cheap generating monstrous SQL. The cure is to log the generated SQL and `EXPLAIN ANALYZE` the hot paths.

**When NoSQL instead of a relational DB?** Relational/PostgreSQL is the right default for most backends — it gives you ACID, joins, and constraints. Reach for NoSQL when:

- **Document stores (MongoDB):** highly variable, schemaless documents and you rarely join across them. (But note: Postgres `JSONB` already covers many "I need flexible documents" cases without leaving SQL.)
- **Key-value (Redis):** caching, sessions, rate limiters, queues — sub-millisecond reads, not your source of truth.
- **Wide-column (Cassandra):** massive write throughput across many nodes with eventual consistency.
- **Search (Elasticsearch):** full-text relevance ranking and faceting.

The honest interview answer: "PostgreSQL by default; add a specialized store only for a need it genuinely can't meet, like caching with Redis or full-text search at scale."

## Common mistakes & gotchas

- **`= NULL` instead of `IS NULL`.** `anything = NULL` is never true; use `IS NULL` / `IS NOT NULL`.
- **Putting a right-table filter in `WHERE` after a `LEFT JOIN`,** which silently turns it into an INNER JOIN. Put it in `ON`.
- **`COUNT(column)` vs `COUNT(*)`:** the former skips NULLs. Easy off-by-some bug.
- **Forgetting `WHERE` on `UPDATE`/`DELETE`** — you change/erase the whole table. Run the `SELECT` version first.
- **Assuming foreign keys are indexed.** Postgres indexes PKs and UNIQUE columns automatically, but **not** FKs — add them yourself.
- **Deep `OFFSET` pagination** that scans and throws away thousands of rows. Use keyset pagination.
- **Over-indexing**, slowing every write for indexes no query uses. Audit with `pg_stat_user_indexes`.
- **String-formatting SQL** (`f"... {value}"`) — SQL injection. Always parameterize with `%s` / bound params.
- **N+1 queries from the ORM.** Use `select_related`/`prefetch_related` (Django) or `joinedload`/`selectinload` (SQLAlchemy).
- **Money in `FLOAT`.** Use `NUMERIC`/`DECIMAL` to avoid rounding errors.
- **MySQL-style loose `GROUP BY`.** Postgres requires every selected non-aggregate column to be grouped.
- **Assuming SERIAL ids are gap-free.** Sequences don't roll back; gaps are normal.

## Best practices

- Let the database enforce integrity: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, and `CHECK` constraints beat app-only validation.
- Always parameterize queries.
- Wrap multi-statement business operations in a transaction; keep transactions short to reduce lock contention.
- Index for your real query patterns; verify with `EXPLAIN ANALYZE`; remove unused indexes.
- Use `NUMERIC` for money, `TIMESTAMPTZ` for time (store UTC), and meaningful constraints.
- Prefer keyset pagination for large, deep lists.
- Normalize to 3NF by default; denormalize only with profiling evidence.
- Use connection pooling (`CONN_MAX_AGE`, or PgBouncer for high concurrency).
- Keep migrations under version control (Django migrations / Alembic) — never hand-edit production schema ad hoc.
- Read the generated SQL when using an ORM; the ORM is a convenience, not a reason to stop understanding the query.

## Interview Q&A

**Q: What's the difference between `WHERE` and `HAVING`?**
**A:** `WHERE` filters individual rows *before* grouping/aggregation; `HAVING` filters groups *after* aggregation. You can't reference an aggregate like `COUNT(*)` in `WHERE` because it doesn't exist yet at that stage — that belongs in `HAVING`.

**Q: Explain the JOIN types.**
**A:** `INNER JOIN` returns only rows with a match in both tables. `LEFT JOIN` returns all rows from the left table plus matches from the right (NULLs where none). `RIGHT JOIN` is the mirror image. `FULL JOIN` returns all rows from both sides, matched where possible. A classic use of `LEFT JOIN ... WHERE right.id IS NULL` is finding rows with no related record.

**Q: When does an index help, and what does it cost?**
**A:** A B-tree index speeds equality and range lookups, joins on the indexed column, and sorts, by avoiding a full table scan. Costs: it must be updated on every write (slowing inserts/updates/deletes) and consumes disk. Index selective, frequently-queried columns; avoid indexes you never use.

**Q: What is a covering index?**
**A:** An index that contains all columns a query needs (key columns plus `INCLUDE` payload), so Postgres answers from the index alone via an index-only scan, never touching the heap table — eliminating the extra random I/O of fetching rows.

**Q: How do you read `EXPLAIN ANALYZE` output? What's a red flag?**
**A:** It shows the actual plan, real timings, and actual vs. estimated rows. A red flag is a Seq Scan on a large table with a selective filter (missing index), or a big gap between estimated and actual rows (stale statistics — run `ANALYZE`). `cost` is a relative unit; `actual time` is real milliseconds.

**Q: What does ACID stand for?**
**A:** Atomicity (all-or-nothing), Consistency (constraints always hold), Isolation (concurrent transactions don't corrupt each other), Durability (committed data survives crashes).

**Q: Name the read anomalies and which isolation level prevents each.**
**A:** Dirty read (reading uncommitted data) — prevented at Read Committed and above. Non-repeatable read (same row, different value on re-read) — prevented at Repeatable Read. Phantom read (same query, different rows) — prevented at Serializable (and, in Postgres's snapshot-based Repeatable Read, also prevented). Postgres defaults to Read Committed and never allows dirty reads.

**Q: What is a deadlock and how do you avoid it?**
**A:** Two transactions each hold a lock the other needs, so neither proceeds; the DB detects it and aborts one. Avoid by acquiring locks in a consistent order everywhere, keeping transactions short, and using explicit row locks deliberately. The aborted transaction should be retried.

**Q: Explain 1NF, 2NF, 3NF briefly.**
**A:** 1NF: atomic values, no repeating groups. 2NF: 1NF plus no partial dependency on part of a composite key. 3NF: 2NF plus no transitive dependency (non-key columns don't depend on other non-key columns). 3NF is the practical target for most OLTP schemas.

**Q: When would you denormalize?**
**A:** When profiling shows a read hotspot that joins/aggregates are too expensive for — e.g., caching a precomputed count or duplicating a frequently-read field to avoid a join. The cost is keeping the duplicate consistent, so denormalize deliberately, not by default.

**Q: JSON vs JSONB in Postgres?**
**A:** `JSON` stores the raw text and re-parses on each access; `JSONB` stores a decomposed binary form that's faster to query, supports operators like `@>`, and can be indexed with GIN. Use `JSONB` unless you must preserve exact input formatting/key order.

**Q: How do you prevent SQL injection in Python?**
**A:** Use parameterized queries — `%s` placeholders with a params tuple in psycopg, or bound parameters in SQLAlchemy/Django. Never build SQL by string-formatting user input. The driver escapes parameters safely; f-strings do not.

**Q: What's the N+1 query problem and how do you fix it?**
**A:** Looping over a result set and lazily triggering one extra query per row (1 + N). Fix with eager loading: `select_related`/`prefetch_related` in Django, `joinedload`/`selectinload` in SQLAlchemy, or a single JOIN in raw SQL.

## Practice exercises

Using the `authors`, `books`, `tags`, and `book_tags` schema from this lesson:

1. Write a query returning each author's name and their number of books, including authors with zero books, ordered by book count descending.
2. Find all books priced above the average price of all books, showing title and price.
3. List authors who have **no** books, using a JOIN (not a subquery).
4. Given the query `SELECT * FROM books WHERE genre = 'fiction' ORDER BY published DESC LIMIT 20;`, propose the single best index and explain why.
5. Write a transaction that inserts a new author and one of their books atomically, in psycopg, rolling back if either fails.
6. Rewrite this correlated subquery as a JOIN: books priced above their own author's average price.

<details>
<summary>Solutions</summary>

**1.**
```sql
SELECT a.name, COUNT(b.id) AS book_count
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
GROUP BY a.id, a.name
ORDER BY book_count DESC;
```
The `LEFT JOIN` keeps zero-book authors; `COUNT(b.id)` (not `COUNT(*)`) correctly returns 0 for them because `b.id` is NULL.

**2.**
```sql
SELECT title, price
FROM books
WHERE price > (SELECT AVG(price) FROM books);
```
A scalar subquery computes the overall average once.

**3.**
```sql
SELECT a.name
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
WHERE b.id IS NULL;
```
Anti-join pattern: LEFT JOIN then keep only rows where the right side didn't match.

**4.**
```sql
CREATE INDEX idx_books_genre_published ON books(genre, published DESC);
```
A composite index on `(genre, published DESC)`: the leading `genre` column satisfies the equality filter, and the `published DESC` ordering lets Postgres return the top 20 already sorted — no separate sort step, no scanning the whole table.

**5.**
```python
import psycopg

with psycopg.connect("postgresql://user:pass@localhost/library") as conn:
    try:
        with conn.cursor() as cur:
            cur.execute(
                "INSERT INTO authors (name, country) VALUES (%s, %s) RETURNING id",
                ("N. K. Jemisin", "USA"),
            )
            author_id = cur.fetchone()[0]
            cur.execute(
                "INSERT INTO books (title, author_id, price) VALUES (%s, %s, %s)",
                ("The Fifth Season", author_id, 17.99),
            )
        conn.commit()          # both inserts persist together
    except Exception:
        conn.rollback()        # neither persists
        raise
```
(The `with conn:` block already commits on clean exit / rolls back on exception; the explicit calls make the intent obvious.)

**6.**
```sql
SELECT b.title, b.price
FROM books b
JOIN (
    SELECT author_id, AVG(price) AS avg_price
    FROM books
    GROUP BY author_id
) a ON a.author_id = b.author_id
WHERE b.price > a.avg_price;
```
Compute each author's average once in a derived table, then join — usually faster than re-running a correlated subquery per row.

</details>

## Cheat sheet

```sql
-- READ
SELECT cols FROM t WHERE pred ORDER BY col DESC LIMIT n OFFSET m;
WHERE x BETWEEN a AND b | x IN (...) | x ILIKE '%y%' | x IS NULL
-- Execution order: FROM > WHERE > GROUP BY > HAVING > SELECT > ORDER BY > LIMIT

-- WRITE
INSERT INTO t (c) VALUES (%s) RETURNING id;
UPDATE t SET c = v WHERE pred;       -- never forget WHERE
DELETE FROM t WHERE pred;
INSERT ... ON CONFLICT (c) DO UPDATE SET ...;   -- upsert

-- JOINS                                  WHERE-vs-HAVING
INNER  -> matches in both                 WHERE  = filter rows (pre-aggregate)
LEFT   -> all left + matches              HAVING = filter groups (post-aggregate)
FULL   -> everything, matched if possible
LEFT JOIN ... WHERE r.id IS NULL  -> anti-join (find missing)

-- AGGREGATE
SELECT g, COUNT(*), AVG(x) FROM t GROUP BY g HAVING COUNT(*) > 1;
COUNT(*) vs COUNT(col) [skips NULL] vs COUNT(DISTINCT col)

-- INDEXES
CREATE INDEX i ON t(col);                 -- B-tree (=, <, >, ORDER BY)
CREATE INDEX i ON t(a, b);                -- composite: left-prefix rule
CREATE INDEX i ON t(a) INCLUDE (b, c);    -- covering -> index-only scan
CREATE INDEX i ON t USING GIN (jsonb_col);-- JSONB / arrays / full-text
-- FKs are NOT auto-indexed; index them. Over-indexing slows writes.

-- PLANS
EXPLAIN ANALYZE SELECT ...;  -- Seq Scan = full scan (red flag if large+selective)
ANALYZE t;                   -- refresh planner statistics

-- TRANSACTIONS / ISOLATION
BEGIN; ... COMMIT | ROLLBACK;
SELECT ... FOR UPDATE;       -- pessimistic row lock (consistent order!)
Levels:   Read Committed (PG default) < Repeatable Read < Serializable
Anomalies: dirty / non-repeatable / phantom read

-- POSTGRES BITS
id BIGINT GENERATED ALWAYS AS IDENTITY     -- modern auto-id (vs SERIAL/sequence)
attrs JSONB; attrs ->> 'k'; attrs @> '{"k":1}'   -- JSONB access + contains
NUMERIC for money, TIMESTAMPTZ for time (UTC)
```

```python
# psycopg (v3) — ALWAYS parameterize
with psycopg.connect(dsn) as conn, conn.cursor() as cur:
    cur.execute("SELECT * FROM t WHERE id = %s", (id,))   # %s, never f-strings

# Django
Book.objects.filter(author__country="USA").select_related("author")  # avoid N+1
with transaction.atomic(): ...

# SQLAlchemy 2.x  (postgresql+psycopg://...)
select(Book).join(Author).where(Author.name == "x")
session.scalars(stmt)
```
