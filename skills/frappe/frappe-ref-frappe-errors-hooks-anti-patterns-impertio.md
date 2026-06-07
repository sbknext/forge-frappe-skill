---
title: Frappe Ref — Frappe Errors: Hooks: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'hooks', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns — Hooks Error Handling

Common mistakes to avoid when handling errors in Frappe/ERPNext hooks.py.

---

## 1. Typo in Hook Dotted Path

### Problem
```python
# hooks.py — function name doesn't match
doc_events = {
    "Sales Invoice": {
        "validate": "myapp.events.sales.validate_invoice"
    }
}
# But actual function is named validate_si()
```

### Fix
```python
# ALWAYS verify: module path + function name match the actual file
# myapp/events/sales.py must contain:
def validate_invoice(doc, method=None):
    pass
```

**Why**: Mismatched dotted paths cause silent failures or ImportError on bench restart.

---

## 2. Wrong Dict Structure for Hook Type

### Problem
```python
# doc_events needs nested dict — string value is WRONG
doc_events = {
    "Sales Invoice": "myapp.events.validate"
}

# scheduler_events daily needs list — string is WRONG
scheduler_events = {
    "daily": "myapp.tasks.daily_sync"
}
```

### Fix
```python
doc_events = {
    "Sales Invoice": {
        "validate": "myapp.events.sales.validate"
    }
}

scheduler_events = {
    "daily": ["myapp.tasks.daily_sync"]
}
```

**Why**: Framework expects specific data structures. Wrong structure causes silent failure.

---

## 3. Throwing in permission_query_conditions

### Problem
```python
def query_conditions(user):
    if not user:
        frappe.throw("User required")  # BREAKS LIST VIEW!
    return f"owner = '{user}'"  # Also: SQL injection!
```

### Fix
```python
def query_conditions(user):
    try:
        user = user or frappe.session.user
        if "System Manager" in frappe.get_roles(user):
            return ""
        return f"owner = {frappe.db.escape(user)}"
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Query Conditions Error")
        return f"owner = {frappe.db.escape(frappe.session.user)}"
```

**Why**: Throwing in permission_query_conditions breaks list views completely.

---

## 4. Throwing in has_permission

### Problem
```python
def has_permission(doc, user=None, permission_type=None):
    if doc.status == "Locked":
        frappe.throw("Document is locked")  # BREAKS DOCUMENT ACCESS
```

### Fix
```python
def has_permission(doc, user=None, permission_type=None):
    try:
        if doc.status == "Locked" and permission_type == "write":
            return False
        return None
    except Exception:
        return None
```

**Why**: Throwing in has_permission breaks document access entirely.

---

## 5. Missing frappe.db.commit() in Scheduler

### Problem
```python
def daily_task():
    for item in frappe.get_all("Item", limit=100):
        frappe.db.set_value("Item", item.name, "synced", 1)
    # ALL CHANGES LOST — no commit!
```

### Fix
```python
def daily_task():
    for item in frappe.get_all("Item", limit=100):
        frappe.db.set_value("Item", item.name, "synced", 1)
    frappe.db.commit()  # REQUIRED
```

**Why**: Scheduler tasks have no auto-commit. Without explicit commit, all changes are lost.

---

## 6. Silent Error Swallowing in Scheduler

### Problem
```python
def daily_sync():
    try:
        sync_records()
    except Exception:
        pass  # Silent death — impossible to debug
```

### Fix
```python
def daily_sync():
    try:
        sync_records()
        frappe.db.commit()
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Daily Sync Error")
```

**Why**: Scheduler has no user feedback. frappe.log_error() is your ONLY debugging tool.

---

## 7. Not Calling super() in Override Class

### Problem
```python
class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        self.custom_validation()  # Parent validation SKIPPED!
```

### Fix
```python
class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        super().validate()  # ALWAYS call parent first
        self.custom_validation()
```

**Why**: Skipping super() bypasses all parent validation, permissions, and business logic.

---

## 8. Swallowing Parent Errors in Override

### Problem
```python
class CustomDoc(OriginalDoc):
    def validate(self):
        try:
            super().validate()
        except Exception:
            pass  # Parent validation errors HIDDEN from user!
```

### Fix
```python
class CustomDoc(OriginalDoc):
    def validate(self):
        try:
            super().validate()
        except frappe.ValidationError:
            raise  # ALWAYS re-raise validation errors
        except Exception:
            frappe.log_error(frappe.get_traceback(), "Parent error")
            raise
```

**Why**: Parent validation errors MUST reach the user. Swallowing them causes data corruption.

---

## 9. Unprotected extend_bootinfo

### Problem
```python
def extend_boot(bootinfo):
    settings = frappe.get_single("My Settings")  # DoesNotExistError!
    bootinfo.config = settings.config
```

### Fix
```python
def extend_boot(bootinfo):
    bootinfo.config = {}
    try:
        if frappe.db.exists("My Settings", "My Settings"):
            settings = frappe.get_single("My Settings")
            bootinfo.config = settings.config or {}
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Bootinfo Error")
```

**Why**: Errors in extend_bootinfo break the entire desk/login page.

---

## 10. Committing in doc_events

### Problem
```python
def on_update(doc, method=None):
    frappe.db.set_value("Counter", "main", "count", 100)
    frappe.db.commit()  # BREAKS TRANSACTION
```

### Fix
```python
def on_update(doc, method=None):
    frappe.db.set_value("Counter", "main", "count", 100)
    # Framework handles commit automatically
```

**Why**: Manual commits in doc_events break transaction management and cause partial saves.

---

## 11. Not Isolating Non-Critical Operations

### Problem
```python
def on_submit(doc, method=None):
    send_notification_email(doc)  # If this fails...
    sync_to_external_system(doc)  # ...this never runs!
    update_dashboard(doc)
```

### Fix
```python
def on_submit(doc, method=None):
    try:
        send_notification_email(doc)
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Email Error")

    try:
        sync_to_external_system(doc)
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Sync Error")

    try:
        update_dashboard(doc)
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Dashboard Error")
```

**Why**: Independent operations MUST NOT block each other.

---

## 12. Breaking Other Apps in Wildcard Handler

### Problem
```python
doc_events = {"*": {"on_update": "myapp.audit.log_all"}}

def log_all(doc, method=None):
    frappe.get_doc({"doctype": "Audit Log", "doc": doc.name}).insert()
    # Error here breaks ALL saves system-wide!
```

### Fix
```python
def log_all(doc, method=None):
    try:
        if doc.doctype in ["Audit Log", "Error Log"]:
            return
        frappe.get_doc({"doctype": "Audit Log", "doc": doc.name}).insert(ignore_permissions=True)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Audit: {doc.doctype}/{doc.name}")
```

**Why**: Wildcard handlers run on ALL documents. Unhandled errors break the entire system.

---

## 13. SQL Injection in Permission Query

### Problem
```python
def query_conditions(user):
    return f"owner = '{user}'"  # SQL INJECTION!
```

### Fix
```python
def query_conditions(user):
    return f"owner = {frappe.db.escape(user)}"
```

**Why**: Unescaped user input allows SQL injection attacks.

---

## 14. No Limit in Scheduler Queries

### Problem
```python
def daily_sync():
    records = frappe.get_all("Item")  # Could return millions!
```

### Fix
```python
def daily_sync():
    records = frappe.get_all("Item", limit=1000)
```

**Why**: Unbounded queries cause memory exhaustion and timeouts in scheduler tasks.

---

## 15. Circular Import from hooks.py

### Problem
```python
# myapp/events/sales.py
from myapp.hooks import doc_events  # CIRCULAR IMPORT!
```

### Fix
```python
# NEVER import from hooks.py in event handlers
# hooks.py is read by the framework, not by your code
# Move shared config to myapp/constants.py instead
```

**Why**: hooks.py is a configuration file read by the framework. Importing from it creates circular dependencies.

---

## 16. Wrong Fixtures dt Key

### Problem
```python
fixtures = [
    {"doctype": "Custom Field", "filters": [...]}  # "doctype" is WRONG
]
```

### Fix
```python
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]}
]
```

**Why**: Fixtures use the `dt` key, not `doctype`. Wrong key causes silent failure during install.

---

## 17. extend_doctype_class [v16+] Method Name Collision

### Problem
```python
# App A extension:
class InvoiceMixin:
    def calculate_tax(self): ...

# App B extension:
class InvoiceMixin:
    def calculate_tax(self): ...  # SILENTLY OVERRIDES App A!
```

### Fix
```python
# App A:
class InvoiceMixin:
    def appa_calculate_tax(self): ...

# App B:
class InvoiceMixin:
    def appb_calculate_tax(self): ...
```

**Why**: With extend_doctype_class, the last extension's method wins silently. Prefix to avoid collisions.

---

## Quick Checklist: Hook Review

Before deploying hooks:

- [ ] Dotted paths match actual module + function names
- [ ] Dict structure correct for each hook type
- [ ] No `frappe.throw()` in permission hooks
- [ ] `frappe.db.commit()` in scheduler tasks
- [ ] `frappe.log_error()` for all caught exceptions
- [ ] `super()` called in override classes with re-raise
- [ ] `try/except` wrapper in extend_bootinfo
- [ ] No `frappe.db.commit()` in doc_events
- [ ] Non-critical operations isolated in try/except
- [ ] Wildcard handlers wrapped in try/except
- [ ] Queries have limits in scheduler
- [ ] User input escaped in SQL (permission hooks)
- [ ] No circular imports from hooks.py
- [ ] Fixtures use `dt` key (not `doctype`)
- [ ] extend_doctype_class methods prefixed with app name
- [ ] bench restarted after hooks.py changes

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-hooks/references/anti-patterns.md*
