---
title: Frappe Ref — Frappe Syntax: Hooks: Overrides (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'reference', 'overrides']
source: Impertio
---

# Override Hooks Reference

Complete reference for override hooks in hooks.py.

---

## override_doctype_class [v14+]

Fully replace the controller class of a DocType. The LAST installed app wins
when multiple apps override the same DocType.

### Syntax

```python
# hooks.py
override_doctype_class = {
    "Sales Invoice": "myapp.overrides.CustomSalesInvoice",
    "ToDo": "myapp.overrides.todo.CustomToDo"
}
```

### Implementation

```python
# myapp/overrides.py
from erpnext.accounts.doctype.sales_invoice.sales_invoice import SalesInvoice

class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        super().validate()  # ALWAYS call super() first
        self.custom_validation()

    def on_submit(self):
        super().on_submit()
        self.create_custom_entry()

    def custom_validation(self):
        if self.grand_total > 100000:
            frappe.msgprint("High value invoice - requires approval")
```

### Warnings

1. **Last app wins**: When multiple apps override the same DocType, ONLY the last installed app's override is active
2. **Fragile**: Updates to the parent class can break your override
3. **ALWAYS call super()**: Forgetting `super()` breaks core functionality (validations, calculations, ledger entries)

---

## extend_doctype_class [v16+]

Add mixins to a controller WITHOUT replacing it. All apps' extensions coexist.
ALWAYS prefer this over `override_doctype_class` on v16+.

### Syntax

```python
# hooks.py
extend_doctype_class = {
    "Address": ["myapp.extensions.address.AddressMixin"],
    "Contact": [
        "myapp.extensions.common.ValidationMixin",
        "myapp.extensions.contact.ContactMixin"
    ]
}
```

### Implementation

```python
# myapp/extensions/address.py
import re
import frappe
from frappe.model.document import Document

class AddressMixin(Document):
    @property
    def full_address(self):
        """Computed property added to Address."""
        return f"{self.address_line1}, {self.city}, {self.country}"

    def validate(self):
        super().validate()
        self.validate_postal_code()

    def validate_postal_code(self):
        if self.country == "Netherlands" and self.pincode:
            if not re.match(r'^\d{4}\s?[A-Z]{2}$', self.pincode):
                frappe.throw("Invalid Dutch postal code format")
```

### Comparison

| Aspect | override_doctype_class | extend_doctype_class |
|--------|------------------------|----------------------|
| Multiple apps | Last app wins | ALL coexist |
| Maintenance | Fragile | Stable |
| Availability | v14+ | v16+ only |
| Risk | High (can break core) | Low (additive) |

---

## override_whitelisted_methods [v14+]

Replace existing API endpoints with custom implementations.

### Syntax

```python
# hooks.py
override_whitelisted_methods = {
    "frappe.client.get_count": "myapp.overrides.custom_get_count",
    "erpnext.selling.doctype.sales_order.sales_order.make_sales_invoice":
        "myapp.overrides.custom_make_sales_invoice"
}
```

### Implementation

```python
# CRITICAL: Method signature MUST be identical to original
def custom_get_count(doctype, filters=None, debug=False, cache=False):
    count = frappe.db.count(doctype, filters)
    log_count_query(doctype)  # Custom addition
    return count
```

ALWAYS find the original function signature first. Mismatched parameters cause
runtime errors.

### Commonly Overridden Methods

| Method | Purpose |
|--------|---------|
| `frappe.client.get_count` | Record counting |
| `frappe.client.get_list` | List queries |
| `frappe.desk.search.search_link` | Link field search |
| `erpnext.*.make_*` | Document creation wizards |

---

## standard_queries [v14+]

Replace the default search query for Link fields.

### Syntax

```python
# hooks.py
standard_queries = {
    "Customer": "myapp.queries.customer_query"
}
```

### Implementation

```python
# myapp/queries.py
import frappe

def customer_query(doctype, txt, searchfield, start, page_len, filters):
    """
    Args:
        doctype: "Customer"
        txt: Search text typed by user
        searchfield: Field being searched (usually "name")
        start: Pagination offset
        page_len: Page size
        filters: Additional filters from the Link field
    """
    return frappe.db.sql("""
        SELECT name, customer_name, customer_group
        FROM `tabCustomer`
        WHERE (name LIKE %(txt)s OR customer_name LIKE %(txt)s)
            AND status = 'Active'
        ORDER BY customer_name
        LIMIT %(start)s, %(page_len)s
    """, {
        "txt": f"%{txt}%",
        "start": start,
        "page_len": page_len
    })
```

---

## doctype_js [v14+]

Extend form scripts of existing DocTypes from your custom app.

### Syntax

```python
# hooks.py
doctype_js = {
    "Sales Invoice": "public/js/sales_invoice.js",
    "Customer": "public/js/customer.js"
}
```

### Implementation

```javascript
// public/js/sales_invoice.js
frappe.ui.form.on("Sales Invoice", {
    refresh: function(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__("Send to ERP"), function() {
                frappe.call({
                    method: "myapp.api.send_invoice",
                    args: { invoice: frm.doc.name },
                    callback: function(r) {
                        frappe.msgprint(__("Invoice sent"));
                    }
                });
            });
        }
    },

    customer: function(frm) {
        if (frm.doc.customer) {
            frappe.call({
                method: "myapp.api.get_customer_discount",
                args: { customer: frm.doc.customer },
                callback: function(r) {
                    if (r.message) {
                        frm.set_value("discount_percentage", r.message);
                    }
                }
            });
        }
    }
});
```

---

## doctype_list_js [v14+]

Extend list view scripts of existing DocTypes.

### Syntax

```python
# hooks.py
doctype_list_js = {
    "Sales Invoice": "public/js/sales_invoice_list.js"
}
```

### Implementation

```javascript
// public/js/sales_invoice_list.js
frappe.listview_settings["Sales Invoice"] = {
    add_fields: ["customer_name", "grand_total"],
    get_indicator: function(doc) {
        if (doc.grand_total > 100000) {
            return [__("High Value"), "orange", "grand_total,>,100000"];
        }
    }
};
```

---

## Hook Resolution Order

When multiple apps define the same hook:

```
Override hooks (override_*):   Last installed app wins
Extend hooks (extend_*, doc_events): All handlers run in installation order
```

Adjust order via: Setup > Installed Applications > Update Hooks Resolution Order

---

## Decision Tree

```
Want to modify existing functionality?
|
+-- API endpoint?
|   +-- override_whitelisted_methods
|
+-- DocType controller?
|   +-- v16+? --> extend_doctype_class (RECOMMENDED)
|   +-- v14/v15? --> override_doctype_class (last app wins)
|
+-- Form UI?
|   +-- doctype_js
|
+-- List view UI?
|   +-- doctype_list_js
|
+-- Link field search?
    +-- standard_queries
```

---

## Version Differences

| Hook | v14 | v15 | v16 |
|------|-----|-----|-----|
| override_whitelisted_methods | Yes | Yes | Yes |
| override_doctype_class | Yes | Yes | Yes |
| extend_doctype_class | -- | -- | NEW |
| doctype_js | Yes | Yes | Yes |
| doctype_list_js | Yes | Yes | Yes |
| standard_queries | Yes | Yes | Yes |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks/references/overrides.md*
