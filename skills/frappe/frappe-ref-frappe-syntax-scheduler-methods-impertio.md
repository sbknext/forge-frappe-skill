---
title: Frappe Ref — Frappe Syntax: Scheduler: Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'scheduler', 'reference', 'methods']
source: Impertio
---

# Scheduler & Background Jobs — Method Reference

## frappe.enqueue()

```python
frappe.enqueue(
    method,                      # str or callable — REQUIRED
    queue="default",             # "short" | "default" | "long" | custom
    timeout=None,                # int — override queue timeout (seconds)
    is_async=True,               # bool — False = synchronous (skip worker)
    now=False,                   # bool — True = run via frappe.call() directly
    job_id=None,                 # str — [v15+] unique ID for deduplication
    enqueue_after_commit=False,  # bool — wait for DB commit before enqueue
    at_front=False,              # bool — place at front of queue
    on_success=None,             # callable — success callback
    on_failure=None,             # callable — failure callback
    **kwargs                     # passed to method as arguments
)
# Returns: RQ Job object (or None if enqueue_after_commit=True)
```

### Method Parameter Formats

```python
# Module path string (recommended for cross-module calls)
frappe.enqueue("myapp.tasks.process_data", customer="CUST-001")

# Function object (for same-module calls)
def my_task(name, value):
    pass

frappe.enqueue(my_task, name="test", value=123)
```

### Return Value

```python
# Normal enqueue — returns RQ Job object
job = frappe.enqueue("myapp.tasks.process", param="value")
print(job.id)       # Job ID string
print(job.status)   # "queued", "started", "finished", "failed"

# With enqueue_after_commit — returns None
job = frappe.enqueue("myapp.tasks.process", enqueue_after_commit=True)
# job is None — NEVER access attributes on it
```

---

## frappe.enqueue_doc()

```python
frappe.enqueue_doc(
    doctype,           # str — DocType name (REQUIRED)
    name=None,         # str — document name
    method=None,       # str — controller method name
    queue="default",   # str — queue name
    timeout=300,       # int — timeout in seconds
    now=False,         # bool — run directly
    **kwargs           # passed to controller method
)
```

The controller method MUST be decorated with `@frappe.whitelist()`.

---

## Document.queue_action()

```python
# From within a controller
self.queue_action(
    method_name,       # str — name of method on this controller
    **kwargs           # passed to method
)
```

---

## frappe.utils.background_jobs.is_job_enqueued()

```python
from frappe.utils.background_jobs import is_job_enqueued

result = is_job_enqueued(job_id)  # str — returns bool
```

[v15+] Checks if a job with the given `job_id` is currently queued or running.

---

## Callback Signatures

### on_success

```python
def on_success_handler(job, connection, result, *args, **kwargs):
    """
    job: RQ Job object
    connection: Redis connection
    result: return value of the executed method
    """
    pass
```

### on_failure

```python
def on_failure_handler(job, connection, type, value, traceback):
    """
    job: RQ Job object
    connection: Redis connection
    type: exception class
    value: exception instance
    traceback: traceback object
    """
    pass
```

---

## frappe.log_error()

```python
frappe.log_error(
    message="Error details or traceback",
    title="Short title for Error Log list"
)
```

Creates an Error Log document visible in the desk.

---

## frappe.publish_realtime()

```python
frappe.publish_realtime(
    event,             # str — event name
    message=None,      # dict — data payload
    user=None,         # str — target user (or broadcast if None)
    doctype=None,      # str — target doctype (for doc-specific updates)
    docname=None,      # str — target document name
    after_commit=False # bool — wait for DB commit
)
```

---

## Scheduler Control

```python
# Enable scheduler
frappe.utils.scheduler.enable_scheduler()

# Disable scheduler
frappe.utils.scheduler.disable_scheduler()

# Check if enabled
is_enabled = frappe.utils.scheduler.is_scheduler_inactive()
```

---

## Queue Utilities

```python
from frappe.utils.background_jobs import get_queue

# Get queue object
queue = get_queue("default")
print(f"Jobs in queue: {len(queue)}")

# Get job status
from rq.job import Job
job = Job.fetch(job_id, connection=frappe.cache())
print(job.get_status())  # "queued", "started", "finished", "failed"
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-scheduler/references/methods.md*
