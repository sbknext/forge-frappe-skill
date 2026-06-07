---
name: frappe-app-development
description: >
  Scaffold, architect, and build production-grade custom Frappe/ERPNext apps. Covers app structure,
  DocType design, hooks.py configuration, background jobs, service layers, REST APIs, permissions,
  and ElasticRun-specific patterns. Use when creating a new Frappe app, adding a module, setting up
  scheduler events, implementing doc_events, writing whitelisted APIs, or debugging Frappe-specific
  issues. Trigger on: "banao Frappe app", "DocType banana hai", "hook lagao", "background job", "frappe.enqueue",
  "whitelist API", "custom app scaffold", "frappe new app".
---

# Frappe App Development

Scaffold and build production-grade custom Frappe / ERPNext applications with correct separation of
concerns, tested patterns, and ElasticRun-grade guardrails.

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
- Site name (e.g., `withrun.saas`, `beta-logi-libera.elasticrun.in`)

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
    "Purchase Order": {
        "on_submit": "myapp.services.po_sync.on_po_submit",
        "on_cancel": "myapp.services.po_sync.on_po_cancel",
    }
}

# Scheduler events
scheduler_events = {
    "daily": ["myapp.tasks.daily_sync.run"],
    "cron": {
        "0 */2 * * *": ["myapp.tasks.po_monitor.run"],  # every 2h
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
      po_sync.py      ← business logic (no frappe.form refs)
      zoho_client.py  ← external API wrapper
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
    "myapp.services.po_sync.sync_po",
    po_name=po.name,
    queue="long",       # short / default / long
    timeout=1800,
    job_id=f"sync_po_{po.name}",  # dedup key
    is_async=True,
)

# Inside the job — always log, never raise silently
def sync_po(po_name):
    logger = frappe.logger("po_sync", allow_site=True)
    try:
        po = frappe.get_doc("Purchase Order", po_name)
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
def get_po_status(po_name: str) -> dict:
    frappe.has_permission("Purchase Order", "read", po_name, throw=True)
    po = frappe.get_doc("Purchase Order", po_name)
    return {
        "name": po.name,
        "status": po.status,
        "supplier": po.supplier,
    }
```

### 7. Caching
```python
# Per-site, TTL in seconds
cache_key = f"myapp:po_config:{frappe.local.site}"
config = frappe.cache().get_value(cache_key)
if not config:
    config = frappe.get_single("My App Settings").__dict__
    frappe.cache().set_value(cache_key, config, expires_in_sec=300)
```

### 8. Permission pattern
```python
# Role check
frappe.only_for(["System Manager", "My App Manager"])

# Document-level
frappe.has_permission("Purchase Order", "write", po_name, throw=True)
```

### 9. Tests
```python
import frappe
from frappe.tests.utils import FrappeTestCase

class TestPOSync(FrappeTestCase):
    def test_sync_creates_zoho_entry(self):
        po = frappe.get_doc({
            "doctype": "Purchase Order",
            "supplier": "Test Supplier",
            # ...
        }).insert()
        result = sync_po(po.name)
        self.assertEqual(result["status"], "Synced")
```

### 10. Migrations (safe)
```bash
# Always dry-run before apply
bench --site <site> migrate --dry-run
bench --site <site> migrate
bench build --app <app_name>
```

## Security guardrails (HARD RULES)

- **NEVER** `frappe.db.sql("... WHERE name = '%s'" % name)` — use `%s` params
- **NEVER** disable CSRF (`frappe.flags.ignore_csrf = True` in prod)
- **NEVER** store secrets in DocType fields — use `site_config.json` + `frappe.conf.get()`
- **ALWAYS** validate file uploads (type, size, extension) before save
- **ALWAYS** use `frappe.only_for()` or `frappe.has_permission()` on sensitive APIs
- **ALWAYS** run `bench --site <site> migrate` after schema changes — never touch DB directly

## ElasticRun-specific notes

- Use `frappe.enqueue` with `job_id` for dedup on high-frequency events (PO submit, spine events)
- Spine/Kafka events hook via `frappe-spine` — don't write raw Kafka consumers in app hooks
- PowerFlow-based workflows → use `frappe-latte` Powerflow engine, not frappe.workflow directly
- Multi-site bench: always specify `--site <site>` in bench commands
- For ERPNext extensions use `epatch` conventions — `extend_doctype_class` pattern for v16+
