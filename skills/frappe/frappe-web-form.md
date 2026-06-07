---
title: Frappe Ref — Web Form (Frappe docs)
category: frappe
tags: [frappe, web-form, portal, website]
source: Frappe docs
---

# Web Form

## When to use

- Expose a subset of a DocType on the public website (guest or logged-in).
- Collect structured submissions without building a custom portal page from scratch.
- Ship standard web forms in an app for repeatable installs across sites.

## Key concepts

- **Web Form** DocType maps selected fields from a target DocType into a website form.
- Forms can be **public** (anyone) or require login.
- **Standard Web Forms** (`Is Standard`, visible in Developer Mode) export `.py` and `.js` files into your app module for version control.
- Non-standard web forms are site-specific records; scripts can be edited directly on the form.

## Procedure / patterns

1. Awesomebar → **New Web Form**.
2. Set Title, target DocType, optional introduction.
3. Click **Get Fields** (all DocType fields) or pick fields manually.
4. Publish.
5. For app-shipped forms: enable Developer Mode, check **Is Standard**, customize exported controller/JS in the app module, commit to git.

Inject assets on standard forms via hooks:

```python
# hooks.py
webform_include_js  = {"ToDo": "public/js/custom_todo.js"}
webform_include_css = {"ToDo": "public/css/custom_todo.css"}
```

## Gotchas

- `Is Standard` only appears when **Developer Mode** is on.
- `webform_include_*` hooks apply to **Standard Web Forms** only; ad-hoc forms use inline script on the Web Form record.
- Web forms create real DocType records — apply permissions and validation on the server controller, not only in client JS.

Source: https://docs.frappe.io/framework/user/en/web-form , https://docs.frappe.io/framework/user/en/python-api/hooks
