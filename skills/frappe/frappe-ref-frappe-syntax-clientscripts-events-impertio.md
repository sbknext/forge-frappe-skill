---
title: Frappe Ref — Frappe Syntax: Clientscripts: Events (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'clientscripts', 'reference', 'events']
source: Impertio
---

# Client Script Events Reference

## Event Execution Order

### On Form Load (new or existing document)
```
setup → onload → refresh → onload_post_render
```

### On Save (new document)
```
validate → before_save → [server save] → after_save → refresh
```

### On Save (existing document)
```
validate → before_save → [server save] → after_save → refresh
```

### On Submit (docstatus 0 → 1)
```
validate → before_submit → [server submit] → on_submit → refresh
```

### On Cancel (docstatus 1 → 2)
```
before_cancel → [server cancel] → after_cancel → refresh
```

### On Amend (creates new doc from cancelled)
```
after_cancel → [new doc created] → setup → onload → refresh
```

### On Field Change
```
{fieldname} handler → dependent field handlers (if any)
```

### On frm.refresh() / frm.reload_doc()
```
before_load → onload → refresh → onload_post_render
```

## Complete Form Event Reference

| Event | Fires When | Parameters | Typical Usage |
|-------|-----------|------------|---------------|
| `setup` | Once per form instance creation | `(frm)` | `set_query`, formatters, one-time config |
| `onload` | Form data loaded, before render | `(frm)` | Data pre-processing, defaults |
| `refresh` | After every load, reload, save | `(frm)` | Buttons, visibility, UI updates |
| `onload_post_render` | DOM fully rendered | `(frm)` | DOM-dependent operations |
| `validate` | Before save/submit | `(frm)` | Validation — `frappe.throw()` blocks save |
| `before_save` | After validate, before server call | `(frm)` | Last-minute value changes |
| `after_save` | After successful server save | `(frm)` | Notifications, cleanup |
| `before_submit` | Before document submission | `(frm)` | Pre-submit validation |
| `on_submit` | After successful submission | `(frm)` | Post-submit actions |
| `before_cancel` | Before cancellation | `(frm)` | Pre-cancel checks |
| `after_cancel` | After successful cancellation | `(frm)` | Cleanup, notifications |
| `before_workflow_action` | Before workflow state change | `(frm)` | Workflow interception |
| `after_workflow_action` | After workflow state change | `(frm)` | Post-workflow actions |
| `timeline_refresh` | After timeline section render | `(frm)` | Timeline customization |

## Field Change Events

ALWAYS use the exact fieldname as the event name:

```javascript
frappe.ui.form.on('Sales Invoice', {
    // Fires when 'customer' field value changes
    customer(frm) {
        if (frm.doc.customer) {
            // Fetch related data
        }
    },

    // Fires when 'posting_date' field value changes
    posting_date(frm) {
        // Recalculate due date
    }
});
```

**CRITICAL**: Field change events fire when:
- User changes the value in the UI
- `frm.set_value()` is called programmatically
- `frappe.model.set_value()` is called on a child row field

They do NOT fire when:
- `frm.doc.field = value` is assigned directly (which is why you NEVER do this)

## Child Table Events

ALWAYS register child table events on the CHILD DocType name, not the parent:

```javascript
// CORRECT — register on child doctype 'Sales Order Item'
frappe.ui.form.on('Sales Order Item', {
    // Field change in child row
    qty(frm, cdt, cdn) {
        let row = frappe.get_doc(cdt, cdn);
        frappe.model.set_value(cdt, cdn, 'amount', row.qty * row.rate);
    },

    // Row added to 'items' table
    items_add(frm, cdt, cdn) {
        let row = frappe.get_doc(cdt, cdn);
        // Set defaults for new row
        frappe.model.set_value(cdt, cdn, 'warehouse', frm.doc.set_warehouse);
    },

    // Row removed from 'items' table
    items_remove(frm) {
        // Recalculate totals — no cdt/cdn available
        calculate_totals(frm);
    },

    // Row moved (drag & drop reorder)
    items_move(frm) {
        // Update idx or recalculate
    },

    // Before row removal [v14+]
    items_before_remove(frm, cdt, cdn) {
        // Return false to prevent removal
    }
});
```

### Child Table Event Naming Convention

Format: `{tablefieldname}_{action}`

| Event Pattern | Description | Parameters |
|---------------|-------------|------------|
| `{table}_add` | Row added | `(frm, cdt, cdn)` |
| `{table}_remove` | Row removed | `(frm)` |
| `{table}_move` | Row reordered | `(frm)` |
| `{table}_before_remove` | Before row removal | `(frm, cdt, cdn)` |

### Child Table Event Parameters

```javascript
frappe.ui.form.on('Sales Order Item', {
    qty(frm, cdt, cdn) {
        // frm  — parent form object (Sales Order)
        // cdt  — child doctype name ('Sales Order Item')
        // cdn  — child row name/ID (e.g., 'abc123def4')

        // Get the row data object
        let row = frappe.get_doc(cdt, cdn);
        // Or equivalently:
        let row2 = locals[cdt][cdn];
    }
});
```

## setup vs refresh — When to Use Which

| Aspect | `setup` | `refresh` |
|--------|---------|-----------|
| **Frequency** | Once per form instance | On every load, reload, save |
| **Timing** | Before data is loaded | After data is loaded and rendered |
| **Use for** | `set_query`, formatters, one-time config | Buttons, field visibility, dynamic UI |
| **frm.doc available?** | Partially (may be empty on new) | Yes, fully populated |

```javascript
frappe.ui.form.on('Sales Order', {
    setup(frm) {
        // GOOD: set_query runs once, filter callback reads frm.doc dynamically
        frm.set_query('customer', () => ({
            filters: { disabled: 0 }
        }));
    },

    refresh(frm) {
        // GOOD: buttons depend on document state
        if (!frm.is_new() && frm.doc.docstatus === 0) {
            frm.add_custom_button(__('Validate'), () => { /* ... */ });
        }
    }
});
```

## Async Events

Events support async/await. This is CRITICAL for `validate` when you need server-side checks:

```javascript
frappe.ui.form.on('Sales Order', {
    // CORRECT: async validate with await
    async validate(frm) {
        let r = await frappe.call({
            method: 'myapp.api.check_credit',
            args: { customer: frm.doc.customer }
        });
        if (!r.message.approved) {
            frappe.throw(__('Credit limit exceeded'));
        }
        // If no throw, save proceeds
    },

    // CORRECT: async refresh for data fetching
    async refresh(frm) {
        if (!frm.is_new()) {
            let data = await frappe.call({
                method: 'myapp.api.get_dashboard_data',
                args: { name: frm.doc.name }
            });
            // Update UI with data
        }
    }
});
```

**NEVER** use a non-awaited `frappe.call` inside `validate` — the save will proceed before the callback fires.

## Triggering Events Programmatically

```javascript
// Trigger the refresh event manually
frm.trigger('refresh');

// Trigger a field change event
frm.trigger('customer');
```

This is useful when a field change handler should re-run after you set a value that indirectly affects display logic.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-clientscripts/references/events.md*
