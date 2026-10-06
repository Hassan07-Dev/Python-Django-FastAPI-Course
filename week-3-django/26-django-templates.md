# 26 — Templates

> Week 3 · Django

## What you will learn

- The **philosophy** behind the Django Template Language (DTL): why it deliberately keeps logic out of templates.
- How to **configure** where Django looks for templates (`DIRS`, `APP_DIRS`).
- The `render()` shortcut and how a **context** dictionary feeds data into a template.
- **Variable lookup** rules — the magic dotted `.` that tries dict keys, attributes, methods, and list indices in order.
- The essential **built-in tags**: `{% if %}`, `{% for %}` (and the `forloop` object), `{% url %}`, `{% block %}`/`{% extends %}`, `{% include %}`, `{% with %}`, `{% csrf_token %}`.
- Built-in **filters** (`date`, `default`, `length`, `truncatewords`, `safe`) and how **autoescaping** protects you from XSS.
- The **template inheritance** pattern: one `base.html` with `{% block %}`s that children fill in.
- Serving **static files** with `{% load static %}`.
- Writing your own **custom tags and filters** (a quick tour).
- A side-by-side mapping to Laravel **Blade** so your existing instincts transfer.

---

## PHP/Laravel bridge

DTL is Django's answer to **Blade**. If you've written `@extends('layouts.app')`, `@section('content')`, `{{ $user->name }}`, and `@foreach` in Laravel, you already understand 80% of DTL — the syntax just shifts. Blade compiles to PHP and lets you drop into raw `<?php ?>` whenever you want; DTL is **intentionally restrictive** and gives you no way to run arbitrary Python in a template. That restriction is a feature: it pushes business logic into views and keeps templates focused on presentation. The variable-printing syntax `{{ }}` is identical, and Blade's `{{ }}` auto-escaping has a direct equivalent in DTL's autoescape — both protect against XSS by default, and both have an "I really mean it, don't escape" escape hatch (`{!! !!}` in Blade, the `|safe` filter in DTL).

---

## Core concept — The DTL philosophy: logic-light by design

Django's template system has a strong opinion: **templates are for presentation, not programming**. You will notice things that feel *missing* compared to Blade or plain PHP:

- You **cannot** call a function with arguments inside a template. `{{ user.get_full_name }}` calls a no-arg method, but `{{ obj.do_thing(42) }}` is a **syntax error** — there's no parentheses-call syntax at all.
- You **cannot** do arbitrary arithmetic. There's no `{{ price * quantity }}`. You compute that in the view, or use the `add` filter for the simple case.
- Assignment is limited — you set names with `{% with %}` or the `as` form of certain tags, not free-form `=`.

The reasoning: when designers and back-end developers share template files, you don't want a stray loop or a fat query buried in the markup. Anything genuinely *logical* belongs in the **view** (a Python function/class), which is fully testable. The template just arranges already-computed data.

> Mental model: the **view** is the chef who cooks the meal; the **template** is the waiter who plates and presents it. The waiter never goes back to re-cook.

DTL is also **pluggable** — Django ships with two engines: its own `DjangoTemplates` and `Jinja2` (which is closer to "real Python in templates"). This lesson covers the default DTL, which is what interviewers mean by "Django templates."

---

## Core concept — Configuring template directories

Templates are configured in `settings.py` under the `TEMPLATES` list. A fresh project looks like this:

```python
# myproject/settings.py
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        # Project-wide template folders, searched first.
        "DIRS": [BASE_DIR / "templates"],
        # If True, also search each installed app's "templates/" folder.
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

Two ways Django finds a template named `"blog/post.html"`:

1. **`DIRS`** — explicit project-level folders. With the config above, Django checks `BASE_DIR/templates/blog/post.html`.
2. **`APP_DIRS: True`** — Django also looks inside every app: `blog/templates/blog/post.html`, `shop/templates/shop/post.html`, etc.

```text
myproject/
├── templates/                 # project-wide (DIRS)
│   └── base.html
└── blog/
    └── templates/
        └── blog/              # <-- note the repeated "blog/" namespace
            ├── post_list.html
            └── post_detail.html
```

**Why the repeated `blog/` folder inside `templates/`?** App template dirs are all merged into one flat search path. If two apps both had `templates/list.html`, Django would load whichever it found *first* — a silent collision. Namespacing under `templates/<appname>/` makes `blog/post_list.html` unambiguous. This is the single most common beginner trip-up: people put the file at `blog/templates/post_list.html` and then can't figure out why `render(request, "blog/post_list.html")` raises `TemplateDoesNotExist`.

`context_processors` are functions that inject variables into **every** template's context automatically — that's how `{{ request }}`, `{{ user }}`, and `{{ messages }}` are available everywhere without you passing them.

---

## Core concept — `render()` and the context

A **view** turns a request into a response. The `render()` shortcut loads a template, fills it with a **context** dictionary, and returns an `HttpResponse`.

```python
# blog/views.py
from django.shortcuts import render, get_object_or_404
from .models import Post

def post_list(request):
    posts = Post.objects.filter(published=True).order_by("-created_at")
    context = {
        "posts": posts,
        "page_title": "Latest posts",
    }
    return render(request, "blog/post_list.html", context)

def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    return render(request, "blog/post_detail.html", {"post": post})
```

`render(request, template_name, context=None, content_type=None, status=None, using=None)`:

- `request` — needed so context processors (and `{% csrf_token %}`, `{{ user }}`) work.
- `template_name` — the dotted path relative to a search root, e.g. `"blog/post_list.html"`.
- `context` — a plain `dict`. Keys become template variable names.

The corresponding template:

```django
{# blog/post_list.html #}
<h1>{{ page_title }}</h1>

<ul>
{% for post in posts %}
  <li>{{ post.title }} — {{ post.created_at|date:"M d, Y" }}</li>
{% endfor %}
</ul>
```

Two delimiters to keep straight:

| Syntax | Purpose | Blade analog |
|---|---|---|
| `{{ ... }}` | **Output** a variable (auto-escaped) | `{{ $var }}` |
| `{% ... %}` | **Tag** — logic/control flow, no output by default | `@if`, `@foreach` |
| `{# ... #}` | **Comment** (not rendered) | `{{-- --}}` |

---

## Core concept — Variable lookup: the dotted path

`{{ post.author.name }}` looks innocent, but Django tries **four** kinds of lookup for each `.` segment, in this exact order, stopping at the first that works:

1. **Dictionary** lookup — `post["author"]`
2. **Attribute** lookup — `post.author`
3. **Method** call (no arguments) — `post.author()`
4. **Numeric index** lookup — `post.author[0]`

This is why the *same* dotted syntax works for a dict, an object, a method, and a list:

```python
# in the view
context = {
    "data": {"key": "from a dict"},          # dict
    "user": SomeUser(name="Ada"),            # object attribute
    "items": ["zero", "one", "two"],         # list index
}
```

```django
{{ data.key }}        {# => from a dict      (dict lookup)        #}
{{ user.name }}       {# => Ada              (attribute lookup)   #}
{{ items.1 }}         {# => one              (index — note: dot, not [1]) #}
{{ user.is_active }}  {# calls user.is_active() if it's a method  #}
```

**Gotcha — keys with special characters.** Because the parser uses `.`, you can't access `data["key with spaces"]` or `data["key.with.dots"]` from a template. If your dict has such keys, rename them in the view or use a custom filter.

**Gotcha — silent failure.** If *no* lookup succeeds, Django doesn't raise — it renders the `string_if_invalid` setting, which defaults to the **empty string**. A typo like `{{ post.titel }}` quietly produces nothing rather than an error. This trips up everyone coming from a stricter system.

```python
# Methods are called automatically — but only if they take no required args
class Post(models.Model):
    title = models.CharField(max_length=200)

    def word_count(self):           # callable with no args
        return len(self.title.split())
```

```django
{{ post.word_count }}   {# => 3   — Django calls the method for you, no () #}
```

If a method needs arguments, the template can't call it — move that work to the view or write a custom tag.

---

## Core concept — Built-in tags

### `{% if %}` / `{% elif %}` / `{% else %}`

```django
{% if post.published and post.author %}
  <span class="badge">Live</span>
{% elif post.draft %}
  <span class="badge">Draft</span>
{% else %}
  <span class="badge">Unknown</span>
{% endif %}
```

Supported operators: `and`, `or`, `not`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `in`, `not in`, `is`, `is not`. No arithmetic, no function calls.

```django
{% if posts|length > 5 %}<p>Lots of posts!</p>{% endif %}
{% if "django" in post.tags %}<p>Tagged Django</p>{% endif %}
```

### `{% for %}` and the `forloop` object

```django
<ol>
{% for post in posts %}
  <li>{{ forloop.counter }}. {{ post.title }}</li>
{% empty %}
  <li>No posts yet.</li>
{% endfor %}
</ol>
```

`{% empty %}` runs when the iterable is empty — no separate `{% if %}` needed. Inside the loop, the `forloop` object exposes handy state:

| Variable | Meaning |
|---|---|
| `forloop.counter` | 1-based index (1, 2, 3, …) |
| `forloop.counter0` | 0-based index (0, 1, 2, …) |
| `forloop.revcounter` | Count down to 1 |
| `forloop.revcounter0` | Count down to 0 |
| `forloop.first` | `True` on the first iteration |
| `forloop.last` | `True` on the last iteration |
| `forloop.parentloop` | The `forloop` of the enclosing loop (when nested) |

```django
{% for row in matrix %}
  {% for cell in row %}
    {# outer index via parentloop #}
    R{{ forloop.parentloop.counter }}C{{ forloop.counter }}={{ cell }}
  {% endfor %}
{% endfor %}
```

### `{% url %}` — reverse URL resolution

Never hardcode paths. Reference the URL by its **name** so changing routes doesn't break templates.

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = "blog"   # enables the "blog:" namespace
urlpatterns = [
    path("", views.post_list, name="post_list"),
    path("<int:pk>/", views.post_detail, name="post_detail"),
]
```

```django
<a href="{% url 'blog:post_list' %}">All posts</a>
<a href="{% url 'blog:post_detail' pk=post.pk %}">Read {{ post.title }}</a>

{# capture into a variable with "as" #}
{% url 'blog:post_detail' pk=post.pk as detail_url %}
<a href="{{ detail_url }}" class="card">…</a>
```

This is the equivalent of Laravel's `route('posts.show', $post)`.

### `{% with %}` — name an expensive expression once

```django
{% with total=order.items.count %}
  <p>You have {{ total }} item{{ total|pluralize }}.</p>
  {% if total > 10 %}<p>Free shipping!</p>{% endif %}
{% endwith %}
```

Without `{% with %}`, `order.items.count` would hit the database **twice**. `{% with %}` caches it for the block's lifetime — a real performance lever, not just sugar.

### `{% include %}` — embed a sub-template

```django
{# reuse a card partial #}
{% for post in posts %}
  {% include "blog/_post_card.html" with post=post only %}
{% endfor %}
```

- `with post=post` passes specific variables.
- `only` restricts the included template to **just** those variables (don't leak the whole parent context). Recommended for predictable partials. This mirrors Blade's `@include('partials.card', ['post' => $post])`.

### `{% csrf_token %}` — required on every POST form

```django
<form method="post" action="{% url 'blog:post_create' %}">
  {% csrf_token %}
  <input type="text" name="title">
  <button type="submit">Save</button>
</form>
```

This renders a hidden `<input>` with a per-session token. Omit it and Django returns **403 Forbidden** on submit. It's the direct equivalent of Blade's `@csrf`.

---

## Core concept — Filters

Filters transform a value with the pipe `|`, optionally taking one argument after `:`. They chain left-to-right.

```django
{{ post.title|upper }}                       {# HELLO WORLD #}
{{ post.body|truncatewords:25 }}             {# first 25 words + "…" #}
{{ post.created_at|date:"D, d M Y" }}        {# Mon, 19 Jun 2026 #}
{{ comment.author|default:"Anonymous" }}     {# fallback if empty/None #}
{{ posts|length }}                           {# 12 #}
{{ price|floatformat:2 }}                     {# 19.50 #}
{{ name|truncatewords:3|upper }}             {# chain: truncate THEN upper #}
```

Frequently used built-ins:

| Filter | Example | Result |
|---|---|---|
| `date` | `value\|date:"Y-m-d"` | `2026-06-19` |
| `time` | `value\|time:"H:i"` | `14:30` |
| `default` | `x\|default:"—"` | `x`, or `—` if falsy |
| `default_if_none` | `x\|default_if_none:"—"` | only swaps on `None` (keeps `""`, `0`) |
| `length` | `items\|length` | item count |
| `truncatewords` | `text\|truncatewords:10` | first 10 words |
| `truncatechars` | `text\|truncatechars:50` | first 50 chars |
| `pluralize` | `n items{{ n\|pluralize }}` | adds `s` when `n != 1` |
| `lower` / `upper` / `title` | `s\|title` | case transforms |
| `join` | `tags\|join:", "` | `python, django` |
| `linebreaks` | `text\|linebreaks` | `\n` → `<p>`/`<br>` |
| `yesno` | `flag\|yesno:"on,off"` | maps booleans to words |
| `safe` | `html\|safe` | mark string as safe HTML (see below) |

### Autoescaping and `|safe` — the XSS story

By default, DTL **HTML-escapes every variable**. This is your front-line defense against cross-site scripting (XSS).

```python
context = {"comment": "<script>alert('xss')</script>"}
```

```django
<p>{{ comment }}</p>
{# Rendered as: <p>&lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;</p> #}
{# The browser shows literal text — the script never runs. Safe. #}
```

To render **trusted** HTML verbatim, use `|safe` — but only on content you control:

```django
<div>{{ article.rendered_html|safe }}</div>   {# renders real tags #}
```

You can also toggle escaping for a block:

```django
{% autoescape off %}
  {{ trusted_markup }}   {# not escaped inside this block #}
{% endautoescape %}

{% autoescape on %}
  {{ user_input }}       {# escaped (the default) #}
{% endautoescape %}
```

> **Security rule:** never apply `|safe` (or `autoescape off`) to data that originated from a user. That's exactly the XSS hole escaping exists to close. Blade's `{!! !!}` carries the identical warning.

---

## Core concept — Template inheritance with `{% extends %}` and `{% block %}`

This is the heart of maintainable templates and the part interviewers love. You define **one** skeleton (`base.html`) with named **blocks**; child templates `{% extends %}` it and override only the blocks they care about.

```django
{# templates/base.html — the project skeleton #}
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{% block title %}My Site{% endblock %}</title>
  <link rel="stylesheet" href="{% static 'css/site.css' %}">
  {% block extra_head %}{% endblock %}
</head>
<body>
  <header>{% include "partials/_nav.html" %}</header>

  <main>
    {% block content %}{% endblock %}
  </main>

  <footer>{% block footer %}&copy; 2026 My Site{% endblock %}</footer>
  {% block scripts %}{% endblock %}
</body>
</html>
```

```django
{# blog/post_list.html — a child that fills in blocks #}
{% extends "base.html" %}

{% block title %}Posts · {{ block.super }}{% endblock %}

{% block content %}
  <h1>All posts</h1>
  <ul>
  {% for post in posts %}
    <li><a href="{% url 'blog:post_detail' pk=post.pk %}">{{ post.title }}</a></li>
  {% empty %}
    <li>Nothing here yet.</li>
  {% endfor %}
  </ul>
{% endblock %}
```

Key rules and idioms:

- **`{% extends %}` must be the first tag** in the child template — nothing (not even whitespace text) before it.
- A child only needs to override the blocks it wants; un-overridden blocks fall back to the parent's default content (note `footer` above keeps the base footer).
- **`{{ block.super }}`** renders the parent block's content *inside* the override — perfect for "append to" rather than "replace." Above, the title becomes `Posts · My Site`.
- Inheritance can be **multi-level**: `base.html` → `blog/base_blog.html` → `blog/post_list.html`.

Blade mapping is nearly one-to-one:

| Blade | DTL |
|---|---|
| `@extends('base')` | `{% extends "base.html" %}` |
| `@section('content') … @endsection` | `{% block content %} … {% endblock %}` |
| `@yield('content')` | `{% block content %}{% endblock %}` (in the parent) |
| `@parent` | `{{ block.super }}` |
| `@include('partials.nav')` | `{% include "partials/_nav.html" %}` |

A subtle but important difference: in Blade the *parent* uses `@yield` and the *child* uses `@section`. In DTL the **same** `{% block %}` tag is used in both parent and child — the parent's block holds the default, the child's block overrides it.

---

## Core concept — Static files in templates

CSS, JS, images, and fonts are **static files**. Don't hardcode `/static/...` URLs; use the `static` tag so the path stays correct across dev and production (where files may live on a CDN with hashed names).

```python
# settings.py
STATIC_URL = "static/"
STATICFILES_DIRS = [BASE_DIR / "static"]   # project-level static during dev
STATIC_ROOT = BASE_DIR / "staticfiles"     # where collectstatic gathers files for prod
```

```django
{% load static %}   {# must load the tag library before using {% static %} #}

<link rel="stylesheet" href="{% static 'css/site.css' %}">
<img src="{% static 'img/logo.png' %}" alt="Logo">
<script src="{% static 'js/app.js' %}"></script>

{# capture the URL into a variable if you need it more than once #}
{% static 'img/logo.png' as logo_url %}
<img src="{{ logo_url }}" srcset="{{ logo_url }} 2x">
```

In production you run `python manage.py collectstatic`, which copies every app's static files into `STATIC_ROOT` to be served by your web server or CDN. The `{% static %}` tag, paired with `ManifestStaticFilesStorage`, can emit cache-busting hashed filenames (`site.4f3a2b.css`) automatically — which is why hardcoding paths is a bad idea.

---

## Core concept — Custom tags and filters (a quick tour)

When a built-in doesn't exist, write your own. They live in a `templatetags/` package inside an app.

```text
blog/
├── __init__.py
├── templatetags/
│   ├── __init__.py        # required — makes it a package
│   └── blog_extras.py
└── views.py
```

```python
# blog/templatetags/blog_extras.py
from django import template

register = template.Library()

# --- A custom FILTER: {{ price|currency:"$" }} ---
@register.filter
def currency(value, symbol="$"):
    return f"{symbol}{value:,.2f}"   # 1234.5 -> "$1,234.50"

# --- A simple TAG returning a value: {% multiply 6 7 %} ---
@register.simple_tag
def multiply(a, b):
    return a * b

# --- An INCLUSION tag: renders a template fragment with computed context ---
@register.inclusion_tag("blog/_latest_posts.html")
def show_latest(count=5):
    from .models import Post   # local import avoids circular imports
    return {"posts": Post.objects.order_by("-created_at")[:count]}
```

Use them after loading the library:

```django
{% load blog_extras %}

{{ product.price|currency:"€" }}        {# €1,234.50 #}
{% multiply 6 7 as answer %}            {# store result #}
<p>6 × 7 = {{ answer }}</p>
{% show_latest 3 %}                     {# renders blog/_latest_posts.html #}
```

- **`@register.filter`** — transforms a single value; takes the value plus at most one extra argument.
- **`@register.simple_tag`** — runs Python and returns a value (use `as name` to capture it). This is the escape hatch for "I need to call a function with arguments," which the template language otherwise forbids.
- **`@register.inclusion_tag`** — builds a context dict and renders a small template, great for reusable widgets (think sidebars, nav menus).

Restart the dev server after adding a new `templatetags` module — Django discovers them at startup.

---

## Common mistakes & gotchas

- **`TemplateDoesNotExist`** — almost always the missing app namespace folder. The file must be at `blog/templates/blog/post_list.html`, not `blog/templates/post_list.html`, when you call `render(request, "blog/post_list.html")`.
- **Calling methods with arguments** — `{{ obj.method(1) }}` is invalid. There is no call syntax. Compute in the view, use `with`, or a `simple_tag`.
- **Silent typos** — `{{ post.titel }}` renders empty, not an error. Double-check variable names; consider setting `string_if_invalid` during development to catch them.
- **Forgetting `{% load static %}`** — using `{% static %}` without loading the library raises `Invalid block tag: 'static'`. Same for any custom tag library.
- **`{% extends %}` not first** — any text or tag before `{% extends %}` (even a blank line of HTML) raises an error.
- **Overusing `|safe`** — applying it to user-generated content reopens XSS. Only mark HTML you fully control.
- **N+1 in templates** — `{% for post in posts %}{{ post.author.name }}{% endfor %}` fires a query per post. Fix it in the view with `select_related("author")`, not in the template.
- **Doing real logic in templates** — if you find yourself nesting `{% if %}`/`{% with %}` deeply, the logic belongs in the view or a tag.
- **Index access** — it's `{{ items.0 }}` with a dot, not `{{ items[0] }}`. Bracket syntax doesn't exist in DTL.
- **`default` vs `default_if_none`** — `default` swaps on *any* falsy value (`""`, `0`, `False`, `None`); `default_if_none` only swaps on `None`. Using `default` on a numeric `0` can surprise you.

---

## Best practices

- Keep templates **logic-light**. If it's a computation, query shaping, or a decision with real branches, do it in the view.
- Always use **`{% url %}`** and **`{% static %}`** — never hardcode paths.
- Build a single **`base.html`** and have everything extend it; use `{{ block.super }}` to extend rather than duplicate.
- Name shared partials with a leading underscore (`_post_card.html`, `_nav.html`) so it's obvious they're includes.
- Pass **`only`** to `{% include %}` so partials don't accidentally depend on the parent's context.
- Reach for **`{% with %}`** to cache repeated `.count`/`.all` lookups and avoid duplicate queries.
- Rely on **autoescaping**; reserve `|safe` for content you sanitized/control, and document why.
- Put reusable widgets behind **`inclusion_tag`s** and reusable transforms behind **filters** instead of copy-pasting markup.
- Prefer **`get_object_or_404`** in views so detail templates never render with `None`.

---

## Interview Q&A

**Q: Why does the Django template language deliberately limit what you can do (no function calls with args, no arithmetic)?**
**A:** To enforce separation of concerns. Templates are presentation; business logic lives in views, which are plain Python and fully testable. Restricting the template language prevents heavy logic and database queries from creeping into markup, keeps templates readable for designers, and reduces the chance of expensive operations hiding in the view layer.

**Q: A teammate put their template at `app/templates/home.html` and `render(request, "app/home.html")` raises `TemplateDoesNotExist`. What's wrong?**
**A:** With `APP_DIRS=True`, every app's `templates/` folder is merged into one flat search path, so you namespace by app: the file must be `app/templates/app/home.html`. The path you pass to `render` (`"app/home.html"`) is relative to that merged root, so the extra `app/` folder is required to disambiguate from other apps.

**Q: Explain Django's variable lookup order for `{{ a.b }}`.**
**A:** Django tries, in order: dictionary lookup `a["b"]`, attribute lookup `a.b`, no-argument method call `a.b()`, then numeric index `a[b]`. The first that succeeds wins. If none do, it renders `string_if_invalid` (empty string by default) rather than raising — so typos fail silently.

**Q: What is autoescaping and how does it relate to `|safe`?**
**A:** DTL HTML-escapes every variable output by default (`<` becomes `&lt;`, etc.), which prevents XSS from user-supplied data. `|safe` marks a string as already-safe so it's rendered verbatim. You should only use `|safe` (or `{% autoescape off %}`) on HTML you fully trust — never on user input, or you reintroduce the XSS hole.

**Q: How does template inheritance work, and what does `{{ block.super }}` do?**
**A:** A parent template (e.g. `base.html`) defines named `{% block %}`s. A child uses `{% extends "base.html" %}` (must be the first tag) and redefines blocks to override them; un-overridden blocks keep the parent's default. `{{ block.super }}` renders the parent block's content inside the child's override, so you can append to the parent rather than fully replace it.

**Q: How is DTL inheritance different from Blade's?**
**A:** Conceptually identical, but Blade uses `@yield('x')` in the parent and `@section('x')` in the child — two different directives. DTL uses the same `{% block x %}` tag in both: the parent's holds the default, the child's overrides. `@parent` maps to `{{ block.super }}`, `@extends` to `{% extends %}`, and `@include` to `{% include %}`.

**Q: Why use `{% url %}` instead of hardcoding `/blog/5/`?**
**A:** `{% url 'blog:post_detail' pk=5 %}` reverses the route by name, so when you change the URL pattern the template stays correct — there's a single source of truth in `urls.py`. Hardcoded paths break silently when routes change and don't respect URL namespaces or prefixes.

**Q: When and why would you use `{% with %}`?**
**A:** To assign an expression to a name for a block, primarily to avoid recomputing/re-querying. `{% with total=order.items.count %}` evaluates the count once; without it, referencing `order.items.count` twice would hit the database twice. It also improves readability for long dotted paths.

**Q: How do you serve static files in a template, and what does `collectstatic` do?**
**A:** Load the library with `{% load static %}` and reference files via `{% static 'css/site.css' %}`, which builds the URL from `STATIC_URL`. In production, `python manage.py collectstatic` copies all apps' static files into `STATIC_ROOT` for the web server/CDN to serve; combined with `ManifestStaticFilesStorage` it produces hashed, cache-busted filenames — which is exactly why you never hardcode static paths.

**Q: The template can't call your model method `discounted_price(rate)` because it takes an argument. What are your options?**
**A:** Compute the value in the view and pass it in the context; precompute it as a no-arg property/method on the model; or write a custom `@register.simple_tag` (or filter) that accepts the argument and call it as `{% discounted_price product 0.1 %}`. Custom tags are the sanctioned way to do parameterized logic in templates.

**Q: What's the difference between `default` and `default_if_none`?**
**A:** `|default:"x"` substitutes when the value is *any* falsy value — `None`, `""`, `0`, `False`, `[]`. `|default_if_none:"x"` substitutes *only* when the value is `None`, preserving falsy-but-meaningful values like `0` or `""`. Use `default_if_none` for numbers/strings where `0`/`""` are valid.

**Q: How do you prevent N+1 queries that show up while iterating in a template?**
**A:** The template is the wrong place to fix it — it's a view/queryset concern. Use `select_related` (for FK/one-to-one) and `prefetch_related` (for many-to-many/reverse FK) in the view so the related data is fetched up front. The template then iterates over already-loaded objects with no extra queries.

---

## Practice exercises

Try these before peeking at the solutions.

1. **Base + child.** Create `base.html` with `title`, `content`, and `footer` blocks (footer has default text). Write `about.html` that extends it, sets the title to `About · <site name>` using `{{ block.super }}`, fills `content`, and leaves the footer untouched.

2. **List with loop state.** Given a context `{"books": [...]}`, render an ordered list showing `forloop.counter`, the book title, and a `"(newest)"` badge on `forloop.first`. Show `"No books."` when the list is empty.

3. **Safe filtering chain.** Given `{"bio": "  django dev  ", "html_snippet": "<b>hi</b>"}`, render the bio uppercased and truncated to 5 words with a `"N/A"` fallback if empty, and render the snippet as real HTML.

4. **URL + CSRF form.** Write a POST form that submits to a URL named `contact:submit`, includes the CSRF token, and a cancel link to `contact:home`.

5. **Custom filter.** Write a `currency` filter so `{{ 1999.5|currency:"£" }}` renders `£1,999.50`. Show the `templatetags` file and the template usage.

6. **Stretch — inclusion tag.** Write an `@register.inclusion_tag` named `recent_comments` that takes a `count` and renders a `_recent_comments.html` partial with the latest comments.

<details><summary>Solutions</summary>

**1. Base + child**

```django
{# templates/base.html #}
<!DOCTYPE html>
<html lang="en">
<head><title>{% block title %}My Site{% endblock %}</title></head>
<body>
  <main>{% block content %}{% endblock %}</main>
  <footer>{% block footer %}&copy; 2026 My Site{% endblock %}</footer>
</body>
</html>
```

```django
{# templates/about.html #}
{% extends "base.html" %}
{% block title %}About · {{ block.super }}{% endblock %}
{% block content %}
  <h1>About us</h1>
  <p>We build things.</p>
{% endblock %}
{# footer block omitted -> base default is used #}
```

**2. List with loop state**

```django
<ol>
{% for book in books %}
  <li>
    {{ forloop.counter }}. {{ book.title }}
    {% if forloop.first %}<span class="badge">(newest)</span>{% endif %}
  </li>
{% empty %}
  <li>No books.</li>
{% endfor %}
</ol>
```

**3. Safe filtering chain**

```django
<p>{{ bio|default:"N/A"|upper|truncatewords:5 }}</p>
<div>{{ html_snippet|safe }}</div>   {# renders <b>hi</b> as bold #}
```

**4. URL + CSRF form**

```django
<form method="post" action="{% url 'contact:submit' %}">
  {% csrf_token %}
  <input type="text" name="message" required>
  <button type="submit">Send</button>
  <a href="{% url 'contact:home' %}">Cancel</a>
</form>
```

**5. Custom filter**

```python
# yourapp/templatetags/money.py
from django import template

register = template.Library()

@register.filter
def currency(value, symbol="$"):
    return f"{symbol}{value:,.2f}"
```

```django
{% load money %}
{{ 1999.5|currency:"£" }}   {# => £1,999.50 #}
```

**6. Inclusion tag**

```python
# yourapp/templatetags/comment_extras.py
from django import template
from yourapp.models import Comment

register = template.Library()

@register.inclusion_tag("yourapp/_recent_comments.html")
def recent_comments(count=5):
    return {"comments": Comment.objects.order_by("-created_at")[:count]}
```

```django
{# yourapp/_recent_comments.html #}
<ul>
{% for c in comments %}
  <li>{{ c.author }}: {{ c.body|truncatewords:10 }}</li>
{% empty %}
  <li>No comments yet.</li>
{% endfor %}
</ul>
```

```django
{% load comment_extras %}
{% recent_comments 3 %}
```

</details>

---

## Cheat sheet

```django
{{ var }}                    output (auto-escaped)
{% tag %}                    logic, no output
{# comment #}                not rendered

{{ a.b.c }}                  lookup: dict -> attr -> method() -> index
{{ items.0 }}               list index uses a DOT, not [0]

{# control flow #}
{% if x and y %}…{% elif z %}…{% else %}…{% endif %}
{% for o in qs %}{{ forloop.counter }}{% empty %}none{% endfor %}
{% with t=order.items.count %}…{% endwith %}

{# routing & assets #}
{% url 'app:name' pk=obj.pk %}
{% url 'app:name' as u %}    capture into var u
{% load static %}{% static 'css/site.css' %}
{% csrf_token %}            required on POST forms

{# inheritance #}
{% extends "base.html" %}    MUST be the first tag
{% block content %}…{% endblock %}
{{ block.super }}            parent block's content
{% include "p/_x.html" with v=v only %}

{# filters (chain with |, arg with :) #}
{{ d|date:"Y-m-d" }}  {{ x|default:"—" }}  {{ qs|length }}
{{ t|truncatewords:25 }}  {{ html|safe }}  {{ n|pluralize }}
{{ s|upper|truncatewords:3 }}            chained
```

| forloop var | meaning |
|---|---|
| `counter` / `counter0` | 1-based / 0-based index |
| `first` / `last` | boolean edges |
| `revcounter` | countdown to 1 |
| `parentloop` | enclosing loop (nested) |

| Blade | DTL |
|---|---|
| `@extends` / `@section` / `@yield` / `@parent` | `{% extends %}` / `{% block %}` / `{% block %}` / `{{ block.super }}` |
| `@include` / `@csrf` / `{{ }}` / `{!! !!}` | `{% include %}` / `{% csrf_token %}` / `{{ }}` / `\|safe` |
| `route()` / `asset()` | `{% url %}` / `{% static %}` |

**Custom code:** `@register.filter` (transform a value) · `@register.simple_tag` (run Python, `as name`) · `@register.inclusion_tag("t.html")` (render a fragment) — all in `app/templatetags/<name>.py` with `register = template.Library()`.
