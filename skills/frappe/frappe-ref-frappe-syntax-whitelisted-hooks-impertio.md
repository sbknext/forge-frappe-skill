---
title: Frappe Ref — Frappe Syntax: Whitelisted: Hooks (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'hooks']
source: Impertio
---

# Hooks Reference

How whitelisted methods interact with `hooks.py` configuration.

## Method Discovery

Frappe discovers whitelisted methods automatically when the module is imported. You do NOT need to register methods in `hooks.py` for them to work. The `@frappe.whitelist()` decorator handles registration at import time.

However, `hooks.py` provides app-level configuration that affects whitelisted method behavior.

---

## require_type_annotated_api_methods [v15+]

Enforce type annotations on ALL whitelisted methods in your app:

```python
# hooks.py
require_type_annotated_api_methods = 1
```

When enabled:
- Every parameter in every `@frappe.whitelist()` method MUST have a type annotation
- Missing annotations raise `FrappeTypeError` at request time
- `self`, `cls`, `*args`, `**kwargs` are exempt
- Override per-method with `@frappe.whitelist(force_types=False)`

---

## override_whitelisted_methods

Override a whitelisted method from another app:

```python
# hooks.py
override_whitelisted_methods = {
    "frappe.client.get_count": "myapp.overrides.custom_get_count"
}
```

The replacement function MUST also be decorated with `@frappe.whitelist()`:

```python
# myapp/overrides.py
import frappe

@frappe.whitelist()
def custom_get_count(doctype, filters=None, debug=False, cache=False):
    """Custom override of frappe.client.get_count."""
    # Your custom logic
    return frappe.db.count(doctype, filters)
```

Use cases:
- Customizing built-in Frappe/ERPNext API behavior
- Adding logging or validation to existing endpoints
- Replacing functionality without forking

---

## boot_session

Add data available in the browser at page load via `frappe.boot`:

```python
# hooks.py
boot_session = "myapp.startup.boot_session"
```

```python
# myapp/startup.py
def boot_session(bootinfo):
    """Add custom data to frappe.boot (available in JS)."""
    bootinfo.my_settings = frappe.get_all(
        "My Setting",
        filters={"user": frappe.session.user},
        fields=["setting_name", "value"]
    )
```

This is NOT a whitelisted method but provides data that client-side code can use without additional API calls.

---

## website_route_rules

Map custom URL paths to whitelisted methods:

```python
# hooks.py
website_route_rules = [
    {"from_route": "/api/custom/<path:name>", "to_route": "myapp.custom_api.handle"}
]
```

---

## Method File Organization

ALWAYS organize whitelisted methods in predictable locations:

```
myapp/
├── api.py                    # Main public API methods
├── public_api.py             # Guest-accessible endpoints
├── utils/
│   └── helpers.py            # Utility API methods
└── myapp/
    └── doctype/
        └── my_doctype/
            └── my_doctype.py # Controller methods (Document class)
```

### Naming Conventions

| File Pattern | URL Pattern | Purpose |
|-------------|-------------|---------|
| `myapp/api.py` | `/api/method/myapp.api.*` | Main app API |
| `myapp/public_api.py` | `/api/method/myapp.public_api.*` | Public endpoints |
| `myapp/utils/helpers.py` | `/api/method/myapp.utils.helpers.*` | Utility functions |
| `myapp/myapp/doctype/X/X.py` | Via `frm.call()` | Controller methods |

---

## Common hooks.py Patterns Affecting APIs

```python
# hooks.py

# Override existing whitelisted methods
override_whitelisted_methods = {
    "original.module.method": "myapp.overrides.replacement_method"
}

# Enforce type annotations on all API methods [v15+]
require_type_annotated_api_methods = 1

# Add rate limiting configuration
rate_limit = {
    "myapp.api.public_submit": {"limit": 10, "seconds": 300}
}
```

---

## Key Rules

1. **NEVER register methods in hooks.py for basic API access** — the decorator handles it
2. **ALWAYS use `override_whitelisted_methods`** to customize existing APIs (not monkey-patching)
3. **ALWAYS decorate override functions** with `@frappe.whitelist()` — the override must also be whitelisted
4. **Use `require_type_annotated_api_methods`** in production apps for type safety [v15+]

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/hooks.md*
