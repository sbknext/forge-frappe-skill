---
title: Frappe Ref — Frappe Syntax: Scheduler: Hooks (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'scheduler', 'reference', 'hooks']
source: Impertio
---

# Scheduler Hooks Reference

## scheduler_events Hook

**Location**: `{app}/{app}/hooks.py`

The `scheduler_events` dictionary maps event frequencies to lists of Python module paths that execute at the specified intervals.

### Complete Syntax

```python
scheduler_events = {
    # Standard events — run on default queue
    "all": [
        "myapp.tasks.every_tick",
    ],
    "hourly": [
        "myapp.tasks.hourly_sync",
        "myapp.tasks.hourly_cleanup",
    ],
    "daily": [
        "myapp.tasks.daily_report",
    ],
    "weekly": [
        "myapp.tasks.weekly_summary",
    ],
    "monthly": [
        "myapp.tasks.monthly_billing",
    ],

    # Long queue events — run on long queue (higher timeout)
    "hourly_long": [
        "myapp.tasks.hourly_heavy_sync",
    ],
    "daily_long": [
        "myapp.tasks.daily_large_export",
    ],
    "weekly_long": [
        "myapp.tasks.weekly_archive",
    ],
    "monthly_long": [
        "myapp.tasks.monthly_full_report",
    ],

    # Cron events — croniter-compatible schedule strings
    "cron": {
        "*/5 * * * *": [
            "myapp.tasks.every_5_minutes",
        ],
        "0 9 * * 1-5": [
            "myapp.tasks.weekday_morning",
        ],
        "0 0 1 * *": [
            "myapp.tasks.first_of_month",
        ],
    },
}
```

### How Scheduler Events Are Registered

1. Developer adds/modifies `scheduler_events` in `hooks.py`
2. Developer runs `bench migrate`
3. Frappe reads hooks from all installed apps
4. Creates/updates `Scheduled Job Type` records in the database
5. The scheduler process checks these records at each tick

### Activation Requirement

After EVERY change to `scheduler_events`:

```bash
bench migrate
```

Without this step, changes to scheduler_events are NOT active.

### Multiple Apps

When multiple apps define scheduler events for the same frequency, ALL registered methods execute. There is no override — events accumulate across apps.

```python
# App A hooks.py
scheduler_events = {"daily": ["app_a.tasks.daily"]}

# App B hooks.py
scheduler_events = {"daily": ["app_b.tasks.daily"]}

# Result: BOTH app_a.tasks.daily AND app_b.tasks.daily run daily
```

### Event Function Signature

All scheduler event functions take NO arguments:

```python
def my_scheduled_task():
    """Scheduler event — no arguments."""
    # Use frappe API to access data
    records = frappe.get_all("DocType", filters={...})
```

### Enabling/Disabling Scheduler

```bash
# Disable scheduler for a site
bench --site mysite disable-scheduler

# Enable scheduler for a site
bench --site mysite enable-scheduler

# Check scheduler status
bench doctor
```

### Site Config Options

In `common_site_config.json`:

```json
{
    "scheduler_tick_interval": 60,
    "pause_scheduler": 0
}
```

| Key | Default | Description |
|-----|---------|-------------|
| `scheduler_tick_interval` | 60 [v15+] / 240 [v14] | Seconds between ticks |
| `pause_scheduler` | 0 | 1 = pause all scheduled events |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-scheduler/references/hooks.md*
