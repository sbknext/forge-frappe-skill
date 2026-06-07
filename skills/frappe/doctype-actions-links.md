---
title: Frappe Ref — DocType Actions and Links (Frappe docs)
category: frappe
tags: [frappe, doctype, dashboard, actions, connections]
source: Frappe docs
---

# DocType Actions and Links

## When to use

- Add toolbar buttons (Actions) on a DocType form that trigger server methods or routes.
- Show related documents (Connections) on the form dashboard for navigation and quick creation.

Added in Frappe v12.1.

## Key concepts

**Actions** — `DocType Action` records create buttons with two types:
- **Server Action** — calls a `@frappe.whitelist()` Python function.
- **Route** — redirects to a desk route.

**Connections** — dashboard section listing linked DocTypes, grouped under transaction labels.

## Procedure / patterns

### Server Action

```python
@frappe.whitelist()
def execute_function(**kwargs):
    # kwargs carries context from the Action config
    ...
```

Configure the Action path in DocType → Actions (e.g. `myapp.api.execute_function`).

### Connections via dashboard script

Create `<doctype>_dashboard.py` in the doctype folder:

```python
def get_data():
    return {
        "fieldname": "sales_invoice",
        "non_standard_fieldnames": {"Journal Entry": "reference_name"},
        "internal_links": {"Sales Order": ["items", "sales_order"]},
        "transactions": [
            {"label": "Payment", "items": ["Payment Entry", "Journal Entry"]},
        ],
    }
```

- `fieldname` — default link field for external connections.
- `internal_links` — connections through child tables.
- `non_standard_fieldnames` — override link field per target DocType.

## Gotchas

- Actions must be whitelisted — permission-check inside the handler.
- Dashboard `get_data()` lives in `<doctype>_dashboard.py`, not the main controller.
- Actions and Links are extensible via Customize Form for site-specific tweaks.

## Source

https://docs.frappe.io/framework/user/en/basics/doctypes/actions-and-links
