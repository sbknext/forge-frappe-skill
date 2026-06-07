---
title: Frappe Ref — Frappe Syntax: Customapp: Templates (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'customapp', 'reference', 'templates']
source: Impertio
---

# Custom App — Templates & Portal Pages

## Templates Directory

```
my_custom_app/
└── templates/
    ├── __init__.py
    ├── includes/            # Reusable Jinja snippets
    │   ├── header.html
    │   └── footer.html
    └── pages/               # Standalone template pages
        ├── __init__.py
        └── custom_page.html
```

Templates in the `templates/includes/` directory are automatically available for inclusion in other Jinja templates across the app.

---

## WWW (Portal Pages)

```
my_custom_app/
└── www/
    ├── projects/
    │   ├── index.html       # Jinja template
    │   └── index.py         # Context controller
    └── contact/
        ├── index.html
        └── index.py
```

**URL mapping**: Directory path maps directly to URL.
- `www/projects/index.html` → `/projects`
- `www/contact/index.html` → `/contact`

---

## Portal Page Template (index.html)

```html
{% extends "templates/web.html" %}

{% block page_content %}
<div class="container">
    <h1>{{ title }}</h1>
    {% for item in items %}
    <div class="item">
        <h3>{{ item.name }}</h3>
        <p>{{ item.description }}</p>
    </div>
    {% endfor %}
</div>
{% endblock %}
```

---

## Portal Page Context (index.py)

```python
import frappe

def get_context(context):
    """Provide template context."""
    context.title = "My Projects"
    context.items = frappe.get_all(
        "Project",
        filters={"status": "Open"},
        fields=["name", "description"],
        order_by="creation desc",
        limit_page_length=20,
    )
    return context
```

---

## Template Variables

Available in all portal templates:

| Variable | Description |
|----------|-------------|
| `frappe.session.user` | Current logged-in user |
| `frappe.utils.now()` | Current datetime |
| `frappe.form_dict` | URL query parameters |
| `csrf_token` | CSRF token for forms |

---

## Hooks for Website

```python
# hooks.py

# Custom routes
website_route_rules = [
    {"from_route": "/custom/<path:app_path>", "to_route": "custom_page"},
]

# Portal menu items
portal_menu_items = [
    {"title": "My Projects", "route": "/projects", "role": "Customer"},
]

# Website context
website_context = {
    "favicon": "/assets/my_custom_app/images/favicon.ico",
}
```

---

## Including App Assets in Templates

```html
<!-- Include CSS -->
<link rel="stylesheet" href="/assets/my_custom_app/css/custom.css">

<!-- Include JS -->
<script src="/assets/my_custom_app/js/custom.js"></script>
```

Assets in `public/` are served at `/assets/{app_name}/`.

---

## Print Format Templates

```
my_custom_app/
└── my_module/
    └── print_format/
        └── custom_invoice/
            ├── custom_invoice.json  # Print format definition
            └── custom_invoice.html  # Jinja template
```

```html
<!-- custom_invoice.html -->
<div class="print-format">
    <h1>{{ doc.name }}</h1>
    <p>Customer: {{ doc.customer }}</p>

    <table>
        {% for item in doc.items %}
        <tr>
            <td>{{ item.item_code }}</td>
            <td>{{ item.qty }}</td>
            <td>{{ frappe.utils.fmt_money(item.amount) }}</td>
        </tr>
        {% endfor %}
    </table>
</div>
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-customapp/references/templates.md*
