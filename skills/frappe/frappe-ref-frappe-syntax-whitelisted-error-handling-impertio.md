---
title: Frappe Ref — Frappe Syntax: Whitelisted: Error Handling (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'error-handling']
source: Impertio
---

# Error Handling Reference

Exception types, error patterns, and logging for whitelisted methods.

## frappe.throw()

Displays error to user and stops execution. ALWAYS use `_()` for translatable messages.

### Signature

```python
frappe.throw(
    msg,                    # Error message (use _() for translation)
    exc=None,               # Exception class (optional)
    title=None,             # Dialog title (optional)
    is_minimizable=False,   # Minimizable dialog (optional)
    wide=False,             # Wide dialog (optional)
    as_list=False           # Show message as bullet list (optional)
)
```

### Examples

```python
# Basic validation error
frappe.throw(_("Required field is missing"))

# With exception type (sets HTTP status code)
frappe.throw(_("Not permitted"), frappe.PermissionError)

# With title
frappe.throw(
    _("Please fill all required fields"),
    title=_("Validation Error")
)

# With interpolated values
frappe.throw(
    _("Amount {0} exceeds maximum {1}").format(amount, max_amount),
    frappe.ValidationError
)
```

---

## Exception Types and HTTP Codes

| Exception | HTTP Code | When to Use |
|-----------|-----------|-------------|
| `frappe.ValidationError` | 417 | Input validation failure |
| `frappe.PermissionError` | 403 | Access denied |
| `frappe.DoesNotExistError` | 404 | Document not found |
| `frappe.DuplicateEntryError` | 409 | Duplicate record |
| `frappe.AuthenticationError` | 401 | Not authenticated |
| `frappe.MandatoryError` | 417 | Required field missing |
| `frappe.TimestampMismatchError` | 409 | Document modified by another user |
| `frappe.DataError` | 417 | Data integrity issue |
| `frappe.OutgoingEmailError` | 500 | Email sending failure |

### Usage Pattern

```python
@frappe.whitelist()
def process_order(order_id):
    if not frappe.db.exists("Sales Order", order_id):
        frappe.throw(
            _("Order {0} not found").format(order_id),
            frappe.DoesNotExistError
        )

    if not frappe.has_permission("Sales Order", "write", order_id):
        frappe.throw(_("Not permitted"), frappe.PermissionError)

    doc = frappe.get_doc("Sales Order", order_id)
    if doc.status == "Closed":
        frappe.throw(
            _("Cannot modify closed order"),
            frappe.ValidationError
        )
```

---

## Error Logging

### frappe.log_error()

Logs to the Error Log DocType. ALWAYS log before showing generic error to user.

```python
@frappe.whitelist()
def external_api_call(data):
    try:
        response = requests.post(url, json=data, timeout=30)
        return response.json()
    except Exception:
        frappe.log_error(
            frappe.get_traceback(),
            "External API Error"
        )
        frappe.throw(_("External service unavailable. Please try later."))
```

### Log with Context

```python
# ALWAYS include context for debugging
frappe.log_error(
    f"User: {frappe.session.user}\n"
    f"DocType: {doctype}\n"
    f"Name: {name}\n\n"
    f"{frappe.get_traceback()}",
    f"API Error: {doctype}"
)
```

---

## Try/Except Patterns

### Basic Pattern

```python
@frappe.whitelist()
def safe_operation(param):
    try:
        result = process_data(param)
        return {"success": True, "data": result}
    except Exception:
        frappe.log_error(frappe.get_traceback(), "safe_operation")
        frappe.throw(_("Operation failed"))
```

### Comprehensive Pattern (REST API Style)

```python
@frappe.whitelist()
def robust_api(doctype, name):
    try:
        frappe.has_permission(doctype, "read", name, throw=True)
        doc = frappe.get_doc(doctype, name)
        return {"success": True, "data": doc.as_dict()}

    except frappe.DoesNotExistError:
        frappe.local.response["http_status_code"] = 404
        return {"success": False, "error": "Document not found"}

    except frappe.PermissionError:
        frappe.local.response["http_status_code"] = 403
        return {"success": False, "error": "Access denied"}

    except frappe.ValidationError as e:
        frappe.local.response["http_status_code"] = 400
        return {"success": False, "error": str(e)}

    except Exception:
        frappe.log_error(frappe.get_traceback(), f"API Error: {doctype}/{name}")
        frappe.local.response["http_status_code"] = 500
        return {"success": False, "error": "Internal server error"}
```

### Pattern with Cleanup

```python
@frappe.whitelist()
def transactional_operation(data):
    doc = None
    try:
        doc = frappe.get_doc(frappe.parse_json(data) if isinstance(data, str) else data)
        doc.insert()
        process_related(doc.name)
        return {"success": True, "name": doc.name}
    except Exception:
        if doc and doc.name:
            frappe.delete_doc(doc.doctype, doc.name, force=True)
        frappe.log_error(frappe.get_traceback(), "Transaction Error")
        frappe.throw(_("Operation failed and was rolled back"))
```

---

## Custom HTTP Status Codes

```python
@frappe.whitelist()
def api_with_status(param):
    if not param:
        frappe.local.response["http_status_code"] = 400
        return {"error": "Parameter required"}

    if not frappe.db.exists("Item", param):
        frappe.local.response["http_status_code"] = 404
        return {"error": "Item not found"}

    frappe.local.response["http_status_code"] = 200
    return {"item": frappe.get_doc("Item", param).as_dict()}
```

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Success (default) |
| 201 | Created | New document created |
| 400 | Bad Request | Input validation error |
| 401 | Unauthorized | Not logged in |
| 403 | Forbidden | No permission |
| 404 | Not Found | Document does not exist |
| 409 | Conflict | Duplicate or version conflict |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected error |

---

## Response Structure on Error

### frappe.throw() Response

```json
{
    "exc_type": "ValidationError",
    "exc": "[Traceback string...]",
    "_server_messages": "[{\"message\": \"Error message\"}]"
}
```

### Manual Error Response

```json
{
    "message": {
        "success": false,
        "error": "Description of what went wrong"
    }
}
```

---

## Critical Rules

1. **NEVER expose raw exceptions**: `frappe.throw(str(e))` can leak DB credentials, paths, and internals
2. **ALWAYS log before generic error**: `frappe.log_error()` first, then `frappe.throw()` with safe message
3. **ALWAYS catch specific exceptions first**: Order from most specific to `Exception`
4. **ALWAYS include context in logs**: user, DocType, document name, action
5. **NEVER swallow exceptions silently**: ALWAYS log or re-raise

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/error-handling.md*
