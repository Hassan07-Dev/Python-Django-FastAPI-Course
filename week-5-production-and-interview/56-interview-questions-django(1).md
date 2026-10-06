# 56 — Interview Questions: Django & DRF

> Week 5 · Production & Interview

## What you will learn

- A curated bank of **40+ Django and DRF interview questions** with strong, concise model answers
- How to *talk* about Django architecture: MVT, the request/response lifecycle, project vs app
- The ORM questions interviewers love: QuerySet laziness, `select_related` vs `prefetch_related`, the N+1 problem, `Q`/`F` objects, `annotate`/`aggregate`, transactions, `get` vs `filter`, bulk operations
- Migrations: how they actually work, data migrations, and faking
- Models & relationships: `on_delete`, `related_name`, M2M `through`
- Views (FBV vs CBV, generics), forms vs serializers, middleware vs signals
- Auth & permissions, custom user models, sessions vs token/JWT
- DRF deep cuts: `Serializer` vs `ModelSerializer`, `ViewSet` vs `APIView`, routers, permissions, throttling, pagination
- Caching, security (CSRF, SQL injection), performance, and settings/deployment talking points

## PHP/Laravel bridge

If you have shipped Laravel APIs, you already know 80% of the *concepts* here — you just need the Django vocabulary. Laravel's MVC maps to Django's **MVT** (the "view" in Django is your controller; the "template" is the view layer). Eloquent ↔ the Django ORM, migrations ↔ migrations, Laravel middleware ↔ Django middleware, Laravel events/listeners ↔ Django signals, Form Requests / API Resources ↔ DRF serializers, Sanctum/Passport ↔ DRF token & JWT auth, and Laravel's service container resolution ↔ Django's much more explicit, less magical wiring. Interviewers will reward you for naming these bridges out loud — it shows you understand the *idea*, not just one framework's syntax.

> **How to use this lesson:** read the question, cover the answer, and try to say it out loud in 2-4 sentences before reading the model answer. Interviewers care less about a perfect definition than about whether you can reason about *trade-offs* and *failure modes*.

---

## Section 1 — Architecture, MVT & request lifecycle

**Q: What is MVT and how does it differ from MVC?**
**A:** MVT = Model–View–Template. The **Model** is your data layer (ORM classes), the **Template** is the presentation layer (HTML), and the **View** is a Python function or class that takes a request and returns a response — it holds the request-handling logic. The mapping to classic MVC is: Django's *View* is MVC's *Controller*, and Django's *Template* is MVC's *View*. Django also describes the URL dispatcher as part of how the "controller" is wired. The practical point: in Django the framework itself is the controller (it routes requests), and your "view" is application logic.

**Q: Walk me through the Django request/response lifecycle.**
**A:** (1) The WSGI/ASGI server hands the request to Django's request handler. (2) **Request middleware** runs top-to-bottom (`__call__` / `process_request`). (3) The **URL resolver** matches `request.path` against `urlpatterns` and finds the view + captured kwargs. (4) **View middleware** (`process_view`) runs, then the **view** executes — typically querying models and rendering a template or serializing data. (5) The view returns an `HttpResponse`. (6) **Response middleware** runs bottom-to-top. (7) The response is returned to the server and sent to the client. Exceptions are caught by `process_exception` hooks along the way.

**Q: WSGI vs ASGI — when do you need ASGI?**
**A:** WSGI is the synchronous server interface (Gunicorn, uWSGI) — one request occupies a worker until it finishes. ASGI is the async interface (Uvicorn, Daphne) that supports `async def` views, WebSockets, long-lived connections, and server-sent events. Use ASGI when you have I/O-bound concurrency, real-time features (Channels), or many slow upstream calls. Django supports both; you choose via the `application` callable in `wsgi.py` / `asgi.py`.

**Q: Project vs app — what's the difference and why does it matter?**
**A:** A **project** is the whole deployable Django site: settings, root URLconf, WSGI/ASGI entrypoints. An **app** is a self-contained, reusable module of functionality (models, views, migrations, admin) that plugs into a project via `INSTALLED_APPS`. One project contains many apps; a well-designed app (like `django.contrib.auth`) can be reused across projects. Rule of thumb: each app should "do one thing" (e.g. `billing`, `accounts`, `catalog`). In Laravel terms, a project is your whole application and apps are like cohesive modules/packages.

**Q: What does `manage.py` do that `django-admin` doesn't?**
**A:** `manage.py` is a thin wrapper that sets `DJANGO_SETTINGS_MODULE` to your project's settings and then calls `django-admin`. So `python manage.py migrate` knows which settings/DB to use, whereas bare `django-admin` would need the env var set manually. Both expose the same management commands.

---

## Section 2 — The ORM

**Q: What does it mean that a QuerySet is "lazy"? When does it hit the database?**
**A:** Building and chaining a QuerySet (`Book.objects.filter(...).exclude(...).order_by(...)`) constructs a *description* of a query and touches the DB **zero times**. It executes only when **evaluated**: iterating it, slicing with a step, calling `list()`, `len()`, `bool()`, `repr()`, `.count()`, `.exists()`, or pickling. This lets you compose queries cheaply and pass them around. The gotcha: re-evaluating the same QuerySet variable hits the DB again unless it's already cached from a full iteration.

```python
qs = Book.objects.filter(stock__gt=0)   # no SQL yet
qs = qs.order_by("price")               # still no SQL
for book in qs:                         # SQL runs HERE, results cached
    print(book.title)
print(len(qs))                          # uses the cache, no new query
```

**Q: `get()` vs `filter()` — what are the differences and the failure modes?**
**A:** `filter()` returns a (possibly empty) **QuerySet**; `get()` returns a **single object** and assumes exactly one match. `get()` raises `Model.DoesNotExist` if nothing matches and `Model.MultipleObjectsReturned` if more than one matches. Use `get()` for unique lookups (primary key, unique field), `filter().first()` when "zero rows" is acceptable, and `get_object_or_404()` in views to convert the miss into a 404.

```python
user = User.objects.get(pk=1)              # raises if missing
user = User.objects.filter(pk=1).first()   # None if missing
```

**Q: Explain the N+1 query problem and how Django solves it.**
**A:** N+1 happens when you fetch N rows then trigger one extra query *per row* to load a related object — e.g. looping books and accessing `book.author.name` issues a query for each author. The cures are `select_related` and `prefetch_related`.

```python
# BAD: 1 query for books + N queries for authors
for b in Book.objects.all():
    print(b.author.name)

# GOOD: 1 query total (SQL JOIN)
for b in Book.objects.select_related("author"):
    print(b.author.name)
```

**Q: `select_related` vs `prefetch_related` — when do you use each?**
**A:** `select_related` follows **single-valued** relationships (`ForeignKey`, `OneToOne`) by performing a SQL **JOIN** and pulling everything in one query. `prefetch_related` handles **multi-valued** relationships (reverse FK, `ManyToMany`) by running a **second query** and joining the results in Python. Rule: forward/`*-to-one` → `select_related`; reverse/`*-to-many` → `prefetch_related`. You can combine them, and use `Prefetch()` to filter or order the prefetched set.

```python
authors = (
    Author.objects
    .prefetch_related("books")          # 2 queries total
    .select_related("profile")          # joined into the main query
)
```

| | `select_related` | `prefetch_related` |
|---|---|---|
| Relationship | FK, OneToOne (to-one) | reverse FK, M2M (to-many) |
| Mechanism | SQL JOIN, one query | separate query + Python join |
| Customizable | no | yes, via `Prefetch()` |

**Q: What are `Q` objects and when do you need them?**
**A:** `Q` objects let you build complex `OR` / `AND` / `NOT` conditions and combine filters dynamically. Plain `filter(a=1, b=2)` is always `AND`; for `OR` you need `Q`. Combine with `|` (OR), `&` (AND), and `~` (NOT).

```python
from django.db.models import Q

Book.objects.filter(Q(title__icontains="django") | Q(title__icontains="flask"))
Book.objects.filter(~Q(stock=0))                      # NOT out of stock
Book.objects.filter(Q(price__lt=10) & Q(stock__gt=0))
```

**Q: What are `F` expressions and why are they important for correctness?**
**A:** `F()` references a model field's value *on the database side*, so the operation happens atomically in SQL instead of in Python. This avoids race conditions and an extra read. Classic example: incrementing a counter.

```python
from django.db.models import F

# Race-safe, single UPDATE, no read-then-write gap
Product.objects.filter(pk=1).update(stock=F("stock") - 1)

# Field-to-field comparison
Book.objects.filter(sold__gt=F("stock"))
```

**Q: Difference between `annotate()` and `aggregate()`?**
**A:** `aggregate()` collapses the whole QuerySet into a **single dict** of summary values. `annotate()` adds a computed column **per row** (per group), so each object gets its own value. Use `aggregate` for "total revenue across all orders"; use `annotate` for "number of books per author".

```python
from django.db.models import Count, Avg

Book.objects.aggregate(avg_price=Avg("price"))        # {'avg_price': 23.5}
Author.objects.annotate(num_books=Count("books"))     # each author .num_books
```

**Q: How do transactions work in Django?**
**A:** Wrap a block in `transaction.atomic` — either as a context manager or decorator. If the block raises, everything rolls back; otherwise it commits at the end. Nested `atomic` blocks use savepoints. For high-concurrency updates, combine with `select_for_update()` to lock rows. Note: by default Django runs in autocommit mode, so each query commits immediately unless wrapped.

```python
from django.db import transaction
from django.db.models import F

@transaction.atomic
def transfer(a, b, amount):
    acc_a = Account.objects.select_for_update().get(pk=a)
    acc_b = Account.objects.select_for_update().get(pk=b)
    acc_a.balance = F("balance") - amount
    acc_b.balance = F("balance") + amount
    acc_a.save(update_fields=["balance"])
    acc_b.save(update_fields=["balance"])
```

**Q: How do you do bulk operations efficiently?**
**A:** `bulk_create()` inserts many rows in one (or few) queries; `bulk_update()` updates many objects' specified fields; `QuerySet.update()` issues a single `UPDATE ... WHERE`. Caveats: `bulk_create`/`bulk_update` and `.update()` **do not call `save()`** and **do not fire `pre_save`/`post_save` signals**, and `.update()` skips `auto_now`. Use them for performance, but know what they bypass.

```python
Book.objects.bulk_create([Book(title=t, price=9) for t in titles])
Book.objects.filter(stock=0).update(active=False)     # 1 query, no signals
```

**Q: `only()` / `defer()` / `values()` / `values_list()` — what are they for?**
**A:** They reduce data transfer. `only("a", "b")` loads just those columns (others are lazy-loaded on access); `defer("body")` loads everything except the named columns. `values()` returns dicts and `values_list()` returns tuples instead of model instances — great when you don't need full objects (e.g. for a dropdown or export). `values_list("id", flat=True)` gives a flat list of one column.

---

## Section 3 — Migrations

**Q: How do Django migrations actually work?**
**A:** `makemigrations` compares your current models against the last migration state and writes a new migration file (Python describing `CreateModel`, `AddField`, etc.). `migrate` applies unapplied migrations to the DB and records each one in the `django_migrations` table so it's never run twice. Django builds a dependency graph from each migration's `dependencies` list to order them correctly across apps.

**Q: What is a data migration and how do you write one?**
**A:** A data migration changes *data* (not schema) — e.g. backfilling a new column or splitting a name field. You create an empty migration (`makemigrations --empty app`) and add a `RunPython` operation with a forward and (ideally) reverse function. Always access models via the historical `apps.get_model()` registry, not by importing the real model, so the migration works even if the model later changes.

```python
from django.db import migrations

def set_slugs(apps, schema_editor):
    Article = apps.get_model("blog", "Article")
    for a in Article.objects.all():
        a.slug = a.title.lower().replace(" ", "-")
        a.save(update_fields=["slug"])

class Migration(migrations.Migration):
    dependencies = [("blog", "0003_article_slug")]
    operations = [migrations.RunPython(set_slugs, migrations.RunPython.noop)]
```

**Q: What does "faking" a migration mean and when is it valid?**
**A:** `migrate --fake <app> <migration>` marks a migration as applied **without running its SQL**. Use it when the DB schema already matches (e.g. you applied a change manually, or you're adopting Django on an existing database). `--fake-initial` skips the initial migration if the tables already exist. Faking is a sharp tool — fake the wrong thing and the recorded state diverges from reality, breaking future migrations.

**Q: How do you reset or squash migrations?**
**A:** `squashmigrations app 0001 0012` collapses a range into one optimized migration (keeping the old ones until everyone has migrated). To fully reset in dev you can delete migration files + the DB and re-run `makemigrations`/`migrate` — but never do that on a shared/production DB. For production, prefer squashing.

**Q: Why might `makemigrations` produce a migration you didn't expect?**
**A:** Common causes: changing a field default, altering `choices`, reordering nothing but changing a `help_text`/`verbose_name`, or a third-party app bump. Also, missing migrations in version control cause "phantom" migrations on other machines. Run `makemigrations --check --dry-run` in CI to catch un-committed model changes.

---

## Section 4 — Models & relationships

**Q: What does `on_delete` control, and what are the main options?**
**A:** `on_delete` (required on `ForeignKey`) defines what happens to *this* row when the referenced row is deleted:

| Option | Effect |
|---|---|
| `CASCADE` | delete this row too |
| `PROTECT` | block the delete (raise `ProtectedError`) |
| `SET_NULL` | set FK to NULL (needs `null=True`) |
| `SET_DEFAULT` | set FK to its default |
| `SET(...)` | set FK to a value/callable |
| `RESTRICT` | block unless deleted via another cascade path |
| `DO_NOTHING` | do nothing (you handle DB integrity yourself) |

**Q: What is `related_name` and why set it?**
**A:** `related_name` names the **reverse accessor** from the related model back to this one. With `author = ForeignKey(Author, related_name="books")`, you write `author.books.all()` instead of the default `author.book_set.all()`. Set it for readability and to resolve clashes when a model has two FKs to the same target (otherwise Django raises a reverse-accessor conflict). Use `related_name="+"` to disable the reverse relation entirely.

**Q: How do you add extra data to a many-to-many relationship?**
**A:** Use a `through` model — an explicit join table you control, which can carry extra fields (e.g. `date_added`, `role`). You then add rows by creating `through`-model instances rather than `.add()`.

```python
class Membership(models.Model):
    person = models.ForeignKey("Person", on_delete=models.CASCADE)
    group = models.ForeignKey("Group", on_delete=models.CASCADE)
    date_joined = models.DateField()
    role = models.CharField(max_length=50)

class Group(models.Model):
    members = models.ManyToManyField("Person", through="Membership")
```

**Q: `null` vs `blank` — what's the difference?**
**A:** `null=True` is **database-level** (the column allows `NULL`). `blank=True` is **validation-level** (forms/serializers allow an empty value). They're independent: a `CharField` should usually use `blank=True` *without* `null=True` (Django convention stores empty strings, not NULL, for text). Numeric/date/FK fields that are optional typically need both.

**Q: What's the point of a model `Meta` class?**
**A:** `Meta` configures model-level behavior that isn't a field: `ordering`, `db_table`, `unique_together` / `constraints`, `indexes`, `verbose_name`, `abstract`, and `permissions`. Prefer `constraints = [UniqueConstraint(...)]` over the older `unique_together`.

---

## Section 5 — Views: FBV, CBV, generics

**Q: Function-based views vs class-based views — trade-offs?**
**A:** FBVs are explicit and easy to read — the whole flow is in one function; great for one-off or unusual logic. CBVs use inheritance and mixins to reuse common patterns (CRUD), reducing boilerplate but adding indirection (you must know the method-resolution order and which mixin sets what). Use FBVs for simple/bespoke views; use CBVs (especially generics) for standard CRUD. There's no performance difference.

**Q: What are generic class-based views?**
**A:** Pre-built CBVs for common tasks: `ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`, `TemplateView`. You subclass one, set `model`/`fields`/`template_name`, and override hooks like `get_queryset()` or `get_context_data()`. They encode the "right" way to do CRUD so you write very little code.

```python
from django.views.generic import ListView

class BookList(ListView):
    model = Book
    paginate_by = 20
    def get_queryset(self):
        return Book.objects.filter(active=True).select_related("author")
```

---

## Section 6 — Forms vs Serializers

**Q: Forms vs DRF serializers — how do they relate?**
**A:** Both validate and clean incoming data and can map to a model, but they target different layers. **Forms** validate HTML form / `POST` data and render widgets for server-rendered pages. **Serializers** convert between complex types (model instances, querysets) and JSON/Python primitives for APIs, handling both **serialization** (object → JSON) and **deserialization + validation** (JSON → object). `ModelForm` ↔ `ModelSerializer` is the closest pairing. In Laravel terms: Forms ≈ Form Requests + Blade form rendering; Serializers ≈ Form Requests (validation) + API Resources (output shaping) combined.

**Q: Where does validation logic go in a serializer?**
**A:** Field-level: `validate_<fieldname>(self, value)`. Object-level (cross-field): `validate(self, attrs)`. Reusable rules: validator functions/classes passed to the field. Keep DB-touching or business rules in `validate()` or the view/service layer, not scattered.

---

## Section 7 — Middleware & signals

**Q: What is middleware and how do you write one?**
**A:** Middleware is a layer that wraps every request/response — used for cross-cutting concerns (auth, sessions, CORS, logging, gzip). It's a callable that gets the next handler and returns a response; you can act before and after the view. Order in `MIDDLEWARE` matters (request phase top-down, response phase bottom-up).

```python
class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response          # one-time config
    def __call__(self, request):
        import time; start = time.monotonic()
        response = self.get_response(request)     # call view (or next mw)
        response["X-Elapsed"] = f"{time.monotonic() - start:.3f}s"
        return response
```

**Q: What are signals and when are they a *bad* idea?**
**A:** Signals (`pre_save`, `post_save`, `post_delete`, `m2m_changed`, custom) let decoupled code react to events — like Laravel events/listeners. They're great for genuinely decoupled side effects (e.g. invalidate a cache when a model changes). They become a **bad idea** when used for core business logic: they create hidden, hard-to-trace control flow ("why did this row change?"), make testing harder, run synchronously inside the same transaction (slowing saves), and don't fire on bulk operations (`bulk_create`, `.update()`). Prefer explicit service functions or overriding `save()` for logic you own; reserve signals for cross-app decoupling.

---

## Section 8 — Auth, permissions & custom user model

**Q: How does Django's auth system work at a high level?**
**A:** `django.contrib.auth` provides a `User` model, **authentication backends** (verify credentials), **sessions** (track the logged-in user), and a **permissions** framework (per-model add/change/delete/view perms + groups). `login()`/`logout()` manage the session; `@login_required` and `PermissionRequiredMixin` guard views; `request.user` is set by `AuthenticationMiddleware`.

**Q: Why and how should you set a custom user model?**
**A:** Almost always set one at project start, even if empty, because switching later is painful. The two routes: subclass `AbstractUser` (keep username/email fields, add your own) or `AbstractBaseUser` + `PermissionsMixin` (full control, e.g. email-as-username). Point `AUTH_USER_MODEL = "accounts.User"` in settings and reference users via `settings.AUTH_USER_MODEL` / `get_user_model()` everywhere, never by importing `User` directly.

```python
from django.contrib.auth.models import AbstractUser
class User(AbstractUser):
    is_verified = models.BooleanField(default=False)
# settings.py
AUTH_USER_MODEL = "accounts.User"
```

**Q: Sessions vs token vs JWT auth — compare them.**
**A:**

| | Session auth | Token auth (DRF) | JWT |
|---|---|---|---|
| State | server-side session store | DB-stored token | **stateless**, signed payload |
| Sent as | cookie | `Authorization: Token ...` | `Authorization: Bearer ...` |
| Revoke | delete session | delete token row | hard (need blocklist) |
| Best for | browser apps (same site) | simple APIs / mobile | distributed/microservices |
| CSRF | needs CSRF protection | not cookie-based, so no CSRF | not cookie-based |

Sessions are stateful and need a server store; JWTs are stateless and scale well but are hard to revoke (mitigate with short access tokens + refresh tokens + a blocklist). DRF's built-in `TokenAuthentication` is a simple DB-backed token; `djangorestframework-simplejwt` provides JWT.

**Q: Authentication vs authorization — DRF terms?**
**A:** **Authentication** answers "who are you?" (sets `request.user`) via `authentication_classes`. **Authorization** answers "are you allowed?" via `permission_classes` (e.g. `IsAuthenticated`, `IsAdminUser`, `DjangoModelPermissions`, or custom). They run in that order on every request.

---

## Section 9 — Django REST Framework

**Q: `Serializer` vs `ModelSerializer` — when each?**
**A:** `Serializer` is fully manual — you declare every field and write `create()`/`update()`. `ModelSerializer` introspects a model and **auto-generates** fields, validators, and basic `create`/`update` from `Meta.model` + `Meta.fields`. Use `ModelSerializer` for standard model-backed endpoints (the common case); drop to `Serializer` for non-model payloads or when you need full control. Always prefer explicit `fields = [...]` over `fields = "__all__"` to avoid leaking new columns.

```python
from rest_framework import serializers
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "price", "author"]
        read_only_fields = ["id"]
```

**Q: `APIView` vs `ViewSet` vs generic views?**
**A:** `APIView` is the base — you write `get`, `post`, etc. by hand (most control). **Generic views** (`ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`) add CRUD via mixins; you set `queryset` + `serializer_class`. **ViewSets** group related actions (`list`, `retrieve`, `create`, `update`, `destroy`) into one class and pair with a **router** that auto-generates URLs. Use `ModelViewSet` for full CRUD on a resource; `APIView` for bespoke logic.

**Q: What do routers do?**
**A:** A router takes a `ViewSet` and **auto-generates the URL patterns** for its actions (list/detail/create/etc.), giving consistent RESTful routes and named URLs. `DefaultRouter` also adds an API root view and format suffixes.

```python
from rest_framework.routers import DefaultRouter
router = DefaultRouter()
router.register("books", BookViewSet, basename="book")
urlpatterns = router.urls
```

**Q: How does throttling work in DRF?**
**A:** Throttling rate-limits requests. Built-in classes: `AnonRateThrottle` (by IP for anonymous), `UserRateThrottle` (by user id), and `ScopedRateThrottle` (per-view scopes). You set rates like `"100/day"` in `DEFAULT_THROTTLE_RATES`. It's coarse-grained protection, not a substitute for a real rate limiter/WAF at the edge.

```python
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": ["rest_framework.throttling.UserRateThrottle"],
    "DEFAULT_THROTTLE_RATES": {"user": "1000/day", "anon": "100/day"},
}
```

**Q: What pagination styles does DRF offer?**
**A:** `PageNumberPagination` (`?page=2`), `LimitOffsetPagination` (`?limit=20&offset=40`), and `CursorPagination` (opaque cursor, stable under inserts, best for large/real-time feeds). Set globally via `DEFAULT_PAGINATION_CLASS` + `PAGE_SIZE`, or per-view. Cursor pagination is the most scalable because it avoids costly large `OFFSET`s.

**Q: How do permissions resolve per object in DRF?**
**A:** Two levels: `has_permission(request, view)` runs for the whole view; `has_object_permission(request, view, obj)` runs for a specific object (called by `get_object()` in generic views). Custom example:

```python
from rest_framework.permissions import BasePermission, SAFE_METHODS
class IsOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:
            return True
        return obj.owner == request.user
```

**Q: How do you handle related/nested data in serializers?**
**A:** Options from cheap to rich: `PrimaryKeyRelatedField` (IDs), `StringRelatedField` (`__str__`), `SlugRelatedField` (a unique field), or a **nested serializer** (full embedded object — read-only unless you implement writable `create`/`update`). Always pair nested/related reads with `select_related`/`prefetch_related` in the view's queryset to avoid N+1.

---

## Section 10 — Caching, security, performance, deployment

**Q: What caching options does Django provide?**
**A:** Backends: in-memory `LocMemCache` (dev), Memcached, **Redis** (`django.core.cache.backends.redis`), database, and file. Levels: **per-view** cache (`@cache_page`), **template fragment** cache (`{% cache %}`), and the **low-level cache API** (`cache.get`/`cache.set`) for arbitrary values. Combine with `select_related` and DB indexes — caching is a complement to, not a substitute for, efficient queries.

**Q: How does Django protect against SQL injection?**
**A:** The ORM **parameterizes** all queries — values are sent separately from SQL, so user input is never concatenated into the statement. You're safe as long as you use the ORM or pass params to `raw()`/`cursor.execute()` via the params argument. You become vulnerable only if you build SQL with string formatting (`f"... {user_input}"`) or use `extra()`/`RawSQL` carelessly.

**Q: How does CSRF protection work in Django?**
**A:** Django issues a CSRF token (cookie + hidden form field / `X-CSRFToken` header). For unsafe methods (POST/PUT/PATCH/DELETE), `CsrfViewMiddleware` checks the submitted token matches. Templates add it with `{% csrf_token %}`. Token-based DRF APIs (Bearer/Token auth, not cookies) don't need CSRF because the credential isn't sent automatically by the browser — but **cookie/session-authenticated** API calls still do.

**Q: Name several Django security features you'd enable in production.**
**A:** `DEBUG = False`, a tight `ALLOWED_HOSTS`, `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE` / `CSRF_COOKIE_SECURE`, HSTS (`SECURE_HSTS_SECONDS`), `SECURE_CONTENT_TYPE_NOSNIFF`, XSS auto-escaping in templates, clickjacking protection (`X-Frame-Options` via middleware), password hashing (PBKDF2/Argon2), and a secret `SECRET_KEY` from the environment. `python manage.py check --deploy` audits most of these.

**Q: Top performance levers in a Django/DRF app?**
**A:** Kill N+1 with `select_related`/`prefetch_related`; add DB **indexes** on filtered/ordered columns; fetch only needed columns (`only`/`values`); paginate; cache hot reads (Redis); use `bulk_create`/`update`; avoid heavy work in serializers; offload slow tasks to **Celery**; and profile with `django-debug-toolbar` / `django-silk` and `connection.queries`.

**Q: How do you manage settings across environments?**
**A:** Keep secrets out of code — load from environment variables (e.g. `django-environ` / `os.environ`). Common patterns: a base `settings.py` plus per-env overrides (`settings/dev.py`, `settings/prod.py`) selected via `DJANGO_SETTINGS_MODULE`, or a single settings file driven by env vars. Never commit `SECRET_KEY`, DB passwords, or API keys.

**Q: How is a Django app typically deployed?**
**A:** App server: **Gunicorn** (WSGI) or **Uvicorn/Daphne** (ASGI) behind a reverse proxy (**Nginx**) that serves static files and terminates TLS. Run `collectstatic` to gather static assets (often served via WhiteNoise or a CDN). Use a process manager (systemd) or containers (Docker) + an orchestrator. Run migrations on deploy, set `DEBUG=False`, and run Celery + a broker (Redis/RabbitMQ) for background jobs.

---

## Common mistakes & gotchas

- **Re-querying a QuerySet in a loop** — slicing/filtering inside a loop re-hits the DB; evaluate once into a list if you'll reuse it.
- **Forgetting `select_related`/`prefetch_related`** — the #1 source of N+1 in real interviews and real apps.
- **Using `.update()` / `bulk_create` and expecting signals or `auto_now`** — they bypass `save()` and signals.
- **`get()` without catching `DoesNotExist`** — crashes with a 500 instead of a clean 404.
- **Mutable model defaults** — `default=[]` or `default={}` shares one object across rows; use a callable (`default=list`).
- **`fields = "__all__"` in serializers** — silently exposes new columns (including sensitive ones) when models change.
- **Putting business logic in signals** — hidden side effects that don't fire on bulk ops and are hard to test.
- **Not setting a custom user model up front** — migrating `AUTH_USER_MODEL` later is genuinely painful.
- **`null=True` on a `CharField`** — creates two "empty" states (`""` and `NULL`); use `blank=True` only.
- **Disabling CSRF globally to "fix" an error** — instead use proper auth or `@csrf_exempt` narrowly with care.

## Best practices

- Always pair API list/detail endpoints with `select_related`/`prefetch_related` and pagination.
- Keep apps small and single-purpose; reference the user via `get_user_model()` / `settings.AUTH_USER_MODEL`.
- Use `transaction.atomic` (and `select_for_update`) around multi-step writes that must be all-or-nothing.
- Prefer explicit serializer `fields = [...]` and `read_only_fields`; validate in `validate_<field>` / `validate`.
- Use `Meta.constraints`/`indexes` over legacy `unique_together`; index what you filter and order by.
- Put cross-cutting concerns in middleware, decoupled side effects in signals, and business logic in services or model methods.
- Run `manage.py check --deploy` and `makemigrations --check` in CI.
- Load secrets from the environment; never commit `SECRET_KEY` or credentials.

## Interview Q&A

(Rapid-fire summary set — the sections above are the deep bank; these reinforce the highest-yield ones.)

**Q: In one sentence, what is a QuerySet and when does it execute?**
**A:** It's a lazy, chainable description of a DB query that executes only on evaluation — iteration, slicing with a step, `list()`, `len()`, `bool()`, `count()`, `exists()`, or pickling.

**Q: A page makes one query per row for related authors. What's wrong and how do you fix it?**
**A:** That's the N+1 problem; fix it with `select_related("author")` (to-one) so all authors load in a single JOINed query, or `prefetch_related` for to-many relations.

**Q: How would you atomically decrement stock under concurrency?**
**A:** `Product.objects.filter(pk=id).update(stock=F("stock") - 1)`, ideally inside `transaction.atomic` with `select_for_update` if other reads depend on it — `F()` keeps the math in SQL and avoids a read-modify-write race.

**Q: When do you choose `ViewSet` + router over `APIView`?**
**A:** For standard CRUD on a resource — `ModelViewSet` + a `DefaultRouter` gives all five actions and consistent RESTful URLs with minimal code; drop to `APIView` for non-CRUD or highly custom logic.

**Q: Why prefer JWT in a microservices setup, and what's the catch?**
**A:** JWTs are stateless and self-contained, so any service can verify them without a shared session store; the catch is revocation — you need short-lived access tokens plus refresh tokens and possibly a blocklist.

**Q: Why are signals risky for core business logic?**
**A:** They create hidden control flow, run synchronously in the save transaction, are hard to test, and don't fire on `bulk_create`/`.update()` — so important logic can silently be skipped.

**Q: How does the ORM prevent SQL injection?**
**A:** It parameterizes queries, sending values separately from the SQL text; you only become vulnerable with manual string-built SQL or careless `raw()`/`extra()`.

**Q: What's the difference between authentication and authorization in DRF?**
**A:** Authentication (`authentication_classes`) establishes *who* the request is (`request.user`); authorization (`permission_classes`) decides *what* they may do — and authentication runs first.

**Q: `aggregate` vs `annotate` in one line each?**
**A:** `aggregate` returns one summary dict over the whole queryset; `annotate` adds a computed value to each row/group.

**Q: What does faking a migration do and when is it safe?**
**A:** `migrate --fake` records a migration as applied without running its SQL — safe only when the DB schema already matches (manual change or adopting Django on an existing DB).

## Practice exercises

Try these out loud (as if in an interview) or in a scratch Django shell, then check the solutions.

1. Given `Order` with a `customer` FK and `items` reverse FK, write the queryset that loads 50 orders with their customer and all items in the fewest queries.
2. Write a race-safe view action that decrements a `Product.stock` by 1 and refuses to go below zero.
3. Convert this FBV idea to a DRF `ModelViewSet` for `Book` with read for anyone and write only for authenticated users.
4. Write a data migration that uppercases all existing `Country.code` values.
5. Explain (in 3 sentences) why `Author.objects.filter(books__title__icontains="x").distinct()` might return duplicate-looking rows without `distinct()`.

<details><summary>Solutions</summary>

**1.**
```python
orders = (
    Order.objects
    .select_related("customer")      # to-one: JOIN, same query
    .prefetch_related("items")       # to-many: 1 extra query
)[:50]
# 2 queries total
```

**2.**
```python
from django.db import transaction
from django.db.models import F
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    @action(detail=True, methods=["post"])
    def buy(self, request, pk=None):
        with transaction.atomic():
            updated = (
                Product.objects
                .filter(pk=pk, stock__gte=1)        # guard at the DB
                .update(stock=F("stock") - 1)       # atomic decrement
            )
        if not updated:
            return Response({"detail": "out of stock"}, status=409)
        return Response({"detail": "ok"})
```

**3.**
```python
from rest_framework import viewsets, permissions

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "price", "author"]

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.select_related("author")
    serializer_class = BookSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

# urls.py
router = DefaultRouter()
router.register("books", BookViewSet, basename="book")
```

**4.**
```python
from django.db import migrations

def upcase(apps, schema_editor):
    Country = apps.get_model("geo", "Country")
    for c in Country.objects.all():
        c.code = c.code.upper()
        c.save(update_fields=["code"])

class Migration(migrations.Migration):
    dependencies = [("geo", "0005_country")]
    operations = [migrations.RunPython(upcase, migrations.RunPython.noop)]
```

**5.** Joining across a to-many relation (`books`) produces one result *row per matching book*, so an author with three matching books appears three times. The ORM maps each row to an `Author`, yielding apparent duplicates. `distinct()` collapses them back to one row per author (at the SQL `SELECT DISTINCT` level).

</details>

## Cheat sheet

```text
ARCHITECTURE
  MVT: Model=data, View=controller-logic, Template=presentation
  Lifecycle: server → request mw → URL resolver → view mw → view → response mw → client
  Project = whole site; App = reusable feature module

ORM
  Lazy: executes on iterate / slice-step / list() / len() / bool() / count() / exists()
  get()  -> object or DoesNotExist / MultipleObjectsReturned
  filter() -> QuerySet (maybe empty); .first() -> obj or None
  select_related   = FK/O2O  -> JOIN, 1 query   (to-one)
  prefetch_related = revFK/M2M -> 2nd query     (to-many)
  Q() = OR/AND/NOT ( | & ~ );  F() = field op in SQL (atomic)
  aggregate() = one dict;  annotate() = per-row value
  bulk_create / bulk_update / .update()  -> fast, NO signals/save()
  transaction.atomic + select_for_update -> all-or-nothing + row lock

MIGRATIONS
  makemigrations -> file ; migrate -> apply + record in django_migrations
  data migration: RunPython + apps.get_model(...)
  --fake = mark applied w/o SQL ; squashmigrations = collapse range

MODELS
  on_delete: CASCADE / PROTECT / SET_NULL / SET_DEFAULT / RESTRICT / DO_NOTHING
  related_name -> reverse accessor ; M2M extra data -> through model
  null=DB level ; blank=validation level

DRF
  Serializer (manual) vs ModelSerializer (auto from model)
  APIView (manual) < Generic views (mixins) < ViewSet (+router for URLs)
  authentication = who ; permission = allowed?
  throttle rates "100/day" ; pagination: PageNumber / LimitOffset / Cursor
  has_permission (view) + has_object_permission (object)

AUTH
  custom user: AUTH_USER_MODEL early ; use get_user_model()
  session (stateful cookie) | token (DB) | JWT (stateless, hard to revoke)

SECURITY / PROD
  ORM parameterizes -> no SQLi ; {% csrf_token %} for cookie POSTs
  prod: DEBUG=False, ALLOWED_HOSTS, *_COOKIE_SECURE, HSTS, check --deploy
  deploy: Gunicorn/Uvicorn + Nginx + collectstatic + Celery/Redis
```
