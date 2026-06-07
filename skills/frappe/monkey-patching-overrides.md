---
title: Frappe Ref — Monkey Patching & Framework Overrides (frappe-apps-manager)
category: frappe
tags: [frappe, hooks, override, monkey-patch, override_doctype_class, whitelisted]
source: vyogotech/frappe-apps-manager
---

# Monkey Patching & Framework Overrides

## When to use

Trigger when you must change **core or third-party app behavior** without editing upstream source — bug fixes pending release, custom validation on core DocTypes, whitelisted API tweaks, or global request/response hooks. Prefer standard hooks first; runtime patching is expert-level.

## Patterns (prefer top to bottom)

### 1. DocType class override (recommended)

```python
# hooks.py
override_doctype_class = {
    "User": "custom_app.overrides.CustomUser",
}
```

```python
from frappe.core.doctype.user.user import User

class CustomUser(User):
    def validate(self):
        super().validate()
        if self.email and not self.email.endswith("@example.com"):
            frappe.throw("Only company emails are allowed")
```

Frappe v16+ also supports `extend_doctype_class` when multiple apps add methods without replacing the controller — inherit from the original class in all cases.

### 2. Whitelisted method override

```python
# hooks.py
override_whitelisted_methods = {
    "frappe.desk.doctype.event.event.get_events": "custom_app.overrides.get_events",
}
```

### 3. doc_events (non-invasive)

When you only need lifecycle hooks, use `doc_events` instead of subclassing — easier upgrades, no class coupling.

### 4. Runtime function patching (last resort)

```python
# hooks.py
after_install = "custom_app.overrides.apply_patches"
# or after_migrate for upgrades
```

```python
import frappe.utils

def apply_patches():
    if not hasattr(frappe.utils, "_original_get_url"):
        frappe.utils._original_get_url = frappe.utils.get_url
    frappe.utils.get_url = enhanced_get_url

def enhanced_get_url(*args, **kwargs):
    url = frappe.utils._original_get_url(*args, **kwargs)
    return url  # custom logic
```

## Gotchas

- **Always call `super()`** when overriding class methods unless fully replacing behavior.
- **Backup originals** before runtime reassignment (`_original_*` attribute).
- **Upgrade risk** — upstream signature/behavior changes can break overrides; add integration tests.
- **Circular imports** — keep override modules thin; import core classes inside functions if needed.
- **Log when applied** — `frappe.logger().info("Applied CustomUser override")` aids production debugging.
- Runtime patches are harder to discover than `hooks.py` entries — document in your app's README.

## Version notes

| Feature | v13–v15 | v16+ |
|---------|---------|------|
| `override_doctype_class` | Supported | Supported |
| `extend_doctype_class` | — | Multiple apps can extend same DocType |
| `override_whitelisted_methods` | Supported | Supported |

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-monkey-patching/SKILL.md
