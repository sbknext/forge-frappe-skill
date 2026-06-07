---
title: Frappe Ref — Script Types & Customization Export (frappe-apps-manager)
category: frappe
tags: [frappe, client-script, server-script, custom-script, doctype_js, fixtures, customization]
source: vyogotech/frappe-apps-manager
---

# Script Types & Customization Export

## When to use

Trigger when choosing between **Client Script** (Desk DB record), **app JS via `doctype_js`**, **Server Script** (sandbox), **controller/hooks**, or **Custom Field / Property Setter** fixtures — and when exporting customizations across sites.

## Decision tree — where does logic live?

```
Must logic ALWAYS run (API, import, bench console, background job)?
├── YES → server only
│         ├── Can install a custom app? → controller + hooks.py (preferred)
│         └── UI-only bench access? → Server Script (sandbox limits apply)
└── NO → desk UX
          ├── Form/list interactivity → Client Script OR doctype_js
          └── Simple one-off tweak → Client Script (Desk) OK for prototypes
```

### Client Script (Desk) vs doctype_js (app file)

| Factor | Desk Client Script | `hooks.py` → `doctype_js` |
|--------|-------------------|---------------------------|
| Version control | Stored in DB | Git-tracked `.js` in app |
| Multi-site deploy | Manual export/sync | `bench migrate` + app install |
| Team review / CI | Weak | Strong |
| Rule of thumb | Prototypes, single-site tweaks | **>50 lines**, reusable logic, production apps |

Legacy name **Custom Script** (pre-v13) = today's **Client Script** DocType — same Desk UI location under Customize.

### Server Script vs controller

```
Need Python on a DocType?
├── Can install custom app?
│   ├── imports, filesystem, complex classes, unit tests → Document controller
│   └── simple validate/calc, no imports → either works
└── No bench CLI (hosted UI only)?
    └── Server Script (see sandbox rules below)
```

**Server Script sandbox (v14–v16):**

- **ALL `import` statements blocked** — everything must use the pre-loaded namespace.
- v15+ / v16: disabled by default → `bench set-config -g server_script_enabled 1`
- Not available on shared Frappe Cloud benches.
- Never `doc.save()` inside Before Save scripts; never `frappe.db.commit()` in Document Events (only Scheduler).

**Import substitutions (blocked → use instead):**

| Blocked | Use instead |
|---------|-------------|
| `import json` | `frappe.parse_json()` / `frappe.as_json()` |
| `from datetime import date` | `frappe.utils.today()` / `frappe.utils.now_datetime()` |
| `import requests` | `frappe.make_get_request()` / `frappe.make_post_request()` |
| `import re` | Restructure without regex (not available) |
| `import os` / `import sys` | Use a custom app controller instead |

**Document Event UI → hook mapping (critical):**

| Server Script UI | Internal hook | Fires when |
|------------------|---------------|------------|
| Before Insert | `before_insert` | Before new doc saved to DB |
| After Insert | `after_insert` | After first DB insert |
| Before Validate | `before_validate` | Before framework validation |
| **Before Save** | **`validate`** | Before save (new + update) |
| After Save | `on_update` | After successful save |
| Before Submit | `before_submit` | Before submit (docstatus 0→1) |
| After Submit | `on_submit` | After submit completes |
| Before Cancel | `before_cancel` | Before cancel |
| After Cancel | `on_cancel` | After cancel completes |
| Before Delete | `on_trash` | Before permanent delete |

UI label **Before Save** maps to `validate`, **not** `before_save` (which runs after `validate`).

### Customization without forking core JSON

| Need | Mechanism | Ship via |
|------|-----------|----------|
| New field on existing DocType | Custom Field | `create_custom_fields()` in `after_install` + filtered fixtures |
| Hide/relabel/reqd existing field | Property Setter | `make_property_setter()` + fixtures |
| Behavior only | `doc_events`, Server Script, Client Script | hooks or Desk |
| Replace controller class | `override_doctype_class` | hooks (own app only) |
| Add methods without replace | `extend_doctype_class` | hooks (v16+) |

## Exporting & managing customizations

### Filtered fixtures (critical)

Never export all Custom Fields on a site:

```python
fixtures = [
    {
        "dt": "Custom Field",
        "filters": [["fieldname", "in", ["custom_tracking_id", "custom_vendor_ref"]]],
    },
    {
        "dt": "Property Setter",
        "filters": [["module", "=", "My App"]],
    },
    {"doctype": "Client Script", "filters": [["module", "=", "My App"]]},
    {"doctype": "Server Script", "filters": [["module", "=", "My App"]]},
]
```

Export: `bench --site mysite export-fixtures` → files in `my_app/fixtures/` → applied on `bench migrate`.

### Frappe Packages (v14+) vs apps

| | Frappe App | Frappe Package |
|---|------------|----------------|
| Structure | `pyproject.toml`, git repo | UI-built tarball |
| Custom Fields on standard DocTypes | Fixtures in app | Use fixtures, not package alone |
| Client/Server Scripts in custom modules | Yes | Yes (packageable) |
| Custom DocTypes | Yes | Yes |

Packages suit UI-built bundles; apps suit versioned development with filtered fixtures.

### Developer mode + DocType export

```bash
bench --site mysite set-config developer_mode 1
# Edit DocType in Desk → Export writes JSON to apps/<app>/...
bench --site mysite migrate
```

Turn off developer mode on production after export workflow.

### Client Script export

- Prefer **`doctype_js`** for production; export Desk Client Scripts only when intentionally shipping DB-stored scripts.
- Module field on Client Script must match your app module for fixture filters.

### Property Setter / Custom Field rules

- Prefix custom fieldnames with `custom_`.
- Always set `insert_after` for field order.
- Never change `fieldtype` on fields with existing data via Property Setter — risk of data loss.
- Never edit another app's DocType `.json` — use Custom Field when the doctype lives in `frappe`/`erpnext`.

## Client vs server validation

| Layer | Purpose |
|-------|---------|
| Client `validate` / `frappe.throw` | UX — instant feedback |
| Server `validate` / permissions | **Authoritative** — API, import, console |

Always pair client validation with server checks when data integrity matters.

## Version notes

| Feature | v13 | v14 | v15 | v16 |
|---------|-----|-----|-----|-----|
| Client Script DocType | Yes | Yes | Yes | Yes |
| Server Script default | enabled | enabled | **disabled** | disabled |
| `extend_doctype_class` | — | — | — | Yes |
| `frappe.qb` in Server Script sandbox | — | Yes | Yes | Yes |

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-client-script-logic/SKILL.md
- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-syntax-serverscripts/SKILL.md
- https://github.com/Venkateshvenki404224/frappe-apps-manager/blob/main/frappe-apps-manager/skills/frappe-doctype-architect/SKILL.md
