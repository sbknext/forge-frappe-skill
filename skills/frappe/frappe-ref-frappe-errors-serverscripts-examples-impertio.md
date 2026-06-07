---
title: Frappe Ref — Frappe Errors: Serverscripts: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'serverscripts', 'reference', 'examples']
source: Impertio
---

# Server Script Error Examples — Real Scenarios

Complete diagnosis-oriented examples showing actual errors, root cause, and fix.

---

## Scenario 1: ImportError — The #1 Server Script Error

**Error:**
```
ImportError: __import__ not found
```

**The broken code:**
```python
# Type: Document Event, Event: Before Save
import json
import datetime

data = json.loads(doc.custom_json)
if datetime.date.today() > doc.due_date:
    frappe.throw("Overdue!")
```

**Root cause:** ALL `import` statements are blocked by the RestrictedPython sandbox (except `json` which is pre-loaded).

**The fix:**
```python
# Type: Document Event, Event: Before Save
data = frappe.parse_json(doc.custom_json)
if frappe.utils.today() > str(doc.due_date):
    frappe.throw("Overdue!")
```

**Complete import replacement table:**

| Blocked Import | Sandbox Equivalent |
|----------------|-------------------|
| `import json` / `json.loads()` | `frappe.parse_json()` |
| `import json` / `json.dumps()` | `frappe.as_json()` |
| `from datetime import datetime` | `frappe.utils.now()`, `frappe.utils.today()` |
| `from datetime import timedelta` | `frappe.utils.add_days()`, `frappe.utils.date_diff()` |
| `import requests` | `frappe.make_get_request()`, `frappe.make_post_request()` |
| `import re` | Not available — use Python string methods |
| `import os` / `import sys` | Not available — security restriction |
| `import math` | Use Python arithmetic operators |

---

## Scenario 2: NameError — Restricted Builtins

**Error:**
```
NameError: name 'dict' is not defined
```

**The broken code:**
```python
result = dict(name=doc.name, status=doc.status)
items = list(doc.items)
total = sum(item.qty for item in doc.items)
```

**Root cause:** Some Python builtins are restricted in the sandbox. `dict()`, `list()`, and `sum()` may not be available depending on Frappe version.

**The fix:**
```python
result = frappe._dict({"name": doc.name, "status": doc.status})
items = [item for item in doc.items]  # List comprehension works
total = 0
for item in doc.items:
    total += item.qty or 0
```

---

## Scenario 3: try/except Blocked

**Error:**
```
SyntaxError: Line 3: Try/Except not allowed
```

**The broken code:**
```python
# Type: Document Event, Event: Before Save
try:
    customer = frappe.get_doc("Customer", doc.customer)
    doc.territory = customer.territory
except Exception:
    doc.territory = "Default"
```

**The fix:**
```python
# Type: Document Event, Event: Before Save
if frappe.db.exists("Customer", doc.customer):
    territory = frappe.db.get_value("Customer", doc.customer, "territory")
    doc.territory = territory or "Default"
else:
    doc.territory = "Default"
```

---

## Scenario 4: Script Not Executing — Wrong Script Type

**Symptom:** Script saved and enabled, but nothing happens when document is saved.

**The broken configuration:**
```
Script Type: API
DocType: Sales Order
Script:
    if not doc.customer:
        frappe.throw("Customer required")
```

**Root cause:** API scripts are triggered by HTTP requests, NOT by document events. The script type should be "Document Event" with event "Before Save".

**The fix:**
```
Script Type: Document Event
DocType: Sales Order
Event: Before Save
Script:
    if not doc.customer:
        frappe.throw("Customer required")
```

---

## Scenario 5: Scheduler Changes Not Saved

**Symptom:** Scheduler runs without errors, but database values remain unchanged.

**The broken code:**
```python
# Type: Scheduler Event, Cron: 0 9 * * *
items = frappe.get_all("Item", filters={"sync_pending": 1}, limit=100)
for item in items:
    frappe.db.set_value("Item", item.name, "sync_pending", 0)
# Missing frappe.db.commit()!
```

**Root cause:** Scheduler scripts do NOT auto-commit. All changes are rolled back when script ends.

**The fix:**
```python
# Type: Scheduler Event, Cron: 0 9 * * *
items = frappe.get_all("Item", filters={"sync_pending": 1}, limit=100)
for item in items:
    frappe.db.set_value("Item", item.name, "sync_pending", 0)
frappe.db.commit()  # REQUIRED — without this, all changes are lost
```

---

## Scenario 6: API Script Returns Empty Response

**Symptom:** Client calls API script but `r.message` is `undefined`.

**The broken code:**
```python
# Type: API, Method: get_customer_info
customer = frappe.form_dict.get("customer")
data = frappe.db.get_value("Customer", customer, ["name", "credit_limit"], as_dict=True)
# Forgot to set response!
```

**The fix:**
```python
# Type: API, Method: get_customer_info
customer = frappe.form_dict.get("customer")
if not customer:
    frappe.throw("Parameter 'customer' is required", exc=frappe.ValidationError)
if not frappe.db.exists("Customer", customer):
    frappe.throw(f"Customer '{customer}' not found", exc=frappe.DoesNotExistError)

data = frappe.db.get_value("Customer", customer, ["name", "credit_limit"], as_dict=True)
frappe.response["message"] = data  # REQUIRED — this is what the client receives
```

---

## Scenario 7: doc.save() in Before Save — Infinite Recursion

**Error:**
```
RecursionError: maximum recursion depth exceeded
```

**The broken code:**
```python
# Type: Document Event, Event: Before Save
doc.custom_total = sum((item.qty or 0) * (item.rate or 0) for item in (doc.items or []))
doc.save()  # Triggers Before Save again → infinite loop!
```

**The fix:**
```python
# Type: Document Event, Event: Before Save
doc.custom_total = 0
for item in (doc.items or []):
    doc.custom_total += (item.qty or 0) * (item.rate or 0)
# No doc.save() — framework saves automatically after Before Save completes
```

---

## Scenario 8: SQL Injection in Permission Query

**Vulnerability demonstrated:**
```python
# Type: Permission Query, DocType: Project
# User submits territory = "'; DROP TABLE tabProject; --"
territory = frappe.db.get_value("User", user, "territory")
conditions = f"`tabProject`.territory = '{territory}'"
# If territory contains SQL, this is an injection!
```

**The fix:**
```python
# Type: Permission Query, DocType: Project
territory = frappe.db.get_value("User", user, "territory")
if territory:
    conditions = f"`tabProject`.territory = {frappe.db.escape(territory)}"
else:
    conditions = f"`tabProject`.owner = {frappe.db.escape(user)}"
```

---

## Scenario 9: Wrong Exception Type in API Script

**Symptom:** Client receives HTTP 417 (default) instead of expected 404.

**The broken code:**
```python
# Type: API
if not frappe.db.exists("Customer", customer):
    frappe.throw("Customer not found")  # Returns 417 Expectation Failed
```

**The fix:**
```python
# Type: API
if not frappe.db.exists("Customer", customer):
    frappe.throw("Customer not found", exc=frappe.DoesNotExistError)  # Returns 404
```

---

## Scenario 10: Scheduler Without Query Limit — Memory Exhaustion

**Error:** Worker process killed (OOM) or script timeout.

**The broken code:**
```python
# Type: Scheduler Event
all_invoices = frappe.get_all("Sales Invoice", fields=["name", "customer", "grand_total"])
# On a system with 500,000+ invoices, this loads everything into memory!
```

**The fix:**
```python
# Type: Scheduler Event
BATCH_SIZE = 50
invoices = frappe.get_all(
    "Sales Invoice",
    filters={"status": "Unpaid", "docstatus": 1},
    fields=["name", "customer"],
    limit=500  # ALWAYS limit
)

for i in range(0, len(invoices), BATCH_SIZE):
    batch = invoices[i:i + BATCH_SIZE]
    for inv in batch:
        process_invoice(inv)
    frappe.db.commit()  # Commit per batch
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-serverscripts/references/examples.md*
