---
title: Frappe Ref — Frappe Core: Api: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'api', 'reference', 'anti-patterns']
source: Impertio
---

# API Anti-Patterns

> Common API mistakes and their correct alternatives.

---

## 1. No Permission Check in Whitelisted Method

```python
# WRONG — any authenticated user can delete
@frappe.whitelist()
def delete_customer(name):
    frappe.get_doc("Customer", name).delete()

# CORRECT — check permission first
@frappe.whitelist()
def delete_customer(name):
    if not frappe.has_permission("Customer", "delete"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    doc = frappe.get_doc("Customer", name)
    doc.delete()
    return {"status": "success"}
```

`@frappe.whitelist()` only checks authentication — it does NOT verify DocType permissions.

---

## 2. SQL Injection in Queries

```python
# WRONG — vulnerable to injection
@frappe.whitelist()
def search(term):
    return frappe.db.sql(f"SELECT * FROM tabCustomer WHERE name LIKE '%{term}%'")

# CORRECT — parameterized query
@frappe.whitelist()
def search(term):
    return frappe.db.sql(
        "SELECT name, customer_name FROM tabCustomer WHERE name LIKE %s",
        (f"%{term}%",), as_dict=True)
```

---

## 3. Hardcoded Credentials

```python
# WRONG
API_KEY = "abc123"
API_SECRET = "secret456"

# CORRECT — from config or environment
api_key = frappe.conf.get("external_api_key")
api_secret = frappe.conf.get("external_api_secret")
if not api_key or not api_secret:
    frappe.throw(_("API credentials not configured"))
```

---

## 4. No Input Validation

```python
# WRONG — trusts all input
@frappe.whitelist()
def create_order(customer, amount):
    order = frappe.new_doc("Sales Order")
    order.customer = customer
    order.grand_total = amount
    order.insert()

# CORRECT — validate everything
@frappe.whitelist()
def create_order(customer, amount):
    if not customer:
        frappe.throw(_("Customer is required"))
    if not frappe.db.exists("Customer", customer):
        frappe.throw(_("Customer {0} does not exist").format(customer))
    try:
        amount = float(amount)
    except (TypeError, ValueError):
        frappe.throw(_("Amount must be a number"))
    if amount <= 0:
        frappe.throw(_("Amount must be positive"))

    order = frappe.new_doc("Sales Order")
    order.customer = customer
    order.grand_total = amount
    order.insert()
    return {"name": order.name}
```

---

## 5. No Error Handling

```python
# WRONG — unhandled exceptions expose stack traces
@frappe.whitelist()
def process(docname):
    doc = frappe.get_doc("Customer", docname)
    doc.delete()
    return "done"

# CORRECT — structured error handling
@frappe.whitelist()
def process(docname):
    try:
        doc = frappe.get_doc("Customer", docname)
        doc.check_permission("delete")
        doc.delete()
        return {"status": "success"}
    except frappe.DoesNotExistError:
        frappe.throw(_("Customer not found"))
    except frappe.PermissionError:
        raise
    except Exception:
        frappe.log_error(title="API Error")
        frappe.throw(_("Operation failed"))
```

---

## 6. allow_guest on State-Changing Endpoints

```python
# WRONG — anyone can modify data
@frappe.whitelist(allow_guest=True)
def update_status(name, status):
    frappe.db.set_value("Order", name, "status", status)

# CORRECT — require authentication
@frappe.whitelist()
def update_status(name, status):
    doc = frappe.get_doc("Order", name)
    doc.check_permission("write")
    doc.status = status
    doc.save()
```

NEVER use `allow_guest=True` on endpoints that modify data.

---

## 7. No Pagination for Large Datasets

```python
# WRONG — could return millions of records
@frappe.whitelist()
def get_all_invoices():
    return frappe.get_all("Sales Invoice")

# CORRECT — enforced pagination
@frappe.whitelist()
def get_invoices(page=0, page_size=20):
    page_size = min(int(page_size), 100)  # Enforce max
    return frappe.get_all("Sales Invoice",
        fields=["name", "customer", "grand_total"],
        limit_start=int(page) * page_size,
        limit_page_length=page_size,
        order_by="modified desc")
```

---

## 8. Synchronous Heavy Operations

```python
# WRONG — blocks worker for minutes
@frappe.whitelist()
def generate_report():
    return expensive_computation()  # Timeout risk

# CORRECT — queue background job
@frappe.whitelist()
def generate_report():
    frappe.enqueue("myapp.tasks.expensive_computation",
                   queue="long", timeout=1800)
    return {"status": "queued", "message": "Processing started"}
```

---

## 9. No Timeout on External Calls

```python
# WRONG — can hang indefinitely
response = requests.get(external_url)

# CORRECT — always set timeout
response = requests.get(external_url, timeout=30)
```

---

## 10. Sensitive Data in Logs

```python
# WRONG — credentials in logs
frappe.logger().info(f"Login: {username}:{password}")

# CORRECT — only non-sensitive info
frappe.logger().info(f"Login attempt for: {username}")
```

---

## 11. Inconsistent Response Format

```python
# WRONG — sometimes string, sometimes dict
@frappe.whitelist()
def get_data(name):
    if not name:
        return "Error: name required"  # String
    return frappe.get_doc("Customer", name).as_dict()  # Dict

# CORRECT — consistent format with frappe.throw for errors
@frappe.whitelist()
def get_data(name):
    if not name:
        frappe.throw(_("Name is required"))
    return {"status": "success", "data": frappe.get_doc("Customer", name).as_dict()}
```

---

## 12. Using Administrator API Keys

```python
# WRONG — overprivileged API access
# Using Administrator's API key for integration

# CORRECT — dedicated API user
# 1. Create "API User" role with ONLY required permissions
# 2. Create dedicated user with that role
# 3. Generate API keys for that user
# 4. Use those restricted keys for integration
```

---

## Pre-Deploy Checklist

- [ ] Permission check present in every whitelisted method?
- [ ] All SQL queries use parameterized `%s` placeholders?
- [ ] Input validation complete for all parameters?
- [ ] Error handling with try/except implemented?
- [ ] Sensitive data NOT logged?
- [ ] Response format consistent across all endpoints?
- [ ] Rate limiting applied where needed?
- [ ] Pagination enforced for list endpoints?
- [ ] Credentials loaded from config, NOT hardcoded?
- [ ] Timeout set on all external HTTP calls?
- [ ] `allow_guest` NOT used on state-changing endpoints?
- [ ] Dedicated API user (NOT Administrator) for integrations?

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-api/references/anti-patterns.md*
