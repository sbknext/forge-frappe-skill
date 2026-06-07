---
title: Frappe Ref — Frappe Errors: Clientscripts: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'clientscripts', 'reference', 'examples']
source: Impertio
---

# Client Script Error Examples — Real Scenarios

Complete diagnosis-oriented examples showing actual error messages, their root cause, and the fix.

---

## Scenario 1: TypeError — Cannot Read Properties of Undefined

**Error in console:**
```
Uncaught TypeError: Cannot read properties of undefined (reading 'credit_limit')
```

**The broken code:**
```javascript
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        let r = await frappe.call({
            method: 'myapp.api.get_customer',
            args: { customer: frm.doc.customer }
        });
        // r.message is null when customer not found!
        frm.set_value('credit_limit', r.message.credit_limit);
    }
});
```

**Root cause:** Server returned `null` for `r.message` — customer not found or API error.

**The fix:**
```javascript
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        if (!frm.doc.customer) {
            frm.set_value('credit_limit', 0);
            return;
        }
        try {
            let r = await frappe.call({
                method: 'myapp.api.get_customer',
                args: { customer: frm.doc.customer }
            });
            frm.set_value('credit_limit', r.message?.credit_limit || 0);
        } catch (error) {
            console.error('Customer fetch failed:', error);
            frm.set_value('credit_limit', 0);
            frappe.show_alert({
                message: __('Could not load customer details'),
                indicator: 'orange'
            }, 5);
        }
    }
});
```

---

## Scenario 2: cur_frm Is Undefined in List View Context

**Error in console:**
```
Uncaught TypeError: Cannot read properties of undefined (reading 'set_value')
    at cur_frm.set_value(...)
```

**The broken code:**
```javascript
// This JS was loaded globally (app.js) and runs in list context too
frappe.ui.form.on('Sales Order', {
    customer(frm) {
        cur_frm.set_value('territory', get_default_territory());
    }
});
```

**Root cause:** `cur_frm` is only set when a form is open. In list view, page view, or report context, it is `undefined`.

**The fix:**
```javascript
frappe.ui.form.on('Sales Order', {
    customer(frm) {
        frm.set_value('territory', get_default_territory());
    }
});
```

**Rule:** ALWAYS use the `frm` parameter. NEVER use `cur_frm`.

---

## Scenario 3: frappe.throw() Not Preventing Save

**User reports:** "I added validation but user can still save the form with invalid data."

**The broken code:**
```javascript
frappe.ui.form.on('Sales Order', {
    delivery_date(frm) {
        if (frm.doc.delivery_date < frappe.datetime.get_today()) {
            frappe.throw(__('Delivery date cannot be in the past'));
        }
    }
});
```

**Root cause:** `frappe.throw()` in a field change event stops JavaScript execution but does NOT prevent the user from clicking Save. Only `validate` event blocks save.

**The fix:**
```javascript
frappe.ui.form.on('Sales Order', {
    delivery_date(frm) {
        if (frm.doc.delivery_date < frappe.datetime.get_today()) {
            frappe.msgprint({
                message: __('Delivery date is in the past'),
                indicator: 'orange'
            });
        }
    },
    validate(frm) {
        if (frm.doc.delivery_date < frappe.datetime.get_today()) {
            frappe.throw(__('Delivery date cannot be in the past'));
        }
    }
});
```

---

## Scenario 4: Unhandled Promise Rejection

**Error in console:**
```
Uncaught (in promise) Object { exc_type: "ValidationError", ... }
```

**The broken code:**
```javascript
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        // No try/catch — if server throws, this crashes silently
        let r = await frappe.call({
            method: 'myapp.api.validate_customer',
            args: { customer: frm.doc.customer }
        });
        frm.set_value('validated', 1);
    }
});
```

**Root cause:** `frappe.call` with `await` throws when server returns error (e.g., `frappe.throw()` server-side). Without `try/catch`, the rejection is unhandled.

**The fix:**
```javascript
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        if (!frm.doc.customer) return;
        try {
            let r = await frappe.call({
                method: 'myapp.api.validate_customer',
                args: { customer: frm.doc.customer }
            });
            frm.set_value('validated', 1);
        } catch (error) {
            console.error('Validation failed:', error);
            frm.set_value('validated', 0);
            // Parse server error message
            if (error._server_messages) {
                try {
                    let msgs = JSON.parse(error._server_messages);
                    let msg = JSON.parse(msgs[0]).message;
                    frappe.msgprint({ message: msg, indicator: 'red' });
                } catch (e) {
                    frappe.msgprint(__('Validation failed'));
                }
            }
        }
    }
});
```

---

## Scenario 5: Child Table Row Access — Wrong Pattern

**Error in console:**
```
TypeError: Cannot read properties of undefined (reading 'item_code')
```

**The broken code:**
```javascript
frappe.ui.form.on('Sales Order Item', {
    qty(frm, cdt, cdn) {
        // Trying to find row by index — fragile and wrong
        let idx = frm.doc.items.findIndex(r => r.name === cdn);
        let row = frm.doc.items[idx];
        row.amount = row.qty * row.rate;  // Direct mutation — no UI refresh
    }
});
```

**Root cause:** Direct array access can fail when rows are reordered or deleted. Direct property mutation does not trigger UI refresh.

**The fix:**
```javascript
frappe.ui.form.on('Sales Order Item', {
    qty(frm, cdt, cdn) {
        let row = frappe.get_doc(cdt, cdn);
        let amount = (row.qty || 0) * (row.rate || 0);
        frappe.model.set_value(cdt, cdn, 'amount', amount);
    }
});
```

---

## Scenario 6: CSRF Token Error After Session Timeout

**Error:** 403 Forbidden with CSRF token mismatch.

**The broken code:**
```javascript
// Manual XHR call without frappe.call
$.ajax({
    url: '/api/method/myapp.api.process',
    type: 'POST',
    data: { name: frm.doc.name },
    success: function(r) { frm.reload_doc(); }
});
```

**Root cause:** Manual AJAX calls do not include the CSRF token header that Frappe requires. After session timeout, even some frappe.call requests may fail.

**The fix:**
```javascript
// ALWAYS use frappe.call — handles CSRF automatically
frappe.call({
    method: 'myapp.api.process',
    args: { name: frm.doc.name },
    callback(r) {
        if (r.message) frm.reload_doc();
    },
    error(r) {
        if (r.status === 401 || r.status === 403) {
            frappe.msgprint(__('Session expired. Please refresh the page.'));
        }
    }
});
```

---

## Scenario 7: set_value in setup — Form Not Ready

**Symptom:** `frm.set_value('company', 'Default Company')` in `setup` silently does nothing.

**The broken code:**
```javascript
frappe.ui.form.on('Sales Order', {
    setup(frm) {
        frm.set_value('company', 'Default Company');  // Ignored — form not loaded
    }
});
```

**Root cause:** `setup` fires before the form document is loaded from the server. `set_value` has no effect because the document fields don't exist yet.

**The fix:**
```javascript
frappe.ui.form.on('Sales Order', {
    setup(frm) {
        // Only filters and formatters in setup
        frm.set_query('customer', () => ({ filters: { disabled: 0 } }));
    },
    onload(frm) {
        // Set defaults for new documents
        if (frm.is_new()) {
            frm.set_value('company', 'Default Company');
        }
    }
});
```

---

## Scenario 8: Mixing .then() and await

**Symptom:** Code runs in wrong order — `doSomethingElse()` executes before `frappe.call` completes.

**The broken code:**
```javascript
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        frappe.call({
            method: 'myapp.api.get_data',
            args: { customer: frm.doc.customer }
        }).then(r => {
            frm.set_value('credit_limit', r.message.limit);
        });
        // This runs BEFORE .then() callback!
        await frm.save();
    }
});
```

**The fix:**
```javascript
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        try {
            let r = await frappe.call({
                method: 'myapp.api.get_data',
                args: { customer: frm.doc.customer }
            });
            if (r.message) {
                await frm.set_value('credit_limit', r.message.limit);
            }
            await frm.save();
        } catch (error) {
            console.error('Error:', error);
        }
    }
});
```

**Rule:** NEVER mix `.then()` and `await` in the same function. Pick one pattern.

---

## HTTP Status Code Quick Reference

| Status | Meaning in Frappe | Common Cause |
|--------|-------------------|--------------|
| 401 | Session expired | User not logged in or token expired |
| 403 | Permission denied | Method not whitelisted or role missing |
| 404 | Not found | Method path wrong or document deleted |
| 417 | Expectation Failed | Server-side `frappe.throw()` |
| 429 | Rate limited | Too many requests |
| 500 | Server error | Unhandled Python exception |
| 502/503 | Gateway error | Server overloaded or restarting |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-clientscripts/references/examples.md*
