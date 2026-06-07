---
title: Frappe Ref — Frappe Errors: Serverscripts: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'serverscripts', 'reference', 'anti-patterns']
source: Impertio
---

# Server Script Anti-Patterns — Error Prevention

Each anti-pattern shows the mistake, why it fails, and the correct approach.

---

## 1. Using import Statements

```python
# ❌ BLOCKED — ImportError: __import__ not found
import json
import datetime
from collections import defaultdict

# ✅ CORRECT — Use frappe namespace
data = frappe.parse_json(doc.json_field)
today = frappe.utils.today()
result = frappe._dict()
```
**Why:** RestrictedPython blocks all imports. Only `json` module is pre-loaded. Use `frappe.utils`, `frappe.parse_json()`, `frappe.make_get_request()`.

---

## 2. Using try/except Blocks

```python
# ❌ BLOCKED — SyntaxError: Try/Except not allowed
try:
    customer = frappe.get_doc("Customer", doc.customer)
except Exception:
    frappe.throw("Not found")

# ✅ CORRECT — Conditional check first
if not frappe.db.exists("Customer", doc.customer):
    frappe.throw(f"Customer '{doc.customer}' not found")
customer = frappe.get_doc("Customer", doc.customer)
```
**Why:** RestrictedPython blocks exception handling [v14-v15]. Use `if/else` with existence checks.

---

## 3. Using raise Statement

```python
# ❌ BLOCKED — SyntaxError: raise not allowed
if amount < 0:
    raise ValueError("Negative amount")

# ✅ CORRECT
if amount < 0:
    frappe.throw("Amount cannot be negative")
```
**Why:** `raise` is blocked. Use `frappe.throw()` which raises `frappe.ValidationError` internally.

---

## 4. Calling doc.save() in Before Save

```python
# ❌ WRONG — Infinite recursion
# Event: Before Save
doc.status = "Validated"
doc.save()  # Triggers Before Save again!

# ✅ CORRECT — Just set the value
# Event: Before Save
doc.status = "Validated"
# Framework saves automatically after Before Save
```
**Why:** `doc.save()` in Before Save triggers the event again, causing infinite recursion.

---

## 5. Forgetting frappe.db.commit() in Scheduler

```python
# ❌ WRONG — All changes lost!
# Type: Scheduler Event
for item in frappe.get_all("Item", filters={"sync_pending": 1}, limit=100):
    frappe.db.set_value("Item", item.name, "sync_pending", 0)

# ✅ CORRECT
for item in frappe.get_all("Item", filters={"sync_pending": 1}, limit=100):
    frappe.db.set_value("Item", item.name, "sync_pending", 0)
frappe.db.commit()  # REQUIRED
```
**Why:** Scheduler scripts do NOT auto-commit. Without explicit commit, all changes are rolled back.

---

## 6. No Query Limit in Scheduler

```python
# ❌ WRONG — May load millions of records
# Type: Scheduler Event
all_records = frappe.get_all("Sales Invoice", fields=["*"])

# ✅ CORRECT — Always limit
records = frappe.get_all("Sales Invoice",
    filters={"status": "Unpaid", "docstatus": 1},
    fields=["name", "customer"],
    limit=500
)
```
**Why:** Unlimited queries exhaust memory and crash the worker process.

---

## 7. Not Escaping User Input in SQL

```python
# ❌ VULNERABLE — SQL injection
territory = frappe.form_dict.get("territory")
conditions = f"`tabCustomer`.territory = '{territory}'"

# ✅ SAFE
territory = frappe.form_dict.get("territory")
conditions = f"`tabCustomer`.territory = {frappe.db.escape(territory)}"
```
**Why:** Unescaped user input allows SQL injection attacks.

---

## 8. Not Checking Record Existence Before get_doc

```python
# ❌ WRONG — Crashes with DoesNotExistError
customer = frappe.get_doc("Customer", doc.customer)

# ✅ CORRECT — Check first
if not frappe.db.exists("Customer", doc.customer):
    frappe.throw(f"Customer '{doc.customer}' not found")
customer = frappe.get_doc("Customer", doc.customer)
```
**Why:** `frappe.get_doc()` raises an exception if record doesn't exist. Without try/except in sandbox, this crashes the script.

---

## 9. Throwing on First Error

```python
# ❌ WRONG — User fixes one error, hits the next
if not doc.customer: frappe.throw("Customer required")
if not doc.delivery_date: frappe.throw("Date required")
if not doc.items: frappe.throw("Items required")

# ✅ CORRECT — Collect all errors
errors = []
if not doc.customer: errors.append("Customer is required")
if not doc.delivery_date: errors.append("Delivery Date is required")
if not doc.items: errors.append("At least one item is required")
if errors:
    frappe.throw("<br>".join(errors), title="Please fix these errors")
```
**Why:** Users should see ALL errors at once, not one at a time.

---

## 10. Exposing Technical Errors

```python
# ❌ WRONG — Confusing for users
if not customer_data:
    frappe.throw(f"KeyError: 'credit_limit' not found in dict")

# ✅ CORRECT — Actionable message
if not customer_data:
    frappe.throw(f"Customer '{doc.customer}' not found. Please select a valid customer.")
```
**Why:** Technical messages confuse users. Provide clear, actionable instructions.

---

## 11. Silent Failures in Scheduler

```python
# ❌ WRONG — No logging, debugging impossible
# Type: Scheduler Event
for inv in invoices:
    if not inv.customer:
        continue  # Silent skip

# ✅ CORRECT — Log all skips and errors
errors = []
for inv in invoices:
    if not inv.customer:
        errors.append(f"{inv.name}: Missing customer")
        continue
    process(inv)

if errors:
    frappe.log_error("\n".join(errors), "Processing Errors")
frappe.db.commit()
```
**Why:** Scheduler has no user to see errors. ALWAYS log for debugging.

---

## 12. Wrong Exception Type in API Scripts

```python
# ❌ WRONG — Returns 417 for "not found" (should be 404)
# Type: API
if not frappe.db.exists("Customer", customer):
    frappe.throw("Not found")  # Default: ValidationError → 417

# ✅ CORRECT — Use proper exception type
if not frappe.db.exists("Customer", customer):
    frappe.throw("Not found", exc=frappe.DoesNotExistError)  # → 404
```
**Why:** Correct HTTP status codes help API consumers handle errors properly.

---

## 13. Modifying doc After on_update Event

```python
# ❌ WRONG — Changes NOT saved
# Event: After Save
doc.sync_status = "Synced"  # Lost!

# ✅ CORRECT — Use frappe.db.set_value
# Event: After Save
frappe.db.set_value(doc.doctype, doc.name, "sync_status", "Synced")
```
**Why:** After save, changes to `doc` object are not persisted. Use `frappe.db.set_value()`.

---

## 14. Assuming doc Exists in API/Scheduler Scripts

```python
# ❌ WRONG — doc is undefined in API scripts!
# Type: API
frappe.response["message"] = doc.customer  # NameError: doc not defined

# ✅ CORRECT — Use frappe.form_dict for API parameters
# Type: API
customer = frappe.form_dict.get("customer")
if not customer:
    frappe.throw("'customer' parameter required", exc=frappe.ValidationError)
frappe.response["message"] = customer
```
**Why:** `doc` is only available in Document Event scripts, not in API or Scheduler scripts.

---

## 15. Assuming Child Table Values Are Not None

```python
# ❌ WRONG — Crashes if qty or rate is None
total = sum(item.qty * item.rate for item in doc.items)

# ✅ CORRECT — Default to 0
total = sum((item.qty or 0) * (item.rate or 0) for item in (doc.items or []))
```
**Why:** Child table fields can be None. Always provide default values.

---

## Pre-Deploy Checklist

- [ ] No `import` statements (except `json`)
- [ ] No `try/except` or `raise` statements
- [ ] No `doc.save()` in Before Save events
- [ ] All database lookups have existence checks
- [ ] Multiple errors collected before `frappe.throw()`
- [ ] Scheduler scripts have `frappe.db.commit()`
- [ ] Scheduler scripts have query `limit`
- [ ] All user input escaped in SQL conditions
- [ ] API scripts use correct exception types
- [ ] Scheduler errors logged with `frappe.log_error()`
- [ ] API scripts set `frappe.response["message"]`
- [ ] Child table values have defaults (`or 0`, `or []`)

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-serverscripts/references/anti-patterns.md*
