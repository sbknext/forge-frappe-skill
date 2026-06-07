---
title: Frappe Ref — Frappe Syntax: Whitelisted: Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'methods']
source: Impertio
---

# Methods Reference

The two types of whitelisted methods: standalone functions and controller (Document) methods.

## Standalone Functions

Module-level functions decorated with `@frappe.whitelist()`. Called via `frappe.call()` from JavaScript or via REST API.

### Definition

```python
# myapp/api.py
import frappe
from frappe import _

@frappe.whitelist()
def get_customer_summary(customer):
    frappe.has_permission("Customer", "read", throw=True)
    doc = frappe.get_doc("Customer", customer)
    return {
        "name": doc.name,
        "customer_name": doc.customer_name,
        "outstanding": doc.outstanding_amount
    }
```

### Client Call

```javascript
frappe.call({
    method: 'myapp.api.get_customer_summary',
    args: { customer: 'CUST-001' }
}).then(r => console.log(r.message));
```

### REST Call

```bash
curl -H "Authorization: token key:secret" \
     "https://site.com/api/method/myapp.api.get_customer_summary?customer=CUST-001"
```

---

## Controller Methods (Document Methods)

Methods defined on a `Document` subclass. Called via `frm.call()` from the form view. The method receives `self` (the document instance) automatically.

### Definition

```python
# myapp/myapp/doctype/sales_order/sales_order.py
import frappe
from frappe import _
from frappe.model.document import Document

class SalesOrder(Document):
    @frappe.whitelist()
    def calculate_commission(self, rate=None):
        """Controller method — 'self' is the current document."""
        rate = float(rate or 0.05)
        return {
            "commission": self.grand_total * rate,
            "order": self.name
        }

    @frappe.whitelist()
    def send_email(self):
        if not self.contact_email:
            frappe.throw(_("No email address"))
        frappe.sendmail(
            recipients=[self.contact_email],
            subject=f"Order {self.name}",
            message=f"Your order {self.name} is confirmed."
        )
        return {"sent_to": self.contact_email}
```

### Client Call

```javascript
// frm.call() automatically passes the document context
frm.call('calculate_commission', { rate: 0.10 })
    .then(r => {
        frm.set_value('commission_amount', r.message.commission);
    });

// With options
frm.call({
    method: 'send_email',
    freeze: true,
    freeze_message: __('Sending...')
}).then(r => {
    frappe.show_alert({ message: __('Sent'), indicator: 'green' });
});
```

### How frm.call() Works Internally

When you call `frm.call('method_name')`:
1. Frappe sends a POST to `/api/method/run_doc_method`
2. Request body includes: `{"doc": <serialized_doc>, "method": "method_name", "args": {...}}`
3. Server loads the document, calls the method on the Document instance
4. Returns the method's return value

---

## Key Differences

| Aspect | Standalone Function | Controller Method |
|--------|-------------------|-------------------|
| Location | Any `.py` file in app | DocType controller file |
| First parameter | Custom (from args) | `self` (document instance) |
| Client call | `frappe.call({method: 'dotted.path'})` | `frm.call('method_name')` |
| URL | `/api/method/dotted.path` | `/api/method/run_doc_method` (internal) |
| Document context | Must load manually | `self` provides document |
| Use case | Utilities, integrations, dashboards | Document-specific actions |

---

## When to Use Which

### Use Standalone Functions When:
- The operation is NOT tied to a specific document
- Multiple DocTypes are involved
- Building a public or integration API
- Creating dashboard or reporting endpoints
- The method is called from outside a form view

### Use Controller Methods When:
- The operation acts on a specific document instance
- You need access to `self` (document fields, child tables)
- The action should appear as a button on the form
- The method logically belongs to the DocType's behavior

---

## kwargs Pattern

For functions that accept dynamic parameters:

```python
@frappe.whitelist()
def flexible_api(**kwargs):
    """Accept any parameters via kwargs."""
    customer = kwargs.get("customer")
    filters = kwargs.get("filters", {})
    if isinstance(filters, str):
        filters = frappe.parse_json(filters)
    return frappe.get_all("Sales Order", filters=filters)
```

NOTE: When using `**kwargs`, type validation [v15+] does NOT apply to the kwargs. ALWAYS validate manually.

---

## Key Rules

1. **ALWAYS add `@frappe.whitelist()`** to controller methods called via `frm.call()` — without it, the call returns "Method not found"
2. **NEVER call controller methods via `/api/method/` directly** — use `frm.call()` or the internal `run_doc_method` endpoint
3. **ALWAYS add permission checks** in both standalone and controller methods
4. **NEVER assume `self` fields are up-to-date** in controller methods — the document may have unsaved changes from the client

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/methods.md*
