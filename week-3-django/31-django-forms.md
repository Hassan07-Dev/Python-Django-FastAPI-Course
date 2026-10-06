# 31 — Forms & Validation

> Week 3 · Django

## What you will learn

- Why Django ships its own forms layer and what three jobs it does (validation, rendering, security)
- Declaring a `forms.Form` with fields and widgets, and how that maps to HTML
- The bound vs. unbound distinction, `is_valid()`, and reading from `cleaned_data`
- Field-level (`clean_<field>`) and form-wide (`clean()`) custom validation
- Rendering forms in templates safely, and the role of the CSRF token
- `ModelForm`: deriving a form from a model with `Meta`, and persisting with `form.save()`
- The canonical GET-vs-POST view pattern and how to display errors
- Formsets and model formsets at a glance
- File and image uploads with `request.FILES` and the right `enctype`
- How all of this maps back to Laravel validation and Form Requests

## PHP/Laravel bridge

In Laravel you usually validate in the controller with `$request->validate([...])` or in a dedicated **Form Request** class (`StoreUserRequest` with a `rules()` method), and you render forms by hand in Blade with helpers like `@csrf`, `old('email')`, and `$errors->first('email')`. Django folds *all* of that — the rules, the error messages, the old-input repopulation, and even the HTML rendering — into one object: the **Form**. A Django `Form` is closest to a Laravel Form Request *plus* a Blade form partial combined. A Django `ModelForm` is the analogue of a Form Request whose rules are auto-derived from your Eloquent model's `$fillable`/casts/column types. And Django's `{% csrf_token %}` is exactly Laravel's `@csrf`.

## Core concept: why a forms layer at all?

A web form does three boring-but-critical jobs, and Django's forms layer does all three so you don't reinvent them per view:

1. **Validation** — coerce raw string input from the request into clean Python types (`int`, `date`, `Decimal`, `bool`) and reject anything invalid with per-field error messages.
2. **Rendering** — produce the HTML `<input>`/`<select>`/`<textarea>` markup, re-fill it with the user's previous input on error, and attach the right labels and error text.
3. **Security** — escape output to prevent XSS, and integrate CSRF protection so POST requests must carry a valid token.

You *can* read `request.POST['email']` by hand, but then you own all the edge cases: empty strings, type coercion, "the user submitted `age=abc`", re-displaying bad input, escaping. Forms make that declarative.

```python
# forms.py
from django import forms

class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    age = forms.IntegerField(min_value=18, required=False)
    message = forms.CharField(widget=forms.Textarea)
    subscribe = forms.BooleanField(required=False)
```

That single class can now validate input, render itself as HTML, and report errors.

## Core concept: fields and widgets

A **field** defines the *data*: its Python type, whether it's required, and its validation rules. A **widget** defines the *HTML*: which input element renders and its attributes. Every field has a sensible default widget, but you can override it.

| Field | Python type after cleaning | Default widget |
|-------|----------------------------|----------------|
| `CharField` | `str` | `TextInput` |
| `EmailField` | `str` (validated email) | `EmailInput` |
| `IntegerField` | `int` | `NumberInput` |
| `DecimalField` | `Decimal` | `NumberInput` |
| `BooleanField` | `bool` | `CheckboxInput` |
| `DateField` | `datetime.date` | `DateInput` |
| `DateTimeField` | `datetime.datetime` | `DateTimeInput` |
| `ChoiceField` | `str` (one of choices) | `Select` |
| `MultipleChoiceField` | `list[str]` | `SelectMultiple` |
| `FileField` | `UploadedFile` | `ClearableFileInput` |
| `ImageField` | `UploadedFile` (validated image) | `ClearableFileInput` |

Common field arguments: `required` (default `True`), `label`, `help_text`, `initial`, `min_length`/`max_length`, `min_value`/`max_value`, `validators`, `error_messages`, and `widget`.

```python
from django import forms

PLAN_CHOICES = [
    ("free", "Free"),
    ("pro", "Pro"),
    ("ent", "Enterprise"),
]

class SignupForm(forms.Form):
    username = forms.CharField(
        max_length=30,
        help_text="Letters and digits only.",
        widget=forms.TextInput(attrs={"class": "form-control", "placeholder": "jane_doe"}),
    )
    password = forms.CharField(widget=forms.PasswordInput)      # renders <input type="password">
    bio = forms.CharField(required=False, widget=forms.Textarea(attrs={"rows": 4}))
    plan = forms.ChoiceField(choices=PLAN_CHOICES, initial="free")
    starts_on = forms.DateField(widget=forms.DateInput(attrs={"type": "date"}))
```

The key idea: **the widget is purely presentational**. Swapping `TextInput` for `Textarea` changes the HTML, never the validation. Want a date-picker? Use `DateInput(attrs={"type": "date"})` — the field still cleans to a `datetime.date`.

## Core concept: binding data, `is_valid()`, and `cleaned_data`

A form is in one of two states:

- **Unbound** — created with no data: `ContactForm()`. Used to render a blank form. `form.is_valid()` is `False` and there is no `cleaned_data`.
- **Bound** — created *with* data: `ContactForm(request.POST)`. Used to validate and process submitted input.

Validation runs lazily the first time you call `is_valid()` (or access `errors`). After a successful validation, the typed, sanitized values live in `form.cleaned_data` — a plain `dict`.

```python
from django import forms

class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    age = forms.IntegerField(min_value=18)

# Bind raw request-style data (values arrive as strings, like request.POST)
form = ContactForm({"name": "  Jane  ", "age": "30"})

print(form.is_valid())          # => True
print(form.cleaned_data)        # => {'name': 'Jane', 'age': 30}
print(type(form.cleaned_data["age"]))   # => <class 'int'>   (coerced from "30")
# Note: CharField strips surrounding whitespace by default ("  Jane  " -> "Jane")

bad = ContactForm({"name": "", "age": "15"})
print(bad.is_valid())           # => False
print(bad.errors)               # => {'name': ['This field is required.'],
                                #     'age': ['Ensure this value is greater than or equal to 18.']}
print(bad.errors.as_json())     # machine-readable error structure
```

Three rules to internalize:

1. **Never trust `request.POST` directly** — read from `cleaned_data` after `is_valid()` returns `True`.
2. `cleaned_data` only contains fields that passed validation; a field that errored is omitted.
3. Strings come in, *typed Python objects* come out. `"30"` becomes `int(30)`, `"2026-06-19"` becomes a `date`.

## Core concept: custom validation — `clean_<field>` and `clean()`

Built-in rules cover length, type, ranges, and choices. For business rules you write two kinds of hooks.

### Field-level: `clean_<fieldname>()`

Runs for one field. It must **return** the cleaned value (return it even if unchanged) and raises `forms.ValidationError` to reject. By the time it runs, `self.cleaned_data[field]` already holds the type-coerced value.

```python
from django import forms

class SignupForm(forms.Form):
    username = forms.CharField(max_length=30)
    password = forms.CharField(widget=forms.PasswordInput)
    confirm = forms.CharField(widget=forms.PasswordInput)

    def clean_username(self):
        username = self.cleaned_data["username"]
        if not username.isalnum():
            raise forms.ValidationError("Username must be letters and digits only.")
        # Imagine a DB uniqueness check here, e.g. User.objects.filter(...).exists()
        return username.lower()     # MUST return the (possibly transformed) value
```

### Form-wide: `clean()`

Runs once after all field-level cleaning. Use it for rules that span *multiple* fields (e.g. "password and confirmation must match"). Call `super().clean()` first, then validate across fields.

```python
    def clean(self):
        cleaned = super().clean()
        password = cleaned.get("password")
        confirm = cleaned.get("confirm")
        if password and confirm and password != confirm:
            # Attach to a specific field...
            self.add_error("confirm", "Passwords do not match.")
            # ...or raise a non-field (form-wide) error:
            # raise forms.ValidationError("Passwords do not match.")
        return cleaned
```

Use `.get()` (not `["..."]`) inside `clean()` because a field that failed earlier won't be present. Errors raised in `clean()` without a field become **non-field errors**, accessible in templates via `form.non_field_errors`.

> Laravel parallel: `clean_<field>` is like a single rule in your `rules()` array or a custom `Rule` object; `clean()` is like the `withValidator()` / `after()` hook for cross-field checks.

## Core concept: rendering forms in templates and CSRF

In a view you pass the form into the template context, then render it. Django gives you several render styles.

```html
<!-- templates/contact.html -->
<form method="post" novalidate>
  {% csrf_token %}                {# REQUIRED for every POST form — like Laravel's @csrf #}

  {{ form.as_p }}                 {# quick: each field wrapped in <p> #}

  <button type="submit">Send</button>
</form>
```

Render styles: `{{ form.as_p }}` (paragraphs), `{{ form.as_div }}` (the modern default, accessible markup), `{{ form.as_ul }}`, `{{ form.as_table }}`. For full control, loop or render fields one by one:

```html
<form method="post" novalidate>
  {% csrf_token %}

  {% if form.non_field_errors %}
    <div class="errors">{{ form.non_field_errors }}</div>
  {% endif %}

  {% for field in form %}
    <div class="field">
      {{ field.label_tag }}
      {{ field }}                  {# the widget HTML #}
      {% if field.help_text %}<small>{{ field.help_text }}</small>{% endif %}
      {% for error in field.errors %}
        <span class="error">{{ error }}</span>
      {% endfor %}
    </div>
  {% endfor %}

  <button type="submit">Save</button>
</form>
```

Or address a field directly: `{{ form.email }}`, `{{ form.email.errors }}`, `{{ form.email.label_tag }}`.

**About CSRF.** Django rejects any POST without a valid CSRF token (HTTP 403). `{% csrf_token %}` injects a hidden `<input>` whose value is tied to the user's session. This is identical in spirit to Laravel's `@csrf` / `VerifyCsrfToken` middleware. You do *not* need it for GET forms. The `csrf` middleware is enabled by default in `MIDDLEWARE`.

**Why no `old('email')` like Laravel?** Because a *bound* form already holds the submitted values. When validation fails and you re-render the same bound form, every widget re-displays the user's last input automatically — repopulation is free.

## Core concept: the view — GET vs POST

This is the single most important pattern in Django forms. One view handles both showing the blank form (GET) and processing the submission (POST).

```python
# views.py
from django.shortcuts import render, redirect
from .forms import ContactForm

def contact(request):
    if request.method == "POST":
        form = ContactForm(request.POST)        # bound form
        if form.is_valid():
            data = form.cleaned_data
            # ... do the work: send email, save a row, etc. ...
            send_contact_email(data["email"], data["message"])
            return redirect("contact_thanks")   # POST/redirect/GET: avoid double-submit
        # invalid -> fall through and re-render with errors
    else:
        form = ContactForm()                    # unbound (blank) form on GET

    return render(request, "contact.html", {"form": form})
```

Why this shape:

- **GET** builds an unbound form for a fresh page.
- **POST** binds `request.POST`; on success you act then **redirect** (the POST/redirect/GET pattern prevents the "resubmit form?" dialog on refresh — same reason Laravel does `return redirect()->route(...)` after a store).
- On failure you re-`render` the *bound* form, which carries both the errors and the user's input.

## Core concept: `ModelForm` — forms derived from models

Most forms exist to create or edit a database row. `ModelForm` reads your model and generates matching fields automatically, including DB-backed validation (e.g. `max_length`, `unique`). This is the closest analogue to a Laravel Form Request paired with mass assignment.

```python
# models.py
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    body = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

```python
# forms.py
from django import forms
from .models import Article

class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "slug", "body", "published"]   # be explicit — preferred
        # exclude = ["created_at"]                          # alternative; avoid in practice
        widgets = {
            "body": forms.Textarea(attrs={"rows": 8}),
        }
        labels = {"slug": "URL slug"}
        help_texts = {"slug": "Lowercase, dashes, no spaces."}
```

`fields` vs `exclude`: **always prefer an explicit `fields` list** (or `fields = "__all__"` only for throwaway/admin code). `exclude` is a security footgun — add a column to the model later and it silently becomes editable, the same mass-assignment risk as a too-broad `$fillable` in Eloquent.

### Saving with `form.save()`

```python
# views.py
from django.shortcuts import render, redirect, get_object_or_404
from .forms import ArticleForm
from .models import Article

def article_create(request):
    if request.method == "POST":
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save()               # INSERT, returns the saved instance
            return redirect("article_detail", pk=article.pk)
    else:
        form = ArticleForm()
    return render(request, "article_form.html", {"form": form})

def article_edit(request, pk):
    article = get_object_or_404(Article, pk=pk)
    # instance= binds the form to an existing row -> save() does an UPDATE
    form = ArticleForm(request.POST or None, instance=article)
    if request.method == "POST" and form.is_valid():
        form.save()
        return redirect("article_detail", pk=article.pk)
    return render(request, "article_form.html", {"form": form})
```

Notes:

- `form.save()` inserts (no `instance`) or updates (with `instance=`) and returns the model object.
- `form.save(commit=False)` returns an unsaved instance so you can set extra fields (e.g. the owner) before hitting the DB:

```python
if form.is_valid():
    article = form.save(commit=False)
    article.author = request.user           # set a field the form didn't expose
    article.save()                          # now write the row
    form.save_m2m()                         # needed after commit=False IF the model has ManyToMany fields
```

- You can still add `clean_<field>`/`clean()` methods to a `ModelForm`; they layer on top of the model-derived rules.
- `request.POST or None` is a handy idiom: on GET it's an empty `QueryDict` (falsy) so the form is unbound; on POST it binds the data.

## Core concept: formsets and model formsets (brief)

A **formset** manages several copies of the same form on one page — think "add 3 phone numbers" or an editable table. A **model formset** does the same backed by model rows (bulk create/edit).

```python
from django import forms
from django.forms import formset_factory, modelformset_factory
from .models import Article

class PhoneForm(forms.Form):
    number = forms.CharField(max_length=20)

# A formset of up to "extra" blank PhoneForms
PhoneFormSet = formset_factory(PhoneForm, extra=3)

# A model formset bound to existing Article rows
ArticleFormSet = modelformset_factory(Article, fields=["title", "published"], extra=0)
```

```python
# view
def edit_articles(request):
    if request.method == "POST":
        formset = ArticleFormSet(request.POST)
        if formset.is_valid():
            formset.save()                    # bulk INSERT/UPDATE
            return redirect("articles")
    else:
        formset = ArticleFormSet(queryset=Article.objects.all())
    return render(request, "articles_edit.html", {"formset": formset})
```

```html
<form method="post">
  {% csrf_token %}
  {{ formset.management_form }}   {# REQUIRED: hidden bookkeeping fields, do not omit #}
  {% for form in formset %}
    {{ form.as_div }}
  {% endfor %}
  <button type="submit">Save all</button>
</form>
```

The one gotcha to remember: **always render `{{ formset.management_form }}`** — it carries counts (`TOTAL_FORMS`, `INITIAL_FORMS`) Django needs to reassemble the submission; omitting it raises a `ManagementForm data is missing` error.

## Core concept: file and image uploads

Three things must line up for uploads to work:

1. The HTML form needs `enctype="multipart/form-data"`.
2. The view must pass **`request.FILES`** as the form's second positional argument (files do not arrive in `request.POST`).
3. Use a `FileField` or `ImageField` (the latter validates it's a real image and requires the **Pillow** library: `pip install Pillow`).

```python
# forms.py
from django import forms

class UploadForm(forms.Form):
    title = forms.CharField(max_length=100)
    document = forms.FileField()
    avatar = forms.ImageField(required=False)
```

```python
# views.py
from django.shortcuts import render
from django.core.files.storage import default_storage
from .forms import UploadForm

def upload(request):
    if request.method == "POST":
        form = UploadForm(request.POST, request.FILES)    # NOTE: request.FILES is required
        if form.is_valid():
            f = form.cleaned_data["document"]
            print(f.name, f.size, f.content_type)         # => report.pdf 20481 application/pdf
            # Save the uploaded file to MEDIA_ROOT
            saved_path = default_storage.save(f"uploads/{f.name}", f)
            return render(request, "upload_ok.html", {"path": saved_path})
    else:
        form = UploadForm()
    return render(request, "upload.html", {"form": form})
```

```html
<!-- templates/upload.html -->
<form method="post" enctype="multipart/form-data">   <!-- without this, files won't be sent -->
  {% csrf_token %}
  {{ form.as_div }}
  <button type="submit">Upload</button>
</form>
```

With a `ModelForm` whose model has a `FileField`/`ImageField`, `form.save()` stores the file under `MEDIA_ROOT` and saves the path in the column — no manual `default_storage.save()` needed.

```python
# settings.py — needed for uploads to land somewhere and be served in dev
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

> Laravel parallel: `request.FILES["document"]` ≈ `$request->file('document')`; `default_storage.save(...)` ≈ `$request->file('document')->store('uploads')`; `MEDIA_ROOT` ≈ the `storage/app` disk.

## Common mistakes & gotchas

- **Reading `request.POST` instead of `cleaned_data`.** Raw POST values are unvalidated strings. Always use `form.cleaned_data` after `is_valid()`.
- **Forgetting `{% csrf_token %}`** in a POST form → 403 Forbidden. Every POST form needs it.
- **Forgetting `enctype="multipart/form-data"`** on upload forms → files silently never arrive.
- **Forgetting to pass `request.FILES`** to the form → `FileField` reports "This field is required" even though the user picked a file.
- **`clean_<field>` that doesn't `return`** the value → the field becomes `None` in `cleaned_data`. Always return.
- **Indexing `cleaned_data["x"]` inside `clean()`** for a field that may have failed → `KeyError`. Use `.get()`.
- **Using `exclude` on a `ModelForm`** → new model columns become editable later (mass-assignment risk). Prefer an explicit `fields` list.
- **Omitting `{{ formset.management_form }}`** → `ManagementForm data is missing or has been tampered with`.
- **Calling `form.save()` without checking `is_valid()` first** → raises `ValueError`. Validate, then save.
- **`commit=False` with ManyToMany fields** but forgetting `form.save_m2m()` → the relations never get written.
- **Expecting client-side validation to be enough** — add `novalidate` if you want to test server-side rules; never rely on the browser's HTML5 validation for security.

## Best practices

- Keep forms in a dedicated `forms.py` per app; keep views thin (validate → act → redirect).
- Prefer `ModelForm` whenever the form maps to a model; only hand-write a `Form` for non-model input (search, contact, login).
- Always list `fields` explicitly on a `ModelForm`.
- Put business validation in `clean_<field>`/`clean()`, not in the view. The form is the single source of truth for "what is valid."
- Follow POST/redirect/GET on success to prevent duplicate submissions.
- Render with `{{ form.as_div }}` (modern, accessible) unless you need custom per-field markup.
- Use `form.save(commit=False)` to inject server-controlled fields (owner, timestamps) the user shouldn't set.
- Validate file uploads' size/type in `clean_<field>`; use `ImageField` (+ Pillow) when you require an image.
- Set custom `error_messages` per field for user-friendly, localizable text.
- Reuse and compose: subclass forms or share validators across forms via the `validators=[...]` argument.

## Interview Q&A

**Q: What are the three responsibilities of Django's forms layer?**
**A:** Validation (coercing and checking raw input, producing typed `cleaned_data` and per-field errors), rendering (generating HTML widgets and re-populating them on error), and security (output escaping plus CSRF integration). It bundles what Laravel splits across Form Requests and Blade form partials.

**Q: What's the difference between a bound and an unbound form?**
**A:** An unbound form is created with no data (`MyForm()`) — used to render a blank page; `is_valid()` is `False` and there's no `cleaned_data`. A bound form is created with data (`MyForm(request.POST)`) — it can be validated, and after `is_valid()` returns `True` its typed values are in `cleaned_data`.

**Q: Where do validated values live, and why not just read `request.POST`?**
**A:** In `form.cleaned_data`, a dict populated only after `is_valid()` passes. `request.POST` holds raw, unvalidated strings; `cleaned_data` holds type-coerced, sanitized Python objects (e.g. `"30"` → `int`, `"2026-06-19"` → `date`) and excludes fields that failed validation.

**Q: When do you use `clean_<field>()` vs `clean()`?**
**A:** `clean_<field>()` validates or transforms a single field and must return its value. `clean()` runs once after all fields are cleaned and is for cross-field rules (e.g. password == confirm). In `clean()` use `cleaned_data.get(...)` because failed fields are absent, and use `add_error()` or raise a `ValidationError` for non-field errors.

**Q: What is the difference between a field and a widget?**
**A:** A field defines the data and validation (type, required, ranges); a widget defines only the HTML rendered (`TextInput`, `Textarea`, `Select`, `PasswordInput`). Changing the widget never changes validation — e.g. a `DateField` with a `DateInput(attrs={"type": "date"})` widget still cleans to a `datetime.date`.

**Q: What does `ModelForm` give you over a plain `Form`?**
**A:** It auto-generates fields from a model, inherits model-level constraints (max length, `unique`, choices), and provides `save()` to insert or update a row. You declare `class Meta` with `model` and `fields`. It's the Django equivalent of a Form Request whose rules come from the model schema, plus mass-assignment-style persistence.

**Q: Explain `form.save(commit=False)`.**
**A:** It builds the model instance from validated data but does *not* hit the database, so you can set server-controlled fields (like `author = request.user`) before calling `instance.save()`. If the model has ManyToMany fields you must also call `form.save_m2m()` afterward, because those relations are saved separately.

**Q: Why prefer `fields` over `exclude` in a ModelForm's Meta?**
**A:** `exclude` lists what *not* to include, so any column added to the model later automatically becomes editable — a mass-assignment vulnerability. An explicit `fields` list is allow-list by default and changes only when you say so, mirroring the reasoning behind a tight Eloquent `$fillable`.

**Q: How does Django handle CSRF in forms?**
**A:** The CSRF middleware rejects POSTs lacking a valid token (403). You include `{% csrf_token %}` in every POST form, which renders a session-tied hidden input. It's the direct analogue of Laravel's `@csrf` and `VerifyCsrfToken` middleware; GET forms don't need it.

**Q: What's required to make file uploads work?**
**A:** Three things: `enctype="multipart/form-data"` on the `<form>`, passing `request.FILES` as the form's second argument (`MyForm(request.POST, request.FILES)`), and a `FileField`/`ImageField` on the form. `ImageField` additionally requires Pillow and validates the file is a real image.

**Q: How does a single view handle both showing and processing a form?**
**A:** Branch on `request.method`: on POST, bind `request.POST` (and `request.FILES`), call `is_valid()`, act, then redirect (POST/redirect/GET); on GET, create an unbound form. On validation failure you re-render the bound form, which carries both errors and the user's input so repopulation is automatic.

**Q: What is a formset and what's the one thing you must not forget when rendering it?**
**A:** A formset manages multiple instances of the same form on one page (created via `formset_factory` / `modelformset_factory`). You must always render `{{ formset.management_form }}`, the hidden bookkeeping fields (`TOTAL_FORMS`, `INITIAL_FORMS`); omitting it raises a `ManagementForm data is missing` error on submit.

## Practice exercises

1. Write a `forms.Form` named `FeedbackForm` with `name` (required, max 80), `email` (required), `rating` (`IntegerField`, 1–5), and `comment` (optional textarea). Bind it to `{"name": "Sam", "email": "sam@x.com", "rating": "7", "comment": ""}` and print whether it's valid plus the errors.

2. Add a `clean_email()` to `FeedbackForm` that rejects any address ending in `@example.com` with a friendly message, and lowercases the address otherwise.

3. Add a form-wide `clean()` rule: if `rating` is 1 or 2, `comment` becomes required ("Please tell us what went wrong."). Make sure it doesn't crash when `rating` failed validation.

4. Given a `Product` model with `name`, `price` (`DecimalField`), `sku` (`unique`), and `in_stock` (`BooleanField`), write a `ProductForm(ModelForm)` exposing only `name`, `price`, `sku`, `in_stock`, and write the create view following GET/POST + redirect.

5. Write an upload form + view that accepts a required `FileField` named `resume`, rejects files larger than 2 MB in `clean_resume()`, and on success prints the file name and size. Include the correct template `<form>` opening tag.

6. (Stretch) Build a `formset_factory` of a `LinkForm` (`label`, `url`) with `extra=2`, and a view that prints every valid link as `"label -> url"`.

<details><summary>Solutions</summary>

```python
# 1. FeedbackForm + binding
from django import forms

class FeedbackForm(forms.Form):
    name = forms.CharField(max_length=80)
    email = forms.EmailField()
    rating = forms.IntegerField(min_value=1, max_value=5)
    comment = forms.CharField(required=False, widget=forms.Textarea)

form = FeedbackForm({"name": "Sam", "email": "sam@x.com", "rating": "7", "comment": ""})
print(form.is_valid())   # => False  (rating 7 > 5)
print(form.errors)       # => {'rating': ['Ensure this value is less than or equal to 5.']}
```

```python
# 2. clean_email
    def clean_email(self):
        email = self.cleaned_data["email"]
        if email.lower().endswith("@example.com"):
            raise forms.ValidationError("Please use a real email address.")
        return email.lower()
```

```python
# 3. cross-field clean()
    def clean(self):
        cleaned = super().clean()
        rating = cleaned.get("rating")       # .get() — rating may have failed
        comment = cleaned.get("comment")
        if rating in (1, 2) and not comment:
            self.add_error("comment", "Please tell us what went wrong.")
        return cleaned
```

```python
# 4. ProductForm + view
# forms.py
from django import forms
from .models import Product

class ProductForm(forms.ModelForm):
    class Meta:
        model = Product
        fields = ["name", "price", "sku", "in_stock"]

# views.py
from django.shortcuts import render, redirect
from .forms import ProductForm

def product_create(request):
    if request.method == "POST":
        form = ProductForm(request.POST)
        if form.is_valid():
            product = form.save()
            return redirect("product_detail", pk=product.pk)
    else:
        form = ProductForm()
    return render(request, "product_form.html", {"form": form})
```

```python
# 5. Upload form with size check
# forms.py
from django import forms

MAX_SIZE = 2 * 1024 * 1024   # 2 MB

class ResumeForm(forms.Form):
    resume = forms.FileField()

    def clean_resume(self):
        f = self.cleaned_data["resume"]
        if f.size > MAX_SIZE:
            raise forms.ValidationError("File must be 2 MB or smaller.")
        return f

# views.py
from django.shortcuts import render
from .forms import ResumeForm

def upload_resume(request):
    if request.method == "POST":
        form = ResumeForm(request.POST, request.FILES)   # request.FILES is essential
        if form.is_valid():
            f = form.cleaned_data["resume"]
            print(f.name, f.size)                         # => resume.pdf 18432
    else:
        form = ResumeForm()
    return render(request, "upload.html", {"form": form})
```

```html
<!-- 5. template opening tag -->
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  {{ form.as_div }}
  <button type="submit">Upload</button>
</form>
```

```python
# 6. formset of LinkForm
from django import forms
from django.forms import formset_factory

class LinkForm(forms.Form):
    label = forms.CharField(max_length=50)
    url = forms.URLField()

LinkFormSet = formset_factory(LinkForm, extra=2)

# views.py
from django.shortcuts import render
from .forms import LinkFormSet

def add_links(request):
    if request.method == "POST":
        formset = LinkFormSet(request.POST)
        if formset.is_valid():
            for f in formset:
                cd = f.cleaned_data
                if cd:                                  # skip blank extra forms
                    print(f"{cd['label']} -> {cd['url']}")
    else:
        formset = LinkFormSet()
    return render(request, "links.html", {"formset": formset})
```

</details>

## Cheat sheet

```python
# Declare
class MyForm(forms.Form):
    name  = forms.CharField(max_length=100)
    email = forms.EmailField(required=False)
    age   = forms.IntegerField(min_value=18)

# Bound vs unbound
MyForm()                      # unbound (blank)
MyForm(request.POST)          # bound (validate)
MyForm(request.POST, request.FILES)   # bound + uploads

# Validate & read
if form.is_valid():
    data = form.cleaned_data  # typed dict
else:
    form.errors               # {field: [msgs]}, form.non_field_errors

# Custom validation
def clean_email(self):        # one field — MUST return value
    return self.cleaned_data["email"].lower()
def clean(self):              # cross-field — use .get()
    cd = super().clean(); ...; return cd

# ModelForm
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "body"]   # prefer explicit list over exclude
obj = form.save()                    # insert / update
obj = form.save(commit=False)        # set extra fields, then obj.save(); form.save_m2m()
ArticleForm(request.POST, instance=existing)   # edit

# Formsets
FS = formset_factory(MyForm, extra=3)
FS = modelformset_factory(Article, fields=["title"], extra=0)
```

| Need | Template snippet |
|------|------------------|
| CSRF (every POST form) | `{% csrf_token %}` |
| Quick render | `{{ form.as_div }}` |
| Whole field | `{{ form.email }}` |
| Field errors | `{{ form.email.errors }}` |
| Form-wide errors | `{{ form.non_field_errors }}` |
| Loop fields | `{% for f in form %}{{ f.label_tag }}{{ f }}{% endfor %}` |
| Formset bookkeeping | `{{ formset.management_form }}` |
| Upload form tag | `<form method="post" enctype="multipart/form-data">` |

| Laravel | Django |
|---------|--------|
| `$request->validate([...])` | `form.is_valid()` + field rules |
| Form Request `rules()` | `Form` fields + `clean_*`/`clean` |
| `@csrf` | `{% csrf_token %}` |
| `old('email')` | bound form re-populates automatically |
| `$errors->first('email')` | `{{ form.email.errors }}` |
| `$request->file('x')` | `request.FILES["x"]` |
| Form Request from model + `$fillable` | `ModelForm` + `Meta.fields` |
