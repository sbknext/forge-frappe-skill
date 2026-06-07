---
title: Frappe Ref — Frappe Core: Permissions: Permission Api Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'permissions', 'reference', 'permission-api-reference']
source: Impertio
---

# Permission API Reference

> Complete API reference for all Frappe permission functions.

---

## Core Permission Checks

### frappe.has_permission()

```python
frappe.has_permission(
    doctype,                    # Required: DocType name
    ptype="read",              # Permission type (read/write/create/delete/submit/cancel/select)
    doc=None,                  # Document name (str) or Document object
    user=None,                 # User email (default: current session user)
    throw=False,               # If True, raises frappe.PermissionError on denial
    debug=False,               # If True, prints evaluation steps to console
    parent_doctype=None        # For child table permission checks
)
# Returns: bool
```

**Examples:**

```python
# DocType-level check
frappe.has_permission("Sales Order", "create")

# Document-level check (by name or object)
frappe.has_permission("Sales Order", "write", "SO-00001")
frappe.has_permission("Sales Order", "write", doc=doc_object)

# For specific user
frappe.has_permission("Sales Order", "read", user="john@example.com")

# Throw on denial
frappe.has_permission("Sales Order", "delete", throw=True)

# Debug mode — prints evaluation log
frappe.has_permission("Sales Order", "read", debug=True)
print(frappe.local.permission_debug_log)
```

---

### Document.has_permission()

```python
doc.has_permission(
    permtype="read",    # Permission type
    debug=False,        # Print debug logs
    user=None           # User (default: current session user)
)
# Returns: bool
```

**Note**: Respects `doc.flags.ignore_permissions` flag.

```python
doc = frappe.get_doc("Sales Order", "SO-00001")
if doc.has_permission("write"):
    doc.status = "Draft"
    doc.save()
```

---

### Document.check_permission()

Raises `frappe.PermissionError` if no permission.

```python
doc.check_permission(
    permtype="read",    # Permission type
    permlevel=None      # Optional: specific perm level
)
# Returns: None (raises frappe.PermissionError if denied)
```

```python
doc = frappe.get_doc("Sales Order", "SO-00001")
doc.check_permission("write")  # Raises error if denied
doc.status = "Closed"
doc.save()
```

---

## Permission Query Functions

### get_doc_permissions()

Get all effective permissions for a document.

```python
from frappe.permissions import get_doc_permissions

perms = get_doc_permissions(doc, user=None, ptype=None)
# Returns: dict {"read": 1, "write": 1, "create": 0, "delete": 0, ...}
```

### get_role_permissions()

Get role-based permissions (without user permissions applied).

```python
from frappe.permissions import get_role_permissions

meta = frappe.get_meta("Sales Order")
perms = get_role_permissions(meta, user=None)
# Returns: dict {"read": 1, "write": 0, ...}
```

---

## User Permission Functions

### add_user_permission()

```python
from frappe.permissions import add_user_permission

add_user_permission(
    doctype,              # DocType to restrict (e.g., "Company")
    name,                 # Value to allow (e.g., "My Company")
    user,                 # User email
    ignore_permissions=False,
    applicable_for=None,  # Apply only to specific DocType (optional)
    is_default=0,         # Use as default value in new documents
    hide_descendants=0    # For tree DocTypes — hide child nodes
)
```

### remove_user_permission()

```python
from frappe.permissions import remove_user_permission

remove_user_permission(doctype, name, user)
```

### get_user_permissions()

```python
from frappe.permissions import get_user_permissions

perms = get_user_permissions(user=None)
# Returns: dict by doctype
# {"Company": [{"doc": "My Company", "is_default": 1}], ...}
```

### clear_user_permissions_for_doctype()

```python
from frappe.permissions import clear_user_permissions_for_doctype

clear_user_permissions_for_doctype(doctype, user=None)
# Clears all user permissions for the doctype (optionally for specific user)
```

---

## Role Permission Management

### add_permission()

```python
from frappe.permissions import add_permission

add_permission(doctype, role, permlevel=0)
```

### update_permission_property()

```python
from frappe.permissions import update_permission_property

update_permission_property(doctype, role, permlevel, ptype, value)
# Example:
update_permission_property("Sales Order", "Sales User", 0, "write", 1)
update_permission_property("Sales Order", "Sales User", 0, "if_owner", 1)
```

### remove_permission()

```python
from frappe.permissions import remove_permission

remove_permission(doctype, role, permlevel=0)
```

### reset_perms()

Reset permissions to DocType defaults (removes customizations).

```python
from frappe.permissions import reset_perms

reset_perms(doctype)
```

---

## Sharing Functions

### frappe.share.add()

```python
from frappe.share import add as add_share

add_share(
    doctype,
    name,
    user=None,       # User email (required unless everyone=1)
    read=1,
    write=0,
    share=0,         # Allow re-sharing
    everyone=0,      # Share with all users
    notify=0         # Send email notification
)
```

### frappe.share.remove()

```python
from frappe.share import remove as remove_share

remove_share(doctype, name, user)
```

### frappe.share.get_shared()

```python
from frappe.share import get_shared

users = get_shared(doctype, name)
# Returns: list of user emails with shared access
```

---

## Utility Functions

### frappe.get_roles()

```python
roles = frappe.get_roles(user=None)
# Returns: list of role names
# ['Guest', 'All', 'Sales User', 'System Manager']
```

### frappe.only_for()

Restrict function execution to specific roles. Raises `frappe.PermissionError` if user lacks all listed roles.

```python
@frappe.whitelist()
def sensitive_action():
    frappe.only_for(["Sales Manager", "System Manager"])
    # Only reaches here if user has at least one of these roles
```

---

## Bypass Permissions

### ignore_permissions Flag

```python
# On document flags
doc.flags.ignore_permissions = True
doc.save()

# On save/insert method
doc.save(ignore_permissions=True)
doc.insert(ignore_permissions=True)
```

### Set User Temporarily

```python
original_user = frappe.session.user
frappe.set_user("Administrator")
# ... privileged operations ...
frappe.set_user(original_user)
```

### System Flags

```python
frappe.flags.in_setup_wizard = True
frappe.flags.in_install = True
frappe.flags.in_migrate = True
```

**ALWAYS** document the reason when bypassing permissions. NEVER use these without clear justification.

---

## Cache Management

ALWAYS clear cache after modifying permissions programmatically:

```python
frappe.clear_cache()
# Or for specific doctype:
frappe.clear_cache(doctype="Sales Order")
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-permissions/references/permission-api-reference.md*
