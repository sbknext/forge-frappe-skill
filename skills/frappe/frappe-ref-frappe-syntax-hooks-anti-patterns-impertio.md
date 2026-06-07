---
title: Frappe Ref — Frappe Syntax: Hooks: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns and Best Practices

Common mistakes in hooks.py configuration and how to avoid them.
For doc_events anti-patterns, see **frappe-syntax-hooks-events**.

---

## Scheduler Anti-Patterns

### NEVER Use Default Queue for Heavy Tasks

```python
# WRONG — timeout after 5 minutes
scheduler_events = {
    "daily": ["myapp.tasks.sync_all_records"]  # May take 20 min
}

# CORRECT — 25 minute timeout
scheduler_events = {
    "daily_long": ["myapp.tasks.sync_all_records"]
}
```

| Queue | Timeout |
|-------|---------|
| default (hourly, daily, etc.) | 300s (5 min) |
| long (hourly_long, daily_long, etc.) | 1500s (25 min) |

### NEVER Define Tasks with Arguments

```python
# WRONG — scheduler passes no arguments
def my_task(company_name):
    process_company(company_name)

# CORRECT — fetch data inside the function
def my_task():
    for company in frappe.get_all("Company"):
        process_company(company.name)
```

### ALWAYS Run bench migrate After Changes

```bash
# REQUIRED after any scheduler_events change
bench --site sitename migrate
```

Scheduler events are cached. Without migrate, changes are NOT picked up.

---

## Override Anti-Patterns

### NEVER Forget super() in Overrides

```python
# WRONG — parent validate skipped, core logic broken
class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        self.custom_validation()

# CORRECT — ALWAYS call super() first
class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        super().validate()
        self.custom_validation()
```

Consequence: Forgetting `super()` skips core validations, calculations, and
side effects. Ledger entries, tax calculations, and stock updates may break.

### NEVER Mismatch Method Signatures

```python
# WRONG — original has 4 params, override has 1
override_whitelisted_methods = {
    "frappe.client.get_count": "myapp.overrides.my_get_count"
}
def my_get_count(doctype):  # Missing parameters!
    return frappe.db.count(doctype)

# CORRECT — exact same signature
def my_get_count(doctype, filters=None, debug=False, cache=False):
    count = frappe.db.count(doctype, filters)
    return count
```

ALWAYS look up the original function signature before writing an override.

---

## Permission Anti-Patterns

### NEVER Skip the User None Check

```python
# WRONG — user can be None, causing errors
def my_query_conditions(user):
    return f"owner = {frappe.db.escape(user)}"

# CORRECT — ALWAYS check for None
def my_query_conditions(user):
    if not user:
        user = frappe.session.user
    return f"owner = {frappe.db.escape(user)}"
```

### NEVER Use Raw String Interpolation in SQL

```python
# WRONG — SQL injection vulnerability
def my_query_conditions(user):
    return f"owner = '{user}'"

# CORRECT — use frappe.db.escape
def my_query_conditions(user):
    if not user:
        user = frappe.session.user
    return f"owner = {frappe.db.escape(user)}"
```

### NEVER Expect get_all to Apply Permission Hooks

```python
# WRONG expectation — get_all IGNORES permission_query_conditions
frappe.db.get_all("Sales Invoice")

# CORRECT — get_list respects permission hooks
frappe.db.get_list("Sales Invoice")
```

---

## Fixture Anti-Patterns

### NEVER Export Fixtures Without Filters

```python
# WRONG — exports ALL custom fields from ALL apps
fixtures = ["Custom Field"]

# CORRECT — filter to your app's records only
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]}
]
```

Risk: Unfiltered exports include other apps' custom fields, causing conflicts.

### NEVER Put Transactional Data in Fixtures

```python
# WRONG — transactional data should not be in fixtures
fixtures = ["Sales Invoice", "Stock Entry"]

# CORRECT — only configuration data
fixtures = ["Custom Field", "Property Setter", "Role"]
```

---

## Boot Info Anti-Patterns

### NEVER Put Secrets in Bootinfo

```python
# WRONG — secrets exposed to browser JavaScript
def extend_boot(bootinfo):
    bootinfo.api_key = frappe.get_single("Settings").secret_key
    bootinfo.db_password = get_db_password()

# CORRECT — only public configuration
def extend_boot(bootinfo):
    bootinfo.app_version = "1.0.0"
    bootinfo.feature_flags = {"new_ui": True}
```

### NEVER Run Heavy Queries in Bootinfo

```python
# WRONG — runs on EVERY page load
def extend_boot(bootinfo):
    bootinfo.all_customers = frappe.get_all("Customer")

# CORRECT — minimal, cached data
def extend_boot(bootinfo):
    count = frappe.cache().get_value("customer_count")
    if count is None:
        count = frappe.db.count("Customer")
        frappe.cache().set_value("customer_count", count, expires_in_sec=3600)
    bootinfo.customer_count = count
```

---

## General Hook Anti-Patterns

### NEVER Import frappe at Module Level in hooks.py

```python
# WRONG — hooks.py is loaded before frappe is initialized
import frappe  # Will cause ImportError during early bootstrap

app_name = "myapp"
```

hooks.py uses plain Python assignments, not frappe API calls. All
`"myapp.module.function"` paths are resolved lazily at runtime.

### NEVER Use Lambdas or Inline Functions

```python
# WRONG — hooks must be dotted path strings
after_install = lambda: print("installed")

# CORRECT — dotted path to a named function
after_install = "myapp.setup.after_install"
```

### NEVER Skip bench migrate After Hook Changes

Any change to hooks.py requires:
```bash
bench --site sitename migrate
```

This applies to ALL hook types, not just scheduler_events.

---

## Best Practices Summary

### ALWAYS Do

| Practice | Reason |
|----------|--------|
| Call `super()` in overrides | Preserve core functionality |
| Run `bench migrate` after changes | Hook config is cached |
| Use `frappe.db.escape()` for SQL | Prevent injection |
| Use `_long` for heavy tasks | Prevent timeouts |
| Filter fixtures by module | Prevent conflicts |
| Check `if not user` in permission hooks | User can be None |
| Cache bootinfo data | Runs on every page load |
| Guard against Guest in bootinfo | Prevent data leaks |

### NEVER Do

| Anti-Pattern | Problem |
|-------------|---------|
| Heavy tasks in default queue | 5 min timeout |
| Tasks with arguments | Scheduler passes none |
| `get_all` with permission hooks | Ignores permissions |
| Secrets in bootinfo | Exposed to browser |
| Fixtures without filters | Exports too much |
| Override without `super()` | Breaks core logic |
| Raw SQL interpolation | Injection risk |
| Skip `bench migrate` | Changes not picked up |

---

## Debug Checklist

When hooks are not working:

1. **Migrate forgotten?**
   ```bash
   bench --site sitename migrate
   ```

2. **Scheduler not running?**
   ```bash
   bench --site sitename scheduler status
   bench --site sitename scheduler enable
   ```

3. **Cache stale?**
   ```bash
   bench --site sitename clear-cache
   ```

4. **Syntax error in hooks.py?**
   ```bash
   python -c "import myapp.hooks"
   ```

5. **Check logs:**
   ```bash
   tail -f ~/frappe-bench/logs/scheduler.log
   tail -f ~/frappe-bench/logs/worker.log
   ```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks/references/anti-patterns.md*
