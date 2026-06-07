---
title: Frappe Ref — Frappe Errors: Database: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'database', 'reference', 'patterns']
source: Impertio
---

# Error Handling Patterns — Database Operations

Complete error handling patterns for Frappe/ERPNext database operations.

---

## Pattern 1: Document CRUD with Full Error Handling

```python
import frappe
from frappe import _

class SafeDocumentManager:
    """Reusable document manager with error handling."""

    @staticmethod
    def get(doctype, name, fields=None):
        """Get document — returns None if not found."""
        if not name:
            return None
        if not frappe.db.exists(doctype, name):
            return None
        if fields:
            return frappe.db.get_value(doctype, name, fields, as_dict=True)
        return frappe.get_doc(doctype, name)

    @staticmethod
    def get_or_throw(doctype, name):
        """Get document or throw user-friendly error."""
        if not name:
            frappe.throw(_("{0} name is required").format(doctype))
        try:
            return frappe.get_doc(doctype, name)
        except frappe.DoesNotExistError:
            frappe.throw(_("{0} '{1}' not found").format(doctype, name))

    @staticmethod
    def create(doctype, data, ignore_duplicates=False):
        """Create document with duplicate handling."""
        try:
            doc = frappe.get_doc({"doctype": doctype, **data})
            doc.insert()
            return {"success": True, "name": doc.name}
        except frappe.DuplicateEntryError:
            if ignore_duplicates:
                existing = frappe.db.get_value(doctype, data, "name")
                if existing:
                    return {"success": True, "name": existing, "existing": True}
            frappe.throw(_("{0} already exists").format(doctype))
        except frappe.MandatoryError as e:
            return {"success": False, "error": f"Missing field: {e}"}
        except frappe.ValidationError as e:
            return {"success": False, "error": str(e)}

    @staticmethod
    def update(doctype, name, updates):
        """Update with concurrent edit handling."""
        if not frappe.db.exists(doctype, name):
            frappe.throw(_("{0} '{1}' not found").format(doctype, name))
        try:
            doc = frappe.get_doc(doctype, name)
            doc.update(updates)
            doc.save()
            return {"success": True}
        except frappe.TimestampMismatchError:
            frappe.throw(_("Modified by another user. Please refresh."))
        except frappe.ValidationError as e:
            return {"success": False, "error": str(e)}

    @staticmethod
    def delete(doctype, name, force=False):
        """Delete with link checking."""
        if not frappe.db.exists(doctype, name):
            return {"success": True, "message": "Already deleted"}
        try:
            frappe.delete_doc(doctype, name, force=force)
            return {"success": True}
        except frappe.LinkExistsError:
            frappe.throw(_("Cannot delete — linked documents exist"))
```

---

## Pattern 2: Safe Get-or-Create (Race Condition Safe)

```python
def get_or_create(doctype, filters, defaults=None):
    """Get existing or create new — handles race conditions."""
    existing = frappe.db.get_value(doctype, filters, "name")
    if existing:
        return frappe.get_doc(doctype, existing)

    doc_data = {"doctype": doctype}
    doc_data.update(filters)
    if defaults:
        doc_data.update(defaults)

    try:
        doc = frappe.get_doc(doc_data)
        doc.insert()
        return doc
    except frappe.DuplicateEntryError:
        # Race condition: another process created it between check and insert
        existing = frappe.db.get_value(doctype, filters, "name")
        if existing:
            return frappe.get_doc(doctype, existing)
        raise  # Unexpected duplicate — re-raise
```

---

## Pattern 3: Batch Operations with Error Isolation

```python
def batch_create(doctype, records, batch_size=100):
    """Create documents in batches with per-record error handling."""
    results = {"created": 0, "duplicates": 0, "failed": 0, "errors": []}

    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        for idx, record in enumerate(batch, i + 1):
            try:
                doc = frappe.get_doc({"doctype": doctype, **record})
                doc.insert()
                results["created"] += 1
            except frappe.DuplicateEntryError:
                results["duplicates"] += 1
            except frappe.MandatoryError as e:
                results["failed"] += 1
                results["errors"].append({"row": idx, "error": f"Missing: {e}"})
            except frappe.ValidationError as e:
                results["failed"] += 1
                results["errors"].append({"row": idx, "error": str(e)[:200]})
            except Exception:
                results["failed"] += 1
                frappe.log_error(frappe.get_traceback(), f"Batch create row {idx}")

        frappe.db.commit()  # Commit per batch

    return results
```

---

## Pattern 4: Safe SQL Query Execution

```python
def safe_query(query, values=None, as_dict=True):
    """Execute SQL with error classification."""
    try:
        return frappe.db.sql(query, values or {}, as_dict=as_dict)
    except frappe.QueryTimeoutError:
        frappe.log_error(f"Timeout: {query[:200]}", "Query Timeout")
        frappe.throw(_("Query too slow. Please add filters."))
    except frappe.QueryDeadlockError:
        frappe.log_error(frappe.get_traceback(), "Deadlock")
        frappe.throw(_("Database busy. Please try again."))
    except frappe.db.InternalError as e:
        msg = str(e).lower()
        if "gone away" in msg or "lost connection" in msg:
            frappe.db.connect()
            return frappe.db.sql(query, values or {}, as_dict=as_dict)
        frappe.log_error(frappe.get_traceback(), "DB Error")
        frappe.throw(_("Database error. Please contact support."))
```

---

## Pattern 5: Query with Retry on Transient Errors

```python
import time

def query_with_retry(func, max_retries=3):
    """Retry on deadlocks and connection errors."""
    for attempt in range(max_retries):
        try:
            return func()
        except frappe.QueryDeadlockError:
            if attempt < max_retries - 1:
                frappe.db.rollback()
                time.sleep(0.5 * (attempt + 1))
            else:
                raise
        except frappe.db.InternalError as e:
            msg = str(e).lower()
            if ("gone away" in msg or "lost connection" in msg) and attempt < max_retries - 1:
                frappe.db.connect()
                time.sleep(0.5)
            else:
                raise

# Usage:
result = query_with_retry(lambda: frappe.get_all("Item", limit=100))
```

---

## Pattern 6: Transaction with Savepoints

```python
def multi_step_operation(data):
    """Multi-step operation with partial rollback."""
    frappe.db.savepoint("step1")
    try:
        parent = frappe.get_doc({"doctype": "Sales Order", **data["parent"]})
        parent.insert()
    except Exception:
        frappe.throw(_("Failed to create order"))

    frappe.db.savepoint("step2")
    try:
        for child in data.get("children", []):
            frappe.get_doc({"doctype": "Task", "order": parent.name, **child}).insert()
    except frappe.ValidationError:
        frappe.db.rollback(save_point="step2")
        # Parent created, children failed — partial success
        return {"parent": parent.name, "partial": True}

    return {"parent": parent.name, "partial": False}
```

---

## Pattern 7: Parameterized SQL (Correct Format)

```python
# ALWAYS use %(name)s with dict — this is the ONLY safe format

# Single parameter
frappe.db.sql(
    "SELECT name FROM `tabItem` WHERE item_code = %(code)s",
    {"code": item_code},
    as_dict=True
)

# Multiple parameters
frappe.db.sql("""
    SELECT name, grand_total
    FROM `tabSales Invoice`
    WHERE customer = %(customer)s
    AND posting_date BETWEEN %(from)s AND %(to)s
    AND docstatus = 1
    ORDER BY posting_date DESC
    LIMIT %(limit)s
""", {
    "customer": customer,
    "from": from_date,
    "to": to_date,
    "limit": 100
}, as_dict=True)

# IN clause — use frappe.db.escape for each value
items = ["ITEM-001", "ITEM-002", "ITEM-003"]
escaped = ", ".join([frappe.db.escape(i) for i in items])
frappe.db.sql(f"""
    SELECT name FROM `tabItem` WHERE name IN ({escaped})
""", as_dict=True)

# Or use query builder for IN clause (safer)
Item = frappe.qb.DocType("Item")
frappe.qb.from_(Item).where(Item.name.isin(items)).run(as_dict=True)
```

---

## Pattern 8: Connection Error Recovery

```python
def with_connection_retry(func):
    """Decorator for automatic reconnection on connection loss."""
    def wrapper(*args, **kwargs):
        max_retries = 3
        for attempt in range(max_retries):
            try:
                return func(*args, **kwargs)
            except frappe.db.InternalError as e:
                msg = str(e).lower()
                is_connection_error = any(x in msg for x in [
                    "gone away", "lost connection", "can't connect", "connection refused"
                ])
                if not is_connection_error:
                    raise
                if attempt < max_retries - 1:
                    frappe.log_error(f"Reconnect attempt {attempt + 1}", "DB Connection")
                    import time
                    time.sleep(1 * (attempt + 1))
                    frappe.db.connect()
                else:
                    frappe.log_error(frappe.get_traceback(), "DB Connection Failed")
                    frappe.throw(_("Database unavailable. Please try again."))
    return wrapper

@with_connection_retry
def reliable_query():
    return frappe.get_all("Sales Invoice", limit=10)
```

---

## Pattern 9: Bulk Update with db.bulk_update [v15+]

```python
# Efficient bulk update — skips ORM, uses direct SQL
updates = {
    "ITEM-001": {"status": "Active", "sync_date": "2025-01-01"},
    "ITEM-002": {"status": "Active", "sync_date": "2025-01-01"},
}

try:
    frappe.db.bulk_update("Item", updates, chunk_size=100)
except frappe.db.InternalError:
    frappe.log_error(frappe.get_traceback(), "Bulk Update Error")
    frappe.throw(_("Bulk update failed"))

# NOTE: bulk_update skips validate/on_update. Use only for mass data fixes.
```

---

## Quick Reference: Database Error Patterns

| Error | Check | Handle |
|-------|-------|--------|
| Document not found | `frappe.db.exists()` | Throw user-friendly message |
| Duplicate entry | Catch `DuplicateEntryError` | Return existing or inform user |
| Missing mandatory | Catch `MandatoryError` | Show which field is missing |
| Link invalid | Catch `LinkValidationError` | Verify target exists first |
| Linked documents | Catch `LinkExistsError` | Show linked docs to user |
| Concurrent edit | Catch `TimestampMismatchError` | Ask user to refresh |
| Text too long | Catch `CharacterLengthExceededError` | Truncate or increase limit |
| Database error | Catch `InternalError` | Log details, show generic message |
| Query timeout | Catch `QueryTimeoutError` [v15+] | Paginate or add filters |
| Deadlock | Catch `QueryDeadlockError` | Retry with backoff |
| Connection lost | Check `InternalError` message | Reconnect and retry |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-database/references/patterns.md*
