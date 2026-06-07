---
title: Frappe Ref — Frappe Core: Database: Methods Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'database', 'reference', 'methods-reference']
source: Impertio
---

# Database & Document Methods Reference

> Complete API reference for frappe.db.* and Document methods. Verified against Frappe v14-v16 official docs.

---

## Document API (Global Functions)

### frappe.get_doc(doctype, name)
Returns a Document object. Raises `DoesNotExistError` if not found.
```python
doc = frappe.get_doc('Sales Invoice', 'SINV-00001')

# Single DocType — no name needed
settings = frappe.get_doc('System Settings')

# Create from dict (in-memory, not saved)
doc = frappe.get_doc({'doctype': 'Task', 'subject': 'New task'})
```

### frappe.get_cached_doc(doctype, name)
Same as `get_doc` but checks Redis cache first. ALWAYS use for read-only access to rarely-changing documents.
```python
company = frappe.get_cached_doc('Company', 'My Company')
```

### frappe.new_doc(doctype)
Creates a new Document with defaults applied.
```python
doc = frappe.new_doc('Task')
doc.subject = 'Review report'
doc.insert()
```

### frappe.get_last_doc(doctype, filters=None, order_by='creation desc')
Returns the most recently created document matching filters.
```python
last_task = frappe.get_last_doc('Task')
last_open = frappe.get_last_doc('Task', filters={'status': 'Open'})
```

### frappe.delete_doc(doctype, name)
Deletes document and its children, linked Communications, Comments, etc.
```python
frappe.delete_doc('Task', 'TASK-001')
```

### frappe.rename_doc(doctype, old_name, new_name, merge=False)
Renames document primary key. Requires "Allow Rename" on DocType.
```python
frappe.rename_doc('Task', 'OLD-NAME', 'NEW-NAME')
frappe.rename_doc('Customer', 'Old Co', 'New Co', merge=True)  # Merge if exists
```

### frappe.get_meta(doctype)
Returns DocType metadata with custom fields and property setters applied.
```python
meta = frappe.get_meta('Sales Invoice')
fields = meta.fields
```

---

## Document Instance Methods

### doc.insert()
Inserts new document into database. Runs validate hooks.
```python
doc.insert()
doc.insert(ignore_permissions=True)
doc.insert(ignore_links=True)
doc.insert(ignore_if_duplicate=True)
doc.insert(ignore_mandatory=True)
```

### doc.save()
Saves changes to existing document. Runs validate and on_update hooks.
```python
doc.save()
doc.save(ignore_permissions=True)
doc.save(ignore_version=True)  # No version record
```

### doc.delete()
Deletes the document. Alias to `frappe.delete_doc`.

### doc.submit()
Submits document (sets docstatus=1). Only for submittable DocTypes.

### doc.cancel()
Cancels submitted document (sets docstatus=2).

### doc.amend()
Creates amendment copy of cancelled document.

### doc.db_set(field, value, **kwargs)
Sets field value directly in database. SKIPS all validations and hooks.
```python
doc.db_set('status', 'Closed')
doc.db_set({'status': 'Closed', 'priority': 'High'})
doc.db_set('status', 'Closed', update_modified=False)
doc.db_set('status', 'Closed', commit=True)
doc.db_set('status', 'Closed', notify=True)  # Triggers realtime update
```

### doc.reload()
Refreshes document with latest database values.
```python
doc.reload()
```

### doc.get_doc_before_save()
Returns document state before current save operation. Use in validate/on_update.
```python
old_doc = doc.get_doc_before_save()
```

### doc.has_value_changed(fieldname)
Returns True if field value changed during current save.
```python
if doc.has_value_changed('status'):
    notify_status_change(doc)
```

### doc.check_permission(permtype)
Throws `frappe.PermissionError` if user lacks specified permission.
```python
doc.check_permission('write')
```

### doc.append(child_table_field, values)
Appends row to child table.
```python
doc.append('items', {'item_code': 'ITEM-001', 'qty': 10})
```

### doc.get_url()
Returns Desk URL for the document.
```python
url = doc.get_url()  # e.g., '/app/task/TASK-001'
```

### doc.add_comment(comment_type, text)
Adds comment visible in document timeline.

### doc.add_tag(tag)
Adds tag for filtering/grouping.

### doc.get_tags()
Returns list of document tags.

### doc.run_method(method_name)
Executes controller method with hooks.

### doc.queue_action(method, **kwargs)
Runs controller method in background via job queue.

### doc.db_insert() / doc.db_update()
Low-level insert/update bypassing all validations. NEVER use in application code — use `doc.insert()` and `doc.save()` instead.

---

## Database API (frappe.db.*)

### frappe.db.get_list(doctype, **kwargs)
Returns list of records with user permissions applied.
```python
tasks = frappe.db.get_list('Task',
    filters={'status': 'Open'},
    or_filters={'priority': 'Urgent'},
    fields=['name', 'subject'],
    order_by='creation desc',
    group_by='status',
    start=0,
    page_length=20
)
```

### frappe.get_all(doctype, **kwargs)
Same as `get_list` but with `ignore_permissions=True`. Use for system/admin context.
```python
all_tasks = frappe.get_all('Task', filters={'status': 'Open'}, pluck='name')
```

### frappe.db.get_value(doctype, name_or_filters, fieldname, **kwargs)
Returns field value(s) from a single record.
```python
# Single field → scalar
status = frappe.db.get_value('Task', 'TASK-001', 'status')

# Multiple fields → tuple
subject, status = frappe.db.get_value('Task', 'TASK-001', ['subject', 'status'])

# As dict
data = frappe.db.get_value('Task', 'TASK-001', ['subject', 'status'], as_dict=True)

# With filters
status = frappe.db.get_value('Task', {'project': 'PROJ-001'}, 'status')

# Cached
country = frappe.db.get_value('Company', 'X', 'country', cache=True)
```

### frappe.db.get_single_value(doctype, fieldname)
Returns field from Single DocType.
```python
tz = frappe.db.get_single_value('System Settings', 'time_zone')
```

### frappe.db.set_value(doctype, name, fieldname, value=None, **kwargs)
Direct database update. SKIPS ORM validations and hooks.
```python
frappe.db.set_value('Task', 'TASK-001', 'status', 'Closed')
frappe.db.set_value('Task', 'TASK-001', {'status': 'Closed', 'priority': 'Low'})
frappe.db.set_value('Task', 'TASK-001', 'status', 'Closed', update_modified=False)
```

### frappe.db.exists(doctype, name_or_filters, cache=False)
Boolean existence check. ALWAYS use instead of try/except with get_doc.
```python
exists = frappe.db.exists('User', 'admin@example.com')
exists = frappe.db.exists('User', {'email': 'admin@example.com'})
exists = frappe.db.exists('User', 'admin@example.com', cache=True)
```

### frappe.db.count(doctype, filters=None)
Returns integer count of matching records.
```python
total = frappe.db.count('Task')
open_count = frappe.db.count('Task', {'status': 'Open'})
```

### frappe.db.delete(doctype, filters)
DML DELETE — can be rolled back.
```python
frappe.db.delete('Error Log', {'creation': ['<', '2024-01-01']})
```

### frappe.db.truncate(doctype)
DDL TRUNCATE — auto-commits, CANNOT be rolled back.
```python
frappe.db.truncate('Error Log')
```

### frappe.db.sql(query, values=None, **kwargs)
Executes raw SQL. ALWAYS use parameterized queries.
```python
results = frappe.db.sql("""
    SELECT name, subject FROM `tabTask`
    WHERE status = %(status)s
""", {'status': 'Open'}, as_dict=True)
```

**Return types:**
- Default: tuple of tuples
- `as_dict=True`: list of dicts
- `as_list=True`: list of lists
- `debug=True`: prints SQL to console

### frappe.db.multisql(queries)
Executes database-engine-specific SQL.
```python
frappe.db.multisql({
    'mariadb': "SELECT IFNULL(field, 0) FROM `tabDoc`",
    'postgres': "SELECT COALESCE(field, 0) FROM `tabDoc`"
})
```

### frappe.db.bulk_update(doctype, doc_updates, **kwargs) [v15+]
Batch updates using CASE expressions. Direct DB — skips ORM.
```python
frappe.db.bulk_update('Task', {
    'TASK-001': {'status': 'Closed'},
    'TASK-002': {'status': 'Closed'}
}, chunk_size=100, update_modified=True)
```

---

## Transaction Control

### frappe.db.commit()
Manual commit. NEVER call inside controller hooks.

### frappe.db.rollback(save_point=None)
Rollback to savepoint or entire transaction.

### frappe.db.savepoint(save_point)
Creates named savepoint.
```python
frappe.db.savepoint('my_savepoint')
try:
    # operations
except Exception:
    frappe.db.rollback(save_point='my_savepoint')
```

### Transaction Hooks [v15+]
```python
frappe.db.before_commit.add(func)
frappe.db.after_commit.add(func)
frappe.db.before_rollback.add(func)
frappe.db.after_rollback.add(func)
```

---

## Schema Methods

### frappe.db.add_index(doctype, fields, index_name=None)
```python
frappe.db.add_index('Task', ['status', 'priority'])
frappe.db.add_index('Task', ['description(500)'])  # TEXT fields need length
```

### frappe.db.add_unique(doctype, fields, constraint_name=None)
```python
frappe.db.add_unique('User', ['email'])
```

### frappe.db.describe(doctype)
Returns table schema description as tuple.

### frappe.db.rename_table(old_name, new_name)
Renames database table. Use `frappe.rename_doc` for DocType renames instead.

### frappe.db.change_column_type(doctype, column, new_type)
Alters column data type.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-database/references/methods-reference.md*
