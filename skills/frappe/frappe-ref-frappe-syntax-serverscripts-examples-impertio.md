---
title: Frappe Ref — Frappe Syntax: Serverscripts: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'serverscripts', 'reference', 'examples']
source: Impertio
---

# Server Script Examples — All Script Types

Every example uses ONLY the pre-loaded sandbox namespace. No import statements.

---

## Document Event Examples

### 1. Field Validation (Before Save)

```python
# Config: DocType = Sales Invoice, Event = Before Save
if doc.grand_total < 0:
    frappe.throw("Grand total MUST NOT be negative")

if doc.discount_percentage and doc.discount_percentage > 50:
    frappe.throw("Discount cannot exceed 50%", title="Validation Error")
```

### 2. Auto-Calculate Fields (Before Save)

```python
# Config: DocType = Sales Order, Event = Before Save
doc.total_qty = sum(frappe.utils.flt(item.qty) for item in doc.items)
doc.total_weight = sum(
    frappe.utils.flt(item.qty) * frappe.utils.flt(item.weight_per_unit)
    for item in doc.items
)

if doc.grand_total > 10000:
    doc.priority = "High"
    doc.requires_approval = 1
```

### 3. Auto-Fill from Linked Document (Before Save)

```python
# Config: DocType = Sales Invoice, Event = Before Save
if doc.customer and not doc.customer_name:
    doc.customer_name = frappe.db.get_value(
        "Customer", doc.customer, "customer_name")

if doc.customer and not doc.territory:
    doc.territory = frappe.db.get_value(
        "Customer", doc.customer, "territory")
```

### 4. Create Related Document (After Insert)

```python
# Config: DocType = Sales Order, Event = After Insert
frappe.get_doc({
    "doctype": "ToDo",
    "allocated_to": doc.owner,
    "reference_type": "Sales Order",
    "reference_name": doc.name,
    "description": f"New order {doc.name} — follow up with customer",
    "date": frappe.utils.add_days(frappe.utils.today(), 1)
}).insert(ignore_permissions=True)
```

### 5. Pre-Submit Validation (Before Submit)

```python
# Config: DocType = Purchase Order, Event = Before Submit
if doc.grand_total > 50000:
    if not doc.budget_approval:
        frappe.throw("Budget approval required for orders over 50,000")
    if doc.approved_by == doc.owner:
        frappe.throw("Order MUST NOT be approved by its creator")
```

### 6. Post-Submit Side Effects (After Submit)

```python
# Config: DocType = Sales Invoice, Event = After Submit
total_invoices = frappe.db.count("Sales Invoice",
    filters={"customer": doc.customer, "docstatus": 1})
frappe.db.set_value("Customer", doc.customer,
    "total_invoices", total_invoices)

if doc.grand_total > 10000:
    frappe.sendmail(
        recipients=[doc.owner],
        subject=f"High-value invoice {doc.name}",
        message=f"Invoice {doc.name} for {doc.grand_total} has been submitted."
    )
```

### 7. Cancel Guard (Before Cancel)

```python
# Config: DocType = Sales Invoice, Event = Before Cancel
payments = frappe.get_all("Payment Entry Reference",
    filters={
        "reference_doctype": "Sales Invoice",
        "reference_name": doc.name,
        "docstatus": 1
    },
    fields=["parent"]
)
if payments:
    frappe.throw(
        f"Cannot cancel: {len(payments)} linked payment(s) exist. "
        "Cancel the payments first.",
        title="Cancellation Blocked"
    )
```

### 8. Set Default Values (Before Insert)

```python
# Config: DocType = Sales Order, Event = Before Insert
if not doc.delivery_date:
    doc.delivery_date = frappe.utils.add_days(frappe.utils.today(), 7)

if not doc.currency:
    doc.currency = frappe.db.get_single_value(
        "Global Defaults", "default_currency") or "USD"
```

### 9. Prevent Infinite Loop with Flags

```python
# Config: DocType = Sales Order, Event = After Save
# Updating a related doc that might trigger this script again
if not doc.flags.get("skip_sync"):
    linked = frappe.get_doc("Project", doc.project)
    linked.flags.skip_sync = True
    linked.total_orders = frappe.db.count("Sales Order",
        filters={"project": doc.project, "docstatus": 1})
    linked.save(ignore_permissions=True)
```

### 10. Child Table Validation (Before Save)

```python
# Config: DocType = Sales Order, Event = Before Save
seen_items = []
for item in doc.items:
    if item.item_code in seen_items:
        frappe.throw(f"Duplicate item {item.item_code} in row {item.idx}")
    seen_items.append(item.item_code)

    if frappe.utils.flt(item.qty) <= 0:
        frappe.throw(f"Quantity must be > 0 in row {item.idx}")

    if frappe.utils.flt(item.rate) <= 0:
        frappe.throw(f"Rate must be > 0 in row {item.idx}")
```

---

## API Examples

### 11. GET Endpoint with Permission Check

```python
# Config: Script Type = API, Method = get_customer_orders, Allow Guest = No
# Endpoint: GET /api/method/get_customer_orders?customer=CUST-001

customer = frappe.form_dict.get("customer")
if not customer:
    frappe.throw("Parameter 'customer' is required")

if not frappe.has_permission("Sales Order", "read"):
    frappe.throw("Access denied", frappe.PermissionError)

orders = frappe.get_all("Sales Order",
    filters={"customer": customer, "docstatus": 1},
    fields=["name", "transaction_date", "grand_total", "status"],
    order_by="transaction_date desc",
    limit=20
)

frappe.response["message"] = {
    "customer": customer,
    "orders": orders,
    "count": len(orders)
}
```

### 12. POST Endpoint with Input Validation

```python
# Config: Script Type = API, Method = update_order_status, Allow Guest = No
# Endpoint: POST /api/method/update_order_status

order_name = frappe.form_dict.get("order")
new_status = frappe.form_dict.get("status")

if not order_name or not new_status:
    frappe.throw("Parameters 'order' and 'status' are required")

valid_statuses = ["Open", "Completed", "On Hold"]
if new_status not in valid_statuses:
    frappe.throw(f"Invalid status. Valid: {', '.join(valid_statuses)}")

if not frappe.has_permission("Sales Order", "write", order_name):
    frappe.throw("No write permission", frappe.PermissionError)

frappe.db.set_value("Sales Order", order_name, "status", new_status)
frappe.response["message"] = {"success": True, "new_status": new_status}
```

### 13. Dashboard Data Endpoint

```python
# Config: Script Type = API, Method = get_sales_dashboard, Allow Guest = No

today = frappe.utils.today()
month_start = frappe.utils.get_first_day(today)

orders_today = frappe.db.count("Sales Order",
    filters={"transaction_date": today, "docstatus": 1})

month_revenue = frappe.db.sql("""
    SELECT COALESCE(SUM(grand_total), 0) as total
    FROM `tabSales Invoice`
    WHERE posting_date >= %(start)s AND docstatus = 1
""", {"start": month_start}, as_dict=True)

top_customers = frappe.get_all("Sales Invoice",
    filters={"posting_date": [">=", month_start], "docstatus": 1},
    fields=["customer", "sum(grand_total) as total"],
    group_by="customer",
    order_by="total desc",
    limit=5
)

frappe.response["message"] = {
    "orders_today": orders_today,
    "month_revenue": month_revenue[0].total if month_revenue else 0,
    "top_customers": top_customers
}
```

### 14. Public Guest Endpoint

```python
# Config: Script Type = API, Method = check_availability, Allow Guest = Yes
# NEVER expose sensitive data in guest endpoints

item_code = frappe.form_dict.get("item")
if not item_code:
    frappe.throw("Parameter 'item' is required")

item = frappe.db.get_value("Item", item_code,
    ["item_name", "stock_uom", "disabled"], as_dict=True)

if not item or item.disabled:
    frappe.response["message"] = {"available": False}
else:
    stock = frappe.db.get_value("Bin",
        {"item_code": item_code}, "sum(actual_qty)") or 0
    frappe.response["message"] = {
        "available": frappe.utils.flt(stock) > 0,
        "item_name": item.item_name,
        "uom": item.stock_uom
    }
```

### 15. External API Integration

```python
# Config: Script Type = API, Method = sync_external_data, Allow Guest = No

api_key = frappe.db.get_single_value("My Settings", "api_key")
if not api_key:
    frappe.throw("API key not configured")

response = frappe.make_get_request(
    "https://api.example.com/data",
    headers={"Authorization": f"Bearer {api_key}"}
)

frappe.response["message"] = {
    "synced": True,
    "records": len(response.get("data", []))
}
```

---

## Scheduler Event Examples

### 16. Daily Overdue Invoice Reminders

```python
# Config: Script Type = Scheduler Event, Cron = 0 9 * * *
today = frappe.utils.today()

overdue = frappe.get_all("Sales Invoice",
    filters={
        "status": "Unpaid",
        "due_date": ["<", today],
        "docstatus": 1
    },
    fields=["name", "customer", "grand_total", "due_date", "owner"],
    limit=200
)

for inv in overdue:
    days_overdue = frappe.utils.date_diff(today, inv.due_date)
    if not frappe.db.exists("ToDo", {
        "reference_type": "Sales Invoice",
        "reference_name": inv.name,
        "status": "Open"
    }):
        frappe.get_doc({
            "doctype": "ToDo",
            "allocated_to": inv.owner,
            "reference_type": "Sales Invoice",
            "reference_name": inv.name,
            "description": f"Invoice {inv.name} is {days_overdue} days overdue"
        }).insert(ignore_permissions=True)

frappe.db.commit()  # ALWAYS commit in scheduler scripts
```

### 17. Weekly Draft Cleanup

```python
# Config: Script Type = Scheduler Event, Cron = 0 2 * * 0
cutoff = frappe.utils.add_days(frappe.utils.today(), -30)

old_drafts = frappe.get_all("Sales Order",
    filters={"docstatus": 0, "modified": ["<", cutoff]},
    fields=["name"],
    limit=100
)

deleted = 0
for draft in old_drafts:
    try:
        frappe.delete_doc("Sales Order", draft.name, force=True)
        deleted += 1
    except Exception:
        frappe.log_error(
            f"Could not delete draft {draft.name}",
            "Cleanup Error"
        )

frappe.db.commit()

if deleted > 0:
    frappe.log_error(f"Deleted {deleted} old draft Sales Orders", "Weekly Cleanup")
```

### 18. Monthly Summary Report

```python
# Config: Script Type = Scheduler Event, Cron = 0 6 1 * *
prev_start = frappe.utils.add_months(
    frappe.utils.get_first_day(frappe.utils.today()), -1)
prev_end = frappe.utils.get_last_day(prev_start)

summary = frappe.db.sql("""
    SELECT
        COUNT(*) as count,
        COALESCE(SUM(grand_total), 0) as revenue,
        COUNT(DISTINCT customer) as customers
    FROM `tabSales Invoice`
    WHERE posting_date BETWEEN %(start)s AND %(end)s
    AND docstatus = 1
""", {"start": prev_start, "end": prev_end}, as_dict=True)[0]

frappe.log_error(
    f"Monthly Report ({prev_start} to {prev_end})\n"
    f"Invoices: {summary.count}\n"
    f"Revenue: {summary.revenue}\n"
    f"Customers: {summary.customers}",
    "Monthly Sales Report"
)
frappe.db.commit()
```

---

## Permission Query Examples

### 19. Role-Based Filtering

```python
# Config: Script Type = Permission Query, DocType = Sales Invoice
# Variables available: user, conditions

roles = frappe.get_roles(user)
if "System Manager" in roles or "Accounts Manager" in roles:
    conditions = ""
elif "Sales User" in roles:
    conditions = f"`tabSales Invoice`.owner = {frappe.db.escape(user)}"
else:
    conditions = "1=0"
```

### 20. Territory-Based Filtering

```python
# Config: Script Type = Permission Query, DocType = Customer

user_territory = frappe.db.get_value("User", user, "territory")

if "Sales Manager" in frappe.get_roles(user):
    conditions = ""
elif user_territory:
    conditions = f"`tabCustomer`.territory = {frappe.db.escape(user_territory)}"
else:
    conditions = "1=0"
```

### 21. Company-Based Filtering

```python
# Config: Script Type = Permission Query, DocType = Sales Order

allowed = frappe.get_all("User Permission",
    filters={"user": user, "allow": "Company"},
    pluck="for_value"
)

if not allowed:
    conditions = ""
elif len(allowed) == 1:
    conditions = f"`tabSales Order`.company = {frappe.db.escape(allowed[0])}"
else:
    escaped = ", ".join(frappe.db.escape(c) for c in allowed)
    conditions = f"`tabSales Order`.company IN ({escaped})"
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-serverscripts/references/examples.md*
