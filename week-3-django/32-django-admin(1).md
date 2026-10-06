# 32 — The Admin Site

> Week 3 · Django

## What you will learn

- Why the Django admin is often called Django's "killer feature" and what you get for free
- How to enable the admin, run migrations, and create a superuser
- Two ways to register models: `admin.site.register()` and the `@admin.register()` decorator
- Customizing the change list and forms with `ModelAdmin`: `list_display`, `list_filter`, `search_fields`, `ordering`, `list_editable`, `readonly_fields`, `fields`/`fieldsets`, `prepopulated_fields`, `date_hierarchy`
- Editing related rows on the same page with `TabularInline` and `StackedInline`
- Writing custom admin **actions** that operate on selected rows
- Overriding `get_queryset()` and `save_model()` to control what staff see and what happens on save
- How the admin ties into Django's permission system
- When the admin is the right tool, when to build something custom, and the security rules that keep it safe

## PHP/Laravel bridge

In Laravel you reach for a package to get a back-office UI: **Laravel Nova** (paid), **Filament**, or **Backpack**. You install it, generate "resources," and wire them to your Eloquent models. Django's admin is that same idea but **built into the framework and free** — no extra package, no licence. A Django `ModelAdmin` class is the direct equivalent of a Filament/Nova *Resource*: it declares which columns show in the table, which filters appear in the sidebar, which fields are editable, and so on. The big mental shift: in Laravel the admin panel is an add-on you opt into; in Django it has been core since 2005 and shapes how Django apps are built (models first, admin almost for free).

## Core concept: what the admin gives you for free

Point the admin at a model and you immediately get a **full CRUD interface**: a searchable, filterable, paginated list of rows ("change list"), an add form, an edit form, delete with confirmation, and an audit log of who changed what. It reads your model's field definitions to pick sensible widgets — a `DateField` gets a date picker, a `ForeignKey` gets a dropdown, a `BooleanField` gets a checkbox, `choices` become a select box. Validation from your model and forms is enforced. Permissions are wired in per-model and per-action.

This is the headline feature because it means a content team can start managing real data on **day one**, before you have written a single template or API endpoint. It is generated automatically from the models you already wrote — you describe data once and get an editing UI for it.

The crucial caveat, repeated throughout this lesson: the admin is for **trusted internal staff**, not your end users. It is a powerful, schema-aware control panel. Treat it like database access, because that is effectively what it is.

## Core concept: enabling the admin and creating a superuser

The admin ships enabled in a fresh `django-admin startproject`. The relevant pieces are already in `settings.py` and `urls.py`:

```python
# settings.py — these apps must be present (they are by default)
INSTALLED_APPS = [
    "django.contrib.admin",         # the admin itself
    "django.contrib.auth",          # users, groups, permissions
    "django.contrib.contenttypes",  # admin/permissions depend on this
    "django.contrib.sessions",      # admin login uses sessions
    "django.contrib.messages",      # the "Changes saved" banners
    "django.contrib.staticfiles",   # serves the admin's CSS/JS
    # ... your apps
    "shop",
]
```

```python
# urls.py
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path("admin/", admin.site.urls),   # admin lives under /admin/
]
```

Before logging in you must run migrations (the admin's log and auth tables must exist) and create an account that can log in:

```bash
python manage.py migrate
python manage.py createsuperuser
# Username: hassan
# Email address: hassan@example.com
# Password: ********
# Superuser created successfully.
```

A **superuser** is a staff user with every permission. Now run the server and visit the admin:

```bash
python manage.py runserver
# Visit http://127.0.0.1:8000/admin/ and log in
```

Two flags on the user model decide admin access:

- `is_staff=True` — required just to *see* the admin login and reach it at all.
- `is_superuser=True` — bypasses every permission check; sees and edits everything.

`createsuperuser` sets both to `True`. A normal staff member gets `is_staff=True` plus a curated set of permissions (more on that later).

## Core concept: registering models

Out of the box the admin shows only Users and Groups. To manage your own models you **register** them. Say we have these models:

```python
# shop/models.py
from django.db import models
from django.utils.text import slugify


class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(max_length=120, unique=True)

    class Meta:
        verbose_name_plural = "categories"  # otherwise admin shows "Categorys"

    def __str__(self):
        return self.name


class Product(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"
        ARCHIVED = "archived", "Archived"

    name = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    category = models.ForeignKey(
        Category, on_delete=models.CASCADE, related_name="products"
    )
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.PositiveIntegerField(default=0)
    status = models.CharField(
        max_length=10, choices=Status.choices, default=Status.DRAFT
    )
    created_at = models.DateTimeField(auto_now_add=True)
    created_by = models.ForeignKey(
        "auth.User", on_delete=models.SET_NULL, null=True, blank=True
    )

    def __str__(self):
        return self.name
```

> Tip: always define `__str__`. The admin uses it for the label of each row and in dropdowns. Without it you get useless `Product object (1)` labels.

### Way 1 — `admin.site.register()`

```python
# shop/admin.py
from django.contrib import admin
from .models import Category, Product

admin.site.register(Category)
admin.site.register(Product)
```

That single line per model already gives you full CRUD. To customize, pass a `ModelAdmin` subclass as the second argument:

```python
# shop/admin.py
from django.contrib import admin
from .models import Category, Product


class ProductAdmin(admin.ModelAdmin):
    list_display = ("name", "price", "status")


admin.site.register(Category)
admin.site.register(Product, ProductAdmin)
```

### Way 2 — the `@admin.register()` decorator (preferred)

The decorator does the same thing but keeps the `ModelAdmin` and its registration together — cleaner and the modern style:

```python
# shop/admin.py
from django.contrib import admin
from .models import Category, Product


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ("name", "slug")


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = ("name", "price", "status")
```

You can register several models with one decorator: `@admin.register(Product, ProductVariant)`. Both forms are equivalent at runtime; pick the decorator for new code. **The file must be named `admin.py` inside the app** — Django's autodiscover imports it automatically when the server starts.

## Core concept: customizing with `ModelAdmin`

`ModelAdmin` is where almost all admin tuning lives. Here is a heavily annotated example exercising the options you will use day to day:

```python
# shop/admin.py
from django.contrib import admin
from .models import Category, Product


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    # --- The change list (the table view) ---

    # Columns shown in the table. Each entry can be a field name,
    # a ModelAdmin method, or a model method.
    list_display = ("name", "category", "price", "stock", "status", "is_in_stock")

    # Right-hand sidebar filters. Works on bool, FK, choices, dates.
    list_filter = ("status", "category", "created_at")

    # Adds a search box. "category__name" follows the FK relationship.
    search_fields = ("name", "slug", "category__name")

    # Default sort for the list (prefix with "-" for descending).
    ordering = ("-created_at",)

    # Edit these columns inline in the table without opening each row.
    # The first column in list_display CANNOT be list_editable; it is
    # the link to the change form, so put an editable field after it.
    list_editable = ("price", "status")

    # A drill-down date navigation bar at the top of the list.
    date_hierarchy = "created_at"

    # How many rows per page (default 100).
    list_per_page = 25

    # --- The add/change form ---

    # Auto-fill slug from name as you type (JS in the browser).
    prepopulated_fields = {"slug": ("name",)}

    # Shown but never editable in the form.
    readonly_fields = ("created_at",)

    # A computed column. boolean=True renders a green/red icon.
    @admin.display(boolean=True, description="In stock?")
    def is_in_stock(self, obj):
        return obj.stock > 0
```

### `list_display` — the table columns

Entries are tried in this order: a `ModelAdmin` method, a model method/attribute, then a model field. Methods receive the row object so you can compute or format values. The `@admin.display` decorator sets the column header (`description`) and, with `boolean=True`, renders a tick/cross icon. Use `ordering=` inside it to make a computed column sortable by a real DB expression:

```python
@admin.display(description="Price (with tax)", ordering="price")
def price_inc_tax(self, obj):
    return round(obj.price * 1.2, 2)
```

> Note: a plain Python method column is **not** sortable unless you give it an `ordering` that maps to something the database can sort by.

### `list_filter` — sidebar filters

Pass field names and Django builds the right widget automatically: `status` (choices) becomes a list of statuses, `category` (FK) lists categories, `created_at` (date) becomes "Today / Past 7 days / This month…". You can also pass a custom `SimpleListFilter` subclass for bespoke filters, but field names cover most needs.

### `search_fields` — the search box

A space-separated query matches rows where **any** listed field contains **all** the terms. Use `__` to span relations (`category__name`). Prefix tricks control matching: `^name` = starts-with, `=name` = exact, `@name` = full-text (on supported backends). Keep the list lean — every field is an `OR` of `LIKE` queries and can get slow on large tables without indexes.

### `fields` vs `fieldsets` — laying out the form

By default the form shows every editable field in model order. Use `fields` to choose and order a subset, including putting two fields on one line with a tuple:

```python
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    fields = ("name", "slug", "category", ("price", "stock"), "status")
```

For larger forms, `fieldsets` groups fields into titled sections with optional CSS classes (e.g. `"collapse"` to start collapsed). You use **either** `fields` **or** `fieldsets`, not both:

```python
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    prepopulated_fields = {"slug": ("name",)}
    readonly_fields = ("created_at", "created_by")
    fieldsets = (
        ("Basics", {
            "fields": ("name", "slug", "category"),
        }),
        ("Pricing & stock", {
            "fields": (("price", "stock"), "status"),
            "description": "Set status to Published to make it live.",
        }),
        ("Audit", {
            "classes": ("collapse",),           # collapsed by default
            "fields": ("created_at", "created_by"),
        }),
    )
```

### `prepopulated_fields` — auto-slugging

`{"slug": ("name",)}` wires browser JavaScript so the slug field fills in (and slugifies) as the user types the name. It only runs on the add form and only while the slug is untouched, which is exactly what you want for SEO slugs. Note: `prepopulated_fields` does not work on fields that are also in `readonly_fields`.

### `readonly_fields` — show but do not edit

Great for `created_at`, computed values, or audit data. A read-only entry can also be a method, letting you render a formatted, non-editable block on the form. Read-only fields are displayed but never submitted, so they cannot be tampered with from the form.

### `date_hierarchy` — drill-down by date

Set it to a `DateField`/`DateTimeField` name and the admin adds a year → month → day navigation bar above the list. Use exactly one, on an indexed date column ideally, since it issues date-truncation queries.

## Core concept: inlines — editing related rows together

A real win of the admin: edit a parent and its children **on the same page**. Suppose products have image rows:

```python
# shop/models.py
class ProductImage(models.Model):
    product = models.ForeignKey(
        Product, on_delete=models.CASCADE, related_name="images"
    )
    image = models.ImageField(upload_to="products/")
    alt_text = models.CharField(max_length=200, blank=True)
    is_primary = models.BooleanField(default=False)
```

Declare an inline and attach it to the parent's `ModelAdmin`:

```python
# shop/admin.py
from django.contrib import admin
from .models import Product, ProductImage


class ProductImageInline(admin.TabularInline):   # rows in a compact table
    model = ProductImage
    extra = 1               # number of blank rows to show for adding
    fields = ("image", "alt_text", "is_primary")
    # readonly_fields, max_num, min_num, can_delete also supported


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    inlines = (ProductImageInline,)
```

- `TabularInline` lays children out as a **compact table** (one row per child) — best for many small rows.
- `StackedInline` lays each child out as a **full stacked form** — best for a few rich rows.

```python
class ProductImageInline(admin.StackedInline):   # each child gets a full form
    model = ProductImage
    extra = 0
```

Switching style is a one-word change. Inlines require a `ForeignKey` from child to parent (or a `ManyToMany` via a through model). Useful knobs: `extra` (blank rows), `max_num`/`min_num` (limits), `can_delete`, and `show_change_link=True` to add a link to each child's own change page.

## Core concept: custom admin actions

Actions are bulk operations applied to the rows you tick in the change list, chosen from the "Action" dropdown. Django ships one (`delete_selected`). You add your own as methods decorated with `@admin.action`:

```python
# shop/admin.py
from django.contrib import admin
from .models import Product


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = ("name", "status", "stock")
    actions = ("make_published", "make_archived", "zero_out_stock")

    @admin.action(description="Mark selected products as Published")
    def make_published(self, request, queryset):
        updated = queryset.update(status=Product.Status.PUBLISHED)
        # Flash a success banner to the user.
        self.message_user(
            request,
            f"{updated} product(s) published.",
            level="success",
        )

    @admin.action(description="Mark selected products as Archived")
    def make_archived(self, request, queryset):
        updated = queryset.update(status=Product.Status.ARCHIVED)
        self.message_user(request, f"{updated} product(s) archived.")

    @admin.action(
        description="Zero out stock",
        permissions=("change",),   # only users with change perm see this
    )
    def zero_out_stock(self, request, queryset):
        updated = queryset.update(stock=0)
        self.message_user(request, f"Reset stock on {updated} product(s).")
```

An action method takes `(self, request, queryset)` where `queryset` is exactly the selected rows. Returning nothing redraws the list; returning an `HttpResponse` (e.g. a CSV download) sends that instead — a common pattern for "Export selected to CSV":

```python
import csv
from django.http import HttpResponse


@admin.action(description="Export selected to CSV")
def export_csv(self, request, queryset):
    response = HttpResponse(content_type="text/csv")
    response["Content-Disposition"] = "attachment; filename=products.csv"
    writer = csv.writer(response)
    writer.writerow(["Name", "Price", "Status"])
    for product in queryset:
        writer.writerow([product.name, product.price, product.status])
    return response
```

> `queryset.update()` is a single bulk SQL `UPDATE` — fast, but it **does not** call each model's `save()` or fire signals. If you need per-row `save()` logic, loop and call `obj.save()` instead.

## Core concept: overriding querysets and save behavior

`ModelAdmin` methods are hooks you can override to change what the admin does. Two of the most useful:

### `get_queryset()` — scope what staff can see

By default the admin lists every row. Override `get_queryset()` to restrict it — for example, so non-superusers only manage their own products, and to add a select_related for performance:

```python
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = ("name", "category", "created_by")

    def get_queryset(self, request):
        qs = super().get_queryset(request)
        qs = qs.select_related("category", "created_by")  # avoid N+1 in list
        if request.user.is_superuser:
            return qs
        return qs.filter(created_by=request.user)
```

`select_related` here is important: `list_display` showing `category` and `created_by` would otherwise run one extra query per row (the classic N+1). The admin gives you the `request`, so you can branch on the logged-in user.

### `save_model()` — stamp data on save

`save_model(request, obj, form, change)` runs when a row is saved from the admin form. `change` is `False` on add and `True` on edit. Use it to fill server-side fields the form should not expose, such as the author:

```python
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    readonly_fields = ("created_by",)

    def save_model(self, request, obj, form, change):
        if not change:                 # only on first create
            obj.created_by = request.user
        super().save_model(request, obj, form, change)
```

Related hooks worth knowing: `save_formset()` (for inline rows), `has_add_permission` / `has_change_permission` / `has_delete_permission` / `has_view_permission` (return `True`/`False` to grant or deny, optionally per-object), and `get_readonly_fields(request, obj)` to make fields read-only conditionally (e.g. lock a field after creation).

## Core concept: admin permissions tie-in

The admin is built directly on Django's auth permission system. For every model Django auto-creates four permissions:

| Permission codename     | Lets the user…                       |
| ----------------------- | ------------------------------------ |
| `app.add_product`       | use the Add form                     |
| `app.change_product`    | open and save the change form        |
| `app.delete_product`    | delete rows                          |
| `app.view_product`      | see the list and read-only forms     |

How access actually resolves in the admin:

- The user must have `is_staff=True` to reach the admin at all.
- A **superuser** passes every check automatically.
- A normal staff user sees a model in the admin only if they hold at least one of its permissions. With only `view_*` they get **read-only** access (list and forms, no save) — handy for analysts.
- Assign permissions individually or, better, bundle them into **Groups** (e.g. an "Editors" group with add/change/view on `Product`). Groups are Django's answer to roles, the same idea as Laravel's roles via spatie/laravel-permission.

Custom actions can gate themselves with the `permissions=(...)` argument seen earlier, and you can override the `has_*_permission` hooks for fine-grained or per-object rules.

## Core concept: when to use the admin vs build custom

The admin is fantastic for **internal, trusted, CRUD-shaped** work, and a poor fit as a customer-facing UI. Use this rule of thumb:

| Use the Django admin when…                          | Build a custom UI/endpoint when…                          |
| --------------------------------------------------- | --------------------------------------------------------- |
| Internal staff manage content/data                  | The audience is end users / the public                    |
| The workflow maps cleanly to model CRUD             | You need bespoke multi-step flows or business screens     |
| You need it working today with near-zero code       | You need full control of UX, branding, and layout         |
| Moderating, data entry, quick fixes, support        | High traffic, public sign-up, or complex client logic     |
| Quick reporting via filters and search              | Rich dashboards, charts, or mobile apps (build an API)    |

For your target role this is the key framing: the admin is your **back office**; your REST API (DRF/FastAPI) is the **product**. They coexist — admin for staff, API for the world. Trying to bend the admin into a public-facing app is a classic anti-pattern.

## Common mistakes & gotchas

- **No `__str__` on the model.** Lists and FK dropdowns show `Product object (1)`. Always define `__str__`.
- **Forgetting to register.** A model with no `admin.py` entry simply will not appear — there is no error.
- **`admin.py` in the wrong place or app not in `INSTALLED_APPS`.** Autodiscover only imports `admin.py` from installed apps.
- **First column in `list_editable`.** It clashes with the link column and raises `ImproperlyConfigured`. Put editable fields after the first column, and every `list_editable` field must also be in `list_display`.
- **N+1 queries in `list_display`.** Showing FK columns without `select_related`/`prefetch_related` in `get_queryset()` quietly fires a query per row.
- **`prepopulated_fields` on a `readonly_fields` field.** It silently will not work; the field must be editable.
- **Using `fields` and `fieldsets` together.** Choose one; together they conflict.
- **Expecting `queryset.update()` in an action to call `save()`.** Bulk update skips `save()` and signals. Loop if you need them.
- **`verbose_name_plural` typos.** Without it, Django pluralizes naively ("Categorys"); set it on `Meta`.
- **Treating the admin as a public app.** It is staff-only by design — never expose it to untrusted users or unauthenticated traffic.

## Best practices

- Prefer the `@admin.register(Model)` decorator; keep one `ModelAdmin` per model.
- Always override `get_queryset()` with `select_related`/`prefetch_related` when `list_display` references related objects.
- Use `list_select_related = ("category",)` as a quick alternative for FK columns in the list.
- Keep `search_fields` short and index columns you search; the same for `date_hierarchy`.
- Use `list_editable` sparingly for cheap bulk tweaks (status, price); use forms for richer edits.
- Stamp audit fields (`created_by`, `updated_by`) in `save_model()`, not in the browser form.
- Model roles with **Groups**; grant the narrowest permissions that get the job done. Give analysts `view_*` only.
- Put the admin behind HTTPS, a strong password policy, and ideally a non-default URL plus IP allowlist or VPN. Consider two-factor (e.g. `django-otp`).
- Never run with `DEBUG=True` in production; never expose the admin to the public internet without access controls.

## Interview Q&A

**Q: What is the Django admin and why is it considered a flagship feature?**
**A:** It is a built-in, auto-generated CRUD interface for your models. From your model definitions Django produces searchable, filterable, paginated list views plus add/edit/delete forms with correct widgets and validation, gated by the permission system. It is a flagship feature because it gives teams a working back office on day one with almost no code — something most frameworks require a paid or third-party package to match.

**Q: How do you enable the admin and create an account to log in?**
**A:** `django.contrib.admin` and its dependencies are in `INSTALLED_APPS` and `admin.site.urls` is wired in `urls.py` by default. Run `python manage.py migrate`, then `python manage.py createsuperuser`, then `runserver` and visit `/admin/`.

**Q: What is the difference between `is_staff` and `is_superuser`?**
**A:** `is_staff=True` is required just to access the admin. `is_superuser=True` additionally bypasses every permission check, so a superuser can see and do everything. A normal staff member has `is_staff=True` plus a specific set of permissions or groups.

**Q: Two ways to register a model, and which is preferred?**
**A:** `admin.site.register(Model, ModelAdmin)` or the `@admin.register(Model)` class decorator. The decorator is preferred for new code because it keeps the `ModelAdmin` and its registration together.

**Q: How do you add columns, filters, and search to the change list?**
**A:** On the `ModelAdmin`: `list_display` for columns (field names or methods), `list_filter` for sidebar filters (auto-built for bool/FK/choices/date), and `search_fields` for the search box (supports `__` to span relations and `^`/`=` prefixes). `ordering` sets the default sort and `date_hierarchy` adds date drill-down.

**Q: What is the difference between `TabularInline` and `StackedInline`?**
**A:** Both let you edit related child rows on the parent's page. `TabularInline` renders children as a compact table (one row each) — good for many small rows. `StackedInline` renders each child as a full stacked form — good for a few rich rows. They are interchangeable in one line of code.

**Q: How do custom admin actions work?**
**A:** Define a method `(self, request, queryset)` on the `ModelAdmin`, decorate it with `@admin.action(description=...)`, and list it in `actions`. It receives the selected rows as a queryset; return nothing to redraw the list or return an `HttpResponse` (e.g. a CSV) to send a custom response. Note `queryset.update()` is bulk SQL and skips `save()`/signals.

**Q: How would you make the admin only show each user their own records?**
**A:** Override `get_queryset(self, request)`, call `super().get_queryset(request)`, and filter by the request user when they are not a superuser (e.g. `.filter(created_by=request.user)`). Add `select_related` there too, to avoid N+1 queries from `list_display`.

**Q: How do you set fields the form should not expose, like the author, on save?**
**A:** Override `save_model(self, request, obj, form, change)`, set `obj.created_by = request.user` (often only when `change is False`), and call `super().save_model(...)`. Keep the field in `readonly_fields` so it shows but cannot be edited.

**Q: How does the admin tie into permissions?**
**A:** Django auto-creates `add`, `change`, `delete`, and `view` permissions per model. The user needs `is_staff` plus the relevant permission; a `view`-only user gets read-only access. Permissions are usually bundled into Groups (roles). Superusers bypass all checks.

**Q: When should you NOT use the admin?**
**A:** For anything customer-facing or untrusted. It is a staff-only, schema-aware control panel — effectively database access. End-user features should go through your own views or REST API, with the admin reserved for internal CRUD and moderation.

**Q: How do you avoid N+1 queries in the admin list?**
**A:** Use `list_select_related` for FK columns, or override `get_queryset()` with `select_related()`/`prefetch_related()`. Without it, each related column in `list_display` fires an extra query per row.

## Practice exercises

1. **Register and tune.** Given the `Product` model above, register it with the `@admin.register` decorator so the list shows name, category, price, status; add filters for status and category; make name and category searchable; default-sort by newest first.
2. **Slugs and read-only audit.** Add `prepopulated_fields` to auto-fill `slug` from `name`, and make `created_at` read-only on the form.
3. **Inline.** Add a `ProductImage` model (FK to `Product`) and edit its rows inline on the product page using `TabularInline` with one blank row.
4. **Bulk action.** Add an admin action "Mark selected as Published" that bulk-updates `status` and flashes how many rows changed.
5. **Scope + stamp.** Override `get_queryset()` so non-superusers only see products they created, and `save_model()` so new products record `created_by = request.user`.
6. **Permissions (conceptual).** Describe how you would give a content team add/change/view (but not delete) on `Product` without making them superusers.

<details><summary>Solutions</summary>

```python
# shop/admin.py
import csv
from django.contrib import admin
from django.http import HttpResponse
from .models import Product, ProductImage


# Exercise 3: inline for related images
class ProductImageInline(admin.TabularInline):
    model = ProductImage
    extra = 1
    fields = ("image", "alt_text", "is_primary")


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    # Exercise 1: change-list tuning
    list_display = ("name", "category", "price", "status")
    list_filter = ("status", "category")
    search_fields = ("name", "category__name")
    ordering = ("-created_at",)

    # Exercise 2: slug + read-only audit field
    prepopulated_fields = {"slug": ("name",)}
    readonly_fields = ("created_at", "created_by")

    inlines = (ProductImageInline,)
    actions = ("make_published",)

    # Exercise 4: bulk action
    @admin.action(description="Mark selected as Published")
    def make_published(self, request, queryset):
        updated = queryset.update(status=Product.Status.PUBLISHED)
        self.message_user(request, f"{updated} product(s) published.")

    # Exercise 5a: scope rows per user (+ avoid N+1)
    def get_queryset(self, request):
        qs = super().get_queryset(request).select_related("category", "created_by")
        if request.user.is_superuser:
            return qs
        return qs.filter(created_by=request.user)

    # Exercise 5b: stamp author on create
    def save_model(self, request, obj, form, change):
        if not change:
            obj.created_by = request.user
        super().save_model(request, obj, form, change)
```

```python
# Exercise 3: the model the inline needs
# shop/models.py
class ProductImage(models.Model):
    product = models.ForeignKey(
        Product, on_delete=models.CASCADE, related_name="images"
    )
    image = models.ImageField(upload_to="products/")
    alt_text = models.CharField(max_length=200, blank=True)
    is_primary = models.BooleanField(default=False)
```

**Exercise 6 (permissions):** Create a Group, e.g. "Editors", in the admin under *Authentication and Authorization → Groups*. Grant it `shop | product | Can add product`, `Can change product`, and `Can view product` — but **not** `Can delete product`. Create staff users with `is_staff=True`, `is_superuser=False`, and add them to the Editors group. They will see Products in the admin, be able to add/edit/view, and have no delete action. This models a "content editor" role without granting database-wide power. The same can be done in code:

```python
from django.contrib.auth.models import Group, Permission

editors, _ = Group.objects.get_or_create(name="Editors")
codenames = ["add_product", "change_product", "view_product"]
perms = Permission.objects.filter(
    content_type__app_label="shop", codename__in=codenames
)
editors.permissions.set(perms)
```

</details>

## Cheat sheet

```python
# Register (preferred)
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    ...

# Or:  admin.site.register(Product, ProductAdmin)
```

| `ModelAdmin` option       | What it does                                              |
| ------------------------- | -------------------------------------------------------- |
| `list_display`            | Columns in the table (fields or methods)                 |
| `list_filter`             | Sidebar filters (bool / FK / choices / date)             |
| `search_fields`           | Search box; `rel__field`, `^`=startswith, `=`=exact      |
| `ordering`                | Default sort; `"-field"` = descending                    |
| `list_editable`           | Edit columns inline in the list (not the first column)   |
| `list_per_page`           | Rows per page                                            |
| `date_hierarchy`          | Date drill-down bar over the list                        |
| `readonly_fields`         | Show but never edit on the form                          |
| `fields` / `fieldsets`    | Choose/group form fields (use one, not both)             |
| `prepopulated_fields`     | Auto-fill slug from another field as you type            |
| `inlines`                 | Edit related rows on the parent page                     |
| `actions`                 | Bulk operations in the Action dropdown                   |
| `list_select_related`     | Quick N+1 fix for FK columns in the list                 |

```python
# Computed boolean column
@admin.display(boolean=True, description="In stock?")
def is_in_stock(self, obj):
    return obj.stock > 0

# Inlines
class ImageInline(admin.TabularInline):   # compact table
    model = ProductImage; extra = 1
class ImageInline(admin.StackedInline):   # full form each

# Action
@admin.action(description="Publish")
def publish(self, request, queryset):
    n = queryset.update(status="published")     # bulk SQL, skips save()/signals
    self.message_user(request, f"{n} published.")

# Hooks
def get_queryset(self, request):          # scope rows + select_related
def save_model(self, request, obj, form, change):  # stamp fields on save
def has_delete_permission(self, request, obj=None): # gate actions
```

```bash
# Setup
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver        # http://127.0.0.1:8000/admin/
```

**Permissions:** auto-created per model — `add_*`, `change_*`, `delete_*`, `view_*`. Need `is_staff` to enter; superuser bypasses all; `view_*` only ⇒ read-only; bundle perms into **Groups** (roles).

**Golden rules:** define `__str__`; register in `app/admin.py`; fix N+1 with `select_related`; the admin is for **trusted staff only** — build an API for end users.
