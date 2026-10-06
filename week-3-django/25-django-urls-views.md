# 25 — URLs & Views (FBV & CBV)

> Week 3 · Django routing and request handling

## What you will learn

- How Django's **URLconf** (`urls.py`) routes an incoming request to a view
- `path()` with **path converters** (`<int:id>`, `<slug:slug>`, `<str:name>`, `<uuid:>`) and when to reach for `re_path()` (regex)
- Splitting routes per app with `include()` and keeping the project tidy
- **Naming** URLs and reversing them with `reverse()`, `reverse_lazy()`, and `{% url %}` — never hardcode a path again
- **Function-based views (FBV):** take an `HttpRequest`, return an `HttpResponse` / `JsonResponse`
- The `HttpRequest` object in depth: `GET`, `POST`, `method`, `headers`, `body`, `path`
- **Class-based views (CBV):** why they exist, how `View.dispatch()` works, and `as_view()`
- **Generic class-based views** (`ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`) at a glance
- Useful **view decorators**: `require_http_methods`, `require_POST`, `login_required`
- A side-by-side map from **Laravel routes + controllers** to Django

## PHP/Laravel bridge

In Laravel you register routes in `routes/web.php` / `routes/api.php` and point each one at a controller method: `Route::get('/posts/{id}', [PostController::class, 'show'])->name('posts.show')`. Django's `urls.py` is the direct analogue of those route files, a **view** is the analogue of a controller action, and Django's `path('posts/<int:id>/', views.show, name='posts.show')` mirrors Laravel's route definition almost one-to-one — including **named routes**. Where Laravel gives you `route('posts.show', $id)` and the `{{ route() }}` Blade helper, Django gives you `reverse('posts.show', args=[id])` and the `{% url %}` template tag. The big mental shift: Django splits routing across multiple `urls.py` files (one per app) wired together with `include()`, instead of one big route file, and a Django "view" is just a callable — it can be a plain function or a class.

## Core concept: the URLconf and how a request is routed

When a request comes in, Django reads `ROOT_URLCONF` from settings (usually `<project>/urls.py`), walks the `urlpatterns` list **top to bottom**, and stops at the **first pattern that matches** the path. That matched view is called with the `HttpRequest` plus any captured URL parameters.

```python
# config/settings.py  (the entry point Django uses to find routes)
ROOT_URLCONF = "config.urls"
```

```python
# config/urls.py  — the project-level URLconf
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("blog/", include("blog.urls")),   # delegate everything under /blog/ to the blog app
    path("api/", include("api.urls")),
]
```

Key points a Laravel dev should internalize:

- `urlpatterns` is an ordinary Python **list**. Order matters — the first match wins (like Laravel, but here it is literally list order).
- Trailing slashes matter. Django convention is to **end paths with `/`**. With `APPEND_SLASH = True` (the default), a request to `/blog` is redirected to `/blog/`.
- A "view" is any callable that takes `request` and returns an `HttpResponse`. Nothing magic.

## Core concept: `path()` and path converters

`path(route, view, kwargs=None, name=None)` is the workhorse. The `route` is a plain string; the `<converter:name>` syntax captures a segment, converts it to a Python type, and passes it to the view as a keyword argument.

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = "blog"   # namespaces this app's URL names -> referenced as "blog:detail"

urlpatterns = [
    path("", views.post_list, name="list"),                 # /blog/
    path("<int:post_id>/", views.post_detail, name="detail"),     # /blog/42/
    path("<slug:slug>/", views.post_by_slug, name="by-slug"),     # /blog/my-first-post/
    path("author/<str:username>/", views.by_author, name="by-author"),  # /blog/author/alice/
    path("uuid/<uuid:pk>/", views.by_uuid, name="by-uuid"),
]
```

The captured value arrives as a **keyword argument** whose name matches the converter label:

```python
# blog/views.py
from django.http import HttpResponse

def post_detail(request, post_id):       # 'post_id' matches <int:post_id>
    # post_id is already an int, not a string — the int converter cast it
    return HttpResponse(f"Post #{post_id} (type: {type(post_id).__name__})")
    # GET /blog/42/  =>  Post #42 (type: int)
```

### Built-in path converters

| Converter | Matches | Python type | Example route | Matches URL |
|-----------|---------|-------------|---------------|-------------|
| `str`     | any non-empty string **without** `/` | `str` | `<str:name>` | `/author/alice/` |
| `int`     | zero or positive integer | `int` | `<int:id>` | `/posts/42/` |
| `slug`    | ASCII letters, digits, hyphens, underscores | `str` | `<slug:slug>` | `/my-post-title/` |
| `uuid`    | a UUID like `075194d3-...` | `uuid.UUID` | `<uuid:pk>` | `/items/075194d3-6885-417e-a8a8-6c931e272f00/` |
| `path`    | any string **including** `/` | `str` | `<path:subpath>` | `/files/a/b/c.txt` |

> Gotcha: `<str:>` does **not** match a `/`. If you need to capture slashes (file paths, nested keys), use `<path:>`. And `<int:>` will not match a negative number — it is zero-or-positive only.

### Custom path converters

When the built-ins are not enough (e.g. a 4-digit year), register your own converter:

```python
# blog/converters.py
class FourDigitYearConverter:
    regex = r"[0-9]{4}"             # what URL text it matches

    def to_python(self, value):    # URL string -> Python value passed to the view
        return int(value)

    def to_url(self, value):       # Python value -> URL string (used by reverse())
        return f"{value:04d}"
```

```python
# blog/urls.py
from django.urls import path, register_converter
from . import converters, views

register_converter(converters.FourDigitYearConverter, "yyyy")

urlpatterns = [
    path("archive/<yyyy:year>/", views.year_archive, name="year-archive"),  # /blog/archive/2026/
]
```

## Core concept: `re_path()` for regular expressions

`path()` covers almost everything, but when you need real regex power (optional groups, alternations, complex patterns) use `re_path()`. Captured **named groups** `(?P<name>...)` become view keyword arguments.

```python
# blog/urls.py
from django.urls import re_path
from . import views

urlpatterns = [
    # /blog/2026/06/  -> year=2026, month=06
    re_path(r"^(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/$", views.month_archive, name="month-archive"),
]
```

```python
def month_archive(request, year, month):
    # NOTE: with re_path, captured groups arrive as *strings* — no automatic int casting
    return HttpResponse(f"Archive for {year}-{month}")
```

> Prefer `path()`. Reach for `re_path()` only when a converter cannot express the pattern. A common bite: `re_path` does **not** cast types, so `year` above is `"2026"` (a string), unlike `<int:year>` with `path()`.

## Core concept: `include()` — splitting URLs per app

A big project should not have one giant `urls.py`. Each app owns a `urls.py`, and the project wires them in with `include()`. This mirrors how you might split Laravel route files, but it is the idiomatic default in Django.

```python
# config/urls.py
from django.urls import path, include

urlpatterns = [
    path("blog/", include("blog.urls")),     # blog/urls.py owns everything under /blog/
    path("shop/", include("shop.urls")),
]
```

- The prefix (`blog/`) is stripped before the included URLconf sees the path, so `blog/urls.py` patterns are written **relative** to that prefix.
- Pair `include()` with `app_name = "blog"` inside the app's `urls.py` to get a **namespace**, so names never collide between apps (`blog:detail` vs `shop:detail`).

## Core concept: naming URLs and reversing them

Never hardcode a URL string in your code or templates. Give each route a `name`, then build the URL from that name. If the path ever changes, every reference updates automatically. This is exactly Laravel's named routes + `route()` helper.

```python
# Python code: reverse() turns a name (+ args) into a path string
from django.urls import reverse

reverse("blog:detail", kwargs={"post_id": 42})   # => "/blog/42/"
reverse("blog:detail", args=[42])                 # => "/blog/42/"  (positional also works)
reverse("blog:list")                               # => "/blog/"
```

```python
# Redirecting using a name (don't build the URL by hand)
from django.shortcuts import redirect

def create_post(request):
    # ... save a post ...
    return redirect("blog:detail", post_id=42)   # 302 to /blog/42/
```

```django
{# In a template: the {% url %} tag is Django's equivalent of Blade's route() #}
<a href="{% url 'blog:detail' post_id=post.id %}">{{ post.title }}</a>
{# renders: <a href="/blog/42/">...</a> #}
```

### `reverse_lazy()` — for module-load time

`reverse()` runs immediately, which fails if the URLconf is not loaded yet (e.g. a class attribute evaluated at import time). Use `reverse_lazy()`, which defers resolution until the value is actually used. You will hit this constantly with CBVs that set `success_url`.

```python
from django.urls import reverse_lazy
from django.views.generic import CreateView
from .models import Post

class PostCreateView(CreateView):
    model = Post
    fields = ["title", "body"]
    # reverse() here would run at import time and may crash; reverse_lazy() defers it:
    success_url = reverse_lazy("blog:list")
```

| You want… | Use |
|-----------|-----|
| Build a URL inside a view/function (runtime) | `reverse(...)` |
| Build a URL as a class attribute / setting / at import time | `reverse_lazy(...)` |
| Build a URL in a template | `{% url 'name' arg %}` |
| Redirect to a named route | `redirect('name', arg)` |

## Core concept: function-based views (FBV)

The simplest view is a function: it receives an `HttpRequest` as its first argument and returns an `HttpResponse`. This is the closest thing to a Laravel closure/controller method.

```python
# blog/views.py
from django.http import HttpResponse, JsonResponse, Http404
from django.shortcuts import get_object_or_404, render
from .models import Post

def post_list(request):
    posts = Post.objects.all()
    # render() = load template + context -> HttpResponse (like Laravel's view())
    return render(request, "blog/post_list.html", {"posts": posts})

def post_detail(request, post_id):
    post = get_object_or_404(Post, pk=post_id)   # 404 if missing — no manual check needed
    return render(request, "blog/post_detail.html", {"post": post})

def health(request):
    return HttpResponse("OK", content_type="text/plain", status=200)
```

### Returning JSON

For APIs, return `JsonResponse`. Pass `safe=False` if the top-level object is a list rather than a dict.

```python
from django.http import JsonResponse

def post_json(request, post_id):
    post = get_object_or_404(Post, pk=post_id)
    return JsonResponse({"id": post.id, "title": post.title})
    # => {"id": 42, "title": "Hello"}  with Content-Type: application/json

def posts_json(request):
    data = list(Post.objects.values("id", "title"))   # a list of dicts
    return JsonResponse(data, safe=False)              # safe=False allows a top-level list
```

> For real REST APIs you will use **Django REST Framework** (later lessons). `JsonResponse` is fine for small endpoints, but DRF gives you serializers, validation, content negotiation, and browsable docs.

## Core concept: the `HttpRequest` object

Every view's first argument is the request. The most-used attributes:

```python
def inspect(request):
    request.method        # "GET", "POST", "PUT", "DELETE", ...
    request.GET           # QueryDict of query-string params  (?q=django&page=2)
    request.POST          # QueryDict of form-encoded body params
    request.path          # "/blog/42/"  (path only, no query string)
    request.headers       # case-insensitive dict-like: request.headers["User-Agent"]
    request.COOKIES       # dict of cookies
    request.body          # raw bytes of the body (use for JSON payloads)
    request.user          # the logged-in user (AnonymousUser if not authenticated)
    request.FILES         # uploaded files
    return HttpResponse("ok")
```

`request.GET` and `request.POST` are **`QueryDict`** objects (immutable, multi-value aware):

```python
def search(request):
    q = request.GET.get("q", "")            # safe lookup with default; "" if missing
    page = int(request.GET.get("page", 1))  # query params are always strings — cast yourself
    tags = request.GET.getlist("tag")       # ?tag=a&tag=b  => ["a", "b"]
    return JsonResponse({"q": q, "page": page, "tags": tags})
```

Reading a **JSON request body** (common for APIs — JSON does not populate `request.POST`):

```python
import json
from django.http import JsonResponse
from django.views.decorators.http import require_POST

@require_POST
def create_comment(request):
    try:
        payload = json.loads(request.body)   # request.body is raw bytes; parse it
    except json.JSONDecodeError:
        return JsonResponse({"error": "invalid JSON"}, status=400)
    text = payload.get("text", "")
    return JsonResponse({"created": True, "text": text}, status=201)
```

> Beginner trap: `request.POST` is **only** populated for form-encoded (`application/x-www-form-urlencoded` or `multipart/form-data`) bodies. A JSON body lives in `request.body` and you must `json.loads()` it yourself (DRF does this for you automatically).

## Core concept: view decorators

Decorators wrap a view to add cross-cutting behavior — the FBV equivalent of Laravel middleware applied per route.

```python
from django.views.decorators.http import require_http_methods, require_POST, require_GET
from django.contrib.auth.decorators import login_required
from django.http import JsonResponse

@require_http_methods(["GET", "POST"])   # 405 Method Not Allowed for anything else
def article(request):
    if request.method == "POST":
        return JsonResponse({"saved": True}, status=201)
    return JsonResponse({"items": []})

@require_POST                            # shortcut for require_http_methods(["POST"])
def submit(request):
    return JsonResponse({"ok": True})

@login_required                          # redirects anonymous users to LOGIN_URL
def dashboard(request):
    return JsonResponse({"user": request.user.username})
```

When you stack decorators, the **top one runs first** (outermost wrapper):

```python
@login_required          # checked first
@require_POST            # then method check
def delete_account(request):
    ...
```

## Core concept: class-based views (CBV) and why they exist

FBVs are great for simple endpoints, but they push you toward `if request.method == ...` ladders and copy-paste between views. **Class-based views** let you:

- Map HTTP methods to **methods on a class** (`get()`, `post()`, `put()`, `delete()`)
- **Reuse** behavior via inheritance and **mixins** (think: composable controller traits)
- Use Django's **generic views** to get list/detail/create/update/delete for free

The base `View` class turns method names into handlers. You wire a class into `urls.py` with `.as_view()`:

```python
# blog/views.py
from django.http import JsonResponse
from django.views import View

class ArticleView(View):
    def get(self, request, article_id):
        return JsonResponse({"id": article_id, "method": "GET"})

    def post(self, request, article_id):
        return JsonResponse({"id": article_id, "method": "POST"}, status=201)
```

```python
# blog/urls.py
from django.urls import path
from .views import ArticleView

urlpatterns = [
    path("articles/<int:article_id>/", ArticleView.as_view(), name="article"),
]
```

### How `dispatch()` works

`as_view()` returns a plain function (the thing `urls.py` actually stores). When a request arrives, that function instantiates the class and calls `dispatch()`. `dispatch()` looks at `request.method`, lowercases it, and calls the matching method (`get`, `post`, …). If the method is not defined, it returns **405 Method Not Allowed** automatically — no manual checks.

```python
# Conceptually, dispatch does this:
# handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
# return handler(request, *args, **kwargs)
```

You can override `dispatch()` to run logic for every method (auth, logging, timing):

```python
class TimedView(View):
    def dispatch(self, request, *args, **kwargs):
        # runs before get()/post() — like a per-view middleware hook
        response = super().dispatch(request, *args, **kwargs)
        response["X-Handled-By"] = self.__class__.__name__
        return response

    def get(self, request):
        return JsonResponse({"ok": True})
```

### Applying decorators to CBVs

Function decorators do not attach directly to a class method without help. Use `method_decorator`:

```python
from django.utils.decorators import method_decorator
from django.contrib.auth.decorators import login_required
from django.views import View

@method_decorator(login_required, name="dispatch")   # protect every method
class SecretView(View):
    def get(self, request):
        return JsonResponse({"secret": 42})
```

## Core concept: generic class-based views (overview)

Generic CBVs implement the patterns you write over and over. You configure them with a few class attributes instead of writing the body.

```python
# blog/views.py
from django.views.generic import (
    ListView, DetailView, CreateView, UpdateView, DeleteView,
)
from django.urls import reverse_lazy
from .models import Post

class PostListView(ListView):
    model = Post
    template_name = "blog/post_list.html"   # default would be blog/post_list.html anyway
    context_object_name = "posts"           # name used in the template (default: object_list)
    paginate_by = 10                         # free pagination!

class PostDetailView(DetailView):
    model = Post
    # expects <int:pk> or <slug:slug> in the URL; provides `object` to the template

class PostCreateView(CreateView):
    model = Post
    fields = ["title", "body"]               # auto-builds a ModelForm
    success_url = reverse_lazy("blog:list")  # where to go after a successful save

class PostUpdateView(UpdateView):
    model = Post
    fields = ["title", "body"]
    success_url = reverse_lazy("blog:list")

class PostDeleteView(DeleteView):
    model = Post
    success_url = reverse_lazy("blog:list")
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = "blog"
urlpatterns = [
    path("", views.PostListView.as_view(), name="list"),
    path("<int:pk>/", views.PostDetailView.as_view(), name="detail"),
    path("new/", views.PostCreateView.as_view(), name="create"),
    path("<int:pk>/edit/", views.PostUpdateView.as_view(), name="update"),
    path("<int:pk>/delete/", views.PostDeleteView.as_view(), name="delete"),
]
```

| Generic view | What it does | Key attributes |
|--------------|--------------|----------------|
| `ListView` | list of objects | `model`, `queryset`, `paginate_by`, `context_object_name` |
| `DetailView` | one object by `pk`/`slug` | `model`, `slug_field`, `context_object_name` |
| `CreateView` | form to create | `model`, `fields` (or `form_class`), `success_url` |
| `UpdateView` | form to edit existing | `model`, `fields`, `success_url` |
| `DeleteView` | confirm + delete | `model`, `success_url` |
| `TemplateView` | render a static template | `template_name` |
| `RedirectView` | permanent/temporary redirect | `url`, `pattern_name` |

> These are for **server-rendered HTML** apps. For JSON REST APIs you will use **DRF's generic views** (`ListAPIView`, `RetrieveAPIView`, `ViewSet`) — same philosophy, JSON output. Covered in the DRF lessons.

## Laravel vs Django: routes & controllers cheat-map

| Laravel | Django |
|---------|--------|
| `routes/web.php`, `routes/api.php` | `urls.py` (per project + per app) |
| `Route::get('/posts/{id}', [C::class,'show'])` | `path('posts/<int:id>/', views.show)` |
| `{id}` route param | `<int:id>` path converter |
| `->name('posts.show')` | `name='posts.show'` (or `app_name` namespace) |
| `route('posts.show', $id)` | `reverse('posts.show', args=[id])` |
| `{{ route('posts.show', $id) }}` (Blade) | `{% url 'posts.show' id %}` |
| `Route::prefix('blog')->group(...)` | `path('blog/', include('blog.urls'))` |
| Controller method `show(Request $r, $id)` | `def show(request, id):` |
| `request->input('q')` | `request.GET.get('q')` / `request.POST.get('q')` |
| `request->method()` | `request.method` |
| Single-action / resource controllers | Class-based / generic views |
| `Route::resource('posts', PostController::class)` | the 5 generic CRUD views wired up |
| Route middleware (`->middleware('auth')`) | `@login_required` / `method_decorator` |

## Common mistakes & gotchas

- **Forgetting the trailing slash.** Django paths conventionally end with `/`. Mixing styles leads to confusing redirects (`APPEND_SLASH`) or 404s.
- **Order of `urlpatterns` matters.** A broad pattern like `<str:slug>/` placed before `new/` will swallow `/new/` (it matches `slug="new"`). Put specific routes first.
- **`re_path` does not cast types.** `(?P<year>[0-9]{4})` gives you a string `"2026"`; `<int:year>` with `path()` gives an int. Don't assume integer math will work after `re_path`.
- **`<str:>` won't match slashes.** Use `<path:>` to capture segments containing `/`.
- **`JsonResponse` with a list needs `safe=False`.** Otherwise you get a `TypeError`. (It's a deliberate guard against a legacy JSON array hijacking vulnerability.)
- **JSON bodies are not in `request.POST`.** Parse `json.loads(request.body)` yourself for `application/json`.
- **Using `reverse()` where you need `reverse_lazy()`.** As a class attribute or setting, `reverse()` runs too early and raises. Use `reverse_lazy()`.
- **Mismatched capture name and view parameter.** `<int:post_id>` must be received as `post_id` (a keyword arg), not positionally named something else.
- **Decorating a CBV method directly.** Plain `@login_required` on a class method does not work; use `method_decorator`.
- **Forgetting `.as_view()`** when wiring a CBV into `urls.py`. You must pass the *result of* `as_view()`, not the class.
- **Forgetting `app_name`** then referencing `blog:detail` — you'll get a `NoReverseMatch`. Either set `app_name` or drop the namespace prefix.

## Best practices

- Always **name** your URLs and build links with `reverse()` / `{% url %}` — never hardcode paths.
- Use **`app_name`** + namespaced names (`"blog:detail"`) to keep names unique across apps.
- Keep `urls.py` **per app** and wire them with `include()`. The project `urls.py` should mostly be `include()` calls.
- Prefer **`path()` + converters**; only drop to `re_path()` when regex is genuinely required.
- Put **specific routes before greedy ones** in `urlpatterns`.
- Use **FBVs for one-off / simple endpoints**, **CBVs (especially generic ones) for standard CRUD**. Don't force everything into one style.
- Validate the HTTP method explicitly (`require_http_methods`) on FBVs that should not accept all verbs.
- For JSON APIs beyond toy endpoints, graduate to **Django REST Framework** rather than hand-rolling `JsonResponse`.
- Use `get_object_or_404()` instead of manual `try/except DoesNotExist` for cleaner 404 handling.

## Interview Q&A

**Q: What is a URLconf, and how does Django decide which view handles a request?**
**A:** A URLconf is the module pointed to by `ROOT_URLCONF` (usually `config/urls.py`) plus any included app `urls.py` files. It exposes a list named `urlpatterns`. Django walks that list top to bottom and dispatches to the **first** pattern whose route matches the request path, passing the `HttpRequest` and any captured URL parameters to that view.

**Q: What is the difference between `path()` and `re_path()`?**
**A:** `path()` uses simple route strings with typed converters like `<int:id>`, which both match and cast the value (e.g. to `int`). `re_path()` uses full regular expressions with named groups `(?P<name>...)`, but captured values arrive as **strings** with no casting. Prefer `path()`; use `re_path()` only when a converter can't express the pattern.

**Q: Name the built-in path converters and what they return.**
**A:** `str` (non-empty, no slash → `str`), `int` (zero or positive → `int`), `slug` (letters/digits/hyphens/underscores → `str`), `uuid` (→ `uuid.UUID`), and `path` (matches anything including slashes → `str`).

**Q: Why name URLs, and how do you reverse them?**
**A:** Naming decouples code/templates from concrete paths, so you can change a URL once and everything still works. In Python use `reverse('app:name', args=[...])` or `reverse('app:name', kwargs={...})`; in templates use `{% url 'app:name' arg %}`. Use `reverse_lazy()` when the URL is needed at import time (e.g. a CBV `success_url`).

**Q: What's the difference between `reverse()` and `reverse_lazy()`?**
**A:** `reverse()` resolves immediately and is for runtime use inside view bodies. `reverse_lazy()` returns a lazy object that resolves only when its string value is actually needed — necessary for class attributes, module-level constants, or settings evaluated before the URLconf is fully loaded.

**Q: What does `include()` do, and why split URLs per app?**
**A:** `include()` mounts another URLconf under a prefix; the prefix is stripped before the included patterns are matched, so apps define routes relative to their mount point. Splitting per app keeps routing modular, reusable, and namespaceable (with `app_name`), instead of one monolithic route file.

**Q: What is the difference between a function-based view and a class-based view?**
**A:** An FBV is a function taking `request` (plus URL kwargs) and returning an `HttpResponse`; you branch on `request.method` yourself. A CBV is a class whose methods (`get`, `post`, …) map to HTTP verbs via `dispatch()`; it enables inheritance, mixins, and Django's generic views. You wire a CBV into `urls.py` with `.as_view()`.

**Q: How does `View.dispatch()` work?**
**A:** `as_view()` returns a closure that, per request, instantiates the view and calls `dispatch()`. `dispatch()` lowercases `request.method`, looks up the matching method via `getattr`, and calls it; if there's no such method it returns `http_method_not_allowed` (HTTP 405). Overriding `dispatch()` lets you run code for every verb (auth, logging).

**Q: How do you return JSON from a view, and what's the `safe=False` gotcha?**
**A:** Use `JsonResponse(data)`, which sets `Content-Type: application/json` and serializes the dict. If the top-level value is a list, you must pass `safe=False` or it raises a `TypeError` — a guard against an old JSON-array hijacking issue. For substantial APIs, use DRF instead.

**Q: Where do query params, form data, and JSON bodies live on the request?**
**A:** Query-string params are in `request.GET` (a `QueryDict`); form-encoded body fields are in `request.POST`; the raw body (including JSON) is in `request.body` as bytes — parse it with `json.loads()`. The HTTP method is `request.method`, headers are in `request.headers`. Remember `request.POST` is **not** populated for JSON content types.

**Q: How do you restrict a view to certain HTTP methods?**
**A:** On an FBV, use `@require_http_methods(["GET", "POST"])` (or `@require_POST` / `@require_GET`); non-matching verbs get a 405. On a CBV, simply define only the handler methods you support — `dispatch()` returns 405 for the rest automatically.

**Q: How do you apply `login_required` (or other decorators) to a class-based view?**
**A:** Wrap with `method_decorator`: `@method_decorator(login_required, name="dispatch")` on the class to protect all verbs, or apply it to a specific method. Alternatively, for auth specifically, inherit from `LoginRequiredMixin`. A bare `@login_required` on a class method does not work.

## Practice exercises

1. **Routing basics.** In a `catalog` app, create routes for: a product list at `/catalog/`, a product detail at `/catalog/<int:product_id>/`, and a category page at `/catalog/category/<slug:slug>/`. Wire the app into the project URLconf under the `catalog/` prefix and give each route a namespaced name.

2. **Reverse it.** Given the routes above, write the Python expressions to produce `/catalog/7/` and `/catalog/category/electronics/` using `reverse()`. Then write the `{% url %}` tag for the detail link inside a template loop over `products`.

3. **FBV with method + JSON.** Write a function-based view `create_or_list_notes(request)` that returns a JSON list of notes on `GET` and, on `POST` with a JSON body `{"text": "..."}`, returns `{"created": true, "text": "..."}` with status 201. Reject other methods with 405 and return 400 for invalid JSON.

4. **Query params.** Write a view `search(request)` that reads `?q=` (default `""`), `?page=` (default `1`, cast to int), and repeated `?tag=` values, returning them as JSON.

5. **CBV conversion.** Convert exercise 3 into a class-based view `NoteView(View)` with `get()` and `post()` methods. Confirm that a `PUT` request returns 405 with no extra code.

6. **Generic view.** Replace a hand-written list view with `ListView` for a `Note` model: 15 per page, template context name `notes`.

<details><summary>Solutions</summary>

**1. Routing basics**

```python
# catalog/urls.py
from django.urls import path
from . import views

app_name = "catalog"
urlpatterns = [
    path("", views.product_list, name="list"),
    path("<int:product_id>/", views.product_detail, name="detail"),
    path("category/<slug:slug>/", views.category, name="category"),
]
```

```python
# config/urls.py
from django.urls import path, include
urlpatterns = [
    path("catalog/", include("catalog.urls")),
]
```

**2. Reverse it**

```python
from django.urls import reverse
reverse("catalog:detail", kwargs={"product_id": 7})         # => "/catalog/7/"
reverse("catalog:category", kwargs={"slug": "electronics"}) # => "/catalog/category/electronics/"
```

```django
{% for product in products %}
  <a href="{% url 'catalog:detail' product_id=product.id %}">{{ product.name }}</a>
{% endfor %}
```

**3. FBV with method + JSON**

```python
import json
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods

NOTES = []  # demo store

@require_http_methods(["GET", "POST"])   # other verbs -> 405 automatically
def create_or_list_notes(request):
    if request.method == "GET":
        return JsonResponse(NOTES, safe=False)
    try:
        payload = json.loads(request.body)
    except json.JSONDecodeError:
        return JsonResponse({"error": "invalid JSON"}, status=400)
    text = payload.get("text", "")
    NOTES.append({"text": text})
    return JsonResponse({"created": True, "text": text}, status=201)
```

**4. Query params**

```python
from django.http import JsonResponse

def search(request):
    q = request.GET.get("q", "")
    page = int(request.GET.get("page", 1))
    tags = request.GET.getlist("tag")
    return JsonResponse({"q": q, "page": page, "tags": tags})
```

**5. CBV conversion**

```python
import json
from django.http import JsonResponse
from django.views import View

class NoteView(View):
    notes = []  # demo store

    def get(self, request):
        return JsonResponse(self.notes, safe=False)

    def post(self, request):
        try:
            payload = json.loads(request.body)
        except json.JSONDecodeError:
            return JsonResponse({"error": "invalid JSON"}, status=400)
        text = payload.get("text", "")
        self.notes.append({"text": text})
        return JsonResponse({"created": True, "text": text}, status=201)
    # No put() defined -> dispatch() returns 405 for PUT requests, for free.
```

```python
# urls.py
path("notes/", NoteView.as_view(), name="notes")
```

**6. Generic view**

```python
from django.views.generic import ListView
from .models import Note

class NoteListView(ListView):
    model = Note
    paginate_by = 15
    context_object_name = "notes"
    # template defaults to <app>/note_list.html
```

```python
# urls.py
path("", NoteListView.as_view(), name="note-list")
```

</details>

## Cheat sheet

```python
# --- urls.py skeleton ---
from django.urls import path, re_path, include, register_converter

app_name = "blog"                                   # enables "blog:name" namespace
urlpatterns = [
    path("", views.list, name="list"),
    path("<int:id>/", views.detail, name="detail"),     # int converter -> int
    path("<slug:slug>/", views.by_slug, name="by-slug"),
    path("<uuid:pk>/", views.by_uuid, name="by-uuid"),
    path("files/<path:subpath>/", views.file, name="file"),  # matches slashes
    re_path(r"^(?P<y>[0-9]{4})/$", views.year, name="year"),  # regex; values are str
]
# project urls.py:  path("blog/", include("blog.urls"))
```

| Converter | → Python | Note |
|-----------|----------|------|
| `<str:>` | str | no `/` |
| `<int:>` | int | ≥ 0 only |
| `<slug:>` | str | a-zA-Z0-9-_ |
| `<uuid:>` | uuid.UUID | |
| `<path:>` | str | includes `/` |

```python
# --- reversing ---
reverse("blog:detail", args=[42])              # "/blog/42/"
reverse("blog:detail", kwargs={"id": 42})      # "/blog/42/"
reverse_lazy("blog:list")                       # for class attrs / settings
redirect("blog:detail", id=42)                  # 302 redirect
# template:  {% url 'blog:detail' id=42 %}
```

```python
# --- FBV ---
def view(request, id):
    request.method                 # "GET" / "POST" / ...
    request.GET.get("q", "")       # query param (+ default)
    request.GET.getlist("tag")     # repeated params -> list
    request.POST.get("name")       # form-encoded body
    json.loads(request.body)       # JSON body (bytes)
    request.headers["User-Agent"]  # headers (case-insensitive)
    return JsonResponse({"id": id})            # dict
    return JsonResponse([1, 2], safe=False)    # list needs safe=False

# --- decorators ---
@require_http_methods(["GET", "POST"])   # else 405
@require_POST                            # POST only
@login_required                          # redirect anon
```

```python
# --- CBV ---
from django.views import View
class ArticleView(View):
    def get(self, request, id): ...
    def post(self, request, id): ...      # undefined verbs -> 405 via dispatch()
# urls.py:  path("a/<int:id>/", ArticleView.as_view())

# protect all verbs:
@method_decorator(login_required, name="dispatch")
class SecretView(View): ...

# --- generic CBVs ---
ListView    # model, paginate_by, context_object_name
DetailView  # model (uses <int:pk> / <slug:slug>)
CreateView  # model, fields, success_url (use reverse_lazy)
UpdateView  # model, fields, success_url
DeleteView  # model, success_url
```
