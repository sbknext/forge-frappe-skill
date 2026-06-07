---
title: Frappe Ref — Frappe Syntax: Controllers: Flags (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'flags']
source: Impertio
---

# Flags System Reference

Complete reference for the Frappe flags system used in Document Controllers.

---

## Two Levels of Flags

| Level | Access | Scope | Lifetime |
|---|---|---|---|
| Document flags | `doc.flags` | Single document instance | Current request |
| Request flags | `frappe.flags` | Global for entire request | Current request |

NEVER rely on flags persisting between requests. Flags are temporary, in-memory only.

---

## Document Flags (doc.flags)

### Permission Bypass Flags

```python
doc.flags.ignore_permissions = True   # Bypass all permission checks
doc.flags.ignore_validate = True      # Skip validate() method
doc.flags.ignore_mandatory = True     # Skip mandatory field checks
doc.flags.ignore_links = True         # Skip link field validation
doc.flags.ignore_version = True       # Skip version record creation
```

### Notification Flags

```python
doc.flags.notify_update = False   # [v15+] Suppress realtime browser update
```

### Passing Flags via insert/save

```python
doc.insert(
    ignore_permissions=True,
    ignore_links=True,
    ignore_if_duplicate=True,
    ignore_mandatory=True
)

doc.save(
    ignore_permissions=True,
    ignore_version=True
)
```

---

## Request Flags (frappe.flags)

### System State Flags

```python
if frappe.flags.in_import:
    return  # Skip heavy validation during data import

if frappe.flags.in_install:
    return  # Skip validation during app installation

if frappe.flags.in_patch:
    return  # Skip checks during migration/patch

if frappe.flags.in_migrate:
    return  # Skip checks during bench migrate

if frappe.flags.in_scheduler:
    pass  # Running in background scheduler job
```

### Email Control

```python
# Suppress ALL emails for current request
frappe.flags.mute_emails = True
for order in orders:
    frappe.get_doc("Sales Order", order).submit()  # No notification emails
frappe.flags.mute_emails = False
```

---

## Custom Flags for Inter-Hook Communication

### Pattern: Status Change Tracking

```python
class SalesInvoice(Document):
    def validate(self):
        old = self.get_doc_before_save()
        if old and old.status != self.status:
            self.flags.status_changed = True
            self.flags.old_status = old.status

    def on_update(self):
        if self.flags.get("status_changed"):
            self.log_status_change(self.flags.old_status, self.status)
```

### Pattern: Recursion Prevention

```python
class Task(Document):
    def on_update(self):
        if self.flags.get("updating_from_project"):
            return  # Prevent infinite loop

        project = frappe.get_doc("Project", self.project)
        project.flags.updating_from_task = True
        project.update_percent_complete()
        project.save()
```

### Pattern: Trigger Source Tracking

```python
class StockEntry(Document):
    def on_submit(self):
        if self.flags.get("from_purchase_receipt"):
            self.add_comment("Info", "Auto-created from Purchase Receipt")

# Called from another controller:
stock_entry.flags.from_purchase_receipt = True
stock_entry.submit()
```

### Pattern: Conditional Notifications

```python
class SalesOrder(Document):
    def validate(self):
        if self.grand_total > 10000:
            self.flags.high_value = True

    def on_submit(self):
        if self.flags.get("high_value"):
            self.notify_finance_team()
```

---

## Flag Best Practices

### ALWAYS check flags safely with get()

```python
# CORRECT - returns None if flag not set
if self.flags.get("high_value"):
    pass

# CORRECT - with default value
if self.flags.get("retry_count", 0) > 3:
    pass

# WRONG - raises AttributeError if not set
if self.flags.high_value:  # Risky!
    pass
```

### NEVER persist flags to database

```python
# WRONG - flags are temporary, not for storage
def validate(self):
    self.some_db_field = self.flags.get("temp_value")

# CORRECT - flags communicate between hooks in same request only
def validate(self):
    self.flags.temp_value = compute_something()

def on_update(self):
    if self.flags.get("temp_value"):
        do_something()
```

### NEVER depend on flags between requests

```python
# WRONG - flag is gone after request ends
doc.flags.process_later = True
doc.save()
# Next request: doc.flags.process_later is None
```

---

## Complete Flag Reference

### doc.flags (Document Level)

| Flag | Type | Effect |
|---|---|---|
| `ignore_permissions` | bool | Bypass permission checks |
| `ignore_validate` | bool | Skip validate() method |
| `ignore_mandatory` | bool | Skip mandatory field checks |
| `ignore_links` | bool | Skip link validation |
| `ignore_version` | bool | Skip version record creation |
| `notify_update` | bool | [v15+] Control realtime updates |

### frappe.flags (Request Level)

| Flag | Type | Effect |
|---|---|---|
| `in_import` | bool | Data import active |
| `in_install` | bool | App installation active |
| `in_patch` | bool | Patch execution active |
| `in_migrate` | bool | Migration active |
| `in_scheduler` | bool | Background scheduler active |
| `mute_emails` | bool | Suppress all emails |

---

## Bulk Operations with Flags

```python
def bulk_update_status(doc_names, new_status):
    """Update documents without validation or emails."""
    frappe.flags.mute_emails = True

    for name in doc_names:
        doc = frappe.get_doc("Sales Order", name)
        doc.flags.ignore_permissions = True
        doc.flags.ignore_validate = True
        doc.status = new_status
        doc.save()

    frappe.flags.mute_emails = False
```

```python
def import_legacy_data(records):
    """Import with all checks bypassed."""
    frappe.flags.in_import = True

    for record in records:
        doc = frappe.get_doc({"doctype": "Customer", **record})
        doc.flags.ignore_permissions = True
        doc.flags.ignore_mandatory = True
        doc.flags.ignore_links = True
        doc.flags.ignore_validate = True
        doc.insert()

    frappe.flags.in_import = False
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/flags.md*
