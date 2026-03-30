---
name: django-admin
description: Expert guide for building, customizing, and extending Django's admin interface (Django 4.x+). Use this skill whenever the user asks about Django admin, ModelAdmin, admin customization, admin actions, inline admin, admin permissions, custom admin views, admin theming, admin dashboard, or anything related to `django.contrib.admin`. Also trigger for questions about registering models, list_display, list_filter, search_fields, admin forms, or admin charts. Always consult this skill before writing any Django admin code.
---

# Django Admin Skill

**Target: Django 4.x and 5.x only.** Use modern Django patterns — `@admin.register`, `@admin.display`, `@admin.action`, `ModelAdmin.get_queryset`, etc. Do not suggest deprecated patterns from Django 3.x or earlier.

This skill guides AI agents through building production-quality Django admin interfaces — from basic model registration to advanced custom views, theming, and permissions.

The user will typically provide a Django model, app, or feature they want exposed in the admin. Your job is to generate idiomatic, complete, and working admin code.

---

## Quick Reference: Which Section to Use

| User Goal | Go To |
|---|---|
| Register a model or customize list view | [Core ModelAdmin](#core-modeladmin) |
| Add inline editing of related models | [Inline Admin](#inline-admin) |
| Add custom actions or buttons | [Actions & Custom Views](#actions--custom-views) |
| Control who sees/edits what | [Permissions](#permissions--access-control) |
| Custom forms or widgets in admin | [Forms & Widgets](#forms--widgets) |
| Change admin theme or branding | [Theming & Branding](#theming--branding) → then read `references/theming.md` |
| Add charts or dashboard widgets | [Dashboard & Analytics](#dashboard--analytics) |

---

## Core ModelAdmin

### Basic Registration

```python
# app/admin.py
from django.contrib import admin
from .models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    # Columns shown in the list view
    list_display = ("title", "author", "status", "created_at")
    
    # Sidebar filters
    list_filter = ("status", "author", "created_at")
    
    # Full-text search fields (use __ for related fields)
    search_fields = ("title", "body", "author__username")
    
    # Fields that link to the detail view
    list_display_links = ("title",)
    
    # Editable columns directly in list view
    list_editable = ("status",)
    
    # Default ordering
    ordering = ("-created_at",)
    
    # Pagination
    list_per_page = 25
    
    # Read-only fields in detail view
    readonly_fields = ("created_at", "updated_at", "slug")
    
    # Date drill-down navigation
    date_hierarchy = "created_at"
    
    # Prepopulated fields (e.g. auto-fill slug from title)
    prepopulated_fields = {"slug": ("title",)}
```

### Fieldsets (grouping fields in detail view)

```python
fieldsets = (
    ("Content", {
        "fields": ("title", "slug", "body")
    }),
    ("Publishing", {
        "fields": ("status", "author", "published_at"),
        "classes": ("collapse",),  # collapsible section
    }),
    ("Metadata", {
        "fields": ("created_at", "updated_at"),
        "classes": ("collapse",),
    }),
)
```

### Computed / Custom Columns

```python
from django.utils.html import format_html

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ("title", "status_badge", "word_count")

    @admin.display(description="Status", ordering="status")
    def status_badge(self, obj):
        color = {"draft": "gray", "published": "green", "archived": "red"}.get(obj.status, "gray")
        return format_html('<span style="color:{}">{}</span>', color, obj.get_status_display())

    @admin.display(description="Words")
    def word_count(self, obj):
        return len(obj.body.split()) if obj.body else 0
```

### Autocomplete for ForeignKey / M2M

```python
# On the *related* model's admin, add search_fields:
@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin):
    search_fields = ("username", "email")  # required for autocomplete to work

# Then on the model using it:
@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    autocomplete_fields = ("author", "tags")  # replaces select with search widget
```

---

## Inline Admin

Use inlines to edit related objects on the same page as the parent.

```python
from django.contrib import admin
from .models import Order, OrderItem

class OrderItemInline(admin.TabularInline):  # or StackedInline for vertical layout
    model = OrderItem
    fields = ("product", "quantity", "unit_price", "total")
    readonly_fields = ("total",)
    extra = 1          # blank rows to show by default
    min_num = 0
    max_num = 10
    can_delete = True
    show_change_link = True  # link to item's own change page

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    inlines = [OrderItemInline]
```

**StackedInline** is better for models with many fields; **TabularInline** is compact (table rows).

---

## Actions & Custom Views

### Custom Admin Actions

```python
from django.contrib import admin, messages

@admin.action(description="Mark selected articles as published")
def publish_articles(modeladmin, request, queryset):
    updated = queryset.update(status="published")
    modeladmin.message_user(request, f"{updated} article(s) published.", messages.SUCCESS)

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    actions = [publish_articles]
```

### Custom Admin View (extra URL on a ModelAdmin)

```python
from django.urls import path
from django.shortcuts import render
from django.contrib.admin.views.decorators import staff_member_required

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):

    def get_urls(self):
        urls = super().get_urls()
        custom_urls = [
            path("stats/", self.admin_site.admin_view(self.stats_view), name="article-stats"),
        ]
        return custom_urls + urls

    def stats_view(self, request):
        context = {
            **self.admin_site.each_context(request),
            "title": "Article Statistics",
            "total": Article.objects.count(),
            "published": Article.objects.filter(status="published").count(),
        }
        return render(request, "admin/articles/stats.html", context)
```

Template at `templates/admin/articles/stats.html`:
```html
{% extends "admin/base_site.html" %}
{% block content %}
<h1>{{ title }}</h1>
<p>Total: {{ total }} | Published: {{ published }}</p>
{% endblock %}
```

### Add a Button to the Change List

```python
# Override changelist template or use a custom changelist view
class ArticleAdmin(admin.ModelAdmin):
    change_list_template = "admin/articles/change_list.html"
```

```html
<!-- templates/admin/articles/change_list.html -->
{% extends "admin/change_list.html" %}
{% block object-tools-items %}
    <li><a href="stats/" class="button">📊 View Stats</a></li>
    {{ block.super }}
{% endblock %}
```

---

## Permissions & Access Control

### Built-in Django Permission Hooks

```python
@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):

    def has_module_perms(self, request):
        # Whole app visible in admin index?
        return request.user.is_staff

    def has_view_permission(self, request, obj=None):
        return request.user.has_perm("articles.view_article")

    def has_add_permission(self, request):
        return request.user.has_perm("articles.add_article")

    def has_change_permission(self, request, obj=None):
        # Optionally restrict to object owner
        if obj and not request.user.is_superuser:
            return obj.author == request.user
        return request.user.has_perm("articles.change_article")

    def has_delete_permission(self, request, obj=None):
        return request.user.is_superuser

    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if request.user.is_superuser:
            return qs
        # Non-superusers only see their own articles
        return qs.filter(author=request.user)
```

### Row-Level Visibility via `get_queryset`

```python
def get_queryset(self, request):
    qs = super().get_queryset(request)
    if request.user.groups.filter(name="Editors").exists():
        return qs.filter(status__in=["draft", "review"])
    return qs
```

### Restricting Fields Per User

```python
def get_fields(self, request, obj=None):
    fields = super().get_fields(request, obj)
    if not request.user.is_superuser:
        fields = [f for f in fields if f not in ("author", "internal_notes")]
    return fields

def get_readonly_fields(self, request, obj=None):
    readonly = list(super().get_readonly_fields(request, obj))
    if not request.user.is_superuser:
        readonly += ["status", "published_at"]
    return readonly
```

---

## Forms & Widgets

### Custom Admin Form

```python
from django import forms
from django.contrib import admin

class ArticleAdminForm(forms.ModelForm):
    # Extra field not on the model
    send_notification = forms.BooleanField(required=False, label="Notify subscribers on save")

    class Meta:
        model = Article
        fields = "__all__"
        widgets = {
            "body": forms.Textarea(attrs={"rows": 20, "cols": 80}),
            "tags": forms.CheckboxSelectMultiple(),
        }

    def clean(self):
        data = super().clean()
        if data.get("status") == "published" and not data.get("body"):
            raise forms.ValidationError("Published articles must have a body.")
        return data

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    form = ArticleAdminForm

    def save_model(self, request, obj, form, change):
        super().save_model(request, obj, form, change)
        if form.cleaned_data.get("send_notification"):
            # trigger notification logic
            pass
```

### Raw ID Fields (for large related sets)

```python
class ArticleAdmin(admin.ModelAdmin):
    raw_id_fields = ("author",)  # shows ID input + lookup popup instead of dropdown
```

---

## Theming & Branding

> **Building a full custom theme?** Read [`references/theming.md`](references/theming.md) — it covers the complete template hierarchy, every CSS variable, dark mode, packaging as a reusable app, and a full working example.

### Quick: Site Title & Header Text

```python
# urls.py  (or admin.py)
from django.contrib import admin

admin.site.site_header = "My App Admin"   # top-left header text
admin.site.site_title = "My App"          # <title> tag suffix
admin.site.index_title = "Dashboard"      # h1 on the index page
```

### Quick: Inject CSS via `base_site.html`

Create `templates/admin/base_site.html` in any app that appears **before** `django.contrib.admin` in `INSTALLED_APPS`:

```html
{% extends "admin/base.html" %}
{% load static %}

{% block extrastyle %}
{{ block.super }}
<link rel="stylesheet" href="{% static 'admin/css/mytheme.css' %}">
{% endblock %}

{% block branding %}
<div id="site-name"><a href="{% url 'admin:index' %}">My App</a></div>
{% endblock %}
```

### Quick: CSS Variable Overrides (Django 4.x+)

Django's admin is fully themeable via CSS custom properties. A minimal retheme:

```css
/* static/admin/css/mytheme.css */
:root {
  --primary:        #6366f1;
  --secondary:      #4338ca;
  --accent:         #fbbf24;
  --primary-fg:     #fff;
  --header-bg:      #1e1b4b;
  --link-fg:        #6366f1;
  --button-bg:      #6366f1;
  --button-hover-bg: #4338ca;
}
```

**All ~40 available variables** are documented in `references/theming.md` section 3.

---

## Dashboard & Analytics

### Simple Stats on the Admin Index

Override `AdminSite.index()` by creating a custom AdminSite:

```python
# myapp/admin.py
from django.contrib.admin import AdminSite
from django.shortcuts import render

class MyAdminSite(AdminSite):
    site_header = "My App"

    def index(self, request, extra_context=None):
        extra_context = extra_context or {}
        extra_context["stats"] = {
            "users": User.objects.count(),
            "articles": Article.objects.count(),
            "orders_today": Order.objects.filter(created_at__date=date.today()).count(),
        }
        return super().index(request, extra_context)

admin_site = MyAdminSite(name="myadmin")
```

```python
# urls.py
from myapp.admin import admin_site
urlpatterns = [
    path("admin/", admin_site.urls),
]
```

Template `templates/admin/index.html`:
```html
{% extends "admin/index.html" %}
{% block content %}
<div style="display:flex; gap:1rem; margin-bottom:1rem;">
  {% for key, val in stats.items %}
  <div style="background:#fff; border:1px solid #ddd; padding:1rem; flex:1; text-align:center;">
    <div style="font-size:2rem; font-weight:bold;">{{ val }}</div>
    <div>{{ key|title }}</div>
  </div>
  {% endfor %}
</div>
{{ block.super }}
{% endblock %}
```

---



### Avoid N+1 queries in list_display

```python
class ArticleAdmin(admin.ModelAdmin):
    list_display = ("title", "author_name")

    # BAD: hits DB once per row
    def author_name(self, obj):
        return obj.author.username

    # GOOD: pre-select related
    def get_queryset(self, request):
        return super().get_queryset(request).select_related("author").prefetch_related("tags")
```

### Override save to set request.user

```python
def save_model(self, request, obj, form, change):
    if not change:  # on create only
        obj.created_by = request.user
    obj.updated_by = request.user
    super().save_model(request, obj, form, change)
```

### Soft-delete support

```python
class ArticleAdmin(admin.ModelAdmin):
    def get_queryset(self, request):
        # Show all (including soft-deleted) to superusers
        if request.user.is_superuser:
            return Article.all_objects.all()
        return super().get_queryset(request)

    def delete_model(self, request, obj):
        obj.deleted_at = timezone.now()
        obj.save()  # soft delete instead of real delete
```

### Confirm before a destructive action

```python
from django.template.response import TemplateResponse

@admin.action(description="Archive selected articles")
def archive_articles(modeladmin, request, queryset):
    if "confirm" in request.POST:
        queryset.update(status="archived")
        modeladmin.message_user(request, "Articles archived.")
        return

    context = {
        **modeladmin.admin_site.each_context(request),
        "queryset": queryset,
        "action": "archive_articles",
    }
    return TemplateResponse(request, "admin/confirm_archive.html", context)
```

---

## Django 4.x / 5.x Modern Features

### Facets (Django 5.0+)

Django 5.0 added **facet counts** to `list_filter` — shows the number of matching objects next to each filter option.

```python
from django.contrib.admin import ShowFacets

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_filter = ("status", "author")
    # Options: ShowFacets.ALWAYS (default in 5.0), ShowFacets.ALLOW, ShowFacets.NEVER
    show_facets = ShowFacets.ALWAYS
```

### `@admin.display` and `@admin.action` Decorators (Django 4.0+)

Prefer these over the old attribute-assignment style:

```python
# Modern — use these
@admin.display(description="Full Name", ordering="last_name")
def full_name(self, obj):
    return f"{obj.first_name} {obj.last_name}"

@admin.action(description="Publish selected")
def publish(modeladmin, request, queryset):
    queryset.update(status="published")

# Old style — avoid in new code
def full_name(self, obj): ...
full_name.short_description = "Full Name"    # deprecated pattern
full_name.admin_order_field = "last_name"    # deprecated pattern
```

### Custom List Filters

```python
from django.contrib.admin import SimpleListFilter

class DecadePublishedFilter(SimpleListFilter):
    title = "decade published"
    parameter_name = "decade"

    def lookups(self, request, model_admin):
        return [
            ("2020s", "2020s"),
            ("2010s", "2010s"),
        ]

    def queryset(self, request, queryset):
        if self.value() == "2020s":
            return queryset.filter(published_at__year__gte=2020)
        if self.value() == "2010s":
            return queryset.filter(published_at__year__range=(2010, 2019))

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_filter = [DecadePublishedFilter, "status"]
```

### `get_inlines()` — Dynamic Inlines (Django 4.0+)

Control which inlines appear based on the object state or user:

```python
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    def get_inlines(self, request, obj):
        inlines = [ImageInline]
        if obj and obj.is_digital:
            inlines.append(DownloadInline)
        else:
            inlines.append(ShippingInline)
        return inlines
```

---

## Common Patterns & Gotchas

### Avoid N+1 Queries in list_display

```python
class ArticleAdmin(admin.ModelAdmin):
    list_display = ("title", "author_name")

    @admin.display(description="Author", ordering="author__username")
    def author_name(self, obj):
        return obj.author.username  # safe because we select_related below

    def get_queryset(self, request):
        return super().get_queryset(request).select_related("author").prefetch_related("tags")
```

### Set request.user on Save

```python
def save_model(self, request, obj, form, change):
    if not change:
        obj.created_by = request.user
    obj.updated_by = request.user
    super().save_model(request, obj, form, change)
```

### Soft-Delete Support

```python
class ArticleAdmin(admin.ModelAdmin):
    def get_queryset(self, request):
        if request.user.is_superuser:
            return Article.all_objects.all()  # custom manager including deleted
        return super().get_queryset(request)

    def delete_model(self, request, obj):
        obj.deleted_at = timezone.now()
        obj.save()  # soft delete — skip super()
```

### Confirmation Step Before Destructive Action

```python
from django.template.response import TemplateResponse

@admin.action(description="Archive selected articles")
def archive_articles(modeladmin, request, queryset):
    if "confirm" in request.POST:
        queryset.update(status="archived")
        modeladmin.message_user(request, "Articles archived.")
        return

    context = {
        **modeladmin.admin_site.each_context(request),
        "queryset": queryset,
        "action": "archive_articles",
    }
    return TemplateResponse(request, "admin/confirm_archive.html", context)
```

---

## File Structure Reminder

```
myapp/
├── admin.py          # all ModelAdmin registrations
├── models.py
└── templates/
    └── admin/
        └── myapp/
            ├── change_list.html   # override list page
            ├── change_form.html   # override detail page
            └── stats.html         # custom view templates
```

For large apps, split admin into multiple files:
```
myapp/
└── admin/
    ├── __init__.py   # imports all sub-modules
    ├── article.py
    ├── user.py
    └── order.py
```
