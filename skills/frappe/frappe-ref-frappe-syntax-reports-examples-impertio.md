---
title: Frappe Ref — Frappe Syntax: Reports: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'reports', 'reference', 'examples']
source: Impertio
---

# Working Report Examples

## Example 1: Simple Query Report (SQL Only)

### Report Configuration
- Report Type: Query Report
- Reference DocType: Sales Order

### SQL Query
```sql
SELECT
  `tabSales Order`.name as "Sales Order:Link/Sales Order:200",
  `tabSales Order`.customer as "Customer:Link/Customer:180",
  `tabSales Order`.transaction_date as "Date:Date:100",
  `tabSales Order`.grand_total as "Grand Total:Currency:120",
  `tabSales Order`.status as "Status:Data:100"
FROM
  `tabSales Order`
WHERE
  `tabSales Order`.docstatus = 1
  AND `tabSales Order`.company = %(company)s
ORDER BY
  `tabSales Order`.transaction_date DESC
LIMIT 500
```

---

## Example 2: Script Report with Chart and Summary

### File: `sales_analytics.py`

```python
import frappe
from frappe import _
from frappe.utils import flt, getdate, add_months


def execute(filters=None):
    columns = get_columns(filters)
    data = get_data(filters)
    chart = get_chart_data(data, filters)
    report_summary = get_report_summary(data, filters)

    return columns, data, None, chart, report_summary


def get_columns(filters):
    return [
        {
            "fieldname": "month",
            "label": _("Month"),
            "fieldtype": "Data",
            "width": 120
        },
        {
            "fieldname": "total_orders",
            "label": _("Total Orders"),
            "fieldtype": "Int",
            "width": 100
        },
        {
            "fieldname": "total_amount",
            "label": _("Total Amount"),
            "fieldtype": "Currency",
            "options": "Company:company:default_currency",
            "width": 150
        },
        {
            "fieldname": "avg_order_value",
            "label": _("Avg Order Value"),
            "fieldtype": "Currency",
            "options": "Company:company:default_currency",
            "width": 150
        }
    ]


def get_data(filters):
    data = frappe.db.sql("""
        SELECT
            DATE_FORMAT(transaction_date, '%%Y-%%m') as month,
            COUNT(name) as total_orders,
            SUM(grand_total) as total_amount,
            AVG(grand_total) as avg_order_value
        FROM
            `tabSales Order`
        WHERE
            docstatus = 1
            AND company = %(company)s
            AND transaction_date BETWEEN %(from_date)s AND %(to_date)s
        GROUP BY
            DATE_FORMAT(transaction_date, '%%Y-%%m')
        ORDER BY
            month
    """, filters, as_dict=True)

    return data


def get_chart_data(data, filters):
    if not data:
        return None

    labels = [row.month for row in data]
    amounts = [flt(row.total_amount) for row in data]
    orders = [row.total_orders for row in data]

    return {
        "data": {
            "labels": labels,
            "datasets": [
                {"name": _("Amount"), "values": amounts},
                {"name": _("Orders"), "values": orders}
            ]
        },
        "type": "bar",
        "fieldtype": "Currency",
        "colors": ["#5e64ff", "#ffa00a"]
    }


def get_report_summary(data, filters):
    if not data:
        return []

    total_amount = sum(flt(row.total_amount) for row in data)
    total_orders = sum(row.total_orders for row in data)
    avg_value = total_amount / total_orders if total_orders else 0

    return [
        {
            "value": total_amount,
            "label": _("Total Revenue"),
            "datatype": "Currency",
            "indicator": "Green"
        },
        {
            "value": total_orders,
            "label": _("Total Orders"),
            "datatype": "Int",
            "indicator": "Blue"
        },
        {
            "value": avg_value,
            "label": _("Average Order Value"),
            "datatype": "Currency",
            "indicator": "Blue"
        }
    ]
```

### File: `sales_analytics.js`

```javascript
frappe.query_reports["Sales Analytics"] = {
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
            fieldname: "from_date",
            label: __("From Date"),
            fieldtype: "Date",
            default: frappe.datetime.add_months(frappe.datetime.get_today(), -12),
            reqd: 1
        },
        {
            fieldname: "to_date",
            label: __("To Date"),
            fieldtype: "Date",
            default: frappe.datetime.get_today(),
            reqd: 1
        }
    ]
};
```

---

## Example 3: Script Report with Tree View

### File: `account_tree.py`

```python
import frappe
from frappe import _
from frappe.utils import flt


def execute(filters=None):
    columns = [
        {
            "fieldname": "account",
            "label": _("Account"),
            "fieldtype": "Link",
            "options": "Account",
            "width": 300
        },
        {
            "fieldname": "balance",
            "label": _("Balance"),
            "fieldtype": "Currency",
            "width": 150
        }
    ]

    data = get_data(filters)
    return columns, data


def get_data(filters):
    accounts = frappe.db.get_all("Account",
        filters={"company": filters.get("company")},
        fields=["name", "parent_account", "is_group", "lft", "rgt"],
        order_by="lft"
    )

    data = []
    for account in accounts:
        balance = get_balance(account.name, filters)
        indent = get_indent(account, accounts)
        data.append({
            "account": account.name,
            "balance": balance,
            "indent": indent,          # Required for tree mode
            "parent_account": account.parent_account,
            "bold": account.is_group   # Bold group accounts
        })

    return data
```

### File: `account_tree.js`

```javascript
frappe.query_reports["Account Tree"] = {
    tree: true,
    initial_depth: 2,
    parent_field: "parent_account",
    filters: [
        {
            fieldname: "company",
            label: __("Company"),
            fieldtype: "Link",
            options: "Company",
            default: frappe.defaults.get_user_default("company"),
            reqd: 1
        }
    ],
    formatter: function(value, row, column, data, default_formatter) {
        value = default_formatter(value, row, column, data);
        if (data && data.bold) {
            value = "<b>" + value + "</b>";
        }
        return value;
    }
};
```

---

## Example 4: Grouped Report with DateRange Filter

### File: `customer_summary.py`

```python
import frappe
from frappe import _
from frappe.utils import flt


def execute(filters=None):
    # DateRange filter comes as [from_date, to_date]
    if filters.get("date_range"):
        filters["from_date"] = filters["date_range"][0]
        filters["to_date"] = filters["date_range"][1]

    columns = [
        {"fieldname": "customer", "label": _("Customer"), "fieldtype": "Link",
         "options": "Customer", "width": 200},
        {"fieldname": "total_orders", "label": _("Orders"), "fieldtype": "Int",
         "width": 80},
        {"fieldname": "total_qty", "label": _("Total Qty"), "fieldtype": "Float",
         "width": 100},
        {"fieldname": "total_amount", "label": _("Total Amount"),
         "fieldtype": "Currency", "width": 150}
    ]

    data = frappe.db.sql("""
        SELECT
            customer,
            COUNT(name) as total_orders,
            SUM(total_qty) as total_qty,
            SUM(grand_total) as total_amount
        FROM `tabSales Order`
        WHERE
            docstatus = 1
            AND transaction_date BETWEEN %(from_date)s AND %(to_date)s
        GROUP BY customer
        ORDER BY total_amount DESC
    """, filters, as_dict=True)

    return columns, data
```

### File: `customer_summary.js`

```javascript
frappe.query_reports["Customer Summary"] = {
    filters: [
        {
            fieldname: "date_range",
            label: __("Date Range"),
            fieldtype: "DateRange",
            default: [
                frappe.datetime.add_months(frappe.datetime.get_today(), -3),
                frappe.datetime.get_today()
            ],
            reqd: 1
        }
    ]
};
```

---

## Example 5: Report with Dynamic Link and Custom Buttons

### File: `party_ledger.py`

```python
import frappe
from frappe import _


def execute(filters=None):
    columns = [
        {"fieldname": "posting_date", "label": _("Date"), "fieldtype": "Date",
         "width": 100},
        {"fieldname": "voucher_type", "label": _("Voucher Type"),
         "fieldtype": "Data", "width": 120},
        {"fieldname": "voucher_no", "label": _("Voucher No"),
         "fieldtype": "Dynamic Link", "options": "voucher_type", "width": 180},
        {"fieldname": "debit", "label": _("Debit"), "fieldtype": "Currency",
         "width": 120},
        {"fieldname": "credit", "label": _("Credit"), "fieldtype": "Currency",
         "width": 120},
        {"fieldname": "balance", "label": _("Balance"), "fieldtype": "Currency",
         "width": 120}
    ]

    data = get_gl_entries(filters)

    # Calculate running balance
    balance = 0
    for row in data:
        balance += flt(row.debit) - flt(row.credit)
        row["balance"] = balance

    return columns, data
```

### File: `party_ledger.js`

```javascript
frappe.query_reports["Party Ledger"] = {
    filters: [
        {
            fieldname: "party_type",
            label: __("Party Type"),
            fieldtype: "Link",
            options: "DocType",
            default: "Customer",
            reqd: 1,
            get_query: function() {
                return {
                    filters: { name: ["in", ["Customer", "Supplier", "Employee"]] }
                };
            }
        },
        {
            fieldname: "party",
            label: __("Party"),
            fieldtype: "Dynamic Link",
            options: "party_type",
            reqd: 1
        },
        {
            fieldname: "from_date",
            label: __("From Date"),
            fieldtype: "Date",
            default: frappe.datetime.add_months(frappe.datetime.get_today(), -12),
            reqd: 1
        },
        {
            fieldname: "to_date",
            label: __("To Date"),
            fieldtype: "Date",
            default: frappe.datetime.get_today(),
            reqd: 1
        }
    ],
    onload: function(report) {
        report.page.add_inner_button(__("Print Statement"), function() {
            let filters = report.get_filter_values();
            frappe.call({
                method: "myapp.api.get_print_statement",
                args: { filters: filters }
            });
        });
    }
};
```

---

## Example 6: Custom Report (No App Required)

For System Managers who need a quick report without deploying an app:

1. Create new Report, set Type = "Script Report"
2. Leave "Is Standard" = "No" (this makes it a Custom Report)
3. Write Python directly in the Script field:

```python
# This goes in the Report document's Script field
result = frappe.db.get_all("Sales Invoice",
    filters={
        "docstatus": 1,
        "posting_date": ["between", [filters.from_date, filters.to_date]]
    },
    fields=["customer", "posting_date", "grand_total", "status"],
    order_by="posting_date desc",
    limit_page_length=0
)

# For custom reports, just set columns and data directly
columns = [
    {"fieldname": "customer", "label": "Customer", "fieldtype": "Link",
     "options": "Customer", "width": 200},
    {"fieldname": "posting_date", "label": "Date", "fieldtype": "Date",
     "width": 100},
    {"fieldname": "grand_total", "label": "Total", "fieldtype": "Currency",
     "width": 120},
    {"fieldname": "status", "label": "Status", "fieldtype": "Data",
     "width": 100}
]
data = result
```

4. Add filters in the Filters table of the Report document
5. No `.py` or `.js` files needed — everything is in the database

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-reports/references/examples.md*
