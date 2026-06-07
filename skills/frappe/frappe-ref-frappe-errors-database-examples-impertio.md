---
title: Frappe Ref — Frappe Errors: Database: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'database', 'reference', 'examples']
source: Impertio
---

# Examples — Database Error Handling

Complete working examples of error handling for Frappe/ERPNext database operations.

---

## Example 1: Customer API with Full Error Handling

```python
# myapp/api/customer.py
import frappe
from frappe import _

@frappe.whitelist()
def get_customer(customer_name):
    """Get customer with proper error handling."""
    if not customer_name:
        frappe.throw(_("Customer name is required"))

    data = frappe.db.get_value(
        "Customer", customer_name,
        ["name", "customer_name", "customer_type", "credit_limit", "disabled"],
        as_dict=True
    )

    if not data:
        frappe.throw(_("Customer '{0}' not found").format(customer_name),
                     exc=frappe.DoesNotExistError)

    if data.disabled:
        frappe.throw(_("Customer '{0}' is disabled").format(data.customer_name))

    return data


@frappe.whitelist()
def create_customer(customer_name, customer_type="Company"):
    """Create customer with duplicate and validation handling."""
    if not customer_name:
        frappe.throw(_("Customer name is required"))

    if frappe.db.exists("Customer", {"customer_name": customer_name}):
        frappe.throw(_("Customer '{0}' already exists").format(customer_name),
                     exc=frappe.DuplicateEntryError)

    try:
        doc = frappe.get_doc({
            "doctype": "Customer",
            "customer_name": customer_name,
            "customer_type": customer_type
        })
        doc.insert()
        return {"success": True, "name": doc.name}

    except frappe.DuplicateEntryError:
        # Race condition — another user created it
        existing = frappe.db.get_value("Customer", {"customer_name": customer_name}, "name")
        frappe.throw(_("Customer was just created by another user"))

    except frappe.MandatoryError as e:
        frappe.throw(_("Missing required field: {0}").format(e))

    except frappe.ValidationError as e:
        frappe.throw(str(e))


@frappe.whitelist()
def update_customer(customer_name, updates):
    """Update customer with concurrent edit handling."""
    if not customer_name:
        frappe.throw(_("Customer name is required"))

    if not frappe.db.exists("Customer", customer_name):
        frappe.throw(_("Customer not found"), exc=frappe.DoesNotExistError)

    try:
        doc = frappe.get_doc("Customer", customer_name)
        if isinstance(updates, str):
            updates = frappe.parse_json(updates)
        doc.update(updates)
        doc.save()
        return {"success": True, "name": doc.name}

    except frappe.TimestampMismatchError:
        frappe.throw(_("Modified by another user. Please refresh."))
    except frappe.CharacterLengthExceededError:
        frappe.throw(_("One or more fields exceed maximum length"))
    except frappe.ValidationError as e:
        frappe.throw(str(e))


@frappe.whitelist()
def delete_customer(customer_name):
    """Delete customer with link checking."""
    if not customer_name:
        frappe.throw(_("Customer name is required"))

    if not frappe.db.exists("Customer", customer_name):
        return {"success": True, "message": "Already deleted"}

    # Pre-check for common linked documents
    linked = frappe.db.count("Sales Invoice", {"customer": customer_name, "docstatus": 1})
    if linked:
        frappe.throw(_("Cannot delete — {0} submitted invoice(s) exist").format(linked))

    try:
        frappe.delete_doc("Customer", customer_name)
        return {"success": True}
    except frappe.LinkExistsError:
        frappe.throw(_("Cannot delete — linked documents exist"))
```

---

## Example 2: Data Import with Error Tracking

```python
# myapp/imports/item_import.py
import frappe
from frappe import _

@frappe.whitelist()
def import_items(items_json):
    """Import items with per-record error tracking."""
    items = frappe.parse_json(items_json)
    if not items:
        frappe.throw(_("No items provided"))

    results = {
        "total": len(items), "created": 0, "updated": 0,
        "failed": 0, "errors": []
    }

    for idx, item_data in enumerate(items, 1):
        code = item_data.get("item_code")
        if not code:
            results["failed"] += 1
            results["errors"].append({"row": idx, "error": "Item code required"})
            continue

        try:
            if frappe.db.exists("Item", code):
                doc = frappe.get_doc("Item", code)
                doc.update(item_data)
                doc.save()
                results["updated"] += 1
            else:
                doc = frappe.get_doc({"doctype": "Item", **item_data})
                doc.insert()
                results["created"] += 1

        except frappe.DuplicateEntryError:
            results["errors"].append({"row": idx, "item": code, "error": "Duplicate"})
        except frappe.MandatoryError as e:
            results["failed"] += 1
            results["errors"].append({"row": idx, "item": code, "error": f"Missing: {e}"})
        except frappe.CharacterLengthExceededError:
            results["failed"] += 1
            results["errors"].append({"row": idx, "item": code, "error": "Field too long"})
        except frappe.ValidationError as e:
            results["failed"] += 1
            results["errors"].append({"row": idx, "item": code, "error": str(e)[:200]})
        except Exception:
            results["failed"] += 1
            frappe.log_error(frappe.get_traceback(), f"Import error: {code}")
            results["errors"].append({"row": idx, "item": code, "error": "Unexpected"})

        if idx % 50 == 0:
            frappe.db.commit()

    frappe.db.commit()
    return results
```

---

## Example 3: Report Query with Error Handling

```python
# myapp/reports/sales_report.py
import frappe
from frappe import _

def execute(filters=None):
    """Sales report with comprehensive query error handling."""
    columns = [
        {"label": _("Invoice"), "fieldname": "name", "fieldtype": "Link",
         "options": "Sales Invoice", "width": 120},
        {"label": _("Customer"), "fieldname": "customer", "fieldtype": "Link",
         "options": "Customer", "width": 150},
        {"label": _("Date"), "fieldname": "posting_date", "fieldtype": "Date", "width": 100},
        {"label": _("Total"), "fieldname": "grand_total", "fieldtype": "Currency", "width": 120},
    ]

    try:
        data = get_report_data(filters)
    except frappe.QueryTimeoutError:
        frappe.throw(_("Report too large. Please narrow your date range."))
    except frappe.db.InternalError:
        frappe.log_error(frappe.get_traceback(), "Sales Report Query Error")
        frappe.throw(_("Database error. Please try again."))

    return columns, data


def get_report_data(filters):
    """Build and execute report query with parameterized SQL."""
    conditions = ["si.docstatus = 1"]
    values = {}

    if filters.get("customer"):
        conditions.append("si.customer = %(customer)s")
        values["customer"] = filters["customer"]

    if filters.get("from_date"):
        conditions.append("si.posting_date >= %(from_date)s")
        values["from_date"] = filters["from_date"]

    if filters.get("to_date"):
        conditions.append("si.posting_date <= %(to_date)s")
        values["to_date"] = filters["to_date"]

    where = " AND ".join(conditions)

    return frappe.db.sql(f"""
        SELECT si.name, si.customer, si.posting_date, si.grand_total
        FROM `tabSales Invoice` si
        WHERE {where}
        ORDER BY si.posting_date DESC
        LIMIT 1000
    """, values, as_dict=True)
```

---

## Example 4: Background Sync with Database Error Recovery

```python
# myapp/tasks/sync.py
import frappe
from frappe import _

def sync_customers():
    """Background sync with connection recovery and per-record error handling."""
    results = {"synced": 0, "failed": 0, "errors": []}

    try:
        customers = frappe.get_all(
            "Customer",
            filters={"sync_status": "Pending"},
            fields=["name", "customer_name"],
            limit=200  # ALWAYS limit
        )

        if not customers:
            frappe.db.commit()
            return

        for customer in customers:
            try:
                external_id = call_external_api(customer)
                frappe.db.set_value("Customer", customer.name, {
                    "sync_status": "Synced",
                    "external_id": external_id
                })
                results["synced"] += 1

            except frappe.db.InternalError as e:
                msg = str(e).lower()
                if "gone away" in msg or "lost connection" in msg:
                    frappe.db.connect()  # Reconnect
                    frappe.log_error("Connection lost during sync", "Sync Recovery")
                else:
                    raise

            except Exception as e:
                results["failed"] += 1
                frappe.db.set_value("Customer", customer.name, {
                    "sync_status": "Failed",
                    "sync_error": str(e)[:500]
                })
                frappe.log_error(frappe.get_traceback(), f"Sync: {customer.name}")

        frappe.db.commit()  # REQUIRED in background job

    except frappe.QueryDeadlockError:
        frappe.db.rollback()
        frappe.log_error("Deadlock during sync", "Sync Deadlock")

    except Exception:
        frappe.db.rollback()
        frappe.log_error(frappe.get_traceback(), "Sync Fatal Error")

    if results["failed"]:
        frappe.log_error(frappe.as_json(results), "Sync Summary")


def call_external_api(customer):
    """Call external API — stub."""
    return "EXT-001"
```

---

## Example 5: Controller with Complete Database Error Handling

```python
# myapp/doctype/custom_order/custom_order.py
import frappe
from frappe import _
from frappe.model.document import Document

class CustomOrder(Document):
    def validate(self):
        self.validate_customer()
        self.validate_items()
        self.calculate_totals()

    def validate_customer(self):
        if not self.customer:
            frappe.throw(_("Customer is required"))

        data = frappe.db.get_value(
            "Customer", self.customer,
            ["customer_name", "disabled", "credit_limit"],
            as_dict=True
        )

        if not data:
            frappe.throw(_("Customer '{0}' not found").format(self.customer))
        if data.disabled:
            frappe.throw(_("Customer is disabled"))

    def validate_items(self):
        if not self.items:
            frappe.throw(_("At least one item is required"))

        errors = []

        # Batch fetch for efficiency — avoids N+1 queries
        codes = [r.item_code for r in self.items if r.item_code]
        existing = {
            d.name: d for d in frappe.get_all(
                "Item",
                filters={"name": ["in", codes]},
                fields=["name", "item_name", "disabled", "is_sales_item"]
            )
        } if codes else {}

        for idx, row in enumerate(self.items, 1):
            if not row.item_code:
                errors.append(_("Row {0}: Item code required").format(idx))
                continue
            item = existing.get(row.item_code)
            if not item:
                errors.append(_("Row {0}: Item '{1}' not found").format(idx, row.item_code))
            elif item.disabled:
                errors.append(_("Row {0}: Item '{1}' disabled").format(idx, item.item_name))

        if errors:
            frappe.throw("<br>".join(errors), title=_("Item Errors"))

    def calculate_totals(self):
        self.total = sum(r.amount or 0 for r in self.items)
        self.grand_total = self.total - (self.discount_amount or 0)

    def on_submit(self):
        try:
            self.create_linked_records()
        except frappe.DuplicateEntryError:
            frappe.throw(_("Linked records already exist"))
        except Exception:
            frappe.log_error(frappe.get_traceback(), f"Submit: {self.name}")
            frappe.throw(_("Error creating linked records"))

    def create_linked_records(self):
        pass
```

---

## Example 6: Transaction Hooks [v15+]

```python
import frappe

def create_with_side_effects(data):
    """Use transaction hooks to coordinate DB changes with external systems."""

    doc = frappe.get_doc({"doctype": "Sales Order", **data})
    doc.insert()

    # File will be created ONLY if DB commit succeeds
    frappe.db.after_commit.add(
        lambda: create_export_file(doc.name)
    )

    # Cleanup file if transaction rolls back
    frappe.db.after_rollback.add(
        lambda: remove_temp_file(doc.name)
    )

    return doc.name


def create_export_file(name):
    """Create export file — only called after successful commit."""
    pass

def remove_temp_file(name):
    """Remove temp file — only called after rollback."""
    pass
```

---

## Quick Reference: Database Error Handling

```python
# Check before get_doc
if frappe.db.exists("Customer", name):
    doc = frappe.get_doc("Customer", name)

# Handle None from get_value
val = frappe.db.get_value("Customer", name, "credit_limit")
val = val or 0  # Default if None

# Safe insert
try:
    doc.insert()
except frappe.DuplicateEntryError:
    pass  # Handle duplicate
except frappe.MandatoryError as e:
    frappe.throw(_("Missing: {0}").format(e))

# Safe save
try:
    doc.save()
except frappe.TimestampMismatchError:
    frappe.throw(_("Modified. Refresh and retry."))

# Safe delete
try:
    frappe.delete_doc("Customer", name)
except frappe.LinkExistsError:
    frappe.throw(_("Linked documents exist"))

# Safe query — ALWAYS use %(name)s format
frappe.db.sql("SELECT * FROM `tabItem` WHERE name = %(n)s", {"n": name})

# Background job — ALWAYS commit
frappe.db.commit()
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-database/references/examples.md*
