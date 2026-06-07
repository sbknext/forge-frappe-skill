---
title: Frappe Ref — Frappe Syntax: Serverscripts: Hooks (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'serverscripts', 'reference', 'hooks']
source: Impertio
---

# Server Scripts vs hooks.py — Interaction and Differences

## How Server Scripts Relate to hooks.py

Server Scripts and `hooks.py` doc_events target the SAME document lifecycle
hooks. They run in a defined order:

```
Document Lifecycle Event (e.g., validate)
│
├─► 1. Controller method  (e.g., SalesInvoice.validate())
├─► 2. hooks.py handler   (e.g., doc_events["Sales Invoice"]["validate"])
└─► 3. Server Script      (e.g., Before Save on Sales Invoice)
```

**ALWAYS** be aware that Server Scripts run AFTER controller methods and
hooks.py handlers. If a controller throws an error, the Server Script NEVER
executes.

---

## When to Use Which

| Criterion | Server Script | hooks.py / Controller |
|---|---|---|
| Deployment | UI only (no bench access) | Requires custom Frappe app |
| Python imports | BLOCKED (sandbox) | Full Python available |
| External libraries | Only `frappe.make_*_request()` | Any pip package |
| File system access | BLOCKED | Full access |
| Unit testing | Not testable | Fully testable with pytest |
| Version control | Stored in DB | Stored in code (Git) |
| Performance | Slightly slower (sandbox overhead) | Native Python speed |
| Complexity limit | Simple logic (<50 lines ideal) | Unlimited |
| Available hooks | 11 document events + API/Scheduler/Permission | All hooks including autoname, on_change, etc. |

---

## hooks.py doc_events Format

For comparison — this is how the same logic looks in hooks.py:

```python
# In your_app/hooks.py
doc_events = {
    "Sales Invoice": {
        "validate": "your_app.overrides.sales_invoice.validate",
        "on_submit": "your_app.overrides.sales_invoice.on_submit",
    }
}
```

```python
# In your_app/overrides/sales_invoice.py
import frappe  # Imports WORK in controllers
from datetime import date  # Any Python module available

def validate(doc, method):
    if doc.grand_total < 0:
        frappe.throw("Total cannot be negative")

def on_submit(doc, method):
    # Full Python available here
    pass
```

### Key Difference: Method Signature

| Context | Signature |
|---|---|
| Server Script | No function definition — `doc` is a global variable |
| hooks.py handler | `def handler(doc, method):` — doc is a parameter |
| Controller method | `def validate(self):` — doc is `self` |

---

## Coexistence Rules

1. **Multiple handlers on same event**: Controller + hooks.py + Server Script
   ALL run for the same event. They do NOT cancel each other.

2. **Execution order is fixed**: Controller → hooks.py → Server Script.
   NEVER rely on a Server Script running before a controller method.

3. **If any handler throws**: Subsequent handlers do NOT run. The entire
   save/submit/cancel operation is rolled back.

4. **Multiple Server Scripts**: If two Server Scripts target the same
   DocType + Event, execution order between them is UNDEFINED.

5. **Data visibility**: All handlers see the same `doc` object. Changes
   made by the controller are visible to hooks.py and Server Scripts.

---

## Migrating Between Server Scripts and hooks.py

### Server Script → hooks.py

When a Server Script becomes too complex:

1. Create a custom Frappe app: `bench new-app my_customizations`
2. Move logic to a Python file with proper imports
3. Register in `hooks.py` doc_events
4. Delete the Server Script from the UI
5. Run `bench migrate` to apply hooks

### hooks.py → Server Script

When you need to let non-developers manage logic:

1. ONLY migrate if the logic fits sandbox restrictions
2. Remove `import` statements — use `frappe.*` namespace
3. Remove function definition — `doc` becomes a global
4. Remove `self` references — use `doc` directly
5. Create Server Script in UI with appropriate event
6. Remove the hooks.py entry

---

## Server Scripts and Custom Apps Together

A common architecture:

```
Custom App (hooks.py)          Server Scripts (UI)
├── Complex validation         ├── Simple field validation
├── External API integration   ├── Quick API endpoints
├── Background jobs            ├── Permission queries
├── Custom report logic        └── Scheduled reminders
└── Unit-tested business rules
```

**Best practice**: Use Server Scripts for logic that business users may need to
adjust without deploying code. Use custom apps for core business logic that
must be version-controlled and tested.

---

## Version Differences

| Feature | v14 | v15+ |
|---|---|---|
| Server Scripts enabled | By default | Must enable explicitly |
| hooks.py doc_events | Fully supported | Fully supported |
| Execution order | Controller → hooks → Script | Same |
| `run_script()` | Available (v13+) | Available |
| Server Script export/import | Via fixtures | Via fixtures |

---

## Exporting Server Scripts (Version Control)

Server Scripts live in the database by default. To version-control them:

```python
# In hooks.py — export Server Scripts as fixtures
fixtures = [
    {"dt": "Server Script", "filters": [["module", "=", "My Module"]]}
]
```

Then: `bench export-fixtures` creates JSON files in your app that can be
committed to Git. `bench migrate` re-imports them.

**ALWAYS** export Server Scripts as fixtures if you need reproducible
deployments across environments.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-serverscripts/references/hooks.md*
