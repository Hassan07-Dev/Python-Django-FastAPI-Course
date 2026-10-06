# 33 — Authentication & Authorization

> Week 3 · Django

## What you will learn

- The difference between **authentication** ("who are you?") and **authorization** ("what are you allowed to do?")
- The built-in `User` model: what fields and methods it ships with
- How to create users and the critical `set_password()` / `check_password()` methods (password hashing)
- `authenticate()`, `login()`, and `logout()` — the login flow
- How Django tracks logged-in users with **sessions**
- `request.user` and the `AnonymousUser` object
- Protecting views: `@login_required`, `LoginRequiredMixin`
- The permission system: default `add` / `change` / `delete` / `view` perms, custom perms, and **groups**
- Checking permissions: `user.has_perm()`, `@permission_required`, `PermissionRequiredMixin`
- **Customizing the user model** with `AbstractUser` / `AbstractBaseUser`, and why you must do it on day one
- Password validators and how Django enforces password strength
- A side-by-side comparison with Laravel auth / guards / gates / policies

## PHP/Laravel bridge

In Laravel, `php artisan make:auth` (or Breeze/Jetstream) scaffolds a `users` table, an `Auth` facade, guards, and the `Hash` facade for bcrypt. Django ships the equivalent **out of the box** the moment you create a project: a `User` model, a session-backed auth system, the `auth` middleware, and PBKDF2 password hashing — no scaffolding command needed. Where Laravel separates **gates** (closures for one-off checks) and **policies** (class-based authorization per model), Django uses a single, database-backed **permission** system with optional **groups** (Laravel has no first-class groups; you'd reach for a package like spatie/laravel-permission). Laravel's `Auth::user()` is Django's `request.user`; Laravel's `auth` middleware is Django's `@login_required`; Laravel's `$this->authorize()` / `@can` is Django's `user.has_perm()` / `@permission_required`.

| Laravel | Django |
|---|---|
| `Auth::user()` | `request.user` |
| `Auth::attempt($creds)` | `authenticate()` + `login()` |
| `Auth::logout()` | `logout(request)` |
| `Hash::make($pw)` | `user.set_password(pw)` |
| `Hash::check($pw, $hash)` | `user.check_password(pw)` |
| `auth` middleware | `@login_required` / `LoginRequiredMixin` |
| Gate / Policy | permissions + `has_perm()` |
| Guest user (null) | `AnonymousUser` instance |
| spatie roles/permissions | built-in `Group` + `Permission` |

---

## Core concept: the built-in User model

Django's auth system lives in `django.contrib.auth`, which is enabled by default in `INSTALLED_APPS`. The default user model is `django.contrib.auth.models.User`. After your first `migrate`, you have a `auth_user` table ready to go.

The model gives you these fields:

| Field | Type | Notes |
|---|---|---|
| `username` | char (unique) | the default login identifier |
| `password` | char | stores the **hash**, never plaintext |
| `email` | email | not unique by default (a common gotcha!) |
| `first_name`, `last_name` | char | optional |
| `is_active` | bool | inactive users cannot authenticate |
| `is_staff` | bool | can access the Django admin |
| `is_superuser` | bool | bypasses **all** permission checks |
| `last_login`, `date_joined` | datetime | managed automatically |

Create a superuser from the command line:

```bash
python manage.py createsuperuser
# Username: admin
# Email address: admin@example.com
# Password: ********
# Superuser created successfully.
```

### Important: always reference the user model indirectly

Never hard-code `from django.contrib.auth.models import User` in your app code. If you later swap in a custom user model (and you will — see below), every import breaks. Instead:

```python
# In models.py, for ForeignKey/OneToOne relations:
from django.conf import settings
from django.db import models

class Article(models.Model):
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    # settings.AUTH_USER_MODEL is a string like "auth.User" or "accounts.User"

# In views/services, to GET the active user class at runtime:
from django.contrib.auth import get_user_model

User = get_user_model()  # returns whatever AUTH_USER_MODEL points to
```

**Why two different tools?** `settings.AUTH_USER_MODEL` is a *string* used in model definitions so migrations don't create a circular import. `get_user_model()` returns the actual *class* and is safe to call anywhere at runtime (views, scripts, tests).

---

## Core concept: creating users & password hashing

A password must **never** be stored in plaintext. Django hashes it with PBKDF2 (SHA-256) by default, with a per-password random salt. The two methods you must know:

- `user.set_password(raw_password)` — hashes the password and stores it on the instance (does **not** save to the DB).
- `user.check_password(raw_password)` — returns `True`/`False` by re-hashing and comparing in constant time.

```python
from django.contrib.auth import get_user_model

User = get_user_model()

# WRONG — stores plaintext, login will never work:
# user = User(username="alice", password="s3cret")

# RIGHT — let the manager hash it:
user = User.objects.create_user(username="alice", email="alice@example.com",
                                password="s3cret")
# create_user() calls set_password() internally, then saves.

print(user.password)
# => 'pbkdf2_sha256$870000$abc123...$base64hash='   (the stored hash)

print(user.check_password("s3cret"))   # => True
print(user.check_password("wrong"))    # => False

# To change a password later:
user.set_password("new-stronger-password")
user.save()                            # set_password does NOT save; you must

# For a superuser programmatically:
admin = User.objects.create_superuser(username="root", email="r@x.com",
                                       password="adminpass")
```

The hash format is self-describing: `algorithm$iterations$salt$hash`. When you bump iterations or change the default hasher, Django transparently **upgrades** a user's hash on their next successful login. This re-hash-and-save is triggered by the auth backend (which passes a `setter` callback into `check_password()`) during `authenticate()`, so it happens automatically — you don't manage it yourself.

---

## Core concept: the login flow — authenticate, login, logout

There are two distinct steps people often conflate:

1. **`authenticate(request, username=..., password=...)`** — verifies credentials against the configured backends. Returns a `User` object if valid, else `None`. It does **not** log anyone in; it just checks. It also rejects users where `is_active=False`.
2. **`login(request, user)`** — attaches the authenticated user to the **session**, so subsequent requests know who they are.

```python
from django.contrib.auth import authenticate, login, logout
from django.shortcuts import render, redirect
from django.contrib import messages


def login_view(request):
    if request.method == "POST":
        username = request.POST["username"]
        password = request.POST["password"]
        user = authenticate(request, username=username, password=password)
        if user is not None:
            login(request, user)          # writes user id into the session
            return redirect("dashboard")
        else:
            messages.error(request, "Invalid credentials.")
    return render(request, "registration/login.html")


def logout_view(request):
    logout(request)                       # flushes the session
    return redirect("login")
```

In practice you'll often just use Django's built-in `LoginView` / `LogoutView` instead of writing these by hand:

```python
# urls.py
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path("login/",  auth_views.LoginView.as_view(), name="login"),
    path("logout/", auth_views.LogoutView.as_view(), name="logout"),
]
```

`LoginView` looks for a template at `registration/login.html`, validates with `AuthenticationForm`, and calls `login()` for you on success.

---

## Core concept: sessions — how Django remembers you

HTTP is stateless, so how does the server know request #2 came from the same logged-in person as request #1? **Sessions.**

The flow:

1. On `login()`, Django stores the user's primary key in a server-side session store (the `django_session` table by default).
2. It sends the browser a `sessionid` cookie containing an opaque random key — **not** the user id, and **not** the password.
3. On every later request, `SessionMiddleware` reads the cookie, looks up the session, and `AuthenticationMiddleware` sets `request.user` accordingly.

```python
# settings.py — relevant middleware order (already there by default)
MIDDLEWARE = [
    "django.contrib.sessions.middleware.SessionMiddleware",       # must come...
    # ...
    "django.contrib.auth.middleware.AuthenticationMiddleware",    # ...before this
]

# Session is the default; the cookie only holds the session KEY.
SESSION_ENGINE = "django.contrib.sessions.backends.db"  # default: db-backed
SESSION_COOKIE_AGE = 1209600           # 2 weeks, in seconds
SESSION_COOKIE_HTTPONLY = True         # JS can't read it (XSS defense)
SESSION_COOKIE_SECURE = True           # HTTPS only — set True in production
SESSION_EXPIRE_AT_BROWSER_CLOSE = False
```

You can also stash arbitrary data per-user in the session dict, just like Laravel's `session()`:

```python
request.session["cart_items"] = 3
request.session.get("cart_items", 0)   # => 3
del request.session["cart_items"]
request.session.set_expiry(3600)       # expire this session in 1 hour
```

> Compared to Laravel: Django's default session is **database-backed**, whereas Laravel defaults to `file`. Both put only an opaque id in the cookie. For high traffic, switch to a cache backend (`cache` / Redis) in both frameworks.

---

## Core concept: request.user and AnonymousUser

Because of `AuthenticationMiddleware`, **every** request has a `request.user`. It is never `None`:

- For a logged-in request: an instance of your user model.
- For a logged-out request: an `AnonymousUser` instance.

This lets your code stay branch-free in many cases — `AnonymousUser` answers the same questions as a real user:

```python
from django.http import HttpResponse


def whoami(request):
    u = request.user
    print(u.is_authenticated)   # True for real users, False for AnonymousUser
    print(u.is_anonymous)       # the inverse
    # AnonymousUser implements the same interface:
    #   .is_active        => False
    #   .is_staff         => False
    #   .has_perm("x")    => False  (always)
    #   .groups / .username => empty
    if request.user.is_authenticated:
        return HttpResponse(f"Hi {request.user.username}")
    return HttpResponse("Hi guest")
```

**Gotcha:** `is_authenticated` is an *attribute* (property), not a method — never write `request.user.is_authenticated()`. It used to be a method in old Django; calling it now is a truthy bound-method bug that's always "logged in."

---

## Core concept: protecting views

### Function-based views: `@login_required`

```python
from django.contrib.auth.decorators import login_required


@login_required                       # redirects to settings.LOGIN_URL if anonymous
def dashboard(request):
    return render(request, "dashboard.html")


# Customize the redirect target:
@login_required(login_url="/accounts/login/")
def billing(request):
    ...
```

```python
# settings.py
LOGIN_URL = "login"               # where @login_required sends anonymous users
LOGIN_REDIRECT_URL = "dashboard"  # where LoginView sends you after success
LOGOUT_REDIRECT_URL = "login"
```

By default the user is bounced to `LOGIN_URL` with a `?next=/billing/` query param, so after logging in they return to where they were headed.

### Class-based views: `LoginRequiredMixin`

For CBVs, mix in `LoginRequiredMixin` — and it **must come first** in the inheritance list so its dispatch logic runs before the view's:

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import ListView
from .models import Invoice


class InvoiceList(LoginRequiredMixin, ListView):   # mixin FIRST
    model = Invoice
    login_url = "/accounts/login/"   # optional override
    redirect_field_name = "next"     # optional
```

---

## Core concept: permissions, groups & authorization

Authentication answered "who are you." Authorization answers "what may you do." Django's system is **model-centric**.

### Default permissions

For every model, Django auto-creates four permissions on migrate:

| Codename | Meaning |
|---|---|
| `add_<model>` | create rows |
| `change_<model>` | edit rows |
| `delete_<model>` | delete rows |
| `view_<model>` | read rows (added in Django 2.1) |

Permissions are identified app-wide by `"<app_label>.<codename>"`, e.g. `"blog.add_article"`. They live in the `auth_permission` table.

### Custom permissions

Declare your own in a model's `Meta`:

```python
from django.db import models


class Report(models.Model):
    title = models.CharField(max_length=200)

    class Meta:
        permissions = [
            ("can_publish", "Can publish a report"),
            ("can_export", "Can export reports to PDF"),
        ]
        # Referenced as "appname.can_publish", "appname.can_export"
```

Run `makemigrations` + `migrate` and the new permissions appear in `auth_permission`.

### Groups — bundles of permissions

A `Group` is a named bucket of permissions. Assign a user to a group and they inherit all of its permissions — perfect for roles like "Editor" or "Moderator." (This is what Laravel needs a package for; Django has it built in.)

```python
from django.contrib.auth.models import Group, Permission
from django.contrib.auth import get_user_model

User = get_user_model()

editors = Group.objects.create(name="Editors")

# Attach permissions to the group:
publish = Permission.objects.get(codename="can_publish")
change = Permission.objects.get(codename="change_report")
editors.permissions.add(publish, change)

# Put a user in the group:
alice = User.objects.get(username="alice")
alice.groups.add(editors)

# Now alice has both permissions via the group:
print(alice.has_perm("reports.can_publish"))   # => True

# Direct per-user permission (bypassing groups):
alice.user_permissions.add(Permission.objects.get(codename="can_export"))
```

> **Permission caching gotcha:** permissions are cached on the user object after the first check within a request. If you add a permission and re-check on the *same* in-memory instance, re-fetch the user (`User.objects.get(pk=...)`) to clear the cache.

### Checking permissions

**In Python / views — `user.has_perm()`:**

```python
def publish_report(request, pk):
    if not request.user.has_perm("reports.can_publish"):
        return HttpResponseForbidden("Nope.")
    ...

# Check several at once:
request.user.has_perms(["reports.can_publish", "reports.can_export"])

# Superusers short-circuit to True for ANY has_perm() check:
# (request.user.is_superuser == True  =>  has_perm() always returns True)
```

**Function views — `@permission_required`:**

```python
from django.contrib.auth.decorators import permission_required


@permission_required("reports.can_publish", raise_exception=True)
def publish(request):
    # raise_exception=True => 403 Forbidden instead of redirect to login
    ...
```

**Class-based views — `PermissionRequiredMixin`:**

```python
from django.contrib.auth.mixins import PermissionRequiredMixin
from django.views.generic import CreateView
from .models import Report


class ReportCreate(PermissionRequiredMixin, CreateView):
    model = Report
    fields = ["title"]
    permission_required = "reports.add_report"
    # Multiple perms (ALL required):
    # permission_required = ("reports.add_report", "reports.can_publish")
    raise_exception = True   # 403 instead of login redirect
```

**In templates:**

```django
{% if perms.reports.can_publish %}
  <button>Publish</button>
{% endif %}
```

---

## Core concept: customizing the user model

> **The single most important production decision in this lesson:** swap in a custom user model **before your first migration**, even if you don't customize anything yet. Changing `AUTH_USER_MODEL` after tables exist is painful — it involves wiping migrations or fragile data migrations because half of Django (admin, FKs, the `auth_user` table) is already bound to the original model.

Two routes:

### `AbstractUser` — keep all the built-in fields, add your own (recommended default)

You inherit `username`, `email`, `is_staff`, `password`, etc., and just extend:

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    bio = models.TextField(blank=True)
    phone = models.CharField(max_length=20, blank=True)
    # Make email unique & the login field instead of username:
    email = models.EmailField(unique=True)

    USERNAME_FIELD = "email"          # log in with email
    REQUIRED_FIELDS = ["username"]    # prompted by createsuperuser besides USERNAME_FIELD
```

```python
# settings.py — tell Django to use it (do this on day one!)
AUTH_USER_MODEL = "accounts.User"
```

### `AbstractBaseUser` — total control over the schema

Use this only when you need a fundamentally different shape (e.g., no username at all). You must define the fields and a custom `Manager`:

```python
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models


class UserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra):
        if not email:
            raise ValueError("Email is required")
        email = self.normalize_email(email)
        user = self.model(email=email, **extra)
        user.set_password(password)      # hashing happens here
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra):
        extra.setdefault("is_staff", True)
        extra.setdefault("is_superuser", True)
        if extra.get("is_staff") is not True:
            raise ValueError("Superuser must have is_staff=True.")
        if extra.get("is_superuser") is not True:
            raise ValueError("Superuser must have is_superuser=True.")
        return self.create_user(email, password, **extra)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    full_name = models.CharField(max_length=150, blank=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = UserManager()

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = []   # email is already USERNAME_FIELD, so don't repeat it
```

`PermissionsMixin` gives you `is_superuser`, `groups`, `user_permissions`, and `has_perm()` so the permission system keeps working. `AbstractBaseUser` gives you `password`, `last_login`, and the hashing machinery.

**Rule of thumb:** start with `AbstractUser`. Reach for `AbstractBaseUser` only when its fields genuinely don't fit.

---

## Core concept: password validators

Django enforces password quality through `AUTH_PASSWORD_VALIDATORS`, applied whenever a form or `validate_password()` checks a new password (e.g. `UserCreationForm`, admin, password change). Note: `set_password()` itself does **not** validate — validation is the form/serializer's job.

```python
# settings.py (default project ships with these four)
AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
     "OPTIONS": {"min_length": 10}},     # tighten the default 8
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]
```

Call it manually when you accept passwords outside a form (e.g. a DRF serializer):

```python
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError

try:
    validate_password("12345678", user=some_user)
except ValidationError as exc:
    print(exc.messages)
    # => ['This password is too common.', 'This password is entirely numeric.']
    # (Both fire here because "12345678" is a common password AND all digits.
    #  A value like "password123" would trip only the common-password check,
    #  since it contains letters and so is not "entirely numeric".)
```

To change the hashing algorithm (e.g. add Argon2), reorder `PASSWORD_HASHERS` — the first entry is used for new hashes, the rest are accepted for verification and transparently upgraded:

```python
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",   # pip install argon2-cffi
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
]
```

---

## Common mistakes & gotchas

- **Setting `user.password = "plain"` directly.** This stores plaintext; login will silently never succeed. Always use `set_password()` or `create_user()`.
- **Forgetting `user.save()` after `set_password()`.** `set_password()` only mutates the in-memory object.
- **Calling `request.user.is_authenticated()` as a method.** It's a property — the call returns a truthy bound method, so the check is *always* true.
- **Changing `AUTH_USER_MODEL` after the first migration.** Set your custom user model before the very first `migrate`. Retrofitting is a major surgery.
- **Importing `User` from `django.contrib.auth.models` in app code.** Use `get_user_model()` / `settings.AUTH_USER_MODEL` so a future swap doesn't break everything.
- **Assuming `email` is unique.** The default `User.email` is **not** unique — two accounts can share an email unless you enforce it.
- **`@permission_required` without `raise_exception=True`.** Without it, a logged-in user lacking the perm is *redirected to login* (confusing) instead of getting a clean 403.
- **Mixin ordering.** `LoginRequiredMixin` / `PermissionRequiredMixin` must come **before** the generic view class, or their `dispatch()` never runs.
- **Permission cache staleness.** After granting a permission, the same in-memory user object still reports the old answer; re-fetch the user.
- **Expecting `is_staff` to mean "has permissions."** `is_staff` only controls admin-site *access*. `is_superuser` is the one that bypasses all permission checks.
- **`authenticate()` returns `None` for inactive users**, even with correct credentials — check `is_active` if you're debugging a "valid password but won't log in" bug.

## Best practices

- Create a **custom user model on day one** (`AbstractUser`), even if empty. Future-you will be grateful.
- Always reference users via `settings.AUTH_USER_MODEL` (in models) and `get_user_model()` (at runtime).
- Make `email` unique if you use it to log in; consider it as `USERNAME_FIELD`.
- Use **groups for roles**, direct `user_permissions` only for one-off exceptions.
- Prefer the built-in `LoginView`/`LogoutView`/`PasswordChangeView` over hand-rolled auth views.
- Set `raise_exception=True` on permission decorators/mixins so unauthorized requests get a 403, not a login redirect.
- In production: `SESSION_COOKIE_SECURE = True`, `SESSION_COOKIE_HTTPONLY = True`, `CSRF_COOKIE_SECURE = True`, and consider `SESSION_EXPIRE_AT_BROWSER_CLOSE`.
- Tighten `MinimumLengthValidator` and consider adding Argon2 to `PASSWORD_HASHERS`.
- Never log or serialize the `password` field; exclude it from APIs and admin display.
- Use `@login_required(login_url=...)` / `LOGIN_URL` consistently so redirects are predictable.

## Interview Q&A

**Q: What's the difference between `authenticate()` and `login()`?**
**A:** `authenticate(request, **creds)` verifies credentials against the auth backends and returns a `User` object (or `None`); it does not establish a session. `login(request, user)` takes an already-authenticated user and persists their id into the session so future requests are recognized. You almost always call them in sequence: authenticate, then if non-None, login.

**Q: How does Django know a user is logged in across requests?**
**A:** Via sessions. On login, Django stores the user's PK in a server-side session and sets a `sessionid` cookie holding only an opaque random key. On each request, `SessionMiddleware` loads the session and `AuthenticationMiddleware` populates `request.user`. The password and user id are never in the cookie.

**Q: Why must you use `set_password()` instead of assigning to `.password`?**
**A:** `password` stores a hash, not plaintext. `set_password()` runs the configured hasher (PBKDF2-SHA256 by default) with a random salt and writes the resulting `algorithm$iterations$salt$hash` string. Direct assignment stores raw text, so `check_password()` — which hashes the input and compares — will never match.

**Q: What is `AnonymousUser` and why does it exist?**
**A:** It's a stand-in object that `AuthenticationMiddleware` assigns to `request.user` for unauthenticated requests, so `request.user` is never `None`. It implements the same interface as a real user (`is_authenticated` is `False`, `has_perm()` always `False`), letting view code avoid null checks.

**Q: How do permissions and groups relate?**
**A:** A `Permission` is an atomic capability (e.g. `blog.add_article`). A `Group` is a named collection of permissions. A user's effective permissions = their direct `user_permissions` plus the permissions of every group they belong to. Groups model roles; direct permissions handle exceptions.

**Q: What permissions does Django create automatically, and how are they named?**
**A:** For each model it creates `add_<model>`, `change_<model>`, `delete_<model>`, and `view_<model>` on migrate. They're referenced as `"<app_label>.<codename>"`, e.g. `"shop.change_order"`. Custom ones are declared in the model's `Meta.permissions`.

**Q: Why customize the user model, and why early?**
**A:** You'll eventually want email-based login, extra profile fields, or unique-email constraints. Django binds the user table to FKs, the admin, and migrations from the first `migrate`, so changing `AUTH_USER_MODEL` afterward is extremely disruptive. The official guidance is to set a custom model (usually `AbstractUser`) on project creation, before any migration.

**Q: `AbstractUser` vs `AbstractBaseUser`?**
**A:** `AbstractUser` keeps all built-in fields (username, email, flags) and lets you add more — the easy, recommended path. `AbstractBaseUser` gives only the password/last_login machinery; you define every field and a custom manager, and add `PermissionsMixin` to retain the permission system. Use it only when the default field set fundamentally doesn't fit.

**Q: How do you protect a view for logged-in users only?**
**A:** Function views: `@login_required` (redirects anonymous users to `LOGIN_URL` with a `?next=`). Class-based views: inherit `LoginRequiredMixin`, placed first in the MRO. For permission checks use `@permission_required("app.codename", raise_exception=True)` or `PermissionRequiredMixin`.

**Q: What does `is_staff` vs `is_superuser` control?**
**A:** `is_staff` only grants access to the Django admin site. `is_superuser` short-circuits every `has_perm()`/`has_module_perms()` check to `True`, effectively granting all permissions. They're independent booleans; a superuser is usually also staff so they can reach the admin.

**Q: What are password validators and when do they run?**
**A:** They're the entries in `AUTH_PASSWORD_VALIDATORS` (min length, common-password blocklist, numeric-only, user-attribute similarity). They run during form/serializer validation or explicit `validate_password()` calls — not inside `set_password()`. So when you accept passwords outside a Django form (e.g. a DRF view), call `validate_password()` yourself.

**Q: How does Django compare to Laravel's authorization?**
**A:** Laravel splits authorization into gates (closures) and policies (model-bound classes), and needs a package for roles. Django uses one database-backed permission system with built-in groups. `has_perm()`/`@permission_required` are the rough equivalent of `@can`/`$this->authorize()`, and `request.user` maps to `Auth::user()`.

## Practice exercises

1. **Programmatic user lifecycle.** In a Django shell, create a user named `bob` with a password, verify `check_password()` returns `True` for the right password and `False` for a wrong one, then change his password and confirm the old one no longer works.

2. **Login flow.** Write a function-based `login_view` that uses `authenticate()` + `login()`, redirects to `"home"` on success, and re-renders with an error message on failure. Add `LOGIN_REDIRECT_URL` and `LOGIN_URL` to settings.

3. **Protect two views.** Protect a function view with `@login_required` and a `ListView` with `LoginRequiredMixin`. Confirm an anonymous request to each is redirected to the login page with a `?next=` parameter.

4. **Roles with groups.** Create an `Editors` group, give it `change_report` and a custom `can_publish` permission, add a user to it, and assert `user.has_perm("reports.can_publish")` is `True`. Then verify a non-member returns `False`.

5. **Permission-gated CBV.** Write a `CreateView` for `Report` guarded by `PermissionRequiredMixin` requiring `reports.add_report` with `raise_exception=True`. Verify a user without the perm gets a 403.

6. **Custom user model.** Start a fresh app, define a `User(AbstractUser)` with `email` unique and `USERNAME_FIELD = "email"`, set `AUTH_USER_MODEL`, migrate, and create a superuser with the new email-based prompt.

<details><summary>Solutions</summary>

**1. Programmatic user lifecycle**

```python
# python manage.py shell
from django.contrib.auth import get_user_model
User = get_user_model()

bob = User.objects.create_user(username="bob", password="initialpw123")
assert bob.check_password("initialpw123") is True
assert bob.check_password("nope") is False

bob.set_password("newpw456")
bob.save()                                  # don't forget to save
assert bob.check_password("initialpw123") is False
assert bob.check_password("newpw456") is True
print("OK")
```

**2. Login flow**

```python
# views.py
from django.contrib.auth import authenticate, login
from django.contrib import messages
from django.shortcuts import render, redirect

def login_view(request):
    if request.method == "POST":
        user = authenticate(request,
                            username=request.POST["username"],
                            password=request.POST["password"])
        if user is not None:
            login(request, user)
            return redirect("home")
        messages.error(request, "Invalid credentials.")
    return render(request, "registration/login.html")
```

```python
# settings.py
LOGIN_URL = "login"
LOGIN_REDIRECT_URL = "home"
```

**3. Protect two views**

```python
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import ListView
from .models import Report

@login_required
def secret(request):
    return render(request, "secret.html")

class ReportList(LoginRequiredMixin, ListView):   # mixin first
    model = Report
# Anonymous GET /secret/ -> 302 to /login/?next=/secret/
```

**4. Roles with groups**

```python
from django.contrib.auth.models import Group, Permission
from django.contrib.auth import get_user_model
User = get_user_model()

editors = Group.objects.create(name="Editors")
editors.permissions.add(
    Permission.objects.get(codename="change_report"),
    Permission.objects.get(codename="can_publish"),
)
alice = User.objects.create_user(username="alice", password="pw12345678")
alice.groups.add(editors)
assert alice.has_perm("reports.can_publish") is True

bob = User.objects.create_user(username="bob2", password="pw12345678")
assert bob.has_perm("reports.can_publish") is False
```

**5. Permission-gated CBV**

```python
from django.contrib.auth.mixins import PermissionRequiredMixin
from django.views.generic import CreateView
from .models import Report

class ReportCreate(PermissionRequiredMixin, CreateView):
    model = Report
    fields = ["title"]
    permission_required = "reports.add_report"
    raise_exception = True   # 403 instead of login redirect
```

**6. Custom user model**

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    email = models.EmailField(unique=True)
    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["username"]
```

```python
# settings.py
AUTH_USER_MODEL = "accounts.User"
```

```bash
python manage.py makemigrations accounts
python manage.py migrate
python manage.py createsuperuser
# Email address: admin@example.com   <- prompts for email now, not username
```

</details>

## Cheat sheet

```python
# --- get the user model ---
from django.contrib.auth import get_user_model
User = get_user_model()                     # runtime class
settings.AUTH_USER_MODEL                    # "app.User" string for FKs

# --- create + passwords ---
User.objects.create_user(username, email, password)   # hashes + saves
User.objects.create_superuser(username, email, password)
user.set_password(raw); user.save()         # hash; MUST save
user.check_password(raw)                     # -> bool

# --- login flow ---
from django.contrib.auth import authenticate, login, logout
u = authenticate(request, username=x, password=y)  # -> User | None
login(request, u)                            # start session
logout(request)                              # end session

# --- request.user ---
request.user.is_authenticated                # property, NOT a method
request.user.is_anonymous
request.user.is_staff / .is_superuser

# --- protect views ---
@login_required                              # FBV
@permission_required("app.codename", raise_exception=True)
class V(LoginRequiredMixin, ListView): ...   # CBV, mixin FIRST
class W(PermissionRequiredMixin, CreateView):
    permission_required = "app.add_model"; raise_exception = True

# --- permissions / groups ---
user.has_perm("app.codename")                # superuser => always True
user.has_perms(["a.x", "a.y"])
from django.contrib.auth.models import Group, Permission
g = Group.objects.create(name="Editors")
g.permissions.add(perm); user.groups.add(g)
user.user_permissions.add(perm)              # direct grant
# default perms per model: add_ / change_ / delete_ / view_
```

```python
# --- settings essentials ---
AUTH_USER_MODEL = "accounts.User"            # set BEFORE first migrate
LOGIN_URL = "login"
LOGIN_REDIRECT_URL = "home"
LOGOUT_REDIRECT_URL = "login"
SESSION_COOKIE_SECURE = True                 # prod
SESSION_COOKIE_HTTPONLY = True
AUTH_PASSWORD_VALIDATORS = [ ... ]           # MinimumLength, CommonPassword, ...
PASSWORD_HASHERS = ["...Argon2PasswordHasher", "...PBKDF2PasswordHasher"]
```

```django
{# --- templates --- #}
{% if user.is_authenticated %} Hi {{ user.username }} {% endif %}
{% if perms.reports.can_publish %} <button>Publish</button> {% endif %}
```

| Task | API |
|---|---|
| Hash password | `set_password()` / `create_user()` |
| Verify password | `check_password()` |
| Check login state | `request.user.is_authenticated` |
| Require login (FBV/CBV) | `@login_required` / `LoginRequiredMixin` |
| Check permission | `user.has_perm("app.code")` |
| Require permission (FBV/CBV) | `@permission_required` / `PermissionRequiredMixin` |
| Role bundle | `Group` + `group.permissions` |
| Custom user | `AbstractUser` (easy) / `AbstractBaseUser` (full control) |
