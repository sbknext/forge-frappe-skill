---
title: Frappe Ref — Frappe Errors: Permissions: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'permissions', 'reference', 'patterns']
source: Impertio
---

# Permission Error Handling Patterns

Complete error handling patterns for Frappe permission system. For quick reference see SKILL.md.

---

## Pattern 1: has_permission Hook with Full Error Safety

```python
# myapp/permissions.py
import frappe

def sales_order_has_permission(doc, user, permission_type):
    """
    Document-level permission hook.

    Args:
        doc: Document object or dict (may lack methods like .get())
        user: User email or None (defaults to current user)
        permission_type: "read", "write", "create", "delete", "submit", "cancel"

    Returns:
        None:  Defer to standard permission system (ALWAYS default)
        False: Deny permission
        NEVER return True — hooks can only restrict, not grant.

    NEVER throw from this function. ALWAYS wrap in try/except.
    """
    try:
        user = user or frappe.session.user

        # ALWAYS let Administrator through first
        if user == "Administrator":
            return None

        roles = frappe.get_roles(user)

        # ALWAYS let System Manager through
        if "System Manager" in roles:
            return None

        # Safe attribute access — doc may be dict or object
        status = doc.get("status") if hasattr(doc, "get") else getattr(doc, "status", None)
        docstatus = doc.get("docstatus") if hasattr(doc, "get") else getattr(doc, "docstatus", 0)
        is_confidential = doc.get("is_confidential") if hasattr(doc, "get") else getattr(doc, "is_confidential", 0)

        # Rule 1: Cancelled documents are read-only
        if docstatus == 2 and permission_type != "read":
            return False

        # Rule 2: Locked documents block modifications
        if status == "Locked" and permission_type in ("write", "delete", "cancel"):
            if "Sales Manager" not in roles:
                return False

        # Rule 3: Only managers can delete
        if permission_type == "delete" and "Sales Manager" not in roles:
            return False

        # Rule 4: Confidential access list
        if is_confidential:
            allowed = _get_confidential_users(doc)
            if user not in allowed:
                return False

        # Rule 5: Territory restriction
        if permission_type == "read":
            if _check_territory_access(doc, user) is False:
                return False

        # ALWAYS return None as default — defer to standard system
        return None

    except Exception:
        frappe.log_error(
            frappe.get_traceback(),
            f"has_permission error: {getattr(doc, 'name', 'unknown')}"
        )
        return None  # SAFE FALLBACK — defer to standard system


def _get_confidential_users(doc):
    """Get users with confidential access. Returns empty list on error."""
    try:
        doc_name = doc.get("name") if hasattr(doc, "get") else getattr(doc, "name", None)
        if not doc_name:
            return []
        owner = doc.get("owner") if hasattr(doc, "get") else getattr(doc, "owner", None)
        allowed = [owner] if owner else []
        allowed.extend(
            frappe.get_all("Sales Order Access",
                filters={"parent": doc_name}, pluck="user") or []
        )
        return allowed
    except Exception:
        return []


def _check_territory_access(doc, user):
    """Check territory access. Returns None to defer, False to deny."""
    try:
        territory = doc.get("territory") if hasattr(doc, "get") else getattr(doc, "territory", None)
        if not territory:
            return None
        user_territories = frappe.get_all("User Permission",
            filters={"user": user, "allow": "Territory",
                      "applicable_for": ["in", ["", "Sales Order"]]},
            pluck="for_value")
        if not user_territories:
            return None  # No territory restrictions for this user
        if territory not in user_territories:
            return False
        return None
    except Exception:
        return None
```

---

## Pattern 2: permission_query_conditions with Team and Territory

```python
# myapp/permissions.py
import frappe

def sales_order_query(user):
    """
    Return SQL WHERE clause fragment for list filtering.

    ALWAYS return a string. Empty string = no restriction.
    ALWAYS use frappe.db.escape(). NEVER throw.
    """
    try:
        user = user or frappe.session.user

        if user == "Guest":
            return "1=0"  # Guest sees nothing

        if user == "Administrator":
            return ""

        roles = frappe.get_roles(user)

        if "System Manager" in roles:
            return ""

        conditions = []

        # Sales Manager: all non-confidential + own confidential
        if "Sales Manager" in roles:
            conditions.append(f"""
                (`tabSales Order`.is_confidential = 0
                OR `tabSales Order`.owner = {frappe.db.escape(user)}
                OR EXISTS (
                    SELECT 1 FROM `tabSales Order Access`
                    WHERE `tabSales Order Access`.parent = `tabSales Order`.name
                    AND `tabSales Order Access`.user = {frappe.db.escape(user)}
                ))
            """)

        # Sales User: own records + team records
        elif "Sales User" in roles:
            team_sql = _get_team_condition(user)
            if team_sql:
                conditions.append(f"""
                    (`tabSales Order`.owner = {frappe.db.escape(user)} OR {team_sql})
                    AND `tabSales Order`.is_confidential = 0
                """)
            else:
                conditions.append(f"""
                    `tabSales Order`.owner = {frappe.db.escape(user)}
                    AND `tabSales Order`.is_confidential = 0
                """)

        # Default: own non-confidential records
        else:
            conditions.append(f"""
                `tabSales Order`.owner = {frappe.db.escape(user)}
                AND `tabSales Order`.is_confidential = 0
            """)

        # Territory filter
        territory_sql = _get_territory_condition(user)
        if territory_sql:
            conditions.append(territory_sql)

        return " AND ".join([f"({c.strip()})" for c in conditions])

    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Query conditions error for {user}")
        return f"`tabSales Order`.owner = {frappe.db.escape(frappe.session.user)}"


def _get_team_condition(user):
    """Build SQL for team members. Returns None on error."""
    try:
        department = frappe.db.get_value("User", user, "department")
        if not department:
            return None
        team = frappe.get_all("User",
            filters={"department": department, "enabled": 1}, pluck="name")
        if not team or len(team) <= 1:
            return None
        escaped = ", ".join([frappe.db.escape(u) for u in team])
        return f"`tabSales Order`.owner IN ({escaped})"
    except Exception:
        return None


def _get_territory_condition(user):
    """Build SQL for territory filter. Returns None on error."""
    try:
        territories = frappe.get_all("User Permission",
            filters={"user": user, "allow": "Territory",
                      "applicable_for": ["in", ["", "Sales Order"]]},
            pluck="for_value")
        if not territories:
            return None
        escaped = ", ".join([frappe.db.escape(t) for t in territories])
        return f"(`tabSales Order`.territory IN ({escaped}) OR `tabSales Order`.territory IS NULL)"
    except Exception:
        return None
```

---

## Pattern 3: API Endpoint Permission Checks

```python
# myapp/api.py
import frappe
from frappe import _

@frappe.whitelist()
def get_order_details(order_name):
    """API endpoint with layered permission checks."""
    if not order_name:
        frappe.throw(_("Order name required"), exc=frappe.ValidationError)

    if not frappe.db.exists("Sales Order", order_name):
        frappe.throw(_("Sales Order {0} not found").format(order_name),
            exc=frappe.DoesNotExistError)

    # Throws PermissionError automatically if denied
    frappe.has_permission("Sales Order", "read", order_name, throw=True)

    doc = frappe.get_doc("Sales Order", order_name)
    result = {"name": doc.name, "customer": doc.customer, "status": doc.status}

    # Filter sensitive fields by role
    if "Sales Manager" in frappe.get_roles():
        result["margin"] = doc.get("margin")
        result["cost"] = doc.get("cost")

    return result


@frappe.whitelist()
def approve_order(order_name):
    """Role-restricted endpoint with audit logging."""
    frappe.only_for(["Sales Manager", "General Manager"])

    doc = frappe.get_doc("Sales Order", order_name)
    if not doc.has_permission("write"):
        frappe.throw(_("Cannot approve — no write permission"),
            exc=frappe.PermissionError)

    doc.status = "Approved"
    doc.approved_by = frappe.session.user
    doc.save()

    return {"status": "success"}


@frappe.whitelist()
def bulk_update(orders, status):
    """Bulk operation with per-document permission check."""
    orders = frappe.parse_json(orders) if isinstance(orders, str) else orders
    results = {"success": [], "failed": [], "permission_denied": []}

    for name in orders:
        if not frappe.db.exists("Sales Order", name):
            results["failed"].append({"name": name, "error": "Not found"})
            continue
        if not frappe.has_permission("Sales Order", "write", name):
            results["permission_denied"].append(name)
            continue
        try:
            frappe.db.set_value("Sales Order", name, "status", status)
            results["success"].append(name)
        except Exception as e:
            results["failed"].append({"name": name, "error": str(e)})

    frappe.db.commit()
    return results
```

---

## Pattern 4: Graceful Permission Degradation

```python
@frappe.whitelist()
def get_dashboard_data():
    """Return data based on user permissions — no errors, just filtered results."""
    data = {"widgets": [], "stats": {}}

    if frappe.has_permission("Sales Order", "read"):
        try:
            data["widgets"].append({"type": "sales", "data": get_sales_summary()})
        except Exception:
            frappe.log_error(frappe.get_traceback(), "Dashboard: Sales Widget")

    if frappe.has_permission("Purchase Order", "read"):
        try:
            data["widgets"].append({"type": "purchase", "data": get_purchase_summary()})
        except Exception:
            frappe.log_error(frappe.get_traceback(), "Dashboard: Purchase Widget")

    return data
```

---

## Pattern 5: Security Audit Logging

```python
def log_access_denied(doc, user, ptype, reason=""):
    """Log denied access for security audit. NEVER let this break the permission check."""
    try:
        frappe.get_doc({
            "doctype": "Activity Log",
            "subject": f"Access Denied: {ptype} on {getattr(doc, 'name', 'unknown')}",
            "content": f"User: {user}, Reason: {reason}",
            "reference_doctype": getattr(doc, "doctype", None),
            "reference_name": getattr(doc, "name", None),
        }).insert(ignore_permissions=True)
    except Exception:
        pass  # NEVER break permission check for logging failure
```

---

## Quick Reference

| Scenario | Method | Returns on Deny |
|----------|--------|----------------|
| has_permission hook | `return False` | Silent deny |
| permission_query_conditions | Return restrictive SQL | Filtered list |
| API endpoint | `frappe.has_permission(throw=True)` | HTTP 403 |
| Role restriction | `frappe.only_for(["Role"])` | HTTP 403 |
| Document method | `doc.check_permission("write")` | HTTP 403 |
| Custom throw | `frappe.throw(msg, exc=PermissionError)` | HTTP 403 |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-permissions/references/patterns.md*
