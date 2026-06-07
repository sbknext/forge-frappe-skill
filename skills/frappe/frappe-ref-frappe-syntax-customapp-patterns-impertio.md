---
title: Frappe Ref — Frappe Syntax: Customapp: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'customapp', 'reference', 'patterns']
source: Impertio
---

# Custom App — Common Patterns

## Pattern 1: Minimal Frappe-Only App

```python
# hooks.py
app_name = "my_tool"
app_title = "My Tool"
app_publisher = "Your Company"
app_description = "Standalone tool"
app_email = "dev@example.com"
app_license = "MIT"

required_apps = ["frappe"]
```

Use when building standalone functionality that does NOT depend on ERPNext modules.

---

## Pattern 2: ERPNext Extension App

```python
# hooks.py
app_name = "erp_extension"
app_title = "ERP Extension"
app_publisher = "Your Company"
app_description = "Extends ERPNext Sales"
app_email = "dev@example.com"
app_license = "MIT"

required_apps = ["frappe", "erpnext"]

fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "ERP Extension"]]},
    {"dt": "Property Setter", "filters": [["module", "=", "ERP Extension"]]},
]

doc_events = {
    "Sales Invoice": {
        "validate": "erp_extension.overrides.sales_invoice.validate",
        "on_submit": "erp_extension.overrides.sales_invoice.on_submit",
    }
}

doctype_js = {
    "Sales Invoice": "public/js/sales_invoice.js",
}
```

Use when adding custom fields, validation, or workflows to existing ERPNext DocTypes.

---

## Pattern 3: Override Pattern (doc_events)

```python
# erp_extension/overrides/sales_invoice.py
import frappe

def validate(doc, method):
    """Called during Sales Invoice validate."""
    if doc.custom_approval_status == "Rejected":
        frappe.throw("Cannot save a rejected invoice")

def on_submit(doc, method):
    """Called after Sales Invoice submit."""
    create_audit_log(doc)
```

The function signature is ALWAYS `(doc, method)` for doc_events hooks.

---

## Pattern 4: Settings Singleton

```python
# Create a Single DocType called "My App Settings"
# Access from code:
settings = frappe.get_single("My App Settings")
api_key = settings.api_key
is_enabled = settings.enable_integration
```

ALWAYS use a Single DocType for app-wide configuration. NEVER hardcode settings.

---

## Pattern 5: Fixture-Based Custom Fields

1. Create Custom Fields via desk UI
2. Set `module` to your app module name
3. Add to hooks.py:

```python
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]},
]
```

4. Export:

```bash
bench --site mysite export-fixtures --app my_app
```

5. Commit the generated JSON files to Git

This ensures Custom Fields are version-controlled and deployed consistently.

---

## Pattern 6: Version-Compatible Code

```python
import frappe

frappe_version = int(frappe.__version__.split(".")[0])

if frappe_version >= 15:
    from frappe.utils.background_jobs import is_job_enqueued
else:
    # v14 fallback
    def is_job_enqueued(job_id):
        from frappe.core.page.background_jobs.background_jobs import get_info
        return job_id in [d.get("job_name") for d in get_info()]
```

---

## Pattern 7: Workspace Definition [v15+]

```json
{
    "doctype": "Workspace",
    "name": "My App",
    "module": "My App",
    "label": "My App",
    "is_standard": 1,
    "links": [
        {
            "label": "Documents",
            "links": [
                {"type": "doctype", "name": "My DocType", "label": "My DocType"}
            ]
        },
        {
            "label": "Settings",
            "links": [
                {"type": "doctype", "name": "My App Settings", "label": "Settings"}
            ]
        }
    ]
}
```

Place in: `my_app/my_app/workspace/my_app/my_app.json`

---

## Pattern 8: App with after_install Hook

```python
# hooks.py
after_install = "my_app.setup.install.after_install"

# my_app/setup/install.py
import frappe

def after_install():
    """Set up default data after app installation."""
    create_default_roles()
    create_default_settings()

def create_default_roles():
    for role_name in ["My App User", "My App Admin"]:
        if not frappe.db.exists("Role", role_name):
            frappe.get_doc({"doctype": "Role", "role_name": role_name}).insert()

def create_default_settings():
    settings = frappe.get_single("My App Settings")
    settings.default_status = "Active"
    settings.save()
```

---

## Pattern 9: Multi-App Dependency Chain

```python
# App C depends on App B which depends on App A
# app_c/hooks.py
required_apps = ["frappe", "app_a", "app_b"]

# bench get-app will auto-install dependencies in order
```

ALWAYS declare ALL dependencies in `required_apps`, not just immediate ones.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-customapp/references/patterns.md*
