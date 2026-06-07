---
title: Frappe Ref — Frappe Core: Database: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'database', 'reference', 'anti-patterns']
source: Impertio
---

# Database Anti-Patterns

> Common database mistakes in Frappe development and their corrections. Every anti-pattern includes a CORRECT alternative.

---

## 1. SQL Injection — CRITICAL SECURITY VULNERABILITY

### NEVER: String Formatting in SQL
```python
# ❌ ALL of these are SQL injection vulnerabilities
user_input = "admin'; DROP TABLE tabUser; --"

frappe.db.sql(f"SELECT * FROM `tabUser` WHERE name = '{user_input}'")
frappe.db.sql("SELECT * FROM `tabUser` WHERE name = '%s'" % user_input)
frappe.db.sql("SELECT * FROM `tabUser` WHERE name = " + user_input)
```

### ALWAYS: Parameterized Queries
```python
# ✅ SAFE — parameterized query
frappe.db.sql(
    "SELECT * FROM `tabUser` WHERE name = %(name)s",
    {'name': user_input}
)

# ✅ SAFE — ORM (always parameterized internally)
frappe.get_all('User', filters={'name': user_input})

# ✅ SAFE — Query Builder (always parameterized)
User = frappe.qb.DocType('User')
frappe.qb.from_(User).select('*').where(User.name == user_input).run()
```

**RULE**: EVERY `frappe.db.sql()` call MUST use `%(param)s` placeholders with a dict. No exceptions.

---

## 2. N+1 Query Problem

### NEVER: Query Inside a Loop
```python
# ❌ N+1 queries — fetches one document per iteration
def get_order_details(order_names):
    results = []
    for name in order_names:
        order = frappe.get_doc('Sales Order', name)        # N queries
        customer = frappe.get_doc('Customer', order.customer)  # N more queries
        results.append({'order': order.name, 'customer': customer.customer_name})
    return results
```

### ALWAYS: Batch Fetch
```python
# ✅ Two queries total, regardless of list size
def get_order_details(order_names):
    orders = frappe.get_all('Sales Order',
        filters={'name': ['in', order_names]},
        fields=['name', 'customer', 'grand_total']
    )
    customer_names = list(set(o.customer for o in orders))
    customers = {c.name: c for c in frappe.get_all('Customer',
        filters={'name': ['in', customer_names]},
        fields=['name', 'customer_name']
    )}
    return [{'order': o.name, 'customer': customers[o.customer].customer_name} for o in orders]
```

---

## 3. Commit Inside Controller Hooks

### NEVER: Manual Commit in Hooks
```python
# ❌ Breaks framework transaction management
class SalesInvoice(Document):
    def validate(self):
        self.calculate_totals()
        frappe.db.commit()       # NEVER do this

    def on_submit(self):
        self.create_gl_entries()
        frappe.db.commit()       # NEVER do this
```

### ALWAYS: Let the Framework Handle Commits
```python
# ✅ Framework commits after successful request
class SalesInvoice(Document):
    def validate(self):
        self.calculate_totals()
        # No commit — framework handles it

    def on_submit(self):
        self.create_gl_entries()
        # No commit — framework handles it
```

**Exception**: `frappe.db.commit()` is acceptable ONLY in background jobs or standalone scripts where you explicitly manage transactions.

---

## 4. SELECT * in Production

### NEVER: Fetch All Fields
```python
# ❌ Fetches ALL columns including large TEXT/LONGTEXT fields
docs = frappe.get_all('Sales Invoice', fields=['*'])
frappe.db.sql("SELECT * FROM `tabSales Invoice`")
```

### ALWAYS: Specify Needed Fields
```python
# ✅ Fetch only what you need
docs = frappe.get_all('Sales Invoice',
    fields=['name', 'customer', 'grand_total', 'status']
)
```

---

## 5. No Pagination

### NEVER: Fetch Unlimited Records
```python
# ❌ Can return millions of records — crashes server
all_logs = frappe.get_all('Error Log')
frappe.db.sql("SELECT * FROM `tabError Log`")
```

### ALWAYS: Limit Results
```python
# ✅ Always use page_length
logs = frappe.get_all('Error Log',
    fields=['name', 'error', 'creation'],
    order_by='creation desc',
    page_length=100
)

# ✅ Always use LIMIT in SQL
frappe.db.sql("""
    SELECT name, error FROM `tabError Log`
    ORDER BY creation DESC LIMIT 100
""")
```

---

## 6. Overusing ignore Flags

### NEVER: Ignore Everything
```python
# ❌ Bypasses all safety checks — data integrity at risk
doc.insert(
    ignore_permissions=True,
    ignore_mandatory=True,
    ignore_links=True
)
```

### ALWAYS: Use Minimum Required Flags
```python
# ✅ Only bypass what you must, with documented reason
doc.flags.ignore_permissions = True  # Reason: system background job
doc.insert()
```

---

## 7. Using db_set/set_value for Business Logic

### NEVER: Direct DB Update for Stateful Changes
```python
# ❌ Skips validate, on_update, and all other hooks
def update_status(name, status):
    frappe.db.set_value('Task', name, 'status', status)
```

### ALWAYS: Use ORM for Business Logic
```python
# ✅ Triggers validate, on_update, permission checks
def update_status(name, status):
    doc = frappe.get_doc('Task', name)
    doc.status = status
    doc.save()
```

**`db_set`/`set_value` is acceptable ONLY for**: hidden fields, counters, timestamps, performance-critical background jobs.

---

## 8. No Error Handling in Batch Operations

### NEVER: Bare Loop Without Error Handling
```python
# ❌ One failure kills the entire batch
def process_data(items):
    for item in items:
        doc = frappe.get_doc('Item', item)
        doc.status = 'Processed'
        doc.save()
    frappe.db.commit()
```

### ALWAYS: Handle Errors Per Item
```python
# ✅ Process what you can, log what fails
def process_data(items):
    processed, errors = [], []

    for item in items:
        try:
            doc = frappe.get_doc('Item', item)
            doc.status = 'Processed'
            doc.save()
            processed.append(item)
        except frappe.DoesNotExistError:
            errors.append({'item': item, 'error': 'Not found'})
        except Exception as e:
            frappe.log_error(frappe.get_traceback(), f'Process Error: {item}')
            errors.append({'item': item, 'error': str(e)})

    return {'processed': len(processed), 'errors': errors}
```

---

## 9. Cache Without Invalidation

### NEVER: Unbounded Cache
```python
# ❌ Cache never cleared — returns stale data forever
@redis_cache
def get_settings():
    return frappe.get_doc('My Settings')
```

### ALWAYS: Set TTL or Explicit Invalidation
```python
# ✅ TTL ensures refresh
@redis_cache(ttl=3600)
def get_settings():
    return frappe.get_doc('My Settings')

# ✅ Explicit invalidation on change
class MySettings(Document):
    def on_update(self):
        get_settings.clear_cache()
```

---

## 10. Blocking Operations in Web Requests

### NEVER: Long-Running Code in Request Handler
```python
# ❌ Request times out — user sees error
@frappe.whitelist()
def process_all_invoices():
    invoices = frappe.get_all('Sales Invoice', filters={'status': 'Unpaid'})
    for inv in invoices:
        send_reminder_email(inv.name)  # Could take minutes
    return "Done"
```

### ALWAYS: Use Background Jobs for Heavy Work
```python
# ✅ Returns immediately, processes in background
@frappe.whitelist()
def process_all_invoices():
    frappe.enqueue(
        'myapp.tasks.send_reminders',
        queue='long',
        timeout=3600
    )
    return "Processing started"

def send_reminders():
    invoices = frappe.get_all('Sales Invoice', filters={'status': 'Unpaid'})
    for inv in invoices:
        send_reminder_email(inv.name)
        frappe.db.commit()  # Commit per iteration in background jobs
```

---

## 11. Using get_doc for Existence Checks

### NEVER: Try/Except with get_doc
```python
# ❌ Loads entire document just to check existence
try:
    doc = frappe.get_doc('User', email)
    exists = True
except:
    exists = False
```

### ALWAYS: Use frappe.db.exists()
```python
# ✅ Lightweight query, returns True/False
exists = frappe.db.exists('User', email)
exists = frappe.db.exists('User', {'email': email})
```

---

## 12. Wrong Table Names in Raw SQL

### NEVER: Missing `tab` Prefix
```python
# ❌ Table not found — Frappe prefixes all tables with `tab`
frappe.db.sql("SELECT * FROM Task")
frappe.db.sql("SELECT * FROM sales_invoice")
```

### ALWAYS: Use Backtick-Quoted `tab` Prefix
```python
# ✅ Correct table naming
frappe.db.sql("SELECT * FROM `tabTask`")
frappe.db.sql("SELECT * FROM `tabSales Invoice`")
```

**Format**: `` `tab{Exact DocType Name}` `` — including spaces, exact capitalization.

---

## 13. Using truncate Without Understanding Consequences

### NEVER: Truncate Without Awareness
```python
# ❌ DDL operation — auto-commits, CANNOT be rolled back
frappe.db.truncate('Important Table')
```

### ALWAYS: Use delete() for Rollback-Safe Deletion
```python
# ✅ DML operation — can be rolled back
frappe.db.delete('Error Log', {'creation': ['<', '2024-01-01']})
```

Use `truncate` ONLY for log/temp tables where rollback is not needed and performance matters.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-database/references/anti-patterns.md*
