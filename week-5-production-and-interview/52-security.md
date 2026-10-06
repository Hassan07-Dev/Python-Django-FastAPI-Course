# 52 — Security Best Practices

> Week 5 · Production & Interview

Every backend you ship is a target. Bots scan the entire IPv4 space within minutes of you opening a port, and they probe for the same handful of mistakes over and over: a query built with string concatenation, a cookie missing `HttpOnly`, a JWT signed with a guessable secret, an endpoint that trusts the `user_id` in the URL. The good news is that the frameworks you've learned — Django, DRF, FastAPI — ship with strong, *secure-by-default* behavior. Most real-world breaches happen when a developer deliberately steps *off* the paved road: switching the ORM for raw SQL, marking user HTML as "safe", disabling CSRF "just to make the demo work". This lesson walks the OWASP Top 10 through a Python lens, shows you exactly which line of code turns a safe pattern into a vulnerability, and gives you a deploy checklist you can recite in an interview.

## What you will learn

- The **OWASP Top 10**, framed concretely for Python backends (what each category means *in your code*)
- **Injection**: how the ORM and parameterized queries protect you, and the exact raw-SQL/string-format patterns that break that protection
- **XSS**: template auto-escaping, the real danger of `mark_safe` / `|safe`, and how to render user-supplied HTML safely
- **CSRF**: what the attack is, how Django's CSRF middleware/token works, and why token-authenticated APIs are exempt
- **Auth & session security**: password hashing, the `Secure` / `HttpOnly` / `SameSite` cookie flags, brute-force and rate limiting
- **JWT pitfalls**: the `alg: none` attack, secret strength, expiry, where to store tokens, and revocation
- **Authorization mistakes**: IDOR and why you must do *object-level* permission checks
- **Secrets management**: keeping keys out of source control and out of the codebase
- **Security headers**, HTTPS/HSTS, dependency vulnerability scanning, input validation, and mass-assignment
- Framework-specific hardening for **Django, DRF, and FastAPI**, plus the production **deploy checklist**

## PHP/Laravel bridge

You've met every one of these defenses in Laravel under different names. Eloquent uses **PDO prepared statements** under the hood exactly like Django's ORM uses parameterized queries — and `DB::raw()` / `whereRaw()` with interpolated strings is the same footgun as Django's `.raw()` with f-strings. Blade's `{{ $x }}` auto-escapes and `{!! $x !!}` is the dangerous raw output, mirroring Django templates' `{{ x }}` versus `{{ x|safe }}`. Laravel's `@csrf` directive and `VerifyCsrfToken` middleware map directly to Django's `{% csrf_token %}` and `CsrfViewMiddleware`. Laravel hashes passwords with **bcrypt/argon2** via `Hash::make()`; Django uses **PBKDF2/argon2** via `make_password()` — both are slow-by-design and salted. `$fillable`/`$guarded` mass-assignment protection becomes "validate with a serializer/Pydantic model and never blindly `**request.data`". The `.env` file and `config()` helper are the same idea as `os.environ` / `django-environ` / Pydantic `BaseSettings`. The concepts transfer one-to-one; only the syntax changes.

## Core concept: the OWASP Top 10 for Python backends

OWASP publishes the canonical list of the most critical web application security risks. You don't need to memorize the version number, but you *should* be able to name the categories and say how each shows up in Python code. Here is the 2021 list translated into things you actually write:

| # | OWASP category | What it means in your Python backend |
|---|---|---|
| A01 | **Broken Access Control** | IDOR — fetching `Order.objects.get(pk=id)` without checking it belongs to the caller; missing `IsAuthenticated`; forgetting object-level checks |
| A02 | **Cryptographic Failures** | Weak/absent password hashing, secrets in code, HTTP instead of HTTPS, JWT signed with `HS256` and a short secret |
| A03 | **Injection** | SQL via string formatting in `.raw()`/`cursor.execute`, OS command injection via `os.system(user_input)`, template injection |
| A04 | **Insecure Design** | No rate limiting on login, no account lockout, trusting client-supplied prices/roles |
| A05 | **Security Misconfiguration** | `DEBUG=True` in prod, `ALLOWED_HOSTS=['*']`, default `SECRET_KEY`, permissive CORS, verbose error pages |
| A06 | **Vulnerable & Outdated Components** | Unpatched dependencies in `requirements.txt` with known CVEs |
| A07 | **Identification & Authentication Failures** | Brute-forceable login, no MFA, session fixation, predictable tokens |
| A08 | **Software & Data Integrity Failures** | Insecure deserialization — `pickle.loads()` on untrusted data, unsigned tokens |
| A09 | **Logging & Monitoring Failures** | No audit log of auth events; logging secrets/passwords |
| A10 | **Server-Side Request Forgery (SSRF)** | `requests.get(user_supplied_url)` letting an attacker reach internal services |

The rest of this lesson drills into the categories you'll be asked about most: injection, XSS, CSRF, auth/session, JWT, access control, secrets, and configuration.

## Core concept: injection (A03)

### Why the ORM keeps you safe

SQL injection happens when user input is concatenated into a query *as code* rather than passed *as data*. The fix is **parameterized queries**: you send the SQL with placeholders and the values separately, and the database driver guarantees the values can never be interpreted as SQL. Django's ORM (and SQLAlchemy) do this for you on every normal query.

```python
# SAFE — the ORM always parameterizes. `username` is bound as a value,
# never spliced into the SQL string.
from django.contrib.auth.models import User

username = request.GET["username"]          # attacker sends: ' OR '1'='1
User.objects.filter(username=username)       # => WHERE username = %s  (param: "' OR '1'='1")
# The literal string "' OR '1'='1" is searched for — no rows match. Attack neutralized.
```

The database receives `WHERE username = %s` and the value is bound separately. There is no string the attacker can send that becomes SQL.

### How raw SQL breaks it

The moment you build SQL with Python string operations, you reopen the door:

```python
from django.db import connection

# DANGEROUS — classic SQL injection via f-string
def get_user_unsafe(username: str):
    with connection.cursor() as cursor:
        # attacker sends username = "admin'; DROP TABLE users; --"
        cursor.execute(f"SELECT * FROM auth_user WHERE username = '{username}'")
        return cursor.fetchall()
# Resulting SQL: SELECT * FROM auth_user WHERE username = 'admin'; DROP TABLE users; --'

# SAFE — even with raw SQL, use parameter binding (%s placeholders + params list).
def get_user_safe(username: str):
    with connection.cursor() as cursor:
        cursor.execute("SELECT * FROM auth_user WHERE username = %s", [username])
        return cursor.fetchall()
```

The same rule applies to Django's `.raw()` and `.extra()`:

```python
# DANGEROUS
User.objects.raw(f"SELECT * FROM auth_user WHERE id = {user_id}")

# SAFE — pass params as the second argument
User.objects.raw("SELECT * FROM auth_user WHERE id = %s", [user_id])
```

And to SQLAlchemy 2.x — `text()` with bound parameters, never f-strings:

```python
from sqlalchemy import text

# DANGEROUS
session.execute(text(f"SELECT * FROM users WHERE email = '{email}'"))

# SAFE — named bind parameters
session.execute(text("SELECT * FROM users WHERE email = :email"), {"email": email})
```

> **Rule of thumb:** the placeholder is `%s` for Django's DB-API cursor and `.raw()`, but `:name` for SQLAlchemy `text()`. Never use Python's `%`, `.format()`, f-strings, or `+` to put user input into SQL. If you find yourself reaching for those, you have a bug.

### Other injection forms

Injection isn't only SQL. Watch for:

```python
import subprocess

# DANGEROUS — OS command injection
filename = request.GET["file"]
subprocess.run(f"convert {filename} out.png", shell=True)   # filename="x; rm -rf /"

# SAFE — pass args as a list, no shell
subprocess.run(["convert", filename, "out.png"], shell=False)
```

`shell=False` with a list argument means the shell never parses the string, so `;`, `|`, and `$()` are inert.

## Core concept: Cross-Site Scripting / XSS (part of A03)

XSS is injecting JavaScript into a page so it runs in another user's browser, stealing cookies or making requests as them. The defense is **output encoding**: HTML-escape any data you put into a page so `<script>` becomes `&lt;script&gt;`.

### Auto-escaping is on by default

Django templates and Jinja2 (FastAPI's usual choice) auto-escape variables:

```python
# Django template — auto-escaped. Safe even if comment contains <script>.
# templates/post.html
#   <div class="comment">{{ comment.body }}</div>
#
# If comment.body == "<script>alert(document.cookie)</script>"
# the rendered HTML is:
#   <div class="comment">&lt;script&gt;alert(document.cookie)&lt;/script&gt;</div>
# The browser shows the text; it does NOT execute the script.
```

### The danger: `mark_safe` and `|safe`

Marking content as safe disables escaping. If that content contains *any* user input, you have XSS:

```python
from django.utils.safestring import mark_safe

# DANGEROUS — never mark user-controlled content safe
bio = mark_safe(user.bio)        # user.bio = "<img src=x onerror=alert(1)>"  → executes

# Template equivalent, equally dangerous:
#   <div>{{ user.bio|safe }}</div>
```

If you genuinely must render user-supplied HTML (a rich-text editor, say), **sanitize it first** with an allow-list library like `nh3` (Rust-backed `ammonia`) or `bleach`:

```python
import nh3

dirty = '<p>Hello <script>steal()</script><b onclick="x">world</b></p>'
clean = nh3.clean(dirty)   # strips <script> and the onclick attribute
# => '<p>Hello <b>world</b></p>'
# Only then is it acceptable to render with |safe.
```

For JSON APIs (DRF/FastAPI) you usually return data, not HTML, so the *frontend* is responsible for escaping on render. But never set `Content-Type: text/html` on a response containing raw user input, and add a strong `Content-Security-Policy` header (below) as defense in depth.

## Core concept: CSRF (part of A01)

Cross-Site Request Forgery tricks a logged-in user's browser into making a state-changing request they didn't intend. Because browsers attach cookies automatically, a malicious page can `POST` to your `/transfer-money` endpoint and the victim's session cookie rides along.

### Django's CSRF protection

Django defends with the **synchronizer token pattern**: it sets a CSRF token and requires the same token to be echoed back on unsafe methods (`POST`/`PUT`/`PATCH`/`DELETE`). An attacker's page can't read your token (same-origin policy), so it can't forge the request.

```python
# settings.py — middleware is enabled by default
MIDDLEWARE = [
    # ...
    "django.middleware.csrf.CsrfViewMiddleware",
    # ...
]

# In a Django form template you MUST include the token:
#   <form method="post">
#     {% csrf_token %}      ← injects <input type="hidden" name="csrfmiddlewaretoken" ...>
#     ...
#   </form>
```

For a JavaScript SPA talking to Django, read the `csrftoken` cookie and send it back in the `X-CSRFToken` header:

```python
# Configure the header name (these are the defaults)
CSRF_COOKIE_NAME = "csrftoken"
CSRF_HEADER_NAME = "HTTP_X_CSRFTOKEN"
# Also harden the cookie itself:
CSRF_COOKIE_SECURE = True       # only sent over HTTPS
CSRF_COOKIE_HTTPONLY = False    # JS must read it to echo it back, so leave False
CSRF_COOKIE_SAMESITE = "Lax"
```

### Why pure-token APIs differ

CSRF is fundamentally a **cookie problem**: it exists *because* the browser auto-sends credentials. If your API authenticates with an `Authorization: Bearer <token>` header instead of a session cookie, there is no auto-sent credential — the attacker's page can't add that header (CORS blocks it), so CSRF doesn't apply. That's why DRF's `TokenAuthentication`/JWT views are CSRF-exempt while `SessionAuthentication` is *not*:

```python
# DRF runs CSRF checks ONLY for SessionAuthentication.
# A token-based API needs no CSRF token, but a cookie/session API does.
# The danger zone: storing a JWT in a cookie reintroduces the CSRF problem —
# then you need SameSite cookies + a CSRF token, just like sessions.
```

## Core concept: authentication & session security (A07)

### Password hashing

Never store passwords reversibly. Use a **slow, salted, adaptive hash** — Django picks PBKDF2 by default and supports Argon2 (recommended). Django's `User` model and `make_password`/`check_password` handle this for you:

```python
from django.contrib.auth.hashers import make_password, check_password

hashed = make_password("hunter2")
# => 'pbkdf2_sha256$870000$<salt>$<hash>'  — algorithm, iterations, salt, digest
check_password("hunter2", hashed)   # => True
check_password("wrong", hashed)     # => False
```

To upgrade to Argon2 (`pip install argon2-cffi`):

```python
# settings.py
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
]
```

In a framework-agnostic FastAPI app, use `passlib` with bcrypt or argon2:

```python
from passlib.context import CryptContext

pwd = CryptContext(schemes=["argon2"], deprecated="auto")
hashed = pwd.hash("hunter2")
pwd.verify("hunter2", hashed)   # => True
```

> Hashing is slow *on purpose* — it makes brute-forcing a leaked database expensive. Never use `md5`, `sha1`, or `sha256` *alone* for passwords; they're fast, which is exactly wrong here.

### Cookie & session flags

When auth relies on cookies, set the protective flags:

```python
# settings.py — session cookie hardening
SESSION_COOKIE_SECURE = True       # never sent over plain HTTP
SESSION_COOKIE_HTTPONLY = True     # JS cannot read it → mitigates XSS cookie theft
SESSION_COOKIE_SAMESITE = "Lax"    # not sent on cross-site POSTs → mitigates CSRF
```

| Flag | What it does | Stops |
|---|---|---|
| `Secure` | Cookie sent only over HTTPS | Network sniffing / MITM |
| `HttpOnly` | Cookie hidden from `document.cookie` | XSS stealing the session |
| `SameSite=Lax/Strict` | Not sent on cross-site requests | CSRF |

### Brute-force / rate limiting

Without rate limiting, attackers can try millions of passwords. Throttle authentication endpoints. In DRF, use scoped throttles:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": ["rest_framework.throttling.ScopedRateThrottle"],
    "DEFAULT_THROTTLE_RATES": {"login": "5/min", "anon": "60/min"},
}

# views.py
from rest_framework.views import APIView

class LoginView(APIView):
    throttle_scope = "login"     # 5 attempts/min per IP, then HTTP 429
    # ... authenticate ...
```

For FastAPI, `slowapi` (a Starlette wrapper around `limits`) provides the same:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/login")
@limiter.limit("5/minute")
async def login(request: Request):
    ...
```

Pair rate limiting with **account lockout** after N failures and generic error messages ("invalid credentials" — never "no such user", which leaks which usernames exist).

## Core concept: JWT pitfalls (A02 / A07)

JSON Web Tokens are great for stateless auth but are riddled with footguns interviewers love to probe.

```python
import jwt   # PyJWT
from datetime import datetime, timedelta, timezone

SECRET = "..."  # MUST be long & random — see secrets section

payload = {
    "sub": "42",
    "exp": datetime.now(timezone.utc) + timedelta(minutes=15),   # short expiry!
    "iat": datetime.now(timezone.utc),
}
token = jwt.encode(payload, SECRET, algorithm="HS256")

# CRITICAL: always pin algorithms on decode.
decoded = jwt.decode(token, SECRET, algorithms=["HS256"])
```

The pitfalls, each a real CVE-class bug:

1. **`alg: none` / algorithm confusion.** Old libraries accepted a token whose header said `{"alg":"none"}` with no signature, or let an attacker swap `RS256`→`HS256` and sign with the *public* key. Defense: **always pass `algorithms=[...]`** explicitly to `decode()` — never let the token pick.
2. **Weak secret.** `HS256` is symmetric; a short secret can be brute-forced offline from a single token. Use ≥ 32 random bytes (or use `RS256`/`EdDSA` asymmetric keys).
3. **No / long expiry.** A token with no `exp` is valid forever. Use short-lived access tokens (5–15 min) + a longer refresh token.
4. **Insecure storage.** `localStorage` is readable by any XSS; a non-`HttpOnly` cookie is too. Prefer an `HttpOnly`, `Secure`, `SameSite` cookie (and then add CSRF protection), or keep the access token in memory.
5. **No revocation.** JWTs are stateless, so you can't "log out" a token before it expires. Mitigate with short expiry + a server-side **deny-list** (revoked `jti`s in Redis) or rotating refresh tokens. `djangorestframework-simplejwt` provides token blacklisting:

```python
# settings.py with simplejwt
INSTALLED_APPS += ["rest_framework_simplejwt.token_blacklist"]
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=1),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
    "ALGORITHM": "HS256",
    "SIGNING_KEY": SECRET,   # pulled from env, not hardcoded
}
```

> Remember: a JWT payload is only **base64-encoded, not encrypted**. Anyone can read it. Never put secrets (passwords, PII you wouldn't expose) in the payload.

## Core concept: authorization mistakes & IDOR (A01)

The most common real-world breach class is **Broken Access Control**, and its poster child is **IDOR** (Insecure Direct Object Reference): you authenticate the user but forget to check that the *specific object* they requested belongs to them.

```python
# DANGEROUS — authenticated but no ownership check.
# User 42 requests /invoices/999/ which belongs to user 7. They see it.
def get_invoice(request, pk):
    invoice = Invoice.objects.get(pk=pk)      # IDOR!
    return JsonResponse({"total": invoice.total})

# SAFE — scope the query to the current user. A foreign pk → 404, not 200.
def get_invoice(request, pk):
    invoice = get_object_or_404(Invoice, pk=pk, owner=request.user)
    return JsonResponse({"total": invoice.total})
```

In DRF, do object-level checks with `has_object_permission` (and let `get_queryset` scope the data):

```python
from rest_framework import permissions, viewsets

class IsOwner(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user

class InvoiceViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticated, IsOwner]

    def get_queryset(self):
        # Belt and suspenders: never even SELECT other users' rows.
        return Invoice.objects.filter(owner=self.request.user)
```

In FastAPI, enforce it inside the dependency/handler:

```python
@app.get("/invoices/{invoice_id}")
async def read_invoice(invoice_id: int, user: User = Depends(current_user)):
    invoice = await Invoice.get(invoice_id)
    if invoice is None or invoice.owner_id != user.id:
        raise HTTPException(status_code=404)   # 404, not 403 — don't leak existence
    return invoice
```

> Return **404** rather than **403** for objects the user can't access — a 403 confirms the object exists, which itself leaks information.

## Core concept: secrets management (A02 / A05)

Never commit secrets — API keys, DB passwords, `SECRET_KEY`, JWT signing keys — to source control. Once pushed, assume they're compromised (Git history keeps them; bots scrape public repos within seconds).

```python
# settings.py — read from the environment, fail loudly if missing
import os
from pathlib import Path

SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]   # KeyError at boot if unset — good
DEBUG = os.environ.get("DJANGO_DEBUG", "0") == "1"
```

Use Pydantic v2 `BaseSettings` for typed config in FastAPI (`pip install pydantic-settings`):

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    secret_key: str
    database_url: str
    jwt_secret: str
    model_config = SettingsConfigDict(env_file=".env")   # local dev only; .env is gitignored

settings = Settings()   # raises ValidationError if any required var is missing
```

Generate strong secrets with the standard library:

```python
import secrets
secrets.token_urlsafe(32)   # => 'k3J...'  43-char URL-safe random string
# In a shell:  python -c "import secrets; print(secrets.token_urlsafe(50))"
```

Operational rules:

- Add `.env`, `*.pem`, `secrets/` to `.gitignore`; commit a `.env.example` with **blank** values.
- In production use a real secrets manager (AWS Secrets Manager, Vault, GCP Secret Manager) or platform env vars — not a `.env` file on disk.
- If a secret leaks, **rotate it**. Removing it from a later commit does not help; it's in history.

## Core concept: security headers, HTTPS & HSTS (A05)

Headers are cheap, high-leverage defense in depth. Django's `SecurityMiddleware` sets most of them from settings:

```python
# settings.py — production
SECURE_SSL_REDIRECT = True                 # 301 http → https
SECURE_HSTS_SECONDS = 31536000             # 1 year: browser refuses plain HTTP for the domain
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True         # X-Content-Type-Options: nosniff
SECURE_REFERRER_POLICY = "same-origin"
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")  # behind a TLS-terminating proxy
```

For FastAPI/Starlette, add headers via middleware (and use `django-csp`-style policies or set CSP manually):

```python
@app.middleware("http")
async def security_headers(request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    response.headers["Referrer-Policy"] = "same-origin"
    return response
```

| Header | Purpose |
|---|---|
| `Strict-Transport-Security` (HSTS) | Force HTTPS; prevents SSL-strip downgrade attacks |
| `Content-Security-Policy` | Restrict where scripts/styles load from → strong XSS mitigation |
| `X-Content-Type-Options: nosniff` | Stop the browser guessing content types |
| `Referrer-Policy` | Don't leak full URLs (with tokens) in the Referer header |

> Always terminate TLS (HTTPS). Without it, cookies and tokens travel in plaintext and any flag is moot. Set HSTS only once HTTPS works everywhere — it's sticky.

## Core concept: dependency vulnerability scanning (A06)

Most of your code is your dependencies, and they get CVEs. Scan regularly and pin versions.

```bash
# pip-audit: checks installed packages against the PyPI Advisory DB
pip install pip-audit
pip-audit                      # scans the current environment
pip-audit -r requirements.txt  # or a requirements file

# Safety is an alternative scanner
pip install safety
safety check -r requirements.txt
```

Add it to CI so a known-vulnerable package fails the build, and use Dependabot/Renovate to open update PRs automatically. Pin exact versions (or use a lockfile via `pip-tools`, `poetry`, or `uv`) so deploys are reproducible.

## Core concept: input validation & mass-assignment

### Validate at the boundary

Never trust client input. Validate type, range, format, and length *before* it touches your logic or DB. DRF serializers and Pydantic models are your validation layer:

```python
from pydantic import BaseModel, EmailStr, Field

class SignupIn(BaseModel):
    email: EmailStr
    password: str = Field(min_length=12, max_length=128)
    age: int = Field(ge=13, le=120)
# FastAPI rejects malformed bodies with a 422 before your handler runs.
```

### Mass-assignment

Mass-assignment is letting the client set fields they shouldn't — e.g. `is_admin` or `balance` — by spraying request data into a model.

```python
# DANGEROUS — attacker POSTs {"username":"x","is_staff":true}
user = User(**request.data)     # they just made themselves staff

# SAFE — only accept an explicit allow-list of fields.
ALLOWED = {"username", "email"}
data = {k: v for k, v in request.data.items() if k in ALLOWED}
user = User(**data)
```

DRF and Pydantic do this declaratively — define a serializer/model with *only* the fields the client may set, and ignore/forbid the rest:

```python
# DRF: writable fields are exactly those on the serializer.
class SignupSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["username", "email", "password"]   # is_staff can never be set here

# Pydantic v2: reject unknown fields outright
from pydantic import BaseModel, ConfigDict, EmailStr

class SignupIn(BaseModel):
    model_config = ConfigDict(extra="forbid")   # 422 if attacker adds is_admin
    username: str
    email: EmailStr
```

## Core concept: framework hardening & the deploy checklist

### Django: catch misconfigurations automatically

```bash
# Runs Django's built-in security audit. ZERO warnings is the goal before deploy.
python manage.py check --deploy
# Flags: DEBUG=True, missing SECURE_* settings, weak SECRET_KEY, insecure cookies, etc.
```

Core production settings:

```python
DEBUG = False                              # never True in prod — leaks stack traces/secrets
ALLOWED_HOSTS = ["api.example.com"]        # never ["*"] — prevents Host-header attacks
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
CSRF_TRUSTED_ORIGINS = ["https://app.example.com"]
```

### DRF: lock down defaults

```python
REST_FRAMEWORK = {
    # Default to requiring auth; opt OUT per-view, never opt in by accident.
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
    "DEFAULT_THROTTLE_RATES": {"anon": "60/min", "user": "1000/day"},
    # Disable the browsable-API renderer in production if not needed.
}
```

### FastAPI: CORS, docs, and HTTPS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],   # NOT ["*"] with credentials
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
# Consider disabling /docs & /redoc in prod, or putting them behind auth:
# app = FastAPI(docs_url=None, redoc_url=None)
```

> CORS is **not** a server-side defense — it's a browser policy. It controls which *origins your JS may read responses from*, not who can call your API (curl ignores CORS). Setting `allow_origins=["*"]` together with `allow_credentials=True` is invalid/insecure and most browsers reject it.

## Common mistakes & gotchas

- **Disabling CSRF to "fix" a 403** — never do this on session-authed views. The right fix is sending the token, not removing the guard. Don't sprinkle `@csrf_exempt` around.
- **`mark_safe`/`|safe` on user content** — instant XSS. Sanitize with `nh3`/`bleach` first, or just don't.
- **Building SQL with f-strings/`.format()`** even "because it's just an internal admin tool." Internal tools get breached too.
- **Leaving `DEBUG = True`** in production — Django's debug page prints settings (including `SECRET_KEY`) and full tracebacks.
- **`ALLOWED_HOSTS = ['*']`** — enables Host-header poisoning (password-reset link spoofing, cache poisoning).
- **`jwt.decode(token, key)` without `algorithms=[...]`** — opens the door to `alg: none` and algorithm-confusion attacks.
- **Storing JWTs in `localStorage`** — any XSS exfiltrates them. Prefer `HttpOnly` cookies (+ CSRF) or in-memory.
- **404 vs 403 leakage** and **username enumeration** ("no such user" vs "wrong password") — keep error messages generic.
- **`CORS_ALLOW_ALL_ORIGINS = True` / `allow_origins=["*"]` with credentials** — and assuming CORS protects the API at all.
- **Logging request bodies/headers** that contain passwords or tokens — scrub sensitive fields before logging.
- **`pickle.loads()` / `yaml.load()` on untrusted data** — both execute arbitrary code. Use `json` or `yaml.safe_load`.
- **Committing `.env`** then deleting it later — it's still in Git history; rotate the secret.

## Best practices

- **Secure by default, opt out deliberately**: set `IsAuthenticated` globally in DRF; require login then poke holes, not the reverse.
- **Validate every input at the boundary** with serializers/Pydantic; use `extra="forbid"` and explicit field lists to kill mass-assignment.
- **Parameterize all queries**; if you must use raw SQL, use placeholders and pass params separately.
- **Keep secrets in the environment / a secrets manager**, never in code; generate them with `secrets.token_urlsafe`.
- **Short-lived access tokens + refresh + blacklist**; pin JWT algorithms; never trust the token's `alg`.
- **Set cookie flags** (`Secure`, `HttpOnly`, `SameSite`) and **security headers** (HSTS, CSP, nosniff); enforce HTTPS.
- **Rate-limit auth endpoints** and add lockout; log auth events but never log credentials.
- **Always do object-level authorization** (`has_object_permission`, scoped `get_queryset`) — assume every ID in a URL is hostile.
- **Scan dependencies** (`pip-audit`) in CI and patch promptly; pin versions with a lockfile.
- **Run `manage.py check --deploy`** and aim for zero warnings before every release.

## Interview Q&A

**Q: How does Django's ORM prevent SQL injection, and how can a developer accidentally reintroduce it?**
**A:** The ORM uses parameterized queries — values are bound separately from the SQL text, so user input is always treated as data, never code. You reintroduce injection by building SQL with string operations: f-strings/`.format()`/`+` inside `cursor.execute()`, `.raw()`, `.extra()`, or SQLAlchemy `text()`. The fix is to keep using placeholders (`%s` for Django's cursor/`.raw()`, `:name` for SQLAlchemy) and pass the values as a separate params argument.

**Q: What's the difference between XSS and CSRF?**
**A:** XSS is injecting *script that runs in the victim's browser* — fixed by output encoding (auto-escaping) and CSP. CSRF is tricking the victim's browser into *sending a forged authenticated request* — fixed by CSRF tokens and `SameSite` cookies. XSS is an injection/output problem; CSRF is a request-authentication problem that exists because cookies are auto-sent.

**Q: Why don't token-authenticated REST APIs need CSRF protection, but session-cookie ones do?**
**A:** CSRF works because browsers automatically attach cookies to cross-site requests. A token sent in an `Authorization` header is not auto-attached — a malicious page can't add that header (CORS forbids it) — so there's nothing to forge. Session/cookie auth *does* auto-send the credential, so it needs a CSRF token. Note: if you store a JWT in a cookie, you've reintroduced the CSRF problem and must add `SameSite` + a token again.

**Q: What's wrong with `jwt.decode(token, secret)` and how do you fix it?**
**A:** It doesn't pin the algorithm, so it may accept `alg: none` (unsigned) or be vulnerable to RS256→HS256 confusion where the public key is used as the HMAC secret. Always pass `algorithms=["HS256"]` (or whatever you actually use) so the verifier — not the attacker-controlled token header — decides the algorithm.

**Q: Where should a frontend store a JWT, and what are the trade-offs?**
**A:** `localStorage` is easy but readable by any XSS, so a single script injection steals the token. An `HttpOnly`/`Secure`/`SameSite` cookie is safe from XSS reads but reintroduces CSRF (mitigated by `SameSite` + a CSRF token). In-memory storage is XSS-safer but lost on refresh. The common pattern is a short-lived access token in memory plus a refresh token in an `HttpOnly` cookie.

**Q: What is IDOR and how do you prevent it in DRF and FastAPI?**
**A:** Insecure Direct Object Reference — an authenticated user accesses another user's object by changing an ID in the URL because you never checked ownership. Prevent it by scoping queries to the current user (`get_queryset` filtering by `owner=request.user`) and adding object-level checks (`has_object_permission` in DRF; an explicit `owner_id == user.id` check in FastAPI). Return 404, not 403, so you don't leak that the object exists.

**Q: Why is a fast hash like SHA-256 a bad choice for passwords?**
**A:** Speed is the enemy here. A leaked database of SHA-256 hashes can be brute-forced billions of guesses per second on a GPU. Password hashes must be *slow and adaptive* (PBKDF2, bcrypt, Argon2) and *salted* so identical passwords hash differently and rainbow tables don't work. Django's `make_password` does this by default; prefer Argon2.

**Q: What do the `Secure`, `HttpOnly`, and `SameSite` cookie flags each protect against?**
**A:** `Secure` keeps the cookie off plaintext HTTP (stops network sniffing). `HttpOnly` hides it from `document.cookie` (stops XSS stealing the session). `SameSite=Lax/Strict` stops the cookie being sent on cross-site requests (stops CSRF). Together they harden cookie-based sessions on three independent axes.

**Q: How do you keep secrets out of source control, and what do you do if one leaks?**
**A:** Read them from environment variables or a secrets manager (Vault/AWS Secrets Manager), gitignore `.env`/key files, and commit only a `.env.example` with blank values. If one leaks, *rotate it immediately* — deleting it from a later commit doesn't help because it's preserved in Git history and may already be scraped.

**Q: What is mass-assignment and how do Python frameworks prevent it?**
**A:** Mass-assignment is letting a client set fields they shouldn't (like `is_admin`) by dumping request data into a model (`Model(**request.data)`). Prevent it with an explicit allow-list: a DRF serializer or Pydantic model that lists *only* the client-writable fields, plus `extra="forbid"` in Pydantic to reject unknown keys. It's the analog of Laravel's `$fillable`.

**Q: What does `python manage.py check --deploy` do?**
**A:** It runs Django's deployment security checklist and warns about misconfigurations: `DEBUG=True`, missing/weak `SECRET_KEY`, missing `SECURE_SSL_REDIRECT`/HSTS, insecure session/CSRF cookies, permissive `ALLOWED_HOSTS`, and more. The goal is zero warnings before shipping.

**Q: How do you handle dependency vulnerabilities in a Python project?**
**A:** Scan with `pip-audit` (or `safety`) against the advisory database, run it in CI so vulnerable packages fail the build, pin versions with a lockfile (`pip-tools`/`poetry`/`uv`) for reproducible deploys, and use Dependabot/Renovate to automate update PRs. Patch promptly, especially for high-severity CVEs.

## Practice exercises

1. **Find the injection.** You're handed `Product.objects.raw("SELECT * FROM shop_product WHERE name LIKE '%" + term + "%'")`. Explain the vulnerability and rewrite it safely.

2. **Fix the XSS.** A template renders `<div>{{ comment.html|safe }}</div>` where `comment.html` is user-submitted rich text. Describe the risk and write code that lets you keep `|safe` without being exploitable.

3. **Harden cookies & headers.** Write the Django `settings.py` block for production that enforces HTTPS, HSTS for one year (with subdomains), secure session and CSRF cookies, `SameSite=Lax`, and `nosniff`.

4. **Close the IDOR.** Given this DRF viewset, make it impossible for one user to read or modify another user's notes:
   ```python
   class NoteViewSet(viewsets.ModelViewSet):
       queryset = Note.objects.all()
       serializer_class = NoteSerializer
       permission_classes = [permissions.IsAuthenticated]
   ```

5. **Safe JWT decode.** Write a function `verify(token: str) -> dict` using PyJWT that validates an `HS256` token, pins the algorithm, requires `exp`, and raises a clear error on an invalid/expired token.

6. **Kill mass-assignment.** A FastAPI signup handler does `user = User(**payload)` where `payload` is the raw request body. Rewrite it with a Pydantic v2 model that only allows `email` and `password`, enforces a 12-char minimum password, and rejects any extra fields.

<details><summary>Solutions</summary>

**1. Injection**

The `term` is concatenated straight into SQL, so input like `%' OR '1'='1` (or worse, a stacked `; DROP ...`) is executed as SQL. Use a placeholder and pass the wildcards in the parameter value:

```python
pattern = f"%{term}%"
Product.objects.raw("SELECT * FROM shop_product WHERE name LIKE %s", [pattern])
# Better: avoid raw SQL entirely
Product.objects.filter(name__icontains=term)
```

**2. XSS**

`|safe` disables escaping, so any `<script>`/`onerror` in the user's HTML runs. Sanitize server-side with an allow-list before storing/rendering:

```python
import nh3
comment.html = nh3.clean(submitted_html)   # strips <script>, event handlers, etc.
comment.save()
# Now <div>{{ comment.html|safe }}</div> renders only allow-listed tags.
```

**3. Cookies & headers**

```python
DEBUG = False
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = "Lax"
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_SAMESITE = "Lax"
```

**4. IDOR**

```python
class IsOwner(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user

class NoteViewSet(viewsets.ModelViewSet):
    serializer_class = NoteSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwner]

    def get_queryset(self):
        return Note.objects.filter(owner=self.request.user)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)   # don't trust an owner field from the client
```

**5. Safe JWT decode**

```python
import jwt

def verify(token: str) -> dict:
    try:
        return jwt.decode(
            token,
            SECRET,
            algorithms=["HS256"],            # pin the algorithm
            options={"require": ["exp"]},    # reject tokens with no expiry
        )
    except jwt.ExpiredSignatureError:
        raise ValueError("token expired")
    except jwt.InvalidTokenError:
        raise ValueError("invalid token")
```

**6. Mass-assignment**

```python
from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict, EmailStr, Field

app = FastAPI()

class SignupIn(BaseModel):
    model_config = ConfigDict(extra="forbid")   # 422 on unknown fields like is_admin
    email: EmailStr
    password: str = Field(min_length=12)

@app.post("/signup")
async def signup(data: SignupIn):
    user = User(email=data.email, password_hash=hash_password(data.password))
    # only email & password can ever be set from the request
    ...
```

</details>

## Cheat sheet

```text
INJECTION
  ✓ ORM filters / parameterized: filter(x=val), execute("... %s", [val]), text(":n", {"n":v})
  ✗ f-strings/.format/+ in execute(), .raw(), text();  subprocess(shell=True, f"...")
XSS
  ✓ auto-escaping (default); sanitize user HTML with nh3/bleach; CSP header
  ✗ mark_safe(user_input), {{ x|safe }} on user content
CSRF
  ✓ {% csrf_token %}, X-CSRFToken header, SameSite cookies
  ✗ @csrf_exempt; token APIs (Bearer header) don't need it
PASSWORDS   make_password / Argon2 / passlib  (slow+salted)   ✗ md5/sha alone
COOKIES     Secure (HTTPS) · HttpOnly (no JS) · SameSite=Lax (no CSRF)
JWT         pin algorithms=[...] · short exp · strong secret · HttpOnly cookie · blacklist
            payload is base64, NOT encrypted — no secrets inside
AUTHZ/IDOR  scope get_queryset to user · has_object_permission · 404 not 403
SECRETS     os.environ / BaseSettings · gitignore .env · rotate on leak · secrets.token_urlsafe(32)
HEADERS     HSTS · CSP · X-Content-Type-Options:nosniff · Referrer-Policy · HTTPS everywhere
VALIDATION  serializers / Pydantic at boundary · extra="forbid" · explicit field allow-list
DEPS        pip-audit / safety in CI · pin versions · Dependabot
CONFIG      DEBUG=False · ALLOWED_HOSTS=[host] · manage.py check --deploy (zero warnings)
            DRF default IsAuthenticated · FastAPI CORS allow_origins=[exact], never * + credentials
```

| Threat | Primary defense in Python | One-line fix |
|---|---|---|
| SQL injection | Parameterized queries | Use ORM / `%s` placeholders |
| XSS | Output escaping + CSP | Don't `mark_safe` user input; sanitize |
| CSRF | Token + SameSite | `{% csrf_token %}`, `SameSite=Lax` |
| Brute force | Rate limiting | Throttle login, lockout |
| Weak auth | Slow hashing | Argon2 / `make_password` |
| JWT abuse | Pin alg + expiry | `algorithms=[...]`, short `exp` |
| IDOR | Object-level authz | Scope `get_queryset`, `has_object_permission` |
| Leaked secrets | Env / manager | `os.environ`, gitignore, rotate |
| MITM | HTTPS + HSTS | `SECURE_HSTS_SECONDS`, `Secure` cookies |
| Vuln deps | Scanning | `pip-audit` in CI |
| Mass-assignment | Field allow-list | Serializer/Pydantic + `extra="forbid"` |
| Misconfig | Audit | `manage.py check --deploy` |
