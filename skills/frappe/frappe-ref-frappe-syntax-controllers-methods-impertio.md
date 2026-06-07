---
title: Frappe Ref — Frappe Syntax: Controllers: Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'methods']
source: Impertio
---

# Document Class Methods Reference

Complete reference for all `doc.*` methods available in Frappe Document Controllers.

---

## Data Access Methods

### doc.get(fieldname, default=None)

Safely retrieve field value with optional default.

```python
customer = self.get("customer")
status = self.get("status", "Draft")
items = self.get("items", [])  # Child table returns list
```

### doc.set(fieldname, value)

Set field value. Works for all field types including child tables.

```python
self.set("status", "Completed")
self.set("items", [{"item_code": "ITEM-001", "qty": 10}])  # Replaces child table
```

### doc.as_dict(no_nulls=False, no_default_fields=False)

Serialize document to dictionary.

```python
data = doc.as_dict()
data_clean = doc.as_dict(no_nulls=True)  # Skip None fields
data_minimal = doc.as_dict(no_default_fields=True)  # Skip name, owner, creation, etc.
```

### doc.get_valid_dict(sanitize=True, convert_dates_to_str=False)

Return dictionary with only valid fields (filtered by permissions and docfield meta).

```python
valid_data = self.get_valid_dict()
export_data = self.get_valid_dict(convert_dates_to_str=True)
```

### doc.has_value_changed(fieldname)

Check if a specific field changed since last save. Available in validate and later hooks.

```python
def validate(self):
    if self.has_value_changed("status"):
        self.status_changed_on = frappe.utils.now()
```

### doc.get_doc_before_save()

Return document as it was before current modifications. Returns `None` for new documents.

```python
def validate(self):
    old = self.get_doc_before_save()
    if old is None:
        pass  # New document
    elif old.status != self.status:
        self.log_status_change(old.status, self.status)
```

### doc.is_new()

Check if document is new (not yet saved to database).

```python
def validate(self):
    if self.is_new():
        self.status = "Draft"
```

---

## Database Operations

### doc.insert(ignore_permissions=False, ignore_links=False, ignore_if_duplicate=False, ignore_mandatory=False)

Insert new document with all lifecycle hooks.

```python
doc = frappe.get_doc({"doctype": "Task", "subject": "New Task"})
doc.insert()
doc.insert(ignore_permissions=True, ignore_mandatory=True)  # Bypass checks
```

### doc.save(ignore_permissions=False, ignore_version=True)

Save existing document with all lifecycle hooks.

```python
doc.customer = "New Customer"
doc.save()
doc.save(ignore_permissions=True)
```

### doc.submit()

Submit document (docstatus 0 -> 1). ONLY for submittable DocTypes.

```python
doc = frappe.get_doc("Sales Order", "SO-00001")
doc.submit()  # Triggers before_submit, on_submit
```

### doc.cancel()

Cancel document (docstatus 1 -> 2).

```python
doc.cancel()  # Triggers before_cancel, on_cancel
```

### doc.delete()

Delete document from database.

```python
doc.delete()  # Triggers on_trash, after_delete
```

### doc.reload()

Reload document from database with latest values.

```python
def on_update(self):
    self.reload()  # Get latest values after DB write
```

### doc.db_set(fieldname, value, notify=True, commit=False, update_modified=True)

Direct database field update. ALWAYS use this instead of modifying self in post-save hooks.

```python
def on_update(self):
    self.db_set("processed", 1)
    self.db_set("status", "Completed", update_modified=False)
    # Or update multiple fields:
    self.db_set({"status": "Completed", "processed_at": frappe.utils.now()})
```

---

## Low-Level Database Methods (USE WITH CAUTION)

### doc.db_insert(*args, **kwargs)

Direct database insert. Bypasses ALL hooks and validation.

```python
# ONLY for bulk operations or Virtual DocTypes
doc.db_insert()  # No validate, no permissions, no hooks
```

### doc.db_update()

Direct database update. Bypasses ALL hooks and validation.

```python
# ONLY for performance-critical bulk operations
doc.db_update()  # No validate, no permissions, no hooks
```

ALWAYS prefer `insert()` and `save()` over `db_insert()` and `db_update()`.

---

## Child Table Methods

### doc.append(fieldname, value=None)

Add row to child table.

```python
row = self.append("items", {
    "item_code": "ITEM-001",
    "qty": 10,
    "rate": 100
})
# row is the new child document object
```

### doc.extend(fieldname, values)

Add multiple rows to child table.

```python
self.extend("items", [
    {"item_code": "ITEM-001", "qty": 10},
    {"item_code": "ITEM-002", "qty": 5},
])
```

### Iterating child tables

```python
for item in self.get("items"):
    print(item.item_code, item.qty)

# Filter
high_value = [i for i in self.items if i.amount > 1000]
```

---

## Method Execution

### doc.run_method(method_name, *args, **kwargs)

Execute controller method AND trigger associated hooks (doc_events, server scripts).

```python
doc.run_method("validate")  # Runs validate + all doc_events for validate
doc.run_method("custom_method", value="test")
```

### doc.queue_action(action, **kwargs)

Execute controller method asynchronously in background.

```python
def on_submit(self):
    self.queue_action("send_emails", emails=email_list)

def send_emails(self, emails):
    for email in emails:
        frappe.sendmail(recipients=email, message="Order submitted")
```

---

## Permission Methods

### doc.has_permission(permtype="read", user=None)

Check if user has permission on this document.

```python
if not self.has_permission("write"):
    frappe.throw(_("No write permission"))
```

### doc.check_permission(permtype="read")

Same as has_permission but throws if no permission.

```python
self.check_permission("submit")  # Throws if not permitted
```

---

## Communication Methods

### doc.add_comment(comment_type, text, comment_email=None, comment_by=None)

Add comment to document timeline.

```python
self.add_comment("Edit", "Document updated by system")
self.add_comment("Info", f"Status changed to {self.status}")
```

Comment types: `Comment`, `Edit`, `Created`, `Submitted`, `Cancelled`, `Info`, `Label`, `Shared`, `Assigned`, `Attachment`

### doc.notify_update()

Publish realtime event that document has changed. Triggers form refresh in browser.

```python
frappe.db.set_value("Sales Order", self.name, "status", "Closed")
self.notify_update()  # Browser refreshes automatically
```

---

## Utility Methods

### doc.get_title()

Return document title (uses title_field configuration).

```python
title = doc.get_title()  # e.g., customer name for Sales Order
```

### doc.get_url()

Return desk URL for this document.

```python
url = doc.get_url()  # /app/sales-order/SO-00001
```

### doc.add_tag(tag_name)

```python
doc.add_tag("urgent")
```

### doc.get_tags()

```python
tags = doc.get_tags()  # ["urgent", "reviewed"]
```

---

## Method Summary

| Method | Parameters | Returns | Saves to DB |
|---|---|---|---|
| `get(field, default)` | str, any | any | No |
| `set(field, value)` | str, any | None | No |
| `as_dict()` | no_nulls, no_default_fields | dict | No |
| `is_new()` | - | bool | No |
| `has_value_changed(field)` | str | bool | No |
| `get_doc_before_save()` | - | Document/None | No |
| `insert(**flags)` | kwargs | self | Yes |
| `save(**flags)` | kwargs | self | Yes |
| `submit()` | - | self | Yes |
| `cancel()` | - | self | Yes |
| `delete()` | - | None | Yes |
| `reload()` | - | None | No |
| `db_set(field, value)` | str/dict, any | None | Yes |
| `run_method(method)` | str, args | any | No |
| `queue_action(action)` | str, kwargs | None | No |
| `append(field, value)` | str, dict | row | No |
| `has_permission(perm)` | str | bool | No |
| `add_comment(type, text)` | str, str | Comment | Yes |
| `notify_update()` | - | None | No |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/methods.md*
