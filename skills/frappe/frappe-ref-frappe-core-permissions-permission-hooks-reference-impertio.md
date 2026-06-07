---
title: Frappe Ref — Frappe Core: Permissions: Permission Hooks Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'permissions', 'reference', 'permission-hooks-reference']
source: Impertio
---

# Permission Hooks Reference

> Complete reference for has_permission and permission_query_conditions hooks.

---

## has_permission Hook

### Purpose

Add custom permission logic for a DocType. Can only **deny** permission, NEVER grant it.

### Configuration

```python
# hooks.py
has_permission = {
    "Sales Order": "myapp.permissions.sales_order_permission",
    "Customer": "myapp.permissions.customer_permission"
}
```

### Function Signature

```python
def my_permission_check(doc, ptype, user):
    """
    Args:
        doc: Document object being checked
        ptype: Permission type (read, write, create, delete, submit, cancel)
        user: User email being checked

    Returns:
        None: No effect — continue standard permission checks
        False: DENY permission
        True: No effect in most versions (NEVER rely on this to grant)
    """
    pass
```

### Examples

#### Deny Editing Cancelled Documents

```python
def sales_order_permission(doc, ptype, user):
    if doc.docstatus == 2 and ptype == "write":
        if "Sales Manager" not in frappe.get_roles(user):
            return False
    return None
```

#### Time-Based Access Control

```python
def time_based_permission(doc, ptype, user):
    if ptype == "write":
        hour = frappe.utils.now_datetime().hour
        if hour < 9 or hour > 18:
            if "System Manager" not in frappe.get_roles(user):
                return False
    return None
```

#### Hierarchical Approval

```python
def approval_permission(doc, ptype, user):
    if ptype == "write" and doc.status == "Pending Approval":
        if doc.grand_total > 100000 and user != doc.department_head:
            return False
    return None
```

### Critical Rules

1. **ALWAYS** return `None` by default — to continue standard checks
2. **NEVER** return `True` to grant — it has no effect in most versions
3. **NEVER** throw errors — return `False` to deny instead
4. **ALWAYS** check `ptype` — do NOT apply write logic to read checks
5. Keep hooks fast — they are called frequently during permission evaluation

---

## permission_query_conditions Hook

### Purpose

Add WHERE clause conditions to `frappe.get_list()` queries for row-level filtering.

### Configuration

```python
# hooks.py
permission_query_conditions = {
    "ToDo": "myapp.permissions.todo_query",
    "Customer": "myapp.permissions.customer_query"
}
```

### Function Signature

```python
def my_query_conditions(user):
    """
    Args:
        user: User email (can be None — ALWAYS default to session user)

    Returns:
        str: Valid SQL WHERE clause fragment
        "": Empty string for no restriction (NEVER return None)
    """
    pass
```

### Examples

#### Owner-Based Filter

```python
def todo_query(user):
    if not user:
        user = frappe.session.user
    return """`tabToDo`.owner = {user} OR `tabToDo`.assigned_by = {user}""".format(
        user=frappe.db.escape(user)
    )
```

#### Role-Based Filter

```python
def customer_query(user):
    if not user:
        user = frappe.session.user
    if "Sales Manager" in frappe.get_roles(user):
        return ""
    return "`tabCustomer`.owner = {user}".format(user=frappe.db.escape(user))
```

#### Territory-Based Filter

```python
def customer_territory_query(user):
    if not user:
        user = frappe.session.user
    if "Sales Manager" in frappe.get_roles(user):
        return ""

    territories = frappe.get_all(
        "User Permission",
        filters={"user": user, "allow": "Territory"},
        pluck="for_value"
    )
    if not territories:
        return ""

    territory_list = ", ".join([frappe.db.escape(t) for t in territories])
    return f"`tabCustomer`.territory IN ({territory_list})"
```

#### Subquery Filter

```python
def project_query(user):
    if not user:
        user = frappe.session.user
    if "Projects Manager" in frappe.get_roles(user):
        return ""
    return """EXISTS (
        SELECT 1 FROM `tabProject User`
        WHERE `tabProject User`.parent = `tabProject`.name
        AND `tabProject User`.user = {user}
    )""".format(user=frappe.db.escape(user))
```

### Critical Rules

1. **ALWAYS** escape user input — `frappe.db.escape(user)`
2. **ALWAYS** use backtick table prefixes — `` `tabDocType`.fieldname ``
3. **ALWAYS** handle `None` user — default to `frappe.session.user`
4. **ALWAYS** return empty string for no restriction — NEVER return `None`
5. Only affects `get_list()` — does NOT affect `get_all()`

---

## get_list vs get_all Behavior

| Method | User Permissions | permission_query_conditions |
|--------|------------------|----------------------------|
| `frappe.get_list()` | Applied | Applied |
| `frappe.get_all()` | Ignored | Ignored |
| `frappe.db.get_list()` | Applied | Applied |
| `frappe.db.get_all()` | Ignored | Ignored |

---

## Combining Both Hooks

ALWAYS implement both hooks together for consistent behavior:

```python
# hooks.py
has_permission = {
    "Project": "myapp.permissions.project_permission"
}
permission_query_conditions = {
    "Project": "myapp.permissions.project_query"
}
```

`has_permission` controls single-document access. `permission_query_conditions` controls list-view filtering. If they apply different logic, users see inconsistent results.

---

## Hook Registration Order

1. Hooks from all installed apps are collected
2. Order follows app installation order in `apps.txt`
3. For `has_permission`: ALL hooks must pass (any `False` = denied)
4. For `permission_query_conditions`: conditions are AND-ed together

---

## Common Mistakes

### SQL Injection

```python
# WRONG — vulnerable
def bad_query(user):
    return f"owner = '{user}'"

# CORRECT — escaped
def good_query(user):
    return f"`tabCustomer`.owner = {frappe.db.escape(user)}"
```

### Missing Table Prefix

```python
# WRONG — ambiguous column in joins
def bad_query(user):
    return f"owner = {frappe.db.escape(user)}"

# CORRECT — explicit table
def good_query(user):
    return f"`tabSales Order`.owner = {frappe.db.escape(user)}"
```

### Throwing in Hook

```python
# WRONG — interrupts evaluation
def bad_permission(doc, ptype, user):
    frappe.throw("Not allowed!")

# CORRECT — return False to deny
def good_permission(doc, ptype, user):
    return False
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-permissions/references/permission-hooks-reference.md*
