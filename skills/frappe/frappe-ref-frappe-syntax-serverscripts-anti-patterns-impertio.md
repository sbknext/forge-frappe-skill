---
title: Frappe Ref — Frappe Syntax: Serverscripts: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'serverscripts', 'reference', 'anti-patterns']
source: Impertio
---

# Server Script Anti-Patterns and Common Mistakes

## Rule #1: NO Imports

The RestrictedPython sandbox blocks `__import__`. Every import statement fails.

```python
# ALL of these produce: ImportError: __import__ not found
import json                        # Use frappe.parse_json() / frappe.as_json()
import re                          # Not available — restructure logic
import math                        # Use sum(), min(), max(), round() builtins
import os                          # Not available
import requests                    # Use frappe.make_get_request()
import frappe                      # Already loaded — NEVER import it
from datetime import date          # Use frappe.utils.today()
from frappe.utils import cint      # Use frappe.utils.cint() directly
from collections import defaultdict  # Use dict with .get(key, default)
```

**ALWAYS** delete every `import` line. The frappe namespace is pre-loaded.

---

## Sandbox Violations

### File System Access — BLOCKED

```python
# WRONG:
open("/tmp/data.txt", "r")         # NameError: name 'open' is not defined
file = open("export.csv", "w")     # Not available

# CORRECT alternative:
frappe.log_error(data, "Export Data")  # Log to Error Log
```

### Dynamic Code Execution — BLOCKED

```python
# WRONG:
eval("1 + 1")                      # Blocked
exec("print('hello')")             # Blocked
compile("code", "", "exec")        # Blocked

# CORRECT: Write the logic directly — no dynamic evaluation
```

### OS / System Commands — BLOCKED

```python
# WRONG:
os.system("ls")                    # Not available
subprocess.run(["echo", "hi"])     # Not available

# CORRECT: Use a custom Frappe app for system-level operations
```

---

## Database Anti-Patterns

### SQL Injection

```python
# NEVER do this:
frappe.db.sql(f"SELECT * FROM tabUser WHERE name = '{user_input}'")
frappe.db.sql("SELECT * FROM tabUser WHERE name = '" + user_input + "'")

# ALWAYS use parameterized queries:
frappe.db.sql("""
    SELECT * FROM `tabUser`
    WHERE name = %(user)s
""", {"user": user_input}, as_dict=True)

# Or use ORM methods (automatically safe):
frappe.get_all("User", filters={"name": user_input})
```

### N+1 Query Problem

```python
# WRONG — N queries for N items:
for item in doc.items:
    name = frappe.db.get_value("Item", item.item_code, "item_name")

# CORRECT — 1 batch query:
codes = [item.item_code for item in doc.items]
items_map = {d.name: d.item_name for d in frappe.get_all(
    "Item",
    filters={"name": ["in", codes]},
    fields=["name", "item_name"]
)}
for item in doc.items:
    name = items_map.get(item.item_code)
```

### Unnecessary Commit in Document Events

```python
# WRONG in Document Event:
doc.total = 100
frappe.db.commit()           # Framework handles commit — NEVER do this

# CORRECT in Document Event:
doc.total = 100              # Framework commits after the event chain

# EXCEPTION — Scheduler scripts ALWAYS need commit:
frappe.db.set_value("Task", task.name, "status", "Done")
frappe.db.commit()           # Required in Scheduler
```

### set_value for Complex Updates

```python
# RISKY — bypasses all validation hooks:
frappe.db.set_value("Sales Invoice", "SINV-001", "grand_total", 1000)

# SAFER — triggers validate, permissions, linked doc updates:
inv = frappe.get_doc("Sales Invoice", "SINV-001")
inv.grand_total = 1000
inv.save()
```

---

## Performance Anti-Patterns

### Fetching Full Document for One Field

```python
# WRONG — loads ALL fields into memory:
customer = frappe.get_doc("Customer", doc.customer)
email = customer.email_id

# CORRECT — fetches only what you need:
email = frappe.db.get_value("Customer", doc.customer, "email_id")
```

### No Limit on Queries

```python
# WRONG — could return thousands of records:
all_invoices = frappe.get_all("Sales Invoice", filters={"docstatus": 1})

# CORRECT — ALWAYS set a limit:
invoices = frappe.get_all("Sales Invoice",
    filters={"docstatus": 1},
    limit=100,
    order_by="creation desc"
)
```

### Selecting All Fields

```python
# WRONG:
orders = frappe.get_all("Sales Order", filters={...}, fields=["*"])

# CORRECT — only needed fields:
orders = frappe.get_all("Sales Order",
    filters={...},
    fields=["name", "grand_total", "status"]
)
```

### Heavy Computation in Before Save

```python
# WRONG — slows down EVERY save:
total = frappe.db.sql("""
    SELECT SUM(grand_total) FROM `tabSales Invoice`
    WHERE customer = %(c)s
""", {"c": doc.customer})[0][0]
doc.lifetime_value = total

# CORRECT — use a Scheduler Event for heavy aggregations
```

---

## Logic Anti-Patterns

### Infinite Loop from Recursive Save

```python
# WRONG in Before Save — triggers Before Save again:
doc.total = calculate_total()
doc.save()                    # INFINITE LOOP

# CORRECT in Before Save — framework saves after event:
doc.total = calculate_total()
# NO doc.save() call — framework handles this
```

### Throw After Database Changes

```python
# WRONG — side effects happen even when save fails:
frappe.get_doc({"doctype": "Log", ...}).insert()
if doc.total < 0:
    frappe.throw("Invalid total")
# The Log record exists even though save was blocked!

# CORRECT — validate FIRST, then side effects:
if doc.total < 0:
    frappe.throw("Invalid total")
frappe.get_doc({"doctype": "Log", ...}).insert()
```

### Relying on Script Execution Order

```python
# WRONG — execution order of multiple Server Scripts is undefined:
# Script A (Before Save): doc.calc_value = complex_calc()
# Script B (Before Save): doc.derived = doc.calc_value * 2

# CORRECT — combine dependent logic in ONE script:
doc.calc_value = complex_calc()
doc.derived = doc.calc_value * 2
```

### Modifying doc in After Save Without Persisting

```python
# WRONG in After Save — field change is lost:
doc.note = "Updated"
# Not saved — doc was already written to DB

# CORRECT in After Save:
doc.db_set("note", "Updated", update_modified=False)
# Or:
frappe.db.set_value(doc.doctype, doc.name, "note", "Updated")
```

---

## Security Anti-Patterns

### No Permission Check in API Scripts

```python
# WRONG — any authenticated user can query:
data = frappe.get_doc("Customer", frappe.form_dict.get("customer"))
frappe.response["message"] = data.as_dict()

# CORRECT:
customer = frappe.form_dict.get("customer")
if not frappe.has_permission("Customer", "read", customer):
    frappe.throw("Access denied", frappe.PermissionError)
data = frappe.get_doc("Customer", customer)
frappe.response["message"] = data.as_dict()
```

### ignore_permissions Overuse

```python
# WRONG — bypasses all security:
doc.save(ignore_permissions=True)         # Why?
frappe.delete_doc("X", name, ignore_permissions=True)  # Dangerous

# CORRECT — only for system-generated records with explicit justification:
# Creating a system ToDo after verifying parent permission
if frappe.has_permission("Sales Order", "write", doc.name):
    frappe.get_doc({"doctype": "ToDo", ...}).insert(ignore_permissions=True)
```

### Sensitive Data in Error Logs

```python
# WRONG:
frappe.log_error(f"Auth failed: user={user}, password={pw}")

# CORRECT:
frappe.log_error(f"Auth failed for user: {user}")
```

### Guest Endpoint Exposing Internal Data

```python
# WRONG — Allow Guest = Yes with sensitive data:
frappe.response["message"] = frappe.get_all("Customer",
    fields=["name", "email_id", "tax_id", "outstanding_amount"])

# CORRECT — only expose non-sensitive fields:
frappe.response["message"] = frappe.get_all("Item",
    filters={"show_on_website": 1},
    fields=["item_name", "stock_uom"]
)
```

---

## Common Mistakes Summary

| Mistake | Fix |
|---|---|
| Any `import` statement | Remove it — use `frappe.*` namespace |
| `doc.save()` in Before Save | Remove it — framework saves automatically |
| `frappe.db.commit()` in Document Event | Remove it — framework commits automatically |
| Missing `frappe.db.commit()` in Scheduler | Add it — scheduler does NOT auto-commit |
| `doc.name` in Before Insert | Use After Insert — name may not exist yet |
| Modifying doc in After Save | Use `doc.db_set()` or `frappe.db.set_value()` |
| `f"WHERE x = '{var}'"` in SQL | Use `%(var)s` with parameters dict |
| `fields=["*"]` in get_all | Specify only needed fields |
| No `limit` in get_all | ALWAYS set a limit |
| No permission check in API script | ALWAYS check `frappe.has_permission()` |
| Permission Query without admin bypass | ALWAYS check for System Manager role first |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-serverscripts/references/anti-patterns.md*
