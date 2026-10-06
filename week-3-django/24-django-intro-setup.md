# 24 — Django Intro, Setup & Project Structure

> Week 3 · Django Fundamentals

## What you will learn

- What Django is, who uses it, and what "batteries included" actually means.
- The **MVT** (Model–View–Template) pattern and how it maps to classic **MVC** and to Laravel.
- How to install Django the right way inside a **virtual environment (venv)**.
- `django-admin startproject` and every file it generates (`manage.py`, `settings.py`, `urls.py`, `wsgi.py`, `asgi.py`).
- The crucial **project vs app** distinction, and `startapp`.
- Registering an app in `INSTALLED_APPS`.
- Running the dev server with `manage.py runserver`.
- A guided tour of `settings.py` — the knobs you will actually touch.
- The full **request → response flow** through Django.
- A side-by-side **Django ↔ Laravel** translation table so your existing instincts transfer.

## PHP/Laravel bridge

If you know Laravel, you already know 80% of Django's *ideas* — only the spelling changes. Django is to Python what Laravel is to PHP: a full-stack, opinionated, "everything included" web framework. Laravel's `artisan` console is Django's `manage.py`; Laravel's `config/` directory collapses into one `settings.py`; Laravel's `routes/web.php` becomes `urls.py`; and Laravel **Controllers** become Django **views**. The biggest mental adjustment is naming: what Laravel (and the rest of the world) calls a "controller", Django calls a **view**, and what you'd call a "view" (the HTML template) Django calls a **template** — hence **MVT** instead of **MVC**. Also, where a Laravel project is usually *one* app, a Django "project" is a container that holds *multiple reusable* "apps".

---

## Core concept 1 — What is Django?

Django is a high-level Python web framework first released in 2005, maintained by the Django Software Foundation. It powers Instagram, Pinterest, Mozilla, NASA, and Disney, among others. Its tagline is *"The web framework for perfectionists with deadlines."*

### "Batteries included"

This is the phrase you will hear constantly. It means Django ships with the common pieces of a web app **already built in**, so you don't assemble a framework from a dozen libraries. Out of the box you get:

| Built-in feature | What it does | Laravel equivalent |
|---|---|---|
| **ORM** | Talk to the DB with Python objects, no SQL needed | Eloquent |
| **Migrations** | Version-control your schema | Laravel migrations |
| **Admin site** | Auto-generated CRUD UI for your models | Nova / Filament (but free & automatic) |
| **Auth system** | Users, passwords, permissions, sessions | Laravel auth / Fortify |
| **Forms & validation** | Render + validate input | Form Requests |
| **Template engine** | Render HTML | Blade |
| **Routing** | Map URLs to code | Routes |
| **Security** | CSRF, XSS, SQL-injection, clickjacking protection | Built-in middleware |
| **i18n / l10n** | Translations, timezones | Localization |

The trade-off: Django is **opinionated**. It has a "Django way" of doing things. That's a feature — you spend less time bikeshedding and more time shipping. (FastAPI, which you'll meet in Week 5, sits at the opposite end: minimal core, you bolt on what you need. Knowing both ends of the spectrum is exactly what interviewers want.)

> For pure **REST APIs**, plain Django is rarely used alone — you add **Django REST Framework (DRF)**, covered later this course. But everything in this lesson (projects, apps, settings, the request flow) is the foundation DRF builds on.

---

## Core concept 2 — MVT vs MVC

Django describes itself as following the **MVT** pattern: **Model – View – Template**. People coming from MVC frameworks get confused because the words don't line up. Here's the truth: MVT *is* MVC, just with different labels and with the "controller" job split between the framework and your view function.

```text
Classic MVC            Django MVT             Laravel (MVC)
-----------            ----------             -------------
Model       ───────►   Model                  Model (Eloquent)
View (UI)   ───────►   Template               View (Blade)
Controller  ───────►   View  (your code)      Controller
   +                      +
(the wiring)           URLconf / framework    Router
```

- **Model** — your data layer (a Python class mapped to a DB table). Same concept everywhere.
- **View** — in Django this is a **function or class that takes a request and returns a response**. This is what other frameworks call the *controller*. This is the #1 naming gotcha.
- **Template** — the HTML file with placeholders. This is what MVC calls the *view*.

The "C" you're missing is essentially the **URL dispatcher** (`urls.py`) plus the framework itself, which routes the incoming request to the right view. Django folks sometimes joke that it's "MTV" and that the framework *is* the controller.

```python
# A minimal Django view (this is your "controller").
from django.http import HttpResponse

def hello(request):                      # receives the request
    return HttpResponse("Hello, world!")  # returns a response
# => Browser shows: Hello, world!
```

That signature — `request in, response out` — is the heart of Django. Everything else is helpers around it.

---

## Core concept 3 — Install Django in a virtual environment

**Never** install Django globally. Each project gets its own isolated environment so dependencies don't collide — the Python equivalent of giving every project its own `composer.json`/`vendor/` rather than installing PHP packages system-wide. The tool is `venv`, which ships with Python.

```bash
# 1. Confirm you have Python 3.12+
python3 --version
# => Python 3.12.4

# 2. Make a folder for the project and enter it
mkdir blogsite && cd blogsite

# 3. Create a virtual environment named ".venv"
python3 -m venv .venv

# 4. Activate it
#    macOS / Linux:
source .venv/bin/activate
#    Windows (PowerShell):
#    .venv\Scripts\Activate.ps1

# Your prompt now shows the env name:
# (.venv) you@machine blogsite %

# 5. Install Django (latest 5.x) into THIS env only
python -m pip install --upgrade pip
python -m pip install "Django>=5.0,<6.0"

# 6. Verify
python -m django --version
# => 5.1.4
```

Once activated, `python` and `pip` point inside `.venv`. To leave the environment later, run `deactivate`.

### Pin your dependencies

Capture exact versions so a teammate (or a server) can reproduce your setup. This is the analogue of Laravel's `composer.lock`.

```bash
python -m pip freeze > requirements.txt
# Later, on another machine:
# python -m pip install -r requirements.txt
```

```text
# requirements.txt  (example contents)
asgiref==3.8.1
Django==5.1.4
sqlparse==0.5.1
```

> Tip: add `.venv/` to your `.gitignore`. You commit `requirements.txt`, not the installed packages — just like you commit `composer.json`/`composer.lock`, not `vendor/`.

---

## Core concept 4 — Create a project: `startproject`

Django gives you a CLI tool, `django-admin`, to scaffold a new project.

```bash
# The trailing "." means "create files in the CURRENT directory"
# instead of nesting another folder. Highly recommended.
django-admin startproject config .
```

Why name it `config`? `startproject myproject` would create a folder named `myproject` *and* require `myproject.settings` everywhere. Naming the inner package **`config`** (a popular convention) keeps imports tidy and decouples them from the project's display name. You can name it anything; `config`, `core`, and `<projectname>` are all common.

After running it, your tree looks like this:

```text
blogsite/                  ← outer folder (just a container, name irrelevant)
├── .venv/                 ← the virtual environment
├── manage.py              ← your command-line entry point
├── requirements.txt
└── config/                ← the project PACKAGE (the inner one)
    ├── __init__.py        ← marks "config" as a Python package
    ├── settings.py        ← ALL configuration lives here
    ├── urls.py            ← root URL routing table
    ├── asgi.py            ← entry point for async servers
    └── wsgi.py            ← entry point for sync (traditional) servers
```

### What each generated file does

| File | Purpose | Laravel analogue |
|---|---|---|
| `manage.py` | Thin wrapper around `django-admin` that knows your settings module. You run *everything* through it. | `artisan` |
| `config/__init__.py` | Empty file that tells Python "this directory is a package". | (no analogue — Python-specific) |
| `config/settings.py` | Every setting: database, installed apps, middleware, templates, secret key, debug flag, timezone… | the whole `config/` directory + `.env` |
| `config/urls.py` | The **root** URL dispatcher — maps URL patterns to views. | `routes/web.php` |
| `config/wsgi.py` | **W**eb **S**erver **G**ateway **I**nterface entry point. Production servers like Gunicorn/uWSGI import this to serve your app synchronously. | `public/index.php` (the bootstrap) |
| `config/asgi.py` | **A**synchronous **S**erver **G**ateway **I**nterface entry point. Used by async servers (Uvicorn/Daphne) for WebSockets, async views, etc. | (newer concept) |

You almost never edit `wsgi.py`/`asgi.py` by hand early on — but knowing what they are is a classic interview question (see Q&A).

Here's the generated `manage.py` (you rarely touch it, but understand it):

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys


def main():
    # Tells Django which settings module to load:
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


if __name__ == "__main__":
    main()
```

The key line is `DJANGO_SETTINGS_MODULE = "config.settings"`: that's how Django finds your configuration. (And note the friendly error message — if you forget to activate your venv, this is the message you'll see.)

---

## Core concept 5 — Run the development server

```bash
python manage.py runserver
```

```text
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly
until you apply the migrations for app(s): admin, auth, contenttypes,
sessions.
Run 'python manage.py migrate' to apply them.

Django version 5.1.4, using settings 'config.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

Open `http://127.0.0.1:8000/` and you'll see Django's rocket-ship welcome page. 🎉

- The server **auto-reloads** when you save a `.py` file (like `php artisan serve` with auto-reload, or Vite HMR for the backend).
- The "unapplied migrations" warning is normal on a fresh project — Django's built-in apps (auth, admin, sessions) ship with migrations. Run `python manage.py migrate` once to create those tables. (Migrations get a full lesson later.)
- Change the port or host easily:

```bash
python manage.py runserver 8080            # different port
python manage.py runserver 0.0.0.0:8000    # listen on all interfaces (e.g. for Docker)
```

> ⚠️ `runserver` is for **development only**. It is single-threaded-ish, not hardened, and not meant for production traffic. In production you run Gunicorn/Uvicorn pointing at `wsgi.py`/`asgi.py` behind Nginx. Saying "I'd use runserver in prod" in an interview is a red flag.

---

## Core concept 6 — Project vs App: the most important distinction

This trips up everyone coming from Laravel, where a project is typically one monolithic app.

- A **project** is the whole web application / deployable unit. It holds settings and ties everything together. You have **one** project.
- An **app** is a **self-contained, reusable module** that does one thing — e.g. `blog`, `accounts`, `payments`, `api`. A project contains **many** apps, and a well-written app can be dropped into *another* project.

Think of it as: **project = the restaurant; apps = the kitchen, the bar, the front-of-house** — each a focused unit, assembled into one business.

```bash
# Create an app called "blog"
python manage.py startapp blog
```

This generates:

```text
blog/
├── __init__.py
├── admin.py          ← register models with the admin site
├── apps.py           ← app configuration class
├── migrations/       ← schema-change history for this app
│   └── __init__.py
├── models.py         ← your data models (DB tables)
├── tests.py          ← tests for this app
└── views.py          ← your views (a.k.a. controllers)
```

Notice there's **no `urls.py`** in a new app — you create it yourself when the app needs routes (a near-universal convention):

```python
# blog/urls.py  (you create this file)
from django.urls import path
from . import views

app_name = "blog"   # namespaces the URLs (e.g. "blog:index")

urlpatterns = [
    path("", views.index, name="index"),
]
```

```python
# blog/views.py
from django.http import HttpResponse


def index(request):
    return HttpResponse("Welcome to the blog!")
```

### Why split into apps?

| Benefit | Explanation |
|---|---|
| **Reusability** | A clean `accounts` app can be reused across projects. |
| **Separation of concerns** | Each app owns its models, views, urls, admin, tests. |
| **Team scaling** | Different people own different apps with fewer merge conflicts. |
| **Pluggability** | Third-party packages *are* just apps you add to `INSTALLED_APPS`. |

Rule of thumb: an app should do **one cohesive thing**. If you can't describe it in a short phrase, it's probably two apps.

---

## Core concept 7 — Wire it together: `INSTALLED_APPS` and URL includes

Creating an app with `startapp` does **not** activate it. You must register it in `settings.py`.

```python
# config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    # --- your apps ---
    "blog",                       # short form, OR...
    # "blog.apps.BlogConfig",     # explicit AppConfig (also valid, more precise)
]
```

Registering an app lets Django discover its models (for migrations), templates, static files, and admin registrations. Forgetting this is the most common "why isn't my app working?" bug.

Then connect the app's URLs to the project's root `urls.py` using `include()`:

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),         # the built-in admin UI
    path("blog/", include("blog.urls")),     # delegate /blog/* to the blog app
    path("", include("blog.urls")),          # or mount the app at the site root
]
```

`include()` says "for any URL starting with this prefix, hand off to that app's URL table." This keeps routing modular — each app manages its own routes, the project just stitches them together. (In Laravel terms it's like splitting `routes/web.php` into per-feature route files and `require`-ing them.)

Now `python manage.py runserver` and visit `http://127.0.0.1:8000/blog/` — you'll see *"Welcome to the blog!"*

---

## Core concept 8 — A tour of `settings.py`

`settings.py` is a plain Python module — settings are just module-level variables (UPPER_CASE by convention). Because it's Python, you can compute values, read env vars, and import things. Here are the settings you'll actually touch, with annotations.

```python
# config/settings.py
from pathlib import Path

# BASE_DIR points at the project root (the folder containing manage.py).
# Path(__file__) = this settings.py file; .resolve() makes it absolute;
# .parent.parent walks up two levels (config/ then the root).
BASE_DIR = Path(__file__).resolve().parent.parent

# SECURITY ----------------------------------------------------------------
# Used for cryptographic signing (sessions, CSRF tokens, password resets).
# NEVER commit a real secret. In production, load from an env var.
SECRET_KEY = "django-insecure-CHANGE-ME-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# DEBUG=True shows detailed error pages and serves static files.
# It MUST be False in production (leaks source, settings, stack traces).
DEBUG = True

# Which Host headers are allowed to reach your app. Empty list is fine
# while DEBUG=True; required (non-empty) once DEBUG=False.
ALLOWED_HOSTS = []

# APPLICATION ------------------------------------------------------------
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "blog",
]

# Middleware = a stack of hooks that wrap every request/response.
# ORDER MATTERS — they run top-to-bottom on the way in, bottom-to-top out.
# (This is exactly Laravel's middleware pipeline.)
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

# The root URLconf module Django starts routing from.
ROOT_URLCONF = "config.urls"

# Production entry points (referenced by Gunicorn/Uvicorn).
WSGI_APPLICATION = "config.wsgi.application"
# ASGI_APPLICATION = "config.asgi.application"  # for async deployments

# TEMPLATES --------------------------------------------------------------
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],   # project-wide template dir
        "APP_DIRS": True,                    # also look in each app's templates/
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

# DATABASE ---------------------------------------------------------------
# SQLite by default — zero config, great for learning. Swap ENGINE/NAME
# for PostgreSQL in real projects.
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
# PostgreSQL example:
# DATABASES = {
#     "default": {
#         "ENGINE": "django.db.backends.postgresql",
#         "NAME": "blogdb",
#         "USER": "bloguser",
#         "PASSWORD": "secret",
#         "HOST": "localhost",
#         "PORT": "5432",
#     }
# }

# INTERNATIONALIZATION ---------------------------------------------------
LANGUAGE_CODE = "en-us"
TIME_ZONE = "UTC"        # store everything in UTC; convert on display
USE_I18N = True
USE_TZ = True            # timezone-aware datetimes (keep this True)

# STATIC FILES (CSS, JS, images) -----------------------------------------
STATIC_URL = "static/"

# Default type for auto-created primary keys.
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

### Settings cheat list

| Setting | What it controls | Watch out for |
|---|---|---|
| `SECRET_KEY` | Crypto signing | Keep it secret; rotate if leaked |
| `DEBUG` | Error detail + static serving | **Always `False` in prod** |
| `ALLOWED_HOSTS` | Valid Host headers | Must be set when `DEBUG=False` |
| `INSTALLED_APPS` | Active apps | Add your apps here or they're invisible |
| `MIDDLEWARE` | Request/response pipeline | **Order matters** |
| `DATABASES` | DB connections | Default is SQLite |
| `ROOT_URLCONF` | Where routing starts | Rarely changed |
| `TEMPLATES["DIRS"]` | Template lookup paths | For project-wide templates |
| `TIME_ZONE` / `USE_TZ` | Time handling | Keep `USE_TZ=True`, store UTC |

> **Production tip (preview of a later lesson):** never hard-code secrets. Read them from the environment, e.g. `SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]` and `DEBUG = os.environ.get("DJANGO_DEBUG", "false").lower() == "true"`. This is Django's version of Laravel's `.env` + `env()`. Libraries like `django-environ` or `python-decouple` make this clean.

---

## Core concept 9 — The request → response flow

When a browser hits your Django site, here's the journey of the request. Memorize this — it's a guaranteed interview question.

```text
 1. Browser sends HTTP request  ──►  e.g. GET /blog/
 2. Web server (Gunicorn/Uvicorn, or runserver in dev) receives it
 3. WSGI/ASGI hands it to Django, which builds an HttpRequest object
 4. MIDDLEWARE (request phase) runs top → bottom
       (security, sessions, CSRF check, authentication, …)
 5. URL DISPATCHER (config.urls → include → blog.urls) matches the path
       and selects the VIEW + captures any URL params
 6. VIEW runs your code:
       • may query MODELS (the ORM hits the DATABASE)
       • may render a TEMPLATE with context data
       • returns an HttpResponse
 7. MIDDLEWARE (response phase) runs bottom → top
 8. Django returns the HttpResponse to the server
 9. Server sends the HTTP response back to the browser
```

In one sentence: **request → middleware → URL resolver → view → (model/template) → response → middleware → out**.

Compare to Laravel: HTTP request → `public/index.php` → middleware → router → controller → (Eloquent / Blade) → response. Same shape, different names.

```python
# A view that touches all three letters of MVT:
from django.shortcuts import render
from .models import Post   # M (Model)


def post_list(request):
    posts = Post.objects.all()                 # query the Model (ORM)
    return render(
        request,
        "blog/post_list.html",                 # the Template (T)
        {"posts": posts},                       # context passed to the template
    )
# render() = load template + fill context + return HttpResponse, in one call.
```

---

## Core concept 10 — Django ↔ Laravel translation table

Pin this to your wall for the next five weeks.

| Concept | Laravel | Django |
|---|---|---|
| CLI tool | `php artisan ...` | `python manage.py ...` |
| New project | `laravel new app` | `django-admin startproject config .` |
| New module | (usually one app) | `python manage.py startapp blog` |
| Run dev server | `php artisan serve` | `python manage.py runserver` |
| Routes | `routes/web.php` | `urls.py` |
| Controller | `app/Http/Controllers` | **view** (`views.py`) |
| View / template | Blade (`*.blade.php`) | **template** (`*.html`) |
| Model | Eloquent model | Django model (`models.py`) |
| ORM query | `Post::all()` | `Post.objects.all()` |
| Migrations | `php artisan migrate` | `python manage.py migrate` |
| Make migration | `php artisan make:migration` | `python manage.py makemigrations` |
| Config | `config/*.php` + `.env` | `settings.py` (+ env vars) |
| Dependency mgr | Composer (`composer.json`) | pip (`requirements.txt`) |
| Isolated deps | one global PHP, `vendor/` | `venv` per project |
| Middleware | `app/Http/Middleware` | `MIDDLEWARE` in settings |
| Tinker REPL | `php artisan tinker` | `python manage.py shell` |
| Admin panel | Nova / Filament (extra) | built-in Django admin (free) |
| Prod entry | `public/index.php` | `wsgi.py` / `asgi.py` |

---

## Common mistakes & gotchas

- **Forgetting to activate the venv.** If `python -m django --version` errors with "No module named django", you almost certainly forgot `source .venv/bin/activate`. Your shell prompt should show `(.venv)`.
- **Confusing "view" with "template".** In Django, a *view* is your controller logic; the *template* is the HTML. This single naming swap causes most beginner confusion.
- **Creating an app but not adding it to `INSTALLED_APPS`.** `startapp` only scaffolds files — migrations, templates, and admin won't see the app until it's registered.
- **Not including the app's URLs.** Even with the app registered, its routes don't exist until you `include("blog.urls")` from the root `urls.py`.
- **Editing `wsgi.py`/`asgi.py` thinking it's needed for dev.** It isn't; `runserver` handles dev. Leave them alone until deployment.
- **Leaving `DEBUG=True` (and a committed `SECRET_KEY`) in production.** A serious security hole — `DEBUG=True` leaks your settings and source in error pages.
- **Running `runserver` in production.** It's a dev-only server. Use Gunicorn/Uvicorn behind Nginx.
- **Forgetting `python manage.py migrate` on a fresh project**, then wondering why login/admin fails — the auth/session tables don't exist yet.
- **Naming a project the same as an app** (e.g. project `blog` and app `blog`). Import collisions ensue. Use `config`/`core` for the project package.
- **Using `pip install django` without a version pin.** Pin it (`"Django>=5.0,<6.0"`) and freeze to `requirements.txt` for reproducibility.

## Best practices

- Name the project package `config` (or `core`) and create files in place with the trailing dot: `startproject config .`.
- One venv per project; commit `requirements.txt`, never `.venv/`.
- Keep apps small and single-purpose; you'll have several.
- Give each app its own `urls.py` and `include()` it from the root URLconf; use `app_name` for URL namespacing.
- Run *all* tooling through `python manage.py ...` so the right settings module is always loaded.
- Store secrets and environment-specific values (SECRET_KEY, DEBUG, DB creds) in environment variables, not in `settings.py`.
- Keep `USE_TZ = True` and store datetimes in UTC.
- Run `python manage.py migrate` immediately after creating a project to set up Django's built-in tables.
- Run `python manage.py check` early and often — it surfaces config problems before they bite.

## Interview Q&A

**Q: What does "batteries included" mean in Django, and name a few of those batteries.**
**A:** It means Django ships with the common building blocks of a web app already integrated, so you don't assemble them from third-party libraries. The big ones: the ORM, schema migrations, an auto-generated admin site, a full authentication/permissions system, forms & validation, a template engine, URL routing, and security protections (CSRF, XSS, SQL injection, clickjacking).

**Q: Explain MVT and how it maps to MVC.**
**A:** MVT = Model–View–Template. The **Model** is the data layer (same as MVC). The **Template** is the presentation/HTML — what MVC calls the *view*. The **View** in Django is the request-handling logic — what MVC calls the *controller*. The "controller" role is effectively handled by the framework plus the URL dispatcher (`urls.py`). So MVT is MVC with renamed parts: Django-View = MVC-Controller, Django-Template = MVC-View.

**Q: What is the difference between a Django project and an app?**
**A:** A **project** is the entire web application/deployable unit — it holds settings and wires everything together; you have one. An **app** is a self-contained, reusable module focused on one feature (e.g. `accounts`, `blog`); a project contains many apps, and a good app can be reused across projects. `startproject` creates the project; `startapp` creates an app.

**Q: Why use a virtual environment, and how do you create one?**
**A:** A venv isolates a project's Python dependencies so versions don't collide across projects (and you don't pollute the system Python). Create and activate it with `python3 -m venv .venv` then `source .venv/bin/activate` (macOS/Linux). It's analogous to giving each project its own `vendor/` instead of installing PHP packages globally.

**Q: What's the difference between `wsgi.py` and `asgi.py`?**
**A:** Both are production entry points. `wsgi.py` implements the synchronous WSGI standard — used by servers like Gunicorn/uWSGI for traditional request/response handling. `asgi.py` implements the asynchronous ASGI standard — used by servers like Uvicorn/Daphne to support async views, WebSockets, and long-lived connections. Dev uses `runserver`, so you rarely touch either until deployment.

**Q: What does `manage.py` do and how does it differ from `django-admin`?**
**A:** `manage.py` is a project-local wrapper around `django-admin` that automatically sets `DJANGO_SETTINGS_MODULE` to your project's settings, so commands run with the right configuration. `django-admin` is the global tool (used for `startproject` before a project exists). In practice, inside a project you run everything through `python manage.py ...`.

**Q: Walk me through what happens when a request hits a Django app.**
**A:** The web/WSGI/ASGI server hands the request to Django, which builds an `HttpRequest`. Request-phase middleware runs top-to-bottom (security, sessions, CSRF, auth). The URL dispatcher matches the path (root `urls.py` → `include`d app URLs) and selects a view. The view runs your logic — possibly querying models via the ORM and rendering a template — and returns an `HttpResponse`. Response-phase middleware runs bottom-to-top, then the response goes back to the client.

**Q: How do you register a new app, and what breaks if you don't?**
**A:** Add it to the `INSTALLED_APPS` list in `settings.py` (e.g. `"blog"` or `"blog.apps.BlogConfig"`). Without it, Django won't discover the app's models (so `makemigrations` ignores them), templates, static files, or admin registrations — the app effectively doesn't exist to the framework.

**Q: What is `SECRET_KEY` used for and how should you manage it?**
**A:** It's used for cryptographic signing — session data, CSRF tokens, password-reset tokens, signed cookies. It must stay secret and out of version control; in production load it from an environment variable. If it leaks, rotate it (which invalidates existing sessions/tokens).

**Q: Why must `DEBUG` be `False` in production?**
**A:** With `DEBUG=True`, Django renders detailed error pages exposing stack traces, local variables, settings, and SQL — a major information leak. It also changes static-file handling and disables `ALLOWED_HOSTS` enforcement. Production must set `DEBUG=False` and provide a non-empty `ALLOWED_HOSTS`.

**Q: Is the Django dev server (`runserver`) suitable for production? Why not?**
**A:** No. It's single-process, not performance-tuned, and not security-hardened — it exists for convenience during development (auto-reload, easy startup). In production you run a WSGI/ASGI server such as Gunicorn or Uvicorn, typically behind a reverse proxy like Nginx, pointing at `wsgi.py`/`asgi.py`.

**Q: What is middleware in Django and why does order matter?**
**A:** Middleware is a stack of hooks that wrap every request and response — used for sessions, authentication, CSRF, security headers, etc. On the way in, middleware runs top-to-bottom; on the way out, bottom-to-top. Order matters because some middleware depends on earlier ones (e.g. `AuthenticationMiddleware` needs `SessionMiddleware` to have run first).

## Practice exercises

Try these before peeking at the solutions.

1. **From zero to running server.** Create a folder `shop`, set up a venv, install Django 5.x, and scaffold a project named `config` *in place*. Start the dev server and confirm the rocket page loads. List the exact commands.
2. **Add an app.** Inside the project, create an app called `catalog`, register it, and add a route so that visiting `/catalog/` returns the text "Catalog is live". Show the files you changed/created and their contents.
3. **Identify the files.** For each generated file — `manage.py`, `settings.py`, `urls.py`, `wsgi.py`, `asgi.py` — write one sentence on its job and its Laravel analogue.
4. **Settings audit.** You're handed a `settings.py` with `DEBUG = True`, `ALLOWED_HOSTS = []`, and a hard-coded `SECRET_KEY`. List exactly what you'd change to make it production-safe and why.
5. **Trace a request.** A user hits `GET /catalog/`. Describe, in order, the components the request passes through until a response is returned.

<details><summary>Solutions</summary>

**1. From zero to running server**

```bash
mkdir shop && cd shop
python3 -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install "Django>=5.0,<6.0"
django-admin startproject config .
python manage.py migrate             # set up built-in tables (optional but recommended)
python manage.py runserver
# Visit http://127.0.0.1:8000/  → Django rocket welcome page.
```

**2. Add an app**

```bash
python manage.py startapp catalog
```

```python
# config/settings.py  — add the app
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "catalog",          # <-- added
]
```

```python
# catalog/views.py
from django.http import HttpResponse


def index(request):
    return HttpResponse("Catalog is live")
```

```python
# catalog/urls.py   (new file)
from django.urls import path
from . import views

app_name = "catalog"

urlpatterns = [
    path("", views.index, name="index"),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("catalog/", include("catalog.urls")),   # <-- added
]
```

Visit `http://127.0.0.1:8000/catalog/` → "Catalog is live".

**3. Identify the files**

- `manage.py` — project-local CLI entry point that sets the settings module; you run all commands through it. *Laravel: `artisan`.*
- `settings.py` — all configuration (apps, middleware, DB, secret key, debug, timezone). *Laravel: `config/` + `.env`.*
- `urls.py` — the root URL dispatcher mapping URLs to views. *Laravel: `routes/web.php`.*
- `wsgi.py` — synchronous production entry point for WSGI servers (Gunicorn/uWSGI). *Laravel: `public/index.php` bootstrap.*
- `asgi.py` — asynchronous production entry point for ASGI servers (Uvicorn/Daphne), enabling async/WebSockets. *Laravel: newer concept, no direct analogue.*

**4. Settings audit**

- Set `DEBUG = False` — `True` leaks stack traces, settings, and SQL on error pages.
- Set `ALLOWED_HOSTS = ["yourdomain.com"]` — required (non-empty) when `DEBUG=False`; protects against Host-header attacks.
- Move `SECRET_KEY` out of source into an env var, e.g. `SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]`; rotate the leaked one. Likewise read DB credentials from the environment.
- (Bonus) enable HTTPS/security settings like `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE` for production.

**5. Trace a request**

1. Server (`runserver` in dev / Gunicorn in prod) receives `GET /catalog/`.
2. Django builds an `HttpRequest`.
3. Request-phase middleware runs top-to-bottom (security, session, CSRF, auth).
4. URL dispatcher: `config.urls` matches `catalog/` and delegates via `include` to `catalog.urls`, which matches `""` → `catalog.views.index`.
5. The `index` view runs and returns `HttpResponse("Catalog is live")` (could query models / render a template).
6. Response-phase middleware runs bottom-to-top.
7. Django returns the response to the server, which sends it to the browser.

</details>

## Cheat sheet

```bash
# --- Environment ---
python3 -m venv .venv                 # create venv
source .venv/bin/activate             # activate (Win: .venv\Scripts\Activate.ps1)
deactivate                            # leave venv
python -m pip install "Django>=5.0,<6.0"
python -m pip freeze > requirements.txt

# --- Scaffolding ---
django-admin startproject config .    # create project in current dir
python manage.py startapp blog        # create an app

# --- Daily commands (all via manage.py) ---
python manage.py runserver            # dev server (http://127.0.0.1:8000/)
python manage.py runserver 8080       # custom port
python manage.py migrate              # apply migrations
python manage.py makemigrations       # create migrations from model changes
python manage.py createsuperuser      # admin login
python manage.py shell                # interactive REPL (like artisan tinker)
python manage.py check                # validate project config
```

| Term | One-liner |
|---|---|
| Project | The whole app; holds settings; one per deployment |
| App | Reusable single-purpose module; many per project |
| Model | Data layer (DB table as a Python class) |
| View | Request handler (= MVC controller) |
| Template | HTML with placeholders (= MVC view) |
| `urls.py` | URL → view routing table |
| `settings.py` | All configuration |
| `manage.py` | CLI entry point (= artisan) |
| `wsgi.py` / `asgi.py` | Sync / async production entry points |
| `INSTALLED_APPS` | Where you register apps |
| `MIDDLEWARE` | Ordered request/response pipeline |

**Request flow:** request → middleware (in) → URL resolver → view → (model/template) → response → middleware (out) → client.

**Golden rules:** activate the venv · run everything through `manage.py` · register apps in `INSTALLED_APPS` · `include()` app URLs · `DEBUG=False` + non-empty `ALLOWED_HOSTS` + env-var secrets in production · never use `runserver` in prod.
