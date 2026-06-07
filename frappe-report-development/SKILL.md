---
name: frappe-report-development
description: >
  Build Frappe reports — Query Reports (SQL), Script Reports (Python), filters, columns, charts,
  formatters, and report permissions. Use when creating analytics/listing reports, dashboards, or
  exportable views in Frappe/ERPNext. Trigger on: "query report", "script report", "Frappe report",
  "report filters", "report chart", "columns formatter", "report builder".
---

# Frappe Report Development

Three report types: **Report Builder** (no code, UI), **Query Report** (one SQL), **Script Report**
(Python, full control). Use Query for simple tabular, Script for computed/multi-step.

## Query Report

Create a `Report` (type = Query Report), put SQL in the query box. Filters are referenced as `%(key)s`:

```sql
SELECT
    so.name        AS "Order:Link/Sales Order:160",
    so.customer    AS "Customer:Link/Customer:200",
    so.grand_total AS "Total:Currency:120"
FROM `tabSales Order` so
WHERE so.docstatus = 1
  AND so.company = %(company)s
  AND so.transaction_date BETWEEN %(from_date)s AND %(to_date)s
ORDER BY so.transaction_date
```
Column header syntax: `Label:Fieldtype/Options:Width`. Filters injected as named `%(...)s` params — safe.

## Script Report

```python
# myapp/myapp/report/order_summary/order_summary.py
import frappe

def execute(filters=None):
    filters = filters or {}
    columns = [
        {"label": "Customer", "fieldname": "customer", "fieldtype": "Link", "options": "Customer", "width": 200},
        {"label": "Total",    "fieldname": "total",    "fieldtype": "Currency", "width": 120},
    ]
    rows = frappe.db.sql("""
        SELECT customer, SUM(grand_total) AS total
        FROM `tabSales Order`
        WHERE docstatus = 1 AND company = %(company)s
        GROUP BY customer
    """, filters, as_dict=True)

    chart = {
        "data": {"labels": [r.customer for r in rows],
                 "datasets": [{"values": [r.total for r in rows]}]},
        "type": "bar",
    }
    return columns, rows, None, chart      # columns, data, message, chart, report_summary
```

## Filters (report .js)

```javascript
frappe.query_reports["Order Summary"] = {
    filters: [
        { fieldname: "company",   label: "Company",   fieldtype: "Link", options: "Company", reqd: 1 },
        { fieldname: "from_date", label: "From",       fieldtype: "Date", default: frappe.datetime.month_start() },
        { fieldname: "to_date",   label: "To",         fieldtype: "Date", default: frappe.datetime.get_today() },
    ],
    formatter(value, row, column, data, default_formatter) {
        value = default_formatter(value, row, column, data);
        if (column.fieldname === "total" && data.total > 100000)
            value = `<span style="color:green">${value}</span>`;
        return value;
    },
};
```

## Rules

- Query Reports: only `%(name)s` params — never string-concat filter values (injection).
- Set report **roles** so only intended users can run/export it.
- Heavy aggregations → add indexes, or precompute into a summary DocType refreshed by a scheduler job.
- Return a `chart` dict for a built-in visualization; `report_summary` for top KPI cards.
- For huge exports, prefer a background job writing a file over a synchronous report run.
