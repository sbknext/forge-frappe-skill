---
title: Frappe Ref — Frappe Impl: Website: Workflows (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'website', 'reference', 'workflows']
source: Impertio
---

# Website Workflows — Extended

## Complete Portal Page with Authentication

```python
# myapp/www/dashboard.py
import frappe

def get_context(context):
    # ALWAYS check login for protected pages
    if frappe.session.user == "Guest":
        frappe.throw("Please log in", frappe.AuthenticationError)

    context.no_cache = 1
    context.show_sidebar = True
    context.title = "My Dashboard"

    user = frappe.session.user
    context.orders = frappe.get_all(
        "Sales Order",
        filters={"owner": user, "docstatus": 1},
        fields=["name", "transaction_date", "grand_total", "status"],
        order_by="transaction_date desc",
        limit=20,
    )
```

```html
<!-- myapp/www/dashboard.html -->
{% extends "templates/web.html" %}

{% block page_content %}
<div class="container">
  <h2>{{ _("My Orders") }}</h2>
  {% if orders %}
  <table class="table">
    <thead>
      <tr>
        <th>{{ _("Order") }}</th>
        <th>{{ _("Date") }}</th>
        <th>{{ _("Total") }}</th>
        <th>{{ _("Status") }}</th>
      </tr>
    </thead>
    <tbody>
      {% for order in orders %}
      <tr>
        <td><a href="/app/sales-order/{{ order.name }}">{{ order.name }}</a></td>
        <td>{{ frappe.format_date(order.transaction_date) }}</td>
        <td>{{ frappe.format_currency(order.grand_total) }}</td>
        <td>{{ order.status }}</td>
      </tr>
      {% endfor %}
    </tbody>
  </table>
  {% else %}
  <p class="text-muted">{{ _("No orders found.") }}</p>
  {% endif %}
</div>
{% endblock %}
```

## Web Form with File Upload and Redirect

1. Create Web Form "Support Ticket" linked to DocType "Issue"
2. Add fields: `subject` (Data), `description` (Text Editor), `attachment` (Attach)
3. Set **Success URL** to `/thank-you`
4. Enable **Allow Attachment**
5. Client Script:

```javascript
frappe.web_form.on("before_submit", function() {
    let subject = frappe.web_form.get_value("subject");
    if (subject && subject.length < 5) {
        frappe.msgprint("Subject must be at least 5 characters");
        return false;
    }
});
```

## has_web_view Complete Setup

### Step 1: DocType Configuration
- Check: Has Web View, Allow Guest to View
- Route prefix: `articles`
- Add fields: `route` (Data, hidden), `published` (Check), `meta_title` (Data), `meta_description` (Small Text)

### Step 2: Controller

```python
# myapp/myapp/doctype/article/article.py
import frappe
from frappe.website.website_generator import WebsiteGenerator

class Article(WebsiteGenerator):
    website = frappe._dict(
        template="templates/generators/article.html",
        condition_field="published",
        page_title_field="title",
    )

    def get_context(self, context):
        context.metatags = {
            "title": self.meta_title or self.title,
            "description": self.meta_description or self.title,
        }
        # Related articles
        context.related = frappe.get_all(
            "Article",
            filters={"published": 1, "name": ("!=", self.name)},
            fields=["title", "route", "published_on"],
            order_by="published_on desc",
            limit=3,
        )
```

### Step 3: Templates

```html
<!-- myapp/templates/generators/article.html -->
{% extends "templates/web.html" %}

{% block page_content %}
<article>
  <h1>{{ doc.title }}</h1>
  <p class="text-muted">{{ frappe.format_date(doc.published_on) }}</p>
  <div>{{ doc.content }}</div>
</article>

{% if related %}
<h3>{{ _("Related Articles") }}</h3>
<ul>
  {% for r in related %}
  <li><a href="/{{ r.route }}">{{ r.title }}</a></li>
  {% endfor %}
</ul>
{% endif %}
{% endblock %}
```

### Step 4: hooks.py

```python
website_generators = ["Article"]
```

## Website Route Rules — Dynamic Pages

```python
# hooks.py
website_route_rules = [
    {"from_route": "/catalog/<category>", "to_route": "catalog"},
    {"from_route": "/catalog/<category>/<item>", "to_route": "catalog/item"},
]
```

```python
# myapp/www/catalog.py
import frappe

def get_context(context):
    category = frappe.form_dict.get("category")
    if not category:
        frappe.throw("Category not found", frappe.DoesNotExistError)

    context.category = frappe.get_doc("Item Group", category)
    context.items = frappe.get_all(
        "Item",
        filters={"item_group": category, "show_in_website": 1},
        fields=["item_name", "route", "image", "description"],
    )
    context.title = context.category.name
    context.no_cache = 1
```

## Website Context Hooks

```python
# hooks.py — inject global context
website_context = {
    "favicon": "/assets/myapp/images/favicon.ico",
    "splash_image": "/assets/myapp/images/logo.svg",
}

# For dynamic context
update_website_context = "myapp.overrides.website_context"

# myapp/overrides.py
def website_context(context):
    context.company_name = frappe.db.get_single_value("Website Settings", "app_name")
    context.footer_links = frappe.get_all(
        "Top Bar Item",
        filters={"parent": "Website Settings", "parentfield": "footer_items"},
        fields=["label", "url"],
    )
```

## Override Base Template

```python
# hooks.py — global override
base_template = "myapp/templates/custom_base.html"

# Route-specific override
base_template_map = {
    r"docs.*": "myapp/templates/docs_base.html",
    r"blog.*": "myapp/templates/blog_base.html",
}
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-website/references/workflows.md*
