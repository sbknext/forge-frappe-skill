---
title: Frappe Ref — Frappe Syntax: Scheduler: Syntax (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'scheduler', 'reference', 'syntax']
source: Impertio
---

# Scheduler & Background Jobs — Syntax Quick Reference

## hooks.py — scheduler_events

```python
scheduler_events = {
    "all": ["dotted.module.path"],
    "hourly": ["dotted.module.path"],
    "daily": ["dotted.module.path"],
    "weekly": ["dotted.module.path"],
    "monthly": ["dotted.module.path"],
    "hourly_long": ["dotted.module.path"],
    "daily_long": ["dotted.module.path"],
    "weekly_long": ["dotted.module.path"],
    "monthly_long": ["dotted.module.path"],
    "cron": {
        "cron_expression": ["dotted.module.path"],
    },
}
```

---

## frappe.enqueue() — Minimal

```python
frappe.enqueue("myapp.tasks.process", customer="CUST-001")
```

## frappe.enqueue() — Full

```python
frappe.enqueue(
    "myapp.tasks.process",
    queue="long",
    timeout=3600,
    job_id="unique-id",
    enqueue_after_commit=True,
    at_front=False,
    on_success=success_fn,
    on_failure=failure_fn,
    param1="value1",
    param2="value2",
)
```

---

## frappe.enqueue_doc()

```python
frappe.enqueue_doc(
    "Sales Invoice",
    "SINV-00001",
    "controller_method",
    queue="long",
    timeout=600,
    kwarg1="value",
)
```

---

## self.queue_action()

```python
# Inside a Document controller
self.queue_action("method_name", param="value")
```

---

## Deduplication [v15+]

```python
from frappe.utils.background_jobs import is_job_enqueued

if not is_job_enqueued("unique-job-id"):
    frappe.enqueue("myapp.tasks.process", job_id="unique-job-id")
```

---

## Scheduler Event Function

```python
def my_task():
    """No arguments. Runs as Administrator."""
    pass
```

---

## Error Handling Template

```python
def batch_job(items):
    for item in items:
        try:
            process(item)
            frappe.db.commit()
        except Exception:
            frappe.db.rollback()
            frappe.log_error(frappe.get_traceback(), f"Error: {item}")
```

---

## Callback Signatures

```python
def on_success(job, connection, result, *args, **kwargs):
    pass

def on_failure(job, connection, type, value, traceback):
    pass
```

---

## Progress Reporting

```python
frappe.publish_realtime("event_name", {"progress": 50}, user="user@example.com")
```

---

## CLI Commands

```bash
bench migrate                          # Activate scheduler_events changes
bench doctor                           # Scheduler status
bench --site mysite enable-scheduler   # Enable scheduler
bench --site mysite disable-scheduler  # Disable scheduler
bench worker --queue short             # Start worker for queue
bench worker --queue short --burst     # Temporary burst worker
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-scheduler/references/syntax.md*
