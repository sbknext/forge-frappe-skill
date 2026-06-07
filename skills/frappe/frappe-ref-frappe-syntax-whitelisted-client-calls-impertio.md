---
title: Frappe Ref — Frappe Syntax: Whitelisted: Client Calls (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'client-calls']
source: Impertio
---

# Client Calls Reference

All patterns for calling whitelisted methods from JavaScript and external clients.

## frappe.call() — Standalone APIs

### Promise-Based (ALWAYS Prefer)

```javascript
frappe.call({
    method: 'myapp.api.get_customer_summary',
    args: { customer: 'CUST-00001' }
}).then(r => {
    console.log(r.message);  // Return value is ALWAYS in r.message
}).catch(err => {
    console.error(err);
});
```

### With Loading Indicator

```javascript
frappe.call({
    method: 'myapp.api.process_data',
    args: { data: myData },
    freeze: true,
    freeze_message: __('Processing...')
}).then(r => {
    frappe.show_alert({ message: __('Done!'), indicator: 'green' });
});
```

### Async/Await

```javascript
async function getData() {
    try {
        let r = await frappe.call({
            method: 'myapp.api.get_data',
            args: { id: 123 }
        });
        return r.message;
    } catch (err) {
        frappe.show_alert({ message: __('Failed'), indicator: 'red' });
    }
}
```

### Shorthand Syntax

```javascript
// Two-argument shorthand
frappe.call('myapp.api.ping', { param: 'value' }).then(r => {
    console.log(r.message);
});
```

---

## frappe.call() Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `method` | string | required | Dotted path to whitelisted Python method |
| `args` | object | `{}` | Arguments passed to the method |
| `type` | string | `"POST"` | HTTP method: GET, POST, PUT, DELETE |
| `callback` | function | — | Success callback (legacy; prefer `.then()`) |
| `error` | function | — | Error callback (legacy; prefer `.catch()`) |
| `always` | function | — | Runs after success or error |
| `freeze` | boolean | `false` | Show full-screen loading overlay |
| `freeze_message` | string | — | Custom loading message |
| `async` | boolean | `true` | NEVER set to `false` — blocks browser UI |
| `btn` | jQuery | — | Button element to disable during request |

### Full Example

```javascript
frappe.call({
    method: 'myapp.api.complex_operation',
    type: 'POST',
    args: {
        customer: 'CUST-001',
        items: JSON.stringify([{item: 'ITEM-001', qty: 10}])
    },
    freeze: true,
    freeze_message: __('Processing...'),
    btn: $(this),
    callback: function(r) {
        if (r.message && r.message.success) {
            frappe.msgprint(__('Completed!'));
        }
    },
    error: function(r) {
        frappe.msgprint({
            title: __('Error'),
            indicator: 'red',
            message: __('Operation failed')
        });
    },
    always: function() {
        console.log('Request completed');
    }
});
```

---

## frm.call() — Controller Methods

For methods defined on a Document class controller.

### Basic Usage

```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        frm.add_custom_button(__('Calculate Tax'), () => {
            frm.call('calculate_taxes', { include_shipping: true })
                .then(r => {
                    if (r.message) {
                        frm.set_value('tax_amount', r.message.tax_amount);
                    }
                });
        });
    }
});
```

### With Loading Indicator

```javascript
frm.call({
    method: 'send_email',
    args: { template: 'order_confirmation' },
    freeze: true,
    freeze_message: __('Sending email...')
}).then(r => {
    frappe.show_alert({ message: __('Email sent'), indicator: 'green' });
});
```

### With Document Reload

```javascript
frm.call('approve_order').then(r => {
    if (r.message) {
        frm.reload_doc();  // Refresh form with updated data
    }
});
```

### Server-Side Requirement

The controller method MUST have `@frappe.whitelist()`:

```python
class SalesOrder(Document):
    @frappe.whitelist()
    def calculate_taxes(self, include_shipping=False):
        tax = self.grand_total * 0.21
        if include_shipping:
            tax += 50
        return {"tax_amount": tax}
```

---

## REST API — External Clients

### Authentication Methods

**Token Auth (ALWAYS use for external integrations):**
```bash
curl -H "Authorization: token api_key:api_secret" \
     -H "Content-Type: application/json" \
     https://site.com/api/method/myapp.api.get_data
```

**Bearer Token (OAuth):**
```bash
curl -H "Authorization: Bearer access_token" \
     -H "Content-Type: application/json" \
     https://site.com/api/method/myapp.api.get_data
```

**Session Auth (login first):**
```bash
# Login
curl -c cookies.txt -X POST https://site.com/api/method/login \
     -d '{"usr":"user@example.com","pwd":"password"}'

# Use session
curl -b cookies.txt https://site.com/api/method/myapp.api.get_data
```

### GET Request

```bash
curl -H "Authorization: token api_key:api_secret" \
     "https://site.com/api/method/myapp.api.get_status?order_id=SO-00001"
```

### POST Request

```bash
curl -H "Authorization: token api_key:api_secret" \
     -H "Content-Type: application/json" \
     -X POST https://site.com/api/method/myapp.api.create_order \
     -d '{"customer":"CUST-001","items":[{"item":"ITEM-001","qty":5}]}'
```

### Python Client

```python
import requests

API_URL = "https://site.com"
HEADERS = {
    "Authorization": "token api_key:api_secret",
    "Content-Type": "application/json"
}

# GET
r = requests.get(
    f"{API_URL}/api/method/myapp.api.get_status",
    params={"order_id": "SO-00001"},
    headers=HEADERS,
    timeout=30
)
data = r.json()["message"]

# POST
r = requests.post(
    f"{API_URL}/api/method/myapp.api.create_order",
    json={"customer": "CUST-001"},
    headers=HEADERS,
    timeout=30
)
result = r.json()["message"]
```

---

## Fetch API (Browser)

### POST with CSRF Token

```javascript
fetch('/api/method/myapp.api.create_item', {
    method: 'POST',
    headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'X-Frappe-CSRF-Token': frappe.csrf_token
    },
    body: JSON.stringify({ item_name: 'New Item' })
})
.then(r => r.json())
.then(data => console.log(data.message))
.catch(err => console.error(err));
```

### GET Request

```javascript
fetch('/api/method/myapp.api.get_status?order_id=SO-00001', {
    headers: {
        'Accept': 'application/json',
        'X-Frappe-CSRF-Token': frappe.csrf_token
    }
})
.then(r => r.json())
.then(data => console.log(data.message));
```

ALWAYS include `X-Frappe-CSRF-Token` header for browser-based fetch calls.

---

## Endpoint URL Patterns

| Type | URL Pattern |
|------|-------------|
| Standalone method | `/api/method/myapp.module.function` |
| Controller method (via frm.call) | `/api/method/run_doc_method` (internal) |
| Document resource | `/api/resource/{DocType}/{name}` |

### Mapping Examples

```
# myapp/api.py -> def get_customers()
/api/method/myapp.api.get_customers

# myapp/utils/helpers.py -> def calculate()
/api/method/myapp.utils.helpers.calculate

# myapp/myapp/doctype/invoice/invoice.py -> class Invoice.send_email()
Called via frm.call('send_email') -> /api/method/run_doc_method
```

---

## Error Handling (Client Side)

### ALWAYS Handle Errors

```javascript
frappe.call({
    method: 'myapp.api.risky_operation',
    args: { data: myData }
}).then(r => {
    if (r.message && r.message.success) {
        frappe.show_alert({ message: __('Success'), indicator: 'green' });
    }
}).catch(err => {
    frappe.show_alert({ message: __('Operation failed'), indicator: 'red' });
    console.error(err);
});
```

### Batch Requests

```javascript
async function batchProcess(items) {
    const results = [];
    for (const item of items) {
        const r = await frappe.call({
            method: 'myapp.api.process_item',
            args: { item: item }
        });
        results.push(r.message);
    }
    return results;
}
```

### Debounced Search

```javascript
let searchTimeout;
function searchCustomers(query) {
    clearTimeout(searchTimeout);
    searchTimeout = setTimeout(() => {
        frappe.call({
            method: 'myapp.api.search_customers',
            args: { query: query }
        }).then(r => updateResults(r.message));
    }, 300);
}
```

---

## Key Rules

1. **ALWAYS use `JSON.stringify()`** for complex args (arrays, objects)
2. **ALWAYS check `r.message`** before accessing — response can be null
3. **NEVER set `async: false`** — it blocks the browser UI thread
4. **ALWAYS include CSRF token** for direct fetch/XMLHttpRequest calls
5. **ALWAYS handle errors** — use `.catch()` or `error` callback
6. **ALWAYS use `freeze: true`** for long-running operations

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/client-calls.md*
