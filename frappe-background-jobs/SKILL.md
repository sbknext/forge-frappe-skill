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

## Custom queues and workers

Add queues in `common_site_config.json`:

```json
{ "workers": { "myqueue": { "timeout": 5000, "background_workers": 4 } } }
```

Run multi-queue workers: `bench worker --queue short,default`. Burst mode for cron spikes: `bench worker --queue short --burst`.

## Scheduler-owned jobs

Jobs triggered by the scheduler run as **Administrator** — documents created inherit Administrator as owner unless you set `owner` explicitly.

Configurable intervals: create `Scheduler Event` + `Scheduled Job Type` (Cron) records instead of hard-coding cron in hooks.

## Patterns & rules

- **Always set `job_id` + `deduplicate=True`** for jobs triggered by high-frequency events (on_submit, webhooks) — prevents pile-ups.
- **`enqueue_after_commit=True`** when the job reads the row that triggered it — otherwise the worker may run before the txn commits and see stale/no data.
- Pick the right queue: `short` for quick I/O, `long` for batch/external sync. Don't run a 20-min job on `default`.
- Workers don't share the web request's transaction — `frappe.db.commit()` explicitly.
- Make jobs **idempotent** (safe to retry) — RQ may re-run on failure.
- Inspect: `bench --site <site> worker` (run), and the RQ jobs at `/app/rq-job`.
- Don't pass whole `doc` objects as kwargs — pass `name` and re-fetch inside the job.

## From Frappe docs

- `frappe.enqueue` / `frappe.enqueue_doc` arguments, queues, workers: https://docs.frappe.io/framework/user/en/api/background_jobs
- After changing `scheduler_events` in `hooks.py`, run `bench --site <site> migrate` for changes to take effect.
