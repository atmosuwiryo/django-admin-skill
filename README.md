# django-admin skill

An AI agent skill for building, customizing, and extending Django's admin interface.

## Install

```bash
npx skills add django-admin.skill
```

## Requirements

- Django 4.x or 5.x
- Python 3.10+

## What's covered

- **ModelAdmin** — `list_display`, `list_filter`, `search_fields`, fieldsets, computed columns, autocomplete
- **Inline admin** — `TabularInline`, `StackedInline`, dynamic inlines via `get_inlines()`
- **Actions & custom views** — custom actions, extra admin URLs, confirmation dialogs, changelist buttons
- **Permissions** — `has_*_permission` hooks, row-level filtering, per-user field control
- **Forms & widgets** — custom `ModelForm`, validation, widgets, `raw_id_fields`
- **Theming** — full CSS variable reference, template hierarchy, dark mode (Django 5.x), packaging as a reusable app
- **Dashboard** — custom `AdminSite`, stat cards, index page overrides
- **Modern features** — facets (5.0+), `@admin.display` / `@admin.action` decorators, `SimpleListFilter`

## Files

```
django-admin/
├── SKILL.md                  # main skill instructions
└── references/
    └── theming.md            # deep-dive: templates, CSS variables, dark mode, packaging
```

## License

MIT
