---
title: Frappe Ref — Frappe Impl: Integrations: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'integrations', 'reference', 'examples']
source: Impertio
---

# Integration Examples — Working Code

## Example 1: OAuth Client Configuration for Grafana

```ini
# /etc/grafana/grafana.ini — [auth.generic_oauth] section
[auth.generic_oauth]
enabled = True
name = Frappe
client_id = a1b2c3d4e5f6
client_secret = secretkey123
scopes = openid all
auth_url = https://erp.example.com/api/method/frappe.integrations.oauth2.authorize
token_url = https://erp.example.com/api/method/frappe.integrations.oauth2.get_token
api_url = https://erp.example.com/api/method/frappe.integrations.oauth2.openid_profile
```

## Example 2: Connected App for Google APIs

```python
import frappe

def get_google_calendar_events():
    """Fetch Google Calendar events via Connected App."""
    connected_app = frappe.get_doc("Connected App", "Google Calendar")
    session = connected_app.get_oauth2_session()

    response = session.get(
        "https://www.googleapis.com/calendar/v3/calendars/primary/events",
        params={"maxResults": 10, "orderBy": "startTime", "singleEvents": True}
    )

    if response.status_code == 200:
        return response.json().get("items", [])
    else:
        frappe.log_error(f"Google Calendar API error: {response.text}")
        return []
```

## Example 3: Webhook with JSON Payload

**Webhook DocType Configuration:**
- DocType: Sales Invoice
- Doc Event: on_submit
- Request URL: https://hooks.slack.com/services/T00/B00/XXX
- Webhook Data (JSON):

```json
{
  "text": "New Invoice {{ doc.name }} submitted",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Invoice {{ doc.name }}*\nCustomer: {{ doc.customer }}\nTotal: {{ doc.currency }} {{ doc.grand_total }}"
      }
    }
  ]
}
```

## Example 4: HMAC Webhook Verification (Receiver Side)

```python
# Flask example — receiving Frappe webhooks
import hmac
import hashlib
import base64
from flask import Flask, request, abort

app = Flask(__name__)
WEBHOOK_SECRET = "my-shared-secret"

@app.route("/webhook", methods=["POST"])
def handle_webhook():
    signature = request.headers.get("X-Frappe-Webhook-Signature")
    if not signature:
        abort(401, "Missing signature")

    expected = base64.b64encode(
        hmac.new(
            WEBHOOK_SECRET.encode("utf-8"),
            request.get_data(),
            hashlib.sha256
        ).digest()
    ).decode("utf-8")

    if not hmac.compare_digest(expected, signature):
        abort(401, "Invalid signature")

    data = request.get_json()
    # Process webhook data
    return {"status": "ok"}, 200
```

## Example 5: External API Call with Retry via enqueue

```python
import frappe
import requests

@frappe.whitelist()
def sync_invoice_to_accounting(invoice_name):
    """Queue external sync — NEVER call directly from validate/save."""
    frappe.enqueue(
        "_do_sync",
        invoice_name=invoice_name,
        queue="short",
        timeout=60,
        enqueue_after_commit=True
    )

def _do_sync(invoice_name, retry_count=0):
    doc = frappe.get_doc("Sales Invoice", invoice_name)
    payload = {
        "reference": doc.name,
        "amount": doc.grand_total,
        "customer": doc.customer,
        "date": str(doc.posting_date)
    }

    try:
        response = requests.post(
            "https://accounting.example.com/api/invoices",
            json=payload,
            headers={"Authorization": "Bearer " + get_api_token()},
            timeout=30
        )
        response.raise_for_status()
        frappe.db.set_value("Sales Invoice", invoice_name,
                            "custom_external_id", response.json()["id"])
        frappe.db.commit()
    except requests.exceptions.RequestException as e:
        if retry_count < 3:
            frappe.enqueue(
                "_do_sync",
                invoice_name=invoice_name,
                retry_count=retry_count + 1,
                queue="short",
                timeout=60
            )
        else:
            frappe.log_error(
                f"Sync failed for {invoice_name} after 3 retries: {e}",
                "Accounting Sync Error"
            )

def get_api_token():
    return frappe.utils.password.get_decrypted_password(
        "Integration Settings", "Integration Settings", "api_token"
    )
```

## Example 6: Data Import via Code

```python
import frappe

def import_items_from_csv(file_path):
    """Programmatic data import with error handling."""
    di = frappe.get_doc({
        "doctype": "Data Import",
        "reference_doctype": "Item",
        "import_type": "Insert New Records",
    })
    di.insert()

    # Attach file
    file_doc = frappe.get_doc({
        "doctype": "File",
        "file_url": file_path,
        "attached_to_doctype": "Data Import",
        "attached_to_name": di.name
    })
    file_doc.insert()

    di.import_file = file_doc.file_url
    di.save()
    di.start_import()

    # Check results
    frappe.db.commit()
    di.reload()
    return {
        "total": di.payload_count,
        "success": di.import_log_count,
        "status": di.status
    }
```

## Example 7: Bulk Export to CSV

```python
import frappe
import csv
import io

@frappe.whitelist()
def export_customers_csv():
    """Export all active customers as CSV."""
    customers = frappe.get_all("Customer",
        filters={"disabled": 0},
        fields=["name", "customer_name", "customer_group", "territory",
                "default_currency", "creation"],
        order_by="customer_name asc",
        limit_page_length=0
    )

    output = io.StringIO()
    writer = csv.DictWriter(output, fieldnames=[
        "name", "customer_name", "customer_group", "territory",
        "default_currency", "creation"
    ])
    writer.writeheader()
    writer.writerows(customers)

    # Save as file
    content = output.getvalue()
    file_doc = frappe.get_doc({
        "doctype": "File",
        "file_name": "customers_export.csv",
        "content": content,
        "is_private": 1
    })
    file_doc.save()
    return file_doc.file_url
```

## Example 8: Inbound Webhook Handler

```python
# myapp/api.py
import frappe
import hmac
import hashlib
import base64

@frappe.whitelist(allow_guest=True)
def receive_payment_webhook():
    """Handle inbound webhook from payment provider."""
    # Verify HMAC
    secret = frappe.db.get_single_value("Payment Settings", "webhook_secret")
    signature = frappe.request.headers.get("X-Signature-256")
    payload = frappe.request.get_data()

    expected = base64.b64encode(
        hmac.new(secret.encode(), payload, hashlib.sha256).digest()
    ).decode()

    if not hmac.compare_digest(expected, signature or ""):
        frappe.throw("Invalid webhook signature", frappe.AuthenticationError)

    data = frappe.parse_json(frappe.request.get_data(as_text=True))

    # Idempotency check
    if frappe.db.exists("Payment Log", {"external_id": data["transaction_id"]}):
        return {"status": "already_processed"}

    # Process payment
    frappe.get_doc({
        "doctype": "Payment Log",
        "external_id": data["transaction_id"],
        "amount": data["amount"],
        "status": data["status"]
    }).insert(ignore_permissions=True)

    frappe.db.commit()
    return {"status": "ok"}
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-integrations/references/examples.md*
