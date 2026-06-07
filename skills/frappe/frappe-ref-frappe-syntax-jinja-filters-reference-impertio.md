---
title: Frappe Ref — Frappe Syntax: Jinja: Filters Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'filters-reference']
source: Impertio
---

# Jinja Filters Reference

> Standard Jinja2 and custom Frappe filters available in templates (v14/v15/v16).

---

## String Filters

| Filter | Syntax | Output |
|--------|--------|--------|
| `lower` | `{{ "HELLO" \| lower }}` | `hello` |
| `upper` | `{{ "hello" \| upper }}` | `HELLO` |
| `title` | `{{ "hello world" \| title }}` | `Hello World` |
| `capitalize` | `{{ "hello" \| capitalize }}` | `Hello` |
| `trim` | `{{ "  text  " \| trim }}` | `text` |
| `striptags` | `{{ "<b>text</b>" \| striptags }}` | `text` |
| `truncate` | `{{ text \| truncate(100) }}` | First 100 chars + `...` |
| `wordwrap` | `{{ text \| wordwrap(60) }}` | Wraps at 60 chars |
| `center` | `{{ "text" \| center(20) }}` | Centered in 20 chars |
| `replace` | `{{ "foo" \| replace("o", "0") }}` | `f00` |

### String Examples

```jinja
{{ doc.customer_name | upper }}
{{ doc.description | truncate(150) }}
{{ doc.notes | trim }}
{{ doc.status | lower | title }}
```

---

## HTML Filters

| Filter | Syntax | Effect |
|--------|--------|--------|
| `escape` / `e` | `{{ val \| escape }}` | HTML-encode special chars |
| `safe` | `{{ html \| safe }}` | Mark as safe HTML (skip auto-escape) |
| `striptags` | `{{ html \| striptags }}` | Remove all HTML tags |
| `urlize` | `{{ text \| urlize }}` | Convert URLs to `<a>` links |

### HTML Safety Rules

- Jinja auto-escapes output by default in Frappe
- ALWAYS use `| escape` or rely on auto-escaping for user input
- NEVER use `| safe` on user-supplied content (XSS risk)
- ONLY use `| safe` for admin-controlled content (e.g., `doc.terms`)

```jinja
{# Safe — auto-escaped #}
{{ doc.description }}

{# Safe — explicitly escaped #}
{{ user_input | escape }}

{# ONLY for trusted admin content #}
{{ doc.terms | safe }}

{# DANGEROUS — NEVER do this #}
{# {{ frappe.form_dict.search | safe }} #}
```

---

## List / Array Filters

| Filter | Syntax | Output |
|--------|--------|--------|
| `length` | `{{ items \| length }}` | Number of items |
| `first` | `{{ items \| first }}` | First item |
| `last` | `{{ items \| last }}` | Last item |
| `join` | `{{ items \| join(", ") }}` | Joined string |
| `sort` | `{{ items \| sort }}` | Sorted list |
| `reverse` | `{{ items \| reverse \| list }}` | Reversed list |
| `unique` | `{{ items \| unique \| list }}` | Deduplicated list |
| `reject` | `{{ items \| reject("none") \| list }}` | Filter out None |
| `select` | `{{ items \| select("string") \| list }}` | Keep only strings |
| `map` | `{{ items \| map(attribute="name") \| list }}` | Extract attribute |
| `selectattr` | `{{ items \| selectattr("qty", "gt", 0) \| list }}` | Filter by attr |
| `groupby` | `{{ items \| groupby("category") }}` | Group by attribute |
| `batch` | `{{ items \| batch(3) }}` | Split into chunks of 3 |

### List Examples

```jinja
{# Count items #}
<p>{{ doc.items | length }} {{ _("items") }}</p>

{# Extract and join names #}
{% set names = doc.items | map(attribute="item_name") | list %}
<p>{{ names | join(", ") }}</p>

{# Filter items with qty > 0 #}
{% set active = doc.items | selectattr("qty", "gt", 0) | list %}
{% for item in active %}
    {{ item.item_name }}
{% endfor %}

{# Group items by category #}
{% for group in doc.items | groupby("item_group") %}
    <h3>{{ group.grouper }}</h3>
    {% for item in group.list %}
        <p>{{ item.item_name }}</p>
    {% endfor %}
{% endfor %}

{# Batch into rows of 3 (grid layout) #}
{% for row in doc.items | batch(3) %}
<div class="row">
    {% for item in row %}
    <div class="col-md-4">{{ item.item_name }}</div>
    {% endfor %}
</div>
{% endfor %}
```

---

## Number Filters

| Filter | Syntax | Output |
|--------|--------|--------|
| `round` | `{{ 3.14159 \| round(2) }}` | `3.14` |
| `int` | `{{ "42" \| int }}` | `42` |
| `float` | `{{ "3.14" \| float }}` | `3.14` |
| `abs` | `{{ -5 \| abs }}` | `5` |

### Number Examples

```jinja
{{ doc.discount_percentage | round(2) }}
{{ doc.qty | int }}
{{ doc.balance | abs }}
```

**NOTE**: For currency and date fields, ALWAYS use `doc.get_formatted()` or `frappe.format()` instead of raw number filters. These respect the system locale and currency settings.

---

## Default Value Filter

```jinja
{# String default #}
{{ doc.customer_group | default("Not Set") }}

{# Numeric default #}
{{ doc.discount_percentage | default(0) }}

{# Empty string default (prevents "None" output) #}
{{ doc.notes | default("") }}

{# Chained with other filters #}
{{ doc.description | default("") | truncate(100) }}
```

ALWAYS use `| default()` for fields that may be `None` or empty, especially before applying other filters like `truncate`, `lower`, or `length`.

---

## Filter Chaining

Filters chain left-to-right. Each filter receives the output of the previous:

```jinja
{# Clean and format text #}
{{ doc.description | default("") | trim | truncate(100) }}

{# Newlines to HTML breaks (requires custom nl2br filter or safe) #}
{{ doc.notes | default("") | replace("\n", "<br>") | safe }}

{# Extract, sort, and join #}
{{ doc.items | map(attribute="item_name") | sort | join(", ") }}

{# Case conversion chain #}
{{ doc.customer_name | lower | title }}
```

---

## Custom Filters via hooks.py

Register custom filters in `hooks.py`:

```python
jenv = {
    "filters": ["myapp.jinja.filters"]
}
```

All public functions in the module become available as filters:

```python
# myapp/jinja/filters.py
def nl2br(text):
    """{{ doc.notes | nl2br | safe }}"""
    return (text or "").replace("\n", "<br>")

def format_currency_custom(value, currency="EUR"):
    """{{ amount | format_currency_custom("USD") }}"""
    if value is None:
        return ""
    return f"{currency} {value:,.2f}"
```

See `references/methods.md` for complete custom filter/method documentation.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/filters-reference.md*
