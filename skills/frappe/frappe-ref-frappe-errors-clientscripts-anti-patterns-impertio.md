---
title: Frappe Ref — Frappe Errors: Clientscripts: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'clientscripts', 'reference', 'anti-patterns']
source: Impertio
---

# Client Script Anti-Patterns — Error Prevention

Each anti-pattern shows the mistake, why it fails, and the correct approach.

---

## 1. Using cur_frm Instead of frm Parameter

```javascript
// ❌ WRONG
frappe.ui.form.on('Sales Order', {
    customer(frm) { cur_frm.set_value('territory', 'Default'); }
});

// ✅ CORRECT
frappe.ui.form.on('Sales Order', {
    customer(frm) { frm.set_value('territory', 'Default'); }
});
```
**Why:** `cur_frm` is undefined in list view, print, or when multiple tabs are open.

---

## 2. frappe.throw() Outside validate Event

```javascript
// ❌ WRONG — Does NOT prevent save
frappe.ui.form.on('Sales Order', {
    customer(frm) {
        if (!frm.doc.customer) frappe.throw(__('Customer required'));
    }
});

// ✅ CORRECT — Blocks save only in validate
frappe.ui.form.on('Sales Order', {
    customer(frm) {
        if (!frm.doc.customer) frappe.msgprint({ message: __('Select a customer'), indicator: 'orange' });
    },
    validate(frm) {
        if (!frm.doc.customer) frappe.throw(__('Customer is required'));
    }
});
```
**Why:** `frappe.throw()` only prevents save when called within `validate`.

---

## 3. No try/catch on Async Calls

```javascript
// ❌ WRONG — Silent failure
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        let r = await frappe.call({ method: 'myapp.api.get_data' });
        frm.set_value('field', r.message.value);
    }
});

// ✅ CORRECT
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        try {
            let r = await frappe.call({ method: 'myapp.api.get_data' });
            if (r.message) frm.set_value('field', r.message.value);
        } catch (error) {
            console.error('API failed:', error);
            frappe.show_alert({ message: __('Could not load data'), indicator: 'red' }, 5);
        }
    }
});
```
**Why:** Unhandled Promise rejections fail silently, confusing users.

---

## 4. Throwing on First Error (One at a Time)

```javascript
// ❌ WRONG — User saves 5 times to find 5 errors
frappe.ui.form.on('Sales Order', {
    validate(frm) {
        if (!frm.doc.customer) frappe.throw(__('Customer required'));
        if (!frm.doc.delivery_date) frappe.throw(__('Date required'));
        if (!frm.doc.items?.length) frappe.throw(__('Items required'));
    }
});

// ✅ CORRECT — All errors at once
frappe.ui.form.on('Sales Order', {
    validate(frm) {
        let errors = [];
        if (!frm.doc.customer) errors.push(__('Customer is required'));
        if (!frm.doc.delivery_date) errors.push(__('Delivery Date is required'));
        if (!frm.doc.items?.length) errors.push(__('At least one item is required'));
        if (errors.length) frappe.throw({ title: __('Please fix'), message: errors.join('<br>') });
    }
});
```
**Why:** Users should see all errors at once, not discover them one at a time.

---

## 5. Using Native alert() / confirm() / prompt()

```javascript
// ❌ WRONG — Blocks thread, looks unprofessional
if (frm.doc.grand_total > 100000) {
    if (!confirm('Large order. Continue?')) return false;
}

// ✅ CORRECT — Use frappe.confirm
frappe.confirm(
    __('Large order ({0}). Continue?', [format_currency(frm.doc.grand_total)]),
    () => { /* proceed */ },
    () => { /* cancel */ }
);
```
**Why:** Native dialogs block the thread, cannot be styled, and look outdated.

---

## 6. Exposing Technical Errors to Users

```javascript
// ❌ WRONG
catch (error) { frappe.throw(error.stack); }

// ✅ CORRECT
catch (error) {
    console.error('Technical error:', error);
    frappe.msgprint({ title: __('Error'), message: __('An error occurred. Please try again.'), indicator: 'red' });
}
```
**Why:** Stack traces confuse users and may expose sensitive internals.

---

## 7. No Null Check on Server Response

```javascript
// ❌ WRONG — Crashes when r.message is null
callback(r) { frm.set_value('limit', r.message.credit_limit); }

// ✅ CORRECT
callback(r) { frm.set_value('limit', r.message?.credit_limit || 0); }
```
**Why:** Server may return null on error, missing record, or permission failure.

---

## 8. Ignoring error Callback on frappe.call

```javascript
// ❌ WRONG — No error handler
frappe.call({
    method: 'myapp.api.process',
    callback(r) { frappe.show_alert({ message: __('Done'), indicator: 'green' }); }
});

// ✅ CORRECT
frappe.call({
    method: 'myapp.api.process',
    callback(r) { if (r.message) frappe.show_alert({ message: __('Done'), indicator: 'green' }); },
    error(r) {
        console.error('Process failed:', r);
        frappe.msgprint({ title: __('Error'), message: __('Failed. Please try again.'), indicator: 'red' });
    }
});
```
**Why:** Without error handler, failures are invisible to the user.

---

## 9. console.log in Production Code

```javascript
// ❌ WRONG — Clutters production console
validate(frm) { console.log('customer:', frm.doc.customer); }

// ✅ CORRECT — Conditional debugging
const DEBUG = frappe.boot.developer_mode;
function debug(...args) { if (DEBUG) console.log('[MyApp]', ...args); }
validate(frm) { debug('customer:', frm.doc.customer); }
```
**Why:** Production console logs may expose data and slow performance.

---

## 10. Mixing .then() and await

```javascript
// ❌ WRONG — Unpredictable execution order
async customer(frm) {
    frappe.call({ method: 'api' }).then(r => { frm.set_value('f', r.message); });
    await doSomethingElse();  // Runs BEFORE .then()
}

// ✅ CORRECT — Consistent await pattern
async customer(frm) {
    try {
        let r = await frappe.call({ method: 'api' });
        frm.set_value('f', r.message);
        await doSomethingElse();
    } catch (error) { console.error(error); }
}
```
**Why:** Mixing patterns creates race conditions and unpredictable behavior.

---

## 11. Not Disabling Controls During Async Operations

```javascript
// ❌ WRONG — User clicks multiple times
frm.add_custom_button(__('Process'), async () => {
    await frappe.call({ method: 'myapp.api.process' });
    frm.reload_doc();
});

// ✅ CORRECT — Disable during operation
frm.add_custom_button(__('Process'), async () => {
    try {
        frm.disable_save();
        await frappe.call({ method: 'myapp.api.process', freeze: true, freeze_message: __('Processing...') });
        frm.reload_doc();
    } catch (error) {
        frappe.msgprint(__('Processing failed'));
    } finally {
        frm.enable_save();
    }
});
```
**Why:** Duplicate clicks cause duplicate server operations.

---

## 12. Missing Translation Wrapper

```javascript
// ❌ WRONG — Not translatable
frappe.throw('Customer is required');

// ✅ CORRECT
frappe.throw(__('Customer is required'));
```
**Why:** Without `__()`, messages appear only in English regardless of user language.

---

## Pre-Deploy Checklist

- [ ] All `frappe.throw()` calls are ONLY in `validate` event
- [ ] All async operations have try/catch
- [ ] All validation errors collected before throwing
- [ ] No `cur_frm` usage anywhere
- [ ] All user-facing strings use `__()`
- [ ] All server responses checked for null
- [ ] All frappe.call have error handler
- [ ] No `alert()`, `confirm()`, or `prompt()`
- [ ] No unconditioned `console.log` statements
- [ ] Controls disabled during async operations

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-clientscripts/references/anti-patterns.md*
