---
title: Frappe Ref — Frappe Core: Api: Rpc Api Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'api', 'reference', 'rpc-api-reference']
source: Impertio
---

# RPC API Reference

> Complete reference for Frappe Remote Procedure Calls via whitelisted methods.

---

## Endpoint Structure

```
GET/POST /api/method/{dotted.path.to.function}
```

The function MUST be decorated with `@frappe.whitelist()`.

---

## HTTP Methods

| Method | Use For | Auto Commit |
|--------|---------|-------------|
| GET | Read-only operations | No |
| POST | State-changing operations | Yes |

**ALWAYS** use GET for queries, POST for mutations.

---

## Writing Whitelisted Methods

### Basic Pattern

```python
import frappe

@frappe.whitelist()
def get_customer_balance(customer):
    """GET /api/method/myapp.api.get_customer_balance?customer=CUST-001"""
    balance = frappe.db.get_value("Sales Invoice",
        {"customer": customer, "docstatus": 1},
        "sum(outstanding_amount)") or 0
    return {"customer": customer, "balance": balance}
```

### With Input Validation

```python
@frappe.whitelist(methods=["POST"])
def create_payment(customer: str, amount: float, payment_type: str = "Receive"):
    if not customer:
        frappe.throw(_("Customer is required"))
    if not frappe.db.exists("Customer", customer):
        frappe.throw(_("Customer {0} does not exist").format(customer))

    amount = float(amount)
    if amount <= 0:
        frappe.throw(_("Amount must be positive"))

    if not frappe.has_permission("Payment Entry", "create"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)

    pe = frappe.new_doc("Payment Entry")
    pe.payment_type = payment_type
    pe.party_type = "Customer"
    pe.party = customer
    pe.paid_amount = amount
    pe.insert()
    return pe.name
```

---

## Decorator Options

| Option | Effect | Version |
|--------|--------|---------|
| `@frappe.whitelist()` | Requires authentication | All |
| `allow_guest=True` | No authentication needed | All |
| `methods=["POST"]` | Restrict HTTP methods | [v14+] |
| `methods=["GET", "POST"]` | Allow specific methods | [v14+] |
| `xss_safe=True` | Skip XSS escaping on response | All |

---

## API Calls

### Via cURL

```bash
# GET for read-only
curl -X GET "https://erp.example.com/api/method/myapp.api.get_customer_balance?customer=CUST-001" \
  -H "Authorization: token api_key:api_secret" \
  -H "Accept: application/json"

# POST for state-changing
curl -X POST "https://erp.example.com/api/method/myapp.api.create_payment" \
  -H "Authorization: token api_key:api_secret" \
  -H "Content-Type: application/json" \
  -d '{"customer": "CUST-001", "amount": 500}'
```

### Via Python

```python
# GET
response = requests.get(
    f'{BASE_URL}/api/method/myapp.api.get_customer_balance',
    params={'customer': 'CUST-001'}, headers=headers)

# POST
response = requests.post(
    f'{BASE_URL}/api/method/myapp.api.create_payment',
    json={'customer': 'CUST-001', 'amount': 500}, headers=headers)
```

---

## Response Structure

### Success

```json
{"message": "return_value_from_function"}
```

The return value is ALWAYS wrapped in a `message` key. Can be string, dict, list, or number.

### Error

```json
{
    "exc_type": "ValidationError",
    "exc": "Traceback...",
    "_server_messages": "[{\"message\": \"Error details\"}]"
}
```

---

## Client-Side Calls (JavaScript)

### frappe.xcall (RECOMMENDED)

```javascript
// Async/await — cleanest syntax
const result = await frappe.xcall('myapp.api.get_customer_balance', {
    customer: 'CUST-001'
});
console.log(result.balance);

// With error handling
try {
    const name = await frappe.xcall('myapp.api.create_payment', {
        customer: 'CUST-001', amount: 500
    });
    frappe.show_alert(__('Payment created: {0}', [name]));
} catch (e) {
    frappe.msgprint(__('Payment failed'));
}
```

### frappe.call (Callback/Promise)

```javascript
// Promise pattern
frappe.call({
    method: 'myapp.api.get_customer_balance',
    args: {customer: 'CUST-001'},
    freeze: true,
    freeze_message: __('Loading...')
}).then(r => console.log(r.message));
```

| Option | Type | Description |
|--------|------|-------------|
| `method` | string | Python method dotted path |
| `args` | object | Arguments to pass |
| `callback` | function | Success callback |
| `error` | function | Error callback |
| `async` | bool | Async call (default: true) |
| `freeze` | bool | Freeze UI during call |
| `freeze_message` | string | Message shown during freeze |
| `btn` | jQuery | Button to disable during call |

### frm.call (Document Context)

```javascript
frm.call('get_linked_doc', {throw_if_missing: true})
    .then(r => console.log(r.message));
```

Requires controller method with `@frappe.whitelist()`:

```python
class MyDocType(Document):
    @frappe.whitelist()
    def get_linked_doc(self, throw_if_missing=False):
        return frappe.get_doc(self.reference_type, self.reference_name)
```

---

## Standard frappe.client Methods

These built-in methods provide CRUD without writing custom endpoints:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `frappe.client.get_value` | POST | Get single field value |
| `frappe.client.get_list` | POST | List with filters and fields |
| `frappe.client.get` | POST | Get full document |
| `frappe.client.insert` | POST | Create new document |
| `frappe.client.save` | POST | Update existing document |
| `frappe.client.delete` | POST | Delete document |
| `frappe.client.submit` | POST | Submit document |
| `frappe.client.cancel` | POST | Cancel document |
| `frappe.client.get_count` | POST | Count documents |

### Examples

```bash
# Get value
POST /api/method/frappe.client.get_value
{"doctype": "Customer", "filters": {"name": "CUST-001"}, "fieldname": "customer_name"}

# Get count
POST /api/method/frappe.client.get_count
{"doctype": "Sales Order", "filters": {"status": "Draft"}}

# Insert
POST /api/method/frappe.client.insert
{"doc": {"doctype": "Customer", "customer_name": "New", "customer_type": "Company"}}

# Submit
POST /api/method/frappe.client.submit
{"doc": {"doctype": "Sales Order", "name": "SO-00001"}}
```

---

## Run Document Method

Execute a whitelisted method on a specific document instance:

```bash
POST /api/method/run_doc_method
{"dt": "Sales Order", "dn": "SO-00001", "method": "get_taxes_and_charges"}
```

[v15+] Also available via v2 API:
```
POST /api/v2/document/Sales Order/SO-00001/method/get_taxes_and_charges
```

---

## Server Script API Type

Alternative to whitelisted Python methods — configured via UI:

1. Server Script > New > Script Type: "API"
2. API Method: `myapp.my_endpoint`
3. Becomes `/api/method/myapp.my_endpoint`
4. Enable Rate Limit (optional) [v15+]

```python
# In Server Script body
response = {"customer": frappe.form_dict.customer}
frappe.response["message"] = response
```

---

## Error Handling Pattern

```python
@frappe.whitelist()
def safe_operation(docname):
    try:
        doc = frappe.get_doc("Sales Order", docname)
        doc.check_permission("write")
        doc.submit()
        return {"success": True, "name": doc.name}
    except frappe.DoesNotExistError:
        frappe.throw(_("Document not found"), frappe.DoesNotExistError)
    except frappe.PermissionError:
        raise  # Re-raise permission errors as-is
    except Exception:
        frappe.log_error(title="API Error")
        frappe.throw(_("Operation failed. Please try again."))
```

---

## Permission Checks

**ALWAYS** check permissions in whitelisted methods:

```python
@frappe.whitelist()
def get_salary(employee):
    if not frappe.has_permission("Salary Slip", "read"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    return frappe.db.get_value("Salary Slip", {"employee": employee}, "gross_pay")
```

The `@frappe.whitelist()` decorator only ensures the user is authenticated — it does NOT check DocType permissions.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-api/references/rpc-api-reference.md*
