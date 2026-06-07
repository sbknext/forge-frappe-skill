---
name: frappe-app-development
description: >
  Scaffold, architect, and build production-grade custom Frappe/ERPNext apps. Covers app structure,
  DocType design, hooks.py configuration, background jobs, service layers, REST APIs, permissions,
  and testing. Use when creating a new Frappe app, adding a module, setting up scheduler events,
  implementing doc_events, writing whitelisted APIs, or debugging Frappe-specific issues.
  Trigger on: "create Frappe app", "new DocType", "add a hook", "background job", "frappe.enqueue",
  "whitelist API", "custom app scaffold", "bench new-app".
---

# Frappe App Development

Scaffold and build production-grade custom Frappe / ERPNext applications with correct separation of
concerns and tested patterns.

## When to use

- Creating a new custom Frappe app from scratch (`bench new-app`)
- Designing DocType structure (fields, child tables, naming series)
- Writing `hooks.py` — `doc_events`, `scheduler_events`, `override_whitelisted_methods`
- Implementing background jobs via `frappe.enqueue`
- Building REST APIs with `@frappe.whitelist()`
- Service-layer pattern: thin controllers, fat service modules
- Adding caching, logging, error handling cross-cutting utilities
- Permission checks with `frappe.has_permission()` / `frappe.only_for()`
- Writing tests with `frappe.tests.utils.FrappeTestCase`
- Deploying and migrating (`bench migrate`, `bench build`)

## Inputs required

- App name and purpose
- Target Frappe version (v14, v15, v16+)
- Module structure (which DocTypes, which APIs)
- Site name

## Core patterns

### 1. App scaffold
```bash
bench new-app <app_name>
bench --site <site> install-app <app_name>
```

### 2. DocType conventions (Frappe way)
- Use `frappe.get_doc()` / `frappe.get_list()` — never raw SQL unless parameterized
- `frappe.db.get_value()` for single field reads
- `frappe.db.sql()` ONLY with `%s` params, never f-strings
- Naming: PascalCase for DocTypes, snake_case for Python, camelCase for JS

### 3. hooks.py — golden patterns
```python
# doc_events — thin controller, delegate to service
doc_events = {
    "Sales Order": {
        "on_submit": "myapp.services.order_sync.on_submit",
        "on_cancel": "myapp.services.order_sync.on_cancel",
    }
}

# Scheduler events
scheduler_events = {
    "daily": ["myapp.tasks.daily_sync.run"],
    "cron": {
        "0 */2 * * *": ["myapp.tasks.monitor.run"],  # every 2h
    }
}

# Override whitelisted methods (use sparingly)
override_whitelisted_methods = {
    "frappe.client.get_list": "myapp.overrides.client.get_list"
}
```

### 4. Service layer pattern
```
myapp/
  myapp/
    services/
      order_sync.py   ← business logic (no frappe.form refs)
      external_api.py ← external API wrapper
    api/
      v1.py           ← @frappe.whitelist() endpoints (thin)
    utils/
      logger.py       ← frappe.logger() wrapper
      cache.py        ← frappe.cache() helpers
    tasks/
      daily_sync.py   ← scheduler entry points
```

### 5. Background jobs
```python
# Enqueue (fire and forget)
frappe.enqueue(
    "myapp.services.order_sync.sync_order",
    order_name=doc.name,
    queue="long",       # short / default / long
    timeout=1800,
    job_id=f"sync_order_{doc.name}",  # dedup key
    is_async=True,
)

# Inside the job — always log, never raise silently
def sync_order(order_name):
    logger = frappe.logger("order_sync", allow_site=True)
    try:
        doc = frappe.get_doc("Sales Order", order_name)
        # ... work ...
        frappe.db.commit()
    except Exception:
        logger.error(frappe.get_traceback())
        raise
```

### 6. Whitelisted REST APIs
```python
import frappe

@frappe.whitelist()
def get_order_status(order_name: str) -> dict:
    frappe.has_permission("Sales Order", "read", order_name, throw=True)
    doc = frappe.get_doc("Sales Order", order_name)
    return {
        "name": doc.name,
        "status": doc.status,
        "customer": doc.customer,
    }
```

### 7. Caching
```python
# Per-site, TTL in seconds
cache_key = f"myapp:config:{frappe.local.site}"
config = frappe.cache().get_value(cache_key)
if not config:
    config = frappe.get_single("My App Settings").as_dict()
    frappe.cache().set_value(cache_key, config, expires_in_sec=300)
```

### 8. Permission pattern
```python
# Role check
frappe.only_for(["System Manager", "My App Manager"])

# Document-level
frappe.has_permission("Sales Order", "write", order_name, throw=True)
```

### 9. Tests
```python
import frappe
from frappe.tests.utils import FrappeTestCase

class TestOrderSync(FrappeTestCase):
    def test_sync_sets_status(self):
        doc = frappe.get_doc({
            "doctype": "Sales Order",
            "customer": "Test Customer",
            # ...
        }).insert()
        result = sync_order(doc.name)
        self.assertEqual(result["status"], "Synced")
```

### 10. Migrations (safe)
```bash
# Run migrate after every schema change
bench --site <site> migrate
bench build --app <app_name>
```

## Security guardrails (HARD RULES)

- **NEVER** `frappe.db.sql("... WHERE name = '%s'" % name)` — use `%s` params
- **NEVER** disable CSRF (`frappe.flags.ignore_csrf = True`) in production
- **NEVER** store secrets in DocType fields — use `site_config.json` + `frappe.conf.get()`
- **ALWAYS** validate file uploads (type, size, extension) before save
- **ALWAYS** use `frappe.only_for()` or `frappe.has_permission()` on sensitive APIs
- **ALWAYS** run `bench --site <site> migrate` after schema changes — never touch the DB directly

## Production patterns

- Use `frappe.enqueue` with `job_id` for dedup on high-frequency events
- Keep app hooks thin — delegate to a service layer that has no `frappe.form` references
- Specify `--site <site>` explicitly in every bench command on multi-site benches
- Prefer Frappe-native workflow + notification primitives over custom cron hacks
