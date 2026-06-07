---
title: Frappe Ref — Frappe Errors: Api: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'api', 'reference', 'examples']
source: Impertio
---

# Examples — API Error Handling

Complete working examples for Frappe API error handling.

---

## Example 1: Complete REST API Module

```python
# myapp/api.py
"""
Order management API with comprehensive error handling.
ALWAYS follow: validate -> exists -> permission -> business -> execute.
"""
import frappe
from frappe import _


@frappe.whitelist()
def create_sales_order(customer, items, delivery_date=None):
    """Create Sales Order via API."""
    # Validate customer
    if not customer:
        frappe.throw(_("Customer is required"), exc=frappe.ValidationError)
    if not frappe.db.exists("Customer", customer):
        frappe.throw(_("Customer not found"), exc=frappe.DoesNotExistError)

    # Parse and validate items
    if not items:
        frappe.throw(_("At least one item required"), exc=frappe.ValidationError)
    if isinstance(items, str):
        try:
            items = frappe.parse_json(items)
        except Exception:
            frappe.throw(_("Invalid items JSON"), exc=frappe.ValidationError)

    for i, item in enumerate(items):
        if not item.get("item_code"):
            frappe.throw(_("Item {0}: item_code required").format(i + 1),
                exc=frappe.ValidationError)
        if not frappe.db.exists("Item", item["item_code"]):
            frappe.throw(_("Item '{0}' not found").format(item["item_code"]),
                exc=frappe.DoesNotExistError)
        qty = item.get("qty", 0)
        if not qty or qty <= 0:
            frappe.throw(_("Item {0}: qty must be > 0").format(i + 1),
                exc=frappe.ValidationError)

    # Permission
    frappe.has_permission("Sales Order", "create", throw=True)

    # Create
    try:
        order = frappe.get_doc({
            "doctype": "Sales Order",
            "customer": customer,
            "delivery_date": delivery_date or frappe.utils.add_days(frappe.utils.today(), 7),
            "items": [{"item_code": it["item_code"], "qty": it["qty"],
                        "rate": it.get("rate", 0)} for it in items]
        })
        order.insert()
        return {"status": "success", "order_name": order.name,
                "grand_total": order.grand_total}

    except frappe.DuplicateEntryError:
        frappe.throw(_("Duplicate order detected"), exc=frappe.DuplicateEntryError)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Order creation failed: {customer}")
        frappe.throw(_("Failed to create order. Please try again."))


@frappe.whitelist()
def get_order_status(order_name):
    """Get order status with permission-filtered fields."""
    if not order_name:
        frappe.throw(_("Order name required"), exc=frappe.ValidationError)
    if not frappe.db.exists("Sales Order", order_name):
        frappe.throw(_("Order not found"), exc=frappe.DoesNotExistError)
    frappe.has_permission("Sales Order", "read", order_name, throw=True)

    order = frappe.get_doc("Sales Order", order_name)
    result = {
        "name": order.name, "customer": order.customer,
        "status": order.status, "docstatus": order.docstatus,
        "items_count": len(order.items)
    }

    # Financial data only for authorized roles
    financial_roles = ["Accounts User", "Accounts Manager", "Sales Manager", "System Manager"]
    if any(r in frappe.get_roles() for r in financial_roles):
        result.update({"grand_total": order.grand_total,
                        "taxes": order.total_taxes_and_charges})

    return result


@frappe.whitelist()
def cancel_order(order_name, reason=None):
    """Cancel order with business validation."""
    if not order_name:
        frappe.throw(_("Order name required"), exc=frappe.ValidationError)
    if not frappe.db.exists("Sales Order", order_name):
        frappe.throw(_("Order not found"), exc=frappe.DoesNotExistError)

    order = frappe.get_doc("Sales Order", order_name)
    if not order.has_permission("cancel"):
        frappe.throw(_("No cancel permission"), exc=frappe.PermissionError)

    if order.docstatus != 1:
        frappe.throw(_("Only submitted orders can be cancelled"), exc=frappe.ValidationError)
    if order.per_delivered > 0:
        frappe.throw(_("Cancel deliveries first"), exc=frappe.ValidationError)

    try:
        order.cancel()
        if reason:
            frappe.get_doc({"doctype": "Comment", "comment_type": "Info",
                "reference_doctype": "Sales Order", "reference_name": order_name,
                "content": f"Cancelled: {reason}"}).insert(ignore_permissions=True)
        return {"status": "success", "message": _("Order cancelled")}
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Cancel error: {order_name}")
        frappe.throw(_("Cancel failed: {0}").format(str(e)))


@frappe.whitelist()
def bulk_update_orders(order_names, updates):
    """Bulk update with per-document permission checks."""
    if isinstance(order_names, str):
        order_names = frappe.parse_json(order_names)
    if isinstance(updates, str):
        updates = frappe.parse_json(updates)

    if not order_names:
        frappe.throw(_("No orders specified"), exc=frappe.ValidationError)
    if not updates:
        frappe.throw(_("No updates specified"), exc=frappe.ValidationError)

    # Whitelist allowed fields
    allowed = {"delivery_date", "po_no", "customer_address"}
    invalid = set(updates.keys()) - allowed
    if invalid:
        frappe.throw(_("Cannot update: {0}").format(", ".join(invalid)),
            exc=frappe.ValidationError)

    results = {"success": [], "failed": [], "permission_denied": [], "not_found": []}

    for name in order_names:
        if not frappe.db.exists("Sales Order", name):
            results["not_found"].append(name)
            continue
        if not frappe.has_permission("Sales Order", "write", name):
            results["permission_denied"].append(name)
            continue
        try:
            for field, value in updates.items():
                frappe.db.set_value("Sales Order", name, field, value)
            results["success"].append(name)
        except Exception as e:
            results["failed"].append({"name": name, "error": str(e)})

    frappe.db.commit()
    return results
```

---

## Example 2: Client-Side API Integration

```javascript
// myapp/public/js/order_api.js

const OrderAPI = {
    async create(customer, items, deliveryDate) {
        return this._call("myapp.api.create_sales_order",
            {customer, items, delivery_date: deliveryDate},
            __("Creating order..."));
    },

    async getStatus(orderName) {
        return this._call("myapp.api.get_order_status",
            {order_name: orderName}, __("Loading..."));
    },

    async cancel(orderName, reason) {
        return this._call("myapp.api.cancel_order",
            {order_name: orderName, reason}, __("Cancelling..."));
    },

    async bulkUpdate(orderNames, updates) {
        return this._call("myapp.api.bulk_update_orders",
            {order_names: orderNames, updates}, __("Updating..."));
    },

    _call(method, args, freezeMsg) {
        return new Promise((resolve, reject) => {
            frappe.call({
                method, args, freeze: true, freeze_message: freezeMsg,
                callback: (r) => resolve(r.message),
                error: (r) => {
                    this._handleError(r);
                    reject(r);
                }
            });
        });
    },

    _handleError(error) {
        let title = __("Error"), message = __("An error occurred"), indicator = "red";

        // Extract server messages
        if (error._server_messages) {
            try {
                const msgs = JSON.parse(error._server_messages);
                if (msgs.length > 0) {
                    const msg = JSON.parse(msgs[0]);
                    message = msg.message || msg;
                }
            } catch (e) {}
        }

        switch (error.exc_type) {
            case "ValidationError": title = __("Validation Error"); break;
            case "PermissionError": title = __("Permission Denied"); break;
            case "DoesNotExistError": title = __("Not Found"); break;
            case "DuplicateEntryError": title = __("Duplicate"); indicator = "orange"; break;
        }

        if (!error.status) {
            title = __("Network Error");
            message = __("Unable to connect to server");
            indicator = "orange";
        }

        frappe.msgprint({title, message, indicator});
    }
};

// Form integration
frappe.ui.form.on("Sales Order", {
    refresh(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__("Cancel with Reason"), async function() {
                const reason = await new Promise((resolve) => {
                    frappe.prompt({fieldname: "reason", fieldtype: "Small Text",
                        label: __("Reason"), reqd: 1},
                        (v) => resolve(v.reason));
                });
                try {
                    await OrderAPI.cancel(frm.doc.name, reason);
                    frappe.show_alert({message: __("Cancelled"), indicator: "green"});
                    frm.reload_doc();
                } catch (e) { /* error already handled */ }
            }, __("Actions"));
        }
    }
});
```

---

## Example 3: External Shipping API Integration

```python
# myapp/integrations/shipping.py
import frappe
from frappe import _
import requests
import time


class ShippingAPIError(Exception):
    pass


class ShippingAPI:
    def __init__(self):
        self.settings = frappe.get_single("Shipping Settings")
        self.base_url = self.settings.api_url.rstrip("/")
        self.api_key = self.settings.get_password("api_key")
        self.timeout = 30
        self.max_retries = 3

    def create_shipment(self, delivery_note_name):
        if not frappe.db.exists("Delivery Note", delivery_note_name):
            frappe.throw(_("Delivery Note not found"), exc=frappe.DoesNotExistError)
        dn = frappe.get_doc("Delivery Note", delivery_note_name)
        if not dn.shipping_address_name:
            frappe.throw(_("Shipping address required"), exc=frappe.ValidationError)

        payload = self._build_payload(dn)
        try:
            result = self._post("/shipments", payload)
            frappe.db.set_value("Delivery Note", delivery_note_name, {
                "tracking_number": result["tracking_number"],
                "shipping_label_url": result.get("label_url")})
            frappe.db.commit()
            return result
        except ShippingAPIError as e:
            frappe.log_error(str(e), f"Shipping error: {delivery_note_name}")
            frappe.throw(str(e))

    def _post(self, endpoint, data):
        return self._request("POST", endpoint, json=data)

    def _get(self, endpoint):
        return self._request("GET", endpoint)

    def _request(self, method, endpoint, **kwargs):
        url = f"{self.base_url}{endpoint}"
        headers = {"Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json"}
        last_error = None

        for attempt in range(self.max_retries):
            try:
                resp = requests.request(method=method, url=url, headers=headers,
                    timeout=self.timeout, **kwargs)

                if resp.status_code in (200, 201):
                    return resp.json()
                if resp.status_code == 401:
                    raise ShippingAPIError(_("Auth failed. Check API key."))
                if resp.status_code == 429:
                    wait = int(resp.headers.get("Retry-After", 60))
                    time.sleep(min(wait, 120))
                    continue
                if 400 <= resp.status_code < 500:
                    raise ShippingAPIError(self._parse_error(resp))
                if resp.status_code >= 500:
                    last_error = f"Server error {resp.status_code}"
                    time.sleep(2 ** attempt)
                    continue

            except requests.exceptions.Timeout:
                last_error = "Timeout"
                time.sleep(2 ** attempt)
            except requests.exceptions.ConnectionError:
                last_error = "Connection failed"
                time.sleep(2 ** attempt)
            except ShippingAPIError:
                raise

        raise ShippingAPIError(_("Service unavailable: {0}").format(last_error))

    def _parse_error(self, resp):
        try:
            data = resp.json()
            return data.get("error", {}).get("message") or data.get("message") or str(data)
        except Exception:
            return resp.text[:200]

    def _build_payload(self, dn):
        addr = frappe.get_doc("Address", dn.shipping_address_name)
        return {
            "reference": dn.name,
            "recipient": {"name": dn.customer_name, "address_line1": addr.address_line1,
                "city": addr.city, "postal_code": addr.pincode, "country": addr.country},
            "packages": [{"weight": it.total_weight or 1, "description": it.item_name}
                for it in dn.items]
        }


# Whitelisted endpoints
@frappe.whitelist()
def create_shipment(delivery_note):
    return ShippingAPI().create_shipment(delivery_note)

@frappe.whitelist()
def track_shipment(tracking_number):
    if not tracking_number:
        frappe.throw(_("Tracking number required"), exc=frappe.ValidationError)
    return ShippingAPI()._get(f"/tracking/{tracking_number}")
```

---

## Example 4: File Upload Endpoint

```python
@frappe.whitelist()
def upload_attachment(doctype, docname):
    """Handle file upload with validation."""
    if not doctype or not docname:
        frappe.throw(_("DocType and name required"), exc=frappe.ValidationError)
    if not frappe.db.exists(doctype, docname):
        frappe.throw(_("Document not found"), exc=frappe.DoesNotExistError)
    frappe.has_permission(doctype, "write", docname, throw=True)

    files = frappe.request.files
    if not files or "file" not in files:
        frappe.throw(_("No file provided"), exc=frappe.ValidationError)

    uploaded = files["file"]

    # Validate file size (10 MB max)
    max_size = 10 * 1024 * 1024
    uploaded.seek(0, 2)
    size = uploaded.tell()
    uploaded.seek(0)
    if size > max_size:
        frappe.throw(_("File too large (max 10 MB)"), exc=frappe.ValidationError)

    # Validate extension
    allowed_ext = {".pdf", ".png", ".jpg", ".jpeg", ".doc", ".docx", ".xlsx"}
    import os
    ext = os.path.splitext(uploaded.filename)[1].lower()
    if ext not in allowed_ext:
        frappe.throw(_("File type not allowed: {0}").format(ext),
            exc=frappe.ValidationError)

    # Save via Frappe file handler
    try:
        file_doc = frappe.get_doc({
            "doctype": "File",
            "file_name": uploaded.filename,
            "content": uploaded.read(),
            "attached_to_doctype": doctype,
            "attached_to_name": docname,
            "is_private": 1
        })
        file_doc.insert(ignore_permissions=True)
        return {"status": "success", "file_url": file_doc.file_url}
    except Exception:
        frappe.log_error(frappe.get_traceback(), "File Upload Error")
        frappe.throw(_("Upload failed. Please try again."))
```

---

## Quick Reference

```python
# Server-side error throwing with correct HTTP status
frappe.throw(_("Bad input"), exc=frappe.ValidationError)       # 417
frappe.throw(_("Not found"), exc=frappe.DoesNotExistError)     # 404
frappe.throw(_("Forbidden"), exc=frappe.PermissionError)       # 403
frappe.throw(_("Duplicate"), exc=frappe.DuplicateEntryError)   # 409

# Client-side error type checking
# r.exc_type === "ValidationError"
# r.exc_type === "PermissionError"
# r.exc_type === "DoesNotExistError"
# !r.status  → network error
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-api/references/examples.md*
