# 23 — Web Foundations: HTTP, REST, WSGI/ASGI

> Week 3 · Web fundamentals for backend developers

## What you will learn

- The HTTP request/response cycle, end to end, and what travels over the wire
- HTTP methods (GET, POST, PUT, PATCH, DELETE) and the concepts of **safety** and **idempotency**
- Status code families and the specific codes you must know cold (200, 201, 204, 301, 302, 400, 401, 403, 404, 409, 422, 429, 500)
- The anatomy of a request and a response: method, path, headers, body
- Where data lives: **query string** vs **path params** vs **request body**
- Content types and why JSON dominates APIs
- Cookies, sessions, and what "stateless" really means
- REST as an architectural style: resources, representations, the uniform interface, statelessness — and what actually makes an API "RESTful"
- CORS at a conceptual level (why your browser blocks that fetch)
- **WSGI vs ASGI**, and why ASGI is the reason FastAPI can be async (this sets up Django-sync vs FastAPI-async in later lessons)

---

## PHP/Laravel bridge

You already know all of this implicitly — you just expressed it through Laravel's vocabulary. A Laravel `Route::get()` / `Route::post()` maps directly onto HTTP methods. `$request->query('q')`, `$request->route('id')`, and `$request->input('name')` are exactly the query-string / path-param / body distinction we'll formalize here. Laravel's `response()->json($data, 201)` is you choosing a status code and content type by hand. The big mental shift is the plumbing underneath: PHP traditionally ran one request per process via **PHP-FPM**, where each request gets a fresh, isolated world and dies at the end (naturally "stateless" and "share-nothing"). Python instead standardizes the web-server-to-app contract as **WSGI** (the synchronous one, like PHP-FPM in spirit) and **ASGI** (the async one that has no real PHP-FPM equivalent and is what makes `async def` endpoints possible). Keep that PHP-FPM-per-request picture in mind — it explains why "statelessness" feels natural in PHP but is something you actively design for in long-lived Python processes.

---

## Core concept: The HTTP request/response cycle

HTTP is a **request/response** protocol over TCP. A client (browser, mobile app, `curl`, another service) opens a connection, sends a **request**, and the server sends back exactly one **response**. Then, conceptually, the exchange is done. HTTP/1.1 keeps the TCP connection alive for reuse (keep-alive), and HTTP/2/3 multiplex many requests over one connection, but the unit of meaning is always one request → one response.

The cycle, step by step:

1. **Client builds a request**: a method, a URL (host + path + query string), headers, and an optional body.
2. **DNS + TCP (+ TLS for HTTPS)**: the hostname resolves to an IP, a connection opens, and for HTTPS a TLS handshake encrypts everything.
3. **Server receives & routes** the request to the right handler based on method + path.
4. **Server runs your code** (queries a DB, etc.) and builds a response: a status code, headers, and an optional body.
5. **Response travels back**; the client reads the status, then headers, then body.

Here's the literal text a minimal HTTP/1.1 request looks like on the wire:

```
GET /api/users/42?include=orders HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOi...

```

(The blank line marks the end of the headers; a GET usually has no body after it.)

And the response:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 51
Cache-Control: no-store

{"id": 42, "name": "Ada Lovelace", "active": true}
```

You can watch this yourself with `curl -v` (verbose). Lines starting with `>` are what your client sent; lines with `<` are what came back:

```bash
curl -v https://httpbin.org/get
# > GET /get HTTP/2
# > Host: httpbin.org
# > accept: */*
# <
# < HTTP/2 200
# < content-type: application/json
# < ...
# (then the JSON body)
```

And here is the same exchange driven from Python with the modern `httpx` client (the async-capable successor to `requests`):

```python
import httpx

resp = httpx.get("https://httpbin.org/get", params={"q": "python"})

print(resp.status_code)        # => 200
print(resp.request.method)     # => GET
print(resp.request.url)        # => https://httpbin.org/get?q=python
print(resp.headers["content-type"])  # => application/json
data = resp.json()             # parse the JSON body into a dict
print(data["args"])            # => {'q': 'python'}
```

---

## Core concept: HTTP methods, safety, and idempotency

The **method** (also called the verb) declares your *intent*. The server isn't forced to honor the semantics, but well-designed APIs do — and clients, proxies, and caches *rely* on these semantics.

| Method | Typical use | Has body? | Safe? | Idempotent? |
|--------|-------------|-----------|-------|-------------|
| GET | Read a resource / collection | No | Yes | Yes |
| POST | Create a resource; non-idempotent actions | Yes | No | No |
| PUT | Replace a resource entirely | Yes | No | Yes |
| PATCH | Partially update a resource | Yes | No | No* |
| DELETE | Remove a resource | Usually no | No | Yes |

Two words you *will* be asked about in an interview:

- **Safe** = the method is read-only and has no side effects on the server. `GET` is safe; you can call it freely and prefetch it. Anything that mutates state is *not* safe.
- **Idempotent** = making the same request N times has the same end-state as making it once. `GET`, `PUT`, and `DELETE` are idempotent. `PUT /users/42 {name: "Ada"}` ten times leaves one user named Ada — same as doing it once. `DELETE /users/42` twice: the first deletes, the second is a no-op (often returns 404), but the *server state* is identical. `POST /users` ten times creates *ten* users — not idempotent.

> \*PATCH is *not guaranteed* idempotent by the spec, because a patch could be relative (e.g. "increment balance by 10"). If your PATCH simply sets fields to absolute values, it happens to be idempotent — but don't rely on PATCH being idempotent in general.

**Why this matters in practice:** safety and idempotency are what let a load balancer or client *safely retry* a request after a network blip. A failed `GET` or `PUT` can be retried automatically; retrying a `POST` risks creating a duplicate. This is exactly why payment APIs ask you to send an **idempotency key** with a POST — it lets the server dedupe retries.

```python
import httpx

# GET — safe & idempotent, read-only
httpx.get("https://httpbin.org/get")

# POST — create; NOT idempotent (two calls = two resources)
httpx.post("https://httpbin.org/post", json={"name": "Ada"})

# PUT — full replace; idempotent
httpx.put("https://httpbin.org/put", json={"id": 42, "name": "Ada", "active": True})

# PATCH — partial update; send only changed fields
httpx.patch("https://httpbin.org/patch", json={"active": False})

# DELETE — remove; idempotent
httpx.delete("https://httpbin.org/delete")
```

---

## Core concept: Status codes

The first line of every response carries a 3-digit status code. The **first digit** tells you the family:

| Family | Meaning | Mnemonic |
|--------|---------|----------|
| **1xx** | Informational | "hold on" |
| **2xx** | Success | "here you go" |
| **3xx** | Redirection | "look elsewhere" |
| **4xx** | Client error | "you messed up" |
| **5xx** | Server error | "I messed up" |

The specific codes you must know:

| Code | Name | When to use it |
|------|------|----------------|
| **200** | OK | Successful GET/PUT/PATCH that returns a body |
| **201** | Created | A POST created a new resource; return it + a `Location` header |
| **204** | No Content | Success, but no body to send (common for DELETE) |
| **301** | Moved Permanently | Resource has a new permanent URL; caches/clients update |
| **302** | Found | Temporary redirect; don't cache the new location |
| **400** | Bad Request | Malformed syntax — the server can't even parse it |
| **401** | Unauthorized | *Not authenticated* — who are you? (misnamed; it's about auth*n*) |
| **403** | Forbidden | Authenticated but *not allowed* — I know who you are, no |
| **404** | Not Found | No resource at this URL |
| **409** | Conflict | Request conflicts with current state (e.g. duplicate email, version clash) |
| **422** | Unprocessable Content | Syntactically valid but *semantically* wrong — validation failed (named "Unprocessable Entity" in the older RFC 7231; renamed "Unprocessable Content" in RFC 9110) |
| **429** | Too Many Requests | Rate limit hit; often paired with a `Retry-After` header |
| **500** | Internal Server Error | An unhandled exception in your code |

The distinctions that trip people up:

- **401 vs 403**: 401 means "I don't know who you are" (no/invalid credentials). 403 means "I know exactly who you are, and you still can't do this." Counterintuitively, 401 is named "Unauthorized" but is about *authentication*.
- **400 vs 422**: 400 = the request is malformed at the protocol/syntax level (broken JSON, missing required structure). 422 = the JSON parsed fine but the *values* failed business/validation rules (email already taken, age is negative). **FastAPI/Pydantic** return **422** for request-validation failures by default. **DRF**, by contrast, returns **400 Bad Request** for serializer/`ValidationError` failures by default (it does not use 422 out of the box) — a real difference worth knowing. (422 is "Unprocessable Content" in the current RFC 9110, formerly "Unprocessable Entity" in RFC 7231 — you'll still hear both names.) Plain Django, which is form-oriented rather than API-first, likewise uses 400 for form errors.
- **201 vs 200**: prefer 201 specifically for "a new thing now exists," and include a `Location` header pointing to it.
- **3xx**: 301 is permanent (browsers and caches remember it; SEO-relevant), 302 is temporary.

```python
import httpx

# A few real status codes via httpbin's /status/<code> endpoint:
print(httpx.get("https://httpbin.org/status/204").status_code)  # => 204
print(httpx.get("https://httpbin.org/status/404").status_code)  # => 404
print(httpx.get("https://httpbin.org/status/500").status_code)  # => 500

r = httpx.get("https://httpbin.org/status/200")
print(r.is_success)      # => True   (2xx)
print(r.is_redirect)     # => False
print(r.is_client_error) # => False  (would be True for 4xx)
print(r.is_server_error) # => False  (would be True for 5xx)

# raise_for_status() turns 4xx/5xx into an exception — great for clients:
try:
    httpx.get("https://httpbin.org/status/503").raise_for_status()
except httpx.HTTPStatusError as e:
    print(e.response.status_code)  # => 503
```

---

## Core concept: Request & response anatomy

Every HTTP message has the same three-part shape: a **start line**, **headers**, and an optional **body**.

**Request:**

- **Start line** = method + path (+ query string) + protocol version: `POST /api/users HTTP/1.1`
- **Headers** = key/value metadata: `Content-Type`, `Authorization`, `Accept`, `User-Agent`, `Cookie`, ...
- **Body** = the payload, for methods that carry one (the JSON you're POSTing)

**Response:**

- **Start line** = protocol + status code + reason phrase: `HTTP/1.1 201 Created`
- **Headers** = `Content-Type`, `Content-Length`, `Location`, `Set-Cookie`, `Cache-Control`, ...
- **Body** = the representation of the resource (your JSON)

A few headers you'll meet constantly:

| Header | Direction | Purpose |
|--------|-----------|---------|
| `Content-Type` | both | What format the body is (`application/json`, `text/html`, ...) |
| `Accept` | request | What formats the client *wants* back (content negotiation) |
| `Authorization` | request | Credentials, e.g. `Bearer <token>` |
| `Content-Length` | both | Size of the body in bytes |
| `Location` | response | Where the created/redirected resource lives |
| `Cache-Control` | both | Caching directives |
| `Set-Cookie` / `Cookie` | response / request | The cookie mechanism |

```python
import httpx

resp = httpx.post(
    "https://httpbin.org/post",
    json={"name": "Ada", "role": "engineer"},  # sets Content-Type: application/json
    headers={"Authorization": "Bearer secret-token", "Accept": "application/json"},
)

# Inspect what we SENT:
print(resp.request.headers["content-type"])  # => application/json
print(resp.request.headers["authorization"]) # => Bearer secret-token

# httpbin echoes the request back to us in its response body:
echoed = resp.json()
print(echoed["json"])     # => {'name': 'Ada', 'role': 'engineer'}
print(echoed["headers"]["Authorization"])  # => Bearer secret-token
```

---

## Core concept: Where data lives — path params vs query string vs body

This is one of the most practical distinctions in API design, and it maps cleanly onto Laravel.

| Location | Looks like | Use it for | Laravel analogue |
|----------|-----------|------------|------------------|
| **Path parameter** | `/users/42` | Identifying *which* resource | `Route::get('/users/{id}')` / `$request->route('id')` |
| **Query string** | `/users?role=admin&page=2` | Filtering, sorting, pagination, optional modifiers on a GET | `$request->query('role')` |
| **Request body** | JSON payload | The data you're creating/updating (POST/PUT/PATCH) | `$request->input('name')` |

Rules of thumb:

- The **path** identifies the resource and is hierarchical: `/projects/7/tasks/3`. If removing the value would change *which* thing you're addressing, it's a path param.
- The **query string** modifies *how* you view a collection — filtering, searching, sorting, paginating. It's optional and order-independent: `?q=python&sort=-created&page=2`.
- The **body** carries the representation you're sending to the server. GET and DELETE typically have no body; POST/PUT/PATCH do.

```python
import httpx

# Path param identifies the resource; query string filters/paginates.
resp = httpx.get(
    "https://httpbin.org/anything/users/42",   # 42 is a path param
    params={"include": "orders", "page": 2},    # ?include=orders&page=2
)
sent = resp.json()
print(sent["url"])   # => https://httpbin.org/anything/users/42?include=orders&page=2
print(sent["args"])  # => {'include': 'orders', 'page': '2'}  (query string)

# Body carries the new data on a POST:
resp = httpx.post("https://httpbin.org/anything/users", json={"name": "Grace"})
print(resp.json()["json"])  # => {'name': 'Grace'}
```

A note on query-string values: they are **always strings** on the wire. `?page=2` arrives as the string `"2"`, and `?active=true` as `"true"`. Frameworks (DRF serializers, FastAPI/Pydantic) coerce these to `int`/`bool` for you — but the raw value is text.

---

## Core concept: Content types and JSON

The `Content-Type` header tells the receiver how to interpret the bytes in the body. The ones you'll see most:

- `application/json` — the lingua franca of REST APIs.
- `application/x-www-form-urlencoded` — classic HTML form posts (`name=Ada&role=eng`).
- `multipart/form-data` — file uploads and forms with binary parts.
- `text/html` — web pages.

JSON maps neatly onto Python types, which is a big reason it's the default for APIs:

| JSON | Python |
|------|--------|
| object `{}` | `dict` |
| array `[]` | `list` |
| string | `str` |
| number | `int` / `float` |
| `true` / `false` | `True` / `False` |
| `null` | `None` |

The standard library handles JSON via the `json` module:

```python
import json

# Python dict -> JSON string (serialize / "dump")
payload = {"id": 42, "name": "Ada", "active": True, "tags": ["a", "b"], "manager": None}
text = json.dumps(payload)
print(text)
# => {"id": 42, "name": "Ada", "active": true, "tags": ["a", "b"], "manager": null}

# JSON string -> Python dict (deserialize / "load")
back = json.loads(text)
print(back["active"], type(back["active"]))  # => True <class 'bool'>
print(back["manager"])                        # => None

# Note the naming: dumpS / loadS work on strings;
# dump / load (no S) work on file objects.
```

In real API code you rarely call `json` directly — DRF serializers and Pydantic models do the (de)serialization plus validation. But knowing the underlying `json` module and that **`true`/`false`/`null` become `True`/`False`/`None`** is interview-grade knowledge.

---

## Core concept: Cookies, sessions, and statelessness

HTTP is **stateless**: each request is independent and carries no memory of previous ones. The server does not "remember" you between requests. That's a feature — it's what lets you scale horizontally by throwing more identical servers behind a load balancer.

But applications *need* continuity ("stay logged in"). The classic solution is **cookies**:

1. On login, the server sends `Set-Cookie: sessionid=abc123; HttpOnly; Secure; SameSite=Lax`.
2. The browser stores it and **automatically attaches** `Cookie: sessionid=abc123` on every subsequent request to that site.
3. The server looks up `abc123` in a session store (DB, Redis, signed cookie) to recover *who you are*.

So statelessness isn't violated — the *state* (the session) lives in a store, and the cookie is just the key. This is **server-side session** auth, and it's exactly how Laravel's session/auth and Django's `SessionMiddleware` work.

The two broad auth styles:

| Style | Where state lives | Sent how | Typical use |
|-------|-------------------|----------|-------------|
| **Session + cookie** | Server (DB/Redis) | Cookie, automatic | Server-rendered apps, same-site |
| **Token (e.g. JWT)** | In the token itself (self-contained) | `Authorization: Bearer ...`, manual | SPAs, mobile, third-party APIs |

Token auth is more common for *REST APIs* precisely because it keeps the server stateless in the strictest sense — nothing to look up; the token carries its own claims. We'll dig into both in the auth lesson.

Cookie flags worth knowing (they come up in security questions): `HttpOnly` (JS can't read it — mitigates XSS token theft), `Secure` (HTTPS only), `SameSite` (controls cross-site sending — mitigates CSRF).

```python
import httpx

# A Client persists cookies across requests, like a browser does.
with httpx.Client() as client:
    # Server sets a cookie:
    client.get("https://httpbin.org/cookies/set/sessionid/abc123")
    # Subsequent request automatically sends it back:
    resp = client.get("https://httpbin.org/cookies")
    print(resp.json())  # => {'cookies': {'sessionid': 'abc123'}}
```

---

## Core concept: REST — what actually makes an API RESTful

REST (REpresentational State Transfer) is an **architectural style**, not a protocol or library. An API is "RESTful" to the degree it honors these constraints:

1. **Resources** are the nouns, addressable by URL. `/users`, `/users/42`, `/users/42/orders`. Use **nouns, not verbs** in URLs — the HTTP method is the verb. So it's `DELETE /users/42`, never `POST /deleteUser?id=42`.
2. **Representations**: a resource (the conceptual user #42) is separate from its *representation* (the JSON, or XML, or HTML you get back). Content negotiation via `Accept`/`Content-Type` lets the same resource be represented differently.
3. **Uniform interface**: a small, consistent set of operations — the HTTP methods — applied uniformly across all resources. Once you know how to GET/POST/PUT/DELETE one resource, you know them all.
4. **Statelessness**: each request contains everything the server needs; the server keeps no per-client session in memory between requests (auth travels on every request).
5. **Client–server separation** and **cacheability** (responses declare whether they can be cached via `Cache-Control`/`ETag`), plus **layered system** (clients can't tell if they're talking to the real server or a proxy/CDN).
6. **HATEOAS** (Hypermedia as the Engine of Application State): responses include links to related actions/resources. This is the most-skipped constraint in practice; most "REST" APIs are pragmatically resource+verb+JSON and stop short of full hypermedia.

A clean, RESTful resource design for users:

| Method + path | Meaning | Success code |
|---------------|---------|--------------|
| `GET /users` | List users (with `?` filters/pagination) | 200 |
| `POST /users` | Create a user | 201 (+ `Location`) |
| `GET /users/42` | Fetch one user | 200 (or 404) |
| `PUT /users/42` | Replace user 42 | 200 / 204 |
| `PATCH /users/42` | Partially update user 42 | 200 |
| `DELETE /users/42` | Delete user 42 | 204 |

Anti-patterns that *fail* the "RESTful" smell test: verbs in URLs (`/getUser`, `/users/create`), using `GET` to mutate data, returning `200 OK` with `{"error": ...}` inside instead of a real error status, and ignoring HTTP methods entirely (tunneling everything through `POST`).

---

## Core concept: CORS in one section

CORS (Cross-Origin Resource Sharing) is a **browser** security mechanism, not a server-side firewall. By default the **Same-Origin Policy** blocks JavaScript on `https://app.example.com` from reading a response from a *different* origin like `https://api.other.com`. An "origin" = scheme + host + port; change any one and it's cross-origin.

The flow:

- For "simple" requests the browser sends the request with an `Origin: https://app.example.com` header. The server must reply with `Access-Control-Allow-Origin: https://app.example.com` (or `*`) or the browser **hides the response from your JS** (the request may still have hit the server!).
- For "non-simple" requests (e.g. `PUT`/`DELETE`, custom headers, JSON content-type in some cases) the browser first sends a **preflight** `OPTIONS` request asking "am I allowed to do this?" The server answers with `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, etc. Only if the preflight passes does the real request go out.

Key points for interviews:

- CORS is enforced **by the browser**, for browser-initiated JS requests. `curl`, Postman, mobile apps, and server-to-server calls are *not* subject to CORS — that's why your API works in Postman but the front-end gets a CORS error.
- It is **not** authentication or authorization; it only governs which web origins may *read* responses in a browser.
- You fix it on the **server** by sending the right `Access-Control-Allow-*` headers (in Django: `django-cors-headers`; in FastAPI: `CORSMiddleware`).

---

## Core concept: WSGI vs ASGI (and why ASGI enables async)

This is the plumbing that connects a web server (Gunicorn, Uvicorn, Nginx) to your Python app. It's the Python equivalent of "how does the web server hand a request to my code?"

**WSGI** (Web Server Gateway Interface) is the *synchronous* standard, born in 2003 as **PEP 333** and updated for Python 3 in 2010 as **PEP 3333**. A WSGI app is, at its core, a **callable** that takes the request environment and a `start_response` function and returns the body:

```python
# A complete, runnable WSGI app. Save as wsgi_app.py
def application(environ, start_response):
    # environ is a dict of request info: method, path, headers, query string...
    method = environ["REQUEST_METHOD"]
    path = environ["PATH_INFO"]

    status = "200 OK"
    headers = [("Content-Type", "text/plain")]
    start_response(status, headers)         # set status + headers
    body = f"{method} {path}\n".encode()    # body must be bytes
    return [body]                            # iterable of bytes
```

```bash
# Run it with any WSGI server. Python ships a reference one for testing:
python -m wsgiref.simple_server   # serves a demo app on :8000
# In real life: gunicorn wsgi_app:application
```

The catch: WSGI is **one request at a time per worker**, blocking. While your view waits on a slow DB query or an external API call, that worker can do nothing else. You scale by running *more workers/processes* (just like PHP-FPM spawns more workers). It works great, and **Django runs on WSGI by default**.

**ASGI** (Asynchronous Server Gateway Interface) is the modern *async-capable* successor. An ASGI app is an `async` callable taking three arguments — `scope` (connection info), `receive` (await incoming events), and `send` (push outgoing events):

```python
# A complete, runnable ASGI app. Save as asgi_app.py
async def application(scope, receive, send):
    assert scope["type"] == "http"
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({
        "type": "http.response.body",
        "body": b"Hello from ASGI\n",
    })
```

```bash
# Run it with an ASGI server like Uvicorn:
pip install uvicorn
uvicorn asgi_app:application
# => Uvicorn running on http://127.0.0.1:8000
```

**Why ASGI matters — the async payoff.** Because the interface is `async`, a single worker can `await` a slow I/O operation (DB, HTTP call, websocket) and, *while waiting*, handle other requests on the same thread via the event loop. One process juggles thousands of concurrent connections that are mostly waiting on I/O — without one-process-per-request. ASGI also supports **long-lived connections** (WebSockets, Server-Sent Events) that WSGI fundamentally cannot model, because WSGI assumes a single request-in, single-response-out.

| | WSGI | ASGI |
|---|------|------|
| Concurrency model | Synchronous, blocking | Asynchronous, `async`/`await` |
| Connections | One request → one response | HTTP + WebSockets + SSE, long-lived |
| Typical server | Gunicorn | Uvicorn, Hypercorn, Daphne |
| Best for | Steady CPU/DB work, simplicity | High I/O concurrency, real-time |
| In this course | **Django** (default) | **FastAPI** (and modern async Django) |
| PHP analogue | PHP-FPM (per-request worker) | *(no direct equivalent)* |

This is the crux that sets up the rest of the course: **Django is WSGI/synchronous by default** (and you scale it with more workers, PHP-FPM-style), while **FastAPI is ASGI/async from the ground up** (one worker, many concurrent awaited requests). Django 3+ also speaks ASGI and supports `async def` views, so the line is blurrier than it used to be — but the mental model above is what gets asked in interviews.

> Rule of thumb you can quote: *async shines for I/O-bound concurrency* (waiting on networks/DBs), not for CPU-bound work — a tight CPU loop blocks the event loop just as it would block a thread. The GIL means CPU-bound parallelism needs processes, not async.

---

## Common mistakes & gotchas

- **Confusing 401 and 403.** 401 = not authenticated (no/bad credentials). 403 = authenticated but not permitted. Memorize: "401 = who are you?", "403 = I know you, no."
- **Using 200 for everything**, then putting an `{"error": ...}` blob in the body. Clients, monitoring, and retries rely on the status code — return the real 4xx/5xx.
- **Verbs in URLs**: `/createUser`, `/users/42/delete`. The method *is* the verb. Use nouns.
- **Mutating data with GET.** GET must be safe; a crawler or prefetch could fire it and wreck your data.
- **Expecting CORS errors to mean the request didn't reach the server.** It often did — CORS only blocks your JS from *reading* the response. Check server logs.
- **Thinking CORS is security/auth.** It governs browser cross-origin reads only; it does nothing for non-browser clients.
- **Forgetting query params are strings.** `?active=true` is the string `"true"`, which is truthy; do real parsing/coercion (your framework usually does).
- **Treating PATCH as guaranteed idempotent.** Only PUT is, per spec.
- **Retrying POSTs blindly** and creating duplicates — use idempotency keys for create operations that may be retried.
- **Assuming a long-lived Python process is stateless like PHP-FPM.** Module-level globals persist across requests in a WSGI/ASGI worker; don't stash per-user state there.
- **Reaching for async everywhere.** Async helps I/O-bound concurrency; for CPU-bound work it gives nothing and can starve the event loop.

---

## Best practices

- Choose status codes deliberately: **201** for creates (with `Location`), **204** for empty successes, **422** for validation failures, **409** for conflicts.
- Design around **resources (nouns)** and let HTTP methods be the verbs; keep URLs predictable and hierarchical.
- Keep your API **stateless** per request: send auth (token/cookie) on every request; don't rely on in-process memory.
- Use **JSON** with an explicit `Content-Type: application/json`; let serializers/Pydantic handle (de)serialization + validation rather than hand-rolling `json.loads`.
- Put identifiers in the **path**, filters/pagination in the **query string**, payloads in the **body**.
- Make destructive/create operations **retry-safe** where possible (idempotency keys for POST).
- Set safe cookie flags (`HttpOnly`, `Secure`, `SameSite`) for session cookies.
- Configure CORS narrowly: list the **specific origins** that need access rather than `*` in production.
- Default to **WSGI/Django** for straightforward CRUD services; reach for **ASGI/FastAPI (async)** when you have heavy I/O concurrency, websockets, or streaming.

---

## Interview Q&A

**Q: What's the difference between *safe* and *idempotent* HTTP methods? Give examples.**
**A:** *Safe* means read-only with no server-side side effects (GET). *Idempotent* means N identical requests leave the same end-state as one (GET, PUT, DELETE). All safe methods are idempotent, but not vice versa: PUT and DELETE are idempotent yet not safe (they mutate). POST is neither — calling it twice typically creates two resources.

**Q: When would you return 401 vs 403?**
**A:** 401 Unauthorized means the client isn't authenticated — no credentials or invalid ones; the fix is to log in / provide a valid token. 403 Forbidden means the client *is* authenticated but lacks permission for this action; logging in again won't help. Despite the name, 401 is about authentication, 403 about authorization.

**Q: 400 vs 422 — which do you use for a validation error?**
**A:** 400 is for requests the server can't even parse (malformed JSON, missing required structure). 422 Unprocessable Content (the RFC 9110 name; older RFCs called it "Unprocessable Entity") is for syntactically valid requests whose *values* fail validation/business rules (e.g. email already in use). **FastAPI/Pydantic** surface request-validation failures as **422** by default; **DRF** surfaces serializer `ValidationError` as **400** by default (not 422). Classic Django forms also use 400. So the "right" code is somewhat framework-dependent — be ready to say which framework you mean.

**Q: What does it mean that HTTP is stateless, and how do "logged-in" sessions work then?**
**A:** Each HTTP request is independent; the server keeps no memory of prior requests. Continuity is achieved with a cookie: on login the server issues a session id via `Set-Cookie`, the browser returns it on every request via `Cookie`, and the server looks it up in a session store to recover identity. The state lives in the store, not in the connection — so the protocol stays stateless and horizontally scalable.

**Q: Path param vs query string vs request body — when do you use each?**
**A:** Path params identify *which* resource (`/users/42`) and are hierarchical. Query strings modify *how* you view a collection — filtering, sorting, pagination (`?role=admin&page=2`) — and are optional. The body carries the data you're sending to create/update a resource (POST/PUT/PATCH). GET/DELETE usually have no body.

**Q: What makes an API "RESTful"?**
**A:** Adhering to REST's constraints: resources addressed by URLs (nouns), a uniform interface via standard HTTP methods, representations decoupled from resources (content negotiation), statelessness (each request self-contained), cacheability, layered system, and ideally HATEOAS (hypermedia links). Pragmatically: noun-based URLs, correct methods, correct status codes, JSON representations, and no per-client server state.

**Q: What is CORS, and is it a security feature of your server?**
**A:** CORS is a *browser* mechanism that relaxes the Same-Origin Policy, deciding whether JavaScript from one origin may read responses from another. The server opts in via `Access-Control-Allow-*` headers; for non-simple requests the browser first sends a preflight `OPTIONS`. It is enforced by the browser only — it does nothing for curl/Postman/server-to-server — and it is not authentication or authorization.

**Q: Difference between WSGI and ASGI?**
**A:** WSGI is the synchronous Python web-app interface (a callable taking `environ` and `start_response`); it handles one request per worker, blocking on I/O, and can't do websockets. ASGI is the async successor (an `async` callable taking `scope`, `receive`, `send`); it supports `async`/`await`, long-lived connections (WebSockets/SSE), and lets one worker handle many concurrent I/O-bound requests via an event loop.

**Q: Why does async (ASGI) help, and when does it *not*?**
**A:** Async lets a single worker `await` slow I/O and serve other requests while waiting, giving high concurrency for I/O-bound workloads (DB, HTTP calls, websockets) without a process per request. It does *not* help CPU-bound work — a tight CPU loop blocks the event loop just like it would block a thread; with the GIL you need multiple processes for CPU parallelism.

**Q: What status code(s) for a successful POST that creates a resource, vs a successful DELETE?**
**A:** A create should return **201 Created**, ideally with a `Location` header pointing to the new resource (and often the created body). A DELETE that succeeds with nothing to return uses **204 No Content**; it can also return 200 with a body if you echo something back.

**Q: Why might an API work in Postman but fail in the browser with a CORS error?**
**A:** Postman (and curl/mobile/server-to-server) doesn't enforce the Same-Origin Policy, so it ignores CORS entirely. Browsers do enforce it: without the right `Access-Control-Allow-Origin` (and a passing preflight for non-simple requests), the browser blocks your JS from reading the response even though the request reached the server. Fix it by configuring CORS headers on the server.

**Q: What's the difference between a 301 and a 302?**
**A:** 301 Moved Permanently tells clients and caches the resource has a new permanent URL — they should update bookmarks/links and it's SEO-relevant. 302 Found is a temporary redirect — clients should keep using the original URL and not cache the new location.

---

## Practice exercises

1. **Verb/code matching.** For each operation, give the correct HTTP method and the success status code: (a) list all products, (b) create a product, (c) replace product 7 entirely, (d) delete product 7, (e) partially update product 7's price.

2. **Classify the error.** Pick the best status code (400, 401, 403, 404, 409, 422, 429): (a) the JSON body is broken and unparseable; (b) no auth token was sent; (c) a valid user tries to delete someone else's post; (d) registering with an email that already exists; (e) the user sent 1000 requests in a minute and hit your limit; (f) `age` was sent as `-5`.

3. **Where does it go?** For `GET /stores/{store_id}/products?category=books&sort=price&page=3`, identify which value is a path param, which are query params, and what (if anything) would be in the body.

4. **JSON round-trip.** Using only the standard `json` module, serialize the Python dict `{"name": "Ada", "active": True, "manager": None}` to a JSON string, print it, then parse it back and print the Python type of each value. Confirm `True`→`true`→`True` and `None`→`null`→`None`.

5. **Minimal ASGI app.** Write a complete ASGI application that responds to any request with status 200 and the plain-text body `pong`. Show the `uvicorn` command to run it.

6. **Idempotency reasoning.** Explain, in 3–4 sentences, why a mobile client can safely auto-retry a failed `PUT /cart/items/9 {qty: 3}` after a timeout, but must be careful auto-retrying `POST /orders`. What mechanism makes the POST retry safe?

<details>
<summary>Solutions</summary>

**1. Verb/code matching**
- (a) `GET /products` → 200
- (b) `POST /products` → 201 (with `Location`)
- (c) `PUT /products/7` → 200 (or 204)
- (d) `DELETE /products/7` → 204
- (e) `PATCH /products/7` → 200

**2. Classify the error**
- (a) **400** — unparseable/malformed request.
- (b) **401** — not authenticated.
- (c) **403** — authenticated but not permitted.
- (d) **409** — conflict with current state (duplicate email). (Some APIs use 422 here; 409 is the cleaner "conflict" signal.)
- (e) **429** — rate limit exceeded.
- (f) **422** — syntactically valid but fails validation (negative age).

**3. Where does it go?**
- Path param: `store_id` (identifies *which* store).
- Query params: `category=books`, `sort=price`, `page=3` (filtering/sorting/pagination).
- Body: none — it's a `GET`.

**4. JSON round-trip**

```python
import json

data = {"name": "Ada", "active": True, "manager": None}
text = json.dumps(data)
print(text)
# => {"name": "Ada", "active": true, "manager": null}

back = json.loads(text)
for key, value in back.items():
    print(key, "->", value, type(value).__name__)
# name -> Ada str
# active -> True bool
# manager -> None NoneType
```

**5. Minimal ASGI app**

```python
# pong_asgi.py
async def application(scope, receive, send):
    assert scope["type"] == "http"
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({"type": "http.response.body", "body": b"pong"})
```

```bash
uvicorn pong_asgi:application
# then: curl http://127.0.0.1:8000  => pong
```

**6. Idempotency reasoning**
`PUT /cart/items/9 {qty: 3}` is idempotent: whether it runs once or ten times, item 9's quantity ends at 3, so a retry after a timeout is safe — worst case it re-sets the same value. `POST /orders` is not idempotent: each successful call creates a new order, so a blind retry after a timeout could place a duplicate order (the first may have succeeded even though the response was lost). The standard fix is an **idempotency key**: the client sends a unique key with the POST, and the server records it so a retried request with the same key returns the original result instead of creating a second order.

</details>

---

## Cheat sheet

**Methods — safe (S) / idempotent (I):**

```
GET     read           S  I
POST    create         -  -
PUT     full replace   -  I
PATCH   partial update -  -   (not guaranteed idempotent)
DELETE  remove         -  I
```

**Status codes:**

```
2xx success     200 OK · 201 Created (+Location) · 204 No Content
3xx redirect    301 Permanent · 302 Temporary
4xx client      400 Bad Request (malformed)   401 Unauthenticated
                403 Forbidden (no permission)  404 Not Found
                409 Conflict   422 Validation failed   429 Rate limited
5xx server      500 Internal Server Error
```

**Mnemonics:** 401 = "who are you?" · 403 = "I know you, no." · 400 = "can't parse" · 422 = "parsed, but invalid."

**Data placement:** path = *which* resource (`/users/42`) · query = filter/sort/paginate (`?page=2`) · body = payload (POST/PUT/PATCH).

**JSON ↔ Python:** `{}`→dict · `[]`→list · `true/false`→`True/False` · `null`→`None`. `json.dumps`/`json.loads` work on strings; `dump`/`load` on files.

**Statelessness:** each request self-contained; sessions = cookie key → server store; tokens = self-contained in `Authorization: Bearer`.

**REST in one line:** noun-based resource URLs + correct HTTP methods + correct status codes + JSON representations + stateless requests.

**CORS:** browser-only enforcement of cross-origin *reads*; server opts in via `Access-Control-Allow-*`; preflight `OPTIONS` for non-simple requests; not auth, ignored by curl/Postman.

**WSGI vs ASGI:**

```
WSGI  sync   1 req/worker, blocking   Gunicorn   Django default   (~PHP-FPM)
ASGI  async  many concurrent, awaits  Uvicorn    FastAPI          WebSockets/SSE
```

Async wins for **I/O-bound** concurrency; useless for CPU-bound (use processes; mind the GIL).
