---
title: Frappe Ref — Frappe Impl: Ui Components: Decision Tree (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'ui', 'components', 'reference', 'decision-tree']
source: Impertio
---

# UI Components — Decision Tree

## Which Component Do I Need?

```
START: What is the interaction?
│
├── USER NEEDS TO PROVIDE INPUT
│   ├── Single value (text, number, date)?
│   │   └── frappe.prompt (quick, lightweight)
│   ├── Multiple fields (form-like)?
│   │   └── frappe.ui.Dialog with fields array
│   ├── Select records from a DocType?
│   │   └── frappe.ui.form.MultiSelectDialog
│   ├── Tabular data entry (rows of items)?
│   │   └── frappe.ui.Dialog with Table fieldtype
│   └── Yes/No decision?
│       └── frappe.confirm
│
├── SYSTEM NEEDS TO SHOW INFORMATION
│   ├── User MUST acknowledge?
│   │   └── frappe.msgprint (modal, blocks interaction)
│   ├── Non-blocking notification?
│   │   └── frappe.show_alert (toast, auto-dismisses)
│   ├── Error that stops execution?
│   │   └── frappe.throw (client) / frappe.throw (server)
│   └── Background task progress?
│       └── frappe.publish_progress (server → client)
│
├── CUSTOMIZE AN EXISTING VIEW
│   ├── List View columns, indicators, buttons?
│   │   └── frappe.listview_settings in {doctype}_list.js
│   ├── Date-based record visualization?
│   │   └── Calendar View via {doctype}_calendar.js
│   ├── Workflow status board (drag-drop)?
│   │   └── Kanban Board (requires Select field on DocType)
│   ├── Hierarchical parent-child display?
│   │   └── Tree View (requires is_tree on DocType)
│   └── Custom cell/value formatting?
│       └── formatters in listview_settings or form
│
├── BUILD A COMPLETE NEW PAGE
│   ├── Dashboard or tool page?
│   │   └── frappe.ui.Page (full toolbar, sidebar, body)
│   ├── Report-style page?
│   │   └── Query Report or Script Report (see frappe-impl-reports)
│   └── Public-facing page?
│       └── Portal page in www/ (see frappe-impl-website)
│
├── LIVE UPDATES WITHOUT REFRESH
│   ├── Notify specific user?
│   │   └── frappe.publish_realtime(event, data, user=email)
│   ├── Update all viewers of a document?
│   │   └── frappe.publish_realtime(event, data, doctype=dt, docname=name)
│   ├── Broadcast to all users?
│   │   └── frappe.publish_realtime(event, data)
│   └── Show progress bar?
│       └── frappe.publish_progress(percent, title, description)
│
└── SCAN INPUT
    ├── Single barcode/QR lookup?
    │   └── frappe.ui.Scanner({ dialog: true, multiple: false })
    └── Continuous scanning (warehouse)?
        └── frappe.ui.Scanner({ dialog: true, multiple: true })
```

## Message Type Selection

```
What kind of message?
├── Error (must stop)           → frappe.throw("message")
├── Warning (must acknowledge)  → frappe.msgprint({ indicator: "orange" })
├── Info (must acknowledge)     → frappe.msgprint("message")
├── Success (non-blocking)      → frappe.show_alert({ indicator: "green" })
├── Confirm before action       → frappe.confirm("question", yes_fn, no_fn)
└── Quick input needed          → frappe.prompt(fields, callback, title)
```

## Dialog Size Selection

```
How much content in the dialog?
├── 1-3 simple fields          → size: "small"
├── 4-8 fields, no table       → (default, no size needed)
├── Table field or many fields → size: "large"
└── Complex multi-section form → size: "extra-large"
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-ui-components/references/decision-tree.md*
