# 37 — DRF: Views, ViewSets & Routers

> Week 4 · Django REST Framework

## What you will learn

- DRF's enhanced **`Request`** and **`Response`** objects, and how **content negotiation** picks JSON vs the browsable API automatically.
- The **`@api_view`** decorator for function-based views — the simplest way to build an endpoint.
- The **`APIView`** class — full control, one method per HTTP verb (`get`, `post`, `put`, ...).
- **Generic views & mixins** (`ListAPIView`, `RetrieveUpdateDestroyAPIView`, `ListCreateAPIView`, ...) and the magic `queryset` / `serializer_class` attributes.
- Overriding **`get_queryset()`** and **`get_serializer_class()`** for per-request logic.
- **ViewSets** and **`ModelViewSet`** — the full CRUD set in one class — plus **`@action`** for custom routes.
- **Routers** (`DefaultRouter` / `SimpleRouter`) and how they auto-generate your URLConf.
- How HTTP methods map to **actions** (`list` / `create` / `retrieve` / `update` / `partial_update` / `destroy`).
- A decision framework: when to reach for **`APIView`** vs **generics** vs **`ViewSets`**.
- Using **`status`** codes and `Response` correctly.

## PHP/Laravel bridge

In Laravel you write a controller (often `php artisan make:controller --api`), give it `index/store/show/update/destroy` methods, and wire it up with `Route::apiResource('books', BookController::class)` — one line that generates five routes following REST conventions. DRF's **`ModelViewSet` + `DefaultRouter`** is almost exactly that: the ViewSet's `list/create/retrieve/update/partial_update/destroy` actions correspond to Laravel's `index/store/show/update/destroy`, and `router.register('books', BookViewSet)` is the twin of `apiResource`. Laravel's `Request` (validation, content negotiation) and `response()->json($data, 201)` map onto DRF's `Request` and `Response(data, status=201)`. The big difference: DRF gives you a sliding scale — raw function views, class views, generic views, and ViewSets — so you can dial in *exactly* how much it does for you, whereas Laravel mostly hands you the resource controller and lets you delete what you don't need.

---

## Core concept: DRF's `Request` and `Response`

Django's plain views receive an `HttpRequest` and must return an `HttpResponse`. DRF wraps these in smarter objects so you stop hand-parsing bodies and hand-serializing JSON.

### The `Request` object

DRF's `Request` *extends* Django's. The headline feature is **`request.data`**: a single, uniform, parsed view of the request body that works for JSON, form-encoded, and multipart data alike. No more `json.loads(request.body)`.

| You want… | Django (plain) | DRF |
|---|---|---|
| Parsed JSON / form body | `json.loads(request.body)` *or* `request.POST` | `request.data` |
| Query string params | `request.GET` | `request.query_params` |
| The authenticated user | `request.user` | `request.user` (same, but auth-aware) |
| Uploaded files | `request.FILES` | `request.data` (files included) |

```python
# request.data is method-agnostic and parser-agnostic.
# Works the same whether the client sent JSON or a form.
title = request.data.get("title")
tags = request.data.get("tags", [])
```

### The `Response` object

You return `Response(data, status=...)` where `data` is **native Python** (dicts, lists, strings, numbers). DRF *renders* it into the right format based on content negotiation — usually JSON, but the browsable HTML API in dev too. You never call `json.dumps` yourself.

```python
from rest_framework.response import Response
from rest_framework import status

return Response({"id": 1, "title": "Dune"}, status=status.HTTP_201_CREATED)
```

### Content negotiation (the "magic" explained)

When a request arrives, DRF inspects the `Accept` header and the URL suffix to choose a **renderer**:

- `Accept: application/json` → JSON renderer → raw JSON.
- A browser sending `Accept: text/html` → BrowsableAPIRenderer → the clickable HTML "browsable API".
- A `.json` suffix on the URL (if you enabled format suffixes) forces JSON.

This is why the *same view* serves machine clients and a friendly debugging UI in a browser. You write Python objects; DRF picks the wire format. The default renderers are configured globally:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
        "rest_framework.renderers.BrowsableAPIRenderer",  # drop this in production if you like
    ],
    "DEFAULT_PARSER_CLASSES": [
        "rest_framework.parsers.JSONParser",
        "rest_framework.parsers.FormParser",
        "rest_framework.parsers.MultiPartParser",
    ],
}
```

---

## Core concept: `@api_view` (function-based views)

The lightest tool. Decorate a normal function, declare which methods it accepts, and you get the DRF `Request`/`Response` machinery (content negotiation, `request.data`, browsable API).

Assume this model and serializer throughout the lesson:

```python
# books/models.py
from django.db import models


class Book(models.Model):
    title = models.CharField(max_length=300)
    author = models.CharField(max_length=200)
    published_year = models.PositiveIntegerField()
    is_published = models.BooleanField(default=True)

    def __str__(self):
        return self.title
```

```python
# books/serializers.py
from rest_framework import serializers
from .models import Book


class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "author", "published_year", "is_published"]
```

Now a function view handling the **collection** (list + create):

```python
# books/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Book
from .serializers import BookSerializer


@api_view(["GET", "POST"])
def book_list(request):
    if request.method == "GET":
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)  # 200 by default

    # POST
    serializer = BookSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)  # auto-400 with error detail
    serializer.save()
    return Response(serializer.data, status=status.HTTP_201_CREATED)
```

And a second view for the **detail** (retrieve / update / delete a single item):

```python
# books/views.py (continued)
from django.shortcuts import get_object_or_404


@api_view(["GET", "PUT", "DELETE"])
def book_detail(request, pk):
    book = get_object_or_404(Book, pk=pk)

    if request.method == "GET":
        return Response(BookSerializer(book).data)

    if request.method == "PUT":
        serializer = BookSerializer(book, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    # DELETE
    book.delete()
    return Response(status=status.HTTP_204_NO_CONTENT)  # no body on 204
```

```python
# books/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("books/", views.book_list),
    path("books/<int:pk>/", views.book_detail),
]
```

**Why use it:** great for one-off, irregular endpoints (a webhook receiver, a `/health` check, a report). **Why avoid it:** you hand-write the same CRUD boilerplate every time — DRF has better tools for standard resources.

---

## Core concept: `APIView` (class-based views)

`APIView` is the class equivalent: one method per HTTP verb. It's the base class everything else in DRF builds on. You get the same `Request`/`Response`, plus class-level hooks for `permission_classes`, `authentication_classes`, and `throttle_classes`.

```python
# books/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.http import Http404
from .models import Book
from .serializers import BookSerializer


class BookListAPIView(APIView):
    def get(self, request):
        serializer = BookSerializer(Book.objects.all(), many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = BookSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)


class BookDetailAPIView(APIView):
    def get_object(self, pk):
        try:
            return Book.objects.get(pk=pk)
        except Book.DoesNotExist:
            raise Http404

    def get(self, request, pk):
        return Response(BookSerializer(self.get_object(pk)).data)

    def put(self, request, pk):
        book = self.get_object(pk)
        serializer = BookSerializer(book, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    def delete(self, request, pk):
        self.get_object(pk).delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

```python
# books/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("books/", views.BookListAPIView.as_view()),
    path("books/<int:pk>/", views.BookDetailAPIView.as_view()),
]
```

Note `.as_view()` — same pattern as Django's class-based views. `APIView` is the right level when your endpoint has custom flow that doesn't fit the CRUD mold but you still want class-based structure (mixins, shared `permission_classes`, etc.).

---

## Core concept: Generic views & mixins (the sweet spot)

You've now written `Book.objects.all()` + `BookSerializer` + validate + save *twice*. DRF's **generic views** capture that pattern. You declare two attributes — `queryset` and `serializer_class` — and the class fills in the verbs.

The two attributes mean:

- **`queryset`** — the base set of objects this view operates on (`Book.objects.all()`).
- **`serializer_class`** — which serializer to use for input validation and output rendering.

The whole CRUD collapses to:

```python
# books/views.py
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer


class BookListCreate(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer


class BookDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

That's a complete, validated, paginated REST resource. `ListCreateAPIView` handles `GET` (list) and `POST` (create); `RetrieveUpdateDestroyAPIView` handles `GET` (one), `PUT`, `PATCH`, and `DELETE`. URLs are the same as the `APIView` example (`.as_view()` + `<int:pk>/`).

### The generic-view menu

Generic views are just **mixins** (`ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`) glued onto `GenericAPIView`. The pre-built combos:

| Generic view | HTTP methods | Use for |
|---|---|---|
| `ListAPIView` | GET (many) | read-only collection |
| `CreateAPIView` | POST | create-only |
| `RetrieveAPIView` | GET (one) | read-only single item |
| `UpdateAPIView` | PUT / PATCH | update-only |
| `DestroyAPIView` | DELETE | delete-only |
| `ListCreateAPIView` | GET / POST | collection endpoint |
| `RetrieveUpdateAPIView` | GET / PUT / PATCH | item, no delete |
| `RetrieveDestroyAPIView` | GET / DELETE | item, read + delete |
| `RetrieveUpdateDestroyAPIView` | GET / PUT / PATCH / DELETE | full single-item endpoint |

Mix your own when no combo fits:

```python
from rest_framework import mixins, generics


class BookListCreate(mixins.ListModelMixin,
                     mixins.CreateModelMixin,
                     generics.GenericAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

(The pre-built `ListCreateAPIView` *is* exactly this — but writing it out shows there's no magic.)

### Overriding `get_queryset()` — per-request data

The `queryset` attribute is static: evaluated once at startup. The moment you need the data to depend on the **request** (the logged-in user, query params, the URL), override **`get_queryset()`** instead.

```python
class MyBookList(generics.ListAPIView):
    serializer_class = BookSerializer

    def get_queryset(self):
        qs = Book.objects.all()
        # Only this user's books:
        qs = qs.filter(owner=self.request.user)
        # Optional ?year=2020 filter:
        year = self.request.query_params.get("year")
        if year:
            qs = qs.filter(published_year=year)
        return qs
```

**Rule of thumb:** static list → `queryset`; anything dynamic → `get_queryset()`. (You still usually keep a `queryset` attribute too, because DRF uses it to infer the `basename` for routers and for permission checks. If you omit it, set `basename` on the router.)

### Overriding `get_serializer_class()` — different shape per situation

Return different serializers for, say, list vs detail, or read vs write, or by user role.

```python
class BookList(generics.ListCreateAPIView):
    queryset = Book.objects.all()

    def get_serializer_class(self):
        if self.request.method == "POST":
            return BookWriteSerializer   # strict input
        return BookListSerializer        # lean output
```

### Customizing save with `perform_create()`

Don't override `post()` just to inject a value at save time — use the hook:

```python
class BookListCreate(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)  # stamp the current user
```

There are matching `perform_update(serializer)` and `perform_destroy(instance)` hooks.

---

## Core concept: ViewSets & `ModelViewSet`

A **ViewSet** groups the logic for a whole resource into *one class*. Instead of HTTP-verb methods (`get`, `post`), a ViewSet has **actions**: `list`, `create`, `retrieve`, `update`, `partial_update`, `destroy`. A **router** then maps URLs+methods onto those actions for you.

`ModelViewSet` is the all-in-one: full CRUD from the same two attributes you already know.

```python
# books/views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer


class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

That *single class* is a complete CRUD API. All the overrides you learned still apply: `get_queryset()`, `get_serializer_class()`, `perform_create()`, `permission_classes`, etc.

### How methods map to actions

This mapping is the heart of ViewSets — memorize it for interviews:

| HTTP method | URL | Action | Laravel equivalent |
|---|---|---|---|
| GET | `/books/` | `list` | `index` |
| POST | `/books/` | `create` | `store` |
| GET | `/books/{pk}/` | `retrieve` | `show` |
| PUT | `/books/{pk}/` | `update` | `update` |
| PATCH | `/books/{pk}/` | `partial_update` | `update` |
| DELETE | `/books/{pk}/` | `destroy` | `destroy` |

`PUT` = full replacement (all fields required); `PATCH` = partial (only the fields you send). DRF wires both to your serializer automatically — `partial_update` just passes `partial=True`.

### ViewSet variants

| Class | Gives you |
|---|---|
| `ViewSet` | Bare base — you write each action by hand (like `APIView` but action-named). |
| `GenericViewSet` | `queryset`/`serializer_class` + generic helpers, but **no** actions — add via mixins. |
| `ReadOnlyModelViewSet` | `list` + `retrieve` only (no writes). |
| `ModelViewSet` | The full CRUD set. |

`ReadOnlyModelViewSet` is perfect for reference data clients should only read:

```python
class CountryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Country.objects.all()
    serializer_class = CountrySerializer
```

### Custom routes with `@action`

CRUD doesn't cover everything — "publish this book", "recent books", "set password". The **`@action`** decorator adds extra endpoints *to the ViewSet* and the router still generates their URLs.

- `detail=True` → operates on one object → URL `/books/{pk}/<name>/`.
- `detail=False` → operates on the collection → URL `/books/<name>/`.

```python
# books/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer


class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    @action(detail=True, methods=["post"])
    def publish(self, request, pk=None):
        book = self.get_object()
        book.is_published = True
        book.save(update_fields=["is_published"])
        return Response({"status": "published"}, status=status.HTTP_200_OK)
        # URL: POST /books/5/publish/

    @action(detail=False, methods=["get"])
    def recent(self, request):
        recent = self.get_queryset().order_by("-published_year")[:5]
        serializer = self.get_serializer(recent, many=True)
        return Response(serializer.data)
        # URL: GET /books/recent/

    # Per-action serializer? Use get_serializer_class:
    def get_serializer_class(self):
        if self.action == "recent":
            return BookSerializer  # could be a lighter one
        return super().get_serializer_class()
```

Note `self.action` — inside a ViewSet you always know which action is running, which makes `get_serializer_class`/`get_queryset`/permission logic clean.

---

## Core concept: Routers — automatic URL generation

You *register* a ViewSet with a router and it produces all the URL patterns. This is the `apiResource` moment.

```python
# books/urls.py
from rest_framework.routers import DefaultRouter
from .views import BookViewSet

router = DefaultRouter()
router.register(r"books", BookViewSet, basename="book")

urlpatterns = router.urls
```

From that one `register()` call you get:

```
^books/$                 -> {GET: list, POST: create}            name="book-list"
^books/{pk}/$            -> {GET: retrieve, PUT: update,
                              PATCH: partial_update, DELETE: destroy}  name="book-detail"
^books/{pk}/publish/$    -> {POST: publish}                       name="book-publish"
^books/recent/$          -> {GET: recent}                         name="book-recent"
```

`@action` endpoints are auto-included. The URL names (`book-list`, `book-detail`, ...) come from the `basename` — useful for `reverse()` and `HyperlinkedModelSerializer`.

### `DefaultRouter` vs `SimpleRouter`

| | `SimpleRouter` | `DefaultRouter` |
|---|---|---|
| CRUD + `@action` routes | yes | yes |
| API **root** view (`/`) listing all registered resources | no | yes |
| **Format suffixes** (`.json`) support | no | yes |
| Trailing slash | yes (configurable) | yes (configurable) |

Use `DefaultRouter` in development for the browsable root index; `SimpleRouter` is leaner for pure APIs. Include router URLs under a prefix like any other URLConf:

```python
# project/urls.py
from django.urls import path, include

urlpatterns = [
    path("api/v1/", include("books.urls")),   # -> /api/v1/books/
]
```

### When you DON'T use a router (generics/APIView)

Routers only understand ViewSets. Plain `APIView`/generic views are wired the normal Django way with `path()` + `.as_view()` (as shown earlier). You can mix both styles in one project — router URLs for resource ViewSets, hand-written `path()`s for irregular endpoints.

---

## Core concept: Choosing APIView vs Generics vs ViewSets

There is no single "best" — pick by how standard the endpoint is.

```
Is it a standard CRUD resource on a model?
├─ No / highly custom flow ............... @api_view  or  APIView
├─ Yes, ONE specific operation ........... a generic view (ListAPIView, etc.)
├─ Yes, a few operations, no full CRUD ... ListCreateAPIView / RetrieveUpdate...
└─ Yes, FULL CRUD (+ maybe extra routes) . ModelViewSet + router
```

| Approach | Boilerplate | Control | Best when |
|---|---|---|---|
| `@api_view` | high | total | one-off, non-CRUD (webhooks, health, RPC-ish) |
| `APIView` | high | total | custom flow, want classes/permissions |
| Generic views | low | medium | a model resource, you want explicit per-endpoint URLs |
| `ViewSet` / `ModelViewSet` | lowest | medium | full CRUD resources, consistent URL conventions, routers |

Mental model: **ModelViewSet = generic views fused into one class + a router that writes your URLs.** Reach for it by default for model CRUD; drop down to generics or `APIView` the moment the endpoint stops being a clean resource.

---

## Core concept: Status codes & `Response`

Never hard-code integers — use the named constants in `rest_framework.status` so your intent is readable. DRF defaults `Response` to `200 OK`, so you only set `status=` when it differs.

```python
from rest_framework import status

status.HTTP_200_OK              # GET / PUT / PATCH success
status.HTTP_201_CREATED        # POST that created a resource
status.HTTP_204_NO_CONTENT     # DELETE success (no body!)
status.HTTP_400_BAD_REQUEST    # validation error (DRF raises this for you)
status.HTTP_401_UNAUTHORIZED   # not authenticated
status.HTTP_403_FORBIDDEN      # authenticated but not allowed
status.HTTP_404_NOT_FOUND      # get_object_or_404 / generics raise this
status.HTTP_405_METHOD_NOT_ALLOWED  # verb not handled by the view
status.HTTP_409_CONFLICT       # e.g. unique-constraint clash
```

What DRF does automatically when you use generics/ViewSets: `is_valid(raise_exception=True)` (and the implicit validation in generics) yields a clean **400** with a field-keyed error body; a missing object yields **404**; an unhandled verb yields **405**. You rarely set status codes by hand once you're on generics/ViewSets — the framework already gets the common ones right.

---

## Common mistakes & gotchas

- **Returning `204` with a body.** `HTTP_204_NO_CONTENT` means *no content* — don't pass data; some clients choke on it.
- **Using `request.POST` instead of `request.data`.** `request.POST` only holds form-encoded data and is empty for JSON bodies. Always use `request.data` in DRF.
- **`queryset` vs `get_queryset()` confusion.** The `queryset` attribute is evaluated **once** at import time. Putting request-dependent logic (e.g. `request.user`) there silently uses stale/wrong data — override `get_queryset()` instead.
- **Forgetting `many=True`.** Serializing a queryset without `many=True` raises errors or returns garbage. Lists need `many=True`; single objects don't.
- **Calling `.as_view()` on a ViewSet without an action map.** ViewSets aren't wired by `.as_view()` like normal views — let a **router** do it, or call `BookViewSet.as_view({"get": "list"})` explicitly if you must.
- **Registering a ViewSet without `basename` when `queryset` is absent.** If you only override `get_queryset()` and drop the `queryset` attribute, the router can't infer the name — pass `basename="book"` to `register()`.
- **PUT vs PATCH.** `PUT` expects the *full* object (omitted fields fail required-validation); `PATCH` is partial. Clients hitting `PUT` with a partial body get 400s — point them at `PATCH`.
- **`@action` method name collisions.** The action's function name becomes the URL segment. Naming an action `list` or `update` shadows the built-ins — pick distinct names.
- **Mixing routers with manual `path()` for the same ViewSet.** Pick one wiring style per resource; double-registration produces confusing duplicate routes.
- **Leaving `BrowsableAPIRenderer` enabled in production** without thought — it's handy but exposes a writable HTML UI; gate it or remove it in prod settings.

## Best practices

- **Default to `ModelViewSet` + `DefaultRouter`** for model-backed CRUD; it's the least code and the most conventional. Step down to generics or `APIView` only when the resource shape demands it.
- **Keep views thin.** Validation lives in serializers; business rules in services/models. A view should orchestrate, not compute.
- **Override hooks, not whole methods.** Prefer `get_queryset`, `get_serializer_class`, `perform_create/update/destroy` over re-implementing `list`/`create` — you keep DRF's pagination, filtering, and error handling for free.
- **Always scope `get_queryset()` to the requesting user** for multi-tenant data — it's your last line of defense against leaking other users' rows.
- **Use named `status` constants**, never raw integers.
- **Use `@action` for verbs that aren't CRUD** (`publish`, `archive`, `set_password`) instead of inventing extra ViewSets or overloading `update`.
- **Version your API via the URL include** (`api/v1/`) so routers stay clean.
- **Set `basename` explicitly** when you customize `get_queryset()` — don't rely on inference.

## Interview Q&A

**Q: What's the difference between `request.data` and `request.POST` in DRF?**
**A:** `request.data` is DRF's unified, parser-driven view of the request body — it works for JSON, form-encoded, and multipart, and is available for any HTTP method (PUT/PATCH included). `request.POST` is Django's form-only dict and is empty for JSON payloads. In DRF you almost always use `request.data`.

**Q: Explain content negotiation in DRF.**
**A:** Based on the request's `Accept` header (and optional URL format suffix), DRF selects a renderer to serialize your native-Python response data — JSON for API clients, the browsable HTML API for browsers. The same logic in reverse (via `Content-Type` and parsers) turns the request body into `request.data`. This lets one view serve multiple formats without code changes.

**Q: When would you choose `APIView` over a generic view?**
**A:** When the endpoint isn't a clean model-CRUD operation — custom multi-step flows, non-model responses, or unusual method handling — but you still want class-based structure (shared `permission_classes`, mixins). Generic views assume the standard "queryset + serializer" CRUD pattern; `APIView` makes no such assumption.

**Q: What do `queryset` and `serializer_class` do on a generic view?**
**A:** `queryset` is the base set of objects the view reads/writes; `serializer_class` is the serializer used for validating input and rendering output. From just these two, a generic view (or ModelViewSet) can implement full CRUD.

**Q: `queryset` attribute vs `get_queryset()` — when do you override the method?**
**A:** The `queryset` attribute is evaluated once at import time, so it can't depend on the request. Override `get_queryset()` whenever the data depends on runtime context — the logged-in user, query params, or URL kwargs (e.g. filtering to `owner=self.request.user`).

**Q: What is a ViewSet, and how does it differ from a view?**
**A:** A ViewSet groups all the logic for a resource into one class organized by *actions* (`list`, `create`, `retrieve`, `update`, `partial_update`, `destroy`) rather than HTTP-verb methods. A router maps URLs and HTTP methods onto those actions, so you don't write a URLConf by hand. `ModelViewSet` provides the full CRUD set from `queryset` + `serializer_class`.

**Q: How do HTTP methods map to ViewSet actions?**
**A:** GET `/items/` → `list`, POST `/items/` → `create`, GET `/items/{pk}/` → `retrieve`, PUT `/items/{pk}/` → `update`, PATCH `/items/{pk}/` → `partial_update`, DELETE `/items/{pk}/` → `destroy`.

**Q: How do you add a non-CRUD endpoint to a ViewSet?**
**A:** With the `@action` decorator. `detail=True` makes it operate on a single object (`/items/{pk}/name/`), `detail=False` on the collection (`/items/name/`); you specify `methods=[...]`. The router auto-generates the URL and a route name.

**Q: `DefaultRouter` vs `SimpleRouter`?**
**A:** Both generate CRUD and `@action` routes. `DefaultRouter` additionally provides an API root view listing all registered resources and supports `.json`-style format suffixes; `SimpleRouter` omits those. Use `DefaultRouter` for a discoverable browsable API, `SimpleRouter` for a lean machine API.

**Q: What's the difference between PUT and PATCH in DRF, and how are they handled?**
**A:** PUT is a full update — all required serializer fields must be present or validation fails. PATCH is a partial update; DRF invokes the serializer with `partial=True` so only the supplied fields are validated and changed. In a ViewSet these correspond to the `update` and `partial_update` actions.

**Q: Where should you stamp the current user when creating an object?**
**A:** In `perform_create(self, serializer)` via `serializer.save(owner=self.request.user)`. Overriding this hook keeps DRF's validation/response flow intact instead of re-implementing `create()`.

**Q: How do you wire a ViewSet to URLs without a router?**
**A:** Call `as_view()` with an explicit action map per route: `BookViewSet.as_view({"get": "list", "post": "create"})` for the collection and `BookViewSet.as_view({"get": "retrieve", "put": "update", "patch": "partial_update", "delete": "destroy"})` for the detail URL. Routers automate exactly this.

## Practice exercises

1. **Function view:** Write an `@api_view(["GET"])` endpoint `/stats/` that returns `{"total_books": N, "published": M}` as JSON using the `Book` model. Use correct status handling.

2. **Generics:** Replace a pair of hand-written `APIView`s with `ListCreateAPIView` and `RetrieveUpdateDestroyAPIView` for `Book`. Then override `get_queryset()` so the list supports an optional `?author=` filter.

3. **Per-action serializer:** Given `BookListSerializer` (lean) and `BookSerializer` (full), make a `ModelViewSet` return the lean one for `list` and the full one everywhere else, using `get_serializer_class()` and `self.action`.

4. **Custom action:** Add a `@action` named `toggle_published` (detail=True, POST) to a `BookViewSet` that flips `is_published` and returns the new value. State the URL it produces.

5. **Routers:** Register `BookViewSet` and a read-only `AuthorViewSet` on a `DefaultRouter`, include them under `api/v1/`, and list every URL pattern the router generates.

6. **Decision drill:** For each, name the best tool (and why): (a) a Stripe webhook receiver, (b) read-only list of countries, (c) full CRUD for `Order` with an extra "cancel" endpoint, (d) a single endpoint that returns one user's dashboard aggregation.

<details><summary>Solutions</summary>

**1. Function view**

```python
# books/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Book


@api_view(["GET"])
def stats(request):
    qs = Book.objects.all()
    data = {
        "total_books": qs.count(),
        "published": qs.filter(is_published=True).count(),
    }
    return Response(data)  # 200 OK by default
```

```python
# books/urls.py
from django.urls import path
from . import views

urlpatterns = [path("stats/", views.stats)]
```

**2. Generics with filtered `get_queryset`**

```python
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer


class BookListCreate(generics.ListCreateAPIView):
    serializer_class = BookSerializer
    queryset = Book.objects.all()  # kept for basename / permission inference

    def get_queryset(self):
        qs = Book.objects.all()
        author = self.request.query_params.get("author")
        if author:
            qs = qs.filter(author__iexact=author)
        return qs


class BookDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

**3. Per-action serializer**

```python
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer, BookListSerializer


class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    def get_serializer_class(self):
        if self.action == "list":
            return BookListSerializer
        return super().get_serializer_class()
```

**4. Custom action**

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer


class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    @action(detail=True, methods=["post"])
    def toggle_published(self, request, pk=None):
        book = self.get_object()
        book.is_published = not book.is_published
        book.save(update_fields=["is_published"])
        return Response({"is_published": book.is_published})
        # URL produced: POST /books/{pk}/toggle_published/
```

**5. Routers**

```python
# views.py
from rest_framework import viewsets
from .models import Book, Author
from .serializers import BookSerializer, AuthorSerializer


class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer


class AuthorViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer
```

```python
# books/urls.py
from rest_framework.routers import DefaultRouter
from .views import BookViewSet, AuthorViewSet

router = DefaultRouter()
router.register(r"books", BookViewSet, basename="book")
router.register(r"authors", AuthorViewSet, basename="author")
urlpatterns = router.urls
```

```python
# project/urls.py
from django.urls import path, include
urlpatterns = [path("api/v1/", include("books.urls"))]
```

Generated URLs:
```
/api/v1/                          API root (DefaultRouter only)
/api/v1/books/        GET, POST   -> list, create
/api/v1/books/{pk}/   GET,PUT,PATCH,DELETE -> retrieve, update, partial_update, destroy
/api/v1/authors/      GET         -> list   (read-only)
/api/v1/authors/{pk}/ GET         -> retrieve (read-only)
```

**6. Decision drill**

- (a) Stripe webhook → **`@api_view(["POST"])`** (or `APIView`): non-CRUD, custom flow, often custom auth/parsing.
- (b) Read-only countries → **`ReadOnlyModelViewSet`** + router (or `ListAPIView` if you only need the list): no writes.
- (c) Full CRUD `Order` + cancel → **`ModelViewSet`** with an `@action(detail=True, methods=["post"]) def cancel` + router.
- (d) One-off dashboard aggregation → **`APIView`** (or `@api_view`): it returns computed data, not a model resource.

</details>

## Cheat sheet

```python
# --- Function view ---
@api_view(["GET", "POST"])
def view(request): ...

# --- Class view (one method per verb) ---
class V(APIView):
    def get(self, request): ...
    def post(self, request): ...

# --- Generic view (queryset + serializer_class) ---
class V(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    def get_queryset(self): ...           # request-dependent data
    def get_serializer_class(self): ...   # different serializer per case
    def perform_create(self, serializer): serializer.save(owner=self.request.user)

# --- ViewSet (one class, actions) ---
class V(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    @action(detail=True, methods=["post"])
    def publish(self, request, pk=None): ...

# --- Router ---
router = DefaultRouter()
router.register(r"books", BookViewSet, basename="book")
urlpatterns = router.urls
```

| Generic combo | Methods |
|---|---|
| `ListAPIView` | GET list |
| `ListCreateAPIView` | GET list, POST |
| `RetrieveAPIView` | GET one |
| `RetrieveUpdateAPIView` | GET, PUT, PATCH |
| `RetrieveUpdateDestroyAPIView` | GET, PUT, PATCH, DELETE |

| HTTP → ViewSet action | |
|---|---|
| GET `/x/` | `list` |
| POST `/x/` | `create` |
| GET `/x/{pk}/` | `retrieve` |
| PUT `/x/{pk}/` | `update` |
| PATCH `/x/{pk}/` | `partial_update` |
| DELETE `/x/{pk}/` | `destroy` |

| Status | Constant |
|---|---|
| 200 | `status.HTTP_200_OK` |
| 201 | `status.HTTP_201_CREATED` |
| 204 | `status.HTTP_204_NO_CONTENT` |
| 400 | `status.HTTP_400_BAD_REQUEST` |
| 401 / 403 | `HTTP_401_UNAUTHORIZED` / `HTTP_403_FORBIDDEN` |
| 404 / 405 | `HTTP_404_NOT_FOUND` / `HTTP_405_METHOD_NOT_ALLOWED` |

**Decision:** non-CRUD → `@api_view`/`APIView` · one model op → generic view · full CRUD (+extras) → `ModelViewSet` + router.

**Key overrides:** `get_queryset()` (dynamic data) · `get_serializer_class()` (shape per case) · `perform_create/update/destroy()` (save-time logic) · `@action` (custom routes) · `self.action` (which action is running).
