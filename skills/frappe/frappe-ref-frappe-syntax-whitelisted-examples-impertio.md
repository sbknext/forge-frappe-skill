---
title: Frappe Ref — Frappe Syntax: Whitelisted: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'examples']
source: Impertio
---

# Working Examples Reference

Complete, production-ready examples of whitelisted methods.

## CRUD API

```python
# myapp/api.py
import frappe
from frappe import _

# CREATE
@frappe.whitelist(methods=["POST"])
def create_task(subject, description=None, assigned_to=None):
    frappe.has_permission("ToDo", "create", throw=True)

    if not subject:
        frappe.throw(_("Subject is required"), frappe.ValidationError)

    doc = frappe.get_doc({
        "doctype": "ToDo",
        "description": subject,
        "allocated_to": assigned_to or frappe.session.user
    })
    if description:
        doc.description = f"{subject}\n\n{description}"
    doc.insert()

    return {"success": True, "name": doc.name, "status": doc.status}


# READ (single)
@frappe.whitelist(methods=["GET"])
def get_task(name):
    frappe.has_permission("ToDo", "read", name, throw=True)
    doc = frappe.get_doc("ToDo", name)
    return {
        "name": doc.name,
        "description": doc.description,
        "status": doc.status,
        "allocated_to": doc.allocated_to
    }


# READ (list with pagination)
@frappe.whitelist(methods=["GET"])
def get_tasks(status=None, limit=20, offset=0):
    frappe.has_permission("ToDo", "read", throw=True)

    limit = min(int(limit), 100)
    offset = int(offset)

    filters = {"allocated_to": frappe.session.user}
    if status:
        filters["status"] = status

    tasks = frappe.get_all(
        "ToDo",
        filters=filters,
        fields=["name", "description", "status", "date"],
        limit_page_length=limit,
        limit_start=offset,
        order_by="modified desc"
    )
    total = frappe.db.count("ToDo", filters)

    return {"data": tasks, "total": total, "limit": limit, "offset": offset}


# UPDATE
@frappe.whitelist(methods=["POST"])
def update_task(name, status=None, description=None):
    frappe.has_permission("ToDo", "write", name, throw=True)
    doc = frappe.get_doc("ToDo", name)
    if status:
        doc.status = status
    if description:
        doc.description = description
    doc.save()
    return {"success": True, "name": doc.name, "status": doc.status}


# DELETE
@frappe.whitelist(methods=["POST"])
def delete_task(name):
    frappe.has_permission("ToDo", "delete", name, throw=True)
    frappe.delete_doc("ToDo", name)
    return {"success": True, "deleted": name}
```

---

## Public Contact Form

```python
# myapp/public_api.py
import frappe
from frappe import _
from frappe.rate_limiter import rate_limit
import re

@frappe.whitelist(allow_guest=True, methods=["POST"])
@rate_limit(limit=5, seconds=300)  # 5 submissions per 5 minutes
def submit_contact(name, email, phone=None, message=None):
    # Validate required
    if not name or not email:
        frappe.throw(_("Name and email are required"), frappe.ValidationError)

    # Validate email format
    if not re.match(r"[^@]+@[^@]+\.[^@]+", email):
        frappe.throw(_("Invalid email format"), frappe.ValidationError)

    # Validate length
    if len(name) > 100:
        frappe.throw(_("Name too long"), frappe.ValidationError)
    if message and len(message) > 5000:
        frappe.throw(_("Message too long"), frappe.ValidationError)

    # Sanitize
    name = frappe.utils.strip_html(name)
    message = frappe.utils.strip_html(message) if message else ""

    doc = frappe.get_doc({
        "doctype": "Communication",
        "communication_type": "Communication",
        "communication_medium": "Email",
        "subject": f"Contact Form: {name}",
        "content": f"From: {name}\nEmail: {email}\nPhone: {phone or 'N/A'}\n\n{message}",
        "sender": email
    })
    doc.insert(ignore_permissions=True)

    return {"success": True, "message": _("Thank you for contacting us!")}
```

---

## Dashboard Statistics API

```python
# myapp/dashboard_api.py
import frappe
from frappe import _
from frappe.utils import nowdate, getdate

@frappe.whitelist(methods=["GET"])
def get_sales_dashboard():
    frappe.only_for(["Sales Manager", "System Manager"])

    today = nowdate()
    month_start = getdate(today).replace(day=1)

    total_orders = frappe.db.count("Sales Order", {
        "docstatus": 1,
        "transaction_date": [">=", month_start]
    })

    revenue = frappe.db.sql("""
        SELECT COALESCE(SUM(grand_total), 0) as total
        FROM `tabSales Order`
        WHERE docstatus = 1 AND transaction_date >= %s
    """, [month_start])[0][0]

    top_customers = frappe.db.sql("""
        SELECT customer, SUM(grand_total) as total
        FROM `tabSales Order`
        WHERE docstatus = 1 AND transaction_date >= %s
        GROUP BY customer
        ORDER BY total DESC
        LIMIT 5
    """, [month_start], as_dict=True)

    return {
        "period": {"start": str(month_start), "end": today},
        "total_orders": total_orders,
        "revenue": float(revenue),
        "top_customers": top_customers
    }
```

---

## File Upload API

```python
# myapp/file_api.py
import frappe
from frappe import _
import base64

@frappe.whitelist(methods=["POST"])
def upload_attachment(doctype, docname, filename, filedata, is_private=True):
    frappe.has_permission(doctype, "write", docname, throw=True)

    # Validate file extension
    ALLOWED_EXTENSIONS = ["pdf", "png", "jpg", "jpeg", "doc", "docx", "xls", "xlsx"]
    ext = filename.rsplit(".", 1)[-1].lower() if "." in filename else ""
    if ext not in ALLOWED_EXTENSIONS:
        frappe.throw(_("File type not allowed: {0}").format(ext))

    # Decode and validate size
    try:
        content = base64.b64decode(filedata)
    except Exception:
        frappe.throw(_("Invalid file data"))

    MAX_SIZE = 5 * 1024 * 1024  # 5MB
    if len(content) > MAX_SIZE:
        frappe.throw(_("File too large (max 5MB)"))

    file_doc = frappe.get_doc({
        "doctype": "File",
        "file_name": filename,
        "attached_to_doctype": doctype,
        "attached_to_name": docname,
        "is_private": int(is_private),
        "content": content
    })
    file_doc.save()

    return {"success": True, "file_name": file_doc.name, "file_url": file_doc.file_url}
```

---

## Batch Processing API

```python
# myapp/batch_api.py
import frappe
from frappe import _

@frappe.whitelist(methods=["POST"])
def batch_update_status(doctype, names, new_status):
    frappe.only_for("System Manager")
    frappe.has_permission(doctype, "write", throw=True)

    if isinstance(names, str):
        names = frappe.parse_json(names)
    if not isinstance(names, list) or not names:
        frappe.throw(_("Names must be a non-empty list"))
    if len(names) > 100:
        frappe.throw(_("Maximum 100 documents per batch"))

    results = {"success": [], "failed": []}
    for name in names:
        try:
            doc = frappe.get_doc(doctype, name)
            doc.status = new_status
            doc.save()
            results["success"].append(name)
        except Exception as e:
            frappe.log_error(frappe.get_traceback(), f"Batch update: {name}")
            results["failed"].append({"name": name, "error": str(e)})

    return {
        "total": len(names),
        "updated": len(results["success"]),
        "failed_count": len(results["failed"]),
        "details": results
    }
```

---

## Controller Method with Client Script

### Server (Python)

```python
# myapp/doctype/custom_order/custom_order.py
import frappe
from frappe import _
from frappe.model.document import Document

class CustomOrder(Document):
    @frappe.whitelist()
    def calculate_commission(self, rate=None):
        if not rate:
            rate = frappe.db.get_value(
                "Sales Person", self.sales_person, "commission_rate"
            ) or 0.05
        commission = self.grand_total * float(rate)
        return {
            "sales_person": self.sales_person,
            "rate": float(rate),
            "commission": commission
        }

    @frappe.whitelist()
    def send_confirmation(self):
        if not self.contact_email:
            frappe.throw(_("No customer email found"))
        frappe.sendmail(
            recipients=[self.contact_email],
            subject=_("Order Confirmation: {0}").format(self.name),
            template="order_confirmation",
            args={"doc": self}
        )
        return {"sent_to": self.contact_email}
```

### Client (JavaScript)

```javascript
// myapp/doctype/custom_order/custom_order.js
frappe.ui.form.on('Custom Order', {
    refresh(frm) {
        if (!frm.is_new()) {
            frm.add_custom_button(__('Calculate Commission'), () => {
                frm.call('calculate_commission', { rate: 0.10 })
                    .then(r => {
                        if (r.message) {
                            frappe.msgprint({
                                title: __('Commission'),
                                indicator: 'green',
                                message: __('Commission: {0}', [
                                    format_currency(r.message.commission)
                                ])
                            });
                        }
                    });
            });

            frm.add_custom_button(__('Send Confirmation'), () => {
                frm.call({
                    method: 'send_confirmation',
                    freeze: true,
                    freeze_message: __('Sending...')
                }).then(r => {
                    if (r.message) {
                        frappe.show_alert({
                            message: __('Sent to {0}', [r.message.sent_to]),
                            indicator: 'green'
                        });
                    }
                });
            }, __('Actions'));
        }
    }
});
```

---

## External Integration with Error Handling

```python
# myapp/integration_api.py
import frappe
from frappe import _
import requests

@frappe.whitelist(methods=["POST"])
def sync_customer(doc_name):
    frappe.only_for(["System Manager", "Integration Manager"])

    doc = frappe.get_doc("Customer", doc_name)
    settings = frappe.get_single("Integration Settings")
    if not settings.api_key:
        frappe.throw(_("Integration not configured"))

    try:
        response = requests.post(
            f"{settings.api_url}/customers",
            json={"name": doc.customer_name, "email": doc.email_id},
            headers={
                "Authorization": f"Bearer {settings.get_password('api_key')}",
                "Content-Type": "application/json"
            },
            timeout=30
        )
        response.raise_for_status()
        result = response.json()

        if result.get("id"):
            doc.db_set("external_id", result["id"])

        return {"success": True, "external_id": result.get("id")}

    except requests.Timeout:
        frappe.log_error(f"Timeout syncing {doc_name}", "External Sync")
        frappe.throw(_("External service timeout"))
    except requests.RequestException as e:
        frappe.log_error(f"Sync error {doc_name}: {e}", "External Sync")
        frappe.throw(_("External service error"))
```

---

## Typed API [v15+]

```python
# myapp/typed_api.py
import frappe
from frappe import _

@frappe.whitelist(methods=["GET"], force_types=True)
def search_items(
    query: str,
    category: str | None = None,
    limit: int = 20,
    include_disabled: bool = False
) -> dict:
    """Fully typed — Frappe validates all params automatically."""
    limit = min(limit, 100)

    filters = {}
    if not include_disabled:
        filters["disabled"] = 0
    if category:
        filters["item_group"] = category

    items = frappe.get_all(
        "Item",
        filters=filters,
        or_filters={"item_name": ["like", f"%{query}%"]} if query else None,
        fields=["name", "item_name", "item_group", "standard_rate"],
        limit_page_length=limit
    )
    return {"items": items, "count": len(items)}
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/examples.md*
