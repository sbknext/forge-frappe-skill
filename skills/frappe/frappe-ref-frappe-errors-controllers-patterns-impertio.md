---
title: Frappe Ref — Frappe Errors: Controllers: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'controllers', 'reference', 'patterns']
source: Impertio
---

# Controller Error Handling Patterns

Reusable patterns for defensive error handling in Frappe Document Controllers.

---

## Pattern 1: Validation Error Collector

```python
import frappe
from frappe import _

class SalesOrder(Document):
    def validate(self):
        errors = []
        warnings = []

        # Required fields
        if not self.customer:
            errors.append(_("Customer is required"))
        if not self.items:
            errors.append(_("At least one item is required"))

        # Business rules
        if self.discount_percent and self.discount_percent > 50:
            errors.append(_("Discount cannot exceed 50%"))

        # Child table validation
        for idx, item in enumerate(self.items or [], 1):
            if not item.item_code:
                errors.append(_("Row {0}: Item Code is required").format(idx))
            if (item.qty or 0) <= 0:
                errors.append(_("Row {0}: Quantity must be positive").format(idx))

        # Warnings (non-blocking)
        if self.grand_total and self.grand_total > 100000:
            warnings.append(_("Large order — may require approval"))

        if warnings:
            frappe.msgprint("<br>".join(warnings), title=_("Warnings"), indicator="orange")
        if errors:
            frappe.throw("<br>".join(errors), title=_("Validation Error"))
```

---

## Pattern 2: Safe Controller Override

```python
from erpnext.selling.doctype.sales_order.sales_order import SalesOrder
import frappe
from frappe import _

class CustomSalesOrder(SalesOrder):
    def validate(self):
        # ALWAYS call parent first
        super().validate()
        # Then add custom logic
        self.validate_credit()

    def on_submit(self):
        super().on_submit()
        # Non-critical post-submit
        try:
            self.sync_external()
        except Exception:
            frappe.log_error(frappe.get_traceback(), f"Sync failed: {self.name}")
            frappe.msgprint(_("Submitted. External sync will retry."), indicator="orange")

    def validate_credit(self):
        if not self.customer:
            return
        limit = frappe.db.get_value("Customer", self.customer, "credit_limit") or 0
        if limit and self.grand_total > limit:
            frappe.throw(
                _("Amount {0} exceeds credit limit {1}").format(
                    frappe.format_value(self.grand_total, {"fieldtype": "Currency"}),
                    frappe.format_value(limit, {"fieldtype": "Currency"})
                )
            )
```

---

## Pattern 3: Isolated Post-Save Operations

```python
class SalesOrder(Document):
    def on_update(self):
        """Each operation independent — one failure doesn't block others."""
        errors = []

        operations = [
            (self.update_quotation, "Quotation update"),
            (self.sync_to_crm, "CRM sync"),
            (self.send_notification, "Notification"),
        ]

        for operation, label in operations:
            try:
                operation()
            except Exception:
                errors.append(label)
                frappe.log_error(frappe.get_traceback(), f"{label} failed: {self.name}")

        if errors:
            frappe.msgprint(
                _("Saved. Failed: {0}").format(", ".join(errors)),
                indicator="orange"
            )
```

---

## Pattern 4: Submit with Proper Hook Separation

```python
class SalesOrder(Document):
    def before_submit(self):
        """ALL validation here — last clean abort point."""
        # Stock check
        for item in self.items:
            if item.warehouse:
                available = self.get_stock(item.item_code, item.warehouse)
                if available < item.qty:
                    frappe.throw(
                        _("Row {0}: Insufficient stock for {1}. Available: {2}").format(
                            item.idx, item.item_code, available
                        )
                    )

        # Approval check
        if self.grand_total > 100000 and not self.manager_approval:
            frappe.throw(_("Manager approval required for orders over 100,000"))

    def on_submit(self):
        """Post-submit actions only — document is already submitted."""
        # Critical: create stock entries
        try:
            self.create_stock_entries()
        except Exception as e:
            frappe.log_error(frappe.get_traceback(), "Stock Entry Error")
            frappe.throw(_("Stock entries failed: {0}").format(str(e)))

        # Non-critical: update customer stats
        try:
            frappe.db.set_value("Customer", self.customer, "last_order_date", self.transaction_date)
        except Exception:
            frappe.log_error(frappe.get_traceback(), "Customer Update Error")
```

---

## Pattern 5: Cancel with Full Cleanup

```python
class SalesOrder(Document):
    def before_cancel(self):
        """Check if cancel is allowed."""
        linked = frappe.get_all("Delivery Note Item",
            filters={"against_sales_order": self.name, "docstatus": 1},
            pluck="parent")

        if linked:
            frappe.throw(
                _("Cannot cancel. Linked Delivery Notes: {0}").format(
                    ", ".join(set(linked))
                )
            )

    def on_cancel(self):
        """Attempt all cleanup operations."""
        errors = []

        for operation, label in [
            (self.release_stock, "Stock release"),
            (self.reverse_gl, "GL reversal"),
            (self.update_linked_docs, "Linked docs"),
        ]:
            try:
                operation()
            except Exception as e:
                errors.append(f"{label}: {str(e)}")
                frappe.log_error(frappe.get_traceback(), f"{label} Error")

        if errors:
            frappe.msgprint(
                _("Cancelled with errors:<br>{0}").format("<br>".join(errors)),
                indicator="orange"
            )
```

---

## Pattern 6: External API Call with Fallback

```python
import requests
from requests.exceptions import Timeout, ConnectionError, RequestException

class PaymentDoc(Document):
    def validate(self):
        if self.requires_verification:
            self.verify_external()

    def verify_external(self):
        try:
            response = requests.post(
                self.api_endpoint,
                json={"ref": self.name},
                timeout=10  # ALWAYS set timeout
            )

            if response.status_code == 200:
                self.verified = 1
            elif response.status_code == 401:
                frappe.throw(_("API credentials invalid"))
            elif response.status_code >= 500:
                frappe.throw(_("External service unavailable"))
            else:
                frappe.throw(_("Verification failed: {0}").format(response.text[:200]))

        except Timeout:
            frappe.throw(_("Verification timed out. Please try again."))
        except ConnectionError:
            frappe.throw(_("Could not connect to verification service."))
        except RequestException as e:
            frappe.log_error(frappe.get_traceback(), "Verification Error")
            frappe.throw(_("Verification error. Please try again."))
```

---

## Pattern 7: Recursion Guard with Flags

```python
class SalesOrder(Document):
    def on_update(self):
        if self.flags.get("skip_linked_update"):
            return

        if self.quotation:
            self.update_quotation()

    def update_quotation(self):
        q = frappe.get_doc("Quotation", self.quotation)
        q.flags.skip_linked_update = True  # Prevent Quotation from updating back
        q.db_set("status", "Ordered")
```

---

## Pattern 8: Batch Processing with Savepoints

```python
class BulkProcessor(Document):
    def on_submit(self):
        results = {"success": [], "failed": []}

        for item in self.items:
            frappe.db.savepoint(f"item_{item.idx}")
            try:
                self.process_item(item)
                results["success"].append(item.item_code)
            except frappe.ValidationError as e:
                frappe.db.rollback(save_point=f"item_{item.idx}")
                results["failed"].append({"item": item.item_code, "error": str(e)})
            except Exception as e:
                frappe.db.rollback(save_point=f"item_{item.idx}")
                frappe.log_error(frappe.get_traceback(), f"Batch Error: {item.item_code}")
                results["failed"].append({"item": item.item_code, "error": "Unexpected error"})

        self.db_set("processed_count", len(results["success"]))
        self.db_set("failed_count", len(results["failed"]))

        if results["failed"]:
            detail = "<br>".join(f"{f['item']}: {f['error']}" for f in results["failed"][:10])
            frappe.msgprint(
                _("{0} processed, {1} failed:<br>{2}").format(
                    len(results["success"]), len(results["failed"]), detail
                ),
                indicator="orange"
            )
```

---

## Pattern 9: Change Detection with Safe Comparison

```python
class Contract(Document):
    def validate(self):
        if not self.is_new():
            self.validate_changes()

    def validate_changes(self):
        old = self.get_doc_before_save()
        if not old:
            return

        # Safe comparison (handle None)
        if (old.get("status") or "") != (self.status or ""):
            self.validate_status_transition(old.status, self.status)

        if (old.get("contract_value") or 0) != (self.contract_value or 0):
            change = abs((self.contract_value or 0) - (old.contract_value or 0))
            if old.contract_value and change / old.contract_value > 0.25:
                frappe.throw(_("Value change exceeds 25% limit"))

    def validate_status_transition(self, old_status, new_status):
        allowed = {
            "Draft": ["Active", "Cancelled"],
            "Active": ["Completed", "Suspended"],
            "Suspended": ["Active", "Cancelled"],
        }
        if old_status and new_status not in allowed.get(old_status, []):
            frappe.throw(_("Cannot change from {0} to {1}").format(old_status, new_status))
```

---

## Pattern 10: Async Background Task with Error Tracking

```python
class DataImport(Document):
    def on_submit(self):
        if not self.import_file:
            frappe.throw(_("Import file is required"))

        frappe.enqueue(
            "myapp.tasks.run_import",
            queue="long",
            timeout=3600,
            job_id=f"import_{self.name}",
            import_name=self.name
        )
        self.db_set("status", "Processing")
        frappe.msgprint(_("Import started. You will be notified when complete."))


# In myapp/tasks.py
def run_import(import_name):
    doc = frappe.get_doc("Data Import", import_name)
    try:
        # Process...
        doc.db_set("status", "Completed")
        frappe.db.commit()
        frappe.publish_realtime("import_done", {"name": import_name}, user=doc.owner)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Import Failed: {import_name}")
        doc.db_set("status", "Failed")
        frappe.db.commit()
        frappe.publish_realtime("import_failed", {"name": import_name}, user=doc.owner)
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-controllers/references/patterns.md*
