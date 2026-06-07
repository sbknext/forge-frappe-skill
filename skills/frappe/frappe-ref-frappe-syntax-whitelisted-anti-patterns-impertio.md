---
title: Frappe Ref — Frappe Syntax: Whitelisted: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns Reference

Common mistakes in whitelisted methods and their correct alternatives.

## Security Anti-Patterns

### NEVER: Skip Permission Check

```python
# WRONG — any logged-in user sees all salaries
@frappe.whitelist()
def get_all_salaries():
    return frappe.get_all("Salary Slip", fields=["*"])

# CORRECT — role-restricted
@frappe.whitelist()
def get_salaries():
    frappe.only_for("HR Manager")
    return frappe.get_all("Salary Slip", fields=["employee_name", "net_pay"])
```

### NEVER: Use User Input in Raw SQL

```python
# WRONG — SQL injection!
@frappe.whitelist()
def search(term):
    return frappe.db.sql(f"SELECT * FROM `tabCustomer` WHERE name LIKE '%{term}%'")

# CORRECT — parameterized query
@frappe.whitelist()
def search(term):
    return frappe.db.sql("""
        SELECT name, customer_name FROM `tabCustomer` WHERE name LIKE %(term)s
    """, {"term": f"%{term}%"}, as_dict=True)

# ALSO CORRECT — ORM method
@frappe.whitelist()
def search(term):
    return frappe.get_all(
        "Customer",
        filters={"name": ["like", f"%{term}%"]},
        fields=["name", "customer_name"]
    )
```

### NEVER: Use ignore_permissions Without Role Check

```python
# WRONG — anyone can create anything
@frappe.whitelist()
def create_anything(data):
    doc = frappe.get_doc(data)
    doc.insert(ignore_permissions=True)

# CORRECT — role check + fixed DocType
@frappe.whitelist()
def create_log(message):
    frappe.only_for("System Manager")
    doc = frappe.get_doc({
        "doctype": "Error Log",  # Fixed — not from user input
        "method": "API Log",
        "error": message
    })
    doc.insert(ignore_permissions=True)
```

### NEVER: Accept Guest Input Without Validation

```python
# WRONG — no validation on public endpoint
@frappe.whitelist(allow_guest=True, methods=["POST"])
def submit_form(data):
    doc = frappe.get_doc(data)
    doc.insert(ignore_permissions=True)

# CORRECT — validate everything
@frappe.whitelist(allow_guest=True, methods=["POST"])
@rate_limit(limit=5, seconds=300)
def submit_form(name, email, message):
    if not name or not email:
        frappe.throw(_("Name and email required"))
    if len(name) > 100 or len(message or "") > 5000:
        frappe.throw(_("Input too long"))
    name = frappe.utils.strip_html(name)
    message = frappe.utils.strip_html(message or "")
    doc = frappe.get_doc({
        "doctype": "Contact Form",  # Fixed DocType
        "name1": name, "email": email, "message": message
    })
    doc.insert(ignore_permissions=True)
    return {"success": True}
```

---

## Input Validation Mistakes

### NEVER: Assume Input Types

```python
# WRONG — crashes with wrong type
@frappe.whitelist()
def calculate(amount, rate):
    return amount * rate  # TypeError if amount="abc"

# CORRECT — explicit type conversion
@frappe.whitelist()
def calculate(amount, rate):
    try:
        amount = float(amount)
        rate = float(rate)
    except (TypeError, ValueError):
        frappe.throw(_("Amount and rate must be numbers"))
    if amount < 0 or rate < 0:
        frappe.throw(_("Values cannot be negative"))
    return amount * rate
```

### NEVER: Parse JSON Without Error Handling

```python
# WRONG — crashes with invalid JSON
@frappe.whitelist()
def process_data(data):
    import json
    parsed = json.loads(data)  # JSONDecodeError if invalid

# CORRECT — safe parsing
@frappe.whitelist()
def process_data(data):
    if isinstance(data, str):
        try:
            data = frappe.parse_json(data)
        except Exception:
            frappe.throw(_("Invalid JSON data"))
    if not isinstance(data, dict):
        frappe.throw(_("Data must be a JSON object"))
    return process(data)
```

### NEVER: Accept Unlimited List Input

```python
# WRONG — resource exhaustion
@frappe.whitelist()
def process_items(items):
    for item in items:  # Could be 1 million items
        heavy_operation(item)

# CORRECT — enforce limit
@frappe.whitelist()
def process_items(items):
    if isinstance(items, str):
        items = frappe.parse_json(items)
    if not isinstance(items, list):
        frappe.throw(_("Items must be a list"))
    if len(items) > 100:
        frappe.throw(_("Maximum 100 items allowed"))
    return [process(item) for item in items]
```

---

## Error Handling Mistakes

### NEVER: Expose Raw Exception Messages

```python
# WRONG — leaks internal information (DB paths, credentials)
except Exception as e:
    frappe.throw(str(e))

# CORRECT — generic message + server-side log
except Exception:
    frappe.log_error(frappe.get_traceback(), "API Error")
    frappe.throw(_("Operation failed. Please contact support."))
```

### NEVER: Ignore External Call Failures

```python
# WRONG — no timeout, no error handling
@frappe.whitelist()
def call_api(data):
    import requests
    response = requests.post(url, json=data)  # May hang forever
    return response.json()

# CORRECT — timeout + error handling
@frappe.whitelist()
def call_api(data):
    import requests
    try:
        response = requests.post(url, json=data, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.Timeout:
        frappe.throw(_("External service timeout"))
    except requests.ConnectionError:
        frappe.throw(_("Cannot connect to external service"))
    except requests.HTTPError:
        frappe.log_error(frappe.get_traceback(), "External API")
        frappe.throw(_("External service error"))
```

---

## Response Anti-Patterns

### NEVER: Return Sensitive Fields

```python
# WRONG — returns api_key, api_secret, reset_password_key
@frappe.whitelist()
def get_user_info(user):
    return frappe.get_doc("User", user).as_dict()

# CORRECT — only needed fields
@frappe.whitelist()
def get_user_info(user):
    doc = frappe.get_doc("User", user)
    return {
        "name": doc.name,
        "full_name": doc.full_name,
        "email": doc.email,
        "user_image": doc.user_image
    }
```

---

## Performance Anti-Patterns

### NEVER: N+1 Query Pattern

```python
# WRONG — 101 queries
@frappe.whitelist()
def get_orders_with_items():
    orders = frappe.get_all("Sales Order", limit=100)
    for order in orders:
        order["items"] = frappe.get_all(
            "Sales Order Item", filters={"parent": order.name}
        )
    return orders

# CORRECT — 2 queries
@frappe.whitelist()
def get_orders_with_items():
    orders = frappe.get_all("Sales Order", fields=["name", "customer"], limit=100)
    if orders:
        all_items = frappe.get_all(
            "Sales Order Item",
            filters={"parent": ["in", [o.name for o in orders]]},
            fields=["parent", "item_code", "qty", "amount"]
        )
        items_map = {}
        for item in all_items:
            items_map.setdefault(item.parent, []).append(item)
        for order in orders:
            order["items"] = items_map.get(order.name, [])
    return orders
```

### NEVER: Return Unbounded Results

```python
# WRONG — could return 100,000+ records
@frappe.whitelist()
def get_all_customers():
    return frappe.get_all("Customer")

# CORRECT — paginated with cap
@frappe.whitelist()
def get_customers(limit=20, offset=0):
    limit = min(int(limit), 100)
    data = frappe.get_all("Customer", limit_page_length=limit, limit_start=int(offset))
    total = frappe.db.count("Customer")
    return {"data": data, "total": total, "has_more": (int(offset) + limit) < total}
```

---

## Client-Side Anti-Patterns

### NEVER: Use Synchronous Calls

```javascript
// WRONG — blocks browser thread
frappe.call({ method: 'myapp.api.get_data', async: false });

// CORRECT — async with promise
frappe.call({ method: 'myapp.api.get_data' }).then(r => { /* handle */ });
```

### NEVER: Ignore Client-Side Errors

```javascript
// WRONG — silent failure
frappe.call({ method: 'myapp.api.risky_operation', args: { data: myData } });

// CORRECT — handle errors
frappe.call({
    method: 'myapp.api.risky_operation',
    args: { data: myData }
}).then(r => {
    frappe.show_alert({ message: __('Success'), indicator: 'green' });
}).catch(err => {
    frappe.show_alert({ message: __('Failed'), indicator: 'red' });
    console.error(err);
});
```

---

## Quick Security Checklist

| Check | Status |
|-------|--------|
| Permission check present | -- |
| Input types validated | -- |
| SQL queries parameterized | -- |
| Error messages generic (no internals) | -- |
| Response contains only needed fields | -- |
| `allow_guest` only with good reason + rate limit | -- |
| `ignore_permissions` only with role check + fixed DocType | -- |
| External calls have timeout | -- |
| List inputs have size limit | -- |
| Pagination for list endpoints | -- |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/anti-patterns.md*
