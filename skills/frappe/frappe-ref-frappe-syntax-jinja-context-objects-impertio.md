---
title: Frappe Ref — Frappe Syntax: Jinja: Context Objects (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'context-objects']
source: Impertio
---

# Context Objects Reference

> Available objects per Jinja template type in Frappe v14/v15/v16.

---

## Print Formats

| Object | Type | Description |
|--------|------|-------------|
| `doc` | Document | The document being printed (full object with all fields) |
| `frappe` | Module | Frappe module with whitelisted methods |
| `frappe.utils` | Module | Utility functions (date, number formatting) |
| `_()` | Function | Translation function |
| `doc.meta` | Meta | DocType metadata |
| `doc.items` | List | Child table rows (by table fieldname) |

### Accessing Document Fields

```jinja
{# Standard fields #}
{{ doc.name }}
{{ doc.doctype }}
{{ doc.docstatus }}
{{ doc.owner }}
{{ doc.creation }}
{{ doc.modified }}

{# DocType-specific fields #}
{{ doc.customer_name }}
{{ doc.posting_date }}
{{ doc.grand_total }}

{# Child tables — access by fieldname #}
{% for row in doc.items %}
    {{ row.item_code }}
    {{ row.qty }}
    {{ row.rate }}
{% endfor %}

{# Formatted values — ALWAYS use for display #}
{{ doc.get_formatted("posting_date") }}
{{ doc.get_formatted("grand_total") }}
```

### Print-Specific Context

```jinja
{# Print heading override #}
{{ doc.select_print_heading or _("Invoice") }}

{# Language for this print #}
{{ doc.language or "en" }}

{# Letter head (if configured) #}
{{ doc.letter_head }}
```

---

## Email Templates

| Object | Type | Description |
|--------|------|-------------|
| `doc` | Document | The linked document (when template is used with a DocType) |
| `frappe` | Module | Frappe module (limited access) |
| `_()` | Function | Translation function |

### Email Context Specifics

```jinja
{# Document fields available directly #}
{{ doc.name }}
{{ doc.customer_name }}
{{ doc.get_formatted("grand_total") }}

{# Frappe methods available #}
{{ frappe.format_date(doc.posting_date) }}
{{ frappe.get_url() }}
{{ frappe.db.get_value("Company", doc.company, "company_name") }}
```

**NOTE**: Email templates render at send-time. The `doc` object reflects the document state at the moment the email is triggered.

---

## Notification Templates

| Object | Type | Description |
|--------|------|-------------|
| `doc` | Document | The document that triggered the notification |
| `frappe` | Module | Frappe module |
| `_()` | Function | Translation function |

### Notification Context

```jinja
{# Document that triggered the notification #}
{{ doc.name }}
{{ doc.doctype }}
{{ doc.modified_by }}

{# Common notification patterns #}
{{ _("{0} has been {1}").format(doc.name, doc.docstatus) }}
{{ frappe.get_fullname(doc.modified_by) }}
```

---

## Portal Pages (www/*.html)

| Object | Type | Description |
|--------|------|-------------|
| `frappe` | Module | Full Frappe module |
| `frappe.session.user` | String | Current authenticated user |
| `frappe.session.csrf_token` | String | CSRF token for forms |
| `frappe.form_dict` | Dict | URL query parameters |
| `frappe.lang` | String | Current language code (e.g., "en") |
| Custom context | Varies | Set via `get_context(context)` in `.py` controller |

### Standard Context Keys (Set in Controller)

| Key | Type | Effect |
|-----|------|--------|
| `title` | String | Page `<title>` and heading |
| `description` | String | Meta description |
| `image` | String | Meta image URL |
| `no_cache` | Boolean | Disable page caching |
| `no_breadcrumbs` | Boolean | Hide breadcrumbs |
| `no_header` | Boolean | Hide page header |
| `show_sidebar` | Boolean | Show web sidebar |
| `sitemap` | Boolean | Include in sitemap |
| `add_breadcrumbs` | Boolean | Auto-generate breadcrumbs |
| `add_next_prev_links` | Boolean | Show prev/next navigation |
| `safe_render` | Boolean | Enable/disable safe render mode |

### Portal Context Examples

```jinja
{# Authentication check #}
{% if frappe.session.user != "Guest" %}
    <p>{{ _("Welcome") }}, {{ frappe.get_fullname() }}</p>
{% else %}
    <a href="/login">{{ _("Log In") }}</a>
{% endif %}

{# CSRF token for forms #}
<form method="POST">
    <input type="hidden" name="csrf_token" value="{{ frappe.session.csrf_token }}">
    ...
</form>

{# Query parameters #}
{% if frappe.form_dict.search %}
    <p>{{ _("Search results for") }}: {{ frappe.form_dict.search }}</p>
{% endif %}

{# Custom context from controller #}
{% for project in projects %}
    <h3>{{ project.title }}</h3>
{% endfor %}
```

### Setting Context via Frontmatter

Portal pages support YAML frontmatter:

```html
---
title: My Page
no_cache: 1
sitemap: 1
---
<h1>{{ title }}</h1>
```

### Setting Context via HTML Comments

```html
<!-- add-breadcrumbs -->
<!-- no-header -->
<!-- no-cache -->
```

---

## Report Print Formats (NOT Jinja)

Report Print Formats use **JavaScript templating**, NOT Jinja.

| Object | Type | Description |
|--------|------|-------------|
| `data` | Array | Report data rows |
| `filters` | Object | Applied report filters |
| `report` | Object | Report configuration |

```html
{# JS Template — NOT Jinja #}
{% for(var i=0; i < data.length; i++) { %}
<tr>
    <td>{%= data[i].name %}</td>
    <td>{%= format_currency(data[i].amount) %}</td>
</tr>
{% } %}
```

**NEVER use `{{ }}` in Report Print Formats. ALWAYS use `{%= %}`.**

---

## frappe.render_template() Context

When using `frappe.render_template()` in Python, you control the context entirely:

```python
html = frappe.render_template(
    "templates/includes/invoice_row.html",
    {"doc": doc, "row": row, "company": company_doc}
)
```

The template receives exactly the variables you pass. `frappe` and `_()` are ALWAYS available automatically.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/context-objects.md*
