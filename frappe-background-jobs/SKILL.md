---
name: frappe-background-jobs
description: >
  Run async work in Frappe with frappe.enqueue and RQ — queues, timeouts, job dedup, enqueue_doc,
  long jobs, progress, retries, and scheduler integration. Use when an operation is slow, calls an
  external API, or should not block the web request. Trigger on: "frappe.enqueue", "background job",
  "RQ worker", "long running task", "queue", "async job", "job_id dedup".
---

# Frappe Background Jobs

Offload anything slow or external to a worker so the request returns fast. Frappe uses RQ on Redis.

## Enqueue

```python
frappe.enqueue(
    method="myapp.services.sync.run",   # dotted path or callable
    queue="long",                       # short (≤300s) / default (≤300s) / long (≤1500s)
    timeout=1800,                       # override queue default
    job_id=f"sync_{doc.name}",          # dedup: same id won't double-queue
    deduplicate=True,
    enqueue_after_commit=True,          # wait for the DB txn to commit first
    is_async=True,                      # False = run inline (tests)
    # any kwargs below are passed to the method:
    doc_name=doc.name,
)
```

## Enqueue a document method

```python
doc.queue_action("submit", timeout=600)          # runs doc.submit() on a worker
frappe.enqueue_doc("Sales Order", doc.name, "run_sync", queue="long")
```

## Inside the job — discipline

```python
def run(doc_name):
    logger = frappe.logger("sync", allow_site=True)
    try:
        doc = frappe.get_doc("Sales Order", doc_name)
        # ... work ...
        frappe.db.commit()              # workers don't auto-commit on success path
    except Exception:
        frappe.db.rollback()
        logger.error(frappe.get_traceback())
        raise                           # let RQ mark it failed (visible in RQ dashboard)
```

## Progress for the UI

```python
frappe.publish_progress(percent=40, title="Syncing", description="40 / 100")
frappe.publish_realtime("sync_done", {"name": doc_name}, user=frappe.session.user)
```

## Patterns & rules

- **Always set `job_id` + `deduplicate=True`** for jobs triggered by high-frequency events (on_submit, webhooks) — prevents pile-ups.
- **`enqueue_after_commit=True`** when the job reads the row that triggered it — otherwise the worker may run before the txn commits and see stale/no data.
- Pick the right queue: `short` for quick I/O, `long` for batch/external sync. Don't run a 20-min job on `default`.
- Workers don't share the web request's transaction — `frappe.db.commit()` explicitly.
- Make jobs **idempotent** (safe to retry) — RQ may re-run on failure.
- Inspect: `bench --site <site> worker` (run), and the RQ jobs at `/app/rq-job`.
- Don't pass whole `doc` objects as kwargs — pass `name` and re-fetch inside the job.
