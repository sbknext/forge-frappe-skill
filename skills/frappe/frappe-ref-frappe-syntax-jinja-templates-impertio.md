---
title: Frappe Ref — Frappe Syntax: Jinja: Templates (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'templates']
source: Impertio
---

# Template Structure Patterns

> Structural patterns for Frappe Jinja templates: base templates, blocks, includes, and macros.

---

## Portal Page Base Template

ALL portal pages MUST extend `templates/web.html`:

```jinja
{% extends "templates/web.html" %}

{% block title %}{{ _("Page Title") }}{% endblock %}

{% block page_content %}
    {# Your page content here #}
{% endblock %}
```

### Available Blocks in web.html

| Block | Purpose |
|-------|---------|
| `title` | Page `<title>` tag |
| `page_content` | Main content area |
| `header` | Page header section |
| `script` | Additional JavaScript |
| `style` | Additional CSS |

### Extending with Additional Blocks

```jinja
{% extends "templates/web.html" %}

{% block title %}{{ _("Dashboard") }}{% endblock %}

{% block style %}
<style>
    .dashboard-card { border: 1px solid #ddd; padding: 15px; margin: 10px 0; }
</style>
{% endblock %}

{% block page_content %}
<div class="dashboard-card">
    <h2>{{ _("Overview") }}</h2>
</div>
{% endblock %}

{% block script %}
<script>
    // Page-specific JavaScript
    frappe.ready(function() {
        console.log("Dashboard loaded");
    });
</script>
{% endblock %}
```

---

## Print Format Template Structure

Print Formats do NOT use `{% extends %}`. They are standalone HTML fragments:

```jinja
<style>
    /* Scoped styles for this print format */
    .print-container { font-family: Arial, sans-serif; }
    .header { margin-bottom: 20px; }
    .item-table { width: 100%; border-collapse: collapse; }
    .item-table th, .item-table td { border: 1px solid #ccc; padding: 6px; }
    .text-right { text-align: right; }
    .footer { margin-top: 30px; font-size: 0.9em; color: #666; }

    /* Page break control */
    .page-break { page-break-before: always; break-before: page; }

    /* Print-only styles */
    @media print {
        .no-print { display: none; }
    }
</style>

<div class="print-container">
    {# Header section #}
    <div class="header">
        <h1>{{ doc.select_print_heading or _("Document Title") }}</h1>
        <p>{{ doc.name }} — {{ doc.get_formatted("posting_date") }}</p>
    </div>

    {# Body section #}
    <table class="item-table">
        <thead>...</thead>
        <tbody>
            {% for row in doc.items %}
            <tr>...</tr>
            {% endfor %}
        </tbody>
    </table>

    {# Totals section #}
    <div class="totals">...</div>

    {# Footer section #}
    <div class="footer">
        <p>{{ _("Prepared by") }}: {{ frappe.get_fullname(doc.owner) }}</p>
    </div>
</div>
```

---

## Include Pattern

Use `{% include %}` to reuse template fragments:

```jinja
{# Main template #}
<div class="header">
    {% include "templates/includes/company_header.html" %}
</div>

<div class="content">
    {% include "templates/includes/item_table.html" %}
</div>

{# With ignore missing (no error if file absent) #}
{% include "templates/includes/optional_footer.html" ignore missing %}
```

### Included Template

```jinja
{# templates/includes/item_table.html #}
{# Receives the same context as the parent template #}
<table class="item-table">
    <thead>
        <tr>
            <th>#</th>
            <th>{{ _("Item") }}</th>
            <th class="text-right">{{ _("Amount") }}</th>
        </tr>
    </thead>
    <tbody>
        {% for row in doc.items %}
        <tr>
            <td>{{ loop.index }}</td>
            <td>{{ row.item_name }}</td>
            <td class="text-right">{{ row.get_formatted("amount", doc) }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>
```

---

## Macro Pattern

Macros define reusable template functions within a single file:

```jinja
{# Define macros at the top of the template #}
{% macro field_row(label, value) %}
<tr>
    <td style="padding: 5px 10px;"><strong>{{ _(label) }}</strong></td>
    <td style="padding: 5px 10px;">{{ value | default("—") }}</td>
</tr>
{% endmacro %}

{% macro status_badge(status) %}
<span class="badge badge-{{ 'success' if status == 'Completed' else 'primary' if status == 'Open' else 'default' }}">
    {{ status }}
</span>
{% endmacro %}

{# Use macros in the template body #}
<table>
    {{ field_row("Customer", doc.customer_name) }}
    {{ field_row("Date", doc.get_formatted("posting_date")) }}
    {{ field_row("Status", status_badge(doc.status)) }}
    {{ field_row("Total", doc.get_formatted("grand_total")) }}
</table>
```

### Macro with Caller Block

```jinja
{% macro card(title) %}
<div class="card" style="border: 1px solid #ddd; padding: 15px; margin: 10px 0;">
    <h3>{{ title }}</h3>
    <div class="card-body">
        {{ caller() }}
    </div>
</div>
{% endmacro %}

{% call card(_("Summary")) %}
    <p>{{ _("Total Items") }}: {{ doc.items | length }}</p>
    <p>{{ _("Grand Total") }}: {{ doc.get_formatted("grand_total") }}</p>
{% endcall %}
```

---

## Email Template Structure

Email templates are HTML fragments (no `{% extends %}`). ALWAYS use inline styles for email compatibility:

```jinja
{# Greeting #}
<p style="font-size: 14px;">{{ _("Dear") }} {{ doc.customer_name }},</p>

{# Body with inline styles #}
<div style="margin: 20px 0; padding: 15px; background: #f9f9f9; border-radius: 4px;">
    <p style="margin: 0;">
        <strong>{{ _("Invoice") }}:</strong> {{ doc.name }}<br>
        <strong>{{ _("Amount") }}:</strong> {{ doc.get_formatted("grand_total") }}<br>
        <strong>{{ _("Due Date") }}:</strong> {{ frappe.format_date(doc.due_date) }}
    </p>
</div>

{# Call to action #}
<p>
    <a href="{{ frappe.get_url() }}/app/sales-invoice/{{ doc.name }}"
       style="background: #5e64ff; color: white; padding: 10px 20px; text-decoration: none; border-radius: 4px;">
        {{ _("View Invoice") }}
    </a>
</p>

{# Sign-off #}
<p style="color: #666; font-size: 12px;">
    {{ _("Best regards") }},<br>
    {{ frappe.db.get_value("Company", doc.company, "company_name") }}
</p>
```

**Rule**: ALWAYS use inline CSS in email templates. External stylesheets and `<style>` blocks are stripped by most email clients.

---

## Notification Template Structure

Notification templates are short HTML fragments:

```jinja
<p>{{ _("{0} has been updated").format(doc.name) }}</p>

<table style="border-collapse: collapse;">
    <tr>
        <td style="padding: 4px 8px;"><strong>{{ _("Status") }}:</strong></td>
        <td style="padding: 4px 8px;">{{ doc.status }}</td>
    </tr>
    <tr>
        <td style="padding: 4px 8px;"><strong>{{ _("Modified by") }}:</strong></td>
        <td style="padding: 4px 8px;">{{ frappe.get_fullname(doc.modified_by) }}</td>
    </tr>
</table>

<p><a href="{{ frappe.get_url() }}/app/{{ doc.doctype | lower | replace(' ', '-') }}/{{ doc.name }}">
    {{ _("View Document") }}
</a></p>
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/templates.md*
