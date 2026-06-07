---
title: Frappe Ref — Frappe Errors: Clientscripts: Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'clientscripts', 'reference', 'patterns']
source: Impertio
---

# Client Script Error Handling Patterns

Reusable patterns for defensive error handling in Frappe Client Scripts.

---

## Pattern 1: Safe Server Call Wrapper

```javascript
/**
 * Wrapper for frappe.call with consistent error handling.
 * ALWAYS use this instead of raw frappe.call for user-triggered actions.
 */
async function safeCall(options) {
    try {
        const r = await frappe.call(options);
        return r.message;
    } catch (error) {
        console.error(`API Error [${options.method}]:`, error);

        if (!navigator.onLine) {
            frappe.msgprint({
                title: __('No Connection'),
                message: __('Check your internet connection and try again.'),
                indicator: 'red'
            });
        } else if (error.status === 401 || error.status === 403) {
            frappe.msgprint({
                title: __('Access Denied'),
                message: __('Session may have expired. Please refresh the page.'),
                indicator: 'red'
            });
        } else if (error.status >= 500) {
            frappe.msgprint({
                title: __('Server Error'),
                message: __('Server error occurred. Please try again later.'),
                indicator: 'red'
            });
        } else if (error._server_messages) {
            // Parse server-side frappe.throw() message
            try {
                let msgs = JSON.parse(error._server_messages);
                let msg = JSON.parse(msgs[0]).message;
                frappe.msgprint({ message: msg, indicator: 'red' });
            } catch (e) {
                frappe.msgprint(__('An error occurred'));
            }
        }

        return null;  // Caller checks for null
    }
}

// Usage
frappe.ui.form.on('Sales Order', {
    async customer(frm) {
        if (!frm.doc.customer) return;
        let data = await safeCall({
            method: 'myapp.api.get_customer_details',
            args: { customer: frm.doc.customer }
        });
        if (data) {
            frm.set_value('credit_limit', data.credit_limit || 0);
        }
    }
});
```

---

## Pattern 2: Validation Error Collector

```javascript
frappe.ui.form.on('Sales Order', {
    validate(frm) {
        let errors = [];

        // Required fields
        if (!frm.doc.customer) errors.push(__('Customer is required'));
        if (!frm.doc.delivery_date) errors.push(__('Delivery Date is required'));

        // Date validation
        if (frm.doc.delivery_date && frm.doc.delivery_date < frappe.datetime.get_today()) {
            errors.push(__('Delivery Date cannot be in the past'));
        }

        // Child table validation
        if (!frm.doc.items || frm.doc.items.length === 0) {
            errors.push(__('At least one item is required'));
        } else {
            frm.doc.items.forEach((row, idx) => {
                if (!row.item_code) errors.push(__('Row {0}: Item is required', [idx + 1]));
                if ((row.qty || 0) <= 0) errors.push(__('Row {0}: Qty must be positive', [idx + 1]));
            });
        }

        if (errors.length) {
            frappe.throw({
                title: __('Please fix the following'),
                message: errors.join('<br>')
            });
        }
    }
});
```

---

## Pattern 3: Async Button with Error Boundary

```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__('Process'), async () => {
                await withErrorBoundary(frm, async () => {
                    await frappe.call({
                        method: 'myapp.api.process_order',
                        args: { name: frm.doc.name },
                        freeze: true,
                        freeze_message: __('Processing...')
                    });
                    frappe.show_alert({ message: __('Done'), indicator: 'green' }, 3);
                    frm.reload_doc();
                });
            });
        }
    }
});

async function withErrorBoundary(frm, asyncFn) {
    try {
        frm.disable_save();
        await asyncFn();
    } catch (error) {
        console.error('Action failed:', error);
        frappe.msgprint({
            title: __('Error'),
            message: __('Operation failed. Please try again.'),
            indicator: 'red'
        });
    } finally {
        frm.enable_save();
    }
}
```

---

## Pattern 4: Graceful Degradation — Optional Data

```javascript
frappe.ui.form.on('Sales Order', {
    async refresh(frm) {
        // Try to load dashboard data, but don't fail if unavailable
        try {
            let stock = await frappe.call({
                method: 'myapp.api.get_stock_summary',
                args: { items: (frm.doc.items || []).map(r => r.item_code).filter(Boolean) }
            });
            if (stock.message) {
                renderStockDashboard(frm, stock.message);
            }
        } catch (error) {
            console.warn('Stock dashboard unavailable:', error);
            frm.dashboard.set_headline(__('Stock info unavailable'), 'orange');
        }
    }
});
```

---

## Pattern 5: Recursion Guard with Flags

```javascript
frappe.ui.form.on('Sales Order', {
    discount_percent(frm) {
        if (frm.flags.recalculating) return;
        frm.flags.recalculating = true;
        try {
            let total = calculateTotal(frm);
            frm.set_value('grand_total', total);
        } finally {
            frm.flags.recalculating = false;
        }
    },

    grand_total(frm) {
        if (frm.flags.recalculating) return;
        // Respond to manual grand_total changes
    }
});
```

---

## Pattern 6: Retry with Exponential Backoff

```javascript
async function fetchWithRetry(method, args, maxRetries = 3) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            let r = await frappe.call({ method, args });
            return r.message;
        } catch (error) {
            if (attempt === maxRetries) throw error;
            console.warn(`Attempt ${attempt} failed, retrying...`);
            await new Promise(resolve =>
                setTimeout(resolve, Math.pow(2, attempt) * 1000)
            );
        }
    }
}
```

---

## Pattern 7: Confirmation Before Destructive Action

```javascript
async function confirmAndExecute(frm, message, action) {
    return new Promise((resolve) => {
        frappe.confirm(
            message,
            async () => {
                try {
                    await action(frm);
                    resolve(true);
                } catch (error) {
                    console.error('Action failed:', error);
                    frappe.msgprint({ message: __('Action failed'), indicator: 'red' });
                    resolve(false);
                }
            },
            () => resolve(false)
        );
    });
}

// Usage
frm.add_custom_button(__('Cancel Order'), async () => {
    await confirmAndExecute(
        frm,
        __('Are you sure you want to cancel this order?'),
        async (frm) => {
            await frappe.call({ method: 'frappe.client.cancel', args: { doctype: frm.doctype, name: frm.doc.name } });
            frm.reload_doc();
        }
    );
});
```

---

## Pattern 8: Conditional Debug Logger

```javascript
const DEBUG = frappe.boot.developer_mode;
const log = {
    info: (...args) => { if (DEBUG) console.log('[MyApp]', ...args); },
    warn: (...args) => { if (DEBUG) console.warn('[MyApp]', ...args); },
    error: (...args) => { console.error('[MyApp]', ...args); }  // Always log errors
};

// Usage — no console.log in production
frappe.ui.form.on('Sales Order', {
    validate(frm) {
        log.info('Validating:', frm.doc.name);
    }
});
```

---

## Pattern 9: Batch Processing with Progress

```javascript
async function processBatch(frm, items, processFn) {
    let results = { success: 0, failed: 0, errors: [] };

    frappe.show_progress(__('Processing'), 0, items.length);

    for (let i = 0; i < items.length; i++) {
        try {
            await processFn(items[i]);
            results.success++;
        } catch (error) {
            results.failed++;
            results.errors.push({ row: i + 1, error: error.message || 'Unknown' });
        }
        frappe.show_progress(__('Processing'), i + 1, items.length);
    }

    frappe.hide_progress();

    if (results.failed === 0) {
        frappe.msgprint({ message: __('All {0} items processed', [results.success]), indicator: 'green' });
    } else {
        let errorList = results.errors.map(e => __('Row {0}: {1}', [e.row, e.error])).join('<br>');
        frappe.msgprint({
            title: __('Completed with Errors'),
            message: __('Success: {0}, Failed: {1}', [results.success, results.failed]) + '<br><br>' + errorList,
            indicator: 'orange'
        });
    }

    return results;
}
```

---

## Pattern 10: Loading State for Async Field Change

```javascript
frappe.ui.form.on('Sales Order', {
    async item_code(frm) {
        if (!frm.doc.item_code) return;

        frm.set_df_property('rate', 'read_only', 1);
        frm.set_df_property('rate', 'description', __('Loading price...'));

        try {
            let r = await frappe.call({
                method: 'myapp.api.get_price',
                args: { item_code: frm.doc.item_code, customer: frm.doc.customer }
            });
            if (r.message) {
                await frm.set_value('rate', r.message.rate);
                frm.set_df_property('rate', 'description', '');
            } else {
                frm.set_df_property('rate', 'description', __('No price found — enter manually'));
            }
        } catch (error) {
            console.error('Price lookup failed:', error);
            frm.set_df_property('rate', 'description', __('Price lookup failed'));
        } finally {
            frm.set_df_property('rate', 'read_only', 0);
        }
    }
});
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-clientscripts/references/patterns.md*
