---
title: Frappe Ref — Frappe Syntax: Jinja: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'patterns']
source: Impertio
---

# Common Jinja Patterns in Frappe

> Reusable patterns for conditional rendering, loops, child tables, grouping, and data display.

---

## Conditional Rendering

### Status-Based Styling

```jinja
{% set status_class = {
    "Paid": "success",
    "Unpaid": "warning",
    "Overdue": "danger",
    "Cancelled": "default"
} %}

<span class="label label-{{ status_class.get(doc.status, 'default') }}">
    {{ doc.status }}
</span>
```

### Show/Hide Sections

```jinja
{# Show section only if data exists #}
{% if doc.terms %}
<div class="terms-section">
    <h4>{{ _("Terms and Conditions") }}</h4>
    {{ doc.terms | safe }}
</div>
{% endif %}

{# Show section only for specific doctypes #}
{% if doc.doctype == "Sales Invoice" and doc.is_return %}
<div class="return-notice" style="color: red;">
    <strong>{{ _("CREDIT NOTE") }}</strong>
</div>
{% endif %}
```

### Permission-Based Content (Portal Pages)

```jinja
{% if frappe.session.user != "Guest" %}
    <div class="user-content">
        <p>{{ _("Welcome") }}, {{ frappe.get_fullname() }}</p>
    </div>
{% else %}
    <a href="/login">{{ _("Log in to continue") }}</a>
{% endif %}
```

---

## Child Table Iteration

### Basic Child Table Loop

```jinja
{% for row in doc.items %}
<tr>
    <td>{{ loop.index }}</td>
    <td>{{ row.item_name }}</td>
    <td class="text-right">{{ row.qty }}</td>
    <td class="text-right">{{ row.get_formatted("rate", doc) }}</td>
    <td class="text-right">{{ row.get_formatted("amount", doc) }}</td>
</tr>
{% else %}
<tr>
    <td colspan="5">{{ _("No items") }}</td>
</tr>
{% endfor %}
```

### Alternating Row Colors

```jinja
{% for row in doc.items %}
<tr style="background: {{ '#f9f9f9' if loop.index is even else '#ffffff' }};">
    <td>{{ row.item_name }}</td>
</tr>
{% endfor %}
```

### First/Last Row Styling

```jinja
{% for row in doc.items %}
<tr class="{% if loop.first %}first-row{% endif %}{% if loop.last %} last-row{% endif %}">
    <td>{{ row.item_name }}</td>
</tr>
{% endfor %}
```

### Multiple Child Tables

```jinja
{# Items table #}
<h3>{{ _("Items") }}</h3>
{% for row in doc.items %}
    <p>{{ row.item_name }}: {{ row.get_formatted("amount", doc) }}</p>
{% endfor %}

{# Taxes table #}
<h3>{{ _("Taxes") }}</h3>
{% for tax in doc.taxes %}
    <p>{{ tax.description }}: {{ tax.get_formatted("tax_amount", doc) }}</p>
{% endfor %}

{# Payment schedule #}
{% if doc.payment_schedule %}
<h3>{{ _("Payment Schedule") }}</h3>
{% for ps in doc.payment_schedule %}
    <p>{{ frappe.format_date(ps.due_date) }}: {{ ps.get_formatted("payment_amount", doc) }}</p>
{% endfor %}
{% endif %}
```

---

## Accumulation Patterns

### Running Total with Namespace

```jinja
{% set ns = namespace(total=0, qty_total=0) %}
{% for row in doc.items %}
<tr>
    <td>{{ row.item_name }}</td>
    <td>{{ row.qty }}</td>
    <td>{{ row.get_formatted("amount", doc) }}</td>
</tr>
{% set ns.total = ns.total + row.amount %}
{% set ns.qty_total = ns.qty_total + row.qty %}
{% endfor %}
<tr>
    <td><strong>{{ _("Total") }}</strong></td>
    <td><strong>{{ ns.qty_total }}</strong></td>
    <td><strong>{{ frappe.format(ns.total, {"fieldtype": "Currency"}) }}</strong></td>
</tr>
```

**NOTE**: ALWAYS use `namespace()` for accumulation. Plain `{% set total = total + x %}` does NOT work inside loops due to Jinja scoping.

---

## Grouping Patterns

### Group Items by Category

```jinja
{% set groups = {} %}
{% for row in doc.items %}
    {% if row.item_group not in groups %}
        {% set _ = groups.update({row.item_group: []}) %}
    {% endif %}
    {% set _ = groups[row.item_group].append(row) %}
{% endfor %}

{% for group_name, items in groups.items() %}
<h4>{{ group_name }}</h4>
<table class="item-table">
    {% for row in items %}
    <tr>
        <td>{{ row.item_name }}</td>
        <td class="text-right">{{ row.get_formatted("amount", doc) }}</td>
    </tr>
    {% endfor %}
</table>
{% endfor %}
```

### Using groupby Filter

```jinja
{% for group in doc.items | groupby("item_group") %}
<h4>{{ group.grouper | default(_("Uncategorized")) }}</h4>
<ul>
    {% for row in group.list %}
    <li>{{ row.item_name }} — {{ row.get_formatted("amount", doc) }}</li>
    {% endfor %}
</ul>
{% endfor %}
```

---

## Data Display Patterns

### Key-Value Table

```jinja
{% macro kv_row(label, value) %}
{% if value %}
<tr>
    <td style="width: 40%; padding: 5px;"><strong>{{ _(label) }}</strong></td>
    <td style="padding: 5px;">{{ value }}</td>
</tr>
{% endif %}
{% endmacro %}

<table style="width: 100%;">
    {{ kv_row("Customer", doc.customer_name) }}
    {{ kv_row("Date", doc.get_formatted("posting_date")) }}
    {{ kv_row("Status", doc.status) }}
    {{ kv_row("Total", doc.get_formatted("grand_total")) }}
    {{ kv_row("Notes", doc.notes | default("")) }}
</table>
```

### Two-Column Layout

```jinja
<div class="row">
    <div class="col-md-6">
        <h4>{{ _("Bill To") }}</h4>
        <p>{{ doc.customer_name }}</p>
        {% if doc.address_display %}
            {{ doc.address_display | safe }}
        {% endif %}
    </div>
    <div class="col-md-6 text-right">
        <h4>{{ _("Ship To") }}</h4>
        {% if doc.shipping_address %}
            {{ doc.shipping_address | safe }}
        {% else %}
            <p>{{ _("Same as billing address") }}</p>
        {% endif %}
    </div>
</div>
```

### Grid Layout with batch Filter

```jinja
{# Display items in 3-column grid #}
{% for row_items in doc.items | batch(3) %}
<div class="row" style="margin-bottom: 10px;">
    {% for item in row_items %}
    <div class="col-md-4">
        <div style="border: 1px solid #ddd; padding: 10px;">
            <strong>{{ item.item_name }}</strong>
            <p>{{ item.get_formatted("rate", doc) }}</p>
        </div>
    </div>
    {% endfor %}
</div>
{% endfor %}
```

---

## Date & Number Patterns

### Date Comparisons

```jinja
{% if doc.due_date and doc.due_date < frappe.utils.nowdate() %}
<span style="color: red;">{{ _("OVERDUE") }}</span>
{% endif %}
```

### Number Formatting

```jinja
{# ALWAYS prefer get_formatted for display #}
{{ doc.get_formatted("grand_total") }}

{# For calculated values not in a field #}
{{ frappe.format(calculated_value, {"fieldtype": "Currency"}) }}

{# Percentage display #}
{{ doc.discount_percentage | round(1) }}%
```

---

## Link Patterns

### Document Links (Portal)

```jinja
<a href="{{ frappe.get_url() }}/app/sales-invoice/{{ doc.name }}">
    {{ _("View Invoice") }}
</a>
```

### Portal Page Links

```jinja
<a href="/orders/{{ order.name }}">{{ order.name }}</a>
```

### Conditional Links

```jinja
{% if doc.customer %}
    {% set customer_url = frappe.get_url() ~ "/app/customer/" ~ doc.customer %}
    <a href="{{ customer_url }}">{{ doc.customer_name }}</a>
{% else %}
    {{ _("No customer linked") }}
{% endif %}
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/patterns.md*
