---
title: Frappe Ref — Frappe Errors: Database: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'database', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns — Database Error Handling

Common mistakes to avoid when handling database errors in Frappe/ERPNext.

---

## 1. SQL Injection via String Formatting

### Problem
```python
# ALL of these are SQL INJECTION vulnerabilities
query = f"SELECT * FROM `tabCustomer` WHERE name = '{customer_name}'"
query = "SELECT * FROM `tabCustomer` WHERE name = '{}'".format(customer_name)
query = "SELECT * FROM `tabCustomer` WHERE name = '%s'" % customer_name
```

### Fix
```python
# ALWAYS use named parameters with dict
frappe.db.sql(
    "SELECT * FROM `tabCustomer` WHERE name = %(name)s",
    {"name": customer_name},
    as_dict=True
)

# Or use ORM — no injection risk
frappe.db.get_value("Customer", customer_name, "*", as_dict=True)
```

**Why**: String formatting allows SQL injection. ALWAYS use `%(name)s` with dict params.

---

## 2. Using %s Instead of %(name)s

### Problem
```python
# Positional %s is fragile and error-prone
frappe.db.sql(
    "SELECT * FROM `tabItem` WHERE name = %s AND warehouse = %s",
    ("ITEM-001", "Main")
)
# Easy to get parameter order wrong!
```

### Fix
```python
# Named parameters are self-documenting and order-independent
frappe.db.sql(
    "SELECT * FROM `tabItem` WHERE name = %(name)s AND warehouse = %(wh)s",
    {"name": "ITEM-001", "wh": "Main"},
    as_dict=True
)
```

**Why**: Named parameters prevent order-dependent bugs and are clearer to read.

---

## 3. Not Checking get_value() for None

### Problem
```python
credit = frappe.db.get_value("Customer", "CUST-001", "credit_limit")
if credit > 1000:  # TypeError if customer not found (None > 1000)
    apply_discount()
```

### Fix
```python
credit = frappe.db.get_value("Customer", "CUST-001", "credit_limit")
if credit is None:
    frappe.throw(_("Customer not found"))
credit = credit or 0
if credit > 1000:
    apply_discount()
```

**Why**: `get_value()` returns None when record not found. ALWAYS check before using.

---

## 4. Not Checking Existence Before get_doc

### Problem
```python
# Crashes with DoesNotExistError
doc = frappe.get_doc("Customer", customer_name)
doc.update(data)
doc.save()
```

### Fix
```python
if not frappe.db.exists("Customer", customer_name):
    frappe.throw(_("Customer not found"))
doc = frappe.get_doc("Customer", customer_name)
doc.update(data)
doc.save()
```

**Why**: `get_doc()` raises DoesNotExistError. Check first or catch the exception.

---

## 5. Ignoring DuplicateEntryError on Insert

### Problem
```python
doc = frappe.get_doc({"doctype": "Customer", **data})
doc.insert()  # Crashes on duplicate!
```

### Fix
```python
try:
    doc = frappe.get_doc({"doctype": "Customer", **data})
    doc.insert()
except frappe.DuplicateEntryError:
    existing = frappe.db.get_value("Customer", {"customer_name": data.get("customer_name")})
    return {"name": existing, "existing": True}
```

**Why**: Unique constraints cause DuplicateEntryError. Handle it on every insert.

---

## 6. Assuming db.set_value Always Works

### Problem
```python
frappe.db.set_value("Customer", "NONEXISTENT", "synced", 1)
return "Success"  # LIE — nothing was updated
```

### Fix
```python
if not frappe.db.exists("Customer", customer_name):
    frappe.throw(_("Customer not found"))
frappe.db.set_value("Customer", customer_name, "synced", 1)
```

**Why**: `db.set_value()` silently does nothing if record doesn't exist.

---

## 7. Committing in Controller Hooks

### Problem
```python
class SalesOrder(Document):
    def validate(self):
        self.calculate_totals()
        frappe.db.commit()  # BREAKS transaction!

    def on_update(self):
        frappe.db.commit()  # ALSO WRONG
```

### Fix
```python
class SalesOrder(Document):
    def validate(self):
        self.calculate_totals()
        # Framework handles commit

    def on_update(self):
        self.update_linked()
        # Framework handles commit
```

**Why**: Manual commits break transaction management. Frappe auto-commits after web requests.

---

## 8. Missing Commit in Background Jobs

### Problem
```python
def background_sync():
    for item in frappe.get_all("Item", limit=100):
        frappe.db.set_value("Item", item.name, "synced", 1)
    # ALL CHANGES LOST — no auto-commit in background!
```

### Fix
```python
def background_sync():
    for item in frappe.get_all("Item", limit=100):
        frappe.db.set_value("Item", item.name, "synced", 1)
    frappe.db.commit()  # REQUIRED
```

**Why**: Background jobs and scheduler tasks have no auto-commit.

---

## 9. Swallowing Database Errors

### Problem
```python
try:
    doc = frappe.get_doc("Customer", name)
    doc.save()
except Exception:
    pass  # Silent failure — impossible to debug
```

### Fix
```python
try:
    doc = frappe.get_doc("Customer", name)
    doc.save()
except frappe.DoesNotExistError:
    frappe.throw(_("Customer not found"))
except frappe.TimestampMismatchError:
    frappe.throw(_("Modified by another user. Refresh and retry."))
except frappe.ValidationError as e:
    frappe.throw(str(e))
except Exception:
    frappe.log_error(frappe.get_traceback(), "Customer Save Error")
    frappe.throw(_("An error occurred"))
```

**Why**: Catch specific exceptions first. NEVER silently swallow errors.

---

## 10. Not Handling Empty Query Results

### Problem
```python
result = frappe.db.sql("SELECT name FROM `tabSales Invoice` WHERE customer = %s LIMIT 1", customer)
return result[0][0]  # IndexError if no results!
```

### Fix
```python
result = frappe.db.sql("SELECT name FROM `tabSales Invoice` WHERE customer = %(c)s LIMIT 1",
                       {"c": customer})
return result[0][0] if result else None
```

**Why**: Empty result sets cause IndexError. ALWAYS check before accessing.

---

## 11. N+1 Query Pattern

### Problem
```python
def get_details(names):
    details = []
    for name in names:
        doc = frappe.get_doc("Customer", name)  # N queries!
        details.append(doc.as_dict())
    return details
```

### Fix
```python
def get_details(names):
    return frappe.get_all(
        "Customer",
        filters={"name": ["in", names]},
        fields=["name", "customer_name", "credit_limit"]
    )  # 1 query!
```

**Why**: N+1 queries are extremely slow. ALWAYS batch fetch.

---

## 12. No Limit on Queries

### Problem
```python
all_invoices = frappe.get_all("Sales Invoice")  # Could return millions!
```

### Fix
```python
invoices = frappe.get_all("Sales Invoice", limit=100)
# Or paginate:
invoices = frappe.get_all("Sales Invoice", limit_start=0, limit_page_length=100)
```

**Why**: Unbounded queries cause memory exhaustion and timeouts.

---

## 13. Exposing Database Errors to Users

### Problem
```python
try:
    return frappe.db.sql(query, filters)
except Exception as e:
    frappe.throw(str(e))  # Exposes SQL details!
```

### Fix
```python
try:
    return frappe.db.sql(query, filters)
except frappe.db.InternalError:
    frappe.log_error(frappe.get_traceback(), "Query Error")
    frappe.throw(_("Database error. Please contact support."))
```

**Why**: Database error messages can expose table names, column names, and SQL structure.

---

## 14. Race Condition on Get-or-Create

### Problem
```python
def get_or_create(name):
    if not frappe.db.exists("Customer", name):
        doc = frappe.get_doc({"doctype": "Customer", "customer_name": name})
        doc.insert()  # DuplicateEntryError — someone else created it!
    return frappe.get_doc("Customer", name)
```

### Fix
```python
def get_or_create(name):
    if not frappe.db.exists("Customer", name):
        try:
            doc = frappe.get_doc({"doctype": "Customer", "customer_name": name})
            doc.insert()
        except frappe.DuplicateEntryError:
            pass  # Race condition — someone else created it
    return frappe.get_doc("Customer", name)
```

**Why**: Between `exists()` and `insert()`, another process can create the record.

---

## 15. Not Handling Concurrent Edits

### Problem
```python
doc = frappe.get_doc("Customer", name)
doc.update(data)
doc.save()  # TimestampMismatchError if modified by another user!
```

### Fix
```python
try:
    doc = frappe.get_doc("Customer", name)
    doc.update(data)
    doc.save()
except frappe.TimestampMismatchError:
    frappe.throw(_("Document modified. Please refresh and try again."))
```

**Why**: Concurrent edits cause TimestampMismatchError. Handle gracefully in APIs.

---

## 16. Using get_doc When get_value Suffices

### Problem
```python
# Loads ENTIRE document just for one field
doc = frappe.get_doc("Customer", name)
return doc.credit_limit
```

### Fix
```python
# Only fetches the needed field
return frappe.db.get_value("Customer", name, "credit_limit") or 0
```

**Why**: `get_doc()` loads entire document with children. Use `get_value()` for single fields.

---

## 17. Catching Generic Exception for All DB Errors

### Problem
```python
try:
    frappe.delete_doc("Customer", name)
except Exception as e:
    frappe.throw(str(e))  # No specific handling
```

### Fix
```python
try:
    frappe.delete_doc("Customer", name)
except frappe.DoesNotExistError:
    frappe.throw(_("Customer not found"))
except frappe.LinkExistsError:
    frappe.throw(_("Cannot delete — linked documents exist"))
except Exception:
    frappe.log_error(frappe.get_traceback(), "Delete Error")
    frappe.throw(_("Delete failed. Contact support."))
```

**Why**: Specific exceptions allow specific error messages and recovery strategies.

---

## Quick Checklist: Database Code Review

Before deploying:

- [ ] All SQL uses `%(name)s` with dict (no string formatting)
- [ ] `get_value()` results checked for None
- [ ] Existence checked before `get_doc()` (or exception caught)
- [ ] `DuplicateEntryError` handled on every `insert()`
- [ ] `TimestampMismatchError` handled on `save()` in APIs
- [ ] `db.set_value()` preceded by existence check
- [ ] No `frappe.db.commit()` in controller hooks / doc_events
- [ ] `frappe.db.commit()` in background/scheduler tasks
- [ ] Database errors logged, not swallowed
- [ ] Empty results handled (no blind array access)
- [ ] Queries have limits / pagination
- [ ] Specific exceptions caught before generic Exception
- [ ] Database errors not exposed to users
- [ ] Race conditions handled on get-or-create
- [ ] `get_value()` used instead of `get_doc()` for single fields

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-database/references/anti-patterns.md*
