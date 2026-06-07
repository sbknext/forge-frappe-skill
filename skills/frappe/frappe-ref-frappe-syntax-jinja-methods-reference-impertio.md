---
title: Frappe Ref — Frappe Syntax: Jinja: Methods Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'methods-reference']
source: Impertio
---

# Frappe Methods Reference for Jinja

> All whitelisted frappe.* methods available in Jinja templates (v14/v15/v16).

---

## Formatting Methods

### doc.get_formatted(fieldname, parent_doc=None)

**RECOMMENDED** — ALWAYS use this for displaying field values in Print Formats.

```jinja
{# Parent document fields #}
{{ doc.get_formatted("posting_date") }}
{{ doc.get_formatted("grand_total") }}
{{ doc.get_formatted("status") }}

{# Child table rows — ALWAYS pass parent doc #}
{% for row in doc.items %}
    {{ row.get_formatted("rate", doc) }}
    {{ row.get_formatted("amount", doc) }}
{% endfor %}
```

**Why**: `get_formatted()` respects system number format, currency, date format, and field options. Raw field access (`doc.grand_total`) outputs the database value without formatting.

### frappe.format(value, df, doc=None)

Formats a raw value using an explicit field definition.

```jinja
{{ frappe.format(1234.56, {"fieldtype": "Currency"}) }}
{# Output: "$ 1,234.56" (depends on system settings) #}

{{ frappe.format("2024-01-15", {"fieldtype": "Date"}) }}
{# Output: "01-15-2024" (depends on date format setting) #}

{{ frappe.format(value, {"fieldtype": "Currency", "options": "currency"}) }}
```

### frappe.format_date(date_string)

Formats a date to human-readable long format.

```jinja
{{ frappe.format_date(doc.posting_date) }}
{# Output: "January 15, 2024" #}

{# v15+ with custom format string #}
{{ frappe.utils.format_date(doc.posting_date, "d MMMM, YYYY") }}
{# Output: "15 January, 2024" #}
```

---

## Document Retrieval Methods

### frappe.get_doc(doctype, name)

Retrieves a complete document object. Use ONLY when you need multiple fields.

```jinja
{% set customer = frappe.get_doc("Customer", doc.customer) %}
<p>{{ customer.customer_name }}</p>
<p>{{ customer.territory }}</p>
<p>{{ customer.customer_group }}</p>
```

**NEVER use `get_doc` when you need only one field** — use `frappe.db.get_value()` instead.

### frappe.get_all(doctype, filters, fields, order_by, start, page_length, pluck)

Returns list of records. Does NOT check user permissions.

```jinja
{% set tasks = frappe.get_all("Task",
    filters={"status": "Open"},
    fields=["title", "due_date"],
    order_by="due_date asc",
    page_length=10) %}

{% for task in tasks %}
    <p>{{ task.title }} — {{ frappe.format_date(task.due_date) }}</p>
{% endfor %}
```

**Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `doctype` | String | DocType name |
| `filters` | Dict | Filter conditions |
| `fields` | List | Fields to return |
| `order_by` | String | Sort clause |
| `start` | Int | Offset for pagination |
| `page_length` | Int | Limit results |
| `pluck` | String | Return flat list of single field values |

### frappe.get_list(doctype, ...)

Same as `get_all` but respects current user's permissions. ALWAYS use in portal pages.

```jinja
{% set orders = frappe.get_list("Sales Order",
    filters={"customer": doc.customer},
    fields=["name", "grand_total", "transaction_date"]) %}
```

---

## Database Methods

### frappe.db.get_value(doctype, name, fieldname)

Retrieves specific field value(s). ALWAYS prefer over `get_doc` for single fields.

```jinja
{# Single value #}
{% set abbr = frappe.db.get_value("Company", doc.company, "abbr") %}
<p>{{ doc.company }} ({{ abbr }})</p>

{# Multiple values (returns tuple) #}
{% set name, group = frappe.db.get_value("Customer", doc.customer,
    ["customer_name", "customer_group"]) %}
```

### frappe.db.get_single_value(doctype, fieldname)

Retrieves a field value from a Single DocType (e.g., System Settings).

```jinja
{% set timezone = frappe.db.get_single_value("System Settings", "time_zone") %}
{% set country = frappe.db.get_single_value("System Settings", "country") %}
```

---

## System & Utility Methods

### frappe.get_system_settings(fieldname)

Shortcut for `frappe.db.get_single_value("System Settings", fieldname)`.

```jinja
{% if frappe.get_system_settings("country") == "India" %}
    <p>GST: {{ doc.get_formatted("gst_amount") }}</p>
{% endif %}
```

### frappe.get_meta(doctype)

Returns DocType metadata (field definitions, properties).

```jinja
{% set meta = frappe.get_meta("Task") %}
<p>{{ meta.fields | length }} fields defined</p>

{% if meta.get_field("priority") %}
    <p>Priority field exists</p>
{% endif %}
```

### frappe.get_fullname(user=None)

Returns the full name of a user. Defaults to current session user.

```jinja
{# Current user #}
<p>{{ _("Prepared by") }}: {{ frappe.get_fullname() }}</p>

{# Specific user #}
<p>{{ _("Owner") }}: {{ frappe.get_fullname(doc.owner) }}</p>
```

### frappe.get_url()

Returns the site URL (e.g., `https://mysite.frappe.cloud`).

```jinja
<a href="{{ frappe.get_url() }}/app/sales-invoice/{{ doc.name }}">
    {{ _("View Invoice") }}
</a>
```

### frappe.render_template(template, context)

Renders another Jinja template with a given context.

```jinja
{# Render a template file #}
{{ frappe.render_template("templates/includes/footer.html", {}) }}

{# Render a string template #}
{{ frappe.render_template("Hello {{ name }}", {"name": "World"}) }}
```

### _() — Translation Function

Translates a string to the current language.

```jinja
<h1>{{ _("Invoice") }}</h1>
<p>{{ _("Total: {0}").format(doc.get_formatted("grand_total")) }}</p>
<p>{{ _("Dear {0}").format(doc.customer_name) }}</p>
```

ALWAYS wrap user-facing strings with `_()`. NEVER translate field values — they are already translatable via the Frappe translation framework.

---

## Session & Request Methods

### frappe.session.user

```jinja
{% if frappe.session.user != "Guest" %}
    <p>{{ _("Logged in as") }}: {{ frappe.session.user }}</p>
{% endif %}
```

### frappe.session.csrf_token

```jinja
<form method="POST">
    <input type="hidden" name="csrf_token" value="{{ frappe.session.csrf_token }}">
</form>
```

### frappe.form_dict

Query parameters dictionary. Available in web requests only.

```jinja
{# URL: /page?status=Open&limit=10 #}
{% if frappe.form_dict %}
    {% set status = frappe.form_dict.status | default("All") %}
    {% set limit = frappe.form_dict.limit | default(20) | int %}
{% endif %}
```

### frappe.lang

Current language code (two-letter lowercase).

```jinja
{% if frappe.lang == "ar" %}
    <div dir="rtl">...</div>
{% endif %}
```

---

## Method Availability by Template Type

| Method | Print Format | Email | Notification | Portal |
|--------|:---:|:---:|:---:|:---:|
| `doc.get_formatted()` | Yes | Yes | Yes | N/A |
| `frappe.format()` | Yes | Yes | Yes | Yes |
| `frappe.format_date()` | Yes | Yes | Yes | Yes |
| `frappe.get_doc()` | Yes | Yes | Yes | Yes |
| `frappe.get_all()` | Yes | Yes | Yes | Yes |
| `frappe.get_list()` | Yes | Yes | Yes | Yes |
| `frappe.db.get_value()` | Yes | Yes | Yes | Yes |
| `frappe.get_fullname()` | Yes | Yes | Yes | Yes |
| `frappe.get_url()` | Yes | Yes | Yes | Yes |
| `frappe.session.user` | Yes | N/A | N/A | Yes |
| `frappe.form_dict` | N/A | N/A | N/A | Yes |
| `frappe.render_template()` | Yes | Yes | Yes | Yes |
| `_()` | Yes | Yes | Yes | Yes |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/methods-reference.md*
