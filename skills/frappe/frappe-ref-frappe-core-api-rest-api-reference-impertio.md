---
title: Frappe Ref — Frappe Core: Api: Rest Api Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'api', 'reference', 'rest-api-reference']
source: Impertio
---

# REST API Reference

> Complete reference for Frappe REST API (resource-based CRUD).

---

## API Versions

| Version | Prefix | Available |
|---------|--------|-----------|
| v1 | `/api/resource/{doctype}` | All versions |
| v2 | `/api/v2/document/{doctype}` | [v15+] |

The v1 API continues to work in v15+. Use v2 for new integrations targeting v15+.

---

## Required Headers

**ALWAYS** include for JSON responses:

```python
headers = {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    'Authorization': 'token api_key:api_secret'
}
```

Without `Accept: application/json`, Frappe MAY return HTML instead of JSON.

---

## List Documents (GET)

```
GET /api/resource/{doctype}
```

### Default Behavior
- Returns 20 records
- Only `name` field

### Query Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `fields` | JSON array | Fields to retrieve | `["name"]` |
| `filters` | JSON array | AND filter conditions | none |
| `or_filters` | JSON array | OR filter conditions | none |
| `order_by` | string | Sort expression | `modified desc` |
| `limit_start` | int | Offset for pagination | `0` |
| `limit_page_length` | int | Number of results | `20` |
| `limit` | int | Alias for limit_page_length [v15+] | — |
| `as_dict` | bool | Response as dict (default) | `true` |
| `debug` | bool | Include SQL query in response | `false` |

### Filter Syntax

Filters MUST be a JSON array. Each condition is `[field, operator, value]` or `[doctype, field, operator, value]`.

```python
# Equality
filters = [["status", "=", "Open"]]

# Comparison
filters = [["amount", ">", 1000]]
filters = [["amount", ">=", 500]]
filters = [["amount", "<", 10000]]
filters = [["amount", "<=", 999]]

# Not equal
filters = [["status", "!=", "Cancelled"]]

# Pattern matching
filters = [["name", "like", "%INV%"]]
filters = [["name", "not like", "%TEST%"]]

# List membership
filters = [["status", "in", ["Open", "Pending"]]]
filters = [["status", "not in", ["Cancelled", "Closed"]]]

# NULL checks
filters = [["reference", "is", "set"]]         # NOT NULL
filters = [["reference", "is", "not set"]]      # IS NULL

# Range
filters = [["date", "between", ["2024-01-01", "2024-12-31"]]]
```

Full operator list: `=`, `!=`, `>`, `<`, `>=`, `<=`, `like`, `not like`, `in`, `not in`, `is`, `between`.

### OR Filters

```
GET /api/resource/Customer?or_filters=[
    ["customer_group","=","Commercial"],
    ["customer_group","=","Individual"]
]
```

### Example: Filtered + Paginated List

```bash
GET /api/resource/Sales Invoice
    ?fields=["name","customer","grand_total","status"]
    &filters=[["status","=","Paid"],["grand_total",">",1000]]
    &order_by=posting_date desc
    &limit_page_length=50
    &limit_start=0
```

Response:
```json
{
    "data": [
        {"name": "SINV-00001", "customer": "Customer A", "grand_total": 1500.00, "status": "Paid"}
    ]
}
```

### Python Example

```python
import requests, json

params = {
    'fields': json.dumps(["name", "customer", "grand_total"]),
    'filters': json.dumps([["status", "=", "Paid"]]),
    'limit_page_length': 100,
    'order_by': 'modified desc'
}

response = requests.get(f'{BASE_URL}/api/resource/Sales Invoice',
                        params=params, headers=headers)
data = response.json()['data']
```

---

## Read Document (GET)

```
GET /api/resource/{doctype}/{name}
```

Response includes all fields and child table data:
```json
{
    "data": {
        "name": "CUST-00001",
        "customer_name": "Test Customer",
        "items": [...]
    }
}
```

---

## Create Document (POST)

```
POST /api/resource/{doctype}
Content-Type: application/json
```

Body: JSON object with field values.

### With Child Table

```python
requests.post(f'{BASE_URL}/api/resource/Sales Order', json={
    "customer": "CUST-001",
    "delivery_date": "2024-02-01",
    "items": [
        {"item_code": "ITEM-001", "qty": 5, "rate": 100},
        {"item_code": "ITEM-002", "qty": 2, "rate": 250}
    ]
}, headers=headers)
```

Response: full document with generated `name`, `owner`, `creation` fields.

---

## Update Document (PUT)

```
PUT /api/resource/{doctype}/{name}
```

Only specified fields are changed (PATCH-like behavior):

```python
requests.put(f'{BASE_URL}/api/resource/Customer/CUST-001',
             json={"customer_group": "Premium"}, headers=headers)
```

### Update Child Table

```python
# Replace entire child table
{"items": [{"item_code": "ITEM-001", "qty": 10}]}  # All others removed

# Update specific child row (by row name)
{"items": [{"name": "row_id_abc", "qty": 10}]}  # Only this row updated
```

---

## Delete Document (DELETE)

```
DELETE /api/resource/{doctype}/{name}
```

Response: `{"message": "ok"}`

---

## Pagination Pattern

```python
def get_all_records(doctype, base_url, headers, filters=None, page_size=100):
    all_data, offset = [], 0
    while True:
        params = {
            'filters': json.dumps(filters or []),
            'limit_start': offset,
            'limit_page_length': page_size
        }
        response = requests.get(f'{base_url}/api/resource/{doctype}',
                                params=params, headers=headers)
        data = response.json().get('data', [])
        if not data:
            break
        all_data.extend(data)
        if len(data) < page_size:
            break  # Last page
        offset += page_size
    return all_data
```

---

## File Upload

```python
files = {'file': ('document.pdf', open('doc.pdf', 'rb'), 'application/pdf')}
data = {'doctype': 'Customer', 'docname': 'CUST-001', 'is_private': 1}

# NOTE: Do NOT set Content-Type header — requests handles multipart boundary
response = requests.post(f'{BASE_URL}/api/method/upload_file',
    files=files, data=data,
    headers={'Authorization': 'token api_key:api_secret'})
```

Response:
```json
{"message": {"name": "file_hash.pdf", "file_url": "/private/files/file_hash.pdf", "is_private": 1}}
```

---

## v2 API Endpoints [v15+]

| Operation | Method | Endpoint |
|-----------|--------|----------|
| List | GET | `/api/v2/document/{doctype}` |
| Create | POST | `/api/v2/document/{doctype}` |
| Read | GET | `/api/v2/document/{doctype}/{name}` |
| Update | PATCH | `/api/v2/document/{doctype}/{name}` |
| Delete | DELETE | `/api/v2/document/{doctype}/{name}` |
| Copy | GET | `/api/v2/document/{doctype}/{name}/copy` |
| Doc Method | POST | `/api/v2/document/{doctype}/{name}/method/{method}` |
| Metadata | GET | `/api/v2/doctype/{doctype}/meta` |
| Count | GET | `/api/v2/doctype/{doctype}/count` |

---

## Standard Response Fields

| Field | Description |
|-------|-------------|
| `name` | Document identifier |
| `doctype` | Document type |
| `docstatus` | 0=Draft, 1=Submitted, 2=Cancelled |
| `owner` | Created by user |
| `creation` | Creation datetime |
| `modified` | Last modified datetime |
| `modified_by` | Last modified by user |

---

## Debug Mode

```
GET /api/resource/Customer?debug=True&limit=5
```

Response includes SQL query and execution time in `exc` field.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-api/references/rest-api-reference.md*
