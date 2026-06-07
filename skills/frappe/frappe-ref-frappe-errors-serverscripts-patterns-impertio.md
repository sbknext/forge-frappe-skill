---
title: Frappe Ref — Frappe Errors: Serverscripts: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'serverscripts', 'reference', 'patterns']
source: Impertio
---

# Server Script Error Handling Patterns

Reusable patterns for defensive error handling in Frappe Server Scripts, organized by script type.

---

## Document Event Patterns

### Pattern 1: Comprehensive Validation with Error Collection

```python
# Type: Document Event, Event: Before Save, DocType: Sales Order

errors = []
warnings = []

# Required fields
if not doc.customer:
    errors.append("Customer is required")
if not doc.delivery_date:
    errors.append("Delivery Date is required")
elif str(doc.delivery_date) < frappe.utils.today():
    warnings.append("Delivery Date is in the past")

# Customer validation
if doc.customer:
    customer = frappe.db.get_value("Customer", doc.customer,
        ["disabled", "credit_limit"], as_dict=True)
    if not customer:
        errors.append(f"Customer '{doc.customer}' not found")
    elif customer.disabled:
        errors.append(f"Customer '{doc.customer}' is disabled")
    elif customer.credit_limit and doc.grand_total > customer.credit_limit:
        warnings.append(f"Total ({doc.grand_total}) exceeds credit limit ({customer.credit_limit})")

# Child table validation
if not doc.items:
    errors.append("At least one item is required")
else:
    for idx, item in enumerate(doc.items, 1):
        if not item.item_code:
            errors.append(f"Row {idx}: Item Code is required")
        if (item.qty or 0) <= 0:
            errors.append(f"Row {idx}: Quantity must be positive")

# Show warnings (non-blocking)
if warnings:
    frappe.msgprint("<br>".join(warnings), title="Warnings", indicator="orange")

# Throw errors (blocking)
if errors:
    frappe.throw("<br>".join(errors), title="Please fix these errors")
```

### Pattern 2: Safe Database Lookup with Fallback

```python
# Type: Document Event, Event: Before Save

# ALWAYS check existence before get_doc (no try/except in sandbox)
if doc.customer:
    if not frappe.db.exists("Customer", doc.customer):
        frappe.throw(f"Customer '{doc.customer}' not found")

    # Safe multi-field lookup with defaults
    data = frappe.db.get_value("Customer", doc.customer,
        ["territory", "customer_group", "credit_limit"], as_dict=True)

    if data:
        doc.territory = data.get("territory") or "Default"
        doc.customer_group = data.get("customer_group") or ""
    else:
        doc.territory = "Default"
```

### Pattern 3: Cross-Document Validation

```python
# Type: Document Event, Event: Before Submit, DocType: Sales Invoice

# Check for already-invoiced items
for item in (doc.items or []):
    if item.sales_order and item.so_detail:
        existing = frappe.db.get_value("Sales Invoice Item", {
            "sales_order": item.sales_order,
            "so_detail": item.so_detail,
            "docstatus": 1,
            "parent": ["!=", doc.name]
        }, "parent")

        if existing:
            frappe.throw(
                f"Row {item.idx}: SO item {item.so_detail} already invoiced in {existing}"
            )
```

---

## API Script Patterns

### Pattern 4: Full API with Input Validation

```python
# Type: API, Method: create_order

# Extract parameters
customer = frappe.form_dict.get("customer")
items = frappe.form_dict.get("items")

# Validate required params
if not customer:
    frappe.throw("'customer' is required", exc=frappe.ValidationError)
if not items:
    frappe.throw("'items' is required", exc=frappe.ValidationError)

# Parse JSON if needed
if isinstance(items, str):
    items = frappe.parse_json(items)

# Validate entities exist
if not frappe.db.exists("Customer", customer):
    frappe.throw(f"Customer '{customer}' not found", exc=frappe.DoesNotExistError)

# Check permissions
if not frappe.has_permission("Sales Order", "create"):
    frappe.throw("No permission to create orders", exc=frappe.PermissionError)

# Validate items
for idx, item in enumerate(items, 1):
    code = item.get("item_code") if isinstance(item, dict) else None
    if not code:
        frappe.throw(f"Item {idx}: 'item_code' required", exc=frappe.ValidationError)
    if not frappe.db.exists("Item", code):
        frappe.throw(f"Item '{code}' not found", exc=frappe.DoesNotExistError)

# Create document
so = frappe.get_doc({
    "doctype": "Sales Order",
    "customer": customer,
    "items": [{"item_code": i.get("item_code"), "qty": i.get("qty", 1)} for i in items]
})
so.insert()

# REQUIRED: Set response
frappe.response["message"] = {"success": True, "name": so.name}
```

### Pattern 5: API with Safe Error Responses

```python
# Type: API, Method: get_report_data

report_type = frappe.form_dict.get("type")
date_from = frappe.form_dict.get("from")
date_to = frappe.form_dict.get("to")

# Validate parameters
if not report_type:
    frappe.throw("'type' parameter is required", exc=frappe.ValidationError)

valid_types = ["sales", "purchase", "stock"]
if report_type not in valid_types:
    frappe.throw(
        f"Invalid type '{report_type}'. Must be: {', '.join(valid_types)}",
        exc=frappe.ValidationError
    )

# Safe date parsing
if date_from and not frappe.utils.validate_date_format(date_from):
    frappe.throw("Invalid 'from' date format. Use YYYY-MM-DD", exc=frappe.ValidationError)

# Build query safely
filters = {"docstatus": 1}
if date_from:
    filters["posting_date"] = [">=", date_from]
if date_to:
    filters["posting_date"] = ["<=", date_to]

data = frappe.get_all("Sales Invoice",
    filters=filters,
    fields=["name", "customer", "grand_total", "posting_date"],
    limit=1000
)

frappe.response["message"] = {"data": data, "count": len(data)}
```

---

## Scheduler Patterns

### Pattern 6: Batch Processing with Error Isolation

```python
# Type: Scheduler Event, Cron: 0 8 * * *

BATCH_SIZE = 50
MAX_ERRORS = 20

stats = {"processed": 0, "errors": []}

records = frappe.get_all("Sales Invoice",
    filters={"status": "Unpaid", "docstatus": 1, "due_date": ["<", frappe.utils.today()]},
    fields=["name", "customer", "owner", "outstanding_amount"],
    limit=500
)

for i in range(0, len(records), BATCH_SIZE):
    if len(stats["errors"]) >= MAX_ERRORS:
        stats["errors"].append("--- STOPPED: Too many errors ---")
        break

    batch = records[i:i + BATCH_SIZE]
    for rec in batch:
        result = process_record(rec)
        if result.get("success"):
            stats["processed"] += 1
        else:
            stats["errors"].append(f"{rec.name}: {result.get('error')}")

    frappe.db.commit()

# Log summary
summary = f"Processed: {stats['processed']}, Errors: {len(stats['errors'])}"
if stats["errors"]:
    summary += "\n" + "\n".join(stats["errors"][:20])
frappe.log_error(summary, "Scheduler Summary")
frappe.db.commit()


def process_record(rec):
    if not frappe.db.exists("Customer", rec.customer):
        return {"success": False, "error": "Customer not found"}

    # Process logic...
    frappe.db.set_value("Sales Invoice", rec.name, "reminder_sent", 1)
    return {"success": True}
```

### Pattern 7: Idempotent Scheduler with Lock

```python
# Type: Scheduler Event, Cron: */15 * * * *

LOCK_KEY = "inventory_sync_lock"
LOCK_TIMEOUT = 600

# Check if already running
lock_time = frappe.cache().get_value(LOCK_KEY)
if lock_time:
    elapsed = frappe.utils.time_diff_in_seconds(frappe.utils.now(), lock_time)
    if elapsed < LOCK_TIMEOUT:
        return  # Another instance running

# Set lock
frappe.cache().set_value(LOCK_KEY, frappe.utils.now())

items = frappe.get_all("Item",
    filters={"sync_status": "Pending", "disabled": 0},
    fields=["name"],
    limit=100
)

for item in items:
    frappe.db.set_value("Item", item.name, "sync_status", "Processing")
    frappe.db.commit()

    # Process...
    frappe.db.set_value("Item", item.name, {
        "sync_status": "Synced",
        "last_synced": frappe.utils.now()
    })
    frappe.db.commit()

# Release lock
frappe.cache().delete_value(LOCK_KEY)
frappe.db.commit()
```

---

## Permission Query Pattern

### Pattern 8: Safe Permission Query

```python
# Type: Permission Query, DocType: Project

user_roles = frappe.get_roles(user) or []

# Admin — full access
if "System Manager" in user_roles:
    conditions = ""

# Manager — department access
elif "Projects Manager" in user_roles:
    dept = frappe.db.get_value("User", user, "department")
    if dept:
        conditions = f"`tabProject`.department = {frappe.db.escape(dept)}"
    else:
        frappe.log_error(f"Manager {user} has no department", "Permission Warning")
        conditions = f"`tabProject`.owner = {frappe.db.escape(user)}"

# User — own records only
elif "Projects User" in user_roles:
    conditions = f"`tabProject`.owner = {frappe.db.escape(user)}"

# No role — no access
else:
    conditions = "1=0"
```

---

## Quick Reference: Error Handling Cheat Sheet

```python
# Stop execution with user message
frappe.throw("Error message")
frappe.throw("Not found", exc=frappe.DoesNotExistError)  # 404
frappe.throw("No access", exc=frappe.PermissionError)     # 403

# Warning (continues execution)
frappe.msgprint("Warning text", indicator="orange")

# Log silently
frappe.log_error("Details", "Title")

# Safe database access
value = frappe.db.get_value("DocType", name, "field") or 0
exists = frappe.db.exists("DocType", name)
data = frappe.db.get_value("DocType", name, ["f1", "f2"], as_dict=True) or {}

# Scheduler requirements
frappe.db.commit()  # REQUIRED
limit=500           # ALWAYS limit queries
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-serverscripts/references/patterns.md*
