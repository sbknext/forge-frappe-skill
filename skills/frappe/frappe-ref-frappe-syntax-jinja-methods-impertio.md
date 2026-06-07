---
title: Frappe Ref — Frappe Syntax: Jinja: Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'methods']
source: Impertio
---

# Custom Jinja Methods & Filters via Hooks

> How to register and implement custom Jinja methods and filters in Frappe v14/v15/v16.

---

## Hook Registration

### hooks.py — jenv Hook

```python
# hooks.py
jenv = {
    "methods": [
        "myapp.jinja.methods"       # All public functions become Jinja globals
    ],
    "filters": [
        "myapp.jinja.filters"       # All public functions become Jinja filters
    ]
}
```

**How it works**:
- Frappe imports the specified Python module
- ALL public functions (no leading `_`) in that module are registered
- Methods become global functions callable as `{{ function_name(args) }}`
- Filters become pipe-able as `{{ value | filter_name(args) }}`

### Module Structure

```
myapp/
├── hooks.py
└── jinja/
    ├── __init__.py        # REQUIRED — can be empty
    ├── methods.py         # Custom global functions
    └── filters.py         # Custom filters
```

ALWAYS create `__init__.py` in the jinja directory. Without it, Python cannot import the module.

---

## Writing Custom Methods

Custom methods are called as global functions in Jinja templates.

### Method File

```python
# myapp/jinja/methods.py
import frappe

def get_company_logo(company):
    """Get company logo URL for use in templates."""
    return frappe.db.get_value("Company", company, "company_logo") or ""

def get_outstanding_invoices(customer, limit=5):
    """Get outstanding invoices for a customer."""
    return frappe.get_all(
        "Sales Invoice",
        filters={
            "customer": customer,
            "docstatus": 1,
            "outstanding_amount": [">", 0]
        },
        fields=["name", "posting_date", "grand_total", "outstanding_amount"],
        order_by="posting_date desc",
        limit_page_length=limit
    )

def format_address(address_name):
    """Format an Address document to a display string."""
    if not address_name:
        return ""
    address = frappe.get_doc("Address", address_name)
    parts = filter(None, [
        address.address_line1,
        address.address_line2,
        address.city,
        address.pincode,
        address.country
    ])
    return ", ".join(parts)
```

### Usage in Templates

```jinja
<img src="{{ get_company_logo(doc.company) }}" alt="Logo">

{% set invoices = get_outstanding_invoices(doc.customer) %}
{% for inv in invoices %}
    <p>{{ inv.name }}: {{ inv.outstanding_amount }}</p>
{% endfor %}

<p>{{ format_address(doc.customer_address) }}</p>
```

---

## Writing Custom Filters

Custom filters receive the piped value as the first argument.

### Filter File

```python
# myapp/jinja/filters.py

def nl2br(text):
    """Convert newlines to <br> tags. Usage: {{ text | nl2br }}"""
    if not text:
        return ""
    return text.replace("\n", "<br>")

def currency_words(amount, currency="EUR"):
    """Format amount with currency prefix. Usage: {{ 100 | currency_words("USD") }}"""
    if amount is None:
        return ""
    return f"{currency} {amount:,.2f}"

def phone_format(phone):
    """Format phone number. Usage: {{ phone | phone_format }}"""
    if not phone:
        return ""
    digits = "".join(c for c in phone if c.isdigit())
    if len(digits) == 10:
        return f"({digits[:3]}) {digits[3:6]}-{digits[6:]}"
    return phone

def wrap_text(text, width=80):
    """Wrap text at specified width. Usage: {{ text | wrap_text(60) }}"""
    if not text:
        return ""
    import textwrap
    return textwrap.fill(text, width=width)
```

### Usage in Templates

```jinja
{{ doc.notes | nl2br | safe }}
{{ doc.grand_total | currency_words("USD") }}
{{ doc.phone | phone_format }}
{{ doc.description | wrap_text(60) }}
```

---

## Rules for Custom Methods/Filters

### ALWAYS

1. Handle `None` and empty string inputs gracefully
2. Return a string (or value that converts to string) — Jinja renders the return value
3. Keep functions pure — NEVER modify documents or database state
4. Use `frappe.db.get_value()` over `frappe.get_doc()` for single-field lookups
5. Create `__init__.py` in the jinja module directory

### NEVER

1. Perform write operations (`frappe.db.set_value`, `doc.save()`) in Jinja methods
2. Raise exceptions — return empty string or fallback value instead
3. Use `print()` or `frappe.log_error()` for debugging in production
4. Name functions starting with `_` — they will NOT be registered
5. Import heavy libraries at module level — use lazy imports if needed

---

## Debugging Custom Methods

```python
# Bench console test
bench console
>>> from myapp.jinja.methods import get_company_logo
>>> get_company_logo("My Company")
"/files/logo.png"

# Verify hook registration
>>> import frappe
>>> frappe.get_jenv()  # Returns the Jinja environment
>>> frappe.get_jenv().globals.keys()  # Lists all registered globals
```

---

## Version Notes

| Feature | v14 | v15 | v16 |
|---------|:---:|:---:|:---:|
| `jenv.methods` hook | Yes | Yes | Yes |
| `jenv.filters` hook | Yes | Yes | Yes |
| Module-level registration | Yes | Yes | Yes |
| Individual function registration | No | No | No |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/methods.md*
