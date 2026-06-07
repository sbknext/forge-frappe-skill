---
title: Frappe Ref — Frappe Syntax: Scheduler: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'scheduler', 'reference', 'examples']
source: Impertio
---

# Scheduler & Background Jobs — Complete Examples

## Example 1: Daily Email Summary

### hooks.py

```python
scheduler_events = {
    "daily": ["myapp.tasks.send_daily_summary"],
}
```

### myapp/tasks.py

```python
import frappe

def send_daily_summary():
    """Send daily summary email to all active managers."""
    managers = frappe.get_all(
        "User",
        filters={"enabled": 1, "role_profile_name": "Manager"},
        fields=["name", "email"],
    )

    for manager in managers:
        try:
            summary = build_summary(manager.name)
            frappe.sendmail(
                recipients=[manager.email],
                subject=f"Daily Summary — {frappe.utils.today()}",
                message=summary,
            )
            frappe.db.commit()
        except Exception:
            frappe.db.rollback()
            frappe.log_error(
                frappe.get_traceback(),
                f"Daily Summary Error: {manager.name}",
            )

def build_summary(user):
    open_tasks = frappe.db.count("ToDo", {"allocated_to": user, "status": "Open"})
    return f"You have {open_tasks} open tasks."
```

---

## Example 2: Cron-Based Data Sync

### hooks.py

```python
scheduler_events = {
    "cron": {
        "*/15 * * * *": ["myapp.integrations.sync.run_sync"],
    }
}
```

### myapp/integrations/sync.py

```python
import frappe
from frappe.utils.background_jobs import is_job_enqueued

def run_sync():
    """Enqueue sync job with deduplication (v15+)."""
    job_id = "external_api_sync"

    if is_job_enqueued(job_id):
        return  # Already running

    frappe.enqueue(
        "myapp.integrations.sync.do_sync",
        job_id=job_id,
        queue="long",
        timeout=1800,
    )

def do_sync():
    """Actual sync logic — runs in background worker."""
    records = fetch_from_external_api()

    for record in records:
        try:
            update_or_create(record)
            frappe.db.commit()
        except Exception:
            frappe.db.rollback()
            frappe.log_error(
                frappe.get_traceback(),
                f"Sync Error: {record.get('id')}",
            )
```

---

## Example 3: frappe.enqueue_doc with Progress

### Controller

```python
class DataImport(Document):
    @frappe.whitelist()
    def start_import(self):
        frappe.enqueue_doc(
            self.doctype,
            self.name,
            "run_import",
            queue="long",
            timeout=3600,
        )
        frappe.msgprint("Import started in background")

    def run_import(self):
        rows = get_import_rows(self.name)
        total = len(rows)

        for i, row in enumerate(rows):
            try:
                process_row(row)
                frappe.db.commit()
            except Exception:
                frappe.db.rollback()
                frappe.log_error(frappe.get_traceback(), f"Import Row Error: {i}")

            frappe.publish_realtime(
                "import_progress",
                {"progress": (i + 1) / total * 100, "current": i + 1, "total": total},
                doctype=self.doctype,
                docname=self.name,
            )
```

---

## Example 4: Retry with Exponential Backoff

```python
def call_external_api(data, retry_count=0, max_retries=3):
    """Call external API with retry on failure."""
    try:
        response = make_api_request(data)
        process_response(response)
    except Exception as e:
        if retry_count < max_retries:
            frappe.enqueue(
                "myapp.tasks.call_external_api",
                queue="default",
                data=data,
                retry_count=retry_count + 1,
                max_retries=max_retries,
                enqueue_after_commit=True,
            )
            frappe.log_error(
                f"Retry {retry_count + 1}/{max_retries}: {str(e)}",
                "API Retry",
            )
        else:
            frappe.log_error(
                frappe.get_traceback(),
                f"API Failed after {max_retries} retries",
            )
            raise
```

---

## Example 5: Heavy Scheduler Event Delegating to Enqueue

```python
# hooks.py
scheduler_events = {
    "daily_long": ["myapp.tasks.daily_cleanup"],
}

# myapp/tasks.py
def daily_cleanup():
    """Scheduler event that delegates to enqueue for heavy work."""
    sites = get_sites_needing_cleanup()

    for site in sites:
        frappe.enqueue(
            "myapp.tasks.cleanup_site",
            queue="long",
            timeout=1800,
            site_name=site,
        )

def cleanup_site(site_name):
    """Actual cleanup — runs as separate background job."""
    batch_size = 500
    offset = 0

    while True:
        old_logs = frappe.get_all(
            "Error Log",
            filters={"creation": ["<", frappe.utils.add_days(None, -30)]},
            fields=["name"],
            limit_page_length=batch_size,
            limit_start=offset,
        )
        if not old_logs:
            break

        for log in old_logs:
            frappe.delete_doc("Error Log", log.name, force=True)

        frappe.db.commit()
        offset += batch_size
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-scheduler/references/examples.md*
