---
title: Frappe Ref — Frappe Syntax: Whitelisted: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'patterns']
source: Impertio
---

# Common Patterns Reference

Proven patterns for building whitelisted method APIs.

## Search Endpoint

```python
@frappe.whitelist(methods=["GET"])
def search_customers(query, limit=20):
    frappe.has_permission("Customer", "read", throw=True)

    limit = min(int(limit), 100)
    query = frappe.utils.strip_html(query or "")[:100]

    if not query:
        return {"data": [], "count": 0}

    data = frappe.get_all(
        "Customer",
        or_filters={
            "customer_name": ["like", f"%{query}%"],
            "name": ["like", f"%{query}%"]
        },
        fields=["name", "customer_name", "email_id", "territory"],
        limit_page_length=limit,
        order_by="customer_name"
    )
    return {"data": data, "count": len(data)}
```

---

## Paginated List with Filters

```python
@frappe.whitelist(methods=["GET"])
def get_orders(status=None, customer=None, page=1, page_size=20):
    frappe.has_permission("Sales Order", "read", throw=True)

    page = max(int(page), 1)
    page_size = min(int(page_size), 100)
    start = (page - 1) * page_size

    filters = {"docstatus": 1}
    if status:
        filters["status"] = status
    if customer:
        filters["customer"] = customer

    data = frappe.get_all(
        "Sales Order",
        filters=filters,
        fields=["name", "customer", "grand_total", "status", "transaction_date"],
        limit_start=start,
        limit_page_length=page_size,
        order_by="transaction_date desc"
    )
    total = frappe.db.count("Sales Order", filters)

    return {
        "data": data,
        "pagination": {
            "page": page,
            "page_size": page_size,
            "total": total,
            "pages": (total + page_size - 1) // page_size,
            "has_next": (start + page_size) < total
        }
    }
```

---

## Aggregate/Dashboard Pattern

```python
@frappe.whitelist(methods=["GET"])
def get_dashboard_stats():
    frappe.only_for(["Sales Manager", "System Manager"])

    return {
        "total_orders": frappe.db.count("Sales Order", {"docstatus": 1}),
        "draft_orders": frappe.db.count("Sales Order", {"docstatus": 0}),
        "revenue": float(frappe.db.sql("""
            SELECT COALESCE(SUM(grand_total), 0)
            FROM `tabSales Order` WHERE docstatus = 1
        """)[0][0]),
        "top_items": frappe.db.sql("""
            SELECT item_code, SUM(qty) as total_qty
            FROM `tabSales Order Item`
            WHERE docstatus = 1
            GROUP BY item_code
            ORDER BY total_qty DESC
            LIMIT 5
        """, as_dict=True)
    }
```

---

## Upsert Pattern (Create or Update)

```python
@frappe.whitelist(methods=["POST"])
def upsert_setting(key, value):
    frappe.only_for("System Manager")

    if not key or value is None:
        frappe.throw(_("Key and value are required"))

    if frappe.db.exists("App Setting", {"key": key}):
        frappe.db.set_value("App Setting", {"key": key}, "value", value)
        return {"action": "updated", "key": key}
    else:
        doc = frappe.get_doc({
            "doctype": "App Setting",
            "key": key,
            "value": value
        })
        doc.insert()
        return {"action": "created", "key": key, "name": doc.name}
```

---

## Webhook Receiver Pattern

```python
from frappe.rate_limiter import rate_limit

@frappe.whitelist(allow_guest=True, methods=["POST"])
@rate_limit(limit=100, seconds=60)
def webhook_receiver():
    """Receive webhooks from external services."""
    import hmac
    import hashlib

    # Verify signature
    payload = frappe.request.data
    signature = frappe.request.headers.get("X-Signature")
    secret = frappe.db.get_single_value("Integration Settings", "webhook_secret")

    expected = hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature or "", expected):
        frappe.local.response["http_status_code"] = 401
        return {"error": "Invalid signature"}

    data = frappe.parse_json(payload)
    # Process webhook data...
    frappe.get_doc({
        "doctype": "Webhook Log",
        "event": data.get("event"),
        "payload": str(data)
    }).insert(ignore_permissions=True)

    return {"received": True}
```

---

## Background Job Pattern

For long-running operations, enqueue a background job:

```python
@frappe.whitelist(methods=["POST"])
def start_bulk_import(file_url):
    frappe.only_for("System Manager")

    # Validate
    if not file_url:
        frappe.throw(_("File URL required"))

    # Enqueue background job
    frappe.enqueue(
        "myapp.tasks.process_import",
        queue="long",
        timeout=600,
        file_url=file_url,
        user=frappe.session.user
    )

    return {"status": "queued", "message": _("Import started in background")}
```

```python
# myapp/tasks.py (NOT whitelisted — runs in background worker)
import frappe

def process_import(file_url, user):
    frappe.set_user(user)
    # Long-running operation...
    frappe.publish_realtime("import_complete", {"status": "done"}, user=user)
```

---

## Multi-Step Workflow Pattern

```python
@frappe.whitelist(methods=["POST"])
def approve_and_notify(doctype, name, comment=None):
    frappe.has_permission(doctype, "submit", name, throw=True)

    doc = frappe.get_doc(doctype, name)

    # Step 1: Validate state
    if doc.workflow_state != "Pending Approval":
        frappe.throw(_("Document is not pending approval"))

    # Step 2: Update
    doc.workflow_state = "Approved"
    if comment:
        doc.add_comment("Comment", comment)
    doc.save()

    # Step 3: Notify
    if doc.owner != frappe.session.user:
        frappe.sendmail(
            recipients=[doc.owner],
            subject=_("{0} {1} Approved").format(doctype, name),
            message=_("Your {0} has been approved by {1}").format(
                name, frappe.session.user
            )
        )

    return {"success": True, "new_state": "Approved"}
```

---

## Realtime Update Pattern

```python
@frappe.whitelist(methods=["POST"])
def process_with_progress(items):
    frappe.only_for("System Manager")

    if isinstance(items, str):
        items = frappe.parse_json(items)

    total = len(items)
    for i, item in enumerate(items):
        process_single(item)

        # Send progress to client
        frappe.publish_realtime(
            "progress",
            {"current": i + 1, "total": total, "item": item},
            user=frappe.session.user
        )

    return {"processed": total}
```

Client side:
```javascript
frappe.realtime.on('progress', (data) => {
    frappe.show_progress(__('Processing'), data.current, data.total);
});
```

---

## Key Rules

1. **ALWAYS cap page sizes** in paginated endpoints — prevent `page_size=999999`
2. **ALWAYS sanitize search queries** — strip HTML, limit length
3. **ALWAYS use background jobs** for operations > 30 seconds
4. **ALWAYS verify webhook signatures** — never trust unverified payloads
5. **ALWAYS use parameterized SQL** — even in aggregate queries

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/patterns.md*
