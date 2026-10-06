# 34 — Middleware & Signals

> Week 3 · Django

Middleware and signals are Django's two big "hook into the framework's lifecycle" mechanisms. Middleware wraps **every** request/response. Signals let one piece of code react to events emitted by another piece of code without the two knowing about each other. If you've used Laravel's HTTP middleware and its event/listener system, you already know the shapes — this lesson maps them onto Django and shows you the idiomatic Python way.

## What you will learn

- The middleware concept and the "onion" request/response pipeline.
- The `MIDDLEWARE` setting, why **order matters**, and what each built-in middleware does.
- How to write custom middleware in the modern factory/callable style with `get_response`.
- The extra hooks: `process_view`, `process_exception`, `process_template_response`.
- How to **short-circuit** a request (return early without hitting the view).
- Signals: the publisher/subscriber (observer) pattern in Django.
- Built-in model signals (`pre_save`, `post_save`, `pre_delete`, `post_delete`, `m2m_changed`) and request signals.
- Connecting receivers with `@receiver`, the `sender` argument, and **where** to register them (`apps.py` → `ready()`).
- When signals help and when they create hidden coupling (and why explicit calls are often better).
- A side-by-side comparison with Laravel middleware and events/listeners.

## PHP/Laravel bridge

Django **middleware** is almost exactly Laravel HTTP middleware: classes that wrap the request/response, registered in a list, executed in order, where each one can inspect/modify the request, call the next layer (Laravel's `$next($request)`, Django's `get_response(request)`), then inspect/modify the response on the way out. Both form an "onion." Django **signals** map onto Laravel **events and listeners**: a sender "dispatches" something, and decoupled "listeners" (receivers) react. Django's built-in model signals (`post_save`, `pre_delete`, …) are the analog of Laravel's **Eloquent model events** (`created`, `saving`, `deleting`, …). One key cultural difference: the Django community is more skeptical of signals than Laravel is of events — Pythonistas often prefer an explicit method call over a magic signal, for the same "explicit is better than implicit" reasons we'll discuss.

---

## Core concept: the middleware onion

A **middleware** is a layer that sits between the web server and your view. Every incoming request passes *down* through all middleware to reach the view, and every response passes back *up* through them in reverse order. Picture an onion: the request pierces inward through each layer to the core (the view), then the response travels back out through the same layers.

```
Request  ─►  SecurityMiddleware ─► SessionMiddleware ─► AuthMiddleware ─► ... ─► VIEW
                                                                                  │
Response ◄─  SecurityMiddleware ◄─ SessionMiddleware ◄─ AuthMiddleware ◄─ ... ◄──┘
```

The crucial mental model: **the first middleware in the list is the outermost layer.** On the way *in* it runs first; on the way *out* it runs last. The last middleware in the list is closest to the view: it runs last on the way in and first on the way out. This is exactly why order matters.

### The MIDDLEWARE setting

Middleware lives in `settings.py` as an ordered list of dotted import paths. A fresh Django 5.x project gives you:

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

What each built-in does, top to bottom:

| Middleware | Job | Why its position matters |
|---|---|---|
| `SecurityMiddleware` | HTTPS redirects, HSTS, several security headers. | Outermost so it can redirect/reject before anything else runs. |
| `SessionMiddleware` | Loads the session from the session cookie into `request.session`; saves it on the way out. | Must run **before** auth, because auth reads the session. |
| `CommonMiddleware` | URL normalization (e.g. `APPEND_SLASH`), `DISALLOWED_USER_AGENTS`, sends `Content-Length`. | General request hygiene. |
| `CsrfViewMiddleware` | Verifies the CSRF token on unsafe methods (POST/PUT/PATCH/DELETE). | Must run **before** the view so it can reject forged POSTs. |
| `AuthenticationMiddleware` | Attaches `request.user` (a `User` or `AnonymousUser`), lazily loaded from the session. | Must run **after** `SessionMiddleware`. |
| `MessageMiddleware` | Enables the `messages` framework (flash messages). | Needs sessions/cookies, so it sits after them. |
| `XFrameOptionsMiddleware` | Sets `X-Frame-Options` to prevent clickjacking. | Sets a response header, fine near the end. |

> **Beginner question: "Why does order matter so much?"** Because each middleware depends on what earlier ones set up. `AuthenticationMiddleware` reads `request.session`, which `SessionMiddleware` populated. Put auth before sessions and `request.user` breaks. The CSRF middleware must come *before* the view so it can reject a bad token *before* your view code runs. Treat the default order as load-bearing — only insert your own middleware where you understand the dependencies.

---

## Core concept: writing custom middleware (factory style)

Modern Django middleware is a **callable that returns a callable**. The outer callable (the "factory") receives `get_response` once at startup. It returns the actual middleware function, which Django calls once **per request**. `get_response` is "the rest of the onion" — calling it descends to the next middleware (and eventually the view) and hands you back the response.

### Function-based middleware

```python
# myapp/middleware.py
import time
import logging

logger = logging.getLogger(__name__)


def timing_middleware(get_response):
    # --- One-time configuration & setup (runs once at startup) ---
    logger.info("timing_middleware initialised")

    def middleware(request):
        # --- Code here runs on the way IN, before the view ---
        start = time.perf_counter()

        response = get_response(request)  # descend to next layer / the view

        # --- Code here runs on the way OUT, after the view ---
        elapsed_ms = (time.perf_counter() - start) * 1000
        response["X-Elapsed-ms"] = f"{elapsed_ms:.1f}"
        logger.info("%s %s took %.1f ms", request.method, request.path, elapsed_ms)
        return response

    return middleware
```

Register it by appending the dotted path to `MIDDLEWARE`:

```python
# settings.py
MIDDLEWARE = [
    # ... built-ins ...
    "myapp.middleware.timing_middleware",
]
```

Mental model in Laravel terms: the code **before** `get_response(request)` is what runs before `$next($request)`; the code **after** is what runs after. `get_response` *is* `$next`.

### Class-based middleware

Identical semantics, OOP shape. `__init__` is the factory (runs once), `__call__` runs per request:

```python
# myapp/middleware.py
class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response          # stored once at startup

    def __call__(self, request):
        start = time.perf_counter()
        response = self.get_response(request)     # call the next layer
        response["X-Elapsed-ms"] = f"{(time.perf_counter() - start) * 1000:.1f}"
        return response
```

> **Beginner question: "Function or class?"** They are equivalent. Use a function for simple cross-cutting logic; use a class when you want `process_view` / `process_exception` hooks (below) or want to keep helper methods tidy.

### The extra hooks: process_view, process_exception, process_template_response

Beyond the in/out wrapping, Django will call specially-named methods on your middleware **if they exist**. These give you finer-grained interception points:

```python
# myapp/middleware.py
class AuditMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # request phase (before view)
        response = self.get_response(request)
        # response phase (after view)
        return response

    def process_view(self, request, view_func, view_args, view_kwargs):
        # Called AFTER URL resolution but BEFORE the view runs.
        # You now know exactly which view will handle the request.
        # Return None  -> continue as normal.
        # Return an HttpResponse -> short-circuit; the view is skipped.
        view_name = getattr(view_func, "__name__", repr(view_func))
        request._audit_view = view_name
        return None

    def process_exception(self, request, exception):
        # Called ONLY if the view raised an unhandled exception.
        # Return None  -> let Django's normal exception handling proceed.
        # Return an HttpResponse -> swallow the error, return this instead.
        logger.exception("View %s raised", getattr(request, "_audit_view", "?"))
        return None  # don't hide the error; just log it

    def process_template_response(self, request, response):
        # Called only when the view returns a TemplateResponse
        # (i.e. a response with a .render() method, not yet rendered).
        # Must return a response object.
        response.context_data["audited"] = True
        return response
```

When each hook fires, in order, for a single request:

1. Each middleware's **request phase** (top → bottom), i.e. the code before `get_response`.
2. `process_view` on each middleware (top → bottom), once the view is resolved.
3. **The view runs.**
4. If the view raised: `process_exception` (bottom → top) until one returns a response.
5. If the view returned a `TemplateResponse`: `process_template_response` (bottom → top).
6. Each middleware's **response phase** (bottom → top), i.e. the code after `get_response`.

> Note the asymmetry: request-phase code and `process_view` run **top→bottom**; everything on the way out (`process_exception`, `process_template_response`, response-phase) runs **bottom→top**. That's the onion.

---

## Core concept: short-circuiting a request

Sometimes you want to answer a request **without ever reaching the view** — maintenance mode, IP banning, a fast health check, rate limiting. You short-circuit by **returning an `HttpResponse` instead of calling `get_response`**.

```python
# myapp/middleware.py
from django.http import HttpResponse, JsonResponse
from django.conf import settings


def maintenance_middleware(get_response):
    def middleware(request):
        # Allow a health check through even during maintenance.
        if getattr(settings, "MAINTENANCE_MODE", False) and request.path != "/healthz":
            # Return early: the view (and inner middleware) never run.
            return HttpResponse("Down for maintenance", status=503)
        return get_response(request)

    return middleware
```

A blocklist example that returns JSON:

```python
def ip_blocklist_middleware(get_response):
    blocked = {"10.0.0.5", "203.0.113.7"}

    def middleware(request):
        client_ip = request.META.get("REMOTE_ADDR")
        if client_ip in blocked:
            return JsonResponse({"detail": "Forbidden"}, status=403)  # short-circuit
        return get_response(request)

    return middleware
```

> **Important short-circuit detail:** only middleware **outer than** (i.e. listed *before*) yours got to run their request phase. When you return early, inner middleware never runs at all, but every middleware that already ran its request phase **still runs its response phase** on your early response. So a short-circuit at position 5 still passes back up through middleware 4, 3, 2, 1. This is the same as a Laravel middleware returning a response instead of calling `$next`.

---

## Core concept: signals — the publisher/subscriber pattern

A **signal** decouples "something happened" from "react to it." A **sender** dispatches a signal; one or more **receivers** (subscribed functions) run in response. The sender does not know who is listening, and listeners do not modify the sender's code. This is the classic **observer / publisher-subscriber** pattern, and it's Django's analog of Laravel's events and listeners.

Django ships several built-in signals. The ones you'll use most:

| Signal | Sent when | Common `kwargs` to the receiver |
|---|---|---|
| `pre_save` | Before a model's `.save()` writes to the DB | `sender`, `instance`, `raw`, `using`, `update_fields` |
| `post_save` | After `.save()` succeeds | `sender`, `instance`, `created` (bool), `raw`, `using`, `update_fields` |
| `pre_delete` | Before a model's `.delete()` | `sender`, `instance`, `using`, `origin` |
| `post_delete` | After `.delete()` | `sender`, `instance`, `using`, `origin` |
| `m2m_changed` | When a ManyToMany relation is added/removed/cleared | `sender`, `instance`, `action`, `pk_set`, `model`, `reverse` |
| `request_started` / `request_finished` | At the very start/end of request handling | `sender`, `environ` (started) |

### Connecting a receiver with @receiver

A receiver is just a function. The `@receiver` decorator subscribes it. **Always accept `**kwargs`** — Django may add new keyword arguments in future versions, and missing ones will crash your receiver.

```python
# blog/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.core.mail import send_mail

from blog.models import Comment


@receiver(post_save, sender=Comment)
def notify_on_new_comment(sender, instance, created, **kwargs):
    # `created` is True only on INSERT, False on UPDATE.
    if not created:
        return
    send_mail(
        subject="New comment posted",
        message=f"Comment #{instance.pk}: {instance.body[:80]}",
        from_email="noreply@example.com",
        recipient_list=["editor@example.com"],
    )
```

### The `sender` argument

`sender` filters **which** publisher you listen to. With `sender=Comment`, the receiver fires only for `Comment.save()`, not for `Post.save()` or `User.save()`. **Omit `sender` and you listen to every model's saves** — almost always a mistake for model signals (it's slow and noisy). Inside the receiver, `sender` is the model **class** and `instance` is the specific row being saved.

```python
# Listen to ALL post_save (rarely what you want):
@receiver(post_save)
def log_every_save(sender, instance, created, **kwargs):
    print(f"{sender.__name__} saved (created={created}) pk={instance.pk}")

# => Comment saved (created=True) pk=42
# => Post saved (created=False) pk=7
```

You can also connect without the decorator, which is handy when wiring up signals dynamically:

```python
post_save.connect(notify_on_new_comment, sender=Comment)
# and to detach:
post_save.disconnect(notify_on_new_comment, sender=Comment)
```

### m2m_changed in detail

ManyToMany changes don't go through `save()`, so they need their own signal. The `action` kwarg tells you what happened; `pk_set` holds the affected primary keys.

```python
# blog/signals.py
from django.db.models.signals import m2m_changed
from blog.models import Post


@receiver(m2m_changed, sender=Post.tags.through)   # note: the THROUGH model
def tags_changed(sender, instance, action, pk_set, **kwargs):
    # action is one of: pre_add, post_add, pre_remove, post_remove,
    #                   pre_clear, post_clear
    if action == "post_add":
        print(f"Post {instance.pk} gained tags {pk_set}")
    elif action == "post_remove":
        print(f"Post {instance.pk} lost tags {pk_set}")

# post.tags.add(tag1, tag2)
# => Post 3 gained tags {11, 12}
```

> **Gotcha:** the `sender` for `m2m_changed` is the **through model** (`Post.tags.through`), not the model itself. This trips up everyone the first time.

### Request signals

`request_started` and `request_finished` fire around the whole request cycle. They're useful for cross-cutting bookkeeping that isn't tied to a model:

```python
# core/signals.py
from django.core.signals import request_started, request_finished
from django.dispatch import receiver


@receiver(request_started)
def on_request_started(sender, environ, **kwargs):
    print("Request started:", environ.get("PATH_INFO"))


@receiver(request_finished)
def on_request_finished(sender, **kwargs):
    print("Request finished")
```

(For per-request timing you'd usually reach for **middleware** instead, since middleware has the request/response objects right there.)

---

## Core concept: WHERE to register receivers — apps.py ready()

Receivers only fire if their module has been **imported**, which connects the decorators. If nobody imports `signals.py`, your receivers silently never run — a top-3 Django beginner trap. The idiomatic fix is to import them in your app config's `ready()` method.

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"

    def ready(self):
        # Import for side effects: this runs the @receiver decorators
        # exactly once when Django finishes loading apps.
        from blog import signals  # noqa: F401
```

Make sure the app is referenced by its config (Django 5.x autodiscovers `default` configs, but being explicit is fine):

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "blog",            # Django finds blog/apps.py:BlogConfig automatically
    # or explicitly: "blog.apps.BlogConfig",
]
```

> **Why not just import `signals` at the top of `models.py`?** It often *works*, but it risks circular imports (signals import models, models import signals) and runs at an unpredictable time during startup. `ready()` runs **after** all apps and models are fully loaded, which is the safe, documented place. Don't query the database in `ready()` though — migrations and other management commands also trigger it.

---

## Core concept: when signals help vs. when they hurt

Signals are powerful but **implicit**. A junior reading `comment.save()` has no way to know an email gets sent, a cache gets busted, and a counter increments — none of that is visible at the call site. That invisible action-at-a-distance is the same critique many Laravel teams have of overusing model events.

**Good uses of signals:**

- Reacting to events in **code you don't own** (e.g. `post_save` on Django's `User`, or a third-party app's model) where you can't edit the source.
- Truly cross-cutting concerns that should apply no matter where the save happens — audit logging, cache invalidation, search-index updates.
- Decoupling apps that shouldn't import each other.

**When to prefer an explicit call instead:**

- When you own the model and the reaction is part of your own domain logic. A plain method (or a service function) is easier to find, test, and reason about:

```python
# Implicit (signal): action-at-a-distance, easy to miss
@receiver(post_save, sender=Order)
def charge_card_on_order(sender, instance, created, **kwargs):
    if created:
        billing.charge(instance)   # surprise side effect on every Order.save()!

# Explicit (preferred when you own the flow): obvious at the call site
class OrderService:
    def place_order(self, order):
        order.save()
        billing.charge(order)      # you can SEE this happens
```

- When ordering or transactions matter. Signal receivers run **synchronously inside** the `save()` call, before the surrounding transaction commits. If you must act only after a successful commit, use `transaction.on_commit(...)` rather than `post_save` directly:

```python
from django.db import transaction

@receiver(post_save, sender=Order)
def index_order(sender, instance, created, **kwargs):
    # Don't index until the transaction actually commits, or a rollback
    # will leave your search index referencing a row that doesn't exist.
    transaction.on_commit(lambda: search.index(instance))
```

- For heavy work (sending email, calling an API), **don't block the request** inside a signal. Enqueue a background job (Celery, etc.) instead.

> Rule of thumb: reach for signals when the sender shouldn't know about the receiver. If the sender legitimately *should* know — because it's all your own domain logic — an explicit call is clearer, and clarity wins.

---

## Middleware vs. signals — when to use which

| Use **middleware** when… | Use **signals** when… |
|---|---|
| The concern is about the **HTTP request/response** (auth, headers, timing, blocking). | The concern is about a **domain event** (a model was saved/deleted). |
| You need to short-circuit or modify every response. | You want to react to something happening anywhere in the codebase. |
| You have the `request`/`response` objects. | There may be no request at all (e.g. a management command saves a model). |

---

## Laravel ↔ Django quick map

| Laravel | Django |
|---|---|
| HTTP middleware (`handle($request, $next)`) | Middleware (`get_response(request)`) |
| `$next($request)` | `get_response(request)` |
| Middleware registered in `Kernel` / route groups | `MIDDLEWARE` list in `settings.py` |
| Returning a response instead of `$next` | Returning `HttpResponse` instead of `get_response` |
| Events + Listeners | Signals + receivers |
| `Event::dispatch(...)` | `signal.send(sender=..., ...)` |
| `EventServiceProvider` `$listen` map | `@receiver` + `apps.py` `ready()` |
| Eloquent model events (`created`, `deleting`, …) | Model signals (`post_save`, `pre_delete`, …) |
| `protected $dispatchesEvents` on a model | built-in `post_save` etc. on every model |

---

## Defining and sending your own custom signal

You're not limited to built-ins. Define a `Signal`, send it, and subscribe like any other.

```python
# billing/signals.py
import django.dispatch

# A custom signal — no required args list in modern Django; just document the kwargs.
invoice_paid = django.dispatch.Signal()
```

```python
# billing/services.py
from billing.signals import invoice_paid

def mark_paid(invoice):
    invoice.status = "paid"
    invoice.save()
    # `sender` is conventionally the class doing the sending.
    invoice_paid.send(sender=invoice.__class__, invoice=invoice, amount=invoice.total)
```

```python
# billing/receivers.py
from django.dispatch import receiver
from billing.signals import invoice_paid

@receiver(invoice_paid)
def send_receipt(sender, invoice, amount, **kwargs):
    print(f"Receipt: invoice {invoice.pk} paid, amount {amount}")
```

`send()` returns a list of `(receiver, return_value)` tuples and raises if any receiver throws. Use `send_robust()` if you want one failing receiver to not break the others (it returns the exception instead of raising).

---

## Common mistakes & gotchas

- **Forgetting to import `signals.py`.** No import → decorators never run → receiver silently does nothing. Wire it in `apps.py` `ready()`.
- **Not accepting `**kwargs` in receivers.** Django passes extra kwargs; without `**kwargs` you get `TypeError`. Every receiver should end its params with `**kwargs`.
- **Omitting `sender` on a model signal** when you meant to target one model — you'll fire for *every* model's save and likely cause recursion or slowdowns.
- **Infinite recursion in `post_save`.** Calling `instance.save()` inside a `post_save` receiver re-triggers the signal forever. Use `update_fields`, a guard flag, or `Model.objects.filter(pk=...).update(...)` (which bypasses `save()` and its signals).
- **Putting custom middleware in the wrong position.** Placing auth-dependent middleware before `AuthenticationMiddleware` means `request.user` isn't set yet.
- **Wrong middleware return contract.** `process_view`/`process_exception` must return either `None` (continue) or an `HttpResponse`. Returning anything else breaks the pipeline.
- **Assuming `m2m_changed` sender is the model.** It's the **through** model (`Post.tags.through`).
- **Expecting `post_save` to mean "committed".** Receivers run inside the transaction. Use `transaction.on_commit()` for after-commit work.
- **Doing slow/blocking work in middleware or signals.** Both run synchronously in the request path; offload to a task queue.
- **Querying the DB in `AppConfig.ready()`.** It runs during migrations too and the schema may not exist yet.

## Best practices

- Keep middleware **thin and fast** — it runs on *every* request.
- Put one concern per middleware; compose by ordering rather than cramming logic together.
- Use `process_view`/`process_exception` only when you genuinely need view-level or error-level interception; otherwise the simple `get_response` wrapper is enough.
- Register receivers in `apps.py` `ready()`; keep them in a dedicated `signals.py` (or `receivers.py`).
- Always include `**kwargs`; always pass `sender=` for model signals.
- Prefer **explicit calls / service objects** for your own domain logic; reserve signals for decoupling and for code you don't own.
- For anything heavy in a signal, enqueue a background job and/or wrap in `transaction.on_commit`.
- Name custom signals and document their kwargs at the definition site.

## Interview Q&A

**Q: Explain the Django middleware request/response flow. Why does order in `MIDDLEWARE` matter?**
**A:** Middleware forms an onion. On the way in, Django runs each middleware's request phase top-to-bottom until it reaches the view; on the way out it runs response phases bottom-to-top. The first entry is the outermost layer. Order matters because middleware depends on earlier ones — e.g. `AuthenticationMiddleware` reads `request.session`, which `SessionMiddleware` must populate first, and `CsrfViewMiddleware` must run before the view to reject forged requests.

**Q: How do you write a custom middleware in modern Django?**
**A:** A callable factory that receives `get_response` once at startup and returns a per-request callable. Code before `get_response(request)` runs on the way in; code after runs on the way out. As a class, `__init__(self, get_response)` is the factory and `__call__(self, request)` runs per request. Optional `process_view`, `process_exception`, and `process_template_response` methods add finer hooks.

**Q: What's the difference between `process_view` and the normal `__call__` request phase?**
**A:** The request phase runs before URL resolution is associated with your code path and doesn't know which view will handle the request. `process_view` runs **after** URL resolution, so it receives the resolved `view_func` plus its args/kwargs — useful for per-view decisions. Returning an `HttpResponse` from it short-circuits and skips the view.

**Q: How do you short-circuit a request in middleware?**
**A:** Return an `HttpResponse` instead of calling `get_response(request)`. Inner middleware and the view are skipped, but every middleware that already ran its request phase still runs its response phase on your early response. Common for maintenance mode, IP bans, and rate limiting.

**Q: What are Django signals and what problem do they solve?**
**A:** Signals implement the observer/publisher-subscriber pattern: a sender dispatches a signal and decoupled receivers react, without the sender knowing who's listening. They're useful for cross-cutting concerns and for reacting to events in code you don't own (e.g. the built-in `User` model).

**Q: Name the main model signals and when each fires.**
**A:** `pre_save`/`post_save` (before/after `.save()`; `post_save` gets `created`), `pre_delete`/`post_delete` (before/after `.delete()`), and `m2m_changed` (when a ManyToMany set changes, with an `action` like `post_add`). `request_started`/`request_finished` wrap the request cycle.

**Q: Why must signal receivers accept `**kwargs`, and why pass `sender=`?**
**A:** Django may send additional keyword arguments (now or in future versions); `**kwargs` keeps the receiver from raising `TypeError`. `sender=` filters which publisher you listen to — for model signals, without it you fire for every model's save, which is slow and error-prone.

**Q: Where should you register receivers, and why not in `models.py`?**
**A:** In the app config's `ready()` method (`apps.py`), importing your `signals` module so the `@receiver` decorators run once after all apps load. Importing in `models.py` risks circular imports and runs at an unpredictable point in startup; `ready()` is the documented, safe place.

**Q: What's the `created` flag in `post_save`?**
**A:** `True` when the save inserted a new row, `False` on an update. It lets a single receiver distinguish "newly created" from "edited."

**Q: When are signals a bad idea?**
**A:** When they hide important domain logic behind an innocuous `.save()` — action-at-a-distance that's hard to find and test. If you own the model and the reaction is core domain behavior, an explicit method or service call is clearer. Signals shine for decoupling and for reacting to code you can't modify.

**Q: A `post_save` receiver calls `instance.save()` and the server hangs. Why?**
**A:** Infinite recursion: each `save()` re-emits `post_save`. Fix by updating via `Model.objects.filter(pk=...).update(...)` (bypasses signals), guarding with a flag, or using `update_fields` and a condition so you don't re-save unnecessarily.

**Q: How do you run code only after the DB transaction commits?**
**A:** Wrap it in `django.db.transaction.on_commit(callback)`. `post_save` fires inside the transaction, so indexing/notifying directly there can reference data that gets rolled back. `on_commit` runs only on a successful commit.

## Practice exercises

1. **Timing header.** Write a function-based middleware that adds an `X-Process-Time` response header with the elapsed milliseconds for every request. Register it last in `MIDDLEWARE`.

2. **Maintenance mode with short-circuit.** Write a middleware that, when `settings.MAINTENANCE_MODE` is `True`, returns a `503` for all paths **except** `/healthz`. Verify that response-phase middleware above it still runs.

3. **Audit logging via `post_save`.** Given a `Comment` model, log a line whenever a *new* comment is created (not on edits). Put the receiver in `comments/signals.py` and wire it up in `apps.py`.

4. **m2m on tags.** For `Post.tags` (a ManyToMany to `Tag`), print which tag PKs were added or removed using `m2m_changed`. Remember the sender is the through model.

5. **Custom signal.** Define an `invoice_paid` signal, send it from a `mark_paid(invoice)` function, and write a receiver that prints a receipt. Use `send_robust` so a failing receiver can't break the caller.

6. **Refactor away a smelly signal.** You find a `post_save` on `Order` that charges a credit card on creation. Explain why this is risky and rewrite it as an explicit service call.

<details><summary>Solutions</summary>

**1. Timing header**
```python
# core/middleware.py
import time

def process_time_middleware(get_response):
    def middleware(request):
        start = time.perf_counter()
        response = get_response(request)
        response["X-Process-Time"] = f"{(time.perf_counter() - start) * 1000:.1f}ms"
        return response
    return middleware

# settings.py
MIDDLEWARE = [
    # ... built-ins ...
    "core.middleware.process_time_middleware",
]
```

**2. Maintenance mode**
```python
# core/middleware.py
from django.conf import settings
from django.http import HttpResponse

def maintenance_middleware(get_response):
    def middleware(request):
        if getattr(settings, "MAINTENANCE_MODE", False) and request.path != "/healthz":
            return HttpResponse("Maintenance", status=503)  # short-circuit
        return get_response(request)
    return middleware
```
Any middleware listed *above* this one already ran its request phase, so its response phase still runs on the `503` (e.g. the timing middleware from #1 would still stamp `X-Process-Time`).

**3. Audit logging via post_save**
```python
# comments/signals.py
import logging
from django.db.models.signals import post_save
from django.dispatch import receiver
from comments.models import Comment

logger = logging.getLogger(__name__)

@receiver(post_save, sender=Comment)
def log_new_comment(sender, instance, created, **kwargs):
    if created:
        logger.info("New comment %s by %s", instance.pk, instance.author_id)

# comments/apps.py
from django.apps import AppConfig

class CommentsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "comments"

    def ready(self):
        from comments import signals  # noqa: F401
```

**4. m2m on tags**
```python
# blog/signals.py
from django.db.models.signals import m2m_changed
from django.dispatch import receiver
from blog.models import Post

@receiver(m2m_changed, sender=Post.tags.through)
def on_tags_changed(sender, instance, action, pk_set, **kwargs):
    if action == "post_add":
        print(f"Post {instance.pk} added tags {pk_set}")
    elif action == "post_remove":
        print(f"Post {instance.pk} removed tags {pk_set}")
```

**5. Custom signal**
```python
# billing/signals.py
import django.dispatch
invoice_paid = django.dispatch.Signal()

# billing/services.py
from billing.signals import invoice_paid

def mark_paid(invoice):
    invoice.status = "paid"
    invoice.save(update_fields=["status"])
    invoice_paid.send_robust(sender=invoice.__class__, invoice=invoice)

# billing/receivers.py
from django.dispatch import receiver
from billing.signals import invoice_paid

@receiver(invoice_paid)
def print_receipt(sender, invoice, **kwargs):
    print(f"Receipt for invoice {invoice.pk}")
```
`send_robust` returns `(receiver, exception_or_result)` tuples instead of raising, so a broken receiver won't break `mark_paid`.

**6. Refactor away a smelly signal**
Risky because `Order.save()` silently charges a card — invisible at the call site, fires for every code path that saves an `Order` (admin, data migration, tests, fixtures), and runs inside the transaction before commit. Rewrite:
```python
class OrderService:
    def place_order(self, order):
        order.save()
        from django.db import transaction
        transaction.on_commit(lambda: billing.charge(order))  # visible, after commit
```
Now charging is explicit, happens only where intended, and only after a successful commit.

</details>

## Cheat sheet

```python
# --- Custom middleware (function) ---
def my_middleware(get_response):           # factory: runs once
    def middleware(request):
        # before view (request phase, top->bottom)
        response = get_response(request)   # descend; omit to short-circuit
        # after view (response phase, bottom->top)
        return response
    return middleware

# --- Class middleware + hooks ---
class M:
    def __init__(self, get_response): self.get_response = get_response
    def __call__(self, request):     return self.get_response(request)
    def process_view(self, request, view_func, args, kwargs): ...   # before view
    def process_exception(self, request, exc): ...                  # on view error
    def process_template_response(self, request, response): ...     # TemplateResponse

# --- Short-circuit ---
return HttpResponse("nope", status=503)    # instead of get_response(request)

# --- Receiver ---
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=MyModel)
def handler(sender, instance, created, **kwargs):   # ALWAYS **kwargs
    if created: ...

# --- Register: blog/apps.py ---
def ready(self):
    from blog import signals  # noqa

# --- Custom signal ---
import django.dispatch
my_signal = django.dispatch.Signal()
my_signal.send(sender=Cls, foo=1)          # or send_robust(...)

# --- After commit ---
from django.db import transaction
transaction.on_commit(lambda: do_thing())
```

| Built-in middleware (default order) | Built-in model signals |
|---|---|
| Security → Session → Common → Csrf → Auth → Messages → XFrameOptions | `pre_save`, `post_save` (`created`), `pre_delete`, `post_delete`, `m2m_changed` |

| Flow direction | Runs… |
|---|---|
| Request phase + `process_view` | top → bottom |
| `process_exception`, `process_template_response`, response phase | bottom → top |
