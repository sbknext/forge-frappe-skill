---
title: Frappe Ref — Frappe Syntax: Scheduler: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'scheduler', 'reference', 'anti-patterns']
source: Impertio
---

# Scheduler & Background Jobs — Anti-Patterns

## 1. Forgetting bench migrate After hooks.py Change

```python
# WRONG — changed scheduler_events but did not run bench migrate
scheduler_events = {
    "daily": ["myapp.tasks.new_task"],  # NOT active until bench migrate!
}
```

**Fix**: ALWAYS run `bench migrate` after ANY change to scheduler_events.

---

## 2. Heavy Logic Directly in Scheduler Event

```python
# WRONG — blocks the scheduler worker for the entire duration
def daily_export():
    for record in frappe.get_all("Sales Invoice", fields=["*"]):
        generate_pdf(record)      # 30+ minutes
        upload_to_cloud(record)
```

```python
# CORRECT — scheduler event enqueues the heavy work
def daily_export():
    frappe.enqueue(
        "myapp.tasks.do_export",
        queue="long",
        timeout=3600,
    )
```

---

## 3. No Error Handling in Background Job

```python
# WRONG — one failure kills the entire job
def process_all():
    for item in frappe.get_all("Item"):
        external_api_call(item.name)  # If this throws, everything stops
```

```python
# CORRECT — per-item error handling with commit/rollback
def process_all():
    for item in frappe.get_all("Item", fields=["name"]):
        try:
            external_api_call(item.name)
            frappe.db.commit()
        except Exception:
            frappe.db.rollback()
            frappe.log_error(frappe.get_traceback(), f"Process Error: {item.name}")
```

---

## 4. Wrong Queue for Task Duration

```python
# WRONG — heavy report on short queue (5 min timeout)
frappe.enqueue("myapp.tasks.annual_report", queue="short")
```

```python
# CORRECT — use long queue with appropriate timeout
frappe.enqueue("myapp.tasks.annual_report", queue="long", timeout=3600)
```

**Rule**: short < 30s, default < 5min, long < 25min, long+timeout for longer.

---

## 5. No Deduplication

```python
# WRONG — clicking button 5 times = 5 duplicate jobs
def on_click():
    frappe.enqueue("myapp.tasks.process", doc=doc.name)
```

```python
# CORRECT — deduplicate with job_id (v15+)
from frappe.utils.background_jobs import is_job_enqueued

def on_click():
    job_id = f"process::{doc.name}"
    if not is_job_enqueued(job_id):
        frappe.enqueue("myapp.tasks.process", job_id=job_id, doc=doc.name)
    else:
        frappe.msgprint("Already processing")
```

---

## 6. Using job_name Instead of job_id (v15+)

```python
# WRONG — deprecated v14 pattern
frappe.enqueue("myapp.tasks.process", job_name="my_job")
```

```python
# CORRECT — v15+ deduplication
frappe.enqueue("myapp.tasks.process", job_id="my_job")
```

---

## 7. Forgetting Administrator Context

```python
# WRONG — document created with no explicit owner
def scheduled_task():
    doc = frappe.new_doc("ToDo")
    doc.description = "Auto-created task"
    doc.insert()
    # doc.owner = "Administrator" — probably not intended
```

```python
# CORRECT — set explicit owner
def scheduled_task():
    doc = frappe.new_doc("ToDo")
    doc.description = "Auto-created task"
    doc.owner = "actual-user@example.com"
    doc.allocated_to = "actual-user@example.com"
    doc.insert(ignore_permissions=True)
```

---

## 8. No Commit in Batch Processing

```python
# WRONG — all changes lost on any error (implicit rollback)
def process_batch():
    for item in large_list:
        frappe.db.set_value("Item", item, "status", "Processed")
    # No commit — if worker crashes, everything is lost
```

```python
# CORRECT — commit per batch
def process_batch():
    for i, item in enumerate(large_list):
        frappe.db.set_value("Item", item, "status", "Processed")
        if i % 100 == 0:
            frappe.db.commit()
    frappe.db.commit()
```

---

## 9. Assuming Execution Order of Multiple Methods

```python
# WRONG — assuming step_1 runs before step_2
scheduler_events = {
    "daily": [
        "myapp.tasks.step_1",  # Execution order is NOT guaranteed!
        "myapp.tasks.step_2",
    ]
}
```

```python
# CORRECT — chain explicitly if order matters
scheduler_events = {
    "daily": ["myapp.tasks.run_steps"],
}

def run_steps():
    step_1()
    step_2()
```

---

## 10. Using enqueue_after_commit and Expecting Return Value

```python
# WRONG — returns None when enqueue_after_commit=True
job = frappe.enqueue("myapp.tasks.process", enqueue_after_commit=True)
print(job.id)  # AttributeError: NoneType has no attribute 'id'
```

```python
# CORRECT — do not use return value with enqueue_after_commit
frappe.enqueue("myapp.tasks.process", enqueue_after_commit=True)
# No return value available
```

---

## Summary

| # | Anti-Pattern | Consequence |
|---|-------------|-------------|
| 1 | No bench migrate | Changes not applied |
| 2 | Heavy logic in scheduler event | Blocks scheduler worker |
| 3 | No error handling | Silent failures, lost data |
| 4 | Wrong queue | Timeout kills job |
| 5 | No deduplication | Duplicate jobs |
| 6 | job_name instead of job_id | Deprecated, unreliable |
| 7 | No explicit owner | Documents owned by Administrator |
| 8 | No commit in batch | Data loss on crash |
| 9 | Assuming execution order | Race conditions |
| 10 | Return value with enqueue_after_commit | NoneType error |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-scheduler/references/anti-patterns.md*
