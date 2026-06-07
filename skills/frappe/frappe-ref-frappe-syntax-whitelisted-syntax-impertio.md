---
title: Frappe Ref — Frappe Syntax: Whitelisted: Syntax (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'syntax']
source: Impertio
---

# Decorator Syntax and Registration

How `@frappe.whitelist()` registers Python functions as HTTP-accessible API endpoints.

## Core Mechanism

When Python loads a module containing `@frappe.whitelist()`, the decorator:

1. Adds the function to the global `frappe.whitelisted` set
2. Records allowed HTTP methods in `frappe.allowed_http_methods_for_whitelisted_func`
3. If `allow_guest=True`, adds to `frappe.guest_methods`
4. If `xss_safe=True` AND `allow_guest=True`, adds to `frappe.xss_safe_methods`
5. [v15+] Wraps with `validate_argument_types()` for type checking

## Decorator Syntax

```python
import frappe

# Minimal — authenticated users, all HTTP methods
@frappe.whitelist()
def my_endpoint():
    return {"status": "ok"}

# Full — all parameters specified
@frappe.whitelist(
    allow_guest=True,
    xss_safe=False,
    methods=["POST"],
    force_types=True      # [v15+] require type annotations
)
def public_submit(name: str, email: str) -> dict:
    return {"received": True}
```

## URL Mapping

The dotted Python import path becomes the API URL:

| Python Location | API URL |
|----------------|---------|
| `myapp/api.py` -> `def get_data()` | `/api/method/myapp.api.get_data` |
| `myapp/utils/helpers.py` -> `def calc()` | `/api/method/myapp.utils.helpers.calc` |
| `myapp/myapp/api.py` -> `def run()` | `/api/method/myapp.myapp.api.run` |

ALWAYS use the full dotted path from the app root, NOT from the site root.

## Controller Methods

For Document class methods, the decorator works on instance methods:

```python
from frappe.model.document import Document

class MyDocType(Document):
    @frappe.whitelist()
    def my_action(self, param1, param2=None):
        # 'self' is the document instance
        return {"name": self.name, "result": param1}
```

Controller methods are NOT called via `/api/method/` directly. They use:
- Client: `frm.call('my_action', {param1: 'value'})`
- Internal endpoint: `/api/method/run_doc_method`

## Registration Verification

To verify a method is whitelisted at runtime:

```python
# Check if a function is whitelisted
from frappe import is_whitelisted
# The framework checks this automatically on every /api/method/ request

# Manual check (debugging only)
func_path = "myapp.api.get_data"
module_path, func_name = func_path.rsplit(".", 1)
module = frappe.get_module(module_path)
func = getattr(module, func_name)
is_registered = func in frappe.whitelisted
```

## What Happens on API Request

1. Client sends request to `/api/method/myapp.api.get_data`
2. Frappe resolves the dotted path to a Python function
3. Checks if function is in `frappe.whitelisted` set — if not, returns 404
4. Checks if user is logged in (unless function is in `guest_methods`)
5. Checks HTTP method against `allowed_http_methods_for_whitelisted_func`
6. [v15+] Validates argument types if annotations present
7. Calls the function with request parameters
8. Wraps return value as `{"message": <return_value>}`
9. POST requests auto-commit via `frappe.db.commit()`

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "Method not found" | Function not decorated or module not importable | Add `@frappe.whitelist()`, check import path |
| 403 Forbidden | User not logged in on non-guest method | Add `allow_guest=True` or authenticate |
| 405 Method Not Allowed | Wrong HTTP verb for restricted method | Match `methods` parameter |
| `FrappeTypeError` [v15+] | Type annotation mismatch | Fix parameter types or add annotations |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/syntax.md*
