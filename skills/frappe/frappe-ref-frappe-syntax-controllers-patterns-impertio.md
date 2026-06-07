---
title: Frappe Ref — Frappe Syntax: Controllers: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'patterns']
source: Impertio
---

# Common Controller Patterns Reference

Proven patterns for Document Controllers in Frappe v14-v16.

---

## Naming Patterns

### Naming Series

Set `autoname = "naming_series:"` in DocType definition. Users select series from dropdown.

```python
# No controller code needed -- configured in DocType
# Series options defined in DocType: SO-.YYYY.-.#####
# Result: SO-2024-00001, SO-2024-00002
```

### Custom Autoname with getseries

```python
from frappe.model.naming import getseries

class Project(Document):
    def autoname(self):
        prefix = f"PRJ-{self.company_abbr}-"
        self.name = getseries(prefix, 5)
        # Result: PRJ-ABC-00001
```

### Autoname Based on Fields

```python
class EmployeeLeave(Document):
    def autoname(self):
        self.name = f"{self.employee}-{self.leave_type}-{self.from_date}"
```

### Document Naming Rules (No Code)

Configure in Setup > Document Naming Rules. Priority-based rules with conditions.
ALWAYS prefer Document Naming Rules over controller autoname for simple patterns.

---

## Validation Patterns

### Change Detection

```python
def validate(self):
    old = self.get_doc_before_save()
    if old is None:
        return  # New document, no changes to detect

    if old.status != self.status:
        self.validate_status_transition(old.status, self.status)

    if old.customer != self.customer:
        frappe.throw(_("Cannot change customer after creation"))
```

### Using has_value_changed [v15+]

```python
def validate(self):
    if self.has_value_changed("status"):
        self.status_changed_on = frappe.utils.now()
        self.flags.status_changed = True
```

### Protected Fields Pattern

```python
PROTECTED_AFTER_SUBMIT = ["customer", "company", "currency"]

def before_update_after_submit(self):
    old = self.get_doc_before_save()
    for field in PROTECTED_AFTER_SUBMIT:
        if old.get(field) != self.get(field):
            frappe.throw(_("Cannot change {0} after submit").format(field))
```

### Child Table Validation

```python
def validate(self):
    if not self.items:
        frappe.throw(_("At least one item is required"))

    seen = set()
    for idx, item in enumerate(self.items, 1):
        if item.item_code in seen:
            frappe.throw(_("Duplicate item {0} in row {1}").format(
                item.item_code, idx
            ))
        seen.add(item.item_code)

        if item.qty <= 0:
            frappe.throw(_("Quantity must be positive in row {0}").format(idx))

        item.amount = item.qty * item.rate

    self.total = sum(item.amount for item in self.items)
```

---

## Submit/Cancel Patterns

### Paired Submit and Cancel

ALWAYS implement on_cancel as the exact reverse of on_submit.

```python
def on_submit(self):
    self.create_ledger_entries()
    self.update_stock()
    self.update_party_balance()

def on_cancel(self):
    self.reverse_ledger_entries()
    self.reverse_stock()
    self.reverse_party_balance()
```

### Linked Document Check Before Cancel

```python
def before_cancel(self):
    linked = frappe.get_all(
        "Purchase Invoice Item",
        filters={"purchase_order": self.name, "docstatus": 1},
        pluck="parent"
    )
    if linked:
        frappe.throw(
            _("Cannot cancel: linked to {0}").format(", ".join(set(linked)))
        )
```

### Approval Gate Before Submit

```python
def before_submit(self):
    if self.grand_total > 50000 and not self.approved_by:
        frappe.throw(_("Manager approval required for orders over 50,000"))
```

---

## Workflow Integration Patterns

### Status Transition Validation

```python
VALID_TRANSITIONS = {
    "Draft": ["Pending Approval", "Cancelled"],
    "Pending Approval": ["Approved", "Rejected"],
    "Approved": ["In Progress"],
    "In Progress": ["Completed", "On Hold"],
    "On Hold": ["In Progress", "Cancelled"],
}

def validate(self):
    old = self.get_doc_before_save()
    if old and old.status != self.status:
        allowed = VALID_TRANSITIONS.get(old.status, [])
        if self.status not in allowed:
            frappe.throw(
                _("Cannot transition from {0} to {1}").format(old.status, self.status)
            )
```

---

## Communication Patterns

### Post-Save Notifications

```python
def on_update(self):
    if self.flags.get("status_changed"):
        frappe.publish_realtime(
            "order_status",
            {"name": self.name, "status": self.status},
            doctype=self.doctype,
            docname=self.name
        )
```

### Async Email via Enqueue

```python
def on_submit(self):
    frappe.enqueue(
        "myapp.email.send_order_confirmation",
        queue="short",
        doc_name=self.name,
        enqueue_after_commit=True
    )
```

---

## Performance Patterns

### Batch Database Reads

```python
def validate(self):
    # CORRECT: One query for all items
    item_codes = [item.item_code for item in self.items]
    prices = {
        d.item_code: d.price
        for d in frappe.get_all("Item Price",
            filters={"item_code": ["in", item_codes], "price_list": self.price_list},
            fields=["item_code", "price"]
        )
    }
    for item in self.items:
        item.rate = prices.get(item.item_code, 0)
```

### Cached Document Reads

```python
def validate(self):
    # CORRECT: Uses document cache
    customer = frappe.get_cached_doc("Customer", self.customer)
    self.customer_group = customer.customer_group
    self.territory = customer.territory
```

### Background Processing

```python
def on_submit(self):
    # Light operations inline
    self.db_set("submitted_at", frappe.utils.now())

    # Heavy operations in background
    self.queue_action("heavy_processing")

def heavy_processing(self):
    """Runs in background worker."""
    self.generate_reports()
    self.sync_external_systems()
```

---

## Flags Communication Pattern

Pass data between hooks in the same request:

```python
def validate(self):
    old = self.get_doc_before_save()
    if old and old.status != self.status:
        self.flags.status_changed = True
        self.flags.old_status = old.status

    if self.grand_total > 10000:
        self.flags.high_value = True

def on_update(self):
    if self.flags.get("status_changed"):
        self.add_comment("Edit",
            f"Status: {self.flags.old_status} -> {self.status}")

def on_submit(self):
    if self.flags.get("high_value"):
        self.request_additional_approval()
```

---

## Recursion Prevention Pattern

When two DocTypes update each other:

```python
class Task(Document):
    def on_update(self):
        if self.flags.get("from_project"):
            return  # Prevent loop

        project = frappe.get_doc("Project", self.project)
        project.flags.from_task = True
        project.update_progress()
        project.save()

class Project(Document):
    def on_update(self):
        if self.flags.get("from_task"):
            return  # Prevent loop

        for task in frappe.get_all("Task", filters={"project": self.name}):
            doc = frappe.get_doc("Task", task.name)
            doc.flags.from_project = True
            doc.expected_end_date = self.expected_end_date
            doc.save()
```

---

## Whitelisted Method Pattern

Expose controller methods to client-side JavaScript:

```python
class SalesOrder(Document):
    @frappe.whitelist()
    def apply_discount(self, discount_percent):
        """Called from JS: frm.call('apply_discount', {discount_percent: 10})"""
        for item in self.items:
            item.discount = discount_percent
            item.amount = item.qty * item.rate * (1 - discount_percent / 100)
        self.save()
        return {"new_total": sum(i.amount for i in self.items)}
```

ALWAYS add `@frappe.whitelist()` decorator. Without it, client calls return 403 Forbidden.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/patterns.md*
