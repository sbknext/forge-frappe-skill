---
title: Frappe Ref — Frappe Syntax: Clientscripts: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'clientscripts', 'reference', 'anti-patterns']
source: Impertio
---

# Client Script Anti-Patterns

## AP-01: Direct Field Value Assignment

**WRONG:**
```javascript
frm.doc.customer_name = 'Test';  // NEVER do this
```

**CORRECT:**
```javascript
frm.set_value('customer_name', 'Test');
```

**Why:** `frm.set_value()` triggers the dirty flag, field change events, dependent field updates, and UI refresh. Direct assignment skips all of these, leading to stale UI and lost data.

---

## AP-02: Child Table Modification Without refresh_field

**WRONG:**
```javascript
let row = frm.add_child('items', { item_code: 'TEST' });
// UI does NOT show the new row
```

**CORRECT:**
```javascript
let row = frm.add_child('items', { item_code: 'TEST' });
frm.refresh_field('items');  // ALWAYS required
```

**Why:** Child table UI is not automatically synchronized. ALWAYS call `frm.refresh_field()` after `add_child`, `clear_table`, or direct row modifications.

---

## AP-03: set_query in refresh Instead of setup

**WRONG:**
```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        frm.set_query('customer', () => ({  // Runs on EVERY refresh
            filters: { disabled: 0 }
        }));
    }
});
```

**CORRECT:**
```javascript
frappe.ui.form.on('Sales Order', {
    setup(frm) {
        frm.set_query('customer', () => ({  // Runs ONCE
            filters: { disabled: 0 }
        }));
    }
});
```

**Why:** `setup` runs once per form instance. `refresh` fires on every load, reload, and save. Queries set in `refresh` are redundantly re-registered. The callback function inside `set_query` already reads `frm.doc` dynamically at execution time.

---

## AP-04: Synchronous Server Calls

**WRONG:**
```javascript
frappe.call({
    method: 'myapp.api.get_data',
    async: false,  // NEVER use async: false
    callback: (r) => { /* ... */ }
});
```

**CORRECT:**
```javascript
let r = await frappe.call({
    method: 'myapp.api.get_data'
});
// Or use callback pattern (still async)
```

**Why:** `async: false` blocks the entire browser thread. The UI freezes, the user cannot interact, and browser may show "page unresponsive" warnings. ALWAYS use async patterns.

---

## AP-05: Hardcoded Strings Without Translation

**WRONG:**
```javascript
frappe.msgprint('Operation completed');
frm.add_custom_button('Generate Report', () => {});
```

**CORRECT:**
```javascript
frappe.msgprint(__('Operation completed'));
frm.add_custom_button(__('Generate Report'), () => {});
```

**Why:** Without `__()` wrapper, strings are not translatable. ALWAYS wrap every user-facing string in `__()`.

---

## AP-06: Callback Hell in Server Calls

**WRONG:**
```javascript
frappe.call({
    method: 'method1',
    callback: (r1) => {
        frappe.call({
            method: 'method2',
            callback: (r2) => {
                frappe.call({
                    method: 'method3',
                    callback: (r3) => { /* deeply nested */ }
                });
            }
        });
    }
});
```

**CORRECT:**
```javascript
async function processData() {
    let r1 = await frappe.call({ method: 'method1' });
    let r2 = await frappe.call({ method: 'method2' });
    let r3 = await frappe.call({ method: 'method3' });
}

// Or for independent calls — use Promise.all
let [r1, r2, r3] = await Promise.all([
    frappe.call({ method: 'method1' }),
    frappe.call({ method: 'method2' }),
    frappe.call({ method: 'method3' })
]);
```

---

## AP-07: No Error Handling on Server Calls

**WRONG:**
```javascript
frappe.call({
    method: 'myapp.api.risky_operation',
    callback: (r) => {
        frm.set_value('result', r.message);  // Crashes if r.message is undefined
    }
});
```

**CORRECT:**
```javascript
frappe.call({
    method: 'myapp.api.risky_operation',
    callback: (r) => {
        if (r.message) {
            frm.set_value('result', r.message);
        }
    },
    error: (r) => {
        frappe.msgprint(__('Operation failed'));
    }
});
```

**Why:** Server calls can fail due to permissions, validation errors, or network issues. ALWAYS check `r.message` before use and provide an error handler.

---

## AP-08: Non-Awaited frappe.call in validate Event

**WRONG:**
```javascript
frappe.ui.form.on('Sales Order', {
    validate(frm) {
        frappe.call({
            method: 'myapp.api.check_credit',
            callback: (r) => {
                if (!r.message.ok) {
                    frappe.throw(__('Credit exceeded'));  // TOO LATE — save already proceeded
                }
            }
        });
    }
});
```

**CORRECT:**
```javascript
frappe.ui.form.on('Sales Order', {
    async validate(frm) {
        let r = await frappe.call({
            method: 'myapp.api.check_credit',
            args: { customer: frm.doc.customer }
        });
        if (!r.message.ok) {
            frappe.throw(__('Credit exceeded'));  // Blocks save correctly
        }
    }
});
```

**Why:** Without `await`, the `validate` function returns immediately and the save proceeds. The callback fires after the save is already in progress. ALWAYS use `async/await` when validate needs server data.

---

## AP-09: refresh_field Inside Loops

**WRONG:**
```javascript
frm.doc.items.forEach(item => {
    item.amount = item.qty * item.rate;
    frm.refresh_field('items');  // Called N times — performance disaster
});
```

**CORRECT:**
```javascript
frm.doc.items.forEach(item => {
    item.amount = item.qty * item.rate;
});
frm.refresh_field('items');  // Called ONCE after all modifications
```

---

## AP-10: Global Variables for Form State

**WRONG:**
```javascript
var current_customer = null;  // Global scope

frappe.ui.form.on('Sales Order', {
    customer(frm) {
        current_customer = frm.doc.customer;
    }
});
```

**CORRECT:**
```javascript
frappe.ui.form.on('Sales Order', {
    customer(frm) {
        frm._customer_cache = {};  // Attach to frm object
    }
});
```

**Why:** Global variables are shared across all open form tabs. If two Sales Orders are open, they overwrite each other's state. ALWAYS attach state to the `frm` object.

---

## AP-11: Direct DOM/jQuery Manipulation

**WRONG:**
```javascript
$('[data-fieldname="customer"]').hide();
$('.form-group[data-fieldname="amount"] input').prop('disabled', true);
```

**CORRECT:**
```javascript
frm.toggle_display('customer', false);
frm.toggle_enable('amount', false);
```

**Why:** Direct DOM manipulation bypasses Frappe's rendering cycle. Frappe may overwrite your changes on the next refresh. ALWAYS use Frappe API methods.

---

## AP-12: Blocking Loops for Server Calls

**WRONG:**
```javascript
frm.doc.items.forEach(item => {
    frappe.call({
        method: 'myapp.api.process_item',
        args: { item: item.name },
        async: false  // Blocks for EVERY item
    });
});
```

**CORRECT:**
```javascript
// Option 1: Batch call (preferred — single server round trip)
await frappe.call({
    method: 'myapp.api.process_items',
    args: { items: frm.doc.items.map(i => i.name) }
});

// Option 2: Parallel execution
await Promise.all(frm.doc.items.map(item =>
    frappe.call({
        method: 'myapp.api.process_item',
        args: { item: item.name }
    })
));
```

**Why:** ALWAYS prefer a single batch call. If batch is not possible, use `Promise.all` for parallel execution. NEVER use `async: false` in a loop.

---

## AP-13: Buttons Without State Guards

**WRONG:**
```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        frm.add_custom_button(__('Process'), () => {
            frm.call('process');  // Fails on new/cancelled documents
        });
    }
});
```

**CORRECT:**
```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        if (!frm.is_new() && frm.doc.docstatus === 0) {
            frm.add_custom_button(__('Process'), () => {
                frm.call('process');
            });
        }
    }
});
```

**Why:** Buttons appear on every document state (new, draft, submitted, cancelled). ALWAYS check `frm.is_new()`, `frm.doc.docstatus`, and relevant field values before adding action buttons.

---

## AP-14: frappe.model.set_value Outside Child Context

**WRONG:**
```javascript
// In parent form event — cdt/cdn are not available
frappe.model.set_value(cdt, cdn, 'qty', 10);
```

**CORRECT:**
```javascript
// In child table event — cdt/cdn are parameters
frappe.ui.form.on('Sales Order Item', {
    item_code(frm, cdt, cdn) {
        frappe.model.set_value(cdt, cdn, 'qty', 10);
    }
});

// From parent context — use direct access + refresh
frm.doc.items[0].qty = 10;
frm.refresh_field('items');
```

---

## Pre-Deployment Checklist

ALWAYS verify before deploying a client script:

- [ ] All user-facing strings wrapped in `__()`
- [ ] No `async: false` anywhere
- [ ] `refresh_field()` called after every child table modification
- [ ] `refresh_field()` called ONCE after loops, not inside loops
- [ ] Error handling on all server calls (`if (r.message)` + error callback)
- [ ] `frm.is_new()` / `docstatus` checks before action buttons
- [ ] `set_query` placed in `setup`, not `refresh`
- [ ] No global state variables — state attached to `frm` object
- [ ] No direct DOM/jQuery manipulation — Frappe API used
- [ ] `validate` event uses `async/await` for any server calls
- [ ] No `frm.doc.field = value` — only `frm.set_value()`

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-clientscripts/references/anti-patterns.md*
