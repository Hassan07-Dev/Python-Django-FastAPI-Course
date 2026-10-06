# 45 — FastAPI: Auth (JWT & OAuth2)

> Week 4 · DRF and FastAPI

## What you will learn

- How FastAPI's built-in security utilities plug into the dependency-injection system you already learned.
- The **OAuth2 password flow** end-to-end: `OAuth2PasswordBearer` and `OAuth2PasswordRequestForm`.
- How to **hash and verify passwords** safely with `pwdlib`/`passlib` + bcrypt.
- How to **create and verify JWTs** (with `PyJWT`) including the `exp`, `sub`, and `iat` claims and a signing secret.
- Writing a **`get_current_user` dependency** that decodes the token and loads the user from your database.
- **Protecting routes** by depending on the current user, and raising `401` with the `WWW-Authenticate` header.
- **Scopes** for fine-grained authorization using `Security()` and `SecurityScopes`.
- The **refresh-token** pattern and **where tokens should live on the client** (and the pitfalls).
- How all of this compares to **DRF SimpleJWT** and **Django's session auth**.

---

## PHP/Laravel bridge

In Laravel you reach for **Sanctum** (SPA/token auth) or **Passport** (full OAuth2 server), and a `auth:api` middleware guard protects routes. JWTs are often added via `tymon/jwt-auth`. FastAPI gives you no batteries-included "Auth" facade — instead it hands you small, composable building blocks (security *dependencies*) and you wire them together. The mental mapping: Laravel's **guard middleware** → a FastAPI **dependency** (`Depends(get_current_user)`); Laravel's **`Hash::make()` / `Hash::check()`** → `pwdlib`/`passlib`; Laravel's **`Gate`/`can()` abilities** → FastAPI **scopes** via `Security()`. The big difference is explicitness: nothing is auto-registered, so you can see (and must write) every step.

---

## Core concept: the OAuth2 "password" flow at a glance

When people say "JWT auth" in a FastAPI REST API, they almost always mean the **OAuth2 Resource Owner Password Credentials flow** ("password flow"). Despite the scary name it is simple:

1. The client POSTs a **username + password** to a `/token` endpoint (as `application/x-www-form-urlencoded`, per the OAuth2 spec).
2. The server **verifies** the password against a stored **hash**, then issues a short-lived **access token** (a signed JWT).
3. For every subsequent request, the client sends `Authorization: Bearer <token>`.
4. A **dependency** on protected routes reads that header, **verifies the JWT signature + expiry**, decodes the `sub` (subject = user id/username), loads the user, and injects it into your handler.

```
client                         server
  |  POST /token (user, pass)     |
  | ----------------------------> |  verify hash, sign JWT
  |  { access_token, token_type } |
  | <---------------------------- |
  |                               |
  |  GET /users/me                |
  |  Authorization: Bearer xxx    |
  | ----------------------------> |  decode + verify JWT -> load user
  |  { ...user... }               |
  | <---------------------------- |
```

The JWT itself is **stateless**: the server does not store it. It trusts the token because it can verify the signature with its secret. That is the whole appeal — and the whole risk.

### Install what we need

```bash
pip install "fastapi[standard]" "pydantic[email]" pyjwt "pwdlib[argon2,bcrypt]" python-multipart
# python-multipart is REQUIRED for form data (OAuth2PasswordRequestForm).
# pwdlib is the modern successor to passlib. PasswordHash.recommended() defaults to
# argon2, so install the argon2 extra (bcrypt is included as a fallback/verifier).
```

> Why `python-multipart`? The OAuth2 spec sends credentials as **form fields**, not JSON. FastAPI needs `python-multipart` to parse form bodies, and it will raise a clear error at startup if it is missing.

---

## Core concept: hashing passwords (never store plaintext)

You never store a password. You store a one-way **hash** with a per-password **salt** and a deliberately slow algorithm (bcrypt/argon2) so that brute-forcing is expensive.

The modern, well-maintained library is **`pwdlib`** (passlib is still common but has been slow to support new bcrypt versions). Both APIs are tiny.

```python
# security.py
from pwdlib import PasswordHash

# Recommended preset: bcrypt (and argon2 if installed). One object, reused everywhere.
password_hash = PasswordHash.recommended()


def hash_password(plain: str) -> str:
    return password_hash.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return password_hash.verify(plain, hashed)


if __name__ == "__main__":
    h = hash_password("s3cret")
    print(h)                          # => $argon2id$v=19$... (one-way, salted)
    print(verify_password("s3cret", h))   # => True
    print(verify_password("wrong", h))    # => False
```

If you are on a codebase that uses **passlib** instead, the equivalent is:

```python
# passlib variant (older but extremely common in tutorials)
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(plain: str) -> str:
    return pwd_context.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

**Why deliberately slow?** A fast hash (MD5/SHA-256) can be brute-forced billions/sec on a GPU. bcrypt/argon2 are tuned to take ~tens of milliseconds *each*, making mass cracking impractical. `PasswordHash.recommended()` and `deprecated="auto"` also let you transparently upgrade hashes over time.

> **Laravel parallel:** `Hash::make()` ≈ `hash_password()`, `Hash::check()` ≈ `verify_password()`. Laravel defaults to bcrypt too.

---

## Core concept: creating and verifying JWTs

A JWT has three dot-separated parts: `header.payload.signature`. The **payload** is just base64url-encoded JSON **claims** — it is *not encrypted*, only *signed*. Anyone can read it; nobody can forge it without your secret.

Standard ("registered") claims you should care about:

| Claim | Meaning | Notes |
|-------|---------|-------|
| `sub` | Subject — *who* the token is about | Put the user id or username here |
| `exp` | Expiry — UNIX timestamp | Library auto-rejects expired tokens |
| `iat` | Issued-at | Useful for auditing / "logout everywhere" |
| `scopes` | (custom) permissions | We add this for fine-grained auth |

```python
# jwt_utils.py
from datetime import datetime, timedelta, timezone

import jwt  # this is PyJWT
from jwt.exceptions import InvalidTokenError

# In real life: load from env / secrets manager. Generate with: openssl rand -hex 32
SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30


def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    now = datetime.now(timezone.utc)
    expire = now + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    to_encode.update({"exp": expire, "iat": now})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def decode_access_token(token: str) -> dict:
    # Raises InvalidTokenError (incl. ExpiredSignatureError) on any problem.
    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])


if __name__ == "__main__":
    token = create_access_token({"sub": "alice"}, timedelta(seconds=2))
    print(token)                         # => eyJhbGciOi... (three base64 parts)
    print(decode_access_token(token))    # => {'sub': 'alice', 'exp': ..., 'iat': ...}

    import time
    time.sleep(3)
    try:
        decode_access_token(token)
    except InvalidTokenError as e:
        print("rejected:", type(e).__name__)   # => rejected: ExpiredSignatureError
```

Key points:

- **`HS256`** is symmetric (one shared secret signs and verifies). Simple and fine for a single service. Use **`RS256`** (asymmetric: private key signs, public key verifies) when other services must verify tokens without holding the signing secret.
- The **secret must be high-entropy and secret**. If it leaks, anyone can mint valid tokens for any user. Never commit it; load it from the environment.
- `jwt.decode` verifies the signature *and* checks `exp` automatically. Always pass `algorithms=[...]` explicitly — omitting it (or allowing `"none"`) is a classic vulnerability.

> **`python-jose` vs `PyJWT`:** Many older tutorials (including the official FastAPI docs, historically) use `python-jose`. **PyJWT** is now the more actively maintained and widely recommended choice; the API shown above is nearly identical (`jwt.encode` / `jwt.decode`). We use PyJWT here.

---

## Core concept: the `/token` endpoint with `OAuth2PasswordRequestForm`

`OAuth2PasswordRequestForm` is a dependency that pulls `username`, `password`, `scope`, `grant_type`, etc. out of the **form body** for you. The endpoint returns a JSON object with **`access_token`** and **`token_type: "bearer"`** — these exact field names are what the spec (and Swagger UI's "Authorize" button) expect.

```python
# main.py
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from pydantic import BaseModel

from security import verify_password, hash_password
from jwt_utils import create_access_token

app = FastAPI()


# --- fake "database" -------------------------------------------------------
class UserInDB(BaseModel):
    username: str
    full_name: str
    hashed_password: str
    disabled: bool = False
    scopes: list[str] = []


fake_db: dict[str, UserInDB] = {
    "alice": UserInDB(
        username="alice",
        full_name="Alice Liddell",
        hashed_password=hash_password("wonderland"),
        scopes=["items:read", "items:write"],
    ),
}


def get_user(username: str) -> UserInDB | None:
    return fake_db.get(username)


def authenticate_user(username: str, password: str) -> UserInDB | None:
    user = get_user(username)
    if not user or not verify_password(password, user.hashed_password):
        return None
    return user


# --- response schema -------------------------------------------------------
class Token(BaseModel):
    access_token: str
    token_type: str


@app.post("/token", response_model=Token)
def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    user = authenticate_user(form_data.username, form_data.password)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    # Put the username in `sub`; carry the user's granted scopes in the token.
    access_token = create_access_token(
        data={"sub": user.username, "scopes": user.scopes}
    )
    return Token(access_token=access_token, token_type="bearer")
```

Try it:

```bash
curl -X POST http://127.0.0.1:8000/token \
  -d "username=alice&password=wonderland"
# => {"access_token":"eyJhbGciOiJIUzI1NiIs...","token_type":"bearer"}
```

**Why `Depends()` with empty parentheses?** `OAuth2PasswordRequestForm` is a *class*; `Depends()` tells FastAPI to instantiate it from the request (here, the form body). You get back an object with `.username`, `.password`, `.scopes`, etc.

**Why return `401` with `WWW-Authenticate: Bearer`?** `401 Unauthorized` means "you are not authenticated." The `WWW-Authenticate` header is the HTTP standard way to tell the client *which* auth scheme to use. (`403 Forbidden` is different: "we know who you are, but you may not do this.")

---

## Core concept: `OAuth2PasswordBearer` and `get_current_user`

`OAuth2PasswordBearer(tokenUrl="token")` is a dependency that:

- Reads the `Authorization: Bearer <token>` header and returns the raw token string.
- If the header is missing, automatically returns `401` with the right `WWW-Authenticate`.
- Tells Swagger UI where the token endpoint is (so the "Authorize" button works).

Then `get_current_user` decodes that token and loads the user.

```python
# main.py (continued)
from fastapi.security import OAuth2PasswordBearer
from jwt_utils import decode_access_token
from jwt.exceptions import InvalidTokenError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> UserInDB:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = decode_access_token(token)
        username: str | None = payload.get("sub")
        if username is None:
            raise credentials_exception
    except InvalidTokenError:
        raise credentials_exception

    user = get_user(username)
    if user is None:
        raise credentials_exception
    return user


def get_current_active_user(
    current_user: Annotated[UserInDB, Depends(get_current_user)],
) -> UserInDB:
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user


# --- a protected route -----------------------------------------------------
class UserPublic(BaseModel):
    username: str
    full_name: str
    disabled: bool


@app.get("/users/me", response_model=UserPublic)
def read_me(current_user: Annotated[UserInDB, Depends(get_current_active_user)]):
    return current_user
```

Now the route is protected purely by **declaring a dependency**. No middleware, no decorator — the type signature *is* the security policy.

```bash
# 1) get a token
TOKEN=$(curl -s -X POST http://127.0.0.1:8000/token \
  -d "username=alice&password=wonderland" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# 2) call the protected route
curl http://127.0.0.1:8000/users/me -H "Authorization: Bearer $TOKEN"
# => {"username":"alice","full_name":"Alice Liddell","disabled":false}

# 3) no token => 401
curl -i http://127.0.0.1:8000/users/me
# => HTTP/1.1 401 Unauthorized
# => www-authenticate: Bearer
# => {"detail":"Not authenticated"}
```

**The dependency chain** is worth pausing on: `read_me` depends on `get_current_active_user`, which depends on `get_current_user`, which depends on `oauth2_scheme`. FastAPI resolves the whole chain per request, and the `WWW-Authenticate` / `401` behavior is inherited from the bottom of the chain. This is FastAPI's superpower — composable, testable, type-checked auth.

---

## Core concept: scopes for fine-grained authorization with `Security()`

Authentication answers *"who are you?"*. Authorization answers *"what may you do?"*. **Scopes** are named permissions (e.g. `items:read`, `items:write`, `admin`) baked into the token and checked per-route.

FastAPI models OAuth2 scopes directly:

1. Declare available scopes on the scheme so Swagger UI shows checkboxes.
2. Use **`Security(dependency, scopes=[...])`** instead of `Depends(...)` to require scopes on a route.
3. Inside `get_current_user`, receive a `SecurityScopes` object listing the scopes required *for this particular request* and verify the token carries them.

```python
# scoped_main.py — focused example of scopes
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Security, status
from fastapi.security import (
    OAuth2PasswordBearer,
    OAuth2PasswordRequestForm,
    SecurityScopes,
)
from jwt.exceptions import InvalidTokenError
from pydantic import BaseModel, ValidationError

from security import hash_password, verify_password
from jwt_utils import create_access_token, decode_access_token

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={
        "items:read": "Read items.",
        "items:write": "Create/update items.",
        "admin": "Full administrative access.",
    },
)


class UserInDB(BaseModel):
    username: str
    hashed_password: str
    scopes: list[str] = []
    disabled: bool = False


fake_db = {
    "alice": UserInDB(
        username="alice",
        hashed_password=hash_password("wonderland"),
        scopes=["items:read", "items:write"],
    ),
    "bob": UserInDB(
        username="bob",
        hashed_password=hash_password("builder"),
        scopes=["items:read"],
    ),
}


class Token(BaseModel):
    access_token: str
    token_type: str


@app.post("/token", response_model=Token)
def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    user = fake_db.get(form_data.username)
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(401, "Bad credentials", {"WWW-Authenticate": "Bearer"})
    # Grant only the scopes the client requested AND the user actually has.
    requested = form_data.scopes or user.scopes
    granted = [s for s in requested if s in user.scopes]
    token = create_access_token({"sub": user.username, "scopes": granted})
    return Token(access_token=token, token_type="bearer")


def get_current_user(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2_scheme)],
) -> UserInDB:
    # Build the right WWW-Authenticate value, including required scopes.
    if security_scopes.scopes:
        authenticate_value = f'Bearer scope="{security_scopes.scope_str}"'
    else:
        authenticate_value = "Bearer"

    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": authenticate_value},
    )
    try:
        payload = decode_access_token(token)
        username = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_scopes = payload.get("scopes", [])
    except (InvalidTokenError, ValidationError):
        raise credentials_exception

    user = fake_db.get(username)
    if user is None:
        raise credentials_exception

    # Authorization check: does the token carry every required scope?
    for scope in security_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not enough permissions",
                headers={"WWW-Authenticate": authenticate_value},
            )
    return user


# Just authentication, no scope required:
@app.get("/users/me")
def me(user: Annotated[UserInDB, Depends(get_current_user)]):
    return {"username": user.username, "scopes": user.scopes}


# Requires the "items:read" scope:
@app.get("/items")
def list_items(
    user: Annotated[UserInDB, Security(get_current_user, scopes=["items:read"])],
):
    return [{"id": 1, "owner": user.username}]


# Requires BOTH scopes:
@app.post("/items")
def create_item(
    user: Annotated[
        UserInDB, Security(get_current_user, scopes=["items:write", "items:read"])
    ],
):
    return {"created_by": user.username}
```

Behavior:

```bash
# Bob only has items:read. Request a token (scopes default to his granted scopes):
TOKEN=$(curl -s -X POST http://127.0.0.1:8000/token \
  -d "username=bob&password=builder" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

curl http://127.0.0.1:8000/items -H "Authorization: Bearer $TOKEN"
# => [{"id":1,"owner":"bob"}]            (has items:read -> OK)

curl -i -X POST http://127.0.0.1:8000/items -H "Authorization: Bearer $TOKEN"
# => HTTP/1.1 403 Forbidden
# => {"detail":"Not enough permissions"}  (missing items:write)
```

**`Depends` vs `Security`:** `Security` is a subclass of `Depends` that *also* carries the required `scopes`. FastAPI aggregates the scopes from every `Security(...)` in the chain and feeds them to `SecurityScopes`. Use `Depends` when you only need authentication; use `Security(..., scopes=[...])` when you also need authorization.

> **Laravel parallel:** scopes ≈ Passport token scopes / Gate abilities. `Security(..., scopes=["items:write"])` ≈ `$this->authorize('items:write')` or a `scope:items:write` middleware.

---

## Core concept: refresh tokens

A good access token is **short-lived** (5–30 min) so a stolen one expires quickly. But you don't want users logging in every 15 minutes. The fix: issue a long-lived **refresh token** alongside the access token. When the access token expires, the client POSTs the refresh token to `/refresh` to get a *new* access token — no password needed.

```python
# refresh.py — the essential pattern
from datetime import timedelta
from typing import Annotated

from fastapi import Body, FastAPI, HTTPException, status
from pydantic import BaseModel

from jwt_utils import create_access_token, decode_access_token
from jwt.exceptions import InvalidTokenError

app = FastAPI()

ACCESS_TTL = timedelta(minutes=15)
REFRESH_TTL = timedelta(days=7)


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


def issue_pair(username: str, scopes: list[str]) -> TokenPair:
    access = create_access_token(
        {"sub": username, "scopes": scopes, "type": "access"}, ACCESS_TTL
    )
    refresh = create_access_token(
        {"sub": username, "type": "refresh"}, REFRESH_TTL
    )
    return TokenPair(access_token=access, refresh_token=refresh)


@app.post("/refresh", response_model=TokenPair)
def refresh(refresh_token: Annotated[str, Body(embed=True)]):
    try:
        payload = decode_access_token(refresh_token)
    except InvalidTokenError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid refresh token")

    # Critical: make sure this is actually a REFRESH token, not an access token.
    if payload.get("type") != "refresh":
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Not a refresh token")

    # (Production) check the token id against a server-side allow/deny list here,
    # so you can revoke it. Then issue a fresh pair (optionally rotating refresh).
    username = payload["sub"]
    return issue_pair(username, scopes=["items:read"])
```

Why refresh tokens need server-side state to be useful:

- The whole point of refresh is **revocation** ("log out everywhere", "user changed password", "device stolen"). Pure-JWT access tokens can't be revoked before `exp`; refresh tokens *can* be, but only if you **store** them (or a `jti` token-id / version number) and check on each refresh.
- **Rotation:** on each refresh, issue a new refresh token and invalidate the old one. If an old refresh token is reused, that signals theft — revoke the whole family.
- Add a `"type"` claim (or separate secrets) so an access token can never be replayed as a refresh token and vice-versa.

---

## Core concept: where to store tokens on the client (and pitfalls)

This is the question that trips people up most, and a common interview probe.

| Storage | XSS exposure | CSRF exposure | Notes |
|---------|-------------|---------------|-------|
| `localStorage` | **High** — any injected JS can read it | None (not auto-sent) | Simple; the default in many tutorials; risky if you have *any* XSS |
| In-memory (JS variable) | Lower — lost on reload, harder to exfiltrate | None | Good for access tokens; pair with a refresh in an HttpOnly cookie |
| `HttpOnly` + `Secure` + `SameSite` cookie | **None** — JS cannot read it | **Yes** — needs CSRF mitigation | Best for refresh tokens; browser sends it automatically |

Pragmatic, widely-recommended setup for a browser SPA:

- **Access token** → keep in **memory** (a JS variable / app state). Short-lived, so a reload simply triggers a refresh.
- **Refresh token** → store in an **`HttpOnly`, `Secure`, `SameSite=Strict/Lax` cookie** scoped to `/refresh`. JS can't read it (XSS-safe), and because it's only sent to `/refresh` with `SameSite`, CSRF risk is contained.

Pitfalls to call out:

- **`localStorage` + XSS = game over.** If an attacker runs JS on your page, they read the token and impersonate the user. Defense-in-depth (CSP, sanitization) matters, but don't put long-lived secrets where JS can read them.
- **JWTs can't be invalidated before expiry** unless you keep a deny-list / token-version. Treat access tokens as short-lived and rely on refresh for revocation.
- **Don't put secrets in the payload.** It's base64, not encryption — anyone can read the claims.
- **Always HTTPS.** A bearer token in cleartext over HTTP is a free pass for anyone sniffing.
- **For mobile/native apps**, use the platform secure storage (Keychain / Keystore), not plain files.

---

## Core concept: how this compares to DRF and Django auth

| Aspect | FastAPI (this lesson) | DRF + SimpleJWT | Django (built-in) |
|--------|----------------------|-----------------|-------------------|
| Default mechanism | You assemble it from `OAuth2PasswordBearer` + PyJWT | `rest_framework_simplejwt` provides `/api/token/` + `/api/token/refresh/` | Session + cookie (server-side sessions) |
| Token type | Stateless JWT | Stateless JWT (access + refresh built-in) | Opaque session id in a cookie |
| "Protect this route" | `Depends(get_current_user)` per route | `permission_classes = [IsAuthenticated]` | `@login_required` / `LoginRequiredMixin` |
| Password hashing | `pwdlib`/`passlib` (you call it) | Django's `make_password`/`check_password` | Same (PBKDF2 by default) |
| Refresh + blacklist | You build it | `token_blacklist` app gives rotation + revocation | N/A (sessions revoked by deleting them) |
| Scopes / perms | `Security(..., scopes=[...])` | `permission_classes`, object perms, `TokenHasScope` (with django-oauth-toolkit) | Django permissions framework |
| Boilerplate | Most (but fully explicit & testable) | Little (install + add URLs) | Least (it's just Django) |

The trade-off mirrors the whole FastAPI-vs-Django story: **FastAPI gives you sharp, explicit primitives** (great for understanding and for fine-grained control); **DRF SimpleJWT gives you a turnkey JWT system**; **Django sessions** are the simplest if your client is a server-rendered app or you're fine with cookies.

```python
# DRF SimpleJWT, for contrast — this is essentially all the wiring you write:
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
}

# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
urlpatterns = [
    path("api/token/", TokenObtainPairView.as_view()),
    path("api/token/refresh/", TokenRefreshView.as_view()),
]
# Then on a view: permission_classes = [IsAuthenticated]  -> done.
```

---

## Common mistakes & gotchas

- **Forgetting `python-multipart`.** `OAuth2PasswordRequestForm` reads form data; without the package FastAPI errors at startup. Install it.
- **Sending JSON to `/token`.** The OAuth2 password flow uses `application/x-www-form-urlencoded`. Use `-d "username=...&password=..."` (curl) or a form body, not JSON.
- **Naive `datetime` for `exp`.** Use timezone-aware UTC (`datetime.now(timezone.utc)`). Naive datetimes can produce wrong expiries.
- **Not passing `algorithms=[...]` to `jwt.decode`,** or allowing `"none"`. This is a real-world JWT vulnerability — always pin the algorithm.
- **Returning `403` for "not logged in"** (it should be `401`) or `401` for "logged in but not allowed" (should be `403`). Match the semantics.
- **Omitting the `WWW-Authenticate: Bearer` header on 401.** It's part of the HTTP auth contract and what `OAuth2PasswordBearer` expects.
- **Putting the same secret/algorithm config in many files.** Centralize it; load `SECRET_KEY` from the environment, never hard-code in source you commit.
- **Treating the JWT payload as secret.** It's readable by anyone. No PII, no secrets in claims.
- **Long-lived access tokens.** A stolen token is valid until `exp`. Keep access tokens short; use refresh tokens for longevity.
- **Confusing `Depends()` and `Security()`.** Scopes only work through `Security(..., scopes=[...])` and a `SecurityScopes` parameter in the dependency.
- **Catching the wrong exception.** PyJWT raises `jwt.exceptions.InvalidTokenError` (parent of `ExpiredSignatureError`); python-jose raises `jose.JWTError`. Match your library.

---

## Best practices

- **Load `SECRET_KEY` from the environment** (e.g. `os.environ["SECRET_KEY"]`); generate with `openssl rand -hex 32`. Rotate it carefully (rotating invalidates all tokens).
- **Short access tokens (5–30 min) + refresh tokens** with server-side rotation/revocation.
- **Hash with bcrypt/argon2** via `pwdlib.PasswordHash.recommended()` (or passlib with `deprecated="auto"`), enabling transparent rehash-on-login.
- **Use `sub` for identity, `scopes` for permissions, `exp`+`iat` for lifetime,** and a `type` claim to distinguish access vs refresh tokens.
- **Centralize auth** in a `security.py` / `auth/` module: scheme, hashing, token utils, `get_current_user`. Keep route files thin.
- **Use `Security(..., scopes=[...])`** for endpoint-level authorization; keep scope names stable and namespaced (`items:read`).
- **Always serve over HTTPS;** store tokens client-side per the table above (memory for access, HttpOnly cookie for refresh).
- **Return the user via a separate response model** (`UserPublic`) so you never leak `hashed_password`.
- **Make auth testable:** because it's just dependencies, you can override them in tests with `app.dependency_overrides[get_current_user] = lambda: fake_user`.

---

## Interview Q&A

**Q: What is the OAuth2 "password flow," and how does FastAPI implement it?**
**A:** It's the Resource Owner Password Credentials grant: the client sends username + password directly to a token endpoint, the server verifies them and returns an access token, and subsequent requests carry `Authorization: Bearer <token>`. FastAPI implements it with `OAuth2PasswordRequestForm` (parses the credentials from the form body at `/token`) and `OAuth2PasswordBearer` (extracts the bearer token from the header on protected routes). It's appropriate for first-party clients where the app itself is trusted with the password.

**Q: Why hash passwords with bcrypt/argon2 instead of SHA-256?**
**A:** Fast hashes like SHA-256 can be computed billions of times per second on a GPU, so they're cheap to brute-force. bcrypt and argon2 are *deliberately slow* and salted, with a tunable work factor, making mass cracking impractical. Argon2 additionally resists GPU/ASIC attacks via memory-hardness.

**Q: Is a JWT encrypted? What can/can't you put in it?**
**A:** A standard signed JWT (JWS) is **not** encrypted — the header and payload are just base64url-encoded JSON that anyone can read. It's *tamper-proof* (signature), not *confidential*. So put identity (`sub`), expiry (`exp`), and permissions (`scopes`) in it, but never passwords, PII, or secrets. If you need confidentiality, use JWE (encrypted JWT), which is rarely necessary for typical APIs.

**Q: How does `get_current_user` work, and why is it a dependency?**
**A:** It depends on `OAuth2PasswordBearer` (which yields the raw token), decodes and verifies the JWT (signature + `exp`), reads `sub`, loads the user from the DB, and returns it — raising `401` if anything fails. Making it a dependency means any route can opt into auth just by adding `current_user: User = Depends(get_current_user)`. It's composable, type-checked, reflected in OpenAPI, and trivially overridable in tests.

**Q: When do you return 401 vs 403?**
**A:** `401 Unauthorized` means "not authenticated" — no/invalid/expired credentials; include `WWW-Authenticate: Bearer`. `403 Forbidden` means "authenticated but not permitted" — e.g. the user is valid but lacks the required scope. In FastAPI, a missing/bad token → 401; a missing scope → 403.

**Q: What's the difference between `Depends` and `Security` in FastAPI?**
**A:** `Security` is a subclass of `Depends` that additionally declares required `scopes`. FastAPI collects scopes from all `Security(...)` calls in the dependency chain and exposes them via a `SecurityScopes` parameter so your dependency can enforce them. Use `Depends` for pure authentication; use `Security(..., scopes=[...])` for scope-based authorization.

**Q: How do scopes work end-to-end?**
**A:** You declare available scopes on `OAuth2PasswordBearer(scopes={...})`. At login you embed the user's granted scopes in the token (`"scopes": [...]`). Each protected route requires scopes via `Security(get_current_user, scopes=["items:write"])`. Inside `get_current_user`, a `SecurityScopes` parameter lists the scopes required for that request; you check each is present in the token's scopes and raise `403` otherwise.

**Q: Why use refresh tokens, and how do they enable logout/revocation?**
**A:** Access tokens are kept short-lived so a stolen one expires quickly, but that would force frequent logins. A long-lived refresh token lets the client obtain new access tokens without re-entering the password. Because pure JWTs can't be invalidated before `exp`, revocation is implemented at the refresh layer: store refresh tokens (or a `jti`/version) server-side, rotate them on each use, and reject/blacklist on logout, password change, or detected reuse.

**Q: Where should the client store tokens?**
**A:** Avoid `localStorage` for sensitive tokens — it's readable by any XSS. The common best practice is to keep the **access token in memory** (short-lived, lost on reload → just refresh) and the **refresh token in an `HttpOnly`, `Secure`, `SameSite` cookie** scoped to the refresh endpoint, which is unreadable by JS and CSRF-contained. Always over HTTPS. Native apps use platform secure storage (Keychain/Keystore).

**Q: HS256 vs RS256 — when do you pick which?**
**A:** HS256 is symmetric: a single shared secret both signs and verifies. It's simplest and fine for a single service. RS256 is asymmetric: a private key signs and a public key verifies, so other services (or clients) can verify tokens without holding the signing secret. Use RS256 in distributed/microservice setups or when third parties must validate tokens.

**Q: A common JWT vulnerability is the "alg=none" / algorithm confusion attack. How do you prevent it?**
**A:** Always pass an explicit allow-list to `jwt.decode(..., algorithms=["HS256"])` and never accept `"none"`. Don't let the *token's own header* choose the verification algorithm. With RS256, also ensure an attacker can't trick the server into verifying an HS256 token using the public RSA key as an HMAC secret — pin the algorithm.

**Q: How does FastAPI auth compare to DRF SimpleJWT?**
**A:** SimpleJWT is turnkey: install it, add `TokenObtainPairView`/`TokenRefreshView` URLs, set `IsAuthenticated`, and you get access + refresh tokens (plus a blacklist app for rotation/revocation). FastAPI ships only primitives, so you assemble the `/token` endpoint, hashing, JWT utils, and `get_current_user` yourself. FastAPI is more explicit and flexible; SimpleJWT is faster to stand up and includes refresh/blacklist out of the box.

---

## Practice exercises

1. **Build the basics.** Create `security.py`, `jwt_utils.py`, and `main.py` from the lesson so that `POST /token` returns a token for `alice/wonderland` and `GET /users/me` returns her profile only with a valid `Authorization: Bearer` header. Verify with curl that a missing/expired token returns `401` with `WWW-Authenticate: Bearer`.

2. **Add registration.** Add `POST /register` that accepts `username`, `full_name`, and `password` (as a Pydantic model), rejects duplicate usernames with `409`, hashes the password, stores the user, and returns the public profile (never the hash).

3. **Add scopes.** Give users a `scopes` list, embed them in the token at login, and protect `GET /items` with `Security(get_current_user, scopes=["items:read"])` and `POST /items` with `["items:write"]`. Confirm a read-only user gets `403` on the write route.

4. **Add refresh.** Issue an access + refresh pair at login (distinguished by a `type` claim). Add `POST /refresh` that accepts a refresh token and returns a new access token, rejecting an access token submitted as a refresh token.

5. **Make it testable.** Write a `pytest` test using `fastapi.testclient.TestClient` that overrides `get_current_user` via `app.dependency_overrides` to return a fake user, and asserts `GET /users/me` returns 200 without any real token.

6. **(Stretch) Rotation + revocation.** Add an in-memory set of valid refresh-token ids (`jti`). On refresh, validate the `jti`, then rotate: invalidate the old `jti` and issue a new one. Add `POST /logout` that revokes the current refresh token.

<details>
<summary>Solutions</summary>

**Exercise 2 — registration**

```python
from typing import Annotated
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from security import hash_password

app = FastAPI()
fake_db: dict[str, dict] = {}


class RegisterIn(BaseModel):
    username: str
    full_name: str
    password: str


class UserPublic(BaseModel):
    username: str
    full_name: str


@app.post("/register", response_model=UserPublic, status_code=status.HTTP_201_CREATED)
def register(payload: RegisterIn):
    if payload.username in fake_db:
        raise HTTPException(status.HTTP_409_CONFLICT, "Username already taken")
    fake_db[payload.username] = {
        "username": payload.username,
        "full_name": payload.full_name,
        "hashed_password": hash_password(payload.password),
    }
    return fake_db[payload.username]  # UserPublic strips hashed_password
```

**Exercise 4 — refresh (core handler)**

```python
from datetime import timedelta
from typing import Annotated
from fastapi import Body, FastAPI, HTTPException, status
from jwt.exceptions import InvalidTokenError
from jwt_utils import create_access_token, decode_access_token

app = FastAPI()


def issue_pair(username: str):
    access = create_access_token(
        {"sub": username, "type": "access"}, timedelta(minutes=15)
    )
    refresh = create_access_token(
        {"sub": username, "type": "refresh"}, timedelta(days=7)
    )
    return {"access_token": access, "refresh_token": refresh, "token_type": "bearer"}


@app.post("/refresh")
def refresh(refresh_token: Annotated[str, Body(embed=True)]):
    try:
        payload = decode_access_token(refresh_token)
    except InvalidTokenError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")
    if payload.get("type") != "refresh":
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Not a refresh token")
    new_access = create_access_token(
        {"sub": payload["sub"], "type": "access"}, timedelta(minutes=15)
    )
    return {"access_token": new_access, "token_type": "bearer"}
```

**Exercise 5 — testable auth with dependency override**

```python
from fastapi.testclient import TestClient
from main import app, get_current_active_user, UserInDB

fake_user = UserInDB(
    username="tester",
    full_name="Test User",
    hashed_password="x",  # never used; we bypass auth
)

app.dependency_overrides[get_current_active_user] = lambda: fake_user
client = TestClient(app)


def test_me_without_token():
    resp = client.get("/users/me")
    assert resp.status_code == 200
    assert resp.json()["username"] == "tester"
```

**Exercise 6 — rotation + revocation (sketch)**

```python
import uuid
from datetime import timedelta
from typing import Annotated
from fastapi import Body, FastAPI, HTTPException, status
from jwt.exceptions import InvalidTokenError
from jwt_utils import create_access_token, decode_access_token

app = FastAPI()
valid_refresh_jtis: set[str] = set()  # in prod: a DB / Redis


def mint_refresh(username: str) -> str:
    jti = str(uuid.uuid4())
    valid_refresh_jtis.add(jti)
    return create_access_token(
        {"sub": username, "type": "refresh", "jti": jti}, timedelta(days=7)
    )


@app.post("/refresh")
def refresh(refresh_token: Annotated[str, Body(embed=True)]):
    try:
        payload = decode_access_token(refresh_token)
    except InvalidTokenError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")
    if payload.get("type") != "refresh" or payload.get("jti") not in valid_refresh_jtis:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Revoked or invalid")
    valid_refresh_jtis.discard(payload["jti"])          # rotate: kill the old one
    new_refresh = mint_refresh(payload["sub"])          # issue a new one
    new_access = create_access_token(
        {"sub": payload["sub"], "type": "access"}, timedelta(minutes=15)
    )
    return {"access_token": new_access, "refresh_token": new_refresh, "token_type": "bearer"}


@app.post("/logout")
def logout(refresh_token: Annotated[str, Body(embed=True)]):
    try:
        payload = decode_access_token(refresh_token)
        valid_refresh_jtis.discard(payload.get("jti"))
    except InvalidTokenError:
        pass
    return {"detail": "logged out"}
```

</details>

---

## Cheat sheet

```python
# Install
# pip install "fastapi[standard]" pyjwt "pwdlib[argon2,bcrypt]" python-multipart

# Hashing (pwdlib)
from pwdlib import PasswordHash
ph = PasswordHash.recommended()
h = ph.hash("pw"); ph.verify("pw", h)            # -> str ; -> bool

# JWT (PyJWT)
import jwt
from datetime import datetime, timedelta, timezone
exp = datetime.now(timezone.utc) + timedelta(minutes=30)
tok = jwt.encode({"sub": "alice", "exp": exp}, SECRET, algorithm="HS256")
data = jwt.decode(tok, SECRET, algorithms=["HS256"])   # verifies sig + exp

# Schemes
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm, SecurityScopes
oauth2 = OAuth2PasswordBearer(tokenUrl="token", scopes={"items:read": "..."})

# /token endpoint
form: OAuth2PasswordRequestForm = Depends()        # .username .password .scopes

# Protect a route (auth only)
user = Depends(get_current_user)

# Protect a route (auth + scope)
user = Security(get_current_user, scopes=["items:write"])

# 401 vs 403
# 401 Unauthorized  -> not authenticated; header WWW-Authenticate: Bearer
# 403 Forbidden     -> authenticated but missing scope/permission
```

| Concept | FastAPI primitive |
|---------|-------------------|
| Parse login form | `OAuth2PasswordRequestForm` |
| Extract bearer token | `OAuth2PasswordBearer` |
| Hash / verify password | `pwdlib.PasswordHash` (or passlib) |
| Encode / decode JWT | `jwt.encode` / `jwt.decode` (PyJWT) |
| Current user (authn) | `Depends(get_current_user)` |
| Scope check (authz) | `Security(get_current_user, scopes=[...])` |
| Required scopes in dep | `SecurityScopes` parameter |
| Not authenticated | `401` + `WWW-Authenticate: Bearer` |
| Authenticated, no perm | `403 Forbidden` |

**Token claim quick map:** `sub`=who · `exp`=expiry · `iat`=issued-at · `scopes`=permissions · `type`=access/refresh · `jti`=token id (for revocation).
