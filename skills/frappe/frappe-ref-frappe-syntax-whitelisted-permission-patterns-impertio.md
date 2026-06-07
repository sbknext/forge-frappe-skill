---
title: Frappe Ref — Frappe Syntax: Whitelisted: Permission Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'permission-patterns']
source: Impertio
---

# Permission Patterns Reference

Security best practices for whitelisted methods. ALWAYS check permissions inside every whitelisted method.

## Why Permissions Are Required

`@frappe.whitelist()` only verifies the user is **logged in** (or allows guests). It does NOT check:
- Whether the user can access a specific DocType
- Whether the user can access a specific document
- Whether the user has the required role

ALWAYS add explicit permission checks inside the method body.

---

## frappe.has_permission()

### Signature

```python
frappe.has_permission(
    doctype,           # DocType name (required)
    ptype="read",      # Permission type: read, write, create, submit, cancel, delete
    doc=None,          # Specific document name or Document object (optional)
    user=None,         # Specific user (default: current session user)
    throw=False        # True = auto-throw PermissionError instead of returning False
)
```

### DocType-Level Check

```python
@frappe.whitelist()
def get_all_orders():
    frappe.has_permission("Sales Order", "read", throw=True)
    return frappe.get_all("Sales Order", limit=20)
```

### Document-Level Check

```python
@frappe.whitelist()
def get_order(name):
    frappe.has_permission("Sales Order", "read", name, throw=True)
    return frappe.get_doc("Sales Order", name).as_dict()
```

### Manual Check with Custom Message

```python
@frappe.whitelist()
def update_order(name, status):
    if not frappe.has_permission("Sales Order", "write", name):
        frappe.throw(
            _("You cannot modify order {0}").format(name),
            frappe.PermissionError
        )
    frappe.db.set_value("Sales Order", name, "status", status)
```

### Permission Types

| Type | Description | Typical Endpoint |
|------|-------------|-----------------|
| `read` | View document | GET endpoints |
| `write` | Modify document | POST/PUT for updates |
| `create` | Create new document | POST for creation |
| `delete` | Delete document | DELETE endpoints |
| `submit` | Submit document | Submit workflows |
| `cancel` | Cancel document | Cancel workflows |

---

## frappe.only_for()

Restricts function to specific roles. Throws `PermissionError` if the user lacks ALL specified roles.

### Single Role

```python
@frappe.whitelist()
def admin_function():
    frappe.only_for("System Manager")
    return {"admin_data": "sensitive"}
```

### Multiple Roles (User Needs ANY One)

```python
@frappe.whitelist()
def hr_or_admin():
    frappe.only_for(["System Manager", "HR Manager"])
    return {"data": "restricted"}
```

### Combined with Document Permission

```python
@frappe.whitelist()
def restricted_update(doctype, name, values):
    frappe.only_for("System Manager")  # Role gate
    frappe.has_permission(doctype, "write", name, throw=True)  # Document gate

    doc = frappe.get_doc(doctype, name)
    doc.update(frappe.parse_json(values) if isinstance(values, str) else values)
    doc.save()
    return doc.as_dict()
```

---

## Custom Permission Logic

### Owner-or-Admin Pattern

```python
@frappe.whitelist()
def get_my_document(doctype, name):
    doc = frappe.get_doc(doctype, name)

    is_owner = doc.owner == frappe.session.user
    is_admin = "System Manager" in frappe.get_roles()

    if not (is_owner or is_admin):
        frappe.throw(_("Only the owner or admin can access this"), frappe.PermissionError)

    return doc.as_dict()
```

### Team-Based Access

```python
@frappe.whitelist()
def get_team_data(team):
    user = frappe.session.user
    team_members = frappe.get_all(
        "Team Member",
        filters={"parent": team},
        pluck="user"
    )
    if user not in team_members and "System Manager" not in frappe.get_roles():
        frappe.throw(_("Not a member of this team"), frappe.PermissionError)

    return frappe.get_doc("Team", team).as_dict()
```

---

## Guest Endpoint Permissions

For `allow_guest=True` endpoints, permission checks differ:

```python
@frappe.whitelist(allow_guest=True, methods=["GET"])
def get_public_items():
    """Public endpoint — ONLY return public-safe fields."""
    return frappe.get_all(
        "Item",
        filters={"show_on_website": 1},
        fields=["name", "item_name", "description", "standard_rate"]
        # NEVER include: cost_price, supplier, internal_notes
    )

@frappe.whitelist(allow_guest=True, methods=["POST"])
def submit_form(name, email, message):
    """Guest write — ALWAYS use ignore_permissions with fixed DocType."""
    doc = frappe.get_doc({
        "doctype": "Web Form Submission",  # Fixed — NEVER from user input
        "sender_name": frappe.utils.strip_html(name),
        "email": email,
        "message": frappe.utils.strip_html(message or "")
    })
    doc.insert(ignore_permissions=True)  # Acceptable: fixed DocType + guest context
    return {"success": True}
```

---

## ignore_permissions Usage

### NEVER Without a Preceding Check

```python
# WRONG — anyone can create anything
@frappe.whitelist()
def create_anything(data):
    doc = frappe.get_doc(data)
    doc.insert(ignore_permissions=True)  # SECURITY HOLE
```

### Acceptable Pattern

```python
@frappe.whitelist()
def system_log(message):
    frappe.only_for("System Manager")  # Role check FIRST
    doc = frappe.get_doc({
        "doctype": "Error Log",         # Fixed DocType
        "method": "API Log",
        "error": message
    })
    doc.insert(ignore_permissions=True)  # Acceptable with role check + fixed DocType
    return doc.name
```

### With DocType Whitelist

```python
@frappe.whitelist()
def create_allowed_doc(data):
    frappe.only_for("System Manager")

    ALLOWED_DOCTYPES = ["ToDo", "Note", "Communication"]
    if isinstance(data, str):
        data = frappe.parse_json(data)
    if data.get("doctype") not in ALLOWED_DOCTYPES:
        frappe.throw(_("Cannot create this document type"))

    doc = frappe.get_doc(data)
    doc.insert()  # Normal permissions — no ignore needed
    return doc.name
```

---

## Checklist for Every Whitelisted Method

1. **Determine access level**: public, authenticated, or role-restricted
2. **Choose check method**: `frappe.has_permission()`, `frappe.only_for()`, or custom
3. **Check BEFORE data access**: NEVER fetch data first, then check permission
4. **Limit response fields**: NEVER return `doc.as_dict()` with sensitive fields
5. **Log suspicious activity**: failed permission checks from unexpected users

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/permission-patterns.md*
