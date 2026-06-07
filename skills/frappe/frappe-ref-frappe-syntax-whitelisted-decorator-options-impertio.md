---
title: Frappe Ref — Frappe Syntax: Whitelisted: Decorator Options (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'decorator-options']
source: Impertio
---

# Decorator Options Reference

Complete reference for all `@frappe.whitelist()` parameters.

## Signature

```python
def whitelist(
    allow_guest: bool = False,
    xss_safe: bool = False,
    methods: list[str] | None = None,
    force_types: bool | None = None       # [v15+]
) -> Callable
```

---

## allow_guest

| Value | Behavior |
|-------|----------|
| `False` (default) | Only authenticated users can call the method |
| `True` | Unauthenticated users (Guest role) can call the method |

```python
# Default — requires login
@frappe.whitelist()
def internal_api():
    return {"data": "authenticated only"}

# Public — anyone can call
@frappe.whitelist(allow_guest=True)
def public_status():
    return {"status": "available"}
```

**ALWAYS apply these safeguards when using `allow_guest=True`:**
- Validate ALL input (type, length, format)
- Apply `@rate_limit()` to prevent abuse
- Sanitize with `frappe.utils.strip_html()`
- NEVER accept arbitrary DocType names from guest input
- NEVER use `ignore_permissions=True` in guest endpoints without strict controls

**Valid use cases for `allow_guest=True`:**
- Public information endpoints (app status, version)
- Contact form submissions
- User registration/signup flows
- Webhook receivers
- Public product catalog queries

---

## methods

| Value | Allowed HTTP Methods |
|-------|---------------------|
| `None` (default) | GET, POST, PUT, DELETE |
| `["GET"]` | GET only |
| `["POST"]` | POST only |
| `["GET", "POST"]` | GET and POST |

```python
# Read-only endpoint
@frappe.whitelist(methods=["GET"])
def get_status(order_id):
    return frappe.db.get_value("Sales Order", order_id, "status")

# Write-only endpoint
@frappe.whitelist(methods=["POST"])
def update_status(order_id, status):
    frappe.db.set_value("Sales Order", order_id, "status", status)
    return {"updated": True}

# Check request method at runtime
@frappe.whitelist(methods=["GET", "POST"])
def dual_endpoint(param):
    if frappe.request.method == "GET":
        return get_data(param)
    else:
        return save_data(param)
```

**Best practice**: ALWAYS restrict to `["GET"]` for read-only and `["POST"]` for write operations. Using the default (all methods) is acceptable but less secure.

---

## xss_safe

| Value | Behavior |
|-------|----------|
| `False` (default) | HTML in response is escaped |
| `True` | HTML is NOT escaped (raw output) |

```python
# Return raw HTML content
@frappe.whitelist(xss_safe=True)
def get_rendered_template():
    return frappe.render_template("myapp/templates/widget.html", {})
```

**NEVER use `xss_safe=True` unless:**
- You control ALL HTML output (no user input in output)
- The response is a rendered server-side template
- You have sanitized any dynamic content with `frappe.utils.strip_html()`

NOTE: `xss_safe` only registers in `frappe.xss_safe_methods` when BOTH `allow_guest=True` AND `xss_safe=True`.

---

## force_types [v15+]

| Value | Behavior |
|-------|----------|
| `None` (default) | Inherits from app-level `require_type_annotated_api_methods` hook |
| `True` | ALL parameters (except self/cls, *args, **kwargs) MUST have type annotations |
| `False` | Type annotations are optional (validation still runs on annotated params) |

```python
# Enforce type annotations — missing annotations raise FrappeTypeError
@frappe.whitelist(force_types=True)
def strict_api(customer: str, limit: int = 10) -> dict:
    return {"data": frappe.get_all("Customer", limit=limit)}

# App-level enforcement via hooks.py:
# require_type_annotated_api_methods = 1
```

When `force_types=True`:
- Every parameter MUST have a type annotation
- Pydantic `TypeAdapter` validates/coerces values at request time
- `FrappeTypeError` is raised for type mismatches
- Supported types: `str`, `int`, `float`, `bool`, `list`, `dict`, and Pydantic models

---

## Parameter Combinations

### Public POST Endpoint (Contact Form)

```python
@frappe.whitelist(allow_guest=True, methods=["POST"])
def submit_form(name, email, message):
    # ALWAYS validate thoroughly with guest access
    if not name or not email:
        frappe.throw(_("Name and email required"))
    doc = frappe.get_doc({
        "doctype": "Web Form Submission",
        "name1": frappe.utils.strip_html(name),
        "email": email,
        "message": frappe.utils.strip_html(message or "")
    })
    doc.insert(ignore_permissions=True)
    return {"success": True}
```

### Public HTML Endpoint

```python
@frappe.whitelist(allow_guest=True, xss_safe=True)
def public_widget():
    return frappe.render_template("myapp/templates/public.html", {})
```

### Strict Typed Internal API [v15+]

```python
@frappe.whitelist(methods=["GET"], force_types=True)
def get_dashboard(period: str, limit: int = 20) -> dict:
    frappe.only_for("System Manager")
    return {"stats": compute_stats(period, limit)}
```

---

## Error Responses

| Situation | Error |
|-----------|-------|
| Unauthenticated call to non-guest method | `frappe.AuthenticationError` (401) |
| Wrong HTTP method | `405 Method Not Allowed` |
| Function not decorated | `Method not found` (404) |
| Type mismatch [v15+] | `FrappeTypeError` (422) |
| Missing type annotation with `force_types=True` [v15+] | `FrappeTypeError` |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/decorator-options.md*
