---
title: Frappe Ref — Frappe Syntax: Reports: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'reports', 'reference', 'anti-patterns']
source: Impertio
---

# Common Report Anti-Patterns

## AP-001: Wrong Column Format for Report Type

### Problem
Using legacy string format in Script Reports or dict format in Query Report SQL aliases.

### Wrong
```python
# In a Script Report .py file — legacy string format does NOT work here
def execute(filters=None):
    columns = [
        "Customer:Link/Customer:200",
        "Amount:Currency:120"
    ]
```

### Correct
```python
# Script Reports ALWAYS use dict format
def execute(filters=None):
    columns = [
        {"fieldname": "customer", "label": _("Customer"),
         "fieldtype": "Link", "options": "Customer", "width": 200},
        {"fieldname": "amount", "label": _("Amount"),
         "fieldtype": "Currency", "width": 120}
    ]
```

**Rule**: ALWAYS use dict format for Script Reports. Legacy string format is ONLY for Query Report SQL aliases.

---

## AP-002: Returning None Instead of Empty Lists

### Problem
Returning `None` for columns or data crashes the report renderer.

### Wrong
```python
def execute(filters=None):
    if not filters.get("company"):
        return None, None  # Crashes!
```

### Correct
```python
def execute(filters=None):
    if not filters.get("company"):
        return [], []  # Safe empty return
```

**Rule**: ALWAYS return `[]` for empty columns and data — NEVER `None`.

---

## AP-003: SQL Injection via String Formatting

### Problem
Building SQL with f-strings or `.format()` using user-supplied filter values.

### Wrong
```python
def get_data(filters):
    # DANGEROUS — SQL injection vulnerability
    return frappe.db.sql(f"""
        SELECT name FROM `tabSales Order`
        WHERE customer = '{filters.get("customer")}'
    """, as_dict=True)
```

### Correct
```python
def get_data(filters):
    return frappe.db.sql("""
        SELECT name FROM `tabSales Order`
        WHERE customer = %(customer)s
    """, filters, as_dict=True)
```

**Rule**: ALWAYS use `%(param)s` placeholders and pass `filters` dict to `frappe.db.sql`. NEVER interpolate user input into SQL strings.

---

## AP-004: Missing docstatus Filter

### Problem
Querying submitted documents without filtering by `docstatus`, which returns Draft and Cancelled documents.

### Wrong
```python
data = frappe.db.sql("""
    SELECT name, grand_total FROM `tabSales Invoice`
    WHERE customer = %(customer)s
""", filters, as_dict=True)
```

### Correct
```python
data = frappe.db.sql("""
    SELECT name, grand_total FROM `tabSales Invoice`
    WHERE docstatus = 1
    AND customer = %(customer)s
""", filters, as_dict=True)
```

**Rule**: ALWAYS include `docstatus = 1` when querying submitted (final) documents. Use `docstatus < 2` to include both Draft and Submitted but exclude Cancelled.

---

## AP-005: Mismatched Chart Labels and Values

### Problem
Chart `labels` array and `datasets[].values` arrays have different lengths, causing rendering errors or blank charts.

### Wrong
```python
chart = {
    "data": {
        "labels": ["Jan", "Feb", "Mar"],      # 3 labels
        "datasets": [
            {"name": "Revenue", "values": [100, 200]}  # 2 values — MISMATCH
        ]
    },
    "type": "bar"
}
```

### Correct
```python
labels = [row.month for row in data]
values = [flt(row.amount) for row in data]

chart = {
    "data": {
        "labels": labels,       # Same source, guaranteed equal length
        "datasets": [
            {"name": _("Revenue"), "values": values}
        ]
    },
    "type": "bar"
}
```

**Rule**: ALWAYS derive labels and values from the same data source to guarantee equal length.

---

## AP-006: Missing Column Width

### Problem
Omitting `width` in column definitions causes columns to render too narrow or overlap.

### Wrong
```python
columns = [
    {"fieldname": "customer", "label": _("Customer"),
     "fieldtype": "Link", "options": "Customer"}
    # No width — renders poorly
]
```

### Correct
```python
columns = [
    {"fieldname": "customer", "label": _("Customer"),
     "fieldtype": "Link", "options": "Customer", "width": 200}
]
```

**Rule**: ALWAYS specify `width` (in pixels) for every column definition.

---

## AP-007: Missing Reference DocType

### Problem
Creating a report without setting Reference DocType. The report is only visible to Administrators.

### Wrong
Report document with no Reference DocType set.

### Correct
ALWAYS set Reference DocType to the primary DocType the report queries. This controls:
- Who can see the report (users with read permission on that DocType)
- Where the report appears in the sidebar

**Rule**: ALWAYS set Reference DocType on every report.

---

## AP-008: Using frappe.query_reports (Plural) for Instance Access

### Problem
Confusing `frappe.query_reports` (the registry object) with `frappe.query_report` (the active instance).

### Wrong
```javascript
// This accesses the configuration object, NOT the running instance
frappe.query_reports["My Report"].refresh();  // Does nothing
```

### Correct
```javascript
// This accesses the active report instance
frappe.query_report.refresh();  // Actually refreshes
frappe.query_report.get_filter_value("company");  // Gets filter value
```

**Rule**: Use `frappe.query_reports["Name"]` ONLY for defining configuration. Use `frappe.query_report` (singular) for runtime operations.

---

## AP-009: Not Handling DateRange Filter Correctly

### Problem
DateRange filter returns a list `[from_date, to_date]`, not individual date values. Using it directly in SQL fails.

### Wrong
```python
def execute(filters=None):
    # filters.date_range = ["2024-01-01", "2024-03-31"]
    data = frappe.db.sql("""
        SELECT name FROM `tabSales Order`
        WHERE transaction_date BETWEEN %(date_range)s  -- FAILS
    """, filters, as_dict=True)
```

### Correct
```python
def execute(filters=None):
    if filters.get("date_range"):
        filters["from_date"] = filters["date_range"][0]
        filters["to_date"] = filters["date_range"][1]

    data = frappe.db.sql("""
        SELECT name FROM `tabSales Order`
        WHERE transaction_date BETWEEN %(from_date)s AND %(to_date)s
    """, filters, as_dict=True)
```

**Rule**: ALWAYS unpack DateRange filters into separate `from_date` and `to_date` keys before using in SQL.

---

## AP-010: Forgetting Translation Wrappers

### Problem
Hardcoding English strings in labels without `_()` or `__()` makes reports untranslatable.

### Wrong
```python
columns = [
    {"fieldname": "customer", "label": "Customer", ...}  # Not translatable
]
report_summary = [
    {"value": total, "label": "Total Revenue", ...}  # Not translatable
]
```

### Correct
```python
columns = [
    {"fieldname": "customer", "label": _("Customer"), ...}
]
report_summary = [
    {"value": total, "label": _("Total Revenue"), ...}
]
```

```javascript
// In JS files
{ fieldname: "customer", label: __("Customer"), ... }
```

**Rule**: ALWAYS wrap label strings with `_()` in Python and `__()` in JavaScript.

---

## AP-011: Missing `as_dict=True` in SQL Queries

### Problem
Forgetting `as_dict=True` when columns use dict format. Data comes back as tuples instead of dicts, and fieldnames do not map to column definitions.

### Wrong
```python
data = frappe.db.sql("""
    SELECT customer, grand_total FROM `tabSales Order`
""", filters)  # Returns list of tuples
```

### Correct
```python
data = frappe.db.sql("""
    SELECT customer, grand_total FROM `tabSales Order`
""", filters, as_dict=True)  # Returns list of dicts
```

**Rule**: When using dict-format columns, ALWAYS use `as_dict=True` in `frappe.db.sql()` so that `fieldname` keys in columns match the dict keys in data rows.

---

## AP-012: Not Escaping Percent Signs in SQL with Date Functions

### Problem
Python string formatting interprets `%` in SQL date functions as format specifiers.

### Wrong
```python
frappe.db.sql("""
    SELECT DATE_FORMAT(posting_date, '%Y-%m') as month
    FROM `tabSales Invoice`
""", filters, as_dict=True)
# Error: not enough arguments for format string
```

### Correct
```python
frappe.db.sql("""
    SELECT DATE_FORMAT(posting_date, '%%Y-%%m') as month
    FROM `tabSales Invoice`
""", filters, as_dict=True)
# Double %% escapes the percent sign
```

**Rule**: ALWAYS use `%%` to escape percent signs in SQL when using `frappe.db.sql` with parameters.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-reports/references/anti-patterns.md*
