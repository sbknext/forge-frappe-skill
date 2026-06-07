---
title: Frappe Ref — Frappe Syntax: Controllers: Hooks (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'hooks']
source: Impertio
---

# Controller-Hooks Interaction Reference

How controllers interact with hooks.py for extension, override, and event handling.

---

## Three Extension Mechanisms

| Mechanism | hooks.py Key | Scope | Multi-App Safe | Version |
|---|---|---|---|---|
| Override class | `override_doctype_class` | Full class replacement | No (conflicts) | All |
| Extend class | `extend_doctype_class` | Add methods/properties | Yes | v16+ |
| Hook events | `doc_events` | Individual events | Yes | All |

---

## override_doctype_class

Replaces the controller class entirely. Only ONE app can override a given DocType.

```python
# hooks.py
override_doctype_class = {
    "Sales Order": "myapp.overrides.sales_order.CustomSalesOrder",
    "Purchase Invoice": "myapp.overrides.purchase_invoice.CustomPurchaseInvoice"
}
```

```python
# myapp/overrides/sales_order.py
from erpnext.selling.doctype.sales_order.sales_order import SalesOrder

class CustomSalesOrder(SalesOrder):
    def validate(self):
        super().validate()  # ALWAYS call super()
        self.custom_validation()

    def on_submit(self):
        super().on_submit()  # ALWAYS call super()
        self.custom_submit_logic()
```

**Rules:**
- ALWAYS import and extend the original class
- ALWAYS call `super()` in overridden methods
- If two apps override the same DocType, only the last loaded wins
- NEVER use this in v16+ when `extend_doctype_class` suffices

---

## extend_doctype_class [v16+]

Adds methods and properties to existing controller via mixins. Multiple apps can extend the same DocType safely.

```python
# hooks.py
extend_doctype_class = {
    "Address": [
        "myapp.extensions.address.GeocodingMixin",
        "myapp.extensions.common.AuditMixin"
    ],
    "Contact": [
        "myapp.extensions.common.AuditMixin"
    ]
}
```

```python
# myapp/extensions/address.py
from frappe.model.document import Document

class GeocodingMixin(Document):
    @property
    def full_address(self):
        return f"{self.address_line1}, {self.city}, {self.country}"

    def validate(self):
        super().validate()
        self.geocode_if_changed()

    def geocode_if_changed(self):
        if self.has_value_changed("city"):
            self.latitude, self.longitude = self.fetch_coords()
```

**Rules:**
- ALWAYS call `super().method()` when overriding lifecycle hooks
- Mixin class MUST extend `Document` (or the appropriate base)
- Multiple mixins are applied in list order
- ALWAYS prefer this over `override_doctype_class` in v16+

---

## doc_events

Hook into individual document events without replacing the controller.

```python
# hooks.py
doc_events = {
    "Sales Order": {
        "validate": "myapp.events.so.validate",
        "on_submit": "myapp.events.so.on_submit",
        "on_cancel": "myapp.events.so.on_cancel",
        "after_insert": "myapp.events.so.after_insert",
        "on_trash": "myapp.events.so.on_trash",
        "on_change": "myapp.events.so.on_change",
    },
    # Wildcard: ALL DocTypes
    "*": {
        "after_insert": "myapp.events.audit.log_creation",
        "on_update": "myapp.events.audit.log_update",
    }
}
```

```python
# myapp/events/so.py
import frappe
from frappe import _

def validate(doc, method=None):
    """ALWAYS use this exact signature: (doc, method=None)"""
    if doc.total > 100000:
        doc.requires_approval = 1

def on_submit(doc, method=None):
    frappe.enqueue(
        "myapp.integrations.sync",
        queue="short",
        doc_name=doc.name,
        enqueue_after_commit=True
    )
```

**Rules:**
- Handler signature MUST be `def handler(doc, method=None):`
- Multiple apps can hook the same event on the same DocType
- Handlers run AFTER the controller method, in app install order
- Use wildcard `"*"` sparingly -- it runs for EVERY DocType

---

## Execution Order per Event

When an event fires, handlers execute in this order:

1. **Controller method** (the method on the DocType class)
2. **doc_events handlers** (from hooks.py, all installed apps, in install order)
3. **Server Scripts** (if configured for this DocType/event)
4. **Webhooks** (if configured for this DocType/event)

---

## Choosing the Right Mechanism

```
Do you need to...

Replace the entire controller class?
  -> override_doctype_class [all versions]
  -> WARNING: Only one app can do this per DocType

Add methods/properties to a class? (v16+)
  -> extend_doctype_class
  -> Multiple apps can extend safely

Hook into one or two events?
  -> doc_events
  -> Simplest, most compatible

Add behavior to ALL DocTypes?
  -> doc_events with "*" wildcard

Extend a controller in v14/v15?
  -> Use doc_events (safest)
  -> Or override_doctype_class (if you need full control)
```

---

## Other hooks.py Keys That Affect Controllers

### export_python_type_annotations [v15+]

```python
# hooks.py
export_python_type_annotations = True
```

Enables auto-generation of type annotations in controller files for IDE support.

### scheduler_events

```python
# hooks.py
scheduler_events = {
    "daily": ["myapp.tasks.daily_cleanup"],
    "hourly": ["myapp.tasks.hourly_sync"],
    "cron": {
        "0 */6 * * *": ["myapp.tasks.six_hourly_task"]
    }
}
```

These are NOT controller methods but standalone functions. NEVER put controller logic in scheduler events.

### override_whitelisted_methods

```python
# hooks.py
override_whitelisted_methods = {
    "frappe.client.get_count": "myapp.overrides.custom_get_count"
}
```

Override globally whitelisted API methods. Use with extreme caution.

---

## Common Patterns

### Combining doc_events with override

Use `doc_events` for event hooks and `override_doctype_class` only when you need new methods or properties:

```python
# hooks.py
override_doctype_class = {
    "Sales Order": "myapp.overrides.so.CustomSalesOrder"
}
doc_events = {
    "Purchase Order": {
        "validate": "myapp.events.po.validate"
    }
}
```

### Multiple Apps Extending Same DocType [v16+]

```python
# App A hooks.py
extend_doctype_class = {
    "Sales Invoice": ["app_a.extensions.TaxMixin"]
}

# App B hooks.py
extend_doctype_class = {
    "Sales Invoice": ["app_b.extensions.ShippingMixin"]
}

# Both mixins are applied. App install order determines method resolution.
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/hooks.md*
