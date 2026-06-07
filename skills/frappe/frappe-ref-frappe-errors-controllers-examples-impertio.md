---
title: Frappe Ref — Frappe Errors: Controllers: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'controllers', 'reference', 'examples']
source: Impertio
---

# Controller Error Examples — Real Scenarios

Complete diagnosis-oriented examples showing actual errors, root cause, and fix.

---

## Scenario 1: Autoname Failure — DuplicateEntryError

**Error:**
```
frappe.exceptions.DuplicateEntryError: ('Custom Doc', 'DOC-CustomerA', ...)
```

**The broken code:**
```python
class CustomDoc(Document):
    def autoname(self):
        self.name = f"DOC-{self.customer}"  # Not unique if customer has multiple docs!
```

**Root cause:** Using customer name as document name without a counter. Multiple documents for the same customer collide.

**The fix:**
```python
class CustomDoc(Document):
    def autoname(self):
        # Option 1: Name with auto-incrementing counter
        self.name = frappe.model.naming.make_autoname(
            f"DOC-{self.customer}-.####"
        )
        # Produces: DOC-CustomerA-0001, DOC-CustomerA-0002, etc.
```

**Alternative:** Set `autoname` in DocType JSON:
- `"autoname": "naming_series:"` — Uses naming_series field
- `"autoname": "format:DOC-{customer}-.####"` — Format with counter
- `"autoname": "hash"` — Random unique hash

---

## Scenario 2: Infinite Recursion — self.save() in validate

**Error:**
```
RecursionError: maximum recursion depth exceeded while calling a Python object
```

**The broken code:**
```python
class SalesOrder(Document):
    def validate(self):
        self.calculate_totals()
        self.save()  # validate → save → validate → save → ...
```

**Root cause:** `self.save()` triggers `validate()` again, creating infinite recursion.

**The fix:**
```python
class SalesOrder(Document):
    def validate(self):
        self.calculate_totals()
        # No self.save() — framework saves automatically after validate
```

**Same issue in on_update:**
```python
# ❌ WRONG
def on_update(self):
    self.status = "Updated"
    self.save()  # on_update → save → on_update → ...

# ✅ CORRECT
def on_update(self):
    self.db_set("status", "Updated")  # Direct DB write, no trigger
```

---

## Scenario 3: on_submit Without is_submittable

**Error:**
```
frappe.exceptions.DocstatusTransitionError: Cannot change docstatus from 0 to 1
```

**The broken code:**
```python
# In custom_doc.py
class CustomDoc(Document):
    def on_submit(self):
        self.create_stock_entries()
```
The DocType JSON lacks `"is_submittable": 1`.

**Root cause:** The submit action (docstatus 0 → 1) is blocked because the DocType is not marked as submittable.

**The fix:** Enable in DocType definition:
```json
{
    "name": "Custom Doc",
    "is_submittable": 1
}
```
Then `before_submit`, `on_submit`, `before_cancel`, and `on_cancel` hooks work.

---

## Scenario 4: Missing super() — Parent Validation Bypassed

**Error:** No error thrown, but critical business logic is silently skipped. For example, ERPNext's standard stock validations don't run.

**The broken code:**
```python
from erpnext.selling.doctype.sales_order.sales_order import SalesOrder

class CustomSalesOrder(SalesOrder):
    def validate(self):
        # All ERPNext validation skipped! Taxes, permissions, workflow — all bypassed.
        self.custom_check()
```

**The fix:**
```python
class CustomSalesOrder(SalesOrder):
    def validate(self):
        super().validate()  # ALWAYS call parent first
        self.custom_check()  # Then add custom logic
```

**This applies to ALL overridden hooks:** `validate`, `on_submit`, `on_cancel`, `before_submit`, etc.

---

## Scenario 5: Changes Lost in on_update

**Symptom:** Field is set in on_update, but database shows old value.

**The broken code:**
```python
class SalesOrder(Document):
    def on_update(self):
        self.sync_status = "Synced"
        self.sync_date = frappe.utils.now()
        # Changes are NOT saved — document already committed!
```

**Root cause:** `on_update` fires AFTER the save. Changes to `self` attributes are not persisted.

**The fix:**
```python
class SalesOrder(Document):
    def on_update(self):
        self.db_set({
            "sync_status": "Synced",
            "sync_date": frappe.utils.now()
        })
        # Or individual: self.db_set("sync_status", "Synced")
```

---

## Scenario 6: frappe.db.commit() Breaks Transaction

**Error:** Partial save — some data committed, other data lost on error.

**The broken code:**
```python
class SalesOrder(Document):
    def validate(self):
        self.calculate_totals()
        frappe.db.set_value("Counter", "main", "count", self.count + 1)
        frappe.db.commit()  # Commits counter change even if save fails!
```

**Root cause:** Manual `commit()` breaks Frappe's request-level transaction. If the save fails later, the counter is already committed but the document is rolled back.

**The fix:**
```python
class SalesOrder(Document):
    def validate(self):
        self.calculate_totals()
        frappe.db.set_value("Counter", "main", "count", self.count + 1)
        # No commit — framework manages the transaction
```

---

## Scenario 7: Validation in on_submit — Partial State

**Error:** Document is submitted (docstatus=1) but stock entries failed.

**The broken code:**
```python
class SalesOrder(Document):
    def on_submit(self):
        if not self.has_stock():
            frappe.throw(_("Insufficient stock"))
            # Document is ALREADY submitted! frappe.throw shows error
            # but docstatus is already 1 — inconsistent state!
```

**The fix:**
```python
class SalesOrder(Document):
    def before_submit(self):
        # ALWAYS validate here — last chance for clean abort
        if not self.has_stock():
            frappe.throw(_("Insufficient stock"))
            # Clean abort — document stays Draft

    def on_submit(self):
        # Only post-submit actions (create entries, send notifications)
        self.create_stock_entries()
```

---

## Scenario 8: NestedSet Circular Reference

**Error:**
```
frappe.exceptions.ValidationError: Item cannot be added as its own parent
```

**The broken code:**
```python
class Territory(NestedSet):
    def validate(self):
        pass  # No parent validation
```

**The fix:**
```python
from frappe.utils.nestedset import NestedSet

class Territory(NestedSet):
    nsm_parent_field = "parent_territory"

    def validate(self):
        super().validate()  # NestedSet checks circular refs
        # Additional check for self-reference
        if self.parent_territory == self.name:
            frappe.throw(_("Territory cannot be its own parent"))
```

---

## Scenario 9: extend_doctype_class Method Conflict [v16+]

**Symptom:** Two apps extend Sales Order — second app's validate replaces first app's.

**The broken code:**
```python
# App A: hooks.py
extend_doctype_class = {"Sales Order": ["app_a.overrides.SalesOrderMixin"]}

# App A: overrides.py
class SalesOrderMixin:
    def validate(self):
        self.custom_a_check()  # Missing super()!

# App B: hooks.py
extend_doctype_class = {"Sales Order": ["app_b.overrides.SalesOrderMixin"]}

# App B: overrides.py
class SalesOrderMixin:
    def validate(self):
        self.custom_b_check()  # Missing super()!
```

**Root cause:** Both mixins override `validate` without calling `super()`. Only the last one in MRO runs.

**The fix:**
```python
# App A
class SalesOrderMixin:
    def validate(self):
        super().validate()  # Calls next in MRO chain
        self.custom_a_check()

# App B
class SalesOrderMixin:
    def validate(self):
        super().validate()  # Calls App A's validate, then original
        self.custom_b_check()
```

---

## Scenario 10: Flags Not Used — Recursive Hook Trigger

**Symptom:** Updating a linked document in on_update triggers that document's on_update, which updates back, creating a loop.

**The broken code:**
```python
class SalesOrder(Document):
    def on_update(self):
        if self.quotation:
            q = frappe.get_doc("Quotation", self.quotation)
            q.db_set("status", "Ordered")
            # If Quotation.on_update updates SalesOrder back → loop!
```

**The fix:**
```python
class SalesOrder(Document):
    def on_update(self):
        if self.flags.get("from_quotation_update"):
            return  # Break the cycle

        if self.quotation:
            q = frappe.get_doc("Quotation", self.quotation)
            q.flags.from_order_update = True
            q.db_set("status", "Ordered")
```

---

## Quick Diagnosis by Error Type

| Error Type | Likely Hook | Common Cause |
|-----------|-------------|--------------|
| `RecursionError` | validate / on_update | `self.save()` in hook |
| `DuplicateEntryError` | autoname | Non-unique name generation |
| `DocstatusTransitionError` | on_submit | `is_submittable` not set |
| `ValidationError` (missing) | validate | `super()` not called |
| Changes lost | on_update | `self.field = x` instead of `db_set()` |
| Partial state | on_submit | Validation too late |
| Circular ref | validate (NestedSet) | Parent set to self |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-controllers/references/examples.md*
