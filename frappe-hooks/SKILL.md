---
name: frappe-hooks
description: >
  Configure hooks.py correctly — doc_events, scheduler_events, override_whitelisted_methods,
  override_doctype_class, jinja, boot_session, permission_query_conditions, has_permission,
  and fixtures. Use when wiring app behaviour into Frappe's lifecycle without patching core.
  Trigger on: "hooks.py", "doc_events", "scheduler_events", "override method", "permission query
  conditions", "boot session", "fixtures".
---

# Frappe hooks.py

`hooks.py` is how an app plugs into Frappe without touching core. Every entry is a dotted path to a
function in your app.

## doc_events (document lifecycle)

```python
doc_events = {
    "Sales Order": {
        "validate":   "myapp.events.so.validate",
        "before_save":"myapp.events.so.before_save",
        "on_submit":  "myapp.events.so.on_submit",
        "on_cancel":  "myapp.events.so.on_cancel",
        "on_trash":   "myapp.events.so.on_trash",
    },
    "*": {  # applies to every DocType
        "on_update": "myapp.audit.track",
    },
}
```
Handler signature: `def on_submit(doc, method): ...`. Raise `frappe.ValidationError` to block.

## scheduler_events

```python
scheduler_events = {
    "all":     ["myapp.tasks.heartbeat.run"],      # every ~4 min
    "hourly":  ["myapp.tasks.hourly.run"],
    "daily":   ["myapp.tasks.daily.run"],
    "weekly":  ["myapp.tasks.weekly.run"],
    "cron": {
        "*/15 * * * *": ["myapp.tasks.poll.run"],
        "0 2 * * *":    ["myapp.tasks.nightly.run"],
    },
}
```
Scheduler must be enabled: `bench --site <site> enable-scheduler`.

## Overrides

```python
# Replace a whitelisted endpoint
override_whitelisted_methods = {
    "frappe.client.get_list": "myapp.overrides.get_list",
}

# Swap the controller class (subclass the original)
override_doctype_class = {
    "Sales Order": "myapp.overrides.sales_order.CustomSalesOrder",
}

# v16+: multiple apps can extend the same controller without replacing it
extend_doctype_class = {
    "Sales Order": "myapp.overrides.sales_order.SalesOrderExtension",
}
```

```python
# myapp/overrides/sales_order.py
from erpnext.selling.doctype.sales_order.sales_order import SalesOrder

class CustomSalesOrder(SalesOrder):
    def validate(self):
        super().validate()
        # extra logic
```

## Row-level security

```python
# Restrict list/report rows by SQL condition
permission_query_conditions = {
    "Sales Order": "myapp.perms.so_query_conditions",
}
has_permission = {
    "Sales Order": "myapp.perms.so_has_permission",
}
```
```python
def so_query_conditions(user):
    user = user or frappe.session.user
    return f"`tabSales Order`.owner = {frappe.db.escape(user)}"
```

## Other useful hooks

```python
app_include_js  = ["/assets/myapp/js/custom.js"]
app_include_css = ["/assets/myapp/css/custom.css"]
boot_session    = "myapp.boot.extend_boot"     # add data to frappe.boot
jenv = {"methods": ["fmt:myapp.utils.fmt"]}    # custom jinja filters/methods
fixtures = ["Custom Field", {"dt": "Role", "filters": [["name", "in", ["My Role"]]]}]

# Jinja methods/filters for print formats & web templates
jinja = {
    "methods": ["myapp.jinja.methods", "myapp.utils.get_fullname"],
    "filters": ["myapp.jinja.filters", "myapp.utils.format_currency"],
}

# Standard Web Form assets (Standard Web Forms only)
webform_include_js  = {"ToDo": "public/js/custom_todo.js"}
webform_include_css = {"ToDo": "public/css/custom_todo.css"}
```

## Scheduler event types (official)

Beyond `all`/`hourly`/`daily`/`weekly`/`monthly`, Frappe provides `_long` variants that run on the long worker:

```python
scheduler_events = {
    "daily_long": ["myapp.tasks.take_backups_daily"],
    "cron": {"0 2 * * *": ["myapp.tasks.nightly"]},
}
```

After changing `scheduler_events`, run `bench --site <site> migrate` for changes to take effect.

For user-configurable intervals, create `Scheduler Event` + `Scheduled Job Type` records (no hook required).

## Hook resolution

Hooks cascade across installed apps — `frappe.get_hooks()` collects values from all apps. Conflicting overrides use **last installed app wins**; reorder via Installed Applications → Update Hooks Resolution Order.

## Rules

- Keep handlers thin — call into a service module, don't put business logic inline in `hooks.py`.
- After editing `hooks.py`, run `bench --site <site> migrate` (or `bench build` for assets).
- `"*"` doc_events fire on every save — keep them cheap.
- Prefer `override_doctype_class` / `doc_events` over runtime monkey-patching — last-resort runtime patches need `_original_*` backups and upgrade tests (see `skills/frappe/monkey-patching-overrides.md`).

## From Frappe docs

- Full hooks reference (`permission_query_conditions`, `has_permission`, assets, install): https://docs.frappe.io/framework/user/en/python-api/hooks
- Background jobs + scheduler events: https://docs.frappe.io/framework/user/en/api/background_jobs
