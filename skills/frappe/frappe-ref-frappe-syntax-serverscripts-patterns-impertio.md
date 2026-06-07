---
title: Frappe Ref — Frappe Syntax: Serverscripts: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'serverscripts', 'reference', 'patterns']
source: Impertio
---

# Server Script Common Patterns

Reusable patterns for the most frequent Server Script use cases. Every example
uses ONLY the pre-loaded sandbox namespace — no import statements.

---

## Pattern 1: Field Validation with Custom Error

```python
# Before Save — validate business rules
errors = []

if not doc.customer:
    errors.append("Customer is required")

if frappe.utils.flt(doc.grand_total) <= 0:
    errors.append("Grand total must be greater than zero")

if doc.delivery_date and doc.delivery_date < frappe.utils.today():
    errors.append("Delivery date cannot be in the past")

if errors:
    frappe.throw("<br>".join(errors), title="Validation Errors")
```

---

## Pattern 2: Auto-Fill from Master Data

```python
# Before Save — fetch related data from linked documents
if doc.customer:
    customer_data = frappe.db.get_value("Customer", doc.customer,
        ["customer_name", "territory", "customer_group", "default_currency"],
        as_dict=True
    )
    if customer_data:
        if not doc.customer_name:
            doc.customer_name = customer_data.customer_name
        if not doc.territory:
            doc.territory = customer_data.territory
        if not doc.customer_group:
            doc.customer_group = customer_data.customer_group
```

---

## Pattern 3: Child Table Aggregation

```python
# Before Save — recalculate totals from child items
doc.total_qty = 0
doc.total_amount = 0

for item in doc.items:
    qty = frappe.utils.flt(item.qty)
    rate = frappe.utils.flt(item.rate)
    item.amount = qty * rate
    doc.total_qty += qty
    doc.total_amount += item.amount

doc.grand_total = doc.total_amount - frappe.utils.flt(doc.discount_amount)
```

---

## Pattern 4: Conditional Logic Based on Roles

```python
# Before Save — restrict actions by role
roles = frappe.get_roles()

if doc.discount_percentage > 20 and "Sales Manager" not in roles:
    frappe.throw("Only Sales Managers can give discounts above 20%")

if doc.is_priority and "System Manager" not in roles:
    frappe.throw("Only System Managers can set priority flag")
```

---

## Pattern 5: Create Downstream Document

```python
# After Submit — create a follow-up document
if doc.requires_delivery:
    delivery = frappe.new_doc("Delivery Note")
    delivery.customer = doc.customer
    delivery.company = doc.company

    for item in doc.items:
        delivery.append("items", {
            "item_code": item.item_code,
            "qty": item.qty,
            "rate": item.rate,
            "against_sales_order": doc.name,
            "so_detail": item.name
        })

    delivery.insert(ignore_permissions=True)
    frappe.msgprint(f"Delivery Note {delivery.name} created", indicator="green")
```

---

## Pattern 6: Duplicate Detection

```python
# Before Insert — prevent duplicate entries
existing = frappe.db.exists("Sales Order", {
    "customer": doc.customer,
    "po_no": doc.po_no,
    "docstatus": ["<", 2]
})

if existing:
    frappe.throw(
        f"A Sales Order ({existing}) already exists for this customer "
        f"with PO number {doc.po_no}",
        title="Duplicate Detected"
    )
```

---

## Pattern 7: Batch Update in Scheduler

```python
# Scheduler Event — process records in batches with commit
batch_size = 100
offset = 0
total_processed = 0

while True:
    records = frappe.get_all("Task",
        filters={"status": "Open", "exp_end_date": ["<", frappe.utils.today()]},
        fields=["name"],
        limit=batch_size,
        start=offset
    )

    if not records:
        break

    for record in records:
        frappe.db.set_value("Task", record.name, "status", "Overdue")
        total_processed += 1

    frappe.db.commit()  # Commit per batch to avoid long transactions
    offset += batch_size

if total_processed > 0:
    frappe.log_error(
        f"Marked {total_processed} tasks as Overdue",
        "Task Overdue Scheduler"
    )
```

---

## Pattern 8: API Endpoint with Pagination

```python
# API Script — paginated list endpoint
page = frappe.utils.cint(frappe.form_dict.get("page", 1))
page_size = frappe.utils.cint(frappe.form_dict.get("page_size", 20))

if page_size > 100:
    page_size = 100  # ALWAYS cap page size

start = (page - 1) * page_size

total = frappe.db.count("Sales Order", filters={"docstatus": 1})

orders = frappe.get_all("Sales Order",
    filters={"docstatus": 1},
    fields=["name", "customer", "grand_total", "status"],
    order_by="creation desc",
    limit=page_size,
    start=start
)

frappe.response["message"] = {
    "data": orders,
    "page": page,
    "page_size": page_size,
    "total": total,
    "total_pages": -(-total // page_size)  # Ceiling division
}
```

---

## Pattern 9: Status Transition Guard

```python
# Before Save — enforce valid status transitions
VALID_TRANSITIONS = {
    "Draft": ["Open", "Cancelled"],
    "Open": ["In Progress", "On Hold", "Cancelled"],
    "In Progress": ["Completed", "On Hold"],
    "On Hold": ["Open", "Cancelled"],
    "Completed": [],
    "Cancelled": []
}

if doc.status and doc.get("_doc_before_save"):
    old_status = doc._doc_before_save.status
    if old_status and doc.status != old_status:
        allowed = VALID_TRANSITIONS.get(old_status, [])
        if doc.status not in allowed:
            frappe.throw(
                f"Cannot change status from '{old_status}' to '{doc.status}'. "
                f"Allowed: {', '.join(allowed) or 'none'}",
                title="Invalid Status Transition"
            )
```

---

## Pattern 10: External API Sync

```python
# Scheduler Event — sync data from external API
api_key = frappe.db.get_single_value("Integration Settings", "api_key")
if not api_key:
    frappe.log_error("API key not configured", "Sync Error")
    return

try:
    response = frappe.make_get_request(
        "https://api.example.com/products",
        headers={"Authorization": f"Bearer {api_key}"}
    )
except Exception:
    frappe.log_error(frappe.get_traceback(), "External API Error")
    return

products = response.get("data", [])
synced = 0

for product in products[:100]:  # ALWAYS limit batch size
    if not frappe.db.exists("Item", product.get("sku")):
        item = frappe.new_doc("Item")
        item.item_code = product.get("sku")
        item.item_name = product.get("name")
        item.item_group = "Products"
        try:
            item.insert(ignore_permissions=True)
            synced += 1
        except Exception:
            frappe.log_error(
                f"Failed to sync: {product.get('sku')}",
                "Sync Error"
            )

frappe.db.commit()

if synced > 0:
    frappe.log_error(f"Synced {synced} new products", "External Sync")
```

---

## Pattern 11: Calling Another Server Script as Library

```python
# v13+ — reuse logic across scripts using run_script()

# Library Server Script named "Calculate Tax":
# tax_rate = frappe.db.get_value("Tax Rule", {"region": kwargs.get("region")}, "rate")
# frappe.flags.result = frappe.utils.flt(kwargs.get("amount")) * frappe.utils.flt(tax_rate) / 100

# Calling script:
tax = run_script("Calculate Tax", region=doc.territory, amount=doc.net_total)
doc.tax_amount = frappe.flags.result
```

---

## Pattern 12: Permission Query with Multiple Conditions

```python
# Permission Query — combine role, territory, and company filtering
roles = frappe.get_roles(user)

if "System Manager" in roles:
    conditions = ""
else:
    clauses = []

    # Owner condition for basic users
    if "Sales User" in roles and "Sales Manager" not in roles:
        clauses.append(f"`tabSales Order`.owner = {frappe.db.escape(user)}")

    # Company filter
    companies = frappe.get_all("User Permission",
        filters={"user": user, "allow": "Company"},
        pluck="for_value"
    )
    if companies:
        escaped = ", ".join(frappe.db.escape(c) for c in companies)
        clauses.append(f"`tabSales Order`.company IN ({escaped})")

    conditions = " AND ".join(clauses) if clauses else ""
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-serverscripts/references/patterns.md*
