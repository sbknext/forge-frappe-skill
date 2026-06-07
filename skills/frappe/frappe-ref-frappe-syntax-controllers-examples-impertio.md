---
title: Frappe Ref — Frappe Syntax: Controllers: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'examples']
source: Impertio
---

# Controller Examples Reference

Complete working examples of Document Controllers for common patterns.

---

## 1. Basic Controller with Validation

```python
# myapp/module/doctype/invoice/invoice.py
import frappe
from frappe import _
from frappe.model.document import Document

class Invoice(Document):
    def validate(self):
        """Runs on EVERY save (insert and update)."""
        self.validate_dates()
        self.calculate_totals()
        self.set_status()

    def validate_dates(self):
        if self.due_date and self.posting_date:
            if self.due_date < self.posting_date:
                frappe.throw(_("Due Date cannot be before Posting Date"))

    def calculate_totals(self):
        self.total = 0
        for item in self.items:
            item.amount = item.qty * item.rate
            self.total += item.amount
        self.tax_amount = self.total * 0.21
        self.grand_total = self.total + self.tax_amount

    def set_status(self):
        if self.is_new():
            self.status = "Draft"
```

---

## 2. Controller with Change Detection

```python
# myapp/module/doctype/project/project.py
import frappe
from frappe import _
from frappe.model.document import Document

class Project(Document):
    def validate(self):
        old = self.get_doc_before_save()
        if old is None:
            self.created_by_user = frappe.session.user
        else:
            self.check_status_transition(old)
            self.check_protected_fields(old)

    def check_status_transition(self, old):
        valid_transitions = {
            "Open": ["In Progress", "Cancelled"],
            "In Progress": ["Completed", "On Hold"],
            "On Hold": ["In Progress", "Cancelled"],
        }
        if old.status != self.status:
            allowed = valid_transitions.get(old.status, [])
            if self.status not in allowed:
                frappe.throw(
                    _("Cannot change status from {0} to {1}").format(
                        old.status, self.status
                    )
                )

    def check_protected_fields(self, old):
        protected = ["customer", "project_type"]
        for field in protected:
            if old.get(field) != self.get(field):
                frappe.throw(_("Cannot change {0} after creation").format(field))

    def on_update(self):
        # Changes to self are NOT saved here
        self.add_comment("Edit", f"Updated by {frappe.session.user}")
        if self.flags.get("status_changed"):
            self.notify_team()
```

---

## 3. Submittable Controller (Submit/Cancel Flow)

```python
# myapp/module/doctype/purchase_order/purchase_order.py
import frappe
from frappe import _
from frappe.model.document import Document

class PurchaseOrder(Document):
    def validate(self):
        """Runs on EVERY save including submit."""
        self.validate_items()
        self.calculate_totals()

    def validate_items(self):
        if not self.items:
            frappe.throw(_("At least one item is required"))
        for item in self.items:
            if item.qty <= 0:
                frappe.throw(_("Quantity must be positive for {0}").format(item.item_code))

    def calculate_totals(self):
        self.total_qty = sum(item.qty for item in self.items)
        self.total_amount = sum(item.amount for item in self.items)

    def before_submit(self):
        """Block submit if conditions not met."""
        if self.total_amount > 50000 and not self.manager_approval:
            frappe.throw(_("Manager approval required for orders over 50,000"))

    def on_submit(self):
        """Create entries AFTER submit. Changes to self NOT saved."""
        self.update_ordered_qty()
        self.notify_supplier()

    def update_ordered_qty(self):
        for item in self.items:
            frappe.db.set_value(
                "Item", item.item_code, "ordered_qty",
                frappe.db.get_value("Item", item.item_code, "ordered_qty") + item.qty
            )

    def before_cancel(self):
        """Check linked docs before allowing cancel."""
        linked = frappe.get_all(
            "Purchase Invoice Item",
            filters={"purchase_order": self.name, "docstatus": 1},
            pluck="parent"
        )
        if linked:
            frappe.throw(
                _("Cannot cancel - linked to: {0}").format(", ".join(set(linked)))
            )

    def on_cancel(self):
        """ALWAYS reverse what on_submit created."""
        self.reverse_ordered_qty()

    def reverse_ordered_qty(self):
        for item in self.items:
            frappe.db.set_value(
                "Item", item.item_code, "ordered_qty",
                frappe.db.get_value("Item", item.item_code, "ordered_qty") - item.qty
            )

    def on_update_after_submit(self):
        """Runs when 'Allow on Submit' fields change on submitted doc."""
        if self.has_value_changed("status"):
            self.add_comment("Edit", f"Status changed to {self.status}")
```

---

## 4. Custom Autoname

```python
# myapp/module/doctype/sales_order/sales_order.py
import frappe
from frappe.model.document import Document
from frappe.model.naming import getseries

class SalesOrder(Document):
    def autoname(self):
        """Custom naming: SO-{CUSTOMER_CODE}-{SERIES}"""
        customer_code = frappe.db.get_value(
            "Customer", self.customer, "customer_code"
        ) or "GEN"
        prefix = f"SO-{customer_code[:3].upper()}-"
        self.name = getseries(prefix, 5)
        # Result: SO-ACM-00001, SO-ACM-00002
```

---

## 5. Controller Override via override_doctype_class

```python
# hooks.py
override_doctype_class = {
    "Sales Invoice": "myapp.overrides.sales_invoice.CustomSalesInvoice"
}
```

```python
# myapp/overrides/sales_invoice.py
import frappe
from frappe import _
from erpnext.accounts.doctype.sales_invoice.sales_invoice import SalesInvoice

class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        super().validate()  # ALWAYS call super() first
        self.validate_credit_limit()

    def validate_credit_limit(self):
        customer = frappe.get_cached_doc("Customer", self.customer)
        if customer.credit_limit and self.grand_total > customer.credit_limit:
            frappe.throw(
                _("Exceeds credit limit of {0}").format(customer.credit_limit)
            )

    def on_submit(self):
        super().on_submit()  # ALWAYS call super()
        self.update_customer_stats()
```

---

## 6. Controller Extension via extend_doctype_class [v16+]

```python
# hooks.py
extend_doctype_class = {
    "Address": ["myapp.extensions.address.GeocodingMixin"]
}
```

```python
# myapp/extensions/address.py
from frappe.model.document import Document

class GeocodingMixin(Document):
    @property
    def full_address(self):
        return f"{self.address_line1}, {self.city}, {self.country}"

    def validate(self):
        super().validate()  # ALWAYS call super()
        self.geocode_if_needed()

    def geocode_if_needed(self):
        if self.has_value_changed("address_line1") or self.has_value_changed("city"):
            coords = self.fetch_coordinates()
            if coords:
                self.latitude = coords["lat"]
                self.longitude = coords["lng"]
```

---

## 7. doc_events Handler

```python
# hooks.py
doc_events = {
    "Sales Invoice": {
        "validate": "myapp.events.sales_invoice.validate",
        "on_submit": "myapp.events.sales_invoice.on_submit",
    }
}
```

```python
# myapp/events/sales_invoice.py
import frappe
from frappe import _

def validate(doc, method=None):
    """Handler signature: ALWAYS use (doc, method=None)."""
    if doc.grand_total < 100:
        frappe.msgprint(_("Order value below minimum threshold"))
    if hasattr(doc, "custom_commission_rate"):
        doc.custom_commission = doc.grand_total * (doc.custom_commission_rate / 100)

def on_submit(doc, method=None):
    frappe.enqueue(
        "myapp.integrations.sync_invoice",
        queue="short",
        invoice_name=doc.name,
        enqueue_after_commit=True
    )
```

---

## 8. Whitelisted Method with Client Call

```python
# Controller
class SalesOrder(Document):
    @frappe.whitelist()
    def recalculate_totals(self):
        """Callable from JS: frm.call('recalculate_totals')"""
        self.calculate_totals()
        return {
            "total_qty": self.total_qty,
            "grand_total": self.grand_total
        }

    @frappe.whitelist()
    def send_to_customer(self, include_terms=True):
        """Callable from JS with arguments."""
        frappe.sendmail(
            recipients=[self.contact_email],
            subject=f"Your Order {self.name}",
            message=f"Order {self.name} confirmed"
        )
        return {"status": "sent", "email": self.contact_email}
```

```javascript
// Client-side JavaScript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        frm.add_custom_button(__('Recalculate'), function() {
            frm.call('recalculate_totals').then(r => {
                frm.reload_doc();
                frappe.msgprint(__('Totals updated'));
            });
        });

        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__('Send to Customer'), function() {
                frm.call('send_to_customer', { include_terms: true })
                    .then(r => {
                        frappe.msgprint(__('Sent to {0}', [r.message.email]));
                    });
            });
        }
    }
});
```

---

## 9. Virtual DocType Controller

```python
# myapp/module/doctype/external_product/external_product.py
import frappe
import requests
from frappe.model.document import Document

class ExternalProduct(Document):
    """Virtual DocType: Is Virtual = 1 in DocType settings. No database table."""

    API_BASE = "https://api.example.com/products"

    def load_from_db(self):
        """Called by frappe.get_doc(). Load from external source."""
        response = requests.get(f"{self.API_BASE}/{self.name}")
        response.raise_for_status()
        data = response.json()
        super(Document, self).__init__({
            "name": data["id"],
            "doctype": "External Product",
            "product_name": data["name"],
            "price": data["unit_price"],
        })

    def db_insert(self, *args, **kwargs):
        """Called by doc.insert(). Create in external source."""
        response = requests.post(self.API_BASE, json={"name": self.product_name})
        response.raise_for_status()
        self.name = response.json()["id"]

    def db_update(self, *args, **kwargs):
        """Called by doc.save(). Update in external source."""
        requests.put(f"{self.API_BASE}/{self.name}", json={"name": self.product_name})

    @staticmethod
    def get_list(args):
        """Return list for List View."""
        response = requests.get(ExternalProduct.API_BASE, params={
            "limit": args.get("page_length", 20),
            "offset": args.get("start", 0),
        })
        return [frappe._dict(item) for item in response.json().get("items", [])]

    @staticmethod
    def get_count(args):
        """Return total count for pagination."""
        response = requests.get(f"{ExternalProduct.API_BASE}/count")
        return response.json().get("count", 0)
```

---

## 10. Tree DocType Controller

```python
# myapp/module/doctype/department/department.py
import frappe
from frappe import _
from frappe.utils.nestedset import NestedSet

class Department(NestedSet):
    """Hierarchical DocType. Requires: Is Tree = 1 in DocType settings."""
    nsm_parent_field = "parent_department"

    def validate(self):
        if self.parent_department == self.name:
            frappe.throw(_("Department cannot be its own parent"))

    def on_update(self):
        super().on_update()  # ALWAYS call super for NestedSet
        self.update_employee_count()

    def update_employee_count(self):
        count = frappe.db.count("Employee", {"department": self.name})
        self.db_set("total_employees", count)
```

---

## 11. Flags for Inter-Hook Communication

```python
class SalesOrder(Document):
    def validate(self):
        old = self.get_doc_before_save()
        if old and old.status != self.status:
            self.flags.status_changed = True
            self.flags.old_status = old.status
        if self.grand_total > 10000:
            self.flags.high_value = True

    def on_update(self):
        if self.flags.get("status_changed"):
            frappe.publish_realtime(
                "status_change",
                {"name": self.name, "old": self.flags.old_status, "new": self.status},
                doctype=self.doctype, docname=self.name
            )

    def on_submit(self):
        if self.flags.get("high_value"):
            self.notify_finance_team()
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/examples.md*
