# 38 — DRF: Auth, Permissions & Throttling

> Week 4 · DRF & FastAPI

Up to now your DRF views have happily served data to anyone who asked. Real APIs don't work like that: they need to know *who is calling* (authentication), decide *whether that caller is allowed to do this* (authorization / permissions), and protect themselves from *being hammered* (throttling). DRF treats all three as pluggable layers that run in a fixed order on every request — and once you learn the pattern, wiring up token auth, role checks, object ownership rules, and rate limits becomes almost declarative. This lesson is the security backbone of every DRF project you'll build, and it's a guaranteed interview topic.

## What you will learn

- The precise difference between **authentication** and **authorization** (and why DRF keeps them separate)
- DRF's built-in **authentication classes**: `SessionAuthentication`, `BasicAuthentication`, `TokenAuthentication`
- **JWT** with `djangorestframework-simplejwt`: the obtain / refresh / rotate flow
- Setting `DEFAULT_AUTHENTICATION_CLASSES` globally vs. per view
- Built-in **permission classes**: `AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`
- Writing **custom permissions** with `has_permission` and object-level `has_object_permission`
- **Throttling**: `AnonRateThrottle`, `UserRateThrottle`, scoped throttles, and the `DEFAULT_THROTTLE_RATES` config
- How the whole request pipeline fits together, and how it maps to **Laravel Sanctum / Passport / middleware / Gates & Policies**

## PHP/Laravel bridge

You've done all of this in Laravel, just under different names. **Authentication** is Laravel's `auth` guards — `Sanctum` for SPA/token auth, `Passport` for full OAuth2/JWT. DRF's `TokenAuthentication` ≈ Sanctum's personal access tokens; DRF + `simplejwt` ≈ Passport's JWT grant. **Authorization** is Laravel's **Gates and Policies** plus the `can:` middleware — DRF's permission classes are the direct analog, and an object-level `has_object_permission` is exactly a Policy method like `update(User $user, Post $post)`. **Throttling** is Laravel's `throttle:60,1` middleware and `RateLimiter` facade — DRF's throttle classes and `DEFAULT_THROTTLE_RATES` do the same job. The big mental shift: in Laravel you attach these as **middleware on routes**; in DRF they are **class attributes on views** (`authentication_classes`, `permission_classes`, `throttle_classes`) that DRF runs automatically inside `APIView.initial()`.

## Core concept: authentication vs. authorization

These two words sound alike and beginners conflate them constantly. Keep them straight:

| | Authentication | Authorization |
|---|---|---|
| Question it answers | **Who are you?** | **Are you allowed to do this?** |
| DRF layer | authentication classes | permission classes |
| Runs… | first | second (needs the user from step 1) |
| Failure status | **401** Unauthorized | **403** Forbidden |
| Laravel analog | guards / Sanctum / Passport | Gates / Policies / `can:` |

The order matters: you can't decide *whether someone may delete a post* until you know *who they are*. DRF enforces exactly this order. On every request, `APIView.initial()` runs:

1. **`perform_authentication`** — tries each authentication class until one succeeds, setting `request.user` and `request.auth`.
2. **`check_permissions`** — runs every permission class's `has_permission()`.
3. **`check_throttles`** — runs every throttle class.

Only then does your `get()` / `post()` handler run. (Object-level permission checks happen later, when you call `self.check_object_permissions(request, obj)` — usually via `get_object()`.)

> Note on status codes: DRF returns **401** if *no* authentication succeeded and the failed authenticator advertises a `WWW-Authenticate` header; otherwise an unauthenticated-but-denied request gets **403**. `SessionAuthentication`, for example, yields 403 because it doesn't send a `WWW-Authenticate` challenge.

## Core concept: authentication classes

An authentication class inspects the incoming request (a header, a cookie, the session) and returns either `(user, auth)` or `None` (meaning "not my kind of credential, try the next one"). DRF tries them **in order** and stops at the first that returns a user.

### The built-ins

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.BasicAuthentication",
    ],
}
```

- **`SessionAuthentication`** — uses Django's normal login session cookie. Perfect when your API is consumed by a same-site browser app (and by the DRF browsable API). Because it relies on cookies, it **enforces CSRF protection** on unsafe methods (POST/PUT/PATCH/DELETE). This is the one beginners trip over: a `403 CSRF Failed` from an AJAX call almost always means you forgot the CSRF token while using SessionAuthentication.
- **`BasicAuthentication`** — reads the `Authorization: Basic <base64(user:pass)>` header. Sends credentials on *every* request, so only use it over HTTPS, and really only for quick testing or server-to-server scripts.
- **`TokenAuthentication`** — the classic DRF token. The client sends `Authorization: Token <key>`; DRF looks the key up in a database table and resolves the user.

### TokenAuthentication setup

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "rest_framework",
    "rest_framework.authtoken",   # creates the authtoken_token table
]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],
}
```

```bash
python manage.py migrate          # creates the token table
```

DRF ships an endpoint that exchanges username+password for a token:

```python
# urls.py
from rest_framework.authtoken.views import obtain_auth_token
from django.urls import path

urlpatterns = [
    path("api/auth-token/", obtain_auth_token),  # POST {username, password} -> {"token": "..."}
]
```

```bash
# Obtain a token
curl -X POST http://localhost:8000/api/auth-token/ \
     -d "username=alice&password=s3cret"
# => {"token":"9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"}

# Use it on a protected endpoint
curl http://localhost:8000/api/posts/ \
     -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
```

The downside of `TokenAuthentication`: tokens are **opaque** (no expiry, one per user by default) and require a **DB lookup on every request**. That's where JWT comes in.

## Core concept: JWT with djangorestframework-simplejwt

A **JWT (JSON Web Token)** is a self-contained, cryptographically signed token. The server can verify it *without a database lookup* because the signature proves it issued the token, and the payload carries the user id and an expiry. The standard library for this in DRF is **`djangorestframework-simplejwt`**.

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}
```

simplejwt uses **two tokens**:

- **access token** — short-lived (default 5 min); sent on every request as `Authorization: Bearer <token>`.
- **refresh token** — long-lived (default 1 day); used *only* to mint a fresh access token when the old one expires, so the user doesn't have to log in again.

Why two? If an access token leaks, it's useless within minutes. The refresh token is sent far less often (only to the refresh endpoint), shrinking its attack surface.

### The obtain / refresh flow

```python
# urls.py
from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path("api/token/", TokenObtainPairView.as_view(), name="token_obtain_pair"),
    path("api/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
    path("api/token/verify/", TokenVerifyView.as_view(), name="token_verify"),
]
```

```bash
# 1. Log in: trade username+password for a token PAIR
curl -X POST http://localhost:8000/api/token/ \
     -H "Content-Type: application/json" \
     -d '{"username": "alice", "password": "s3cret"}'
# => {
#      "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9....",
#      "access":  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...."
#    }

# 2. Call a protected endpoint with the ACCESS token (note: Bearer, not Token)
curl http://localhost:8000/api/posts/ \
     -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...."

# 3. Access expired (401)? Use the REFRESH token to get a new access token
curl -X POST http://localhost:8000/api/token/refresh/ \
     -H "Content-Type: application/json" \
     -d '{"refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...."}'
# => {"access": "eyJhbGciOiJIUzI1Ni...new..."}
```

### Tuning lifetimes and rotation

```python
# settings.py
from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,      # issue a brand-new refresh token on each refresh
    "BLACKLIST_AFTER_ROTATION": True,   # invalidate the old refresh token (needs the blacklist app)
    "AUTH_HEADER_TYPES": ("Bearer",),   # the keyword in the Authorization header
    # SIGNING_KEY is omitted on purpose: it defaults to settings.SECRET_KEY with HS256.
    # Do NOT set "SIGNING_KEY": None explicitly — that overrides the default and breaks signing.
}

# To use blacklisting / logout, add the app and migrate:
INSTALLED_APPS = [
    # ...
    "rest_framework_simplejwt.token_blacklist",
]
```

A JWT is **stateless**: there's no "delete the row to log out." With `token_blacklist` enabled you *can* blacklist a refresh token (a logout-like effect), but the already-issued access token stays valid until it expires — which is exactly why you keep access tokens short.

```python
# A logout view that blacklists the refresh token (revoke future refreshes)
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework_simplejwt.tokens import RefreshToken


class LogoutView(APIView):
    def post(self, request):
        try:
            token = RefreshToken(request.data["refresh"])
            token.blacklist()
        except Exception:
            return Response(status=status.HTTP_400_BAD_REQUEST)
        return Response(status=status.HTTP_205_RESET_CONTENT)
```

### TokenAuthentication vs. JWT — the comparison

| | `TokenAuthentication` (authtoken) | simplejwt (JWT) |
|---|---|---|
| Storage | row in DB per token | nothing stored (stateless) |
| Verification | DB lookup every request | signature check, no DB hit |
| Expiry | none by default | built-in, short-lived access |
| Refresh | re-login | refresh token endpoint |
| Revocation | delete the row | blacklist (opt-in) |
| Header | `Authorization: Token …` | `Authorization: Bearer …` |
| Best for | simple internal APIs | SPAs, mobile, microservices |

## Core concept: setting authentication globally vs. per view

Set a project-wide default once, then override on the few views that differ:

```python
# settings.py — applies to EVERY view unless overridden
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
        "rest_framework.authentication.SessionAuthentication",  # so the browsable API still works
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

```python
# views.py — override on a single view
from rest_framework.authentication import TokenAuthentication
from rest_framework.permissions import AllowAny
from rest_framework.generics import ListAPIView


class PublicFeedView(ListAPIView):
    authentication_classes = []        # no auth at all
    permission_classes = [AllowAny]    # anyone can read
    # ... queryset / serializer_class
```

Function-based views use decorators:

```python
from rest_framework.decorators import api_view, authentication_classes, permission_classes
from rest_framework_simplejwt.authentication import JWTAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response


@api_view(["GET"])
@authentication_classes([JWTAuthentication])
@permission_classes([IsAuthenticated])
def me(request):
    return Response({"username": request.user.username})
```

## Core concept: built-in permission classes

A permission class decides *yes or no*. The built-ins cover the common cases:

| Class | Allows the request when… |
|---|---|
| `AllowAny` | always (the default if you set none — be careful!) |
| `IsAuthenticated` | `request.user` is logged in |
| `IsAdminUser` | `request.user.is_staff` is `True` |
| `IsAuthenticatedOrReadOnly` | anyone may read (GET/HEAD/OPTIONS); only authenticated users may write |

```python
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from rest_framework.viewsets import ModelViewSet
from .models import Post
from .serializers import PostSerializer


class PostViewSet(ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    # GET /posts/        -> open to all
    # POST /posts/       -> 403 unless authenticated
```

Permissions are **combined with AND** by default — every class in the list must pass. DRF also supports boolean composition with `&`, `|`, and `~`:

```python
from rest_framework.permissions import IsAuthenticated, IsAdminUser

# "must be authenticated AND (admin OR our custom owner rule)"
permission_classes = [IsAuthenticated & (IsAdminUser | IsOwner)]
```

> Gotcha: if you set `DEFAULT_PERMISSION_CLASSES` to nothing, DRF falls back to `AllowAny` — your "protected" API is wide open. Always set an explicit default, ideally `IsAuthenticated`.

## Core concept: writing custom permissions

When the built-ins aren't enough — "only the author may edit their own post" — you subclass `BasePermission` and implement one or both methods:

- **`has_permission(self, request, view)`** — runs **before** the view body, for the request as a whole. Use it for "is this user even allowed to hit this endpoint?" (You have no specific object yet.)
- **`has_object_permission(self, request, view, obj)`** — runs **per object**, but only when you call `self.check_object_permissions(request, obj)`. Generic views call this for you inside `get_object()` (detail/update/delete), so it does **not** run on `list` or `create`.

```python
# permissions.py
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """Anyone may read; only the object's owner may modify."""

    def has_object_permission(self, request, view, obj):
        # Read-only methods (GET, HEAD, OPTIONS) are always allowed.
        if request.method in permissions.SAFE_METHODS:
            return True
        # Write methods only for the owner. `obj.author` is a User FK.
        return obj.author == request.user
```

```python
# views.py
from rest_framework import viewsets
from .permissions import IsOwnerOrReadOnly
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwnerOrReadOnly]
```

This is the direct analog of a Laravel **Policy**: `has_object_permission(request, view, obj)` ≈ `PostPolicy::update(User $user, Post $post)`.

A `has_permission` example — gate an endpoint to a role/group:

```python
class IsEditor(permissions.BasePermission):
    message = "Only members of the 'editors' group may do that."  # used in the 403 body

    def has_permission(self, request, view):
        return (
            request.user.is_authenticated
            and request.user.groups.filter(name="editors").exists()
        )
```

> Critical subtlety: `has_object_permission` is **only** triggered if your view actually calls `check_object_permissions`. The generic/`ModelViewSet` machinery does this in `get_object()`. If you write a plain `APIView` and fetch the object yourself, you **must** call `self.check_object_permissions(request, obj)` explicitly, or your object-level rule is silently skipped — a common security bug.

```python
# Plain APIView: you must invoke the object-level check yourself
from rest_framework.views import APIView
from django.shortcuts import get_object_or_404


class PostDetail(APIView):
    permission_classes = [IsOwnerOrReadOnly]

    def get_object(self, pk):
        obj = get_object_or_404(Post, pk=pk)
        self.check_object_permissions(self.request, obj)  # <-- do not forget this
        return obj
```

## Core concept: throttling (rate limiting)

Throttling caps how many requests a client may make in a window — to stop abuse, brute-forcing, and runaway scripts. DRF computes a cache key per client, counts requests in the time window, and returns **429 Too Many Requests** (with a `Retry-After` header) when the limit is exceeded. Throttle state lives in Django's **cache**, so configure a real cache (Redis/Memcached) in production; the default local-memory cache is per-process and won't work across multiple workers.

### Built-in throttle classes

- **`AnonRateThrottle`** — limits *anonymous* users, keyed by client IP.
- **`UserRateThrottle`** — limits *authenticated* users, keyed by user id (falls back to IP for anon).
- **`ScopedRateThrottle`** — applies a named rate to a specific view via a `throttle_scope` attribute.

### Global config

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "20/hour",     # 20 requests/hour for anonymous callers
        "user": "1000/day",    # 1000 requests/day for logged-in users
    },
}
```

Rates are strings of the form `number/period`, where period is one of `second`, `minute`, `hour`, `day` (only the first letter matters, so `"60/min"` and `"60/m"` both work).

### Scoped throttles — different limits per endpoint

Use this to clamp a sensitive endpoint (like login) harder than the rest of the API:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/day",
        "user": "1000/day",
        "login": "5/min",       # custom scope name
        "uploads": "10/hour",
    },
}
```

```python
# views.py
from rest_framework.throttling import ScopedRateThrottle
from rest_framework.views import APIView


class LoginView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = "login"       # ties this view to the "5/min" rate above
    # ... your login logic
```

### Custom throttle for fine-grained control

Subclass a base throttle to add your own scope, then register its rate:

```python
# throttles.py
from rest_framework.throttling import UserRateThrottle


class BurstRateThrottle(UserRateThrottle):
    scope = "burst"      # references DEFAULT_THROTTLE_RATES["burst"]


class SustainedRateThrottle(UserRateThrottle):
    scope = "sustained"
```

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_RATES": {
        "burst": "10/min",     # no more than 10/min...
        "sustained": "500/day" # ...and no more than 500/day
    },
}
```

```python
# views.py — stack both: a request must satisfy EVERY throttle
class CommentViewSet(viewsets.ModelViewSet):
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
    # ...
```

```bash
# When over the limit:
# HTTP 429 Too Many Requests
# Retry-After: 38
# {"detail": "Request was throttled. Expected available in 38 seconds."}
```

## Putting it all together — the request pipeline

```
incoming request
      │
      ▼
┌──────────────────────────┐
│ authentication_classes   │  who are you?  -> sets request.user / request.auth (401 on hard fail)
├──────────────────────────┤
│ permission_classes       │  has_permission()  -> may you hit this endpoint? (403)
├──────────────────────────┤
│ throttle_classes         │  are you over your rate? (429)
└──────────────────────────┘
      │
      ▼
your view handler (get/post/...)
      │
      ▼ (only for detail/update/delete via get_object)
  has_object_permission()  -> may you touch THIS object? (403)
```

## Common mistakes & gotchas

- **Confusing 401 and 403.** 401 = "I don't know who you are" (authentication failed). 403 = "I know who you are, but you can't do this" (permission failed).
- **Forgetting `request.user` is `AnonymousUser`, not `None`,** for unauthenticated requests. Check `request.user.is_authenticated`, never `if request.user`.
- **`has_object_permission` never running.** It only fires when the view calls `check_object_permissions` — automatic in generic/ViewSet `get_object()`, but **manual** in a plain `APIView`. It also never runs on `list`/`create` (there's no single object).
- **Using `Token` vs `Bearer` in the header.** `TokenAuthentication` expects `Authorization: Token …`; simplejwt expects `Authorization: Bearer …`. Mixing them gives a confusing 401.
- **CSRF 403s with `SessionAuthentication`.** Session auth enforces CSRF on unsafe methods. Send the CSRF token, or use a token/JWT authenticator for non-browser clients.
- **Empty `DEFAULT_PERMISSION_CLASSES` means `AllowAny`.** Forgetting to set a default leaves the whole API public. Set `IsAuthenticated` as the floor.
- **Throttling with the default cache in production.** `LocMemCache` is per-process, so each gunicorn worker counts separately and limits leak. Use Redis/Memcached.
- **Throttling is not a security boundary.** IP-based anon throttling is easily spoofed/rotated; treat it as abuse mitigation, not authentication.
- **Putting secrets in the JWT payload.** A JWT is *signed*, not *encrypted* — anyone can base64-decode and read the claims. Never store passwords or sensitive data there.
- **Long-lived access tokens.** Keep access tokens short (5–15 min) and rely on refresh tokens; otherwise a leaked access token is valid for ages and can't be easily revoked.

## Best practices

- Set sane project-wide defaults (`DEFAULT_AUTHENTICATION_CLASSES`, `DEFAULT_PERMISSION_CLASSES` = `IsAuthenticated`, `DEFAULT_THROTTLE_RATES`) and override per view only where needed.
- Prefer **JWT (simplejwt)** for SPAs/mobile/microservices; use `TokenAuthentication` only for simple internal tools; use `SessionAuthentication` for same-site browser apps and the browsable API.
- Keep `SessionAuthentication` in the default list during development so the DRF browsable API stays usable.
- Enable `ROTATE_REFRESH_TOKENS` + `BLACKLIST_AFTER_ROTATION` for a meaningful logout/revocation story.
- Express object ownership rules in `has_object_permission`, and always set `obj.<owner>` automatically in `perform_create` (e.g. `serializer.save(author=self.request.user)`) so users can't spoof ownership.
- Throttle authentication endpoints hard with a `ScopedRateThrottle` (e.g. `5/min`) to blunt brute-force attacks.
- Always run over **HTTPS** — Basic auth, tokens, and JWTs are all bearer credentials sent in headers.
- Use a **Redis** cache backend for throttling in production.

## Interview Q&A

**Q: What's the difference between authentication and authorization in DRF?**
**A:** Authentication answers "who are you?" and is handled by authentication classes, which set `request.user`/`request.auth` and fail with **401**. Authorization answers "are you allowed?" and is handled by permission classes, which fail with **403**. Authentication always runs first because permissions need the identified user to make a decision.

**Q: How does DRF decide between `TokenAuthentication` and JWT, and when would you pick each?**
**A:** `TokenAuthentication` stores an opaque token per user in the database and does a DB lookup on every request — simple but no expiry and harder to scale. JWT (`simplejwt`) is stateless: the token is signed and self-validating with a short-lived access token plus a refresh token, so no per-request DB hit and built-in expiry. Pick `TokenAuthentication` for small internal APIs; pick JWT for SPAs, mobile apps, and microservices where statelessness and short-lived credentials matter.

**Q: Walk me through the simplejwt obtain/refresh flow.**
**A:** The client POSTs username+password to `/api/token/` (`TokenObtainPairView`) and receives an **access** token (short-lived) and a **refresh** token (long-lived). It calls protected endpoints with `Authorization: Bearer <access>`. When the access token expires (401), it POSTs the refresh token to `/api/token/refresh/` to get a new access token without re-entering credentials. With rotation enabled, refreshing also issues a new refresh token and blacklists the old one.

**Q: Why two tokens instead of one?**
**A:** It limits blast radius. The access token is sent on every request, so it's the more exposed credential — keeping it short-lived (minutes) means a leak is quickly useless. The refresh token is long-lived but only travels to the refresh endpoint, so it's exposed far less often, and it can be blacklisted to revoke a session.

**Q: A JWT can be read by the client — is that a security problem?**
**A:** A JWT is signed, not encrypted, so anyone can base64-decode and read the payload. That's fine for non-secret claims like user id and expiry; the signature prevents tampering. The problem is only if you put *secrets* in the payload — never do that.

**Q: What's the difference between `has_permission` and `has_object_permission`?**
**A:** `has_permission` runs once per request before the handler, for endpoint-level checks ("can this user use this view at all?"). `has_object_permission` runs per object — but only when the view calls `check_object_permissions`, which generic views/ViewSets do inside `get_object()` for detail/update/delete. It does not run on list or create. In a plain `APIView` you must call it manually.

**Q: How do you implement "only the owner can edit, but anyone can read"?**
**A:** Subclass `BasePermission` and implement `has_object_permission`: return `True` for `request.method in SAFE_METHODS`, otherwise return `obj.owner == request.user`. Add it to `permission_classes` alongside `IsAuthenticated`. It's the DRF version of a Laravel Policy.

**Q: How is throttling configured and what status does it return?**
**A:** Set `DEFAULT_THROTTLE_CLASSES` (e.g. `AnonRateThrottle`, `UserRateThrottle`) and `DEFAULT_THROTTLE_RATES` like `{"anon": "100/day", "user": "1000/day"}`. DRF tracks counts in the cache per client (IP for anon, user id for authenticated) and returns **429 Too Many Requests** with a `Retry-After` header when exceeded. For per-endpoint limits use `ScopedRateThrottle` with a `throttle_scope`.

**Q: How do you set different rate limits for different endpoints?**
**A:** Use `ScopedRateThrottle`: define named scopes in `DEFAULT_THROTTLE_RATES` (e.g. `"login": "5/min"`), set `throttle_classes = [ScopedRateThrottle]` and `throttle_scope = "login"` on the view. Or subclass `UserRateThrottle` with a custom `scope` and stack multiple (e.g. burst + sustained) on a view.

**Q: How do you make these settings global vs. per-view?**
**A:** Globally via the `REST_FRAMEWORK` dict in settings (`DEFAULT_AUTHENTICATION_CLASSES`, `DEFAULT_PERMISSION_CLASSES`, `DEFAULT_THROTTLE_CLASSES`). Per view, set the `authentication_classes`, `permission_classes`, or `throttle_classes` attributes on the view class (or use the matching decorators on function-based views), which override the global defaults for that view only.

**Q: How does all this map to Laravel?**
**A:** Authentication classes ≈ Sanctum/Passport guards; permission classes ≈ Gates/Policies and the `can:` middleware (object-level `has_object_permission` ≈ a Policy method); throttling ≈ `throttle:` middleware / `RateLimiter`. The key difference is wiring: Laravel attaches these as route middleware, while DRF declares them as class attributes that the view runs automatically.

**Q: What happens if you forget to set `DEFAULT_PERMISSION_CLASSES`?**
**A:** DRF defaults to `AllowAny`, so every view is publicly accessible. That's a frequent and dangerous oversight — always set an explicit default, typically `IsAuthenticated`.

## Practice exercises

1. **Token auth wiring.** Configure a DRF project to use `TokenAuthentication` globally with `IsAuthenticated` as the default permission. Add the `obtain_auth_token` endpoint and use `curl` to obtain a token and call a protected view.
2. **Switch to JWT.** Install `djangorestframework-simplejwt`, wire up the obtain/refresh/verify endpoints, and set access-token lifetime to 15 minutes with refresh-token rotation + blacklisting enabled.
3. **Owner-only editing.** Write an `IsOwnerOrReadOnly` permission and apply it to a `PostViewSet` so anyone may read posts but only the author may update/delete. Make sure `perform_create` assigns the current user as author.
4. **Role gate.** Write an `IsEditor` permission (`has_permission`) that only allows users in the `editors` group, with a custom `message`, and apply it to a "publish" action.
5. **Scoped throttle on login.** Add a `ScopedRateThrottle` limiting your login endpoint to `5/min` while the rest of the API uses `anon: 100/day` and `user: 1000/day`.
6. **Stacked throttles.** Create `BurstRateThrottle` (10/min) and `SustainedRateThrottle` (500/day) and apply both to a comments endpoint.

<details>
<summary>Solutions</summary>

**1. Token auth wiring**

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "rest_framework",
    "rest_framework.authtoken",
]
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

```python
# urls.py
from rest_framework.authtoken.views import obtain_auth_token
from django.urls import path

urlpatterns = [path("api/auth-token/", obtain_auth_token)]
```

```bash
python manage.py migrate
curl -X POST http://localhost:8000/api/auth-token/ -d "username=alice&password=s3cret"
# => {"token":"<key>"}
curl http://localhost:8000/api/posts/ -H "Authorization: Token <key>"
```

**2. Switch to JWT**

```python
# settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
}
INSTALLED_APPS = [
    # ...
    "rest_framework_simplejwt.token_blacklist",
]
```

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView, TokenRefreshView, TokenVerifyView,
)
from django.urls import path

urlpatterns = [
    path("api/token/", TokenObtainPairView.as_view()),
    path("api/token/refresh/", TokenRefreshView.as_view()),
    path("api/token/verify/", TokenVerifyView.as_view()),
]
```

**3. Owner-only editing**

```python
# permissions.py
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author == request.user
```

```python
# views.py
from rest_framework import viewsets, permissions
from .permissions import IsOwnerOrReadOnly
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)  # owner can't be spoofed
```

**4. Role gate**

```python
# permissions.py
from rest_framework import permissions


class IsEditor(permissions.BasePermission):
    message = "Only members of the 'editors' group may publish."

    def has_permission(self, request, view):
        u = request.user
        return u.is_authenticated and u.groups.filter(name="editors").exists()
```

```python
# views.py
from rest_framework.decorators import action
from rest_framework.response import Response
from .permissions import IsEditor


class PostViewSet(viewsets.ModelViewSet):
    # ...
    @action(detail=True, methods=["post"], permission_classes=[IsEditor])
    def publish(self, request, pk=None):
        post = self.get_object()
        post.published = True
        post.save()
        return Response({"status": "published"})
```

**5. Scoped throttle on login**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/day",
        "user": "1000/day",
        "login": "5/min",
    },
}
```

```python
# views.py
from rest_framework.throttling import ScopedRateThrottle
from rest_framework_simplejwt.views import TokenObtainPairView


class ThrottledLoginView(TokenObtainPairView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = "login"
```

**6. Stacked throttles**

```python
# throttles.py
from rest_framework.throttling import UserRateThrottle


class BurstRateThrottle(UserRateThrottle):
    scope = "burst"


class SustainedRateThrottle(UserRateThrottle):
    scope = "sustained"
```

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_RATES": {
        "burst": "10/min",
        "sustained": "500/day",
    },
}
```

```python
# views.py
from .throttles import BurstRateThrottle, SustainedRateThrottle


class CommentViewSet(viewsets.ModelViewSet):
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
    # queryset / serializer_class ...
```

</details>

## Cheat sheet

```python
# ---- settings.py: project-wide defaults ----
from datetime import timedelta

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",  # Bearer
        "rest_framework.authentication.SessionAuthentication",        # browsable API
    ],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {"anon": "100/day", "user": "1000/day"},
}
SIMPLE_JWT = {"ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
              "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
              "ROTATE_REFRESH_TOKENS": True, "BLACKLIST_AFTER_ROTATION": True}
```

| Concept | Class / setting | Notes |
|---|---|---|
| Session auth | `SessionAuthentication` | cookie + CSRF; browsable API |
| Basic auth | `BasicAuthentication` | `Authorization: Basic …`; HTTPS only |
| Token auth | `TokenAuthentication` | `Authorization: Token …`; DB lookup |
| JWT | `JWTAuthentication` (simplejwt) | `Authorization: Bearer …`; stateless |
| Open access | `AllowAny` | the implicit default — beware |
| Logged in | `IsAuthenticated` | recommended default |
| Staff only | `IsAdminUser` | `user.is_staff` |
| Read-open/write-auth | `IsAuthenticatedOrReadOnly` | SAFE_METHODS open |
| Custom (endpoint) | `has_permission(request, view)` | before handler |
| Custom (object) | `has_object_permission(request, view, obj)` | needs `check_object_permissions` |
| Throttle anon | `AnonRateThrottle` | keyed by IP |
| Throttle user | `UserRateThrottle` | keyed by user id |
| Throttle per view | `ScopedRateThrottle` + `throttle_scope` | named rate |

**Status codes:** 401 = not authenticated · 403 = authenticated but not allowed · 429 = throttled.

**JWT endpoints:** `TokenObtainPairView` (login) · `TokenRefreshView` (refresh) · `TokenVerifyView` (verify).

**Rate format:** `"<num>/<period>"` where period ∈ `second | minute | hour | day`.

**Header keywords:** `Token` (authtoken) vs `Bearer` (JWT) vs `Basic` (basic).

**Remember:** authenticate → permit → throttle → handler → object-permission (on detail/update/delete).
