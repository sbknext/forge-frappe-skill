---
title: Frappe Ref — Frappe Errors: Permissions: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'permissions', 'reference', 'examples']
source: Impertio
---

# Examples — Permission Error Handling

Complete working examples for Frappe permission error handling.

---

## Example 1: Complete hooks.py Permission Setup

```python
# myapp/hooks.py
app_name = "myapp"
app_title = "My App"

has_permission = {
    "Sales Order": "myapp.permissions.sales_order_has_permission",
    "Confidential Report": "myapp.permissions.confidential_has_permission",
}

permission_query_conditions = {
    "Sales Order": "myapp.permissions.sales_order_query",
    "Confidential Report": "myapp.permissions.confidential_query",
}
```

```python
# myapp/permissions.py
import frappe
from frappe import _


# ── Sales Order Permissions ──────────────────────────────────────────

def sales_order_has_permission(doc, user, permission_type):
    """
    Rules:
    - Cancelled orders: read-only except System Manager
    - Orders > 100k: need Sales Manager for submit
    - Territory restrictions apply via User Permissions
    """
    try:
        user = user or frappe.session.user
        if user == "Administrator":
            return None

        roles = frappe.get_roles(user)
        if "System Manager" in roles:
            return None

        docstatus = doc.get("docstatus") if hasattr(doc, "get") else getattr(doc, "docstatus", 0)
        grand_total = doc.get("grand_total") if hasattr(doc, "get") else getattr(doc, "grand_total", 0)

        # Cancelled = read-only
        if docstatus == 2 and permission_type != "read":
            return False

        # Large orders need manager approval
        if permission_type == "submit" and (grand_total or 0) > 100000:
            if "Sales Manager" not in roles:
                return False

        # Territory check
        territory = doc.get("territory") if hasattr(doc, "get") else getattr(doc, "territory", None)
        if territory and permission_type in ("read", "write"):
            if not _has_territory_access(user, territory):
                return False

        return None

    except Exception:
        frappe.log_error(frappe.get_traceback(),
            f"SO permission error: {getattr(doc, 'name', 'unknown')}")
        return None


def sales_order_query(user):
    """Sales Order list filter."""
    try:
        user = user or frappe.session.user
        if user == "Administrator":
            return ""

        roles = frappe.get_roles(user)
        if "System Manager" in roles or "Sales Manager" in roles:
            return ""

        territories = _get_user_territories(user)
        if territories:
            escaped = ", ".join([frappe.db.escape(t) for t in territories])
            return f"""
                (`tabSales Order`.territory IN ({escaped})
                OR `tabSales Order`.owner = {frappe.db.escape(user)})
            """

        return f"`tabSales Order`.owner = {frappe.db.escape(user)}"

    except Exception:
        frappe.log_error(frappe.get_traceback(), "SO query error")
        return f"`tabSales Order`.owner = {frappe.db.escape(frappe.session.user)}"


# ── Confidential Report Permissions ──────────────────────────────────

def confidential_has_permission(doc, user, permission_type):
    """Strict access — only owner + explicit access list."""
    try:
        user = user or frappe.session.user
        if user == "Administrator":
            return None

        doc_name = doc.get("name") if hasattr(doc, "get") else getattr(doc, "name", None)
        if not doc_name:
            return None

        owner = doc.get("owner") if hasattr(doc, "get") else getattr(doc, "owner", None)
        if user == owner:
            return None

        has_access = frappe.db.exists("Confidential Report Access",
            {"parent": doc_name, "user": user})
        if not has_access:
            _log_denied_access(doc_name, user, permission_type)
            return False

        # Check access level
        access = frappe.db.get_value("Confidential Report Access",
            {"parent": doc_name, "user": user},
            ["read_access", "write_access"], as_dict=True)

        if permission_type == "read" and not access.get("read_access"):
            return False
        if permission_type in ("write", "delete") and not access.get("write_access"):
            return False

        return None

    except Exception:
        frappe.log_error(frappe.get_traceback(),
            f"Confidential permission error: {getattr(doc, 'name', 'unknown')}")
        return False  # DENY on error for confidential docs


def confidential_query(user):
    """Show only accessible confidential reports."""
    try:
        user = user or frappe.session.user
        if user == "Administrator":
            return ""
        return f"""
            (`tabConfidential Report`.owner = {frappe.db.escape(user)}
            OR EXISTS (
                SELECT 1 FROM `tabConfidential Report Access`
                WHERE `tabConfidential Report Access`.parent = `tabConfidential Report`.name
                AND `tabConfidential Report Access`.user = {frappe.db.escape(user)}
                AND `tabConfidential Report Access`.read_access = 1
            ))
        """
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Confidential query error")
        return f"`tabConfidential Report`.owner = {frappe.db.escape(frappe.session.user)}"


# ── Helpers ──────────────────────────────────────────────────────────

def _has_territory_access(user, territory):
    """Check if user has territory access. Returns True if no restrictions."""
    try:
        has_match = frappe.db.exists("User Permission",
            {"user": user, "allow": "Territory", "for_value": territory})
        has_any = frappe.db.count("User Permission",
            {"user": user, "allow": "Territory"})
        return has_match or not has_any  # No restrictions = access all
    except Exception:
        return True


def _get_user_territories(user):
    """Get user's permitted territories."""
    try:
        return frappe.get_all("User Permission",
            filters={"user": user, "allow": "Territory"},
            pluck="for_value") or []
    except Exception:
        return []


def _log_denied_access(doc_name, user, ptype):
    """Log denied access. NEVER let this break the permission check."""
    try:
        frappe.log_error(
            f"Access denied: {user} attempted {ptype} on {doc_name}",
            "Permission Audit")
    except Exception:
        pass
```

---

## Example 2: Permission Debug Script

Run this in the Frappe console to diagnose permission issues:

```python
# bench --site mysite.local console

import frappe
from frappe.permissions import get_doc_permissions

user = "john@example.com"
doctype = "Sales Order"
docname = "SO-00001"

# 1. Check user roles
print("=== Roles ===")
print(frappe.get_roles(user))

# 2. Check User Permissions
print("\n=== User Permissions ===")
for up in frappe.get_all("User Permission",
    filters={"user": user},
    fields=["allow", "for_value", "applicable_for", "is_default"]):
    print(f"  {up.allow} = {up.for_value} (for: {up.applicable_for or 'all'})")

# 3. Check DocType permissions for user's roles
print(f"\n=== DocPerm rows for {doctype} ===")
for dp in frappe.get_all("DocPerm",
    filters={"parent": doctype},
    fields=["role", "permlevel", "read", "write", "create", "delete",
            "submit", "cancel", "if_owner"],
    order_by="permlevel asc, role asc"):
    if dp.role in frappe.get_roles(user):
        print(f"  L{dp.permlevel} {dp.role}: R={dp.read} W={dp.write} "
              f"C={dp.create} D={dp.delete} if_owner={dp.if_owner}")

# 4. Check effective permissions on specific document
print(f"\n=== Effective permissions on {docname} ===")
doc = frappe.get_doc(doctype, docname)
perms = get_doc_permissions(doc, user=user)
for k, v in perms.items():
    if v:
        print(f"  {k}: {v}")

# 5. Check sharing
print(f"\n=== Sharing on {docname} ===")
for share in frappe.get_all("DocShare",
    filters={"share_doctype": doctype, "share_name": docname, "user": user},
    fields=["read", "write", "share", "everyone"]):
    print(f"  read={share.read} write={share.write} share={share.share}")
```

---

## Example 3: Client-Side Permission Handling

```javascript
// myapp/public/js/sales_order.js
frappe.ui.form.on("Sales Order", {
    refresh: function(frm) {
        // Show buttons based on permissions
        if (frm.doc.docstatus === 0 && frappe.perm.has_perm("Sales Order", 0, "write")) {
            frm.add_custom_button(__("Special Action"), function() {
                perform_special_action(frm);
            });
        }

        // Approve button — managers only
        if (frm.doc.status === "Pending Approval" && frappe.user.has_role("Sales Manager")) {
            frm.add_custom_button(__("Approve"), function() {
                frappe.call({
                    method: "myapp.api.approve_order",
                    args: {order_name: frm.doc.name},
                    freeze: true,
                    callback: function(r) {
                        if (r.message && r.message.status === "success") {
                            frappe.show_alert({message: __("Approved"), indicator: "green"});
                            frm.reload_doc();
                        }
                    },
                    error: function(r) {
                        if (r.exc_type === "PermissionError") {
                            frappe.msgprint({
                                title: __("Permission Denied"),
                                message: __("You lack permission to approve this order."),
                                indicator: "red"
                            });
                        }
                    }
                });
            }, __("Actions"));
        }

        // Toggle sensitive fields by role
        const is_manager = ["Sales Manager", "System Manager"].some(
            role => frappe.user.has_role(role));
        frm.toggle_display("margin", is_manager);
        frm.toggle_display("cost_center", is_manager);
    }
});
```

---

## Example 4: Controller with Permission Checks

```python
# myapp/doctype/confidential_document/confidential_document.py
import frappe
from frappe import _
from frappe.model.document import Document

class ConfidentialDocument(Document):
    def validate(self):
        if self.is_confidential and not self.is_new():
            old = self.get_doc_before_save()
            if old and not old.is_confidential:
                if not self._can_set_confidential():
                    frappe.throw(_("You lack permission to mark as confidential"),
                        exc=frappe.PermissionError)

    def _can_set_confidential(self):
        return any(r in frappe.get_roles()
            for r in ["System Manager", "Compliance Manager"])

    @frappe.whitelist()
    def grant_access(self, user):
        """Grant access — only owner or admin."""
        if self.owner != frappe.session.user:
            if "System Manager" not in frappe.get_roles():
                frappe.throw(_("Only owner can grant access"), exc=frappe.PermissionError)
        if not frappe.db.exists("User", user):
            frappe.throw(_("User {0} not found").format(user))
        if not frappe.db.exists("Confidential Document Access",
            {"parent": self.name, "user": user}):
            self.append("access_list", {"user": user})
            self.save()
        return {"status": "success"}
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-permissions/references/examples.md*
