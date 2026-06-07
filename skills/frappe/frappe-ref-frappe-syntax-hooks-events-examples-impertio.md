---
title: Frappe Ref — Frappe Syntax: Hooks Events: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'events', 'reference', 'examples']
source: Impertio
---

# Working doc_events Examples

## 1. Validate Before Save — Block Invalid Data

```python
# hooks.py
doc_events = {
    "Sales Invoice": {
        "validate": "myapp.events.sales_invoice.validate_invoice"
    }
}
```

```python
# myapp/events/sales_invoice.py
import frappe

def validate_invoice(doc, method=None):
    """ALWAYS use frappe.throw() to block saves — NEVER use raise."""
    if not doc.items:
        frappe.throw("Sales Invoice must have at least one item")

    for item in doc.items:
        if item.rate <= 0:
            frappe.throw(f"Row {item.idx}: Rate must be greater than zero")
```

---

## 2. Set Defaults in before_validate

```python
# hooks.py
doc_events = {
    "Purchase Order": {
        "before_validate": "myapp.events.purchase_order.set_defaults"
    }
}
```

```python
# myapp/events/purchase_order.py
import frappe

def set_defaults(doc, method=None):
    """Set missing values BEFORE validation runs."""
    if not doc.delivery_date:
        doc.delivery_date = frappe.utils.add_days(doc.transaction_date, 7)

    if not doc.payment_terms_template:
        doc.payment_terms_template = frappe.db.get_single_value(
            "Buying Settings", "payment_terms_template"
        )
```

---

## 3. Post-Submit Logic — Create Linked Documents

```python
# hooks.py
doc_events = {
    "Sales Order": {
        "on_submit": "myapp.events.sales_order.create_project"
    }
}
```

```python
# myapp/events/sales_order.py
import frappe

def create_project(doc, method=None):
    """Create a project when a Sales Order is submitted."""
    if doc.project:
        return  # Project already linked

    project = frappe.new_doc("Project")
    project.project_name = f"Project for {doc.name}"
    project.company = doc.company
    project.sales_order = doc.name
    project.expected_start_date = doc.delivery_date
    project.flags.ignore_permissions = True
    project.insert()

    # Update the Sales Order with the project link
    doc.db_set("project", project.name, update_modified=False)
```

---

## 4. Cancel Logic — Reverse Linked Documents

```python
# hooks.py
doc_events = {
    "Sales Order": {
        "on_cancel": "myapp.events.sales_order.cancel_linked_project"
    }
}
```

```python
# myapp/events/sales_order.py
import frappe

def cancel_linked_project(doc, method=None):
    """ALWAYS clean up linked documents on cancel."""
    if not doc.project:
        return

    project = frappe.get_doc("Project", doc.project)
    if project.status != "Cancelled":
        project.status = "Cancelled"
        project.flags.ignore_permissions = True
        project.save()
```

---

## 5. Wildcard Handler — Global Audit Trail

```python
# hooks.py
doc_events = {
    "*": {
        "on_update": "myapp.events.audit.log_change",
        "on_trash": "myapp.events.audit.log_deletion",
    }
}
```

```python
# myapp/events/audit.py
import frappe

def log_change(doc, method=None):
    """Log every document change. Runs for ALL DocTypes."""
    if doc.doctype in ("Comment", "Version", "Activity Log"):
        return  # Avoid infinite loops on logging DocTypes

    frappe.get_doc({
        "doctype": "Activity Log",
        "subject": f"{doc.doctype} {doc.name} updated by {frappe.session.user}",
        "reference_doctype": doc.doctype,
        "reference_name": doc.name,
    }).insert(ignore_permissions=True)


def log_deletion(doc, method=None):
    """Log document deletions."""
    if doc.doctype in ("Comment", "Version", "Activity Log"):
        return

    frappe.get_doc({
        "doctype": "Activity Log",
        "subject": f"{doc.doctype} {doc.name} deleted by {frappe.session.user}",
        "reference_doctype": doc.doctype,
        "reference_name": doc.name,
    }).insert(ignore_permissions=True)
```

---

## 6. Using doc.flags for Cross-Event Communication

```python
# hooks.py
doc_events = {
    "Sales Invoice": {
        "validate": "myapp.events.sales_invoice.check_credit",
        "on_submit": "myapp.events.sales_invoice.notify_customer",
    }
}
```

```python
# myapp/events/sales_invoice.py
import frappe

def check_credit(doc, method=None):
    """Check credit limit during validation."""
    customer_balance = get_customer_outstanding(doc.customer)
    if customer_balance + doc.grand_total > get_credit_limit(doc.customer):
        doc.flags.over_credit_limit = True
        frappe.msgprint("Customer is over credit limit — manager approval required")


def notify_customer(doc, method=None):
    """Send notification on submit, with extra warning if over credit limit."""
    template = "invoice_submitted"
    if doc.flags.get("over_credit_limit"):
        template = "invoice_submitted_over_credit"

    frappe.sendmail(
        recipients=[doc.contact_email],
        template=template,
        args={"doc": doc},
    )
```

---

## 7. extend_doctype_class [v16+]

```python
# hooks.py
extend_doctype_class = {
    "Sales Invoice": [
        "myapp.overrides.sales_invoice.SalesInvoiceMixin"
    ]
}
```

```python
# myapp/overrides/sales_invoice.py
import frappe

class SalesInvoiceMixin:
    def validate(self):
        """Extension methods run as part of the controller chain."""
        if self.is_return and not self.return_against:
            frappe.throw("Return invoice must reference the original invoice")

    def get_custom_report_data(self):
        """Custom methods are available on the doc instance."""
        return frappe.db.sql("""
            SELECT item_code, qty, rate
            FROM `tabSales Invoice Item`
            WHERE parent = %s
        """, self.name, as_dict=True)
```

---

## 8. Handling Amendments

```python
# hooks.py
doc_events = {
    "Sales Order": {
        "after_insert": "myapp.events.sales_order.handle_amendment"
    }
}
```

```python
# myapp/events/sales_order.py
import frappe

def handle_amendment(doc, method=None):
    """Detect and handle amended documents."""
    if not doc.amended_from:
        return  # Not an amendment — normal insert

    # Copy custom data from the original document
    original = frappe.get_doc("Sales Order", doc.amended_from)
    doc.db_set("custom_reference", original.custom_reference, update_modified=False)

    frappe.msgprint(f"Amendment created from {doc.amended_from}")
```

---

## 9. Blocking Deletion with on_trash

```python
# hooks.py
doc_events = {
    "Customer": {
        "on_trash": "myapp.events.customer.prevent_deletion"
    }
}
```

```python
# myapp/events/customer.py
import frappe

def prevent_deletion(doc, method=None):
    """ALWAYS check for linked transactions before allowing deletion."""
    linked_invoices = frappe.db.count("Sales Invoice", {"customer": doc.name})
    if linked_invoices > 0:
        frappe.throw(
            f"Cannot delete Customer {doc.name}: "
            f"{linked_invoices} Sales Invoice(s) exist"
        )
```

---

## 10. Async Processing with frappe.enqueue

```python
# hooks.py
doc_events = {
    "Sales Invoice": {
        "on_submit": "myapp.events.sales_invoice.schedule_pdf_generation"
    }
}
```

```python
# myapp/events/sales_invoice.py
import frappe

def schedule_pdf_generation(doc, method=None):
    """NEVER do slow operations synchronously in event handlers."""
    frappe.enqueue(
        "myapp.events.sales_invoice.generate_and_email_pdf",
        queue="long",
        timeout=300,
        doc_name=doc.name,
    )


def generate_and_email_pdf(doc_name):
    """Runs asynchronously in background worker."""
    doc = frappe.get_doc("Sales Invoice", doc_name)
    pdf = frappe.get_print(doc.doctype, doc.name, print_format="Standard")
    # ... attach and email
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks-events/references/examples.md*
