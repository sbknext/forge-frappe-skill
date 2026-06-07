---
name: frappe-permissions
description: >
  Implement Frappe's permission model correctly — roles, role permissions manager, permission
  levels (perm_level), user permissions, share, permission_query_conditions, has_permission hooks,
  and frappe.only_for / has_permission checks in code. Use when securing DocTypes, restricting rows
  per user, or field-level access. Trigger on: "permissions", "role", "user permission", "perm level",
  "restrict rows", "frappe.has_permission", "field level security", "permission query conditions".
---

# Frappe Permissions

Frappe authorizes at four layers: role → DocType perms → field perm-levels → row (user permissions /
share / query conditions). Use the lightest layer that solves the need.

## 1. Role + DocType permissions

Set in **Role Permissions Manager** (or the DocType's `permissions` in JSON). Each rule = role +
permlevel + flags (`read/write/create/delete/submit/cancel/amend/report/export/share`).

## 2. Field-level (perm_level)

```json
{ "fieldname": "internal_cost", "fieldtype": "Currency", "permlevel": 1 }
```
Only roles granted level-1 in the permission rules can read/write level-1 fields. Level 0 = everyone
with base access. Great for hiding sensitive columns from junior roles.

## 3. Row-level — User Permissions

Restrict a user to records linked to specific masters (e.g. only their Company/Territory):
`User Permission`: User = `clerk@x`, Allow = `Company`, For Value = `ACME`. Frappe auto-filters any
DocType with a `Company` link.

## 4. Row-level — dynamic query conditions (hooks.py)

```python
permission_query_conditions = {"Sales Order": "myapp.perms.so_conditions"}
has_permission             = {"Sales Order": "myapp.perms.so_has_permission"}
```
```python
def so_conditions(user):
    user = user or frappe.session.user
    if "Sales Manager" in frappe.get_roles(user):
        return ""                                  # see all
    return f"`tabSales Order`.owner = {frappe.db.escape(user)}"

def so_has_permission(doc, ptype, user=None):
    user = user or frappe.session.user
    if ptype == "read" and doc.event_type == "Public":
        return True
    if ptype == "write" and doc.owner == user:
        return True
    return False  # None falls back to default DocType perms
```
`so_conditions` filters lists/reports; `so_has_permission` guards single-doc access — implement both. The hook receives `permission_type` (`read`, `write`, `submit`, etc.).

## 5. In code

```python
frappe.only_for(["System Manager", "Sales Manager"])         # raises if none held
frappe.has_permission("Sales Order", "write", doc, throw=True)
if frappe.has_permission("Sales Order", "submit", doc):
    ...
roles = frappe.get_roles(frappe.session.user)
```

## Automatic roles (v15+)

| Role | Scope |
|---|---|
| Guest | Everyone, including unauthenticated |
| All | All registered users (incl. website users) |
| Administrator | Only the default Administrator user |
| Desk User | System Users only (not website users) — new in v15 |

## 6. Share (ad-hoc)

```python
frappe.share.add("Sales Order", "SO-0001", "user@x", write=1, notify=1)
```

## Rules

- **Whitelisting an API ≠ authorizing it** — call `frappe.has_permission(..., throw=True)` inside.
- `frappe.get_all` bypasses user perms (admin-ish); `frappe.get_list` enforces them. Choose consciously.
- Don't gate purely in client JS — always enforce server-side; JS is a UX hint only.
- Prefer User Permissions for "scope by master" needs before writing query-condition code.
- Test permissions as the target user: `frappe.set_user("clerk@x")` in tests, then assert.

## From Frappe docs

- `permission_query_conditions` and `has_permission` hooks: https://docs.frappe.io/framework/user/en/python-api/hooks
- `doc.check_permission(permtype)` on Document instances: https://docs.frappe.io/framework/user/en/api/document
