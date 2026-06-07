---
title: Frappe Ref — Frappe Impl: Website: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'website', 'reference', 'examples']
source: Impertio
---

# Website Examples — Complete Code

## Example 1: Simple Static Portal Page

```
myapp/www/
├── about.html
├── about.py
└── about.css
```

```html
<!-- about.html -->
{% extends "templates/web.html" %}
{% block page_content %}
<div class="container my-5">
  <h1>{{ title }}</h1>
  <p class="lead">{{ tagline }}</p>
  <div>{{ description }}</div>
</div>
{% endblock %}
```

```python
# about.py
import frappe

def get_context(context):
    settings = frappe.get_doc("About Us Settings")
    context.title = "About Us"
    context.tagline = settings.company_description
    context.description = settings.company_history
    context.metatags = {
        "title": "About Us | My Company",
        "description": settings.company_description[:160],
    }
```

## Example 2: Web Form with Conditional Fields

Create via UI: Web Form → "Job Application"

Fields:
- `applicant_name` (Data, reqd)
- `email` (Data, reqd, options: Email)
- `position` (Link, options: Job Opening, reqd)
- `cover_letter` (Text Editor)
- `resume` (Attach)
- `experience_years` (Int)

Client Script:
```javascript
frappe.web_form.on("after_load", function() {
    // Hide experience field for internship positions
    frappe.web_form.on("position", function() {
        let pos = frappe.web_form.get_value("position");
        frappe.call({
            method: "frappe.client.get_value",
            args: {
                doctype: "Job Opening",
                filters: { name: pos },
                fieldname: "designation"
            },
            callback: function(r) {
                if (r.message && r.message.designation === "Intern") {
                    frappe.web_form.set_df_property("experience_years", "hidden", 1);
                } else {
                    frappe.web_form.set_df_property("experience_years", "hidden", 0);
                }
            }
        });
    });
});
```

## Example 3: SEO-Optimized Blog Configuration

```python
# hooks.py
website_context = {
    "favicon": "/assets/myapp/images/favicon.ico",
}

update_website_context = "myapp.website.context"

website_route_rules = [
    {"from_route": "/blog/category/<category>", "to_route": "blog_category"},
]
```

```python
# myapp/website.py
def context(ctx):
    ctx.metatags = ctx.get("metatags", {})
    ctx.metatags.setdefault("og:site_name", "My Company Blog")
    ctx.metatags.setdefault("twitter:site", "@mycompany")
```

## Example 4: Protected Portal with Sidebar

```python
# myapp/www/portal/index.py
import frappe

def get_context(context):
    if frappe.session.user == "Guest":
        frappe.local.flags.redirect_location = "/login"
        raise frappe.Redirect

    context.no_cache = 1
    context.show_sidebar = True
    context.title = "Customer Portal"
    context.sidebar_items = [
        {"label": "Orders", "url": "/portal/orders", "active": True},
        {"label": "Invoices", "url": "/portal/invoices"},
        {"label": "Support", "url": "/portal/tickets"},
    ]
```

```html
<!-- myapp/www/portal/index.html -->
{% extends "templates/web.html" %}
{% block page_content %}
<div class="portal-welcome">
  <h2>{{ _("Welcome, {0}").format(frappe.session.user) }}</h2>
  <div class="row">
    {% for item in sidebar_items %}
    <div class="col-md-4">
      <a href="{{ item.url }}" class="card p-3">
        <h4>{{ item.label }}</h4>
      </a>
    </div>
    {% endfor %}
  </div>
</div>
{% endblock %}
```

## Example 5: Website Settings Configuration

Via the Website Settings DocType:
- **Home Page**: `home` (route of homepage)
- **Brand Image**: Company logo for navbar
- **Favicon**: Browser tab icon
- **Navbar Items**: Top Bar Item child table (label + URL)
- **Footer Items**: Footer links child table
- **Banner HTML**: Custom HTML above navbar
- **Head HTML**: Injected into `<head>` (analytics, custom fonts)
- **Footer Address**: Company address in footer
- **Google Analytics ID**: UA-XXXXX or G-XXXXX

## Example 6: Custom 404 Page

```python
# hooks.py
website_catch_all = "myapp.www.not_found.handler"

# myapp/www/not_found.py
import frappe

def handler(path):
    return frappe.get_template("myapp/www/404.html").render({
        "path": path,
        "title": "Page Not Found",
    })
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-website/references/examples.md*
