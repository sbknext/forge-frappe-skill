---
title: Frappe Ref — Frappe Syntax: Whitelisted: Response Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'response-patterns']
source: Impertio
---

# Response Patterns Reference

How to structure responses from whitelisted methods.

## Return Value (Default)

The return value is auto-wrapped in `{"message": <return_value>}`:

```python
@frappe.whitelist()
def get_summary(customer):
    return {"customer": customer, "total": 15000}
# Response: {"message": {"customer": "...", "total": 15000}}
```

### Return Type Behavior

| Python Return | JSON Response |
|--------------|---------------|
| `dict` | `{"message": {"key": "value"}}` |
| `list` | `{"message": [1, 2, 3]}` |
| `str` | `{"message": "text"}` |
| `int` / `float` | `{"message": 42}` |
| `None` (or no return) | `{"message": null}` |

---

## frappe.response Object

For advanced control over the response:

```python
@frappe.whitelist()
def custom_response():
    frappe.response["message"] = {"data": "value"}
    frappe.response["count"] = 10        # Extra field outside message
    frappe.response["status"] = "success"
    # No return needed
```

Response:
```json
{"message": {"data": "value"}, "count": 10, "status": "success"}
```

NOTE: If you use both `return` and `frappe.response["message"]`, the `return` value overwrites `frappe.response["message"]`.

---

## Custom HTTP Status Codes

```python
# 201 Created
@frappe.whitelist(methods=["POST"])
def create_item(data):
    doc = frappe.get_doc(frappe.parse_json(data) if isinstance(data, str) else data)
    doc.insert()
    frappe.local.response["http_status_code"] = 201
    return {"name": doc.name, "created": True}

# 400 Bad Request
@frappe.whitelist()
def validate_input(param):
    if not param:
        frappe.local.response["http_status_code"] = 400
        return {"error": "Parameter required"}
    return {"valid": True}
```

---

## File Download

### Attachment Download

```python
@frappe.whitelist()
def download_file(name):
    frappe.has_permission("File", "read", name, throw=True)
    file_doc = frappe.get_doc("File", name)

    frappe.response.filename = file_doc.file_name
    frappe.response.filecontent = file_doc.get_content()
    frappe.response.type = "download"
    frappe.response.display_content_as = "attachment"
```

### Inline Display (PDF in Browser)

```python
@frappe.whitelist()
def view_pdf(name):
    content = generate_pdf(name)
    frappe.response.filename = f"{name}.pdf"
    frappe.response.filecontent = content
    frappe.response.type = "download"
    frappe.response.display_content_as = "inline"
```

### CSV Export

```python
@frappe.whitelist()
def export_csv(doctype):
    import csv
    from io import StringIO

    frappe.has_permission(doctype, "read", throw=True)
    data = frappe.get_all(doctype, fields=["name", "status"], limit=1000)

    output = StringIO()
    writer = csv.DictWriter(output, fieldnames=["name", "status"])
    writer.writeheader()
    writer.writerows(data)

    frappe.response.filename = f"{doctype}_export.csv"
    frappe.response.filecontent = output.getvalue()
    frappe.response.type = "download"
```

---

## Response Types

| Type | Usage | Setting |
|------|-------|---------|
| `json` (default) | API responses | Automatic |
| `download` | File downloads | `frappe.response.type = "download"` |
| `csv` | CSV export | `frappe.response.type = "csv"` |
| `pdf` | PDF generation | `frappe.response.type = "pdf"` |
| `redirect` | HTTP redirect | `frappe.response.type = "redirect"` |
| `binary` | Binary data | `frappe.response.type = "binary"` |

### HTTP Redirect

```python
@frappe.whitelist()
def redirect_to_doc(doctype, name):
    frappe.response.type = "redirect"
    frappe.response.location = f"/app/{frappe.scrub(doctype)}/{name}"
```

---

## Paginated Response

ALWAYS include pagination metadata for list endpoints:

```python
@frappe.whitelist(methods=["GET"])
def get_orders(page=1, page_size=20):
    page = int(page)
    page_size = min(int(page_size), 100)  # ALWAYS cap page size
    start = (page - 1) * page_size

    data = frappe.get_all(
        "Sales Order",
        fields=["name", "customer", "grand_total", "status"],
        limit_start=start,
        limit_page_length=page_size,
        order_by="modified desc"
    )
    total = frappe.db.count("Sales Order")

    return {
        "data": data,
        "pagination": {
            "page": page,
            "page_size": page_size,
            "total": total,
            "pages": (total + page_size - 1) // page_size,
            "has_next": (start + page_size) < total
        }
    }
```

---

## Custom Response Headers

```python
@frappe.whitelist()
def cached_data():
    frappe.local.response.headers["Cache-Control"] = "public, max-age=300"
    frappe.local.response.headers["X-Custom-Header"] = "value"
    return {"data": "cacheable"}
```

---

## Consistent API Response Structure

ALWAYS use a consistent structure for APIs consumed by external clients:

```python
# Success
@frappe.whitelist()
def api_success(param):
    result = process(param)
    return {
        "success": True,
        "data": result,
        "message": "Operation completed"
    }

# Error
@frappe.whitelist()
def api_error(param):
    if not param:
        frappe.local.response["http_status_code"] = 400
        return {
            "success": False,
            "error": "Parameter required",
            "error_code": "MISSING_PARAM"
        }
```

---

## Key Rules

1. **ALWAYS wrap return values consistently** — use `{"success": bool, "data": ...}` for external APIs
2. **NEVER return sensitive fields** — filter `doc.as_dict()` to only necessary fields
3. **ALWAYS cap page_size** — prevent memory exhaustion from `page_size=999999`
4. **ALWAYS include pagination metadata** — total count, page info, has_next
5. **ALWAYS set correct HTTP status** — 201 for created, 400 for bad input, etc.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/response-patterns.md*
