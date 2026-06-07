---
title: Frappe Ref — Frappe Syntax: Controllers: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns Reference

Common controller mistakes and their correct alternatives.

---

## Lifecycle Hook Mistakes

### WRONG: Modifying self in on_update

Changes to `self` after `on_update` are NOT saved to database.

```python
# WRONG - change is lost
def on_update(self):
    self.status = "Completed"  # NOT saved

# CORRECT - use db_set for post-save changes
def on_update(self):
    self.db_set("status", "Completed")
    # Or for multiple fields:
    self.db_set({"status": "Completed", "completed_at": frappe.utils.now()})

# CORRECT - move calculation to validate (changes ARE saved)
def validate(self):
    if self.all_items_delivered():
        self.status = "Completed"
```

### WRONG: Calling save() in on_update (infinite loop)

```python
# WRONG - infinite recursion: on_update -> save -> on_update -> ...
def on_update(self):
    self.counter = (self.counter or 0) + 1
    self.save()

# CORRECT - use db_set (no hooks triggered)
def on_update(self):
    new_count = (self.counter or 0) + 1
    self.db_set("counter", new_count, update_modified=False)

# CORRECT - use flag to prevent recursion
def on_update(self):
    if self.flags.get("in_recursive_update"):
        return
    self.flags.in_recursive_update = True
    # ... operations ...
```

### WRONG: Validation in on_update

```python
# WRONG - document already saved when this throws
def on_update(self):
    if self.grand_total < 0:
        frappe.throw("Invalid total")  # Data already in DB!

# CORRECT - validate BEFORE save
def validate(self):
    if self.grand_total < 0:
        frappe.throw("Invalid total")  # Blocks save entirely
```

### WRONG: Using after_insert for all-save logic

```python
# WRONG - only runs on first save, never on updates
def after_insert(self):
    self.send_notification()  # Never triggers on update!

# CORRECT - use on_update for all saves, check if needed
def on_update(self):
    if self.is_new():
        self.send_welcome_notification()
    else:
        self.send_update_notification()
```

---

## Database Mistakes

### WRONG: Calling frappe.db.commit() in controllers

```python
# WRONG - breaks transaction management
def on_update(self):
    frappe.db.sql("UPDATE tabItem SET ...")
    frappe.db.commit()  # Can cause partial updates on error

# CORRECT - Frappe commits automatically at end of request
def on_update(self):
    frappe.db.sql("UPDATE tabItem SET ...")
    # No commit needed
```

### WRONG: Using db_insert/db_update for normal operations

```python
# WRONG - bypasses ALL hooks and validation
def create_related(self):
    doc = frappe.get_doc({"doctype": "Task", "title": "New"})
    doc.db_insert()  # No validate, no permissions

# CORRECT - use insert() for normal operations
def create_related(self):
    doc = frappe.get_doc({"doctype": "Task", "title": "New"})
    doc.insert()  # All hooks and validation run
```

### WRONG: SQL injection via string formatting

```python
# WRONG - SQL injection vulnerability
def get_items(self):
    return frappe.db.sql(f"SELECT * FROM tabItem WHERE name = '{self.item_code}'")

# CORRECT - parameterized query
def get_items(self):
    return frappe.db.sql("SELECT * FROM tabItem WHERE name = %s", [self.item_code])

# CORRECT - use ORM
def get_items(self):
    return frappe.get_all("Item", filters={"name": self.item_code})
```

---

## Permission Mistakes

### WRONG: No permission check on whitelisted methods

```python
# WRONG - anyone can update salary
@frappe.whitelist()
def update_salary(employee, new_salary):
    frappe.db.set_value("Employee", employee, "salary", new_salary)

# CORRECT - check permissions
@frappe.whitelist()
def update_salary(employee, new_salary):
    if not frappe.has_permission("Employee", "write"):
        frappe.throw(_("Not permitted"))
    if "HR Manager" not in frappe.get_roles():
        frappe.throw(_("Only HR Manager can update salary"))
    frappe.db.set_value("Employee", employee, "salary", new_salary)
```

### WRONG: ignore_permissions everywhere

```python
# WRONG - security bypass without justification
def on_update(self):
    doc = frappe.get_doc("Sales Invoice", self.invoice)
    doc.flags.ignore_permissions = True
    doc.submit()

# CORRECT - only where justified, with documentation
def on_update(self):
    # System operation: auto-submit invoice from approved order
    # Permission bypass justified: order approval already verified
    doc = frappe.get_doc("Sales Invoice", self.invoice)
    doc.flags.ignore_permissions = True
    doc.submit()
```

---

## Performance Mistakes

### WRONG: N+1 query problem

```python
# WRONG - N database queries in loop
def validate(self):
    for item in self.items:
        stock = frappe.db.get_value("Bin",
            {"item_code": item.item_code, "warehouse": item.warehouse},
            "actual_qty"
        )  # 1 query per item!

# CORRECT - batch query
def validate(self):
    item_codes = [item.item_code for item in self.items]
    stock_data = frappe.get_all("Bin",
        filters={"item_code": ["in", item_codes]},
        fields=["item_code", "warehouse", "actual_qty"]
    )
    stock_map = {(d.item_code, d.warehouse): d.actual_qty for d in stock_data}

    for item in self.items:
        stock = stock_map.get((item.item_code, item.warehouse), 0)
        if item.qty > stock:
            frappe.throw(f"Insufficient stock for {item.item_code}")
```

### WRONG: Heavy operations in validate

```python
# WRONG - blocks UI for 30+ seconds
def validate(self):
    self.generate_100_page_pdf()
    self.send_emails_to_all_customers()

# CORRECT - enqueue heavy tasks
def on_update(self):
    frappe.enqueue(
        "myapp.tasks.generate_pdf",
        queue="long",
        doc_name=self.name,
        enqueue_after_commit=True
    )
```

### WRONG: Not using cache for repeated lookups

```python
# WRONG - multiple DB calls for same record
def validate(self):
    customer = frappe.get_doc("Customer", self.customer)
    customer_email = frappe.get_value("Customer", self.customer, "email")

# CORRECT - use cached doc
def validate(self):
    customer = frappe.get_cached_doc("Customer", self.customer)
    email = customer.email
```

---

## Override Mistakes

### WRONG: Missing super() call

```python
# WRONG - all parent validation skipped
class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        self.my_check()  # Parent validate never runs!

# CORRECT - ALWAYS call super()
class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        super().validate()
        self.my_check()
```

### WRONG: doc_events with wrong signature

```python
# WRONG - incorrect parameter names
def on_validate(document):
    pass

# CORRECT - use (doc, method=None) signature
def on_validate(doc, method=None):
    pass
```

### WRONG: Full override for minor changes

```python
# WRONG - override entire class for one check
override_doctype_class = {"Sales Invoice": "myapp.override.CustomSI"}

class CustomSI(SalesInvoice):
    def validate(self):
        super().validate()
        if self.total < 100:
            frappe.msgprint("Small order")

# CORRECT - use doc_events for simple additions
doc_events = {
    "Sales Invoice": {
        "validate": "myapp.events.si_validate"
    }
}

def si_validate(doc, method=None):
    if doc.total < 100:
        frappe.msgprint("Small order")
```

---

## Async Mistakes

### WRONG: Enqueue without error handling

```python
# WRONG - failures are silently lost
def on_submit(self):
    frappe.enqueue("myapp.tasks.process", doc_name=self.name)

# CORRECT - with error handling and retry
def on_submit(self):
    frappe.enqueue(
        "myapp.tasks.process",
        doc_name=self.name,
        queue="short",
        timeout=300,
        retry=3,
        enqueue_after_commit=True
    )

# myapp/tasks.py
def process(doc_name):
    try:
        doc = frappe.get_doc("MyDocType", doc_name)
        doc.do_processing()
    except Exception:
        frappe.log_error(title=f"Process failed: {doc_name}")
        raise  # Re-raise for retry mechanism
```

### WRONG: Synchronous external API in validate

```python
# WRONG - user waits for external API (could be 30+ seconds)
def validate(self):
    response = requests.get("https://api.external.com/validate", timeout=30)

# CORRECT - async external call
def on_update(self):
    frappe.enqueue(
        "myapp.integrations.validate_external",
        doc_name=self.name,
        queue="short",
        enqueue_after_commit=True
    )
```

---

## Summary

| Anti-Pattern | Correct Approach |
|---|---|
| `self.x = ...` in on_update | `self.db_set("x", ...)` |
| `self.save()` in on_update | `self.db_set()` or flags |
| Validation in on_update | Move to `validate` |
| `frappe.db.commit()` | Remove -- Frappe handles it |
| Query in loop | Batch query with `get_all` |
| Heavy ops in validate | `frappe.enqueue()` |
| Override without super() | ALWAYS `super().method()` |
| `ignore_permissions` everywhere | Only where justified |
| Sync external API in validate | Async with enqueue |
| f-string in SQL | Parameterized `%s` queries |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/anti-patterns.md*
