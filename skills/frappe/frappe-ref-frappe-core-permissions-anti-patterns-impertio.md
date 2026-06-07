---
title: Frappe Ref — Frappe Core: Permissions: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'permissions', 'reference', 'anti-patterns']
source: Impertio
---

# Permission Anti-Patterns

> Common permission mistakes and their correct alternatives.

---

## 1. Checking Role Instead of Permission

```python
# WRONG — bypasses User Permissions, hooks, and if_owner
if "Sales Manager" in frappe.get_roles():
    doc.save(ignore_permissions=True)

# CORRECT — uses full permission system
if not doc.has_permission("write"):
    frappe.throw(_("No permission"), frappe.PermissionError)
doc.save()
```

---

## 2. Using get_all for User-Facing Queries

```python
# WRONG — ignores User Permissions and query hooks
orders = frappe.get_all("Sales Order", filters={"status": "Open"})

# CORRECT — respects all permission layers
orders = frappe.get_list("Sales Order", filters={"status": "Open"})
```

---

## 3. SQL Injection in Query Hook

```python
# WRONG — vulnerable to injection
def bad_query(user):
    return f"owner = '{user}'"

# CORRECT — escaped
def good_query(user):
    if not user:
        user = frappe.session.user
    return f"`tabCustomer`.owner = {frappe.db.escape(user)}"
```

---

## 4. Missing Table Prefix in Query Hook

```python
# WRONG — ambiguous column in JOINs
def bad_query(user):
    return f"owner = {frappe.db.escape(user)}"

# CORRECT — explicit table prefix
def good_query(user):
    return f"`tabSales Order`.owner = {frappe.db.escape(user)}"
```

---

## 5. Throwing Errors in has_permission Hook

```python
# WRONG — interrupts permission evaluation, may expose info
def bad_permission(doc, ptype, user):
    frappe.throw("Not allowed!")

# CORRECT — return False to deny
def good_permission(doc, ptype, user):
    return False
```

---

## 6. Returning True in has_permission Hook

```python
# WRONG — True does NOT grant permission in most versions
def bad_permission(doc, ptype, user):
    if is_special_user(user):
        return True  # Has NO effect!
    return None

# CORRECT — hooks can only deny, not grant
def good_permission(doc, ptype, user):
    if should_deny(doc, user):
        return False
    return None  # Let standard checks decide
```

---

## 7. ignore_permissions Without Documentation

```python
# WRONG — no explanation why permissions are bypassed
doc.save(ignore_permissions=True)

# CORRECT — documented reason
# System action: auto-processed by scheduler job (runs as Administrator)
doc.add_comment("Info", "System: Auto-processed by scheduler")
doc.flags.ignore_permissions = True
doc.save()
```

---

## 8. Granting Perm Level 1+ Without Level 0

```json
// WRONG — causes error
{"role": "HR User", "permlevel": 1, "read": 1}

// CORRECT — level 0 first
[
  {"role": "HR User", "permlevel": 0, "read": 1},
  {"role": "HR User", "permlevel": 1, "read": 1, "write": 1}
]
```

---

## 9. Using db_set/set_value for User-Facing Updates

```python
# WRONG — bypasses permissions, validation, and hooks
frappe.db.set_value("DocType", doc_name, "field", "value")

# CORRECT — respects all checks
doc = frappe.get_doc("DocType", doc_name)
doc.check_permission("write")
doc.field = "value"
doc.save()
```

`frappe.db.set_value()` bypasses: permission checks, validate methods, before/after save hooks, and child table validation.

---

## 10. Not Clearing Cache After Permission Changes

```python
# WRONG — changes may not take effect
from frappe.permissions import add_permission
add_permission("Sales Order", "New Role")

# CORRECT — clear cache
add_permission("Sales Order", "New Role")
frappe.clear_cache()
```

---

## 11. Sensitive Data in Error Messages

```python
# WRONG — leaks information
frappe.throw(f"Cannot view {doc.employee_name}'s salary: {doc.salary}")

# CORRECT — generic error
frappe.throw(_("Permission denied"), frappe.PermissionError)
```

---

## 12. Hardcoding Administrator Bypass

```python
# WRONG — security risk
if frappe.session.user == "Administrator":
    perform_operation()

# CORRECT — use permission system
if not frappe.has_permission("Sensitive DocType", "write"):
    frappe.throw(_("Access Denied"), frappe.PermissionError)
perform_operation()
```

---

## 13. Assuming Data Masking in Custom SQL [v16+]

```python
# WRONG — masking does NOT apply to raw SQL
data = frappe.db.sql("SELECT phone FROM tabCustomer", as_dict=True)

# CORRECT — manual masking
data = frappe.db.sql("SELECT phone FROM tabCustomer", as_dict=True)
if not frappe.has_permission("Customer", "mask"):
    for row in data:
        if row.phone and len(row.phone) > 5:
            row.phone = row.phone[:3] + "X" * (len(row.phone) - 3)
```

---

## Pre-Deploy Checklist

- [ ] Using `has_permission()` instead of role checks?
- [ ] Using `get_list()` instead of `get_all()` for user queries?
- [ ] All SQL inputs escaped with `frappe.db.escape()`?
- [ ] Table prefixes used in query conditions?
- [ ] `ignore_permissions` usage documented with comments?
- [ ] Permission cache cleared after programmatic changes?
- [ ] Error messages do NOT leak sensitive data?
- [ ] Level 0 granted before higher perm levels?
- [ ] Hooks return `None` or `False`, NEVER throw errors?
- [ ] Tested with non-admin user account?

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-permissions/references/anti-patterns.md*
