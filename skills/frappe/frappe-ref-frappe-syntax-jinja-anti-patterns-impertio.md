---
title: Frappe Ref — Frappe Syntax: Jinja: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns: Jinja Mistakes in Frappe

> Common mistakes in Frappe Jinja templates and their correct alternatives. Each anti-pattern includes the problem, why it fails, and the deterministic fix.

---

## AP-01: Query in Loop (N+1 Problem)

### WRONG

```jinja
{% for item in doc.items %}
    {% set stock = frappe.db.get_value("Bin", {"item_code": item.item_code}, "actual_qty") %}
    <p>{{ item.item_name }}: {{ stock }} in stock</p>
{% endfor %}
```

**Problem**: 100 items = 100+ database queries. Causes timeouts on large documents.

### CORRECT

Pre-fetch all data in the Python controller:

```python
# Controller or custom print format script
def get_context(context):
    item_codes = [item.item_code for item in doc.items]
    bins = frappe.get_all("Bin",
        filters={"item_code": ["in", item_codes]},
        fields=["item_code", "actual_qty"])
    context.stock_qty = {b.item_code: b.actual_qty for b in bins}
```

```jinja
{% for item in doc.items %}
    <p>{{ item.item_name }}: {{ stock_qty.get(item.item_code, 0) }} in stock</p>
{% endfor %}
```

**Rule**: NEVER execute `frappe.get_doc()`, `frappe.db.get_value()`, or `frappe.get_all()` inside a `{% for %}` loop.

---

## AP-02: Unescaped User Input (XSS)

### WRONG

```jinja
{{ user_comment | safe }}
{{ frappe.form_dict.search | safe }}
{{ doc.custom_html_field | safe }}
```

**Problem**: User-supplied content can contain `<script>` tags. Marking as `safe` disables Jinja's auto-escaping, allowing script injection.

### CORRECT

```jinja
{# Auto-escaped by default — safe for user input #}
{{ user_comment }}
{{ frappe.form_dict.search }}

{# ONLY use safe for admin-controlled content #}
{{ doc.terms | safe }}
{{ doc.address_display | safe }}
```

**Rule**: NEVER use `| safe` on any value that originates from user input, URL parameters, or external data.

---

## AP-03: Heavy Calculations in Templates

### WRONG

```jinja
{% set total = 0 %}
{% for item in doc.items %}
    {% set discount = item.rate * (item.discount_percentage / 100) %}
    {% set tax = (item.rate - discount) * 0.21 %}
    {% set item_total = (item.rate - discount + tax) * item.qty %}
    {# NOTE: This does NOT work — set is block-scoped in for loops #}
    {% set total = total + item_total %}
{% endfor %}
<p>Total: {{ total }}</p>
```

**Problem**: Complex calculations in Jinja are slow, hard to test, and the `{% set %}` scoping means `total` stays at 0.

### CORRECT

```python
# In Python controller
def get_context(context):
    total = 0
    for item in doc.items:
        discount = item.rate * (item.discount_percentage / 100)
        tax = (item.rate - discount) * 0.21
        total += (item.rate - discount + tax) * item.qty
    context.calculated_total = total
```

```jinja
<p>{{ _("Total") }}: {{ calculated_total }}</p>
```

**Rule**: ALWAYS do calculations in Python. Templates are for display only.

---

## AP-04: Hardcoded Strings (Not Translatable)

### WRONG

```jinja
<th>Invoice Number</th>
<th>Amount</th>
<p>Thank you for your business!</p>
```

**Problem**: These strings will NEVER be translated for non-English users.

### CORRECT

```jinja
<th>{{ _("Invoice Number") }}</th>
<th>{{ _("Amount") }}</th>
<p>{{ _("Thank you for your business!") }}</p>

{# With variables — use {0} placeholder #}
<p>{{ _("Total: {0}").format(doc.get_formatted("grand_total")) }}</p>
```

**Rule**: ALWAYS wrap every user-facing string with `_()`. NEVER translate field values (they use Frappe's translation system automatically).

---

## AP-05: Missing Default Values

### WRONG

```jinja
{{ doc.customer_group }}
{{ doc.notes | truncate(100) }}
{{ doc.description | lower }}
```

**Problem**: If a field is `None`, the output shows "None" as text. Applying filters to `None` can cause errors.

### CORRECT

```jinja
{{ doc.customer_group | default("") }}
{{ doc.notes | default("") | truncate(100) }}

{# Or guard with a conditional #}
{% if doc.description %}
    {{ doc.description | lower }}
{% endif %}
```

**Rule**: ALWAYS use `| default()` for fields that may be `None` or empty, especially before chaining other filters.

---

## AP-06: Raw Field Values Instead of Formatted

### WRONG

```jinja
<p>{{ doc.grand_total }}</p>
<p>{{ doc.posting_date }}</p>
<p>{{ "%.2f" | format(doc.grand_total) }}</p>
```

**Problem**: Outputs raw database values like `1234.56` and `2024-01-15` without currency symbols, locale-specific number formatting, or date format preferences.

### CORRECT

```jinja
<p>{{ doc.get_formatted("grand_total") }}</p>
<p>{{ doc.get_formatted("posting_date") }}</p>

{# For child table rows — pass parent doc #}
{% for row in doc.items %}
    <td>{{ row.get_formatted("amount", doc) }}</td>
{% endfor %}
```

**Rule**: ALWAYS use `get_formatted()` for currency, date, number, and percentage fields in Print Formats.

---

## AP-07: get_doc for Single Field Lookup

### WRONG

```jinja
{% set customer = frappe.get_doc("Customer", doc.customer) %}
<p>{{ customer.customer_group }}</p>
```

**Problem**: `get_doc` loads the entire document with all child tables. For one field, this wastes memory and database queries.

### CORRECT

```jinja
{% set group = frappe.db.get_value("Customer", doc.customer, "customer_group") %}
<p>{{ group }}</p>
```

**Rule**: ALWAYS use `frappe.db.get_value()` when you need 1-3 fields. Use `frappe.get_doc()` ONLY when you need the full document or many fields.

---

## AP-08: Jinja Syntax in Report Print Formats

### WRONG

```html
<!-- This does NOT work in Report Print Formats -->
{% for item in data %}
    <tr><td>{{ item.name }}</td></tr>
{% endfor %}
```

**Problem**: Report Print Formats use JavaScript templating, NOT Jinja. The `{{ }}` syntax will not render.

### CORRECT

```html
<!-- JS Template syntax for Report Print Formats -->
{% for(var i=0; i < data.length; i++) { %}
<tr>
    <td>{%= data[i].name %}</td>
</tr>
{% } %}
```

**Rule**: NEVER use `{{ }}` in Report Print Formats. ALWAYS use `{%= %}` for output and JavaScript for logic.

---

## AP-09: Disabling safe_render Without Reason

### WRONG

```python
def get_context(context):
    context.safe_render = False  # "To make things work"
```

**Problem**: `safe_render` blocks templates containing `.__` to prevent access to Python internals. Disabling it opens the template to code injection.

### CORRECT

```python
def get_context(context):
    # NEVER disable safe_render unless you have reviewed ALL template
    # inputs and confirmed they cannot contain user-controlled data.
    # If a template fails with safe_render, fix the template instead.
    pass
```

**Rule**: NEVER disable `safe_render` without a documented security review. If a template breaks, fix the template — do not weaken security.

---

## AP-10: Forgetting Child Table Parent Doc

### WRONG

```jinja
{% for row in doc.items %}
    <td>{{ row.get_formatted("rate") }}</td>
    <td>{{ row.get_formatted("amount") }}</td>
{% endfor %}
```

**Problem**: Without the parent doc, `get_formatted()` cannot determine the correct currency from the parent document's currency field.

### CORRECT

```jinja
{% for row in doc.items %}
    <td>{{ row.get_formatted("rate", doc) }}</td>
    <td>{{ row.get_formatted("amount", doc) }}</td>
{% endfor %}
```

**Rule**: ALWAYS pass the parent `doc` as second argument to `get_formatted()` on child table rows.

---

## Summary

| # | Anti-Pattern | Rule |
|---|-------------|------|
| AP-01 | Query in loop | NEVER query inside `{% for %}` |
| AP-02 | `\| safe` on user input | NEVER use `\| safe` on untrusted data |
| AP-03 | Calculations in Jinja | ALWAYS calculate in Python |
| AP-04 | Hardcoded strings | ALWAYS use `_()` for text |
| AP-05 | Missing defaults | ALWAYS use `\| default()` |
| AP-06 | Raw field values | ALWAYS use `get_formatted()` |
| AP-07 | get_doc for one field | ALWAYS prefer `db.get_value()` |
| AP-08 | Jinja in Report formats | ALWAYS use `{%= %}` for reports |
| AP-09 | Disabling safe_render | NEVER disable without review |
| AP-10 | Missing parent doc | ALWAYS pass `doc` to child `get_formatted()` |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/anti-patterns.md*
