# 39 — DRF: Pagination, Filtering & Versioning

> Week 4 · Django REST Framework

## What you will learn

- The three built-in **pagination** styles — `PageNumberPagination`, `LimitOffsetPagination`, `CursorPagination` — what their responses look like, and when each one is the right call.
- How to configure pagination **globally** vs **per-view**, and how to write a **custom paginator** (custom page size, query param names, response envelope).
- **Filtering** with `django-filter`: declarative `FilterSet` classes vs the quick `filterset_fields` shortcut, plus range/choice/method filters.
- DRF's two extra filter backends: **`SearchFilter`** (`search_fields`, fuzzy text search) and **`OrderingFilter`** (`ordering_fields`, client-controlled sorting).
- **Combining** multiple filter backends on one view, and how query params stack up.
- **API versioning**: the four built-in schemes (`URLPathVersioning`, `NamespaceVersioning`, `AcceptHeaderVersioning`, `QueryParameterVersioning`) and their trade-offs.
- **Renderers and parsers** — how DRF decides input/output formats (JSON, the browsable API, form/multipart uploads).
- Generating an **OpenAPI / Swagger** schema with `drf-spectacular`.
- What a well-designed **list endpoint** looks like overall.

## PHP/Laravel bridge

In Laravel you reach for `->paginate(15)` on an Eloquent query and get a `LengthAwarePaginator` with `data`, `current_page`, `last_page`, `links`, etc. — baked into the framework. DRF splits this into a pluggable **pagination class** you attach to a view or set globally, and the response envelope (`count`, `next`, `previous`, `results`) is the DRF default rather than Laravel's `meta`/`links`. Laravel filtering is usually hand-rolled `when($request->x, fn($q) => $q->where(...))` chains or a package like Spatie Query Builder; DRF formalizes this with **filter backends** (`django-filter`, `SearchFilter`, `OrderingFilter`) that read query params and narrow the queryset for you. API versioning in Laravel is typically just `Route::prefix('v1')`; DRF gives you four first-class schemes and exposes the chosen version as `request.version` so a single view can branch on it. Think of this lesson as "the list-endpoint toolkit" — the stuff that turns a raw `GET /books/` into a production API.

## Core concept: pagination

Never return an unbounded list. A `GET /books/` that streams 2 million rows will melt your DB and the client. Pagination caps each response and hands the client a way to ask for "the next slice."

DRF pagination only kicks in on views that return a **list** (generic `ListAPIView`, `ListModelMixin`, `ModelViewSet`'s `list` action). It does **not** touch detail/create/update responses. It also requires the view to use `self.paginate_queryset(...)` / `self.get_paginated_response(...)` — the generic views and viewsets do this for you automatically.

### Setup we'll reuse

```python
# catalog/models.py
from django.db import models


class Book(models.Model):
    title = models.CharField(max_length=300)
    author = models.CharField(max_length=200)
    genre = models.CharField(max_length=50)
    price = models.DecimalField(max_digits=8, decimal_places=2)
    published_year = models.PositiveIntegerField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]  # stable default ordering (matters for cursor pagination)

    def __str__(self):
        return self.title
```

```python
# catalog/serializers.py
from rest_framework import serializers
from .models import Book


class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "author", "genre", "price", "published_year", "created_at"]
```

### Global vs per-view configuration

Set a project-wide default in `settings.py`:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,  # required when a global pagination class is set
}
```

Override on a single view by setting `pagination_class`:

```python
# catalog/views.py
from rest_framework import generics
from rest_framework.pagination import LimitOffsetPagination
from .models import Book
from .serializers import BookSerializer


class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    pagination_class = LimitOffsetPagination  # this view ignores the global default
```

> **Gotcha:** if you set `DEFAULT_PAGINATION_CLASS` but forget `PAGE_SIZE`, DRF silently returns **all** rows — the paginator has no page size to work with, so it no-ops. Always pair them.

To **disable** pagination on one view, set `pagination_class = None`.

### Style 1 — PageNumberPagination

The classic "page 1, 2, 3…" approach. Client asks for `?page=2`.

```python
# catalog/pagination.py
from rest_framework.pagination import PageNumberPagination


class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = "page_size"  # let clients override: ?page_size=50
    max_page_size = 100                  # but cap it so nobody asks for 1,000,000
    page_query_param = "page"            # default; rename here if you like ("p")
```

Response shape:

```jsonc
// GET /books/?page=2&page_size=20
{
    "count": 137,                                   // total rows matching the query
    "next": "https://api.example.com/books/?page=3&page_size=20",
    "previous": "https://api.example.com/books/?page=1&page_size=20",
    "results": [ /* 20 serialized books */ ]
}
```

- **`page=last`** is a special value that jumps to the final page.
- Asking for a page past the end returns **404**, not an empty list — a deliberate DRF choice.
- **Pros:** human-friendly, lets clients jump to any page, exposes a total `count`.
- **Cons:** that `count` requires a `SELECT COUNT(*)` on every request (expensive on huge tables), and offset-style paging **skips or duplicates rows** if data is inserted/deleted while a user pages through.

### Style 2 — LimitOffsetPagination

SQL-style `LIMIT`/`OFFSET`. The client controls both the window size (`limit`) and where it starts (`offset`).

```python
# settings.py (or per-view)
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.LimitOffsetPagination",
    "PAGE_SIZE": 20,  # used as the default limit when ?limit= is omitted
}
```

```jsonc
// GET /books/?limit=10&offset=20   (skip 20 rows, take 10)
{
    "count": 137,
    "next": "https://api.example.com/books/?limit=10&offset=30",
    "previous": "https://api.example.com/books/?limit=10&offset=10",
    "results": [ /* 10 books */ ]
}
```

- **Pros:** flexible window; good for "infinite scroll where the client tracks its own offset" and for jumping to arbitrary positions.
- **Cons:** same `COUNT(*)` cost and the same "rows shift under you" inconsistency as page-number paging. Large offsets are **slow** in most databases (the engine still walks past all skipped rows).

Customize the query-param names and cap the limit:

```python
from rest_framework.pagination import LimitOffsetPagination


class MyLimitOffset(LimitOffsetPagination):
    default_limit = 20
    max_limit = 100
    limit_query_param = "limit"
    offset_query_param = "offset"
```

### Style 3 — CursorPagination

The scalable choice for large, append-heavy datasets (activity feeds, logs, timelines). Instead of an offset, the "next" link contains an **opaque cursor** that encodes "give me rows after this ordering value." There is **no `count`** and **no random page access** — only next/previous.

```python
# catalog/pagination.py
from rest_framework.pagination import CursorPagination


class BookCursorPagination(CursorPagination):
    page_size = 20
    ordering = "-created_at"          # MUST be a unique-ish, stable, indexed field
    cursor_query_param = "cursor"
```

```jsonc
// GET /books/?cursor=cD0yMDI2LTA2LTE5
{
    "next": "https://api.example.com/books/?cursor=cD0yMDI2LTA2LTAx",
    "previous": null,
    "results": [ /* 20 books */ ]
    // note: no "count" — that's intentional
}
```

- **Pros:** O(1)-ish — no `COUNT(*)`, constant-time even at row 10,000,000 because it uses a `WHERE created_at < ?` keyset query, not `OFFSET`. **Consistent** under concurrent inserts/deletes: you never skip or repeat a row.
- **Cons:** clients can only go forward/back one page at a time (no "jump to page 50"), and the cursor is opaque. The `ordering` field must be **monotonic and effectively unique** — paginate on a non-unique field like `genre` and rows can be lost across page boundaries. Pair it with the PK (e.g. order on `-created_at, -id`) or use a strictly unique column.

### Choosing a style — quick guide

| Need | Use |
|------|-----|
| Admin tables, "page 3 of 12", want a total count | `PageNumberPagination` |
| Client-controlled window, arbitrary jumps | `LimitOffsetPagination` |
| Huge tables, feeds, "load more", consistency under writes | `CursorPagination` |
| Performance is critical and you don't need a count | `CursorPagination` |

### A custom response envelope

If your frontend expects Laravel-style `meta`, override `get_paginated_response`:

```python
from collections import OrderedDict
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response


class EnvelopePagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = "page_size"
    max_page_size = 100

    def get_paginated_response(self, data):
        return Response(OrderedDict([
            ("data", data),
            ("meta", OrderedDict([
                ("total", self.page.paginator.count),
                ("per_page", self.get_page_size(self.request)),
                ("current_page", self.page.number),
                ("last_page", self.page.paginator.num_pages),
            ])),
            ("links", OrderedDict([
                ("next", self.get_next_link()),
                ("previous", self.get_previous_link()),
            ])),
        ]))
```

## Core concept: filtering

Filtering lets the client narrow the list via query params: `?genre=fantasy&price__lt=20`. DRF does this through **filter backends** — small classes that take the view's queryset plus the request and return a narrowed queryset. You attach them with `filter_backends`, then declare what's filterable.

### django-filter — the workhorse

`django-filter` is a separate package (DRF's own filtering is intentionally minimal). Install and register it:

```bash
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "django_filters",
]

REST_FRAMEWORK = {
    "DEFAULT_FILTER_BACKENDS": ["django_filters.rest_framework.DjangoFilterBackend"],
}
```

#### The quick way: `filterset_fields`

For simple exact-match filtering, just list fields on the view:

```python
# catalog/views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer


class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ["genre", "author", "published_year"]
```

```bash
# Exact matches, AND-combined across params:
GET /books/?genre=fantasy&published_year=2023
```

`filterset_fields` only does **exact** lookups. For ranges, contains, gt/lt, choices — you need a `FilterSet`.

#### The powerful way: a `FilterSet`

A `FilterSet` is to filtering what a serializer is to data: a declarative class describing each filter, its lookup, and its query-param name.

```python
# catalog/filters.py
import django_filters
from .models import Book


class BookFilter(django_filters.FilterSet):
    # Range filter -> exposes ?min_price= and ?max_price=
    min_price = django_filters.NumberFilter(field_name="price", lookup_expr="gte")
    max_price = django_filters.NumberFilter(field_name="price", lookup_expr="lte")

    # Case-insensitive substring match -> ?title=ring  (matches "Lord of the Rings")
    title = django_filters.CharFilter(field_name="title", lookup_expr="icontains")

    # Year range
    year_after = django_filters.NumberFilter(field_name="published_year", lookup_expr="gte")
    year_before = django_filters.NumberFilter(field_name="published_year", lookup_expr="lte")

    # Constrained choices -> ?genre=fantasy ; rejects anything not in the list
    genre = django_filters.ChoiceFilter(
        choices=[("fantasy", "Fantasy"), ("scifi", "Sci-Fi"), ("history", "History")]
    )

    # A custom method filter for logic that isn't a single lookup
    cheap = django_filters.BooleanFilter(method="filter_cheap")

    def filter_cheap(self, queryset, name, value):
        # ?cheap=true -> books under $10
        if value:
            return queryset.filter(price__lt=10)
        return queryset

    class Meta:
        model = Book
        # Meta.fields can declare auto-generated filters with explicit lookups:
        fields = {
            "author": ["exact", "icontains"],  # -> ?author= and ?author__icontains=
        }
```

Wire it up with `filterset_class` (which **replaces** `filterset_fields`):

```python
# catalog/views.py
class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = BookFilter
```

```bash
GET /books/?min_price=10&max_price=30&genre=fantasy&title=ring&cheap=false
# -> fantasy books priced 10–30 whose title contains "ring"
```

All filter params are **AND-combined** and every filter is **optional** — omit a param and that filter is skipped. Invalid values (e.g. `genre=banana` for the `ChoiceFilter`) return a **400** with a clear error.

### SearchFilter — fuzzy text search

`SearchFilter` adds a single `?search=` param that does an `OR` query across several fields. It's built into DRF (no extra package).

```python
from rest_framework.filters import SearchFilter


class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [SearchFilter]
    search_fields = ["title", "author"]          # ?search=tolkien matches either
    # search_param = "q"                          # rename the param if you want ?q=
```

```bash
GET /books/?search=tolkien      # icontains across title OR author
```

Prefix a field to change the match behavior:

| Prefix | Lookup | Meaning |
|--------|--------|---------|
| (none) | `icontains` | case-insensitive substring |
| `^` | `istartswith` | starts-with |
| `=` | `iexact` | exact match |
| `@` | full-text search | Postgres `SearchVector` (Postgres only) |
| `$` | `iregex` | regex |

```python
    search_fields = ["^title", "=author", "genre"]
```

You can also search across relations with `__`: `search_fields = ["publisher__name"]`.

### OrderingFilter — client-controlled sorting

`OrderingFilter` lets the client sort via `?ordering=`. A leading `-` means descending.

```python
from rest_framework.filters import OrderingFilter


class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [OrderingFilter]
    ordering_fields = ["price", "published_year", "title"]  # whitelist of sortable fields
    ordering = ["-published_year"]                          # default when ?ordering omitted
```

```bash
GET /books/?ordering=price            # cheapest first
GET /books/?ordering=-price,title     # most expensive first, ties broken by title
```

> **Security note:** always set `ordering_fields` to an explicit list. Using `ordering_fields = "__all__"` lets clients sort by *any* model field, including unindexed or sensitive ones — a recipe for slow queries.

### Combining filter backends

The real power comes from stacking all three. They run in order, each narrowing the queryset further.

```python
# catalog/views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import generics
from rest_framework.filters import SearchFilter, OrderingFilter
from .models import Book
from .serializers import BookSerializer
from .filters import BookFilter


class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = BookFilter
    search_fields = ["title", "author"]
    ordering_fields = ["price", "published_year", "title"]
    ordering = ["-published_year"]
```

```bash
GET /books/?genre=fantasy&min_price=10&search=ring&ordering=-price&page=2
# 1. DjangoFilterBackend: genre=fantasy AND price>=10
# 2. SearchFilter:        ... AND (title OR author icontains "ring")
# 3. OrderingFilter:      ... ORDER BY price DESC
# 4. Pagination:          ... LIMIT/OFFSET for page 2
```

That single line is the whole point of DRF: a production-grade list endpoint — filter, search, sort, paginate — assembled declaratively.

## Core concept: API versioning

APIs change. Versioning lets you ship a breaking change (renamed field, new auth, different shape) without breaking existing clients. DRF picks the version from the request, validates it against an allow-list, and exposes it as `request.version` so your view/serializer can branch.

Global config:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
    "VERSION_PARAM": "version",  # param/kwarg name used by URL & query schemes
}
```

Using the version inside a view:

```python
# catalog/views.py
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer, BookSerializerV2


class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()

    def get_serializer_class(self):
        # request.version is "v1" or "v2", validated against ALLOWED_VERSIONS
        if self.request.version == "v2":
            return BookSerializerV2
        return BookSerializer
```

### The four schemes

#### 1. `URLPathVersioning` — version in the path (most common)

```python
# urls.py
from django.urls import path
from catalog.views import BookListView

urlpatterns = [
    path("api/<version>/books/", BookListView.as_view()),
]
```

```bash
GET /api/v1/books/    # request.version == "v1"
GET /api/v2/books/    # request.version == "v2"
```

- **Pros:** explicit, cache-friendly (the URL itself differs), trivially testable in a browser, obvious in logs.
- **Cons:** every URL carries the version; "the same resource" has two URLs (some REST purists dislike this).

#### 2. `NamespaceVersioning` — version via URLconf namespaces

Same `/api/v1/...` URLs, but the version comes from the **namespace** of an `include()` rather than a `<version>` kwarg. This lets each version have a totally different URLconf, and it makes `reverse()` version-aware.

```python
# urls.py
from django.urls import include, path

urlpatterns = [
    path("api/v1/", include(("catalog.urls", "catalog"), namespace="v1")),
    path("api/v2/", include(("catalog.urls", "catalog"), namespace="v2")),
]
```

- **Pros:** clean separation; `reverse("v2:book-list")` builds correctly versioned hyperlinks.
- **Cons:** more URLconf machinery; the version isn't a free path variable, it's tied to routing.

#### 3. `AcceptHeaderVersioning` — version in the `Accept` header

```bash
GET /books/
Accept: application/json; version=2.0
```

```python
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.AcceptHeaderVersioning",
    "ALLOWED_VERSIONS": ["1.0", "2.0"],
}
```

- **Pros:** the URL stays clean and stable — arguably the most "RESTful" approach (content negotiation).
- **Cons:** invisible in the address bar; harder to test by hand; some HTTP caches ignore the header unless you set `Vary: Accept`.

#### 4. `QueryParameterVersioning` — version in a query param

```bash
GET /books/?version=2.0
```

- **Pros:** easy to try in a browser without custom headers.
- **Cons:** mixes versioning into query params (which usually mean filtering/paging), and it's easy to forget — clients silently fall back to `DEFAULT_VERSION`.

### Trade-offs at a glance

| Scheme | Where version lives | Cache-friendly | Browser-testable | Notes |
|--------|--------------------|----------------|------------------|-------|
| `URLPathVersioning` | `/api/v2/books/` | Yes | Yes | Most popular; recommended default |
| `NamespaceVersioning` | URL namespace | Yes | Yes | Best when versions diverge a lot |
| `AcceptHeaderVersioning` | `Accept` header | Needs `Vary` | No | Cleanest URLs, "most RESTful" |
| `QueryParameterVersioning` | `?version=` | Partial | Yes | Easy but fragile |

**Practical advice:** pick **`URLPathVersioning`** unless you have a strong reason not to. It's explicit, debuggable, cache-friendly, and what most public APIs (Stripe-style aside) actually do.

## Core concept: renderers and parsers

DRF separates **content negotiation** into two halves:

- **Renderers** turn your `Response` data into bytes for the client (output: which `Content-Type`).
- **Parsers** turn the incoming request body into Python data (input: how to read `request.data`).

Defaults:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
        "rest_framework.renderers.BrowsableAPIRenderer",  # the clickable HTML UI
    ],
    "DEFAULT_PARSER_CLASSES": [
        "rest_framework.parsers.JSONParser",
        "rest_framework.parsers.FormParser",        # application/x-www-form-urlencoded
        "rest_framework.parsers.MultiPartParser",   # multipart/form-data -> file uploads
    ],
}
```

### How negotiation works

- **Output:** DRF reads the client's `Accept` header and the `?format=` query param. `Accept: application/json` (or `?format=json`) returns JSON; a browser sending `Accept: text/html` gets the **Browsable API** — an interactive HTML page with forms, great for manual testing.
- **Input:** DRF reads the request's `Content-Type` to choose a parser. Send `Content-Type: application/json` → `JSONParser`; send a file as `multipart/form-data` → `MultiPartParser`, and the file shows up in `request.FILES` / `request.data`.

In production you often **strip the Browsable API** to save bytes and avoid exposing an interactive UI, while keeping it in development:

```python
# settings.py
RENDERERS = ["rest_framework.renderers.JSONRenderer"]
if DEBUG:
    RENDERERS.append("rest_framework.renderers.BrowsableAPIRenderer")

REST_FRAMEWORK = {"DEFAULT_RENDERER_CLASSES": RENDERERS}
```

Per-view override (e.g. a file-upload endpoint that only accepts multipart):

```python
from rest_framework.parsers import MultiPartParser, FormParser
from rest_framework.views import APIView
from rest_framework.response import Response


class CoverUploadView(APIView):
    parser_classes = [MultiPartParser, FormParser]

    def post(self, request):
        cover = request.FILES["cover"]  # the uploaded file
        return Response({"filename": cover.name, "size": cover.size})
```

## Core concept: OpenAPI / Swagger with drf-spectacular

An OpenAPI schema is a machine-readable description of your API (endpoints, params, request/response shapes). From it you get **Swagger UI**, client-SDK generation, and contract tests. DRF ships a basic schema generator, but the de-facto standard is **`drf-spectacular`** (OpenAPI 3).

```bash
pip install drf-spectacular
```

```python
# settings.py
INSTALLED_APPS = ["drf_spectacular", ...]

REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

SPECTACULAR_SETTINGS = {
    "TITLE": "Book Catalog API",
    "DESCRIPTION": "A demo REST API for the course.",
    "VERSION": "1.0.0",
    "SERVE_INCLUDE_SCHEMA": False,  # don't expose the raw /schema in the docs UI
}
```

```python
# urls.py
from django.urls import path
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="swagger-ui"),
    path("api/redoc/", SpectacularRedocView.as_view(url_name="schema"), name="redoc"),
]
```

The best part: `drf-spectacular` **reads your filter backends and pagination** and documents the `?search=`, `?ordering=`, `?page=` params automatically. Add detail with the `@extend_schema` decorator:

```python
from drf_spectacular.utils import extend_schema, OpenApiParameter
from rest_framework import generics


@extend_schema(
    summary="List books",
    parameters=[
        OpenApiParameter("genre", str, description="Filter by genre"),
    ],
)
class BookListView(generics.ListAPIView):
    ...
```

Export the schema to a file for CI / client generation:

```bash
python manage.py spectacular --file schema.yml
```

## Putting it together: a good list-endpoint design

A well-designed list endpoint is **predictable and self-describing**:

```python
# catalog/views.py — the full picture
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import generics
from rest_framework.filters import SearchFilter, OrderingFilter
from .models import Book
from .serializers import BookSerializer
from .filters import BookFilter
from .pagination import StandardResultsSetPagination


class BookListView(generics.ListAPIView):
    serializer_class = BookSerializer
    pagination_class = StandardResultsSetPagination

    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = BookFilter
    search_fields = ["title", "author"]
    ordering_fields = ["price", "published_year", "title"]
    ordering = ["-published_year"]  # sensible, deterministic default

    def get_queryset(self):
        # Always start from an explicit, ordered queryset; trim columns/joins as needed.
        return Book.objects.all().only(
            "id", "title", "author", "genre", "price", "published_year", "created_at"
        )
```

Design checklist:

- **Always paginate.** Pick a sane default page size (10–50) with a capped `max_page_size`.
- **Always have a deterministic default ordering** — pagination on an unordered queryset gives non-repeatable results.
- **Whitelist** filterable, searchable, and orderable fields — never expose `__all__`.
- **Validate inputs** — a `FilterSet` returns 400 on bad values instead of silently ignoring them.
- **Document it** — let `drf-spectacular` surface every param.
- Avoid the **N+1** trap: pair list endpoints with `select_related`/`prefetch_related` for related fields the serializer touches.

## Common mistakes & gotchas

- **Setting `DEFAULT_PAGINATION_CLASS` without `PAGE_SIZE`** → pagination silently disabled, you return every row.
- **CursorPagination on a non-unique / unindexed `ordering` field** → rows skipped or duplicated across pages, and no index means full scans. Order on something unique and indexed (`-created_at, -id` or the PK).
- **Expecting a `count` from `CursorPagination`** → there isn't one by design; if the client needs a total, use page-number paging or add a separate count endpoint.
- **`ordering_fields = "__all__"`** → clients can sort by any column; combine with a missing index and you've handed users a slow-query button.
- **Forgetting `django_filters` in `INSTALLED_APPS`** → `DjangoFilterBackend` raises at import/runtime.
- **Using `filterset_fields` and `filterset_class` together** → `filterset_class` wins; `filterset_fields` is ignored. Pick one.
- **Filtering an unordered queryset then paginating** → page contents shuffle between requests. Always order.
- **Large `OFFSET` values** are slow even though the API call looks instant in dev with 50 rows; it bites at scale. Prefer cursors for deep paging.
- **Pagination only applies to list views.** If your custom `APIView.get()` returns a list manually, you must call `self.paginate_queryset()` / `self.get_paginated_response()` yourself.
- **Stripping `BrowsableAPIRenderer` and then wondering why `/books/` returns raw JSON in the browser** — that's expected; it's a production hardening, not a bug.
- **No `Vary: Accept` with `AcceptHeaderVersioning`** → shared caches may serve a v1 response to a v2 client.

## Best practices

- Default to **`URLPathVersioning`** — explicit, cacheable, debuggable.
- Use **`CursorPagination`** for feeds, logs, and any table that grows without bound; **`PageNumberPagination`** for admin-style UIs that want totals and page jumps.
- Always set a **`max_page_size` / `max_limit`** so a client can't request the whole table in one call.
- Keep one **canonical `FilterSet` per resource** instead of scattering ad-hoc query logic across views.
- Whitelist everything client-controllable (`ordering_fields`, search fields, filter fields). Treat the query string as untrusted input.
- Put filter/search/ordering config in the view but the **filter logic in a `FilterSet`** — it's testable in isolation.
- Set **global defaults** for pagination, filter backends, and renderers in `REST_FRAMEWORK`, then override per-view only when needed.
- Adopt **`drf-spectacular`** early and export the schema in CI so the docs never drift from the code.
- In production, ship **JSON-only renderers** and keep the Browsable API behind `DEBUG`.

## Interview Q&A

**Q: What are the three built-in DRF pagination styles and when would you use each?**
**A:** `PageNumberPagination` (`?page=2`) for human-friendly, jump-to-any-page UIs that want a total `count`; `LimitOffsetPagination` (`?limit=10&offset=20`) when the client wants to control the window size and start position; and `CursorPagination` (`?cursor=...`) for large, append-heavy datasets — it's consistent under concurrent writes and avoids the `COUNT(*)` and large-`OFFSET` performance costs, at the price of no random page access and no total count.

**Q: Why is `CursorPagination` faster on huge tables than offset-based paging?**
**A:** Offset paging issues `LIMIT n OFFSET m`; the database still scans and discards the first `m` rows, so deep pages get linearly slower. Cursor paging encodes the last seen ordering value and uses a keyset query — `WHERE created_at < ? ORDER BY created_at DESC LIMIT n` — which jumps straight to the right spot via an index, giving roughly constant-time pages. It also skips the `SELECT COUNT(*)` that page/offset paginators run every request.

**Q: What's the difference between `filterset_fields` and a `FilterSet` class?**
**A:** `filterset_fields` is a shortcut listing model fields for **exact-match** filtering only. A `FilterSet` is a declarative class where you define each filter explicitly — choose the lookup (`gte`, `icontains`, range), rename the query param (`min_price`), constrain choices, add custom method filters, and get **400 validation** on bad input. Use `filterset_fields` for trivial cases and a `FilterSet` for anything real.

**Q: How do `SearchFilter` and `DjangoFilterBackend` differ?**
**A:** `SearchFilter` is built into DRF and does a single fuzzy `?search=` query OR-combined across `search_fields` (substring by default) — great for a search box. `DjangoFilterBackend` (from the `django-filter` package) does structured, per-field filtering with explicit lookups and validation. They're complementary; you usually combine both plus `OrderingFilter` on one view.

**Q: How do you combine multiple filter backends, and in what order do they run?**
**A:** List them in `filter_backends`, e.g. `[DjangoFilterBackend, SearchFilter, OrderingFilter]`. They execute in that order, each receiving and narrowing the queryset returned by the previous one, then pagination runs last. The net effect is `filter AND search ORDER BY ... LIMIT/OFFSET`.

**Q: Walk me through DRF's four versioning schemes and which you'd pick.**
**A:** `URLPathVersioning` (`/api/v2/books/`) — explicit, cache-friendly, browser-testable. `NamespaceVersioning` — same URLs but version derived from URLconf namespaces, making `reverse()` version-aware. `AcceptHeaderVersioning` (`Accept: ...; version=2.0`) — cleanest URLs, most "RESTful", but invisible and needs `Vary: Accept`. `QueryParameterVersioning` (`?version=2.0`) — easy to test, but fragile and mixes versioning with query params. I'd default to `URLPathVersioning` for its explicitness and cache behavior.

**Q: Once DRF determines the version, how do you actually use it?**
**A:** It's exposed as `request.version` (validated against `ALLOWED_VERSIONS`, falling back to `DEFAULT_VERSION`). Views commonly branch in `get_serializer_class()` to return a v2 serializer, or alter the queryset/behavior. It's also available to serializers via `self.context["request"].version` for version-aware hyperlinks.

**Q: What's the difference between a renderer and a parser?**
**A:** A **renderer** serializes the outgoing `Response` into bytes and sets the `Content-Type` (JSON, Browsable API HTML); DRF chooses it via content negotiation on the `Accept` header / `?format=`. A **parser** deserializes the incoming request body into `request.data` based on `Content-Type` (`JSONParser`, `FormParser`, `MultiPartParser` for uploads). Renderers are output, parsers are input.

**Q: How do you handle file uploads in DRF?**
**A:** Set `parser_classes = [MultiPartParser, FormParser]` on the view (these are defaults anyway). The client sends `multipart/form-data`; the file appears in `request.FILES` (and `request.data`), and a serializer with a `FileField`/`ImageField` validates and saves it.

**Q: Why prefer `drf-spectacular` over DRF's built-in schema generation?**
**A:** It targets OpenAPI 3 (DRF's built-in is older/limited), produces accurate schemas that reflect your serializers, pagination, and filter backends, ships Swagger UI and ReDoc views, supports rich customization via `@extend_schema`, and can export to a file for client-SDK generation and contract tests in CI.

**Q: What makes a default ordering important for pagination?**
**A:** Pagination slices an ordered sequence. Without a deterministic `ORDER BY`, the database may return rows in any order, so the same `?page=2` can return different rows on different requests — and rows can be skipped or duplicated across pages. Set a stable default (`Meta.ordering` or the view's `ordering`), ideally on an indexed, unique-enough column.

**Q: How would you stop a client from requesting an unbounded result set?**
**A:** Set a global pagination class with a sane `PAGE_SIZE`, set `max_page_size`/`max_limit` on the paginator so even client-overridable page sizes are capped, whitelist `ordering_fields` instead of `__all__`, and validate filters via a `FilterSet` so bad params 400 rather than scanning the whole table.

## Practice exercises

1. **Page-number with overridable size.** Configure `PageNumberPagination` globally with `PAGE_SIZE=10`, then create a custom paginator that lets clients request up to 50 via `?page_size=`. Verify `?page_size=999` is clamped to 50.

2. **Cursor pagination, done right.** Add a `CursorPagination` subclass ordered on `-created_at` to a `Book` list view. Explain in a comment why ordering on `genre` instead would be a bug, and fix the ordering to be safe even if `created_at` ties.

3. **A real `FilterSet`.** Write `BookFilter` exposing: `?min_price=` / `?max_price=` (range on `price`), `?genre=` (choices: fantasy/scifi/history), `?title=` (case-insensitive contains), and `?in_print=` (a boolean method filter returning books where `published_year >= 2000`). Confirm `?genre=banana` returns 400.

4. **Full list endpoint.** Build one `ListAPIView` combining `DjangoFilterBackend` (your `BookFilter`), `SearchFilter` over `title`/`author`, and `OrderingFilter` over `price`/`published_year`, with a default ordering of `-published_year`. Write the single URL that filters fantasy books $10–$30, searches "ring", sorts by price descending, and asks for page 2.

5. **Versioning switch.** Set up `URLPathVersioning` with `ALLOWED_VERSIONS=["v1","v2"]`. Add a `BookSerializerV2` that renames `published_year` to `year`. Make the view return v2's serializer when `request.version == "v2"`. Give the two URLs that hit each version.

6. **Docs.** Add `drf-spectacular`, wire up `/api/schema/` and `/api/docs/`, and confirm the Swagger UI shows your `?search=`, `?ordering=`, and `?min_price=` params automatically.

<details><summary>Solutions</summary>

**1. Page-number with overridable size**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 10,
}

# catalog/pagination.py
from rest_framework.pagination import PageNumberPagination


class FlexiblePagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = "page_size"
    max_page_size = 50  # ?page_size=999 is clamped to 50
```

```python
# catalog/views.py
class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    pagination_class = FlexiblePagination
```

**2. Cursor pagination, done right**

```python
from rest_framework.pagination import CursorPagination


class BookCursorPagination(CursorPagination):
    page_size = 20
    # Ordering on `genre` would be a BUG: genre is non-unique, so the keyset
    # WHERE clause can't distinguish rows with the same genre at a page boundary
    # -> rows get skipped or duplicated. Order on a monotonic, effectively-unique
    # field. We add -id as a tiebreaker so even identical created_at values
    # produce a strict, stable total order.
    ordering = ("-created_at", "-id")
```

**3. A real `FilterSet`**

```python
# catalog/filters.py
import django_filters
from .models import Book


class BookFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name="price", lookup_expr="gte")
    max_price = django_filters.NumberFilter(field_name="price", lookup_expr="lte")
    title = django_filters.CharFilter(field_name="title", lookup_expr="icontains")
    genre = django_filters.ChoiceFilter(
        choices=[("fantasy", "Fantasy"), ("scifi", "Sci-Fi"), ("history", "History")]
    )
    in_print = django_filters.BooleanFilter(method="filter_in_print")

    def filter_in_print(self, queryset, name, value):
        if value:
            return queryset.filter(published_year__gte=2000)
        return queryset

    class Meta:
        model = Book
        fields = []  # all filters declared explicitly above

# ?genre=banana -> 400 Bad Request:
# {"genre": ["Select a valid choice. banana is not one of the available choices."]}
```

**4. Full list endpoint**

```python
# catalog/views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import generics
from rest_framework.filters import SearchFilter, OrderingFilter
from .models import Book
from .serializers import BookSerializer
from .filters import BookFilter


class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = BookFilter
    search_fields = ["title", "author"]
    ordering_fields = ["price", "published_year"]
    ordering = ["-published_year"]
```

```bash
GET /books/?genre=fantasy&min_price=10&max_price=30&search=ring&ordering=-price&page=2
```

**5. Versioning switch**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
}

# catalog/serializers.py
from rest_framework import serializers
from .models import Book


class BookSerializerV2(serializers.ModelSerializer):
    year = serializers.IntegerField(source="published_year")

    class Meta:
        model = Book
        fields = ["id", "title", "author", "genre", "price", "year"]

# catalog/views.py
class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()

    def get_serializer_class(self):
        if self.request.version == "v2":
            return BookSerializerV2
        return BookSerializer

# urls.py
from django.urls import path
from catalog.views import BookListView

urlpatterns = [
    path("api/<version>/books/", BookListView.as_view(), name="book-list"),
]
```

```bash
GET /api/v1/books/   # uses BookSerializer  -> "published_year"
GET /api/v2/books/   # uses BookSerializerV2 -> "year"
```

**6. Docs**

```bash
pip install drf-spectacular
```

```python
# settings.py
INSTALLED_APPS += ["drf_spectacular"]
REST_FRAMEWORK["DEFAULT_SCHEMA_CLASS"] = "drf_spectacular.openapi.AutoSchema"

# urls.py
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns += [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="swagger-ui"),
]
# Visit /api/docs/ — the filter/search/ordering params appear automatically because
# drf-spectacular introspects the view's filter_backends.
```

</details>

## Cheat sheet

```python
# ---- PAGINATION (settings.py global) ----
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,  # REQUIRED, else pagination silently off
}

# Page number:   ?page=2&page_size=20   -> {count, next, previous, results}
# Limit/offset:  ?limit=10&offset=20    -> {count, next, previous, results}
# Cursor:        ?cursor=<opaque>       -> {next, previous, results}  (NO count)

class P(PageNumberPagination):
    page_size = 20; page_size_query_param = "page_size"; max_page_size = 100

class C(CursorPagination):
    page_size = 20; ordering = ("-created_at", "-id")  # unique+indexed!

# Per-view:  pagination_class = P        Disable:  pagination_class = None
```

```python
# ---- FILTERING ----
filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]

filterset_fields = ["genre", "author"]   # exact match only (quick)
filterset_class  = BookFilter            # full control (replaces filterset_fields)

search_fields    = ["title", "author"]   # ?search=  (^starts =exact @fulltext $regex)
ordering_fields  = ["price", "year"]     # ?ordering=-price   (NEVER "__all__")
ordering         = ["-published_year"]   # default order

# FilterSet field types:
NumberFilter(field_name="price", lookup_expr="gte")   # ?x=  -> price >= x
CharFilter(field_name="title", lookup_expr="icontains")
ChoiceFilter(choices=[("a","A")])                     # validates -> 400 on bad value
BooleanFilter(method="filter_x")                      # custom logic
```

```python
# ---- VERSIONING ----
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1", "ALLOWED_VERSIONS": ["v1", "v2"],
}
# URLPath:    path("api/<version>/books/")     -> /api/v2/books/   (recommended)
# Namespace:  include(..., namespace="v2")     -> reverse-aware
# AcceptHdr:  Accept: application/json; version=2.0
# QueryParam: ?version=2.0
# Read it:    request.version   (often branch in get_serializer_class)
```

```python
# ---- RENDERERS / PARSERS / DOCS ----
# Renderers (OUT): JSONRenderer, BrowsableAPIRenderer   (Accept / ?format=)
# Parsers  (IN):  JSONParser, FormParser, MultiPartParser (file uploads -> request.FILES)
parser_classes = [MultiPartParser, FormParser]   # per-view for uploads

# OpenAPI:  pip install drf-spectacular
DEFAULT_SCHEMA_CLASS = "drf_spectacular.openapi.AutoSchema"
# /api/schema/  +  SpectacularSwaggerView at /api/docs/
# python manage.py spectacular --file schema.yml
```

| Decision | Default pick |
|----------|--------------|
| Pagination (general UI) | `PageNumberPagination` + `max_page_size` |
| Pagination (huge/feed) | `CursorPagination` on indexed unique field |
| Filtering | `DjangoFilterBackend` + `FilterSet` |
| Search box | `SearchFilter` (`search_fields`) |
| Sorting | `OrderingFilter` (whitelist `ordering_fields`) |
| Versioning | `URLPathVersioning` |
| Docs | `drf-spectacular` |
