---
title: Frappe Ref — Frappe Syntax: Hooks: Bootinfo (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'reference', 'bootinfo']
source: Impertio
---

# Boot & Session Hooks Reference

Complete reference for `extend_bootinfo`, `notification_config`, and session
hooks in hooks.py.

---

## extend_bootinfo

Inject global values into `frappe.boot` that are available in client-side
JavaScript on every page load.

### Syntax

```python
# hooks.py
extend_bootinfo = "myapp.boot.extend_boot"
```

### Handler Signature

```python
def extend_boot(bootinfo):
    """
    Args:
        bootinfo: frappe.boot object (dict-like).
                  Values added here appear in frappe.boot on the client.
    """
```

### Implementation Examples

#### Feature Flags

```python
# myapp/boot.py
import frappe

def extend_boot(bootinfo):
    settings = frappe.get_single("My App Settings")
    bootinfo.feature_flags = {
        "new_dashboard": settings.enable_new_dashboard,
        "beta_features": settings.enable_beta
    }
```

```javascript
// Client-side access
if (frappe.boot.feature_flags.new_dashboard) {
    load_new_dashboard();
}
```

#### User Permissions Cache

```python
def extend_boot(bootinfo):
    user = frappe.session.user
    bootinfo.custom_permissions = {
        "can_approve_orders": has_approval_rights(user),
        "max_discount_percent": get_max_discount(user),
        "allowed_warehouses": get_user_warehouses(user)
    }
```

#### Company Defaults

```python
def extend_boot(bootinfo):
    if frappe.session.user != "Guest":
        default_company = frappe.defaults.get_user_default("Company")
        if default_company:
            bootinfo.company_settings = frappe.db.get_value(
                "Company",
                default_company,
                ["default_currency", "country", "tax_id"],
                as_dict=True
            )
```

### What frappe.boot Contains by Default

| Property | Content |
|----------|---------|
| `frappe.boot.user` | Current user info |
| `frappe.boot.home_page` | Home page route |
| `frappe.boot.user_info` | User metadata |
| `frappe.boot.lang` | Active language |
| `frappe.boot.sysdefaults` | System defaults |
| `frappe.boot.notification_settings` | Notification config |
| `frappe.boot.modules` | Available modules |
| `frappe.boot.desk_settings` | Desk configuration |

---

## notification_config

Configure client-side notification behavior.

### Syntax

```python
# hooks.py
notification_config = "myapp.notifications.get_config"
```

### Implementation

```python
# myapp/notifications.py
def get_config():
    return {
        "for_doctype": {
            "Sales Order": {"status": ("!=", "Completed")},
            "Issue": {"status": "Open"}
        },
        "for_module": {
            "Selling": {"color": "orange", "count": get_open_orders}
        }
    }

def get_open_orders():
    return frappe.db.count("Sales Order", {"status": "Draft"})
```

---

## Session & Authentication Hooks

### on_login

Triggered immediately after successful authentication.

```python
# hooks.py
on_login = "myapp.auth.on_login"

# myapp/auth.py
def on_login(login_manager):
    """
    Args:
        login_manager: frappe.auth.LoginManager instance
                       login_manager.user = username
    """
    user = login_manager.user
    frappe.db.set_value("User", user, "last_login_ip", frappe.local.request_ip)
```

### on_logout

Triggered when user logs out.

```python
# hooks.py
on_logout = "myapp.auth.on_logout"

# myapp/auth.py
def on_logout():
    """No arguments. Use frappe.session for context."""
    frappe.cache().delete_key(f"user_cache:{frappe.session.user}")
```

### on_session_creation

Triggered when a new session is created (after login).

```python
# hooks.py
on_session_creation = "myapp.auth.on_session_creation"

# myapp/auth.py
def on_session_creation():
    """No arguments. Use frappe.session for context."""
    frappe.logger().info(f"New session: {frappe.session.user}")
```

### auth_hooks

Request authentication validators. Called on EVERY authenticated request.

```python
# hooks.py
auth_hooks = ["myapp.auth.validate_request"]

# myapp/auth.py
def validate_request():
    """Called on every request. Throw to block access."""
    if is_blocked_ip(frappe.local.request_ip):
        frappe.throw("Access denied", frappe.AuthenticationError)
```

---

## Execution Order

```
1. User login successful
2. on_login hook (receives login_manager)
3. Session created
4. on_session_creation hook (no args)
5. Boot data collected
6. extend_bootinfo hook (receives bootinfo)
7. frappe.boot sent to client
```

---

## Critical Rules

### NEVER Put Secrets in Bootinfo

```python
# WRONG — API keys/secrets exposed to browser JavaScript
def extend_boot(bootinfo):
    bootinfo.api_key = frappe.get_single("Settings").secret_key

# CORRECT — only public configuration
def extend_boot(bootinfo):
    bootinfo.app_version = "1.0.0"
    bootinfo.feature_flags = {"new_ui": True}
```

### NEVER Run Heavy Queries in Bootinfo

```python
# WRONG — runs on EVERY page load
def extend_boot(bootinfo):
    bootinfo.all_customers = frappe.get_all("Customer")  # Thousands of records!

# CORRECT — minimal, cached data
def extend_boot(bootinfo):
    cache_key = f"customer_count:{frappe.session.user}"
    count = frappe.cache().get_value(cache_key)
    if count is None:
        count = frappe.db.count("Customer")
        frappe.cache().set_value(cache_key, count, expires_in_sec=3600)
    bootinfo.customer_count = count
```

### ALWAYS Guard Against Guest Users

```python
def extend_boot(bootinfo):
    if frappe.session.user == "Guest":
        return  # NEVER load user-specific data for guests
    bootinfo.user_settings = get_user_settings()
```

---

## Debugging

### Inspect Boot Data in Browser

```javascript
// In browser console
console.log(frappe.boot);
console.log(JSON.stringify(frappe.boot.my_custom_key, null, 2));
```

### Test Server-Side

```python
# In bench console
bootinfo = frappe._dict()
from myapp.boot import extend_boot
extend_boot(bootinfo)
print(bootinfo)
```

---

## Version Differences

| Feature | v14 | v15 | v16 |
|---------|-----|-----|-----|
| extend_bootinfo | Yes | Yes | Yes |
| notification_config | Yes | Yes | Yes |
| on_session_creation | Yes | Yes | Yes |
| on_login / on_logout | Yes | Yes | Yes |
| auth_hooks | Yes | Yes | Yes |
| Boot data compression | Basic | Improved | Improved |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks/references/bootinfo.md*
