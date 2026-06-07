---
title: Frappe Ref — Frappe Errors: Hooks: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'hooks', 'reference', 'examples']
source: Impertio
---

# Examples — Hooks Error Handling

Complete working examples of error handling in Frappe/ERPNext hooks.py configurations.

---

## Example 1: Complete hooks.py with Error-Safe Handlers

```python
# myapp/hooks.py

app_name = "myapp"
app_title = "My App"
app_publisher = "My Company"

# Asset includes — ALWAYS use assets/ prefix
app_include_js = "assets/myapp/js/myapp.js"
app_include_css = "assets/myapp/css/myapp.css"

# Document Events — nested dict with dotted paths
doc_events = {
    "*": {
        "on_update": "myapp.events.audit.log_change",
        "on_trash": "myapp.events.audit.log_delete"
    },
    "Sales Invoice": {
        "validate": "myapp.events.sales_invoice.validate",
        "on_submit": "myapp.events.sales_invoice.on_submit",
        "on_cancel": "myapp.events.sales_invoice.on_cancel"
    }
}

# Scheduler Events — list values, cron uses nested dict
scheduler_events = {
    "daily": [
        "myapp.tasks.daily_cleanup"
    ],
    "daily_long": [
        "myapp.tasks.sync_inventory"
    ],
    "cron": {
        "0 9 * * 1-5": ["myapp.tasks.weekday_morning_report"]
    }
}

# Permission Hooks — flat dict, doctype → dotted path
permission_query_conditions = {
    "Sales Invoice": "myapp.permissions.si_query_conditions"
}

has_permission = {
    "Sales Invoice": "myapp.permissions.si_has_permission"
}

# Boot Extension — single dotted path
extend_bootinfo = "myapp.boot.extend_boot"

# Override — one app per DocType (last writer wins)
override_doctype_class = {
    "Sales Invoice": "myapp.overrides.sales_invoice.CustomSalesInvoice"
}

# Extend [v16+] — multiple extensions, list of dotted paths
# extend_doctype_class = {
#     "Sales Invoice": ["myapp.extensions.si.SalesInvoiceMixin"]
# }

# Fixtures — use "dt" key, NOT "doctype"
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]},
    {"dt": "Property Setter", "filters": [["module", "=", "My App"]]}
]
```

---

## Example 2: Sales Invoice Event Handlers

```python
# myapp/events/sales_invoice.py
import frappe
from frappe import _

def validate(doc, method=None):
    """
    Validate handler — errors prevent save.
    Runs AFTER controller validate.
    """
    errors = []
    warnings = []

    # Custom field validation
    if doc.custom_requires_approval:
        if not doc.custom_approver:
            errors.append(_("Approver required when approval is enabled"))
        elif not frappe.db.exists("User", doc.custom_approver):
            errors.append(_("Approver '{0}' not found").format(doc.custom_approver))

    # External validation (wrapped — non-blocking)
    try:
        validate_with_external_system(doc)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"External validation: {doc.name}")
        warnings.append(_("External validation unavailable"))

    if warnings:
        frappe.msgprint("<br>".join(warnings), title=_("Warnings"), indicator="orange")

    if errors:
        frappe.throw("<br>".join(errors), title=_("Validation Error"))


def on_submit(doc, method=None):
    """Post-submit — document already submitted. Isolate operations."""
    # Critical operation
    try:
        create_custom_gl_entries(doc)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"GL entries: {doc.name}")
        frappe.throw(_("Accounting entries failed. Contact support."))

    # Non-critical — queue for reliability
    frappe.enqueue(
        "myapp.tasks.sync_invoice",
        invoice=doc.name,
        queue="short",
        job_id=f"sync_invoice_{doc.name}"
    )


def on_cancel(doc, method=None):
    """Cancel handler — reverse all operations, continue on partial failure."""
    cleanup_errors = []

    try:
        reverse_custom_gl_entries(doc)
    except Exception as e:
        cleanup_errors.append(f"GL reversal: {str(e)}")
        frappe.log_error(frappe.get_traceback(), f"GL reversal: {doc.name}")

    try:
        cancel_external_sync(doc)
    except Exception as e:
        cleanup_errors.append(f"Sync cancel: {str(e)}")
        frappe.log_error(frappe.get_traceback(), f"Sync cancel: {doc.name}")

    if cleanup_errors:
        frappe.msgprint(
            _("Cancelled with errors:<br>{0}").format("<br>".join(cleanup_errors)),
            indicator="orange"
        )


# Helper stubs
def validate_with_external_system(doc): pass
def create_custom_gl_entries(doc): pass
def reverse_custom_gl_entries(doc): pass
def cancel_external_sync(doc): pass
```

---

## Example 3: Scheduler Task with Full Error Tracking

```python
# myapp/tasks.py
import frappe
from frappe.utils import now_datetime, add_days, today

def daily_cleanup():
    """
    hooks.py: scheduler_events = {"daily": ["myapp.tasks.daily_cleanup"]}
    """
    results = {"deleted_logs": 0, "deleted_files": 0, "errors": []}

    # Task 1: Clean old error logs
    try:
        cutoff = add_days(today(), -30)
        count = frappe.db.count("Error Log", {"creation": ["<", cutoff]})
        if count > 0:
            frappe.db.delete("Error Log", {"creation": ["<", cutoff]})
            results["deleted_logs"] = count
        frappe.db.commit()
    except Exception as e:
        results["errors"].append(f"Error logs: {str(e)}")
        frappe.log_error(frappe.get_traceback(), "Cleanup: Error Log")
        frappe.db.rollback()

    # Task 2: Clean orphan temp files
    try:
        temp_files = frappe.get_all(
            "File",
            filters={
                "is_private": 1,
                "attached_to_doctype": "",
                "creation": ["<", add_days(today(), -7)]
            },
            limit=500
        )
        for f in temp_files:
            try:
                frappe.delete_doc("File", f.name, ignore_permissions=True)
                results["deleted_files"] += 1
            except Exception:
                pass  # Individual file errors are non-critical
        frappe.db.commit()
    except Exception as e:
        results["errors"].append(f"Temp files: {str(e)}")
        frappe.log_error(frappe.get_traceback(), "Cleanup: Temp Files")

    # Log summary if errors occurred
    if results["errors"]:
        frappe.log_error(frappe.as_json(results), "Daily Cleanup — With Errors")


def weekday_morning_report():
    """
    hooks.py: scheduler_events = {"cron": {"0 9 * * 1-5": ["myapp.tasks.weekday_morning_report"]}}
    """
    try:
        recipients = get_report_recipients()
        if not recipients:
            frappe.log_error("No recipients configured", "Morning Report")
            return

        report_data = compile_morning_report()
        for recipient in recipients:
            try:
                send_report_email(recipient, report_data)
            except Exception:
                frappe.log_error(frappe.get_traceback(), f"Report email: {recipient}")

        frappe.db.commit()
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Morning Report Fatal Error")


def get_report_recipients(): return []
def compile_morning_report(): return {}
def send_report_email(r, d): pass
```

---

## Example 4: Permission Hooks (Full Implementation)

```python
# myapp/permissions.py
import frappe

def si_query_conditions(user):
    """Sales Invoice list filter. NEVER throw."""
    try:
        if not user:
            user = frappe.session.user
        roles = frappe.get_roles(user)
        if "System Manager" in roles or "Accounts Manager" in roles:
            return ""
        if "Sales Manager" in roles:
            return get_team_condition(user, "Sales Invoice")
        return f"`tabSales Invoice`.owner = {frappe.db.escape(user)}"
    except Exception:
        frappe.log_error(frappe.get_traceback(), "SI Query Error")
        return f"`tabSales Invoice`.owner = {frappe.db.escape(frappe.session.user)}"


def si_has_permission(doc, user=None, permission_type=None):
    """Sales Invoice document-level permission. NEVER throw."""
    try:
        user = user or frappe.session.user
        roles = frappe.get_roles(user)
        if "System Manager" in roles:
            return None
        if doc.docstatus == 2 and permission_type != "read":
            return False
        if doc.grand_total and doc.grand_total > 100000:
            if permission_type == "submit" and "Invoice Approver" not in roles:
                return False
        return None
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"SI Permission: {getattr(doc, 'name', '?')}")
        return None


def get_team_condition(user, doctype):
    """Build team-based SQL condition. Returns owner filter on error."""
    try:
        dept = frappe.db.get_value("User", user, "department")
        if not dept:
            return f"`tab{doctype}`.owner = {frappe.db.escape(user)}"
        team = frappe.get_all("User", filters={"department": dept, "enabled": 1}, pluck="name") or [user]
        escaped = ", ".join([frappe.db.escape(u) for u in team])
        return f"`tab{doctype}`.owner IN ({escaped})"
    except Exception:
        return f"`tab{doctype}`.owner = {frappe.db.escape(user)}"
```

---

## Example 5: Boot Extension (Error-Safe)

```python
# myapp/boot.py
import frappe

def extend_boot(bootinfo):
    """NEVER let errors break page load."""
    bootinfo.myapp = {"settings": {}, "user_config": {}, "feature_flags": {}}

    try:
        if frappe.db.exists("My App Settings", "My App Settings"):
            s = frappe.get_single("My App Settings")
            bootinfo.myapp["settings"] = {
                "default_view": s.default_view or "list",
                "items_per_page": s.items_per_page or 20
            }
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Bootinfo: Settings")

    try:
        user = frappe.session.user
        if user != "Guest":
            bootinfo.myapp["user_config"] = {
                "can_approve": "Approver" in frappe.get_roles(user),
                "can_export": frappe.has_permission("Sales Invoice", "export", user=user)
            }
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Bootinfo: User Config")
```

---

## Example 6: Hook Registration Debugging

```python
# Run in bench console to diagnose hook issues:
# bench --site mysite console

import frappe

# Check if a specific hook is registered
print(frappe.get_hooks("doc_events"))

# Check scheduler tasks
print(frappe.get_hooks("scheduler_events"))

# Verify a dotted path resolves
try:
    module_path = "myapp.events.sales_invoice"
    func_name = "validate"
    module = frappe.get_module(module_path)
    func = getattr(module, func_name)
    print(f"Found: {func}")
except ImportError as e:
    print(f"Module not found: {e}")
except AttributeError:
    print(f"Function '{func_name}' not in module")

# Check installed apps
print(frappe.get_installed_apps())

# Check scheduler status
from frappe.utils.scheduler import is_scheduler_inactive
print(f"Scheduler inactive: {is_scheduler_inactive()}")
```

---

## Quick Reference: Hook Error Patterns

```python
# doc_events validate — collect and throw
def validate(doc, method=None):
    errors = []
    if not doc.field:
        errors.append(_("Field required"))
    if errors:
        frappe.throw("<br>".join(errors))

# doc_events on_update — isolate operations
def on_update(doc, method=None):
    try:
        non_critical_operation()
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Error")

# scheduler — always try/except and commit
def scheduled_task():
    try:
        do_work()
        frappe.db.commit()
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Task Error")

# permission_query_conditions — never throw
def query_conditions(user):
    try:
        return build_condition(user)
    except Exception:
        return f"owner = {frappe.db.escape(frappe.session.user)}"

# has_permission — never throw
def has_permission(doc, user=None, permission_type=None):
    try:
        return check_permission(doc, user)
    except Exception:
        return None

# extend_bootinfo — never throw
def extend_boot(bootinfo):
    bootinfo.data = {}
    try:
        bootinfo.data = load_data()
    except Exception:
        pass
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-hooks/references/examples.md*
