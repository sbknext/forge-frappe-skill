---
title: Frappe Ref — Frappe Syntax: Customapp: Syntax (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'customapp', 'reference', 'syntax']
source: Impertio
---

# Custom App — Syntax Quick Reference

## __init__.py

```python
# my_custom_app/__init__.py — REQUIRED
__version__ = "0.0.1"
```

---

## pyproject.toml [v15+]

```toml
[build-system]
requires = ["flit_core >=3.4,<4"]
build-backend = "flit_core.buildapi"

[project]
name = "my_custom_app"
authors = [{ name = "Company", email = "dev@example.com" }]
description = "Description"
requires-python = ">=3.10"
readme = "README.md"
dynamic = ["version"]
dependencies = []

[tool.bench.frappe-dependencies]
frappe = ">=15.0.0,<16.0.0"
```

---

## setup.py [v14]

```python
from setuptools import setup, find_packages

setup(
    name="my_custom_app",
    version="0.0.1",
    description="Description",
    author="Company",
    author_email="dev@example.com",
    packages=find_packages(),
    zip_safe=False,
    include_package_data=True,
    install_requires=[],
)
```

---

## hooks.py (Minimal)

```python
app_name = "my_custom_app"
app_title = "My Custom App"
app_publisher = "Company"
app_description = "Description"
app_email = "dev@example.com"
app_license = "MIT"

required_apps = ["frappe"]

fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My Custom App"]]},
]
```

---

## modules.txt

```
My Custom App
Integrations
Settings
```

---

## patches.txt

```ini
[pre_model_sync]
myapp.patches.v1_0.backup_data

[post_model_sync]
myapp.patches.v1_0.populate_fields
```

---

## Patch Function

```python
import frappe

def execute():
    """One-line description."""
    pass
```

---

## Fixture Hook

```python
fixtures = [
    "Category",
    {"dt": "Custom Field", "filters": [["module", "=", "My Module"]]},
    {"dt": "Property Setter", "filters": [["module", "=", "My Module"]]},
]
```

---

## Bench Commands

```bash
bench new-app my_custom_app
bench --site mysite install-app my_custom_app
bench get-app https://github.com/org/app
bench --site mysite migrate
bench build --app my_custom_app
bench --site mysite export-fixtures --app my_custom_app
bench --site mysite clear-cache
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-customapp/references/syntax.md*
