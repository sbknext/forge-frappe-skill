---
title: Frappe Ref — Frappe Syntax: Reports: Query Report (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'reports', 'reference', 'query-report']
source: Impertio
---

# Query Report API Reference

## Overview

Query Reports execute SQL directly against the database. They are stored in the Report document and do NOT require Developer Mode. System Manager role is required to create them.

## Creating a Query Report

1. Navigate to "New Report" via the awesomebar
2. Set **Report Type** = "Query Report"
3. Set **Reference DocType** (controls who can access the report)
4. Set **Module** (determines sidebar placement)
5. Write SQL in the **Query** field

## SQL Query with Legacy Column Format

The column format is embedded in SQL aliases:

```sql
SELECT
  `tabSales Order`.name as "Sales Order:Link/Sales Order:200",
  `tabSales Order`.customer as "Customer:Link/Customer:180",
  `tabSales Order`.transaction_date as "Date:Date:100",
  `tabSales Order`.grand_total as "Grand Total:Currency:120",
  `tabSales Order`.status as "Status:Data:100",
  `tabSales Order`.per_delivered as "Delivered %:Percent:100",
  `tabSales Order`.company as "Company:Link/Company:150"
FROM
  `tabSales Order`
WHERE
  `tabSales Order`.docstatus = 1
ORDER BY
  `tabSales Order`.transaction_date DESC
```

### Column Alias Format

```
"Label:Fieldtype/Options:Width"
```

| Part | Required | Description |
|------|----------|-------------|
| Label | Yes | Display name |
| Fieldtype | Yes | Data type (Link, Data, Date, Currency, Int, Float, Percent, Check) |
| Options | Only for Link/Currency | DocType name for Link; currency field for Currency |
| Width | Yes | Column width in pixels |

### Examples by Fieldtype

```sql
-- Link column
name as "Invoice:Link/Sales Invoice:200"

-- Currency column
grand_total as "Total:Currency:120"

-- Date column
posting_date as "Date:Date:100"

-- Integer column
qty as "Quantity:Int:80"

-- Float column
rate as "Rate:Float:100"

-- Percent column
per_billed as "Billed %:Percent:80"

-- Plain data
customer_name as "Customer Name:Data:200"

-- Check (boolean)
is_return as "Is Return:Check:60"
```

## Filters in Query Reports

### SQL Placeholder Filters

Use `%(filter_name)s` placeholders in SQL:

```sql
SELECT
  name as "Work Order:Link/Work Order:200",
  production_item as "Item:Link/Item:150",
  qty as "Qty:Int:80"
FROM
  `tabWork Order`
WHERE
  docstatus = 1
  AND company = %(company)s
  AND production_item LIKE %(item)s
ORDER BY creation DESC
```

### Filter Definition in JS File

Create a `.js` file alongside the Report document (for Standard reports):

```javascript
frappe.query_reports["My Query Report"] = {
    filters: [
        {
            fieldname: "company",
            label: __("Company"),
            fieldtype: "Link",
            options: "Company",
            default: frappe.defaults.get_user_default("company"),
            reqd: 1
        },
        {
            fieldname: "item",
            label: __("Item"),
            fieldtype: "Link",
            options: "Item"
        }
    ]
};
```

### Filter with Wildcard Default

```javascript
{
    fieldname: "item",
    label: __("Item"),
    fieldtype: "Link",
    options: "Item",
    default: "%"  // matches all when no selection
}
```

## Query Report with Python Script

For complex Query Reports that need Python logic, create a `.py` file:

```python
import frappe

def execute(filters=None):
    conditions = get_conditions(filters)
    columns = get_columns()
    data = frappe.db.sql("""
        SELECT
            so.name, so.customer, so.grand_total
        FROM
            `tabSales Order` so
        WHERE
            so.docstatus = 1
            {conditions}
        ORDER BY so.creation DESC
    """.format(conditions=conditions), filters, as_dict=True)

    return columns, data

def get_columns():
    return [
        {
            "fieldname": "name",
            "label": "Sales Order",
            "fieldtype": "Link",
            "options": "Sales Order",
            "width": 200
        },
        {
            "fieldname": "customer",
            "label": "Customer",
            "fieldtype": "Link",
            "options": "Customer",
            "width": 180
        },
        {
            "fieldname": "grand_total",
            "label": "Grand Total",
            "fieldtype": "Currency",
            "width": 120
        }
    ]

def get_conditions(filters):
    conditions = ""
    if filters.get("company"):
        conditions += " AND so.company = %(company)s"
    if filters.get("customer"):
        conditions += " AND so.customer = %(customer)s"
    return conditions
```

## Column Definition in Report Document (v13+)

Since Frappe v13, you can define columns directly in the Report document UI:
- Add rows to the Columns table
- Set Label, Fieldtype, Width, Options per column
- This replaces the need for legacy string format in SQL aliases

When using this approach, your SQL `SELECT` column names MUST match the fieldnames configured in the Columns table.

## Permissions

- Query Reports inherit permissions from the **Reference DocType**
- Users who can read the Reference DocType can run the report
- System Manager role is required to CREATE Query Reports
- ALWAYS set a Reference DocType — reports without one are only visible to Administrators

## Critical Rules

- **ALWAYS** use parameterized queries with `%(filter)s` — NEVER concatenate user input into SQL strings
- **ALWAYS** filter by `docstatus` when querying submitted documents — omitting it returns Draft documents
- **NEVER** use `SELECT *` — ALWAYS specify exact columns needed
- **ALWAYS** prefix table names with `tab` in backticks: `` `tabSales Order` ``
- **NEVER** forget the backticks around table names with spaces

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-reports/references/query-report.md*
