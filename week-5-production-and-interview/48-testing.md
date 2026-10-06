# 48 — Testing (pytest, Django & FastAPI)

> Week 5 · Production & Interview Prep

## What you will learn

- Why automated testing matters and how the **test pyramid** (unit / integration / e2e) guides where to spend effort.
- Python's built-in `unittest` versus **pytest** — and why almost everyone reaches for pytest.
- Core pytest mechanics: plain `assert`, **fixtures**, `conftest.py`, `@pytest.mark.parametrize`, **markers**, and the most useful plugins.
- Testing **Django**: `TestCase`, the transactional test database, the test `Client`, `setUpTestData`, and `pytest-django`.
- Testing **DRF** APIs with `APITestCase`, `APIClient`, and `force_authenticate`.
- Testing **FastAPI** with `TestClient` / `httpx` and `app.dependency_overrides` to swap in fakes.
- **Mocking** with `unittest.mock` (`Mock`, `patch`, `return_value`, `side_effect`) — and the golden rule of *where* to patch.
- Building test data with **factory_boy** instead of hand-written fixtures.
- Measuring **coverage** and running a pragmatic **TDD** loop.
- A side-by-side mapping from **PHPUnit** so your existing instincts transfer.

## PHP/Laravel bridge

You already know this world: in Laravel you write tests under `tests/`, extend `TestCase`, use `$this->assertEquals(...)`, hit endpoints with `$this->getJson('/api/users')`, get a fresh DB per test via `RefreshDatabase`, and build records with **model factories** (`User::factory()->create()`). Python's stack maps almost one-to-one: **pytest** ≈ PHPUnit (but with plain `assert` and far less ceremony), **Django `TestCase`** wraps each test in a DB transaction the same way `DatabaseTransactions` does, **DRF `APIClient`** ≈ Laravel's HTTP test helpers, **factory_boy** ≈ Eloquent factories, and **`unittest.mock.patch`** ≈ Mockery / `Http::fake()`. The biggest mental shift is pytest's **fixture** system — it replaces Laravel's `setUp()` + service-container juggling with small, composable functions that get *injected by name*.

---

## Core concept — Why test, and the test pyramid

Tests exist to let you change code **without fear**. They document intent, catch regressions before users do, and make refactoring safe. For an interview, you must be able to articulate *what kind* of test you reach for and *why*.

The **test pyramid** describes the healthy ratio:

```
        /\        e2e / system  — few, slow, brittle, high confidence
       /  \       (real browser, real services)
      /----\
     /      \     integration   — some, medium speed
    /        \    (DB, your API end-to-end in-process, message queues)
   /----------\
  /            \  unit           — many, fast, isolated
 /______________\ (one function/class, no I/O)
```

| Level | Scope | Speed | Example |
|-------|-------|-------|---------|
| **Unit** | One function/class, dependencies faked | milliseconds | "discount calculator returns 10% off" |
| **Integration** | Several units + real infra (DB, ORM) | tens of ms | "POST /orders writes a row and returns 201" |
| **e2e** | Whole system through the front door | seconds | "user signs up via the browser" |

Rule of thumb: **many fast unit tests, fewer integration tests, very few e2e tests.** An inverted pyramid (mostly slow e2e) gives a flaky, slow suite nobody trusts.

---

## Core concept — `unittest` vs `pytest`

Python ships with `unittest` (JUnit-style, class-based). It works, but it is verbose:

```python
# test_math_unittest.py  — the built-in style
import unittest

def add(a, b):
    return a + b

class TestAdd(unittest.TestCase):
    def test_adds_positive(self):
        self.assertEqual(add(2, 3), 5)

    def test_adds_negative(self):
        self.assertEqual(add(-1, -1), -2)

if __name__ == "__main__":
    unittest.main()
```

The same thing in **pytest** — no class, no `self`, no `assertEqual`, just `assert`:

```python
# test_math.py  — pytest style
def add(a, b):
    return a + b

def test_adds_positive():
    assert add(2, 3) == 5

def test_adds_negative():
    assert add(-1, -1) == -2
```

```bash
pip install pytest
pytest                # auto-discovers test_*.py / *_test.py, functions named test_*
pytest -v             # verbose: one line per test
pytest -k "positive"  # only tests whose name matches the expression
pytest test_math.py::test_adds_positive   # a single test
```

pytest **rewrites** the `assert` statement so failures are richly explained — you don't lose the readable diff you'd get from `assertEqual`:

```
>       assert add(2, 3) == 6
E       assert 5 == 6
E        +  where 5 = add(2, 3)
```

**Why pytest wins:** plain asserts, fixtures instead of `setUp`, `parametrize`, a huge plugin ecosystem, and it can *run* your existing `unittest` classes unchanged. Use pytest. (It is roughly Python's PHPUnit, minus the boilerplate.)

### Testing that an exception is raised

```python
import pytest

def withdraw(balance, amount):
    if amount > balance:
        raise ValueError("insufficient funds")
    return balance - amount

def test_overdraw_raises():
    with pytest.raises(ValueError, match="insufficient"):
        withdraw(100, 250)
```

`match` is a regex checked against the exception message — handy and concise.

---

## Core concept — Fixtures and `conftest.py`

A **fixture** is a function that produces something a test needs (a DB connection, a user, a temp dir). Tests "request" a fixture simply by naming it as a parameter — pytest injects it. This is dependency injection for tests.

```python
# test_cart.py
import pytest

class Cart:
    def __init__(self):
        self.items = []
    def add(self, item):
        self.items.append(item)
    def total(self):
        return sum(i["price"] for i in self.items)

@pytest.fixture
def empty_cart():
    return Cart()

@pytest.fixture
def cart_with_items(empty_cart):          # fixtures can depend on fixtures
    empty_cart.add({"name": "pen", "price": 2})
    empty_cart.add({"name": "ink", "price": 5})
    return empty_cart

def test_empty_cart_total(empty_cart):    # request by name
    assert empty_cart.total() == 0

def test_filled_cart_total(cart_with_items):
    assert cart_with_items.total() == 7
```

### Setup *and* teardown with `yield`

Code before `yield` is setup; code after runs as teardown — even if the test fails. This is the pytest equivalent of `setUp` + `tearDown`.

```python
@pytest.fixture
def db_connection():
    conn = connect("sqlite://")   # setup
    yield conn                    # hand it to the test
    conn.close()                  # teardown, always runs
```

### Fixture scope — reuse expensive setup

```python
@pytest.fixture(scope="session")   # created once per test run
def http_client():
    return SomeExpensiveClient()
```

| Scope | Created once per |
|-------|-----------------|
| `function` (default) | each test |
| `class` | test class |
| `module` | `.py` file |
| `package` | package |
| `session` | whole `pytest` run |

### `conftest.py` — shared fixtures, no import needed

Put fixtures in a `conftest.py` and pytest makes them available to every test in that directory tree automatically (no import statement). This is the idiomatic home for cross-cutting setup.

```python
# conftest.py
import pytest

@pytest.fixture
def api_base_url():
    return "http://testserver"
```

### Built-in fixtures you'll use constantly

```python
def test_writes_file(tmp_path):           # tmp_path: a unique temp dir (pathlib.Path)
    f = tmp_path / "data.txt"
    f.write_text("hi")
    assert f.read_text() == "hi"

def test_logs(caplog):                     # caplog: capture log records
    ...

def test_env(monkeypatch):                 # monkeypatch: safely patch attrs/env, auto-undone
    monkeypatch.setenv("API_KEY", "test")
    assert os.environ["API_KEY"] == "test"
```

---

## Core concept — `parametrize`, markers, and plugins

### `parametrize` — one test, many inputs

Instead of copy-pasting a test for each case, feed it a table of inputs. Each row becomes a separate, individually-reported test.

```python
import pytest

def is_even(n):
    return n % 2 == 0

@pytest.mark.parametrize("number, expected", [
    (2, True),
    (3, False),
    (0, True),
    (-4, True),
])
def test_is_even(number, expected):
    assert is_even(number) == expected
# => 4 test cases reported separately: test_is_even[2-True], [3-False], ...
```

### Markers — tag and select tests

```python
import pytest

@pytest.mark.slow
def test_full_import():
    ...

@pytest.mark.skip(reason="endpoint not built yet")
def test_future_feature():
    ...

@pytest.mark.skipif(sys.platform == "win32", reason="POSIX only")
def test_unix_perms():
    ...

@pytest.mark.xfail(reason="known bug, tracked in #123")
def test_known_bug():
    assert buggy() == "right answer"
```

Register custom markers in `pyproject.toml` so pytest doesn't warn, then select with `-m`:

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "slow: marks slow tests",
]
```

```bash
pytest -m slow          # only slow tests
pytest -m "not slow"    # everything except slow tests
```

### Plugins worth knowing (interview-ready list)

| Plugin | What it gives you |
|--------|-------------------|
| `pytest-django` | Django settings, DB fixtures, `@pytest.mark.django_db` |
| `pytest-cov` | coverage integrated into the run (`--cov`) |
| `pytest-asyncio` | `async def` tests (needed for async FastAPI internals) |
| `pytest-mock` | the `mocker` fixture (a friendlier `unittest.mock`) |
| `pytest-xdist` | parallel test execution (`-n auto`) |
| `factory_boy` | model/data factories (library, not strictly a plugin) |
| `freezegun` | freeze `datetime.now()` for time-dependent tests |

---

## Core concept — Testing Django

Django ships its own test framework layered on `unittest`. The key class is `django.test.TestCase`.

### The transactional test database

Django creates a **separate test database** (prefixed `test_`), and `TestCase` wraps **each test method in a database transaction that is rolled back** at the end. So every test starts from a clean slate and tests can't pollute each other — exactly like Laravel's `RefreshDatabase` / `DatabaseTransactions`.

```python
# blog/tests/test_models.py
from django.test import TestCase
from blog.models import Article

class ArticleModelTests(TestCase):
    def test_slug_is_generated(self):
        a = Article.objects.create(title="Hello World")
        self.assertEqual(a.slug, "hello-world")
        # this row vanishes after the test — transaction rolled back
```

```bash
python manage.py test                  # run the whole suite
python manage.py test blog             # one app
python manage.py test blog.tests.test_models.ArticleModelTests   # dotted path, not pytest's ::
python manage.py test --keepdb         # reuse the test DB between runs (faster)
```

### `setUpTestData` — create shared rows once per class

`setUp` runs before *every* test (slow if you create lots of data). `setUpTestData` is a classmethod that runs **once per test class**; the rows live inside the outer transaction and are visible (read-only-ish) to all tests. Prefer it for fixtures you don't mutate.

```python
from django.test import TestCase
from blog.models import Author, Article

class ArticleQueryTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = Author.objects.create(name="Ada")
        cls.a1 = Article.objects.create(title="One", author=cls.author)
        cls.a2 = Article.objects.create(title="Two", author=cls.author)

    def test_author_article_count(self):
        self.assertEqual(self.author.articles.count(), 2)

    def test_filter_by_title(self):
        self.assertEqual(Article.objects.filter(title="One").count(), 1)
```

### The test `Client` and Django's assertions

`self.client` lets you exercise views end-to-end (URL routing → view → template/JSON) in-process, no running server.

```python
from django.test import TestCase
from django.urls import reverse
from django.contrib.auth import get_user_model

class ViewTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = get_user_model().objects.create_user(
            username="ada", password="pw12345"
        )

    def test_home_status_and_template(self):
        resp = self.client.get(reverse("home"))
        self.assertEqual(resp.status_code, 200)
        self.assertTemplateUsed(resp, "home.html")
        self.assertContains(resp, "Welcome")          # checks 200 + substring

    def test_dashboard_requires_login(self):
        resp = self.client.get(reverse("dashboard"))
        self.assertRedirects(resp, "/accounts/login/?next=/dashboard/")

    def test_dashboard_when_authenticated(self):
        self.client.force_login(self.user)            # skip the login form
        resp = self.client.get(reverse("dashboard"))
        self.assertEqual(resp.status_code, 200)
```

Handy Django assertions: `assertContains`, `assertNotContains`, `assertTemplateUsed`, `assertRedirects`, `assertQuerySetEqual`, `assertNumQueries` (catch N+1!).

```python
def test_no_n_plus_one(self):
    with self.assertNumQueries(2):
        list(Article.objects.select_related("author").all())
```

> **`TestCase` vs `TransactionTestCase`:** `TestCase` rolls back via a transaction (fast) but means you **cannot test `transaction.atomic` / `on_commit` behavior** properly. When you need real commits (e.g. testing `transaction.on_commit` hooks, or DB constraints that fire on commit), use `TransactionTestCase`, which truncates tables instead — slower but honest.

### pytest-django — the same power, pytest-style

Many teams use pytest for Django too. Install `pytest-django`, point it at your settings, and mark DB-touching tests.

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings
python_files = tests.py test_*.py *_tests.py
```

```python
# test_models_pytest.py
import pytest
from blog.models import Article

@pytest.mark.django_db                 # grants DB access (rolled back after)
def test_article_str():
    a = Article.objects.create(title="Hi")
    assert str(a) == "Hi"

@pytest.fixture
def author(db):                        # `db` fixture = DB access for this fixture
    from blog.models import Author
    return Author.objects.create(name="Ada")

def test_author_has_no_articles(author):
    assert author.articles.count() == 0
```

pytest-django also gives you a `client` fixture (the Django test client) and `admin_client` (logged-in admin) for free.

---

## Core concept — Testing DRF APIs

Django REST Framework provides test classes that understand JSON, content negotiation, and auth. `APITestCase` is `TestCase` plus an `APIClient`.

```python
# api/tests/test_articles.py
from rest_framework import status
from rest_framework.test import APITestCase
from django.urls import reverse
from django.contrib.auth import get_user_model
from blog.models import Article

class ArticleAPITests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = get_user_model().objects.create_user("ada", password="pw")

    def test_list_articles_returns_200(self):
        Article.objects.create(title="One")
        url = reverse("article-list")
        resp = self.client.get(url)
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(len(resp.data), 1)

    def test_create_requires_auth(self):
        url = reverse("article-list")
        resp = self.client.post(url, {"title": "New"}, format="json")
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)

    def test_create_when_authenticated(self):
        self.client.force_authenticate(user=self.user)   # bypass real auth
        url = reverse("article-list")
        resp = self.client.post(url, {"title": "New"}, format="json")
        self.assertEqual(resp.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Article.objects.count(), 1)
        self.client.force_authenticate(user=None)        # reset
```

**`force_authenticate(user=...)`** attaches the user to the request without exercising tokens/sessions — fast and reliable for testing *authorization logic* rather than the auth mechanism itself. To test the real token flow, send the header instead:

```python
self.client.credentials(HTTP_AUTHORIZATION="Bearer " + access_token)
```

Note `format="json"` — without it, `APIClient` sends multipart form data. Always pass `format="json"` for JSON APIs.

If you prefer pytest, DRF ships `APIClient` you can wrap in a fixture:

```python
import pytest
from rest_framework.test import APIClient

@pytest.fixture
def api_client():
    return APIClient()

@pytest.mark.django_db
def test_list(api_client):
    resp = api_client.get("/api/articles/")
    assert resp.status_code == 200
```

---

## Core concept — Testing FastAPI

FastAPI gives you a `TestClient` (built on Starlette's, which wraps **httpx**). It runs your app in-process — no server, no network.

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

DB: dict[int, Item] = {}

@app.post("/items/{item_id}", status_code=201)
def create_item(item_id: int, item: Item):
    DB[item_id] = item
    return item

@app.get("/items/{item_id}")
def read_item(item_id: int):
    if item_id not in DB:
        raise HTTPException(status_code=404, detail="not found")
    return DB[item_id]
```

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_create_and_read():
    resp = client.post("/items/1", json={"name": "pen", "price": 2.5})
    assert resp.status_code == 201
    assert resp.json() == {"name": "pen", "price": 2.5}

    resp = client.get("/items/1")
    assert resp.status_code == 200
    assert resp.json()["name"] == "pen"

def test_missing_item_returns_404():
    resp = client.get("/items/999")
    assert resp.status_code == 404
    assert resp.json() == {"detail": "not found"}
```

### `dependency_overrides` — swap real dependencies for fakes

FastAPI's dependency-injection system is *built* for testing. Anything provided via `Depends(...)` can be replaced through `app.dependency_overrides`. This is how you avoid a real database, a real auth check, or a real external client during tests.

```python
# deps.py
def get_db():
    db = RealDatabase()
    try:
        yield db
    finally:
        db.close()

def get_current_user():
    # in production: decode a JWT, hit the DB, etc.
    ...
```

```python
# app.py
from fastapi import FastAPI, Depends
from deps import get_db, get_current_user

app = FastAPI()

@app.get("/me")
def me(user=Depends(get_current_user)):
    return {"username": user.username}
```

```python
# test_app.py
from fastapi.testclient import TestClient
from app import app
from deps import get_current_user

class FakeUser:
    username = "ada"

def override_get_current_user():
    return FakeUser()

def test_me_returns_current_user():
    app.dependency_overrides[get_current_user] = override_get_current_user
    client = TestClient(app)
    resp = client.get("/me")
    assert resp.status_code == 200
    assert resp.json() == {"username": "ada"}
    app.dependency_overrides.clear()      # always clean up
```

For testing **async** endpoints with their async dependencies fully exercised, use `httpx.ASGITransport` with an `AsyncClient` and `pytest-asyncio`:

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app import app

@pytest.mark.asyncio
async def test_async_endpoint():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        resp = await ac.get("/me")
    assert resp.status_code == 200
```

---

## Core concept — Mocking with `unittest.mock`

A **mock** is a stand-in object that records how it was called and returns whatever you tell it to. Use mocks to isolate the unit under test from slow / external / nondeterministic things (network, time, payment APIs, email).

```python
from unittest.mock import Mock

svc = Mock()
svc.charge.return_value = {"status": "ok"}     # canned return

result = svc.charge(amount=100)
assert result == {"status": "ok"}

svc.charge.assert_called_once_with(amount=100) # verify the interaction
```

`side_effect` is richer than `return_value`:

```python
from unittest.mock import Mock

# raise an exception
m = Mock(side_effect=TimeoutError("network down"))

# return different values on successive calls
seq = Mock(side_effect=[1, 2, 3])
assert (seq(), seq(), seq()) == (1, 2, 3)

# compute the return value from the args
doubler = Mock(side_effect=lambda x: x * 2)
assert doubler(5) == 10
```

### `patch` — replace something *in place* during a test

```python
# notifications.py
import httpx

def send_alert(message: str) -> int:
    resp = httpx.post("https://hooks.example.com", json={"text": message})
    return resp.status_code
```

```python
# test_notifications.py
from unittest.mock import patch
import notifications

@patch("notifications.httpx.post")          # patch where it is USED
def test_send_alert(mock_post):
    mock_post.return_value.status_code = 200
    code = notifications.send_alert("hi")
    assert code == 200
    mock_post.assert_called_once()
    # inspect the call:
    args, kwargs = mock_post.call_args
    assert kwargs["json"] == {"text": "hi"}
```

You can also use `patch` as a context manager:

```python
with patch("notifications.httpx.post") as mock_post:
    mock_post.return_value.status_code = 500
    assert notifications.send_alert("x") == 500
```

### The #1 mocking gotcha: **patch where it's *looked up*, not where it's defined**

If `notifications.py` does `from httpx import post` (importing the *name* into its own module), then `httpx.post` was copied into `notifications`'s namespace. You must patch `notifications.post`, **not** `httpx.post`. Rule: patch the name **in the module that uses it**.

```python
# If notifications.py has:   from httpx import post
@patch("notifications.post")     # ✅ patches the name actually called
# NOT @patch("httpx.post")       # ❌ too late; the name was already bound
```

### `pytest-mock`: the `mocker` fixture (cleaner)

```python
def test_send_alert(mocker):
    mock_post = mocker.patch("notifications.httpx.post")
    mock_post.return_value.status_code = 200
    assert notifications.send_alert("hi") == 200
```

`mocker` auto-undoes the patch at test end — no decorator stacking, no `with` nesting.

---

## Core concept — Factories with factory_boy

Hand-writing `Model.objects.create(...)` for every field gets tedious and brittle. **factory_boy** (≈ Laravel model factories) generates valid objects with sensible defaults you can override per test.

```python
# factories.py
import factory
from factory.django import DjangoModelFactory
from django.contrib.auth import get_user_model
from blog.models import Article

class UserFactory(DjangoModelFactory):
    class Meta:
        model = get_user_model()

    username = factory.Sequence(lambda n: f"user{n}")   # user0, user1, ...
    email = factory.LazyAttribute(lambda o: f"{o.username}@example.com")

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Faker("sentence", nb_words=4)
    author = factory.SubFactory(UserFactory)            # builds a related User too
    published = False
```

```python
# usage in tests
import pytest
from factories import ArticleFactory, UserFactory

@pytest.mark.django_db
def test_published_filter():
    ArticleFactory(published=True)
    ArticleFactory(published=True)
    ArticleFactory(published=False)              # overrides per call
    from blog.models import Article
    assert Article.objects.filter(published=True).count() == 2

@pytest.mark.django_db
def test_batch():
    UserFactory.create_batch(5)                  # five users at once
```

- `Sequence` — unique values to avoid uniqueness clashes.
- `Faker` — realistic fake data (names, emails, sentences).
- `SubFactory` — auto-create related objects (foreign keys).
- `LazyAttribute` — derive a field from other fields on the instance.
- `.build()` makes an unsaved instance; `.create()` saves it.

factory_boy works for plain SQLAlchemy too via `factory.alchemy.SQLAlchemyModelFactory`.

---

## Core concept — Coverage and the TDD loop

### Measuring coverage

Coverage tells you which lines ran during the tests. It shows *gaps*, not *quality* — 100% coverage with weak assertions still proves little. Aim for high coverage on business logic; don't chase 100% on trivial glue.

```bash
pip install pytest-cov
pytest --cov=myapp --cov-report=term-missing
```

```
Name                 Stmts   Miss  Cover   Missing
--------------------------------------------------
myapp/services.py       40      3    92%   55-57
myapp/models.py         22      0   100%
--------------------------------------------------
TOTAL                   62      3    95%
```

```bash
pytest --cov=myapp --cov-report=html     # browsable report in htmlcov/index.html
pytest --cov=myapp --cov-fail-under=85   # fail CI if below 85%
```

### A pragmatic TDD loop (Red → Green → Refactor)

1. **Red** — write a failing test that states the desired behavior.
2. **Green** — write the *minimum* code to make it pass.
3. **Refactor** — clean up code *and* test while the bar stays green.

```python
# Step 1 (Red): test first — function doesn't exist yet
def test_apply_discount():
    assert apply_discount(100, 0.10) == 90

# Step 2 (Green): minimal implementation
def apply_discount(price, rate):
    return price - price * rate

# Step 3 (Refactor): add edge cases as new red tests, then improve
import pytest
@pytest.mark.parametrize("price, rate, expected", [
    (100, 0.10, 90),
    (100, 0.0, 100),
    (0, 0.5, 0),
])
def test_apply_discount_cases(price, rate, expected):
    assert apply_discount(price, rate) == expected
```

You don't have to do strict TDD everywhere, but "write the test, watch it fail, then fix" is the safest way to be sure a test actually *tests* something.

---

## Common mistakes & gotchas

- **Patching in the wrong place.** Patch the name in the module that *uses* it, not where it's defined. If `from x import y` was used, patch `mymodule.y`. This is the single most common mocking bug.
- **Forgetting `format="json"` in DRF tests** — `APIClient` defaults to multipart, so your JSON body silently arrives mangled.
- **Tests depending on order or shared mutable state.** Each test must pass in isolation (`pytest -p no:randomly` aside). Don't rely on data another test left behind.
- **Using `TestCase` when you need real commits.** `transaction.on_commit` callbacks and commit-time constraints won't fire under `TestCase`'s rollback; use `TransactionTestCase` / `pytest.mark.django_db(transaction=True)`.
- **Over-mocking.** Mocking everything tests your mocks, not your code. Mock at the *boundary* (network, clock, third-party SDK); use the real DB for ORM logic.
- **Forgetting to clear `app.dependency_overrides`** in FastAPI — overrides leak into later tests. Clear them in teardown or a fixture.
- **Hitting the network in unit tests.** Real HTTP calls make tests slow and flaky and can fail offline. Mock them.
- **Time/randomness in assertions.** `datetime.now()` and `random` make tests nondeterministic — freeze time (`freezegun`) and seed/patch randomness.
- **No DB marker under pytest-django.** A test that touches the ORM without `@pytest.mark.django_db` (or the `db` fixture) raises an error — by design, to keep pure unit tests DB-free.
- **Confusing coverage with correctness.** A line being executed ≠ a line being *asserted on*.

## Best practices

- **One behavior per test**, with a descriptive name (`test_create_requires_auth`) — the name should read like a spec.
- Follow **Arrange–Act–Assert** (a.k.a. Given/When/Then) structure inside each test.
- Keep tests **fast and isolated**; push slow/e2e tests into a marked group (`-m "not slow"` for the inner loop, full suite in CI).
- Put shared fixtures in **`conftest.py`**; use **factories** for data, not giant JSON fixture files.
- Prefer `setUpTestData` over `setUp` in Django for unchanging rows (created once per class).
- **Test behavior, not implementation** — assert on outputs and observable effects, not private internals, so refactors don't break tests.
- Use `force_authenticate` / `dependency_overrides` to test *authorization logic* quickly; have a *few* tests that exercise the real auth path too.
- Run tests in **CI on every push**, with `--cov-fail-under` to prevent coverage regressions; parallelize with `pytest -n auto` (xdist) when the suite grows.
- Make a failing test the first step of every bug fix (it becomes a permanent regression guard).

## Interview Q&A

**Q: What is the test pyramid and why does it matter?**
**A:** It's a guideline for test distribution: many fast, isolated **unit** tests at the base; fewer **integration** tests in the middle; very few slow, brittle **e2e** tests at the top. It matters because it optimizes for a fast, reliable feedback loop. An inverted pyramid (mostly e2e) is slow, flaky, and expensive to maintain, so people stop trusting and running it.

**Q: Why pytest over `unittest`?**
**A:** Less boilerplate (plain `assert` with rich failure introspection, no `self.assertEqual`), a powerful **fixture** system for composable setup/teardown via dependency injection, `parametrize` for table-driven tests, **markers** for selecting/skipping, and a large plugin ecosystem (`pytest-django`, `pytest-cov`, `pytest-asyncio`). pytest can also run existing `unittest` classes unchanged.

**Q: What's a fixture, and how is it different from `setUp`?**
**A:** A fixture is a function that produces a resource a test needs; tests request it by naming it as a parameter (DI). Unlike a single monolithic `setUp`, fixtures are **composable** (fixtures can depend on fixtures), **scoped** (function/class/module/session), and handle teardown via `yield`. `conftest.py` shares them across files without imports.

**Q: How does Django keep tests isolated at the database level?**
**A:** Django creates a dedicated test database and `TestCase` wraps each test method in a **transaction that's rolled back** afterward, so every test starts clean and tests can't pollute each other. It's analogous to Laravel's `RefreshDatabase`/`DatabaseTransactions`.

**Q: When would you use `TransactionTestCase` instead of `TestCase`?**
**A:** When you need **real commits** — e.g. testing `transaction.on_commit` hooks, behavior that spans multiple transactions, or commit-time database constraints. `TestCase`'s rollback hides those. `TransactionTestCase` truncates tables instead of rolling back; it's slower but exercises real commit semantics. Under pytest-django that's `@pytest.mark.django_db(transaction=True)`.

**Q: `setUp` vs `setUpTestData` in Django?**
**A:** `setUp` runs before *every* test method; `setUpTestData` is a classmethod that runs **once per class** and creates rows inside the class-level transaction shared by all its tests. Use `setUpTestData` for read-only shared data — it's much faster than recreating it per test.

**Q: How do you test a DRF endpoint that requires authentication?**
**A:** Use `APITestCase`/`APIClient` and call `self.client.force_authenticate(user=user)` to attach a user without exercising the token/session machinery — fast for testing authorization. To test the real auth flow, set the header with `self.client.credentials(HTTP_AUTHORIZATION="Bearer ...")`. And remember `format="json"` on write requests.

**Q: How do you replace a real database or external service in FastAPI tests?**
**A:** Through `app.dependency_overrides`. Anything injected via `Depends(...)` can be swapped for a fake by assigning `app.dependency_overrides[real_dep] = fake_dep`. It's the cleanest seam in FastAPI for testing — just remember to `clear()` overrides afterward. Use `TestClient` (sync) or `httpx.AsyncClient` + `ASGITransport` + `pytest-asyncio` for async paths.

**Q: Explain `return_value` vs `side_effect` on a Mock.**
**A:** `return_value` is the fixed value the mock returns on every call. `side_effect` is more flexible: set it to an **exception** (to raise), an **iterable** (return successive values per call), or a **callable** (compute the return from the args). If `side_effect` returns `DEFAULT`, `return_value` is used.

**Q: What's the most common mistake when using `patch`?**
**A:** Patching the wrong target. You must patch the name **where it is looked up**, i.e. in the module that uses it. If a module did `from lib import func`, you patch `mymodule.func`, not `lib.func`, because the name was already bound into `mymodule` at import time.

**Q: What are factories and why prefer them over fixtures files?**
**A:** Factories (factory_boy, like Laravel model factories) generate valid model instances with sensible defaults you override per test, including related objects via `SubFactory` and unique values via `Sequence`/`Faker`. They keep tests readable and resilient — adding a required model field only means updating one factory, not dozens of static fixtures.

**Q: Does high code coverage mean the code is well tested?**
**A:** No. Coverage only measures which lines *executed*, not whether the behavior was *asserted* or whether edge cases were covered. It's useful for finding untested code, but you can have 100% coverage with worthless assertions. Treat it as a floor and a gap-finder, not a quality score.

**Q: How does this all map to PHPUnit/Laravel?**
**A:** pytest ≈ PHPUnit; pytest fixtures/`conftest.py` ≈ `setUp`/traits/container bindings; Django `TestCase` rollback ≈ `RefreshDatabase`; DRF `APIClient` ≈ Laravel's `getJson`/`postJson`; `force_authenticate` ≈ `actingAs`; factory_boy ≈ Eloquent factories; `unittest.mock.patch` ≈ Mockery / `Http::fake()`; `pytest --cov` ≈ PHPUnit with Xdebug/pcov coverage.

## Practice exercises

1. **pytest basics + parametrize.** Write `slugify(text)` that lowercases, trims, and replaces spaces with hyphens (`"  Hello World "` → `"hello-world"`). Write a parametrized test covering at least four inputs including empty string and extra whitespace.

2. **Fixtures + exceptions.** Implement a `BankAccount` class with `deposit`/`withdraw` (raise `ValueError` on overdraft). Write a fixture that returns an account with balance 100, and tests that deposit increases the balance, withdraw decreases it, and overdrawing raises with a matching message.

3. **Mocking.** Given `def latest_price(symbol): return httpx.get(f"https://api.example.com/{symbol}").json()["price"]`, write a test that patches the HTTP call so no network is hit, returns a fake price, and asserts both the returned value and that the correct URL was requested.

4. **Django (pytest-django).** For an `Article` model with a `published` boolean, write a test (using `@pytest.mark.django_db` and a factory) asserting that a `published_articles()` manager/queryset returns only published rows.

5. **DRF.** Write an `APITestCase` test proving that `POST /api/articles/` returns **403** when anonymous and **201** when authenticated (use `force_authenticate`), and that exactly one row is created.

6. **FastAPI.** Given an endpoint `GET /me` that depends on `get_current_user`, override the dependency with a fake user and assert the response body — then verify the override is cleared so a second test sees the real (failing) dependency.

<details>
<summary>Solutions</summary>

```python
# 1. slugify
import pytest

def slugify(text):
    return "-".join(text.strip().lower().split())

@pytest.mark.parametrize("raw, expected", [
    ("  Hello World ", "hello-world"),
    ("Python", "python"),
    ("a  b   c", "a-b-c"),
    ("", ""),
])
def test_slugify(raw, expected):
    assert slugify(raw) == expected
```

```python
# 2. BankAccount + fixture
import pytest

class BankAccount:
    def __init__(self, balance=0):
        self.balance = balance
    def deposit(self, amount):
        self.balance += amount
    def withdraw(self, amount):
        if amount > self.balance:
            raise ValueError("insufficient funds")
        self.balance -= amount

@pytest.fixture
def account():
    return BankAccount(balance=100)

def test_deposit(account):
    account.deposit(50)
    assert account.balance == 150

def test_withdraw(account):
    account.withdraw(40)
    assert account.balance == 60

def test_overdraw_raises(account):
    with pytest.raises(ValueError, match="insufficient"):
        account.withdraw(1000)
```

```python
# 3. Mocking the HTTP call
# prices.py
import httpx
def latest_price(symbol):
    return httpx.get(f"https://api.example.com/{symbol}").json()["price"]

# test_prices.py
from unittest.mock import patch
import prices

@patch("prices.httpx.get")
def test_latest_price(mock_get):
    mock_get.return_value.json.return_value = {"price": 42.0}
    assert prices.latest_price("ACME") == 42.0
    mock_get.assert_called_once_with("https://api.example.com/ACME")
```

```python
# 4. Django queryset (pytest-django)
import pytest
from factories import ArticleFactory
from blog.models import Article

@pytest.mark.django_db
def test_published_only():
    ArticleFactory(published=True)
    ArticleFactory(published=True)
    ArticleFactory(published=False)
    assert Article.objects.filter(published=True).count() == 2
```

```python
# 5. DRF auth
from rest_framework import status
from rest_framework.test import APITestCase
from django.urls import reverse
from django.contrib.auth import get_user_model
from blog.models import Article

class CreateArticleTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = get_user_model().objects.create_user("ada", password="pw")

    def test_anonymous_forbidden(self):
        resp = self.client.post(reverse("article-list"),
                                {"title": "X"}, format="json")
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)
        self.assertEqual(Article.objects.count(), 0)

    def test_authenticated_creates(self):
        self.client.force_authenticate(user=self.user)
        resp = self.client.post(reverse("article-list"),
                                {"title": "X"}, format="json")
        self.assertEqual(resp.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Article.objects.count(), 1)
```

```python
# 6. FastAPI dependency override
from fastapi.testclient import TestClient
from app import app
from deps import get_current_user

class FakeUser:
    username = "ada"

def test_me_with_override():
    app.dependency_overrides[get_current_user] = lambda: FakeUser()
    client = TestClient(app)
    resp = client.get("/me")
    assert resp.json() == {"username": "ada"}
    app.dependency_overrides.clear()   # next test sees the real dependency
```

</details>

## Cheat sheet

```bash
# Running pytest
pytest                       # discover & run
pytest -v                    # verbose
pytest -k "auth and not slow"# select by name expression
pytest -m slow               # select by marker
pytest path/test_x.py::test_y# single test
pytest -x --lf               # stop on first fail; rerun last failures
pytest -n auto               # parallel (pytest-xdist)
pytest --cov=app --cov-report=term-missing   # coverage

# Django
python manage.py test app --keepdb
```

| Need | pytest / Python | Django | DRF | FastAPI |
|------|-----------------|--------|-----|---------|
| Test a unit | `def test_x(): assert ...` | same | same | same |
| Setup/teardown | `@pytest.fixture` + `yield` | `setUpTestData` / `setUp` | `setUpTestData` | `@pytest.fixture` |
| DB access | `@pytest.mark.django_db` | automatic (rollback) | automatic | own session/override |
| HTTP call | — | `self.client.get(...)` | `APIClient` `format="json"` | `TestClient(app)` |
| Auth as user | — | `client.force_login(u)` | `force_authenticate(u)` | `dependency_overrides` |
| Many inputs | `@pytest.mark.parametrize` | — | — | — |
| Real commits | — | `TransactionTestCase` | same | — |

```python
# Mock essentials
from unittest.mock import Mock, patch
m = Mock(return_value=1)            # fixed return
m = Mock(side_effect=ValueError)    # raise / iterable / callable
@patch("module_that_uses_it.name") # patch WHERE USED
m.assert_called_once_with(a, b=1)   # verify interaction

# factory_boy
factory.Sequence(lambda n: f"u{n}")  # unique
factory.Faker("email")               # fake data
factory.SubFactory(OtherFactory)     # FK
Factory.create_batch(5)              # many

# Common assertions
pytest.raises(ValueError, match="...")     # exception
self.assertContains(resp, "text")          # Django: 200 + substring
self.assertRedirects(resp, "/url/")
self.assertNumQueries(2)                   # catch N+1
```

**TDD loop:** Red (failing test) → Green (minimum code) → Refactor (clean, stay green).
