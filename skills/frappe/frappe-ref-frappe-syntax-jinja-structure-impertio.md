---
title: Frappe Ref — Frappe Syntax: Jinja: Structure (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'structure']
source: Impertio
---

# File Structure for Frappe Templates

> Directory structure and file organization for each template type in Frappe v14/v15/v16.

---

## Portal Pages (www/)

Portal pages live in the `www/` directory of your app. The directory structure maps directly to URL routes.

### Single Page

```
myapp/
└── www/
    ├── about.html          →  /about
    ├── about.py             →  Controller for /about
    ├── about.css            →  Auto-loaded CSS
    └── about.js             →  Auto-loaded JS
```

### Nested Pages

```
myapp/
└── www/
    └── projects/
        ├── index.html       →  /projects
        ├── index.py         →  Controller for /projects
        ├── index.css        →  Auto-loaded CSS
        ├── index.js         →  Auto-loaded JS
        └── detail.html      →  /projects/detail
```

### File Naming Rules

- ALWAYS use lowercase filenames
- ALWAYS use hyphens for multi-word names: `my-page.html` (NOT `my_page.html`)
- The `.py` controller filename MUST match the `.html` filename exactly
- `.css` and `.js` files with matching names are auto-included

### Controller Pattern

```python
# www/projects/index.py
import frappe

def get_context(context):
    context.title = "Projects"
    context.no_cache = True
    context.projects = frappe.get_all("Project",
        filters={"is_public": 1},
        fields=["name", "title", "description"])
    return context
```

### Overriding Standard Pages

To override Frappe's built-in pages (e.g., `/about`, `/contact`), place a file with the same name in your app's `www/` folder. Your app's version takes precedence if it is listed AFTER frappe in `sites/apps.txt`.

---

## Print Formats

### Custom Print Format (Jinja)

Custom Print Formats are stored in the database (DocType: Print Format). They are created via:
- **Setup > Print > Print Format** in the UI
- Or as fixtures in your app

### Standard Print Format (App-Level)

```
myapp/
└── myapp/
    └── module_name/
        └── print_format/
            └── my_invoice_format/
                ├── my_invoice_format.json    →  Print Format metadata
                └── my_invoice_format.html    →  Jinja template
```

### Print Format JSON

```json
{
    "doctype": "Print Format",
    "name": "My Invoice Format",
    "doc_type": "Sales Invoice",
    "module": "My Module",
    "print_format_type": "Jinja",
    "raw_printing": 0,
    "custom_format": 1
}
```

**Key fields**:
| Field | Description |
|-------|-------------|
| `doc_type` | The DocType this format applies to |
| `print_format_type` | ALWAYS `"Jinja"` for custom templates |
| `custom_format` | `1` for fully custom HTML |
| `raw_printing` | `1` for raw text (thermal printers) |

---

## Email Templates

Email Templates are stored in the database (DocType: Email Template).

### Structure

```
{
    "doctype": "Email Template",
    "name": "Payment Reminder",
    "subject": "Payment Reminder for {{ doc.name }}",
    "response": "<p>Dear {{ doc.customer_name }},</p>..."
}
```

### As App Fixtures

```
myapp/
└── myapp/
    └── module_name/
        └── email_template/
            └── payment_reminder/
                └── payment_reminder.json
```

The `subject` and `response` fields contain Jinja templates rendered with the `doc` context.

---

## Notification Templates

Notification Templates are configured in the database (DocType: Notification).

### Structure

| Field | Purpose |
|-------|---------|
| `subject` | Jinja template for notification subject |
| `message` | Jinja template for notification body |
| `document_type` | DocType that triggers the notification |
| `event` | Trigger event (Save, Submit, Days After, etc.) |

Both `subject` and `message` receive the `doc` object in their Jinja context.

---

## Custom Jinja Methods/Filters

```
myapp/
├── hooks.py                     →  Register via jenv hook
└── myapp/
    └── jinja/
        ├── __init__.py          →  REQUIRED (can be empty)
        ├── methods.py           →  Global Jinja functions
        └── filters.py           →  Jinja filters
```

### hooks.py Registration

```python
jenv = {
    "methods": ["myapp.jinja.methods"],
    "filters": ["myapp.jinja.filters"]
}
```

---

## Template Include Files

Shared template fragments go in the `templates/includes/` directory:

```
myapp/
└── myapp/
    └── templates/
        ├── includes/
        │   ├── company_header.html
        │   ├── item_table.html
        │   └── footer.html
        └── pages/
            └── custom_page.html
```

### Usage

```jinja
{% include "myapp/templates/includes/company_header.html" %}
```

ALWAYS use the full dotted path from the app root when including templates.

---

## Letter Head

Letter Heads are stored in the database (DocType: Letter Head).

| Field | Description |
|-------|-------------|
| `content` | HTML/Jinja content for the header |
| `footer` | HTML/Jinja content for the footer |
| `image` | Image URL for the letterhead |
| `is_default` | Whether this is the default letterhead |

Letter Heads are automatically rendered above and below Print Formats. They support Jinja syntax.

---

## Version Differences

| Feature | v14 | v15 | v16 |
|---------|:---:|:---:|:---:|
| www/ portal pages | Yes | Yes | Yes |
| Print Format (Jinja) | Yes | Yes | Yes |
| Email Template | Yes | Yes | Yes |
| Notification Template | Yes | Yes | Yes |
| Custom jenv methods | Yes | Yes | Yes |
| Markdown portal pages (.md) | Yes | Yes | Yes |
| YAML frontmatter in pages | Yes | Yes | Yes |
| Chrome PDF rendering | No | No | Yes |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/structure.md*
