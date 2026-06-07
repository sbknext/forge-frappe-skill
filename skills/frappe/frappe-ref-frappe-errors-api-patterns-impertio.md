---
title: Frappe Ref — Frappe Errors: Api: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'api', 'reference', 'patterns']
source: Impertio
---

# API Error Handling Patterns

Complete error handling patterns for Frappe API development. For quick reference see SKILL.md.

---

## Pattern 1: Complete Whitelisted Method with Validation Pipeline

```python
# myapp/api.py
import frappe
from frappe import _

@frappe.whitelist()
def process_payment(invoice_name, payment_method, amount):
    """
    ALWAYS follow this order: validate -> exists -> permission -> business -> execute.

    HTTP Status Codes:
        200: Success
        400/417: Validation error (frappe.ValidationError)
        403: Permission denied (frappe.PermissionError)
        404: Not found (frappe.DoesNotExistError)
        500: Unhandled server error
    """
    # ── 1. INPUT VALIDATION ──
    errors = []
    if not invoice_name:
        errors.append(_("Invoice name is required"))
    if not payment_method:
        errors.append(_("Payment method is required"))

    valid_methods = ["Cash", "Card", "Bank Transfer", "Check"]
    if payment_method and payment_method not in valid_methods:
        errors.append(_("Invalid payment method: {0}").format(payment_method))

    try:
        amount = float(amount) if amount else 0
        if amount <= 0:
            errors.append(_("Amount must be greater than zero"))
    except (ValueError, TypeError):
        errors.append(_("Invalid amount format"))

    if errors:
        frappe.throw("<br>".join(errors), exc=frappe.ValidationError)

    # ── 2. EXISTENCE CHECK ──
    if not frappe.db.exists("Sales Invoice", invoice_name):
        frappe.throw(_("Sales Invoice {0} not found").format(invoice_name),
            exc=frappe.DoesNotExistError)

    # ── 3. PERMISSION CHECK ──
    frappe.has_permission("Payment Entry", "create", throw=True)

    # ── 4. BUSINESS VALIDATION ──
    invoice = frappe.get_doc("Sales Invoice", invoice_name)
    if invoice.docstatus != 1:
        frappe.throw(_("Invoice must be submitted before payment"),
            exc=frappe.ValidationError)
    if amount > invoice.outstanding_amount:
        frappe.throw(_("Amount ({0}) exceeds outstanding ({1})").format(
            frappe.format_value(amount, {"fieldtype": "Currency"}),
            frappe.format_value(invoice.outstanding_amount, {"fieldtype": "Currency"})),
            exc=frappe.ValidationError)

    # ── 5. EXECUTE WITH ERROR HANDLING ──
    try:
        payment = frappe.get_doc({
            "doctype": "Payment Entry",
            "payment_type": "Receive",
            "party_type": "Customer",
            "party": invoice.customer,
            "paid_amount": amount,
            "received_amount": amount,
            "mode_of_payment": payment_method,
            "references": [{
                "reference_doctype": "Sales Invoice",
                "reference_name": invoice_name,
                "allocated_amount": amount
            }]
        })
        payment.insert()
        payment.submit()

        return {
            "status": "success",
            "payment_entry": payment.name,
            "message": _("Payment recorded successfully")
        }

    except frappe.DuplicateEntryError:
        frappe.throw(_("Duplicate payment detected"), exc=frappe.DuplicateEntryError)
    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Payment Error: {invoice_name}")
        frappe.throw(_("Payment failed. Please try again."))
```

---

## Pattern 2: API Response Wrapper Decorator

```python
# myapp/api_utils.py
import frappe
from frappe import _
from functools import wraps

def api_response(func):
    """
    Decorator for consistent API responses across all endpoints.

    Success: {"status": "success", "data": ...}
    Error:   {"status": "error", "type": "...", "message": "..."}
    """
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            result = func(*args, **kwargs)
            return {"status": "success", "data": result}

        except frappe.ValidationError as e:
            frappe.local.response["http_status_code"] = 400
            return {"status": "error", "type": "ValidationError", "message": str(e)}

        except frappe.PermissionError as e:
            frappe.local.response["http_status_code"] = 403
            return {"status": "error", "type": "PermissionError",
                    "message": str(e) or _("Permission denied")}

        except frappe.DoesNotExistError as e:
            frappe.local.response["http_status_code"] = 404
            return {"status": "error", "type": "NotFound",
                    "message": str(e) or _("Resource not found")}

        except frappe.DuplicateEntryError as e:
            frappe.local.response["http_status_code"] = 409
            return {"status": "error", "type": "Conflict",
                    "message": str(e) or _("Duplicate entry")}

        except Exception:
            frappe.log_error(frappe.get_traceback(), "API Error")
            frappe.local.response["http_status_code"] = 500
            return {"status": "error", "type": "ServerError",
                    "message": _("An unexpected error occurred")}

    return wrapper


# Usage
@frappe.whitelist()
@api_response
def get_customer_orders(customer):
    if not customer:
        raise frappe.ValidationError(_("Customer is required"))
    if not frappe.db.exists("Customer", customer):
        raise frappe.DoesNotExistError(_("Customer not found"))
    return frappe.get_list("Sales Order",
        filters={"customer": customer},
        fields=["name", "transaction_date", "grand_total", "status"])
```

---

## Pattern 3: External API Client with Retry

```python
# myapp/integrations/api_client.py
import frappe
from frappe import _
import requests
import time

class ExternalAPIClient:
    """External API client with retry, rate limit, and error handling."""

    def __init__(self, settings_doctype="External API Settings"):
        self.settings = frappe.get_single(settings_doctype)
        self.base_url = self.settings.base_url.rstrip("/")
        self.max_retries = 3
        self.timeout = 30

    def _get_headers(self):
        return {
            "Authorization": f"Bearer {self.settings.get_password('api_key')}",
            "Content-Type": "application/json",
            "Accept": "application/json"
        }

    def request(self, method, endpoint, data=None, params=None):
        """
        Make request with retry logic.

        NEVER retry 4xx errors (except 429). ALWAYS retry 5xx and timeouts.
        """
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        last_error = None

        for attempt in range(self.max_retries):
            try:
                response = requests.request(
                    method=method, url=url, json=data, params=params,
                    headers=self._get_headers(), timeout=self.timeout)

                # Success
                if response.status_code in (200, 201):
                    return response.json()

                # Auth error — NEVER retry
                if response.status_code == 401:
                    frappe.log_error(f"Auth failed: {response.text[:500]}", "API Auth")
                    frappe.throw(_("Authentication failed"), exc=frappe.AuthenticationError)

                # Permission error — NEVER retry
                if response.status_code == 403:
                    frappe.throw(_("API access denied"), exc=frappe.PermissionError)

                # Not found — NEVER retry
                if response.status_code == 404:
                    frappe.throw(_("Resource not found: {0}").format(endpoint),
                        exc=frappe.DoesNotExistError)

                # Rate limit — wait and retry
                if response.status_code == 429:
                    wait = int(response.headers.get("Retry-After", 60))
                    time.sleep(min(wait, 120))
                    continue

                # Other client errors — NEVER retry
                if 400 <= response.status_code < 500:
                    error_msg = self._parse_error(response)
                    frappe.throw(_("API error: {0}").format(error_msg),
                        exc=frappe.ValidationError)

                # Server errors — retry with backoff
                if response.status_code >= 500:
                    last_error = f"Server error {response.status_code}"
                    time.sleep(2 ** attempt)
                    continue

            except requests.exceptions.Timeout:
                last_error = "Timeout"
                time.sleep(2 ** attempt)
                continue
            except requests.exceptions.ConnectionError:
                last_error = "Connection failed"
                time.sleep(2 ** attempt)
                continue
            except (frappe.ValidationError, frappe.AuthenticationError,
                    frappe.PermissionError, frappe.DoesNotExistError):
                raise  # NEVER retry Frappe exceptions
            except Exception as e:
                last_error = str(e)
                frappe.log_error(frappe.get_traceback(), "API Client Error")
                break

        frappe.log_error(f"Failed after {self.max_retries} attempts: {last_error}",
            "API Client Failure")
        frappe.throw(_("External service unavailable. Please try again later."))

    def _parse_error(self, response):
        try:
            data = response.json()
            return (data.get("error", {}).get("message")
                    or data.get("message") or data.get("detail") or str(data))
        except Exception:
            return response.text[:200]

    def get(self, endpoint, params=None):
        return self.request("GET", endpoint, params=params)

    def post(self, endpoint, data):
        return self.request("POST", endpoint, data=data)

    def put(self, endpoint, data):
        return self.request("PUT", endpoint, data=data)

    def delete(self, endpoint):
        return self.request("DELETE", endpoint)
```

---

## Pattern 4: Webhook Handler with Signature Verification

```python
# myapp/webhooks.py
import frappe
import json
import hmac
import hashlib

@frappe.whitelist(allow_guest=True)
def incoming_webhook():
    """
    Webhook receiver with full error handling.

    ALWAYS: verify signature, parse JSON safely, return 200 quickly.
    NEVER: process synchronously, trust unverified payloads.
    """
    payload = frappe.request.data
    signature = frappe.request.headers.get("X-Webhook-Signature")

    # 1. Verify signature
    if not _verify_signature(payload, signature):
        frappe.local.response["http_status_code"] = 401
        return {"error": "Invalid signature"}

    # 2. Parse JSON safely
    try:
        data = json.loads(payload)
    except json.JSONDecodeError:
        frappe.local.response["http_status_code"] = 400
        return {"error": "Invalid JSON"}

    # 3. Check for duplicate (idempotency)
    event_id = data.get("id")
    if event_id and frappe.db.exists("Webhook Event Log", {"event_id": event_id}):
        return {"status": "already_processed"}

    # 4. Enqueue processing — return 200 immediately
    frappe.enqueue("myapp.webhooks.process_event",
        queue="short", data=data, event_id=event_id)

    return {"status": "accepted"}


def _verify_signature(payload, signature):
    if not signature:
        return False
    try:
        secret = frappe.get_single("Webhook Settings").get_password("secret")
        computed = hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()
        return hmac.compare_digest(computed, signature)
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Webhook signature verification")
        return False


def process_event(data, event_id=None):
    """Process webhook event in background."""
    try:
        event_type = data.get("type")
        handlers = {
            "payment.completed": handle_payment,
            "order.created": handle_order,
        }
        handler = handlers.get(event_type)
        if handler:
            handler(data.get("data", {}))

        # Mark as processed
        if event_id:
            frappe.get_doc({
                "doctype": "Webhook Event Log",
                "event_id": event_id,
                "event_type": event_type,
                "processed_at": frappe.utils.now()
            }).insert(ignore_permissions=True)
            frappe.db.commit()

    except Exception:
        frappe.log_error(frappe.get_traceback(), f"Webhook processing: {event_id}")
```

---

## Pattern 5: Client-Side Error Handler

```javascript
// myapp/public/js/api_handler.js
class APIHandler {
    static async call(options) {
        return new Promise((resolve, reject) => {
            frappe.call({
                freeze: true,
                freeze_message: __("Processing..."),
                ...options,
                callback: (r) => {
                    if (r.message && r.message.status === "error") {
                        this.handleError(r.message);
                        reject(r.message);
                    } else {
                        resolve(r.message);
                    }
                },
                error: (r) => {
                    this.handleError(r);
                    reject(r);
                }
            });
        });
    }

    static handleError(error) {
        const info = this.parseError(error);
        frappe.msgprint({title: info.title, message: info.message, indicator: info.indicator});
    }

    static parseError(error) {
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

        // Map error types to titles
        switch (error.exc_type || error.type) {
            case "ValidationError": title = __("Validation Error"); break;
            case "PermissionError": title = __("Permission Denied"); break;
            case "DoesNotExistError": title = __("Not Found"); break;
            case "DuplicateEntryError": title = __("Duplicate"); indicator = "orange"; break;
            case "RateLimitExceededError": title = __("Rate Limited"); indicator = "orange"; break;
        }

        // Network error
        if (!error.status && !error.type && !error.exc_type) {
            title = __("Network Error");
            message = __("Unable to connect. Check your connection.");
            indicator = "orange";
        }

        return {title, message, indicator};
    }
}
```

---

## Quick Reference: Error Throwing

```python
# Validation (HTTP 417)
frappe.throw(_("Invalid input"), exc=frappe.ValidationError)

# Not found (HTTP 404)
frappe.throw(_("Not found"), exc=frappe.DoesNotExistError)

# Permission denied (HTTP 403)
frappe.throw(_("Access denied"), exc=frappe.PermissionError)

# Duplicate (HTTP 409)
frappe.throw(_("Duplicate"), exc=frappe.DuplicateEntryError)

# Log + throw (generic 417)
frappe.log_error(frappe.get_traceback(), "Context")
frappe.throw(_("Something went wrong"))
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-api/references/patterns.md*
