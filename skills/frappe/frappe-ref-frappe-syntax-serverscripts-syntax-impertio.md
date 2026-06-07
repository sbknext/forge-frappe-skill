---
title: Frappe Ref — Frappe Syntax: Serverscripts: Syntax (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'serverscripts', 'reference', 'syntax']
source: Impertio
---

# Server Script Syntax — Quick Cheat Sheet

## Script Configuration Fields

| Field | Document Event | API | Scheduler | Permission Query |
|---|---|---|---|---|
| Script Type | Document Event | API | Scheduler Event | Permission Query |
| Reference DocType | Required | - | - | Required |
| DocType Event | Required | - | - | - |
| API Method | - | Required | - | - |
| Allow Guest | - | Optional | - | - |
| Event Frequency | - | - | Required | - |
| Cron Format | - | - | If Cron | - |

---

## Available Variables Per Script Type

### Document Event

```python
doc           # The current document object (frappe.model.Document)
frappe        # Core namespace
```

### API

```python
frappe.form_dict    # Request parameters (GET query + POST body)
frappe.request      # Werkzeug request object
frappe.response     # Response dict — set frappe.response["message"] for output
frappe              # Core namespace
```

### Scheduler Event

```python
frappe        # Core namespace
# No doc, no form_dict — you query what you need
```

### Permission Query

```python
user          # str — email of the user being checked
conditions    # str — set this to a SQL WHERE fragment
frappe        # Core namespace
```

---

## Cron Format Reference

```
┌───────────── minute (0-59)
│ ┌─────────── hour (0-23)
│ │ ┌───────── day of month (1-31)
│ │ │ ┌─────── month (1-12)
│ │ │ │ ┌───── day of week (0-6, 0=Sunday)
│ │ │ │ │
* * * * *
```

| Schedule | Cron Expression |
|---|---|
| Every minute | `* * * * *` |
| Every 15 minutes | `*/15 * * * *` |
| Every hour | `0 * * * *` |
| Daily at 9:00 | `0 9 * * *` |
| Daily at midnight | `0 0 * * *` |
| Weekdays at 8:00 | `0 8 * * 1-5` |
| Sunday at 02:00 | `0 2 * * 0` |
| 1st of month at 06:00 | `0 6 1 * *` |
| Every 6 hours | `0 */6 * * *` |

---

## Common Patterns — One-Liners

### Validation

```python
if not doc.customer:
    frappe.throw("Customer is required")

if doc.grand_total < 0:
    frappe.throw("Total MUST NOT be negative")

if doc.discount_percentage > 50:
    frappe.throw("Max discount is 50%", title="Validation Error")
```

### Safe Type Conversion

```python
qty = frappe.utils.flt(doc.qty)          # None/str → float, default 0.0
count = frappe.utils.cint(doc.count)     # None/str → int, default 0
name = frappe.utils.cstr(doc.name)       # None → ""
```

### Date Operations

```python
today = frappe.utils.today()                          # "2024-01-15"
tomorrow = frappe.utils.add_days(today, 1)            # "2024-01-16"
next_month = frappe.utils.add_months(today, 1)        # "2024-02-15"
days_ago = frappe.utils.date_diff(today, some_date)   # int
month_start = frappe.utils.get_first_day(today)       # "2024-01-01"
month_end = frappe.utils.get_last_day(today)           # "2024-01-31"
```

### JSON Operations

```python
data = frappe.parse_json(json_string)    # str → dict/list
text = frappe.as_json(python_obj)        # dict/list → str
```

### Database Shortcuts

```python
# Check existence
if frappe.db.exists("Customer", "CUST-001"):
    pass

# Get one value
name = frappe.db.get_value("Customer", "CUST-001", "customer_name")

# Get multiple fields
vals = frappe.db.get_value("Customer", "CUST-001",
    ["customer_name", "territory"], as_dict=True)

# Count
n = frappe.db.count("Sales Invoice", filters={"status": "Unpaid"})

# Flat list of names
names = frappe.get_all("Customer", filters={...}, pluck="name")
```

### API Response

```python
frappe.response["message"] = {"status": "ok", "data": result}
```

### Permission Query Output

```python
# Full access
conditions = ""

# Owner only
conditions = f"`tabDocType`.owner = {frappe.db.escape(user)}"

# No access
conditions = "1=0"
```

---

## Filter Operators

| Operator | Example | Meaning |
|---|---|---|
| `=` (default) | `{"status": "Open"}` | Equals |
| `!=` | `{"status": ["!=", "Closed"]}` | Not equals |
| `>` | `{"amount": [">", 1000]}` | Greater than |
| `<` | `{"date": ["<", today]}` | Less than |
| `>=` | `{"amount": [">=", 100]}` | Greater or equal |
| `<=` | `{"amount": ["<=", 9999]}` | Less or equal |
| `like` | `{"name": ["like", "SO-%"]}` | SQL LIKE |
| `not like` | `{"name": ["not like", "TEST%"]}` | SQL NOT LIKE |
| `in` | `{"status": ["in", ["A", "B"]]}` | In list |
| `not in` | `{"status": ["not in", ["X"]]}` | Not in list |
| `is` | `{"field": ["is", "set"]}` | IS NOT NULL |
| `is` | `{"field": ["is", "not set"]}` | IS NULL |
| `between` | `{"date": ["between", [d1, d2]]}` | Between two values |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-serverscripts/references/syntax.md*
