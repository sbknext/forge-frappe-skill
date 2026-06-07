---
title: Frappe Ref — Frappe Syntax: Customapp: Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'customapp', 'reference', 'methods']
source: Impertio
---

# Custom App — Bench Command Reference

## App Creation

```bash
# Create new app (interactive prompts for metadata)
bench new-app my_custom_app
# Prompts: Title, Description, Publisher, Email, Icon, Color, License

# Create app in a specific directory (rarely needed)
cd apps && bench new-app my_custom_app
```

---

## App Installation

```bash
# Install app on a site
bench --site mysite install-app my_custom_app

# Uninstall app from a site
bench --site mysite uninstall-app my_custom_app

# List installed apps
bench --site mysite list-apps
```

---

## Getting Apps from Git

```bash
# Get app from GitHub
bench get-app https://github.com/org/my_custom_app

# Get specific branch
bench get-app https://github.com/org/my_custom_app --branch develop

# Get specific tag/version
bench get-app https://github.com/org/my_custom_app --branch v1.0.0
```

---

## Migration & Build

```bash
# Run migrations (patches + schema sync + fixtures)
bench --site mysite migrate

# Skip failing patches (NEVER in production)
bench --site mysite migrate --skip-failing

# Build frontend assets for specific app
bench build --app my_custom_app

# Build all apps
bench build

# Clear cache
bench --site mysite clear-cache
```

---

## Patch Management

```bash
# Create a new patch interactively
bench create-patch
# Prompts: App, DocType, Description, Filename

# Run a specific patch manually (development only)
bench --site mysite run-patch myapp.patches.v1_0.my_patch
```

---

## Fixture Management

```bash
# Export fixtures for specific app
bench --site mysite export-fixtures --app my_custom_app

# Export fixtures for all apps
bench --site mysite export-fixtures
```

---

## Development Commands

```bash
# Start development server
bench start

# Watch for file changes and auto-rebuild
bench watch

# Run tests for an app
bench --site mysite run-tests --app my_custom_app

# Run specific test
bench --site mysite run-tests --module myapp.my_module.doctype.my_doctype.test_my_doctype
```

---

## Programmatic App Info

```python
import frappe

# Get installed apps
apps = frappe.get_installed_apps()

# Get app version
version = frappe.get_attr("my_custom_app.__version__")

# Get app hooks
hooks = frappe.get_hooks(app_name="my_custom_app")

# Check if app is installed
is_installed = "my_custom_app" in frappe.get_installed_apps()
```

---

## required_apps in hooks.py

```python
# hooks.py — declare app dependencies
required_apps = ["frappe"]                   # Frappe-only app
required_apps = ["frappe", "erpnext"]        # ERPNext extension
required_apps = ["frappe", "erpnext", "hrms"]  # HRMS extension
```

`required_apps` is checked during `bench get-app` — missing dependencies are auto-installed.

---

## Version Check Pattern

```python
import frappe

frappe_version = int(frappe.__version__.split(".")[0])

if frappe_version >= 15:
    # v15+ specific code
    pass
else:
    # v14 fallback
    pass
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-customapp/references/methods.md*
