---
title: Frappe Ref — Frappe Syntax: Hooks: Scheduler Events (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'reference', 'scheduler-events']
source: Impertio
---

# Scheduler Events Reference

Complete reference for `scheduler_events` in hooks.py.

---

## Syntax

```python
scheduler_events = {
    # Standard frequencies (default queue, 300s timeout)
    "all": ["myapp.tasks.every_minute"],
    "hourly": ["myapp.tasks.hourly_task"],
    "daily": ["myapp.tasks.daily_task"],
    "weekly": ["myapp.tasks.weekly_task"],
    "monthly": ["myapp.tasks.monthly_task"],

    # Long queue variants (long queue, 1500s timeout)
    "hourly_long": ["myapp.tasks.heavy_hourly"],
    "daily_long": ["myapp.tasks.heavy_daily"],
    "weekly_long": ["myapp.tasks.heavy_weekly"],
    "monthly_long": ["myapp.tasks.heavy_monthly"],

    # Cron expressions (default queue, 300s timeout)
    "cron": {
        "0 9 * * 1-5": ["myapp.tasks.weekday_morning"],
        "*/30 * * * *": ["myapp.tasks.half_hourly_sync"]
    }
}
```

---

## All Event Types

### Standard Frequencies (Default Queue)

| Event | Frequency | Timeout | When |
|-------|-----------|---------|------|
| `all` | ~60 seconds | 300s | Every scheduler tick |
| `hourly` | Every hour | 300s | At :00 |
| `daily` | Every day | 300s | At 00:00 |
| `weekly` | Every week | 300s | Sunday 00:00 |
| `monthly` | Every month | 300s | 1st of month 00:00 |

### Long Queue Variants

| Event | Frequency | Timeout | When |
|-------|-----------|---------|------|
| `hourly_long` | Every hour | 1500s | At :00 |
| `daily_long` | Every day | 1500s | At 00:00 |
| `weekly_long` | Every week | 1500s | Sunday 00:00 |
| `monthly_long` | Every month | 1500s | 1st of month 00:00 |

### Cron (Custom Timing)

The `cron` key is a dict where keys are cron expressions and values are lists
of dotted paths.

---

## Cron Syntax

```
* * * * *
| | | | |
| | | | +-- Day of week (0-6, Sunday=0)
| | | +---- Month (1-12)
| | +------ Day of month (1-31)
| +-------- Hour (0-23)
+---------- Minute (0-59)
```

### Special Values

| Value | Meaning | Example |
|-------|---------|---------|
| `*` | Every | Every minute, hour, etc. |
| `*/n` | Every nth | `*/5` = every 5 |
| `n-m` | Range | `1-5` = Monday through Friday |
| `n,m` | List | `1,15` = 1st and 15th |

### Common Cron Patterns

| Pattern | Meaning |
|---------|---------|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour at :00 |
| `0 9 * * *` | Daily at 09:00 |
| `0 9 * * 1-5` | Weekdays at 09:00 |
| `0 0 1 * *` | First day of month at 00:00 |
| `0 17 * * 5` | Friday at 17:00 |
| `*/15 9-17 * * 1-5` | Every 15 min during business hours |
| `30 14 * * *` | Daily at 14:30 |

---

## Task Implementation

### Basic Task

```python
# myapp/tasks.py
import frappe

def daily_report():
    """Scheduled tasks receive NO arguments."""
    report = generate_report()
    frappe.sendmail(
        recipients=["manager@example.com"],
        subject="Daily Report",
        message=report
    )
```

### Task with Error Handling

```python
def hourly_sync():
    """ALWAYS wrap in try/except and log errors."""
    try:
        records = frappe.get_all("Sync Queue", filters={"status": "Pending"})
        for record in records:
            process_sync(record.name)
            frappe.db.commit()  # Commit per record for large batches
    except Exception:
        frappe.log_error(
            title="Hourly Sync Failed",
            message=frappe.get_traceback()
        )
```

### Long Running Task

```python
def monthly_aggregation():
    """Use _long variant for tasks exceeding 5 minutes."""
    for company in frappe.get_all("Company"):
        aggregate_data(company.name)
        frappe.db.commit()  # Commit per iteration to prevent memory buildup
```

---

## Queue Selection Guide

| Scenario | Use | Reason |
|----------|-----|--------|
| Quick check (< 5 min) | `hourly`, `daily`, etc. | Default queue, 5 min timeout |
| Heavy processing (5-25 min) | `hourly_long`, `daily_long`, etc. | Long queue, 25 min timeout |
| Precise timing needed | `cron` | Exact schedule control |
| Sub-hourly frequency | `cron` with `*/n * * * *` | Standard events are hourly minimum |
| Near real-time | `all` | ~60s interval, default queue |

---

## Critical Rules

### 1. ALWAYS Run bench migrate After Changes

```bash
bench --site sitename migrate
```

Scheduler events are cached. Without migrate, changes are NOT picked up.

### 2. NEVER Define Tasks with Arguments

```python
# WRONG — tasks receive no arguments
def my_task(company_name):
    process_company(company_name)

# CORRECT — fetch data inside the function
def my_task():
    for company in frappe.get_all("Company"):
        process_company(company.name)
```

### 3. ALWAYS Commit in Long Loops

```python
def process_all_invoices():
    invoices = frappe.get_all("Sales Invoice", limit=0)
    for inv in invoices:
        process_invoice(inv.name)
        frappe.db.commit()  # Prevent memory buildup
```

### 4. ALWAYS Use Error Handling

```python
def risky_task():
    try:
        do_work()
    except Exception:
        frappe.log_error(title="Task Failed", message=frappe.get_traceback())
```

---

## Debugging

### Manual Execution

```python
# In bench console
frappe.get_doc("Scheduled Job Type", "myapp.tasks.daily_report").execute()
```

### Check Scheduler Status

```bash
bench --site sitename scheduler status
bench --site sitename scheduler enable
bench --site sitename scheduler disable
```

### View Logs

```bash
tail -f ~/frappe-bench/logs/scheduler.log
tail -f ~/frappe-bench/logs/worker.log
```

---

## Complete Example

```python
# hooks.py
scheduler_events = {
    "hourly": [
        "myapp.tasks.check_pending_orders"
    ],
    "daily": [
        "myapp.tasks.send_daily_digest",
        "myapp.tasks.cleanup_temp_files"
    ],
    "daily_long": [
        "myapp.tasks.recalculate_all_balances"
    ],
    "weekly_long": [
        "myapp.tasks.generate_weekly_analytics"
    ],
    "cron": {
        "0 9 * * 1-5": ["myapp.tasks.send_payment_reminders"],
        "*/30 * * * *": ["myapp.tasks.sync_external_api"],
        "0 17 * * 5": ["myapp.tasks.weekly_summary"]
    }
}
```

---

## Version Differences

| Feature | v14 | v15 | v16 |
|---------|-----|-----|-----|
| All standard events | Yes | Yes | Yes |
| Cron syntax | Yes | Yes | Yes |
| `_long` variants | Yes | Yes | Yes |
| Scheduler UI | Basic | Improved | Improved |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks/references/scheduler-events.md*
