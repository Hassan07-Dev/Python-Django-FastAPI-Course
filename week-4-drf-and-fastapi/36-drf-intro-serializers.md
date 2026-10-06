# 36 — DRF: Intro & Serializers

> Week 4 · Django REST Framework

## What you will learn

- What **Django REST Framework (DRF)** adds on top of plain Django, and why you almost always reach for it when building APIs.
- How to install DRF and wire it into `INSTALLED_APPS`.
- The central role of **serializers**: converting Python objects ↔ JSON *and* validating incoming data.
- The mapping from Laravel's **API Resources** and **Form Requests** onto DRF serializers.
- `Serializer` vs `ModelSerializer` — when to use each.
- Declaring fields and the most important field options: `read_only`, `write_only`, `required`, `source`, `default`.
- Validation at three levels: field-level (`validate_<field>`), object-level (`validate()`), and reusable `validators`.
- Writing `create()` and `update()` to control how objects are saved.
- **Nested serializers** and `many=True`.
- `SerializerMethodField` for computed, read-only output.
- A brief look at `to_representation` / `to_internal_value` (the engine under the hood).
- Serializing a **single object** vs a **queryset**, and handling **partial updates** (PATCH).

## PHP/Laravel bridge

If you've built APIs in Laravel, you already know two halves of what a DRF serializer does — DRF just merges them into one class. A serializer's **output side** (`.data`) is Laravel's **API Resource** (`JsonResource` / `ResourceCollection`): it shapes a model into the JSON you send back, renaming and computing fields. A serializer's **input side** (`.validated_data`, `.is_valid()`) is Laravel's **Form Request**: it validates and sanitizes the request body before you touch the database. Where Laravel keeps `rules()` in a `FormRequest` and `toArray()` in a `Resource`, DRF puts validation rules and the output shape together in one `Serializer`. And where a `ModelResource` mirrors a model, DRF's `ModelSerializer` auto-generates fields from your model the way `php artisan make:resource --model` *wishes* it could.

## Core concept: What DRF adds on top of Django

Plain Django is built for server-rendered HTML: views return `HttpResponse` with templates, forms validate HTML form posts, and sessions/CSRF assume a browser. You *can* build a JSON API with vanilla Django (`JsonResponse`, manual `json.loads`, manual validation), but you'd hand-roll a lot.

DRF is a Django app that adds the machinery REST APIs need:

| Concern | Plain Django | Django REST Framework |
|---|---|---|
| Serialization / validation | Manual `json.loads`, Django `Form` | **Serializers** (objects ↔ JSON + validation) |
| Request parsing | `request.POST` (form-encoded only) | `request.data` (JSON, form, multipart — content-negotiated) |
| Responses | `JsonResponse` | `Response` (renderer-aware: JSON, browsable API) |
| Views | `View` / function views | `APIView`, generics, `ViewSet` |
| Auth | Sessions, login forms | Token, JWT, OAuth, session — pluggable per-view |
| Permissions | `@login_required` | `IsAuthenticated`, `IsAdminUser`, object-level perms |
| Browsable docs | none | the **browsable API** (auto HTML UI for your endpoints) |
| Pagination / filtering / throttling | DIY | built in, configurable |

This lesson focuses on the heart of it — **serializers**. Everything else (views, routers, auth) is covered in later lessons.

### Install and register

```bash
pip install djangorestframework
# Pin it in requirements: djangorestframework>=3.15
```

Add it to `INSTALLED_APPS` in `settings.py`:

```python
# settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",   # <-- add this
    "blog",             # your own app
]
```

Adding `rest_framework` is what makes the browsable API templates, default settings, and management hooks available. Optional global config lives under one dict:

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
        "rest_framework.renderers.BrowsableAPIRenderer",  # remove in prod-only APIs
    ],
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.AllowAny",  # tighten later
    ],
}
```

## Core concept: What a serializer is for

A **serializer** does two jobs, in two directions:

1. **Serialization (output):** take a Python object (usually a Django model instance) and turn it into primitives (`dict` / `list` / `str` / `int`) that a renderer can emit as JSON. This is `serializer.data`.
2. **Deserialization (input):** take untrusted incoming data (a parsed JSON body), **validate** it, coerce types, and give you clean `validated_data` you can safely save. This is `serializer.is_valid()` + `serializer.validated_data`.

Think of it as the typed, validated boundary between the messy outside world (JSON over HTTP) and your trusted Python/ORM layer.

Our running model for the lesson:

```python
# blog/models.py
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()


class Author(models.Model):
    name = models.CharField(max_length=120)
    email = models.EmailField(unique=True)

    def __str__(self):
        return self.name


class Article(models.Model):
    DRAFT, PUBLISHED = "draft", "published"
    STATUS_CHOICES = [(DRAFT, "Draft"), (PUBLISHED, "Published")]

    author = models.ForeignKey(Author, related_name="articles", on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    body = models.TextField()
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default=DRAFT)
    views = models.PositiveIntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

## Core concept: `Serializer` vs `ModelSerializer`

### The explicit `Serializer`

A base `Serializer` is like a Django `Form`: you declare every field by hand, and you write `create()` / `update()` yourself. Maximum control, more boilerplate.

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Author


class AuthorSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=120)
    email = serializers.EmailField()

    def create(self, validated_data):
        return Author.objects.create(**validated_data)

    def update(self, instance, validated_data):
        instance.name = validated_data.get("name", instance.name)
        instance.email = validated_data.get("email", instance.email)
        instance.save()
        return instance
```

### `ModelSerializer` — the workhorse

`ModelSerializer` inspects your model and **auto-generates fields**, plus default `create()` and `update()` implementations. This is what you'll use 90% of the time.

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Author, Article


class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = Author
        fields = ["id", "name", "email"]


class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ["id", "author", "title", "body", "status", "views", "created_at"]
        read_only_fields = ["views", "created_at"]
```

The `Meta` class is the contract:

- `model` — which model to mirror.
- `fields` — an explicit allow-list. **Always prefer this over `fields = "__all__"`**, which silently exposes new columns you add later (a real leak risk).
- `exclude` — the opposite of `fields` (can't use both).
- `read_only_fields` — fields included in output but ignored on input.

> **Rule of thumb:** start with `ModelSerializer`. Drop to plain `Serializer` only when your payload doesn't map cleanly to a single model (e.g. a login request, a search query, an aggregate response).

### Using it — both directions

```python
# In a Django shell:  python manage.py shell
from blog.models import Author
from blog.serializers import AuthorSerializer

# --- OUTPUT: object -> dict ---
a = Author.objects.create(name="Ada Lovelace", email="ada@example.com")
s = AuthorSerializer(a)
s.data
# => {'id': 1, 'name': 'Ada Lovelace', 'email': 'ada@example.com'}

# --- INPUT: dict -> validated -> object ---
incoming = {"name": "Alan Turing", "email": "alan@example.com"}
s = AuthorSerializer(data=incoming)   # NOTE: data=... means "deserialize"
s.is_valid()           # => True   (raises nothing; returns bool)
s.validated_data       # => {'name': 'Alan Turing', 'email': 'alan@example.com'}
new_author = s.save()  # calls create() because no instance was passed
new_author.id          # => 2
```

The key distinction: **`Serializer(instance)`** serializes *out*; **`Serializer(data=...)`** deserializes *in*. Calling `.data` before `.is_valid()` on an input serializer raises an `AssertionError` — validate first.

## Core concept: Fields and field options

Even with `ModelSerializer` you'll often **override** individual fields to tweak behaviour. The most important options:

| Option | Meaning |
|---|---|
| `read_only=True` | Appears in output; **ignored** in input. (e.g. `id`, computed fields) |
| `write_only=True` | Accepted in input; **never** appears in output. (e.g. `password`) |
| `required=False` | Field may be omitted from input (defaults still apply). |
| `allow_null=True` | `None`/`null` is a valid value. |
| `default=...` | Value used when the field is absent from input. Implies `required=False`. |
| `source="..."` | Map a JSON key to a different attribute/method on the object. |
| `validators=[...]` | List of reusable validator callables. |

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Article


class ArticleSerializer(serializers.ModelSerializer):
    # Rename JSON key "headline" but read/write the model's "title" attribute.
    headline = serializers.CharField(source="title", max_length=200)

    # Output-only: pull author's name via dotted source (relation traversal).
    author_name = serializers.CharField(source="author.name", read_only=True)

    # Input-only: a password-like secret you never echo back.
    edit_token = serializers.CharField(write_only=True, required=False)

    # Default applied when the client omits "status".
    status = serializers.ChoiceField(
        choices=Article.STATUS_CHOICES, default=Article.DRAFT
    )

    class Meta:
        model = Article
        fields = [
            "id", "headline", "author", "author_name",
            "body", "status", "views", "edit_token", "created_at",
        ]
        read_only_fields = ["views", "created_at"]
```

A couple of beginner-friendly clarifications:

- **`source`** is the single most confusing-but-useful option. `headline = CharField(source="title")` means *"the JSON field is called `headline`, but read and write the object's `.title`."* Dotted paths (`source="author.name"`) traverse relations and are read-only by nature. If the JSON key already equals the attribute name, omit `source` (and never write `source="title"` on a field literally named `title` — DRF errors on the redundancy).
- **`read_only` + `default`** are mutually exclusive — a read-only field never reads input, so a default makes no sense.
- **`write_only`** is how you accept secrets. Combine with `read_only` siblings to fully control the in/out shape independently.

## Core concept: Validation (three levels)

Validation runs inside `is_valid()`. There are three layers, executed in this order: built-in field validation → field-level methods → object-level method.

### 1. Field-level: `validate_<fieldname>(self, value)`

Validates one field in isolation. Return the (possibly cleaned) value, or `raise serializers.ValidationError`.

```python
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ["id", "title", "body", "status"]

    def validate_title(self, value):
        if "free money" in value.lower():
            raise serializers.ValidationError("Spammy title not allowed.")
        return value.strip()   # return cleaned value
```

### 2. Object-level: `validate(self, attrs)`

Runs after all field-level checks pass. Use it when a rule spans **multiple fields**. `attrs` is the dict of validated field values.

```python
    def validate(self, attrs):
        # A published article must have a non-trivial body.
        if attrs.get("status") == Article.PUBLISHED and len(attrs.get("body", "")) < 20:
            raise serializers.ValidationError(
                {"body": "Published articles need at least 20 characters."}
            )
        return attrs
```

> Raising with a **dict** attaches the error to a specific field (`{"body": "..."}`); raising with a **string/list** produces a `non_field_errors` entry. Prefer dicts so clients can map errors to form inputs.

### 3. Reusable `validators`

For rules you reuse across serializers, pass callables (or DRF's built-ins) to the field's `validators` list.

```python
from rest_framework.validators import UniqueValidator
from django.core.validators import MinLengthValidator


class AuthorSerializer(serializers.ModelSerializer):
    email = serializers.EmailField(
        validators=[UniqueValidator(queryset=Author.objects.all(),
                                    message="Email already registered.")]
    )
    name = serializers.CharField(validators=[MinLengthValidator(2)])

    class Meta:
        model = Author
        fields = ["id", "name", "email"]
```

### How errors surface

```python
s = ArticleSerializer(data={"title": "free money!!!", "body": "x", "status": "published"})
s.is_valid()     # => False
s.errors
# => {'title': ['Spammy title not allowed.'],
#     'body': ['Published articles need at least 20 characters.']}

# In a view you usually let it raise and become a 400:
s.is_valid(raise_exception=True)   # raises rest_framework.exceptions.ValidationError -> HTTP 400
```

`is_valid(raise_exception=True)` is the idiomatic call inside DRF views — it turns validation failures into a clean `400 Bad Request` with the error dict as the body, no try/except needed.

## Core concept: `create()` and `update()`

When you call `serializer.save()`, DRF dispatches to:

- **`create(self, validated_data)`** — if the serializer was constructed *without* an instance.
- **`update(self, instance, validated_data)`** — if it *was* given an instance.

`ModelSerializer` provides both for free. You override them when saving needs extra logic — e.g. injecting the current user, hashing a value, or popping `write_only` extras before they hit the model.

```python
class ArticleSerializer(serializers.ModelSerializer):
    edit_token = serializers.CharField(write_only=True, required=False)

    class Meta:
        model = Article
        fields = ["id", "author", "title", "body", "status", "edit_token"]

    def create(self, validated_data):
        # edit_token is not a model field; remove it before create().
        validated_data.pop("edit_token", None)
        # Inject the request user as author (request passed in via context).
        author = self.context["request"].user.author_profile
        return Article.objects.create(author=author, **validated_data)

    def update(self, instance, validated_data):
        validated_data.pop("edit_token", None)
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        return instance
```

Two important details:

- **Extra args to `save()`** are merged into `validated_data`. `serializer.save(author=request.user)` is the cleanest way to inject server-side fields without overriding `create()` at all.
- **`context`** carries request-scoped data into the serializer. Views pass it automatically; in raw code do `ArticleSerializer(data=..., context={"request": request})`.

## Core concept: Nested serializers and `many=True`

### `many=True` — serializing a list/queryset

To serialize *many* objects, pass `many=True`. DRF wraps your serializer in a `ListSerializer`.

```python
qs = Article.objects.all()
data = ArticleSerializer(qs, many=True).data
# => [ {...}, {...}, ... ]   a list of dicts

one = ArticleSerializer(qs.first()).data
# => { ... }                 a single dict
```

Rule: **single object → no `many`**; **queryset/list → `many=True`**. Forgetting `many=True` on a queryset gives a confusing error because DRF tries to read model fields off the queryset object.

### Nesting one serializer inside another (read)

Embed related objects by using a serializer as a field:

```python
class ArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)   # nested OUTPUT

    class Meta:
        model = Article
        fields = ["id", "author", "title", "body", "status"]

# Output now embeds the author object instead of just its id:
# {
#   "id": 1,
#   "author": {"id": 7, "name": "Ada Lovelace", "email": "ada@example.com"},
#   "title": "...", "body": "...", "status": "draft"
# }
```

Reverse relations use `many=True`:

```python
class AuthorSerializer(serializers.ModelSerializer):
    # related_name="articles" on the FK lets us embed an author's articles.
    articles = ArticleSerializer(many=True, read_only=True)

    class Meta:
        model = Author
        fields = ["id", "name", "email", "articles"]
```

> **Writable nested serializers are not automatic.** A nested serializer is `read_only` by default for writes; if you want to *create* an article and its author in one POST, you must override `create()` to pop the nested data and create both objects yourself. For writes it's usually simpler to accept the related object's **id** instead — `author = serializers.PrimaryKeyRelatedField(queryset=Author.objects.all())` — and only nest on output.

A common pattern: nest on read, id on write.

```python
class ArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    author_id = serializers.PrimaryKeyRelatedField(
        queryset=Author.objects.all(), source="author", write_only=True
    )

    class Meta:
        model = Article
        fields = ["id", "author", "author_id", "title", "body", "status"]
# POST sends {"author_id": 7, ...}; GET returns the full nested "author".
```

## Core concept: `SerializerMethodField` for computed output

When the value isn't stored on the model — it's *derived* — use `SerializerMethodField`. It is **read-only by definition** and calls a method named `get_<fieldname>`.

```python
class ArticleSerializer(serializers.ModelSerializer):
    is_popular = serializers.SerializerMethodField()
    word_count = serializers.SerializerMethodField()

    class Meta:
        model = Article
        fields = ["id", "title", "body", "views", "is_popular", "word_count"]

    def get_is_popular(self, obj) -> bool:
        return obj.views > 1000

    def get_word_count(self, obj) -> int:
        return len(obj.body.split())
```

This is the direct analog of adding a computed key in a Laravel API Resource's `toArray()`. Use it for flags, counts, formatted strings, and links — anything you compute rather than store. (For heavy queries inside `get_*`, watch for N+1 problems and use `select_related`/`prefetch_related` in your view.)

## Core concept: `to_representation` and `to_internal_value` (brief)

Under the hood, everything you've seen is built from two methods:

- **`to_representation(self, instance)`** — object → primitive dict. This is what `.data` ultimately calls. Override it to reshape the *entire* output (e.g. wrap, flatten, conditionally hide fields).
- **`to_internal_value(self, data)`** — incoming primitive dict → validated dict (runs field validation/coercion). Override to massage raw input *before* per-field validation.

```python
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ["id", "title", "status"]

    def to_representation(self, instance):
        rep = super().to_representation(instance)   # get the default dict first
        rep["status"] = rep["status"].upper()       # then tweak it
        return rep
# Output: {"id": 1, "title": "...", "status": "DRAFT"}
```

You rarely override these directly — field options, `SerializerMethodField`, and `validate()` cover most needs — but knowing they exist explains *how* serializers work and gives you an escape hatch for unusual shapes.

## Core concept: Single vs queryset, and partial updates

### Single object vs queryset (recap)

```python
ArticleSerializer(article).data            # one dict
ArticleSerializer(queryset, many=True).data  # list of dicts
```

### Partial updates (PATCH)

A full update (`PUT`) expects every required field. A partial update (`PATCH`) lets the client send only the fields they want to change — pass `partial=True`.

```python
article = Article.objects.get(pk=1)

# PATCH: only update the title; other fields keep their values.
s = ArticleSerializer(article, data={"title": "New title"}, partial=True)
s.is_valid(raise_exception=True)
s.save()   # calls update(); 'required' is relaxed for omitted fields
```

With `partial=True`, DRF skips `required` checks for absent fields, so your `update()` (or the default one) only touches what was sent. DRF's generic views wire this automatically: `PUT` → `partial=False`, `PATCH` → `partial=True`. You'll see those views in the next lesson; here, just remember the flag.

## Common mistakes & gotchas

- **Calling `.data` before `.is_valid()`** on an input serializer → `AssertionError`. Always validate first; use `.validated_data` for clean input and `.data` for output.
- **`fields = "__all__"`** leaks every column, including ones you add later (tokens, internal flags). Use an explicit allow-list.
- **Forgetting `many=True`** on a queryset, or *adding* it for a single object. Match the flag to the shape.
- **Expecting writable nested serializers to "just work."** They don't — override `create()`/`update()` or accept ids for writes.
- **Putting a non-model field in `fields` without declaring it** (or without `write_only`/popping it in `create`) → DRF tries to set a model attribute that doesn't exist and errors on save.
- **`read_only=True` + `default=...`** together — contradictory; DRF ignores or errors. Pick one.
- **Object-level `validate()` raising a bare string** when the error belongs to a field — clients can't map it. Raise `{"field": "msg"}`.
- **N+1 queries from nested serializers / `SerializerMethodField`.** Serializing 100 articles that each lazy-load `author` = 101 queries. Fix in the view with `select_related("author")` / `prefetch_related("articles")`.
- **`source` redundant with the field name** (`title = CharField(source="title")`) raises an assertion. Only set `source` when names differ.
- **`UniqueValidator` vs model `unique=True`.** `ModelSerializer` auto-adds `UniqueValidator` from the model, but if you redeclare the field you may drop it — re-add it explicitly.

## Best practices

- Default to **`ModelSerializer`**; reach for plain `Serializer` only for non-model payloads (login, search, reports).
- Always use an explicit **`fields`** list; treat it as your public API contract.
- Keep **input and output shapes independent** with `read_only` / `write_only` — never echo secrets, never accept server-controlled fields like `id` or `views`.
- Validate at the **right layer**: single-field rules in `validate_<field>`, cross-field rules in `validate()`, reusable rules in `validators`.
- In views, call **`is_valid(raise_exception=True)`** so failures become clean 400s.
- Inject server-side data via **`serializer.save(extra=...)`** or `context`, not by trusting client input.
- **Nest on read, ids on write** unless you genuinely need atomic nested creation.
- Pair nested/computed fields with **`select_related` / `prefetch_related`** in the view to avoid N+1.
- Name serializers per use-case when shapes diverge (`ArticleListSerializer`, `ArticleDetailSerializer`) rather than overloading one with conditionals.

## Interview Q&A

**Q: What does Django REST Framework add on top of plain Django?**
**A:** A toolkit purpose-built for APIs: serializers (object↔JSON + validation), content-negotiated request parsing (`request.data`) and responses, API-oriented views (`APIView`, generics, `ViewSet`), pluggable authentication and permissions, pagination/filtering/throttling, and a browsable HTML API for development. Vanilla Django can produce JSON, but you'd hand-roll all of this.

**Q: What is a serializer, and which two jobs does it do?**
**A:** It's the validated boundary between JSON and Python. Outward (serialization) it turns model instances into primitive dicts for JSON output (`.data`). Inward (deserialization) it validates and coerces incoming data into clean `validated_data` you can safely persist (`.is_valid()` / `.save()`).

**Q: `Serializer` vs `ModelSerializer` — when do you use each?**
**A:** `ModelSerializer` introspects a model to auto-generate fields plus default `create()`/`update()`; use it whenever the payload maps to a model (most of the time). Plain `Serializer` requires you to declare every field and write save logic; use it for non-model payloads like a login request, search query, or aggregate response.

**Q: Explain `read_only` vs `write_only`.**
**A:** `read_only` fields appear in output but are ignored on input — good for `id`, timestamps, and computed values. `write_only` fields are accepted on input but never serialized out — good for secrets like passwords or tokens. Together they let you shape the in and out payloads independently.

**Q: What does `source` do?**
**A:** It decouples the JSON key from the object attribute. `headline = CharField(source="title")` exposes `headline` in JSON while reading/writing `.title`. Dotted paths like `source="author.name"` traverse relations (read-only). You omit `source` when the names already match.

**Q: Describe the three levels of validation.**
**A:** Field-level `validate_<field>(self, value)` checks one field and returns a cleaned value; object-level `validate(self, attrs)` enforces rules across multiple fields after field checks pass; and reusable `validators=[...]` (e.g. `UniqueValidator`) attach callables to a field. They run in that order inside `is_valid()`.

**Q: When are `create()` and `update()` called, and how do they differ?**
**A:** `serializer.save()` dispatches to `create(validated_data)` if the serializer was built without an instance, or `update(instance, validated_data)` if it was given one. `ModelSerializer` supplies both; you override them to inject server-side data, pop write-only extras, or handle nested writes.

**Q: How do you serialize a list of objects?**
**A:** Pass `many=True` (`Serializer(queryset, many=True).data`), which wraps the serializer in a `ListSerializer` and returns a list of dicts. A single object takes no `many` flag.

**Q: Are nested serializers writable by default?**
**A:** No. Nested serializers serialize related objects on output but are read-only for writes; to create/update nested data you must override `create()`/`update()`. The common pattern is to nest on read and accept the related object's id on write (e.g. `PrimaryKeyRelatedField` with `source` and `write_only`).

**Q: What is `SerializerMethodField` for?**
**A:** Read-only, computed output. You declare `x = SerializerMethodField()` and implement `get_x(self, obj)` returning the derived value (flags, counts, formatted strings). It's the DRF equivalent of a computed key in a Laravel API Resource.

**Q: How do you handle a partial update?**
**A:** Construct the serializer with `partial=True` (`Serializer(instance, data=..., partial=True)`). DRF then relaxes `required` for omitted fields, so only the supplied fields are validated and saved. Generic views map `PATCH` to `partial=True` and `PUT` to `partial=False`.

**Q: What's the difference between `.validated_data` and `.data`?**
**A:** `.validated_data` is the cleaned input dict, available only after `is_valid()` on an input serializer — use it for saving. `.data` is the serialized output dict — what you send back. Calling `.data` on an unvalidated input serializer raises an assertion.

## Practice exercises

1. **Basic ModelSerializer.** Given the `Author` model, write `AuthorSerializer` exposing `id` (read-only), `name`, and `email`, where `email` must be unique with the message `"Email already registered."`.

2. **Field shaping.** Write an `ArticleSerializer` where the JSON key is `headline` (mapped to model `title`), `views` and `created_at` are read-only, and an `edit_token` is accepted but never returned.

3. **Three-layer validation.** Add to `ArticleSerializer`: reject any `title` containing the word `"draft"` (field-level), and require that a `status == "published"` article has a `body` of at least 20 characters (object-level, error attached to `body`).

4. **Computed field.** Add `reading_minutes` to `ArticleSerializer` — a read-only field equal to `ceil(word_count / 200)`.

5. **Nested read / id write.** Make `ArticleSerializer` embed the full `author` object on output but accept an `author_id` on input.

6. **Partial update.** Write the few lines that PATCH an existing article's `status` to `"published"` using a serializer, raising on invalid input.

<details>
<summary>Solutions</summary>

```python
# blog/serializers.py
from math import ceil
from rest_framework import serializers
from rest_framework.validators import UniqueValidator
from .models import Author, Article


# 1 -------------------------------------------------------------
class AuthorSerializer(serializers.ModelSerializer):
    email = serializers.EmailField(
        validators=[UniqueValidator(queryset=Author.objects.all(),
                                    message="Email already registered.")]
    )

    class Meta:
        model = Author
        fields = ["id", "name", "email"]   # id is read-only automatically


# 2, 3, 4, 5 ----------------------------------------------------
class ArticleSerializer(serializers.ModelSerializer):
    # 2: rename + write-only secret
    headline = serializers.CharField(source="title", max_length=200)
    edit_token = serializers.CharField(write_only=True, required=False)

    # 5: nested on read, id on write
    author = AuthorSerializer(read_only=True)
    author_id = serializers.PrimaryKeyRelatedField(
        queryset=Author.objects.all(), source="author", write_only=True
    )

    # 4: computed read-only field
    reading_minutes = serializers.SerializerMethodField()

    class Meta:
        model = Article
        fields = [
            "id", "headline", "author", "author_id",
            "body", "status", "views", "created_at",
            "edit_token", "reading_minutes",
        ]
        read_only_fields = ["views", "created_at"]   # 2

    def get_reading_minutes(self, obj) -> int:       # 4
        return ceil(len(obj.body.split()) / 200)

    def validate_headline(self, value):              # 3 field-level
        if "draft" in value.lower():
            raise serializers.ValidationError('Title cannot contain "draft".')
        return value.strip()

    def validate(self, attrs):                       # 3 object-level
        status = attrs.get("status", getattr(self.instance, "status", None))
        body = attrs.get("body", getattr(self.instance, "body", "") or "")
        if status == Article.PUBLISHED and len(body) < 20:
            raise serializers.ValidationError(
                {"body": "Published articles need at least 20 characters."}
            )
        return attrs

    def create(self, validated_data):
        validated_data.pop("edit_token", None)
        return super().create(validated_data)

    def update(self, instance, validated_data):
        validated_data.pop("edit_token", None)
        return super().update(instance, validated_data)


# 6 -------------------------------------------------------------
# (run in a shell / view)
def publish(article):
    s = ArticleSerializer(article, data={"status": "published"}, partial=True)
    s.is_valid(raise_exception=True)   # 400 on bad input (e.g. body too short)
    return s.save()
```

</details>

## Cheat sheet

```python
# Install + register
pip install djangorestframework
INSTALLED_APPS += ["rest_framework"]

# Output (serialize)
Serializer(obj).data                 # one dict
Serializer(qs, many=True).data       # list of dicts

# Input (deserialize)
s = Serializer(data=payload)
s.is_valid(raise_exception=True)     # -> 400 on failure
s.validated_data                     # cleaned dict
s.save()                             # create() (no instance) / update() (with instance)

# Update / partial update
Serializer(obj, data=payload).save()                 # full (PUT)
Serializer(obj, data=payload, partial=True).save()   # partial (PATCH)
```

| Field option | Effect |
|---|---|
| `read_only=True` | in output, ignored on input |
| `write_only=True` | accepted on input, hidden from output |
| `required=False` | may be omitted from input |
| `default=...` | value when absent (implies not required) |
| `source="x"` / `"a.b"` | map JSON key → object attr / relation |
| `validators=[...]` | reusable per-field validators |

| Validation hook | Scope |
|---|---|
| `validate_<field>(self, value)` | one field |
| `validate(self, attrs)` | cross-field |
| `validators=[...]` | reusable callables |

| Need | Tool |
|---|---|
| Mirror a model | `ModelSerializer` + `Meta.fields` |
| Custom/non-model payload | base `Serializer` |
| Computed read-only value | `SerializerMethodField` + `get_<f>` |
| Embed related object (read) | nested serializer / `many=True` |
| Accept related object (write) | `PrimaryKeyRelatedField(source=..., write_only=True)` |
| Reshape whole output/input | `to_representation` / `to_internal_value` |

| DRF | Laravel analog |
|---|---|
| Serializer output (`.data`) | API Resource (`JsonResource`) |
| Serializer input (`is_valid`) | Form Request (`rules()`) |
| `ModelSerializer` | model-bound Resource |
| `many=True` | `ResourceCollection` |
| `SerializerMethodField` | computed key in `toArray()` |
