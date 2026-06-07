---
title: Frappe Ref — Frappe Syntax: Hooks: Permissions (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'reference', 'permissions']
source: Impertio
---

# Permission Hooks Reference

Complete reference for permission hooks in hooks.py.

---

## permission_query_conditions

Dynamically filter list views based on user/role. Returns a SQL WHERE fragment.

### Syntax

```python
# hooks.py
permission_query_conditions = {
    "Sales Invoice": "myapp.permissions.si_query_conditions",
    "Project": "myapp.permissions.project_query_conditions"
}
```

### Handler Signature

```python
def si_query_conditions(user):
    """
    Args:
        user: str or None — ALWAYS check for None

    Returns:
        str: SQL WHERE fragment (without WHERE keyword)
             "" = no restrictions (show all)
             "1=0" = show nothing
    """
```

### Implementation Example

```python
import frappe

def si_query_conditions(user):
    if not user:
        user = frappe.session.user

    if user == "Administrator":
        return ""

    roles = frappe.get_roles(user)

    if "Sales Manager" in roles:
        return ""

    if "Sales User" in roles:
        return f"`tabSales Invoice`.owner = {frappe.db.escape(user)}"

    return "1=0"
```

### Subquery Example

```python
def project_query_conditions(user):
    if not user:
        user = frappe.session.user

    if "Projects Manager" in frappe.get_roles(user):
        return ""

    return f"""
        `tabProject`.name IN (
            SELECT parent FROM `tabProject User`
            WHERE user = {frappe.db.escape(user)}
        )
    """
```

### CRITICAL: get_list vs get_all

| Method | Applies permission_query_conditions | Behavior |
|--------|-------------------------------------|----------|
| `frappe.db.get_list` | YES | Respects permissions |
| `frappe.db.get_all` | NO | Ignores permissions entirely |

ALWAYS use `frappe.db.get_list` when permission filtering is required.

---

## has_permission

Custom document-level permission logic.

### Syntax

```python
# hooks.py
has_permission = {
    "Sales Invoice": "myapp.permissions.si_has_permission",
    "Event": "myapp.permissions.event_has_permission"
}
```

### Handler Signature

```python
def si_has_permission(doc, user=None, permission_type=None):
    """
    Args:
        doc: Document object
        user: str or None — ALWAYS check for None
        permission_type: "read", "write", "create", "delete",
                         "submit", "cancel", "amend", "print",
                         "email", "share"

    Returns:
        True: Grant access
        False: Deny access
        None: Fallback to default permission check
    """
```

### Implementation Example

```python
def si_has_permission(doc, user=None, permission_type=None):
    if not user:
        user = frappe.session.user

    # Closed invoices CANNOT be edited
    if permission_type == "write" and doc.status == "Closed":
        return False

    # Cancelled invoices CANNOT be deleted
    if permission_type == "delete" and doc.docstatus == 2:
        return False

    # Fallback to standard permissions
    return None


def event_has_permission(doc, user=None, permission_type=None):
    if not user:
        user = frappe.session.user

    if permission_type == "read" and doc.event_type == "Public":
        return True

    if doc.event_type == "Private" and doc.owner != user:
        return False

    return None
```

### All Permission Types

| Type | When Checked |
|------|-------------|
| `read` | Opening document, list view |
| `write` | Editing document |
| `create` | Creating new document |
| `delete` | Deleting document |
| `submit` | Submitting document |
| `cancel` | Cancelling document |
| `amend` | Amending document |
| `print` | Printing document |
| `email` | Emailing document |
| `share` | Sharing document |

---

## Evaluation Order

```
1. has_permission hook (document-level)
   |
2. Role Permissions (DocType level)
   |
3. User Permissions (field-level restrictions)
   |
4. permission_query_conditions (list filtering only)
```

---

## Combined Example

```python
# hooks.py
permission_query_conditions = {
    "Sales Invoice": "myapp.permissions.si_query"
}
has_permission = {
    "Sales Invoice": "myapp.permissions.si_permission"
}

# myapp/permissions.py
import frappe

def si_query(user):
    """List view filter — controls what appears in lists."""
    if not user:
        user = frappe.session.user

    if "Accounts Manager" in frappe.get_roles(user):
        return ""

    default_company = frappe.defaults.get_user_default("Company")
    if default_company:
        return f"`tabSales Invoice`.company = {frappe.db.escape(default_company)}"

    return "1=0"


def si_permission(doc, user=None, permission_type=None):
    """Document-level check — controls access to individual documents."""
    if not user:
        user = frappe.session.user

    if permission_type == "write" and doc.docstatus == 1:
        return False

    return None
```

---

## Critical Rules

1. ALWAYS check `if not user: user = frappe.session.user`
2. ALWAYS use `frappe.db.escape(user)` for SQL values — NEVER raw interpolation
3. ALWAYS return `None` (not `True`) to fallback to standard permissions
4. NEVER use `get_all` when you need permission filtering — use `get_list`
5. ALWAYS return `""` (empty string) for "no restrictions" in query conditions

---

## Debugging

```python
# Check if user has permission
has_perm = frappe.has_permission("Sales Invoice", "read", doc=invoice)

# Test query conditions directly
from myapp.permissions import si_query_conditions
print(si_query_conditions("user@example.com"))

# Test has_permission directly
doc = frappe.get_doc("Sales Invoice", "SI-00001")
from myapp.permissions import si_has_permission
print(si_has_permission(doc, "user@example.com", "write"))
```

---

## Version Differences

| Feature | v14 | v15 | v16 |
|---------|-----|-----|-----|
| permission_query_conditions | Yes | Yes | Yes |
| has_permission | Yes | Yes | Yes |
| Only works with get_list | Yes | Yes | Yes |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks/references/permissions.md*
