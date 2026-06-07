---
title: Frappe Ref — Frappe Syntax: Jinja: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'jinja', 'reference', 'examples']
source: Impertio
---

# Complete Jinja Examples

> Working examples for Print Formats, Email Templates, Notification Templates, and Portal Pages.

---

## Print Format: Sales Invoice

```jinja
<style>
    .invoice-header { background: #f5f5f5; padding: 15px; margin-bottom: 20px; }
    .text-right { text-align: right; }
    .item-table { width: 100%; border-collapse: collapse; margin: 20px 0; }
    .item-table th, .item-table td { border: 1px solid #ddd; padding: 8px; }
    .item-table th { background: #f9f9f9; }
    .totals { margin-top: 20px; }
    .terms { margin-top: 30px; padding-top: 20px; border-top: 1px solid #ddd; }
</style>

<div class="invoice-header">
    <div class="row">
        <div class="col-md-6">
            <h1>{{ doc.select_print_heading or _("Invoice") }}</h1>
            <p><strong>{{ doc.name }}</strong></p>
        </div>
        <div class="col-md-6 text-right">
            <p><strong>{{ _("Date") }}:</strong> {{ doc.get_formatted("posting_date") }}</p>
            <p><strong>{{ _("Due Date") }}:</strong> {{ doc.get_formatted("due_date") }}</p>
        </div>
    </div>
</div>

<div class="row">
    <div class="col-md-6">
        <h4>{{ _("Bill To") }}</h4>
        <p><strong>{{ doc.customer_name }}</strong></p>
        {% if doc.address_display %}
            {{ doc.address_display | safe }}
        {% endif %}
    </div>
    <div class="col-md-6 text-right">
        {% set company_name = frappe.db.get_value("Company", doc.company, "company_name") %}
        <h4>{{ _("From") }}</h4>
        <p><strong>{{ company_name }}</strong></p>
    </div>
</div>

<table class="item-table">
    <thead>
        <tr>
            <th style="width: 5%">#</th>
            <th style="width: 35%">{{ _("Item") }}</th>
            <th style="width: 25%">{{ _("Description") }}</th>
            <th class="text-right" style="width: 10%">{{ _("Qty") }}</th>
            <th class="text-right" style="width: 12%">{{ _("Rate") }}</th>
            <th class="text-right" style="width: 13%">{{ _("Amount") }}</th>
        </tr>
    </thead>
    <tbody>
        {%- for row in doc.items -%}
        <tr>
            <td>{{ loop.index }}</td>
            <td>
                {{ row.item_name }}
                {% if row.item_code != row.item_name -%}
                    <br><small>{{ _("Item Code") }}: {{ row.item_code }}</small>
                {%- endif %}
            </td>
            <td>{{ row.description | default("") | striptags | truncate(100) }}</td>
            <td class="text-right">{{ row.qty }} {{ row.uom | default(row.stock_uom) }}</td>
            <td class="text-right">{{ row.get_formatted("rate", doc) }}</td>
            <td class="text-right">{{ row.get_formatted("amount", doc) }}</td>
        </tr>
        {%- endfor -%}
    </tbody>
</table>

<div class="row totals">
    <div class="col-md-6"></div>
    <div class="col-md-6">
        <table class="item-table">
            <tr>
                <td><strong>{{ _("Net Total") }}</strong></td>
                <td class="text-right">{{ doc.get_formatted("net_total") }}</td>
            </tr>
            {% if doc.total_taxes_and_charges %}
            <tr>
                <td><strong>{{ _("Taxes") }}</strong></td>
                <td class="text-right">{{ doc.get_formatted("total_taxes_and_charges") }}</td>
            </tr>
            {% endif %}
            {% if doc.discount_amount %}
            <tr>
                <td><strong>{{ _("Discount") }}</strong></td>
                <td class="text-right">-{{ doc.get_formatted("discount_amount") }}</td>
            </tr>
            {% endif %}
            <tr style="font-size: 1.2em;">
                <td><strong>{{ _("Grand Total") }}</strong></td>
                <td class="text-right"><strong>{{ doc.get_formatted("grand_total") }}</strong></td>
            </tr>
        </table>
    </div>
</div>

{% if doc.terms %}
<div class="terms">
    <h4>{{ _("Terms and Conditions") }}</h4>
    {{ doc.terms | safe }}
</div>
{% endif %}

<div class="row" style="margin-top: 50px;">
    <div class="col-md-6">
        <p>{{ _("Prepared by") }}: {{ frappe.get_fullname(doc.owner) }}</p>
    </div>
    <div class="col-md-6 text-right">
        <p>{{ _("Printed on") }}: {{ frappe.format_date(frappe.utils.nowdate()) }}</p>
    </div>
</div>
```

---

## Print Format: Delivery Note with Page Breaks

```jinja
<style>
    .page-break { page-break-before: always; break-before: page; }
</style>

{# Page 1: Header and items #}
<h1>{{ _("Delivery Note") }} — {{ doc.name }}</h1>
<p>{{ _("Date") }}: {{ doc.get_formatted("posting_date") }}</p>

<table class="item-table">
    <thead>
        <tr>
            <th>#</th>
            <th>{{ _("Item") }}</th>
            <th class="text-right">{{ _("Qty") }}</th>
        </tr>
    </thead>
    <tbody>
        {% for row in doc.items %}
        <tr>
            <td>{{ loop.index }}</td>
            <td>{{ row.item_name }}</td>
            <td class="text-right">{{ row.qty }} {{ row.uom | default("") }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>

{# Page 2: Signature block #}
<div class="page-break"></div>
<h2>{{ _("Acknowledgement") }}</h2>
<p>{{ _("Received by") }}: ____________________</p>
<p>{{ _("Date") }}: ____________________</p>
<p>{{ _("Signature") }}: ____________________</p>
```

---

## Email Template: Payment Reminder

```jinja
<p>{{ _("Dear") }} {{ doc.customer_name }},</p>

<p>{{ _("This is a friendly reminder that invoice") }} <strong>{{ doc.name }}</strong>
{{ _("for") }} {{ doc.get_formatted("grand_total") }} {{ _("is now due for payment.") }}</p>

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
    <tr>
        <td style="padding: 8px; border: 1px solid #ddd;">
            <strong>{{ _("Invoice Number") }}</strong>
        </td>
        <td style="padding: 8px; border: 1px solid #ddd;">{{ doc.name }}</td>
    </tr>
    <tr>
        <td style="padding: 8px; border: 1px solid #ddd;">
            <strong>{{ _("Due Date") }}</strong>
        </td>
        <td style="padding: 8px; border: 1px solid #ddd;">
            {{ frappe.format_date(doc.due_date) }}
        </td>
    </tr>
    <tr>
        <td style="padding: 8px; border: 1px solid #ddd;">
            <strong>{{ _("Amount Due") }}</strong>
        </td>
        <td style="padding: 8px; border: 1px solid #ddd;">
            <strong>{{ doc.get_formatted("outstanding_amount") }}</strong>
        </td>
    </tr>
</table>

{% if doc.items %}
<p><strong>{{ _("Items") }}:</strong></p>
<ul>
{% for item in doc.items %}
    <li>{{ item.item_name }} — {{ item.qty }} x {{ item.get_formatted("rate", doc) }}</li>
{% endfor %}
</ul>
{% endif %}

<p>{{ _("Please make payment at your earliest convenience.") }}</p>

<p>{{ _("Best regards") }},<br>
{{ frappe.db.get_value("Company", doc.company, "company_name") }}</p>
```

---

## Email Template: Welcome New Customer

```jinja
<p>{{ _("Dear") }} {{ doc.customer_name }},</p>

<p>{{ _("Welcome! Your account has been created.") }}</p>

<p>{{ _("Your customer ID is") }}: <strong>{{ doc.name }}</strong></p>

{% set contact = frappe.db.get_value("Dynamic Link",
    {"link_doctype": "Customer", "link_name": doc.name, "parenttype": "Contact"},
    "parent") %}

{% if contact %}
    {% set email = frappe.db.get_value("Contact", contact, "email_id") %}
    <p>{{ _("Contact email") }}: {{ email | default(_("Not set")) }}</p>
{% endif %}

<p><a href="{{ frappe.get_url() }}/me">{{ _("Log in to your portal") }}</a></p>

<p>{{ _("Best regards") }},<br>
{{ frappe.db.get_value("Company", doc.company, "company_name") }}</p>
```

---

## Notification Template: Task Overdue

```jinja
<p>{{ _("Task") }} <strong>{{ doc.name }}</strong> {{ _("is overdue.") }}</p>

<table style="border-collapse: collapse; margin: 10px 0;">
    <tr>
        <td style="padding: 5px 10px;"><strong>{{ _("Subject") }}:</strong></td>
        <td style="padding: 5px 10px;">{{ doc.subject }}</td>
    </tr>
    <tr>
        <td style="padding: 5px 10px;"><strong>{{ _("Due Date") }}:</strong></td>
        <td style="padding: 5px 10px;">{{ frappe.format_date(doc.exp_end_date) }}</td>
    </tr>
    <tr>
        <td style="padding: 5px 10px;"><strong>{{ _("Assigned To") }}:</strong></td>
        <td style="padding: 5px 10px;">{{ frappe.get_fullname(doc.owner) }}</td>
    </tr>
</table>

<p><a href="{{ frappe.get_url() }}/app/task/{{ doc.name }}">{{ _("View Task") }}</a></p>
```

---

## Portal Page: Customer Orders

### www/orders/index.html

```jinja
{% extends "templates/web.html" %}

{% block title %}{{ _("My Orders") }}{% endblock %}

{% block page_content %}
<div class="container">
    <h1>{{ _("My Orders") }}</h1>

    {% if frappe.session.user == "Guest" %}
        <p>{{ _("Please log in to view your orders.") }}</p>
        <a href="/login" class="btn btn-primary">{{ _("Log In") }}</a>
    {% else %}
        <p class="text-muted">{{ _("Welcome") }}, {{ frappe.get_fullname() }}</p>

        {% if orders %}
        <table class="table">
            <thead>
                <tr>
                    <th>{{ _("Order") }}</th>
                    <th>{{ _("Date") }}</th>
                    <th>{{ _("Status") }}</th>
                    <th class="text-right">{{ _("Total") }}</th>
                </tr>
            </thead>
            <tbody>
                {% for order in orders %}
                <tr>
                    <td><a href="/orders/{{ order.name }}">{{ order.name }}</a></td>
                    <td>{{ frappe.format_date(order.transaction_date) }}</td>
                    <td>
                        <span class="badge badge-{{ 'success' if order.status == 'Completed' else 'warning' }}">
                            {{ order.status }}
                        </span>
                    </td>
                    <td class="text-right">{{ frappe.format(order.grand_total, {"fieldtype": "Currency"}) }}</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
        {% else %}
            <p class="text-muted">{{ _("No orders found.") }}</p>
        {% endif %}
    {% endif %}
</div>
{% endblock %}
```

### www/orders/index.py

```python
import frappe

def get_context(context):
    context.title = "My Orders"
    context.no_cache = True

    if frappe.session.user == "Guest":
        context.orders = []
        return context

    customer = frappe.db.get_value("Customer",
        {"email_id": frappe.session.user}, "name")

    if customer:
        context.orders = frappe.get_list("Sales Order",
            filters={"customer": customer},
            fields=["name", "transaction_date", "status", "grand_total"],
            order_by="transaction_date desc",
            limit_page_length=20)
    else:
        context.orders = []

    return context
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-jinja/references/examples.md*
