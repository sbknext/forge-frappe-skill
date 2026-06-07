---
title: Frappe Ref — Frappe Syntax: Scheduler: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'scheduler', 'reference', 'patterns']
source: Impertio
---

# Scheduler & Background Jobs — Patterns

## Pattern 1: Scheduler Event Delegates to Enqueue

NEVER do heavy work directly in a scheduler event. ALWAYS delegate to `frappe.enqueue()`.

```python
# hooks.py
scheduler_events = {
    "daily": ["myapp.tasks.daily_export"],
}

# myapp/tasks.py
def daily_export():
    """Lightweight scheduler event — just enqueues the heavy work."""
    frappe.enqueue(
        "myapp.tasks.do_export",
        queue="long",
        timeout=3600,
    )

def do_export():
    """Heavy work runs in background worker, not in scheduler."""
    # Export logic here
    pass
```

---

## Pattern 2: Batch Processing with Progress

```python
def process_large_dataset(user=None):
    """Process records in batches with progress updates."""
    batch_size = 500
    total = frappe.db.count("MyDocType", {"status": "Pending"})
    processed = 0

    while processed < total:
        records = frappe.get_all(
            "MyDocType",
            filters={"status": "Pending"},
            fields=["name"],
            limit_page_length=batch_size,
        )
        if not records:
            break

        for record in records:
            try:
                do_work(record.name)
                frappe.db.commit()
                processed += 1
            except Exception:
                frappe.db.rollback()
                frappe.log_error(frappe.get_traceback(), f"Error: {record.name}")
                processed += 1

        if user:
            frappe.publish_realtime(
                "processing_progress",
                {"progress": processed / total * 100},
                user=user,
            )
```

---

## Pattern 3: Deduplication Guard

```python
from frappe.utils.background_jobs import is_job_enqueued

def safe_enqueue(method, job_id, **kwargs):
    """Enqueue only if not already running (v15+)."""
    if not is_job_enqueued(job_id):
        return frappe.enqueue(method, job_id=job_id, **kwargs)
    return None
```

---

## Pattern 4: Callbacks for User Notification

```python
def start_heavy_task(doc_name, user):
    """Start task with success/failure notifications."""

    def on_success(job, connection, result, *args, **kwargs):
        frappe.publish_realtime(
            "show_alert",
            {"message": f"Task completed for {doc_name}", "indicator": "green"},
            user=user,
        )

    def on_failure(job, connection, type, value, traceback):
        frappe.publish_realtime(
            "show_alert",
            {"message": f"Task failed for {doc_name}", "indicator": "red"},
            user=user,
        )
        frappe.log_error(f"Job {job.id} failed: {value}", "Task Failed")

    frappe.enqueue(
        "myapp.tasks.heavy_task",
        queue="long",
        on_success=on_success,
        on_failure=on_failure,
        doc_name=doc_name,
    )
```

---

## Pattern 5: enqueue_after_commit

Use when the background job needs data that is being saved in the current transaction:

```python
class SalesInvoice(Document):
    def on_submit(self):
        # Invoice data is not yet committed to DB
        frappe.enqueue(
            "myapp.tasks.send_invoice_notification",
            enqueue_after_commit=True,  # Waits for commit
            invoice=self.name,
        )
```

Without `enqueue_after_commit`, the worker might start before the document is committed, causing a "not found" error.

---

## Pattern 6: Conditional Scheduling

```python
def configurable_sync():
    """Only run if integration is enabled in settings."""
    settings = frappe.get_single("My App Settings")

    if not settings.enable_sync:
        return  # Skip silently

    frappe.enqueue(
        "myapp.integrations.sync.do_sync",
        queue="long",
        timeout=1800,
    )
```

---

## Pattern 7: Multi-Site Awareness

```python
def scheduled_task():
    """Task that respects multi-site setup."""
    site = frappe.local.site

    # Site-specific logic
    config = frappe.get_site_config()
    if config.get("disable_scheduled_exports"):
        return

    # Proceed with task
    pass
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-scheduler/references/patterns.md*
