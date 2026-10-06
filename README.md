# Python + Django + FastAPI — Backend Developer Mastery Course

> A complete, self-paced course taking you from **zero Python** to **interview-ready backend/API developer** in **5 weeks**.
> Built for someone who already knows **PHP/Laravel** — every topic includes a *"PHP/Laravel bridge"* so you learn by mapping what you already know onto Python.

---

## 🎯 What this course makes you able to do

By the end you will confidently:

- Write idiomatic, modern Python (3.12) — data structures, OOP, decorators, generators, async, typing.
- Build and reason about **Django** apps end to end (ORM, views, auth, migrations, admin).
- Build production REST APIs with **Django REST Framework** and **FastAPI** (+ Pydantic).
- Explain **when to choose Django vs FastAPI**, and the trade-offs (a favorite interview question).
- Handle the "production" topics interviewers probe: testing, databases/SQL, caching, Celery, Docker, security, performance, and system design.
- Answer **150+ real interview questions** across Python, Django, DRF, and FastAPI, and solve common coding challenges.

---

## 🗺️ How to use this course

Each lesson is a standalone markdown file with the same structure:

1. **What you'll learn** — the lesson's goals.
2. **PHP/Laravel bridge** — maps the concept onto what you already know.
3. **Core concepts** — explanations with **complete, runnable examples**.
4. **Common mistakes & gotchas** — what trips people up.
5. **Best practices** — how pros do it.
6. **Interview Q&A** — realistic questions with strong answers.
7. **Practice exercises** — do these; solutions included.
8. **Cheat sheet** — quick recall before the interview.

**Recommended daily flow (~2–3 hrs):**
1. Read the lesson, *typing out* every example yourself in a REPL or `.py` file (don't copy-paste).
2. Break the examples on purpose — change values, see errors. Errors are the fastest teacher.
3. Do the exercises before peeking at solutions.
4. Cover the answers in the Interview Q&A and try to answer aloud first.
5. End each week by skimming all that week's cheat sheets.

---

## ⚙️ One-time setup (do this on Day 1, see Lesson 01)

```bash
# Check Python is installed (need 3.10+, ideally 3.12)
python3 --version

# Create a course workspace + virtual environment (like a per-project Composer install)
mkdir ~/python-practice && cd ~/python-practice
python3 -m venv .venv
source .venv/bin/activate          # macOS/Linux  (Windows: .venv\Scripts\activate)

# You'll install these as the course progresses:
pip install django djangorestframework fastapi "uvicorn[standard]" \
            pydantic sqlalchemy alembic pytest httpx requests
```

---

## 🌉 The big-picture PHP/Laravel → Python map

| You know (PHP/Laravel)        | Python/Django/FastAPI equivalent                         |
|-------------------------------|----------------------------------------------------------|
| `composer`                    | `pip` + `venv` (and `requirements.txt` / `pyproject.toml`)|
| Composer `vendor/`            | `.venv/` virtual environment                             |
| Laravel (full framework)      | **Django** (batteries-included, MVT)                     |
| Lumen / Slim (micro)          | **FastAPI** / Flask                                      |
| Eloquent ORM                  | **Django ORM** / SQLAlchemy                              |
| Blade templates               | Django Template Language                                  |
| Artisan (`php artisan`)       | `manage.py` / `django-admin`                             |
| Migrations                    | Django migrations (`makemigrations`/`migrate`)           |
| Form Requests / Validation    | Django Forms, DRF Serializers, **Pydantic**              |
| API Resources                 | DRF Serializers                                           |
| Middleware                    | Django/FastAPI middleware                                |
| Events & Listeners            | Django signals                                            |
| Service Container / DI        | FastAPI `Depends()` dependency injection                 |
| Queues (Horizon)              | **Celery** + Redis                                       |
| PHPUnit                       | **pytest** / `unittest`                                  |
| `$_GET`/`$_POST`/`Request`    | `request.GET` / Pydantic models / path & query params    |

---

## 📅 5-Week Schedule & Full Index

### Week 1 — Python Fundamentals (`week-1-python-fundamentals/`)
Build a rock-solid Python core. Nothing framework-specific yet.

| # | Lesson |
|---|--------|
| 01 | [Getting Started: install, REPL, venv, pip, project layout](week-1-python-fundamentals/01-getting-started.md) |
| 02 | [Variables & Data Types](week-1-python-fundamentals/02-variables-and-data-types.md) |
| 03 | [Operators & Expressions](week-1-python-fundamentals/03-operators-and-expressions.md) |
| 04 | [Strings & Formatting](week-1-python-fundamentals/04-strings.md) |
| 05 | [Control Flow (if, loops, match-case)](week-1-python-fundamentals/05-control-flow.md) |
| 06 | [Lists & Tuples](week-1-python-fundamentals/06-lists-and-tuples.md) |
| 07 | [Dictionaries & Sets](week-1-python-fundamentals/07-dicts-and-sets.md) |
| 08 | [Comprehensions & Generator Expressions](week-1-python-fundamentals/08-comprehensions.md) |
| 09 | [Functions (args, kwargs, scope, lambda)](week-1-python-fundamentals/09-functions.md) |
| 10 | [Modules, Packages & Imports](week-1-python-fundamentals/10-modules-packages-imports.md) |

### Week 2 — Advanced Python (`week-2-python-advanced/`)
The features interviewers love: OOP, decorators, generators, async, typing.

| # | Lesson |
|---|--------|
| 11 | [OOP: Classes & Objects](week-2-python-advanced/11-oop-classes-objects.md) |
| 12 | [OOP: Inheritance & Polymorphism](week-2-python-advanced/12-oop-inheritance-polymorphism.md) |
| 13 | [Dunder / Magic Methods & Dataclasses](week-2-python-advanced/13-dunder-magic-methods.md) |
| 14 | [Decorators](week-2-python-advanced/14-decorators.md) |
| 15 | [Iterators & Generators](week-2-python-advanced/15-iterators-generators.md) |
| 16 | [Error Handling & Exceptions](week-2-python-advanced/16-error-handling-exceptions.md) |
| 17 | [Context Managers](week-2-python-advanced/17-context-managers.md) |
| 18 | [Type Hints & the `typing` module](week-2-python-advanced/18-type-hints-typing.md) |
| 19 | [Functional Tools (functools, itertools, collections)](week-2-python-advanced/19-functional-tools.md) |
| 20 | [Async & asyncio](week-2-python-advanced/20-async-asyncio.md) |
| 21 | [File I/O, JSON & datetime](week-2-python-advanced/21-file-io-json-datetime.md) |
| 22 | [Pythonic Idioms & PEP 8](week-2-python-advanced/22-pythonic-idioms-pep8.md) |

### Week 3 — Django (`week-3-django/`)
The batteries-included framework, end to end.

| # | Lesson |
|---|--------|
| 23 | [Web Foundations: HTTP, REST, WSGI/ASGI](week-3-django/23-web-http-rest-fundamentals.md) |
| 24 | [Django Intro, Setup & Project Structure](week-3-django/24-django-intro-setup.md) |
| 25 | [URLs & Views (FBV & CBV)](week-3-django/25-django-urls-views.md) |
| 26 | [Templates](week-3-django/26-django-templates.md) |
| 27 | [Models & ORM Basics](week-3-django/27-django-models-orm-basics.md) |
| 28 | [Migrations](week-3-django/28-django-migrations.md) |
| 29 | [QuerySets & Advanced ORM](week-3-django/29-django-queryset-orm-advanced.md) |
| 30 | [Model Relationships](week-3-django/30-django-relationships.md) |
| 31 | [Forms & Validation](week-3-django/31-django-forms.md) |
| 32 | [The Admin Site](week-3-django/32-django-admin.md) |
| 33 | [Authentication & Authorization](week-3-django/33-django-auth.md) |
| 34 | [Middleware & Signals](week-3-django/34-django-middleware-signals.md) |
| 35 | [Settings, Static & Media Files](week-3-django/35-django-settings-static-media.md) |

### Week 4 — DRF & FastAPI (`week-4-drf-and-fastapi/`)
Building real REST APIs in both ecosystems.

| # | Lesson |
|---|--------|
| 36 | [DRF: Intro & Serializers](week-4-drf-and-fastapi/36-drf-intro-serializers.md) |
| 37 | [DRF: Views, ViewSets & Routers](week-4-drf-and-fastapi/37-drf-views-viewsets-routers.md) |
| 38 | [DRF: Auth, Permissions & Throttling](week-4-drf-and-fastapi/38-drf-auth-permissions-throttling.md) |
| 39 | [DRF: Pagination, Filtering & Versioning](week-4-drf-and-fastapi/39-drf-pagination-filtering-versioning.md) |
| 40 | [FastAPI: Intro & Setup](week-4-drf-and-fastapi/40-fastapi-intro-setup.md) |
| 41 | [FastAPI: Path, Query & Body Params](week-4-drf-and-fastapi/41-fastapi-path-query-body.md) |
| 42 | [Pydantic: Models & Validation](week-4-drf-and-fastapi/42-pydantic-models-validation.md) |
| 43 | [FastAPI: Dependency Injection](week-4-drf-and-fastapi/43-fastapi-dependency-injection.md) |
| 44 | [FastAPI: Async & Databases (SQLAlchemy)](week-4-drf-and-fastapi/44-fastapi-async-databases-sqlalchemy.md) |
| 45 | [FastAPI: Auth (JWT & OAuth2)](week-4-drf-and-fastapi/45-fastapi-auth-jwt-oauth2.md) |
| 46 | [FastAPI: Background Tasks, Middleware & Structure](week-4-drf-and-fastapi/46-fastapi-background-middleware-errors.md) |
| 47 | [Django vs FastAPI — The Comparison](week-4-drf-and-fastapi/47-django-vs-fastapi-comparison.md) |

### Week 5 — Production & Interview Prep (`week-5-production-and-interview/`)
The senior-flavored topics + pure interview drilling.

| # | Lesson |
|---|--------|
| 48 | [Testing (pytest, Django & FastAPI)](week-5-production-and-interview/48-testing.md) |
| 49 | [Databases & SQL (PostgreSQL)](week-5-production-and-interview/49-databases-sql-postgres.md) |
| 50 | [Caching, Redis & Celery](week-5-production-and-interview/50-caching-redis-celery.md) |
| 51 | [Deployment: Docker, Gunicorn & Uvicorn](week-5-production-and-interview/51-deployment-docker.md) |
| 52 | [Security Best Practices](week-5-production-and-interview/52-security.md) |
| 53 | [Performance Optimization](week-5-production-and-interview/53-performance.md) |
| 54 | [System Design for Python Backends](week-5-production-and-interview/54-system-design.md) |
| 55 | [Interview Questions — Python](week-5-production-and-interview/55-interview-questions-python.md) |
| 56 | [Interview Questions — Django & DRF](week-5-production-and-interview/56-interview-questions-django.md) |
| 57 | [Interview Questions — FastAPI](week-5-production-and-interview/57-interview-questions-fastapi.md) |
| 58 | [Coding Challenges](week-5-production-and-interview/58-coding-challenges.md) |

---

## ✅ Progress Tracker

Tick these off as you go.

- [ ] **Week 1** — Python Fundamentals (Lessons 01–10)
- [ ] **Week 2** — Advanced Python (Lessons 11–22)
- [ ] **Week 3** — Django (Lessons 23–35)
- [ ] **Week 4** — DRF & FastAPI (Lessons 36–47)
- [ ] **Week 5** — Production & Interview (Lessons 48–58)
- [ ] Final pass: re-read every cheat sheet + all interview Q&A (Lessons 55–58)

---

*Tip: study in order — each lesson assumes the ones before it. Good luck. You've got this. 🐍🚀*
