---
title: Frappe Ref — Frappe Impl: Website: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'website', 'reference', 'anti-patterns']
source: Impertio
---

# Website Anti-Patterns

## AP-1: Hard-coding HTML in get_context

**Wrong**:
```python
def get_context(context):
    context.content = "<h1>Welcome</h1><p>Hello " + user + "</p>"
```

**Correct**: Use Jinja templates for all HTML rendering. `get_context` provides data only.
```python
def get_context(context):
    context.user_name = frappe.get_value("User", frappe.session.user, "full_name")
```

## AP-2: Missing no_cache on Dynamic Pages

**Wrong**: Serving user-specific content without `no_cache`.
```python
def get_context(context):
    context.my_orders = get_user_orders()  # User-specific but cached!
```

**Correct**: ALWAYS set `no_cache = 1` when content varies per user or request.
```python
def get_context(context):
    context.no_cache = 1
    context.my_orders = get_user_orders()
```

## AP-3: Guest Web Form Without Rate Limiting

**Wrong**: Allowing guest submissions with no protection.

**Correct**: ALWAYS configure rate limits in `site_config.json`:
```json
{ "rate_limit": { "web_form": "5/hour" } }
```

## AP-4: Missing Published Field on has_web_view

**Wrong**: Enabling Has Web View without a `published` field — ALL records become public.

**Correct**: ALWAYS add a `published` (Check) field and set `condition_field="published"` in the WebsiteGenerator config.

## AP-5: Using website_route_rules for Simple Redirects

**Wrong**:
```python
website_route_rules = [
    {"from_route": "/old-page", "to_route": "new-page"},
]
```

**Correct**: Use `website_redirects` for URL redirects (returns proper 301/304):
```python
website_redirects = [
    {"source": "/old-page", "target": "/new-page"},
]
```

## AP-6: Business Logic in www/ Controllers

**Wrong**: Putting validation, calculations, or data mutations in `www/*.py`.

**Correct**: Keep business logic in DocType controllers or whitelisted methods. The www/ controller is for presentation context only.

## AP-7: Skipping CSRF Protection

**Wrong**: Setting `ignore_csrf` in production site_config.

**Correct**: NEVER disable CSRF in production. If a third-party needs POST access, use `allowed_referrers` or API keys instead.

## AP-8: Not Extending templates/web.html

**Wrong**: Writing standalone HTML without extending the base template.
```html
<html><body><h1>My Page</h1></body></html>
```

**Correct**: ALWAYS extend `templates/web.html` for consistent navbar, footer, and asset loading:
```html
{% extends "templates/web.html" %}
{% block page_content %}
<h1>My Page</h1>
{% endblock %}
```

## AP-9: Exposing Internal Routes to Guests

**Wrong**: Using `website_route_rules` that expose DocType names without checking permissions.

**Correct**: ALWAYS verify permissions in `get_context`:
```python
def get_context(context):
    if not frappe.has_permission("Project", "read"):
        frappe.throw("Not Permitted", frappe.PermissionError)
```

## AP-10: Forgetting sitemap Exclusion for Private Pages

**Wrong**: Login-required portal pages appearing in sitemap.xml.

**Correct**: Set `context.sitemap = 0` for all authenticated-only pages.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-website/references/anti-patterns.md*
