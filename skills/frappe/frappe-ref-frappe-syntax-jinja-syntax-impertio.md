---
title: Frappe Ref — Frappe Syntax: Jinja: Syntax (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'syntax']
source: Impertio
---

# Jinja Syntax Reference

> Complete Jinja2 syntax reference for Frappe templates (v14/v15/v16).

---

## Delimiters

| Delimiter | Purpose | Example |
|-----------|---------|---------|
| `{{ }}` | Output expression | `{{ doc.name }}` |
| `{% %}` | Statement (control flow) | `{% if doc.paid %}` |
| `{# #}` | Comment (not rendered) | `{# TODO: fix this #}` |
| `{%- -%}` | Strip whitespace | `{%- if True -%}` |

### Whitespace Control

Add `-` inside delimiters to strip leading/trailing whitespace:

```jinja
{# Normal — produces blank lines #}
{% if doc.items %}
    content
{% endif %}

{# Stripped — no extra blank lines #}
{%- if doc.items -%}
    content
{%- endif -%}
```

---

## Tags (Statements)

### Conditionals

```jinja
{% if condition %}
    ...
{% elif other_condition %}
    ...
{% else %}
    ...
{% endif %}
```

ALWAYS close with `{% endif %}`. Nesting is supported.

### Truthiness Rules

| Value | Truthy? |
|-------|---------|
| Non-empty string | Yes |
| Non-zero number | Yes |
| Non-empty list | Yes |
| `None` | No |
| `0` | No |
| `""` (empty string) | No |
| `[]` (empty list) | No |

### For Loops

```jinja
{% for item in doc.items %}
    {{ loop.index }}. {{ item.item_name }}
{% else %}
    No items found.
{% endfor %}
```

ALWAYS close with `{% endfor %}`. The `{% else %}` block runs when the iterable is empty.

### Loop Variables

| Variable | Type | Description |
|----------|------|-------------|
| `loop.index` | int | Current iteration (1-indexed) |
| `loop.index0` | int | Current iteration (0-indexed) |
| `loop.revindex` | int | Iterations remaining (1-indexed) |
| `loop.first` | bool | `True` on first iteration |
| `loop.last` | bool | `True` on last iteration |
| `loop.length` | int | Total number of items |
| `loop.cycle` | func | Cycle through values: `loop.cycle("odd", "even")` |

### Variable Assignment

```jinja
{% set total = 0 %}
{% set customer_name = doc.customer_name | default("Unknown") %}
{% set items_list = doc.items | list %}
```

**NOTE**: `{% set %}` inside a `{% for %}` block creates a block-scoped variable. To accumulate values across loop iterations, use namespace:

```jinja
{% set ns = namespace(total=0) %}
{% for row in doc.items %}
    {% set ns.total = ns.total + row.amount %}
{% endfor %}
Total: {{ ns.total }}
```

### Macros (Reusable Snippets)

```jinja
{% macro render_field(label, value) %}
<div class="field">
    <label>{{ _(label) }}</label>
    <span>{{ value | default("—") }}</span>
</div>
{% endmacro %}

{{ render_field("Customer", doc.customer_name) }}
{{ render_field("Date", doc.get_formatted("posting_date")) }}
```

### Include

```jinja
{% include "templates/includes/address.html" %}
{% include "templates/includes/footer.html" ignore missing %}
```

`ignore missing` prevents errors if the included file does not exist.

### Extends / Blocks

```jinja
{# Base template: templates/web.html #}
{% extends "templates/web.html" %}

{% block title %}{{ _("My Page") }}{% endblock %}

{% block page_content %}
    <h1>{{ _("Content here") }}</h1>
{% endblock %}
```

ALWAYS use `{% extends %}` as the FIRST tag in portal page templates.

---

## Expressions

### Attribute Access

```jinja
{{ doc.name }}           {# Dot notation #}
{{ doc["name"] }}        {# Bracket notation #}
{{ doc.items[0].qty }}   {# Nested access #}
```

### Operators

| Operator | Example |
|----------|---------|
| `+` `-` `*` `/` `//` `%` `**` | `{{ 10 + 5 }}` |
| `==` `!=` `>` `<` `>=` `<=` | `{% if qty > 0 %}` |
| `and` `or` `not` | `{% if a and b %}` |
| `in` | `{% if "Open" in statuses %}` |
| `is` | `{% if value is defined %}` |
| `~` (string concat) | `{{ "Hello " ~ name }}` |

### Ternary (Inline If)

```jinja
{{ "Paid" if doc.status == "Paid" else "Unpaid" }}
{{ doc.discount_amount if doc.discount_amount else 0 }}
```

---

## Tests

| Test | Example | Description |
|------|---------|-------------|
| `defined` | `{% if var is defined %}` | Variable exists |
| `undefined` | `{% if var is undefined %}` | Variable does not exist |
| `none` | `{% if val is none %}` | Value is `None` |
| `string` | `{% if val is string %}` | Value is a string |
| `number` | `{% if val is number %}` | Value is numeric |
| `even` / `odd` | `{% if loop.index is even %}` | Even/odd number |
| `divisibleby` | `{% if loop.index is divisibleby(3) %}` | Divisibility check |

---

## Escaping

Jinja auto-escapes HTML by default in Frappe. To output raw HTML from trusted sources:

```jinja
{# Auto-escaped (safe for user input) #}
{{ doc.description }}

{# Raw HTML — NEVER use for user input #}
{{ doc.terms | safe }}

{# Explicit escape #}
{{ value | escape }}
{{ value | e }}

{# Escape Jinja delimiters in output #}
{{ "{{ this is literal }}" }}
{% raw %}
    {{ this will not be parsed }}
{% endraw %}
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/syntax.md*
