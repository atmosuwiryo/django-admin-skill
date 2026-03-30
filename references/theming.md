# Django Admin Theming Reference (Django 4.x / 5.x)

This reference covers everything needed to build a complete, production-quality custom Django admin theme using only built-in Django — no third-party packages.

---

## Table of Contents

1. [How Django Admin Templates Work](#1-how-django-admin-templates-work)
2. [Template Hierarchy & Key Blocks](#2-template-hierarchy--key-blocks)
3. [CSS Custom Properties (Variables)](#3-css-custom-properties-variables)
4. [Dark Mode (Django 5.x)](#4-dark-mode-django-5x)
5. [Packaging as a Reusable Django App](#5-packaging-as-a-reusable-django-app)
6. [Complete Theme Example](#6-complete-theme-example)
7. [Checklist](#7-checklist)

---

## 1. How Django Admin Templates Work

Django's admin uses Django's standard template loader. To override any admin template:

1. Add your app (or a dedicated theme app) to `INSTALLED_APPS` **before** `django.contrib.admin`
2. Create a `templates/admin/` directory inside that app
3. Mirror the filename from Django's admin templates

```
INSTALLED_APPS = [
    "mytheme",           # must come BEFORE django.contrib.admin
    "django.contrib.admin",
    ...
]
```

Django's original admin templates live at:
```
django/contrib/admin/templates/admin/
```

You can inspect them with:
```bash
python -c "import django; print(django.__file__)"
# then navigate to contrib/admin/templates/
```

**Override strategy**: always `{% extends "admin/base.html" %}` (or the appropriate parent) and only override the blocks you need. Never copy-paste the entire template — it breaks on Django upgrades.

---

## 2. Template Hierarchy & Key Blocks

### Core template chain

```
admin/base.html                  ← root: html, head, body skeleton
└── admin/base_site.html         ← branding, header, nav — override this first
    ├── admin/index.html         ← admin homepage / app index
    ├── admin/change_list.html   ← model list view
    │   └── admin/change_list_results.html
    ├── admin/change_form.html   ← add/edit object view
    ├── admin/delete_confirmation.html
    ├── admin/object_history.html
    └── admin/login.html         ← login page
```

### Blocks in `admin/base.html`

| Block | Purpose |
|---|---|
| `title` | `<title>` tag content |
| `extrastyle` | Extra `<style>` or `<link>` tags in `<head>` |
| `extrahead` | Extra content in `<head>` (after styles) |
| `branding` | The `#branding` div — site name/logo |
| `nav-global` | Top navigation bar |
| `content-related` | Right-hand sidebar |
| `content` | Main page content area |
| `footer` | Page footer |
| `usertools` | User info + logout in header |

### Blocks in `admin/base_site.html`

This is the primary override target. It extends `base.html` and adds:

| Block | Purpose |
|---|---|
| `branding` | Logo / site title HTML |
| `nav-global` | Global nav links |

### `admin/login.html` blocks

| Block | Purpose |
|---|---|
| `extrastyle` | Extra styles for login page |
| `content_title` | "Log in" heading |
| `content` | The login form area |
| `after_login_form` | Content after the form |

---

## 3. CSS Custom Properties (Variables)

Django 4.x introduced a full CSS custom property system. Override these in `:root` to retheme the entire admin without touching templates.

All variables are defined in `django/contrib/admin/static/admin/css/base.css`.

### Color Palette

```css
:root {
  --primary: #79aec8;           /* nav bar, links, focus rings */
  --secondary: #417690;         /* module headers, buttons */
  --accent: #f5dd5d;            /* highlight accent */
  --primary-fg: #fff;           /* text on primary color */

  --body-fg: #333;              /* main body text */
  --body-bg: #fff;              /* page background */
  --body-quiet-color: #666;     /* muted text */
  --body-loud-color: #000;      /* emphasized text */

  --header-color: #ffc;         /* header text */
  --header-branding-color: var(--accent);
  --header-bg: var(--secondary);
  --header-link-color: var(--primary-fg);

  --breadcrumbs-fg: #c4dce8;
  --breadcrumbs-link-fg: var(--body-bg);
  --breadcrumbs-bg: var(--primary);

  --link-fg: #447e9b;           /* hyperlink color */
  --link-hover-color: #036;
  --link-selected-fg: #5b80b2;

  --hairline-color: #e8e8e8;    /* thin borders/dividers */
  --border-color: #ccc;         /* standard borders */

  --error-fg: #ba2121;          /* error messages */
  --message-success-bg: #dfd;
  --message-warning-bg: #ffc;
  --message-error-bg: #ffefef;

  --darkened-bg: #f8f8f8;       /* alternate row bg, sidebar bg */
  --selected-bg: #e4e4f4;       /* selected row highlight */
  --selected-row: #ffc;         /* selected table row */

  --button-fg: #fff;
  --button-bg: var(--primary);
  --button-hover-bg: #609ab6;
  --default-button-bg: var(--secondary);
  --default-button-hover-bg: #205067;
  --close-button-bg: #888;
  --close-button-hover-bg: #747474;
  --delete-button-bg: #ba2121;
  --delete-button-hover-bg: #a41515;
}
```

### Object Tools & Misc

```css
:root {
  --object-tools-fg: var(--button-fg);
  --object-tools-bg: var(--close-button-bg);
  --object-tools-hover-bg: var(--close-button-hover-bg);

  --font-family-primary:
    "Source Sans Pro", Roboto, "Lucida Grande",
    "DejaVu Sans", Arial, Verdana, sans-serif;
  --font-family-monospace: "Source Code Pro", "DejaVu Sans Mono", monospace;
}
```

### How to Apply Overrides

Create `static/admin/css/custom_theme.css` in your theme app:

```css
/* mytheme/static/admin/css/custom_theme.css */
:root {
  --primary: #6366f1;           /* indigo */
  --secondary: #4338ca;
  --accent: #fbbf24;
  --primary-fg: #ffffff;
  --header-bg: var(--secondary);
  --link-fg: #6366f1;
  --link-hover-color: #4338ca;
  --button-bg: var(--primary);
  --button-hover-bg: var(--secondary);
}
```

Load it via `base_site.html`:

```html
{% extends "admin/base.html" %}
{% block extrastyle %}
{{ block.super }}
<link rel="stylesheet" href="{% static 'admin/css/custom_theme.css' %}">
{% endblock %}
```

---

## 4. Dark Mode (Django 5.x)

Django 5.0 added built-in dark mode support via `prefers-color-scheme`. It ships with a second set of CSS variables under `@media (prefers-color-scheme: dark)` in `dark_mode.css`.

### How It Works

Django 5.0 auto-loads `dark_mode.css` which overrides the `:root` variables for dark environments. The variables are the same names — you just re-declare them inside the media query.

### Extending Dark Mode in Your Theme

```css
/* mytheme/static/admin/css/custom_theme.css */

/* Light mode */
:root {
  --primary: #6366f1;
  --secondary: #4338ca;
  --body-bg: #ffffff;
  --body-fg: #111827;
  --darkened-bg: #f9fafb;
  --border-color: #e5e7eb;
}

/* Dark mode — same variable names, new values */
@media (prefers-color-scheme: dark) {
  :root {
    --primary: #818cf8;
    --secondary: #6366f1;
    --body-bg: #111827;
    --body-fg: #f9fafb;
    --darkened-bg: #1f2937;
    --border-color: #374151;
    --header-bg: #1e1b4b;
    --hairline-color: #374151;
    --selected-bg: #1e1b4b;
  }
}
```

### Disabling Django's Built-in Dark Mode

If you want full control and don't want Django's `dark_mode.css` at all:

```python
# In your ModelAdmin or custom AdminSite
class MyAdminSite(AdminSite):
    def each_context(self, request):
        context = super().each_context(request)
        # Remove dark mode CSS from media
        return context
```

Or override the `<head>` block to conditionally exclude it — but it's easier to just override the variables.

---

## 5. Packaging as a Reusable Django App

Structure for a self-contained, installable theme app:

```
mytheme/
├── __init__.py
├── apps.py
├── static/
│   └── admin/
│       └── css/
│           └── mytheme.css          ← all CSS variable overrides
└── templates/
    └── admin/
        ├── base_site.html           ← branding + load CSS
        └── login.html               ← custom login page (optional)
```

### `apps.py`

```python
from django.apps import AppConfig

class MythemeConfig(AppConfig):
    name = "mytheme"
    verbose_name = "My Admin Theme"
```

### `templates/admin/base_site.html`

```html
{% extends "admin/base.html" %}
{% load static %}

{% block title %}{{ title }} | {{ site_title|default:_('Django site admin') }}{% endblock %}

{% block extrastyle %}
{{ block.super }}
<link rel="stylesheet" type="text/css" href="{% static 'admin/css/mytheme.css' %}">
{% endblock %}

{% block branding %}
<div id="site-name">
  <a href="{% url 'admin:index' %}">
    {# Optional: swap text for an SVG logo #}
    <img src="{% static 'admin/img/logo.svg' %}" alt="{{ site_header }}" height="30"
         onerror="this.style.display='none'; this.nextElementSibling.style.display='inline'">
    <span style="display:none">{{ site_header|default:_('Django administration') }}</span>
  </a>
</div>
{% endblock %}

{% block nav-global %}{% endblock %}
```

### `templates/admin/login.html`

```html
{% extends "admin/login.html" %}
{% load static %}

{% block extrastyle %}
{{ block.super }}
<style>
  #login-form {
    background: var(--body-bg);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 2rem;
    box-shadow: 0 4px 24px rgba(0,0,0,.08);
  }
  .login #header {
    background: var(--header-bg);
    border-bottom: 3px solid var(--accent);
  }
</style>
{% endblock %}
```

### `settings.py` for consumers

```python
INSTALLED_APPS = [
    "mytheme",                   # before django.contrib.admin
    "django.contrib.admin",
    ...
]

# Optional: set these in settings or in urls.py
# admin.site.site_header = "My App"
# admin.site.site_title = "My App Admin"
```

---

## 6. Complete Theme Example

A full minimal dark-primary theme (indigo/dark):

```css
/* mytheme/static/admin/css/mytheme.css */

/* ── Fonts ─────────────────────────────────────────── */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap');

:root {
  --font-family-primary: 'Inter', system-ui, sans-serif;

  /* ── Brand Colors ──────────────────────────────────── */
  --primary:              #6366f1;
  --secondary:            #4338ca;
  --accent:               #fbbf24;
  --primary-fg:           #ffffff;

  /* ── Header ─────────────────────────────────────────── */
  --header-bg:            #1e1b4b;
  --header-color:         #e0e7ff;
  --header-link-color:    #c7d2fe;
  --header-branding-color: var(--accent);

  /* ── Breadcrumbs ─────────────────────────────────────── */
  --breadcrumbs-bg:       var(--primary);
  --breadcrumbs-fg:       #e0e7ff;
  --breadcrumbs-link-fg:  #fff;

  /* ── Body ─────────────────────────────────────────────── */
  --body-bg:              #ffffff;
  --body-fg:              #1f2937;
  --body-quiet-color:     #6b7280;
  --darkened-bg:          #f9fafb;
  --border-color:         #e5e7eb;
  --hairline-color:       #f3f4f6;

  /* ── Links ─────────────────────────────────────────────── */
  --link-fg:              #6366f1;
  --link-hover-color:     #4338ca;
  --link-selected-fg:     #4338ca;

  /* ── Buttons ─────────────────────────────────────────────── */
  --button-fg:            #fff;
  --button-bg:            var(--primary);
  --button-hover-bg:      var(--secondary);
  --default-button-bg:    var(--secondary);
  --default-button-hover-bg: #312e81;
  --delete-button-bg:     #dc2626;
  --delete-button-hover-bg: #b91c1c;

  /* ── Messages ─────────────────────────────────────────────── */
  --message-success-bg:   #d1fae5;
  --message-warning-bg:   #fef3c7;
  --message-error-bg:     #fee2e2;

  /* ── Module headers ──────────────────────────────────────── */
  /* Django uses --secondary for module headers automatically */
}

/* Dark mode overrides */
@media (prefers-color-scheme: dark) {
  :root {
    --primary:           #818cf8;
    --secondary:         #6366f1;
    --primary-fg:        #ffffff;

    --header-bg:         #0f0c29;
    --header-color:      #e0e7ff;

    --body-bg:           #111827;
    --body-fg:           #f9fafb;
    --body-quiet-color:  #9ca3af;
    --darkened-bg:       #1f2937;
    --border-color:      #374151;
    --hairline-color:    #1f2937;
    --selected-bg:       #1e1b4b;

    --link-fg:           #818cf8;
    --link-hover-color:  #a5b4fc;

    --button-bg:         #6366f1;
    --button-hover-bg:   #4f46e5;
  }
}

/* ── Small polish tweaks ─────────────────────────────── */
#header {
  border-bottom: 3px solid var(--accent);
}

.module h2, .module caption, thead th {
  border-radius: 4px 4px 0 0;
}

#content-main .module {
  border-radius: 6px;
  overflow: hidden;
}
```

---

## 7. Checklist

Before shipping a custom theme, verify:

- [ ] App listed in `INSTALLED_APPS` **before** `django.contrib.admin`
- [ ] `{% load static %}` present in all templates that use `{% static %}`
- [ ] `{{ block.super }}` called in `extrastyle` so Django's own CSS loads first
- [ ] All CSS overrides use `:root` variables (not hard-coded hex values in selectors)
- [ ] Dark mode tested with OS set to dark preference
- [ ] Login page tested (it has a separate layout from the rest of admin)
- [ ] `collectstatic` run — theme CSS must be in `STATICFILES_DIRS` or the app's `static/` folder
- [ ] No reliance on internal Django admin CSS class names that may change between versions
- [ ] Tested on the change list, change form, delete confirmation, and login pages
