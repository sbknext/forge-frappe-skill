---
title: Frappe Ref — Frappe Core: Api: Webhooks Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'api', 'reference', 'webhooks-reference']
source: Impertio
---

# Webhooks Reference

> Complete reference for Frappe webhook configuration, security, and handling.

---

## Overview

Webhooks are user-defined HTTP callbacks that trigger on document events, sending HTTP requests to configured URLs.

---

## Configuration (via UI)

1. Webhook DocType > New
2. Select DocType (e.g., "Sales Order")
3. Select Doc Event
4. Enter Request URL
5. Optional: Add HTTP Headers (API keys, auth tokens)
6. Optional: Set Conditions (Jinja2 syntax)
7. Optional: Set Webhook Secret for HMAC verification

---

## Available Events

| Event | Trigger Moment |
|-------|----------------|
| `after_insert` | After new document is created and saved |
| `on_update` | After every save operation |
| `on_submit` | After document submit (docstatus: 0 > 1) |
| `on_cancel` | After document cancel (docstatus: 1 > 2) |
| `on_trash` | Before document deletion |
| `on_update_after_submit` | After amendment to submitted doc |
| `on_change` | On every change (catch-all) |

---

## Request Structure

Frappe sends automatically:

```
POST {webhook_url}
Content-Type: application/json

{
    "doctype": "Sales Order",
    "name": "SO-00001",
    "data": {
        "name": "SO-00001",
        "customer": "Customer A",
        "grand_total": 1500.00,
        "status": "Draft",
        ...all document fields...
    }
}
```

---

## Webhook Conditions

Conditions use Jinja2 syntax. Webhook only triggers when condition evaluates to `True`:

```jinja2
{# Only for large orders #}
{{ doc.grand_total > 10000 }}

{# Only premium customers #}
{{ doc.customer_group == "Premium" }}

{# Specific statuses #}
{{ doc.status in ["Submitted", "Paid"] }}

{# Combination #}
{{ doc.grand_total > 5000 and doc.customer_group == "Premium" }}
```

---

## Data Format Options

### Form-Based

Configure fields individually in Webhook Data table:

| Fieldname | Key |
|-----------|-----|
| `customer` | `customer` |
| `grand_total` | `amount` |

Sent as form-encoded: `customer=Customer%20A&amount=1500`

### JSON-Based (with Jinja)

Select "JSON" as Request Structure and write a Jinja template:

```json
{
    "order_id": "{{ doc.name }}",
    "customer": "{{ doc.customer }}",
    "total": {{ doc.grand_total }},
    "items": [
        {% for item in doc.items %}
        {
            "item_code": "{{ item.item_code }}",
            "qty": {{ item.qty }}
        }{% if not loop.last %},{% endif %}
        {% endfor %}
    ]
}
```

---

## Webhook Security — HMAC Signature

When "Webhook Secret" is configured, Frappe adds a signature header:

```
X-Frappe-Webhook-Signature: base64_encoded_hmac_sha256_of_payload
```

### Python Verification

```python
import hmac
import hashlib
import base64

def verify_webhook_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = base64.b64encode(
        hmac.new(secret.encode(), payload, hashlib.sha256).digest()
    ).decode()
    return hmac.compare_digest(expected, signature)
```

### Complete Handler Example (Flask)

```python
from flask import Flask, request, jsonify
import hmac, hashlib, base64, logging, os

app = Flask(__name__)
logger = logging.getLogger(__name__)
WEBHOOK_SECRET = os.environ.get('WEBHOOK_SECRET')

def verify_signature(payload: bytes, signature: str) -> bool:
    expected = base64.b64encode(
        hmac.new(WEBHOOK_SECRET.encode(), payload, hashlib.sha256).digest()
    ).decode()
    return hmac.compare_digest(expected, signature)

@app.route('/webhook/order', methods=['POST'])
def handle_order_webhook():
    # 1. Verify signature
    signature = request.headers.get('X-Frappe-Webhook-Signature')
    if signature and not verify_signature(request.data, signature):
        logger.warning('Invalid webhook signature')
        return jsonify({'error': 'Invalid signature'}), 401

    # 2. Parse data
    try:
        data = request.json
        doctype = data.get('doctype')
        docname = data.get('name')
        doc_data = data.get('data', {})
    except Exception as e:
        logger.error(f'Failed to parse: {e}')
        return jsonify({'error': 'Invalid payload'}), 400

    # 3. Process (keep fast — queue long operations)
    logger.info(f'Webhook: {doctype}/{docname}')
    try:
        if doctype == 'Sales Order':
            process_sales_order(docname, doc_data)
    except Exception as e:
        logger.error(f'Processing failed: {e}')
        # Return 200 anyway to prevent endless retries

    # 4. Return quickly
    return jsonify({'status': 'received'}), 200
```

---

## Best Practices

1. **ALWAYS** set a Webhook Secret and verify HMAC signatures
2. **ALWAYS** return quickly (< 30 seconds) — queue long operations
3. **ALWAYS** return HTTP 200 even on processing errors (prevents endless retries)
4. **ALWAYS** implement idempotent operations (same webhook may arrive multiple times)
5. **ALWAYS** log webhook payloads for debugging
6. **NEVER** put sensitive data in webhook payloads without encryption
7. **NEVER** rely on webhook delivery order
8. **NEVER** perform synchronous heavy operations in webhook handlers

---

## Webhook Debugging

### In Frappe

1. **Webhook Request Log**: Shows all sent webhooks with request/response details
2. **Error Log**: Shows failed webhook requests
3. Enable/disable individual webhooks without deleting configuration

### Testing Locally

```bash
# Expose local server with ngrok
ngrok http 5000

# Simulate webhook
curl -X POST "http://localhost:5000/webhook/order" \
  -H "Content-Type: application/json" \
  -d '{"doctype":"Sales Order","name":"SO-00001","data":{"status":"Draft"}}'
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-api/references/webhooks-reference.md*
