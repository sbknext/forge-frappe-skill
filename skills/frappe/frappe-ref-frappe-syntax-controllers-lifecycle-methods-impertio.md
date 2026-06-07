---
title: Frappe Ref — Frappe Syntax: Controllers: Lifecycle Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'lifecycle-methods']
source: Impertio
---

# Lifecycle Methods Reference

Complete reference for all Document Controller lifecycle hooks in Frappe v14-v16.

---

## Complete Hook Table

| Hook | Runs During | When | Can Modify self? | Version |
|---|---|---|---|---|
| `before_insert` | insert | Before naming, before DB write | Yes | All |
| `before_naming` | insert | Before name generation | Yes (naming params) | All |
| `autoname` | insert | During name generation | Sets self.name | All |
| `before_validate` | insert, save, submit | Before validate() | Yes | All |
| `validate` | insert, save, submit | Main validation | Yes (saved to DB) | All |
| `before_save` | insert, save, submit | After validate, before DB write | Yes (saved to DB) | All |
| `after_insert` | insert | After DB insert, before on_update | Yes (NOT saved) | All |
| `on_update` | insert, save | After DB write | No (use db_set) | All |
| `on_change` | insert, save, submit, cancel, db_set | After any value change | No | All |
| `before_submit` | submit | Before docstatus changes to 1 | Yes (saved to DB) | All |
| `on_submit` | submit | After docstatus = 1 in DB | No (use db_set) | All |
| `before_cancel` | cancel | Before docstatus changes to 2 | Yes | All |
| `on_cancel` | cancel | After docstatus = 2 in DB | No (use db_set) | All |
| `before_update_after_submit` | update_after_submit | Before submitted doc update | Yes | All |
| `on_update_after_submit` | update_after_submit | After submitted doc update | No | All |
| `on_trash` | delete | Before DB delete | Yes | All |
| `after_delete` | delete | After DB delete | N/A | All |
| `before_rename` | rename | Before name change | Yes | All |
| `after_rename` | rename | After name change | No | All |
| `before_print` | print | Before print format renders | Yes | All |
| `before_discard` | discard | Before draft discard | Yes | v15+ |
| `on_discard` | discard | After draft discard | No | v15+ |

---

## Execution Order Diagrams

### INSERT (New Document)

```
doc.insert()
  |
  v
1. before_insert
   - Last chance to modify fields before naming
   - self.name is NOT yet available
  |
  v
2. before_naming
   - Modify naming_series or naming parameters
   - Runs BEFORE autoname
  |
  v
3. autoname
   - Generate self.name programmatically
   - Overrides DocType Auto Name setting
  |
  v
4. before_validate
   - self.name IS now available
   - Pre-validation setup
  |
  v
5. validate
   - MAIN validation and calculations
   - Use frappe.throw() to block save
   - Changes to self ARE saved
  |
  v
6. before_save
   - Final chance for modifications before DB write
   - After all validation has passed
  |
  v
7. [db_insert - INTERNAL]
   - Document written to database
   - No custom code possible here
  |
  v
8. after_insert
   - Document exists in DB with name
   - Changes to self are NOT saved
   - Runs ONLY for new documents (not updates)
  |
  v
9. on_update
   - After successful save
   - Changes to self are NOT saved
   - Use self.db_set() or frappe.db.set_value()
  |
  v
10. on_change
    - After any value change
    - MUST be idempotent (may run multiple times)
```

### SAVE (Existing Document)

```
doc.save()
  |
  v
1. before_validate
  |
  v
2. validate
   - Changes to self ARE saved
  |
  v
3. before_save
  |
  v
4. [db_update - INTERNAL]
  |
  v
5. on_update
   - Changes to self are NOT saved
  |
  v
6. on_change
```

### SUBMIT (docstatus 0 -> 1)

```
doc.submit()
  |
  v
1. before_validate
  |
  v
2. validate
  |
  v
3. before_submit
   - Last chance to block submit with frappe.throw()
   - Changes to self ARE saved
  |
  v
4. [db_update with docstatus=1 - INTERNAL]
  |
  v
5. on_submit
   - Create ledger entries, stock entries here
   - Changes to self are NOT saved
  |
  v
6. on_update
  |
  v
7. on_change
```

### CANCEL (docstatus 1 -> 2)

```
doc.cancel()
  |
  v
1. before_cancel
   - Check for linked submitted documents
   - Use frappe.throw() to block cancel
  |
  v
2. [db_update with docstatus=2 - INTERNAL]
  |
  v
3. on_cancel
   - ALWAYS reverse what on_submit created
   - Changes to self are NOT saved
  |
  v
4. on_change
```

### UPDATE AFTER SUBMIT

```
doc.save() [when docstatus == 1]
  |
  v
1. before_update_after_submit
   - Only fields with "Allow on Submit" can change
  |
  v
2. [db_update - INTERNAL]
  |
  v
3. on_update_after_submit
  |
  v
4. on_change
```

### DELETE

```
doc.delete() / frappe.delete_doc()
  |
  v
1. on_trash
   - Clean up related data
   - NEVER delete linked submitted documents here
  |
  v
2. [db_delete - INTERNAL]
  |
  v
3. after_delete
   - Document no longer exists in DB
```

### DISCARD [v15+]

```
doc.discard()
  |
  v
1. before_discard
   - Only for draft documents (docstatus=0)
  |
  v
2. [db_set docstatus=2 - INTERNAL]
  |
  v
3. on_discard
```

### RENAME

```
frappe.rename_doc()
  |
  v
1. before_rename
  |
  v
2. [rename in DB - INTERNAL]
  |
  v
3. after_rename
```

---

## Hook Method Signatures

```python
class MyDocType(Document):
    # Naming hooks
    def before_naming(self, *args, **kwargs): ...
    def autoname(self): ...

    # Insert hooks
    def before_insert(self): ...
    def after_insert(self): ...

    # Validation and save hooks
    def before_validate(self): ...
    def validate(self): ...
    def before_save(self): ...
    def on_update(self): ...
    def on_change(self): ...

    # Submit hooks
    def before_submit(self): ...
    def on_submit(self): ...

    # Cancel hooks
    def before_cancel(self): ...
    def on_cancel(self): ...

    # Update after submit hooks
    def before_update_after_submit(self): ...
    def on_update_after_submit(self): ...

    # Delete hooks
    def on_trash(self): ...
    def after_delete(self): ...

    # Rename hooks
    def before_rename(self, old_name, new_name, merge=False): ...
    def after_rename(self, old_name, new_name, merge=False): ...

    # Print hooks
    def before_print(self, print_settings=None): ...

    # Discard hooks [v15+]
    def before_discard(self): ...
    def on_discard(self): ...
```

---

## Key Rules

1. **validate** is the ONLY hook where changes to `self` are reliably saved to DB
2. **on_update**, **on_submit**, **on_cancel** run AFTER the DB write -- changes to self are NOT saved
3. **on_change** MUST be idempotent -- it runs after every value change including `db_set`
4. **after_insert** runs ONLY for new documents, NEVER for updates
5. **before_submit** can block submit with `frappe.throw()` -- use for approval checks
6. ALWAYS implement `on_cancel` as the reverse of `on_submit`
7. NEVER call `frappe.db.commit()` inside any hook

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/lifecycle-methods.md*
