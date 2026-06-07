---
title: Frappe Ref — Frappe Errors: Hooks: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'hooks', 'reference', 'patterns']
source: Impertio
---

# Error Handling Patterns — Hooks

Complete error handling patterns for Frappe/ERPNext hooks.py configurations.

---

## Pattern 1: Hook Registration Diagnosis

```python
# Verify a hook is registered and callable
import frappe

def diagnose_hook(hook_type, doctype=None):
    """Diagnose why a hook is not firing."""
    hooks = frappe.get_hooks(hook_type)

    if not hooks:
        print(f"No hooks registered for '{hook_type}'")
        return

    if isinstance(hooks, dict) and doctype:
        hooks = hooks.get(doctype, [])
        if not hooks:
            print(f"No hooks for '{hook_type}' on DocType '{doctype}'")
            # Check for typo — list all registered DocTypes
            all_doctypes = list(frappe.get_hooks(hook_type).keys())
            print(f"Registered DocTypes: {all_doctypes}")
            return

    for hook_path in hooks:
        try:
            module_path, func_name = hook_path.rsplit(".", 1)
            module = frappe.get_module(module_path)
            func = getattr(module, func_name)
            print(f"OK: {hook_path} → {func}")
        except ImportError as e:
            print(f"IMPORT ERROR: {hook_path} → {e}")
        except AttributeError:
            print(f"FUNCTION NOT FOUND: {hook_path}")

# Usage:
# diagnose_hook("doc_events", "Sales Invoice")
# diagnose_hook("scheduler_events")
```

---

## Pattern 2: doc_events Multi-Operation Handler

```python
# myapp/events/sales_invoice.py
import frappe
from frappe import _

def on_submit(doc, method=None):
    """
    Post-submit handler with isolated operations.
    Document is already submitted — errors show message but do not roll back.
    """
    errors = []

    # Operation 1: Update linked quotation (critical)
    try:
        if doc.quotation:
            frappe.db.set_value("Quotation", doc.quotation, "status", "Ordered")
    except Exception as e:
        errors.append(f"Quotation update: {str(e)}")
        frappe.log_error(frappe.get_traceback(), "Quotation Update Error")

    # Operation 2: Send notification (non-critical)
    try:
        send_invoice_notification(doc)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Notification failed: {doc.name}")

    # Operation 3: External sync (non-critical, queue for reliability)
    try:
        frappe.enqueue(
            "myapp.tasks.sync_invoice",
            invoice=doc.name,
            queue="short",
            job_id=f"sync_{doc.name}"
        )
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Enqueue failed: {doc.name}")

    if errors:
        frappe.msgprint(
            _("Submitted with errors:<br>{0}").format("<br>".join(errors)),
            title=_("Warning"),
            indicator="orange"
        )
```

---

## Pattern 3: Scheduler Task with Batch Commits

```python
# myapp/tasks.py
import frappe
from frappe.utils import now_datetime

def sync_inventory():
    """
    hooks.py: scheduler_events = {"daily_long": ["myapp.tasks.sync_inventory"]}

    ALWAYS: try/except, log errors, commit explicitly, limit queries.
    """
    results = {"processed": 0, "failed": 0, "errors": []}

    try:
        items = frappe.get_all(
            "Item",
            filters={"sync_enabled": 1},
            fields=["name", "item_code"],
            limit=1000  # ALWAYS limit
        )

        BATCH_SIZE = 100
        for i in range(0, len(items), BATCH_SIZE):
            batch = items[i:i + BATCH_SIZE]
            for item in batch:
                try:
                    sync_single_item(item)
                    results["processed"] += 1
                except frappe.ValidationError as e:
                    results["failed"] += 1
                    results["errors"].append({"item": item.name, "error": str(e)[:200]})
                except Exception:
                    results["failed"] += 1
                    frappe.log_error(frappe.get_traceback(), f"Sync error: {item.name}")

            frappe.db.commit()  # Commit after each batch

    except Exception:
        frappe.log_error(frappe.get_traceback(), "Inventory Sync Fatal Error")
        return

    if results["errors"]:
        frappe.log_error(
            frappe.as_json(results),
            f"Sync Summary: {results['processed']} ok, {results['failed']} failed"
        )
```

---

## Pattern 4: Permission Query with Safe Fallback

```python
# myapp/permissions.py
import frappe

def sales_invoice_query_conditions(user):
    """
    hooks.py: permission_query_conditions = {
        "Sales Invoice": "myapp.permissions.sales_invoice_query_conditions"
    }

    NEVER throw. ALWAYS return valid SQL or empty string.
    NOTE: Only affects frappe.db.get_list(), NOT get_all().
    """
    try:
        if not user:
            user = frappe.session.user

        roles = frappe.get_roles(user)

        if "System Manager" in roles:
            return ""  # No restrictions

        if "Accounts Manager" in roles:
            return "`tabSales Invoice`.docstatus < 2"

        if "Sales Manager" in roles:
            team_users = get_team_members(user)
            if team_users:
                escaped = ", ".join([frappe.db.escape(u) for u in team_users])
                return f"`tabSales Invoice`.owner IN ({escaped})"

        return f"`tabSales Invoice`.owner = {frappe.db.escape(user)}"

    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Query conditions error: {user}")
        return f"`tabSales Invoice`.owner = {frappe.db.escape(frappe.session.user)}"


def get_team_members(user):
    """Get team members — returns empty list on error."""
    try:
        dept = frappe.db.get_value("User", user, "department")
        if not dept:
            return [user]
        return frappe.get_all("User", filters={"department": dept, "enabled": 1}, pluck="name") or [user]
    except Exception:
        return [user]
```

---

## Pattern 5: has_permission with Graceful Degradation

```python
def project_has_permission(doc, user=None, permission_type=None):
    """
    hooks.py: has_permission = {"Project": "myapp.permissions.project_has_permission"}

    NEVER throw. Return False to deny, None to defer to default.
    """
    try:
        user = user or frappe.session.user

        if "System Manager" in frappe.get_roles(user):
            return None  # Defer to default (allow)

        if doc.status == "Archived" and permission_type in ["write", "delete"]:
            return False

        if doc.get("is_confidential"):
            members = frappe.get_all("Project User", filters={"parent": doc.name}, pluck="user") or []
            if user not in members and doc.owner != user:
                return False

        return None  # Defer to default

    except Exception:
        frappe.log_error(
            frappe.get_traceback(),
            f"Permission error: {doc.name if hasattr(doc, 'name') else 'unknown'}"
        )
        return None
```

---

## Pattern 6: Override Class with Parent Error Handling

```python
# myapp/overrides/sales_invoice.py
from erpnext.accounts.doctype.sales_invoice.sales_invoice import SalesInvoice
import frappe
from frappe import _

class CustomSalesInvoice(SalesInvoice):
    """
    hooks.py: override_doctype_class = {
        "Sales Invoice": "myapp.overrides.sales_invoice.CustomSalesInvoice"
    }
    """

    def validate(self):
        # ALWAYS call parent — re-raise ValidationError
        try:
            super().validate()
        except frappe.ValidationError:
            raise
        except Exception:
            frappe.log_error(frappe.get_traceback(), f"Parent validate: {self.name}")
            raise

        self.validate_custom_fields()

    def validate_custom_fields(self):
        if self.custom_requires_po and not self.po_no:
            frappe.throw(_("PO Number required for this customer"))

    def on_submit(self):
        try:
            super().on_submit()
        except Exception:
            raise  # Parent on_submit errors are ALWAYS critical

        # Non-critical custom logic
        try:
            self.create_custom_entries()
        except Exception:
            frappe.log_error(frappe.get_traceback(), f"Custom entries: {self.name}")
            frappe.msgprint(_("Custom entries will be created later."), indicator="orange")
```

---

## Pattern 7: extend_bootinfo with Safe Loading

```python
# myapp/boot.py
import frappe

def extend_boot(bootinfo):
    """
    hooks.py: extend_bootinfo = "myapp.boot.extend_boot"

    NEVER let errors break login. ALWAYS wrap in try/except.
    """
    bootinfo.myapp = {"settings": {}, "permissions": {}}

    try:
        if frappe.db.exists("My App Settings", "My App Settings"):
            settings = frappe.get_single("My App Settings")
            bootinfo.myapp["settings"] = {
                "feature_enabled": settings.feature_enabled or False,
                "max_items": settings.max_items or 100
            }
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Bootinfo: settings load error")

    try:
        user = frappe.session.user
        if user and user != "Guest":
            bootinfo.myapp["permissions"] = {
                "can_approve": "Approver" in frappe.get_roles(user)
            }
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Bootinfo: permissions load error")
```

---

## Pattern 8: Wildcard doc_events (NEVER Break Saves)

```python
def log_all_changes(doc, method=None):
    """
    hooks.py: doc_events = {"*": {"on_update": "myapp.audit.log_all_changes"}}

    CRITICAL: Errors here break ALL document saves system-wide.
    """
    skip_types = ["Error Log", "Activity Log", "Communication", "Version", "Audit Log"]
    if doc.doctype in skip_types:
        return

    try:
        frappe.get_doc({
            "doctype": "Audit Log",
            "reference_doctype": doc.doctype,
            "reference_name": doc.name,
            "action": method,
            "user": frappe.session.user
        }).insert(ignore_permissions=True)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Audit: {doc.doctype}/{doc.name}")
```

---

## Pattern 9: extend_doctype_class [v16+] Safe Extension

```python
# myapp/extensions/sales_invoice.py
import frappe
from frappe import _

class SalesInvoiceMixin:
    """
    hooks.py: extend_doctype_class = {
        "Sales Invoice": ["myapp.extensions.sales_invoice.SalesInvoiceMixin"]
    }

    ALWAYS prefix methods with app name to avoid conflicts.
    """

    def myapp_check_approval(self):
        """Prefixed method — no conflict with other extensions."""
        try:
            if not self.custom_approver:
                frappe.throw(_("Approver not set"))
            if not frappe.db.exists("User", self.custom_approver):
                frappe.throw(_("Approver not found"))
        except frappe.DoesNotExistError:
            frappe.throw(_("Approver user does not exist"))
```

---

## Quick Reference: Hook Error Handling

| Hook | Error Strategy | Fallback |
|------|---------------|----------|
| doc_events (validate) | Collect, throw once | N/A |
| doc_events (on_update+) | Isolate, log non-critical | Continue |
| scheduler_events | Try/except all, commit batches | Log summary |
| permission_query_conditions | Never throw | Return owner filter |
| has_permission | Never throw | Return None |
| extend_bootinfo | Never throw | Return defaults |
| override class | super() in try/except | Re-raise |
| extend class [v16+] | Prefix methods | Avoid conflicts |
| wildcard (*) events | Never break saves | Log and continue |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-hooks/references/patterns.md*
