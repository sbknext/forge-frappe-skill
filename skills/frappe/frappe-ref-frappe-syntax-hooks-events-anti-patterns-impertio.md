---
title: Frappe Ref — Frappe Syntax: Hooks Events: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'events', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns and Common Mistakes

## 1. Calling doc.save() Inside validate or before_save

**WRONG** — causes infinite recursion:
```python
def validate(doc, method=None):
    doc.custom_field = "value"
    doc.save()  # NEVER — triggers validate again → infinite loop
```

**CORRECT** — mutate the doc directly (it will be saved automatically):
```python
def validate(doc, method=None):
    doc.custom_field = "value"  # Just set the value — Frappe saves it
```

---

## 2. Using raise Instead of frappe.throw()

**WRONG** — raw exceptions bypass Frappe's error handling:
```python
def validate(doc, method=None):
    if not doc.customer:
        raise ValueError("Customer is required")  # NEVER
```

**CORRECT** — `frappe.throw()` shows a user-friendly message and rolls back:
```python
def validate(doc, method=None):
    if not doc.customer:
        frappe.throw("Customer is required")  # ALWAYS
```

---

## 3. Using after_insert for Logic That Should Run on Every Save

**WRONG** — `after_insert` fires only once (on creation):
```python
doc_events = {
    "Sales Invoice": {
        "after_insert": "myapp.events.update_customer_balance"  # Misses updates!
    }
}
```

**CORRECT** — use `on_update` for logic that must run on every save:
```python
doc_events = {
    "Sales Invoice": {
        "on_update": "myapp.events.update_customer_balance"
    }
}
```

---

## 4. Creating Linked Documents in validate

**WRONG** — the parent document is not yet committed:
```python
def validate(doc, method=None):
    project = frappe.new_doc("Project")
    project.sales_order = doc.name
    project.insert()  # NEVER — doc may not be saved if later validation fails
```

**CORRECT** — create linked documents in `on_submit` or `after_insert`:
```python
def on_submit(doc, method=None):
    project = frappe.new_doc("Project")
    project.sales_order = doc.name
    project.insert()  # Safe — doc is committed
```

---

## 5. Slow Operations in Synchronous Handlers

**WRONG** — blocks the user's request:
```python
def on_submit(doc, method=None):
    import requests
    response = requests.post("https://external-api.com/notify", json={...})  # NEVER
    # User waits for external API response
```

**CORRECT** — use `frappe.enqueue()` for slow operations:
```python
def on_submit(doc, method=None):
    frappe.enqueue(
        "myapp.tasks.notify_external",
        queue="short",
        doc_name=doc.name,
    )
```

---

## 6. Missing super() Call in Controller Overrides

**WRONG** — breaks parent class logic:
```python
# override_doctype_class
class CustomToDo(ToDo):
    def validate(self):
        self.custom_field = "value"
        # Missing super().validate() — original validation skipped!
```

**CORRECT** — ALWAYS call super():
```python
class CustomToDo(ToDo):
    def validate(self):
        super().validate()  # ALWAYS call super() first
        self.custom_field = "value"
```

---

## 7. Assuming Hooks Run as Administrator

**WRONG** — hooks run as the current session user:
```python
def on_submit(doc, method=None):
    other_doc = frappe.get_doc("Salary Slip", doc.employee)
    other_doc.custom_field = "value"
    other_doc.save()  # May fail with PermissionError
```

**CORRECT** — explicitly set permission flags when needed:
```python
def on_submit(doc, method=None):
    other_doc = frappe.get_doc("Salary Slip", doc.employee)
    other_doc.custom_field = "value"
    other_doc.flags.ignore_permissions = True  # ALWAYS be explicit
    other_doc.save()
```

---

## 8. Wrong Event Name (Silent Failure)

**WRONG** — misspelled event names are silently ignored:
```python
doc_events = {
    "Sales Invoice": {
        "on_validated": "myapp.events.handler"  # WRONG — correct name is "validate"
    }
}
```

Frappe does NOT warn about unrecognized event names. The handler simply never fires.

**Valid event names** (exhaustive list):
- `before_insert`, `after_insert`
- `before_naming`, `autoname`
- `before_validate`, `validate`
- `before_save`, `on_update`, `on_change`
- `before_submit`, `on_submit`
- `before_cancel`, `on_cancel`
- `on_trash`, `after_delete`
- `before_rename`, `after_rename`
- `before_update_after_submit`, `on_update_after_submit`

ALWAYS copy event names from this list — NEVER type them from memory.

---

## 9. Modifying doc.name Outside of Naming Events

**WRONG** — causes database integrity issues:
```python
def validate(doc, method=None):
    doc.name = f"CUSTOM-{doc.name}"  # NEVER modify name here
```

**CORRECT** — use `autoname` or `before_naming` only:
```python
# In the controller class
class MyDocType(Document):
    def autoname(self):
        self.name = f"CUSTOM-{self.naming_series}"
```

---

## 10. Relying on on_change for Critical Logic

**WRONG** — `on_change` only fires when values differ from DB:
```python
doc_events = {
    "Sales Invoice": {
        "on_change": "myapp.events.send_critical_notification"  # May not fire!
    }
}
```

If a user clicks "Save" without changing any values, `on_change` does NOT fire.

**CORRECT** — use `on_update` for logic that must run on every save:
```python
doc_events = {
    "Sales Invoice": {
        "on_update": "myapp.events.send_critical_notification"
    }
}
```

---

## 11. Infinite Loop with Wildcard Handlers

**WRONG** — wildcard handler creates a doc, which triggers the handler again:
```python
doc_events = {
    "*": {
        "on_update": "myapp.events.audit.log_all_changes"
    }
}

def log_all_changes(doc, method=None):
    frappe.get_doc({
        "doctype": "Custom Log",
        "message": f"{doc.doctype} updated"
    }).insert()  # This insert triggers on_update for "Custom Log" → infinite loop!
```

**CORRECT** — ALWAYS exclude logging/audit DocTypes:
```python
def log_all_changes(doc, method=None):
    if doc.doctype in ("Custom Log", "Activity Log", "Comment", "Version"):
        return  # Break the loop

    frappe.get_doc({
        "doctype": "Custom Log",
        "message": f"{doc.doctype} updated"
    }).insert(ignore_permissions=True)
```

---

## 12. Using override_doctype_class When Multiple Apps Extend

**WRONG** — only the last-installed app's override applies:
```python
# App A hooks.py
override_doctype_class = {"Sales Invoice": "app_a.overrides.CustomSI"}

# App B hooks.py
override_doctype_class = {"Sales Invoice": "app_b.overrides.CustomSI"}
# App B silently wins — App A's override is lost
```

**CORRECT (v16+)** — use `extend_doctype_class` for cumulative extensions:
```python
# App A hooks.py
extend_doctype_class = {"Sales Invoice": ["app_a.mixins.SIMixin"]}

# App B hooks.py
extend_doctype_class = {"Sales Invoice": ["app_b.mixins.SIMixin"]}
# Both extensions apply via MRO
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks-events/references/anti-patterns.md*
