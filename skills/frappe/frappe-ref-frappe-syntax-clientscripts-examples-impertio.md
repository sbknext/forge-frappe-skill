---
title: Frappe Ref — Frappe Syntax: Clientscripts: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'clientscripts', 'reference', 'examples']
source: Impertio
---

# Client Script Examples

## Example 1: Complete Form Setup with Filters and Defaults

```javascript
frappe.ui.form.on('Sales Order', {
    setup(frm) {
        // ALWAYS place set_query in setup — runs once
        frm.set_query('customer', () => ({
            filters: { disabled: 0 }
        }));

        frm.set_query('item_code', 'items', () => ({
            filters: { is_sales_item: 1, disabled: 0 }
        }));

        // Server-side query for complex filtering
        frm.set_query('delivery_warehouse', 'items', () => ({
            query: 'myapp.queries.get_warehouses_for_company',
            filters: { company: frm.doc.company }
        }));
    },

    onload(frm) {
        if (frm.is_new()) {
            frm.set_value('delivery_date',
                frappe.datetime.add_days(frappe.datetime.now_date(), 7));
        }
    }
});
```

## Example 2: Conditional Field Visibility and Mandatory

```javascript
frappe.ui.form.on('Sales Invoice', {
    refresh(frm) {
        let has_shipping = frm.doc.shipping_amount > 0;
        frm.toggle_display(['shipping_address', 'shipping_method'], has_shipping);
        frm.toggle_reqd('shipping_address', has_shipping);

        // Status-based read-only
        let is_submitted = frm.doc.docstatus === 1;
        frm.toggle_enable('customer', !is_submitted);
        frm.toggle_enable('posting_date', !is_submitted);
    },

    shipping_amount(frm) {
        // Re-trigger visibility when shipping amount changes
        frm.trigger('refresh');
    }
});
```

## Example 3: Custom Buttons with State Guards

```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        // ALWAYS guard buttons with document state checks
        if (frm.doc.docstatus === 1) {
            // Grouped buttons under "Create" dropdown
            frm.add_custom_button(__('Sales Invoice'), () => {
                frappe.model.open_mapped_doc({
                    method: 'erpnext.selling.doctype.sales_order.sales_order.make_sales_invoice',
                    frm: frm
                });
            }, __('Create'));

            frm.add_custom_button(__('Delivery Note'), () => {
                frappe.model.open_mapped_doc({
                    method: 'erpnext.selling.doctype.sales_order.sales_order.make_delivery_note',
                    frm: frm
                });
            }, __('Create'));
        }

        // Button for draft documents only
        if (!frm.is_new() && frm.doc.docstatus === 0) {
            frm.add_custom_button(__('Validate Stock'), async () => {
                let r = await frappe.call({
                    method: 'myapp.api.check_stock',
                    args: { sales_order: frm.doc.name },
                    freeze: true,
                    freeze_message: __('Checking stock...')
                });
                if (r.message.all_available) {
                    frappe.show_alert({
                        message: __('All items available'),
                        indicator: 'green'
                    });
                } else {
                    frappe.msgprint({
                        title: __('Stock Warning'),
                        message: __('Unavailable: {0}',
                            [r.message.unavailable.join(', ')]),
                        indicator: 'orange'
                    });
                }
            });
        }
    }
});
```

## Example 4: Form Validation with frappe.throw

```javascript
frappe.ui.form.on('Sales Invoice', {
    validate(frm) {
        // Validate total
        if (frm.doc.grand_total <= 0) {
            frappe.throw(__('Grand total must be greater than zero'));
        }

        // Validate items exist
        if (!frm.doc.items || frm.doc.items.length === 0) {
            frappe.throw(__('At least one item is required'));
        }

        // Validate date logic
        if (frm.doc.due_date && frm.doc.due_date < frm.doc.posting_date) {
            frappe.throw(__('Due date cannot be before posting date'));
        }

        // Validate each row
        frm.doc.items.forEach((item, idx) => {
            if (item.qty <= 0) {
                frappe.throw(__('Row {0}: Quantity must be positive', [idx + 1]));
            }
            if (!item.rate) {
                frappe.throw(__('Row {0}: Rate is required', [idx + 1]));
            }
        });
    }
});
```

## Example 5: Async Validation with Server Check

```javascript
frappe.ui.form.on('Sales Order', {
    // CRITICAL: async validate with await — NEVER use callback pattern here
    async validate(frm) {
        if (frm.doc.customer) {
            let r = await frappe.call({
                method: 'myapp.api.check_credit_limit',
                args: {
                    customer: frm.doc.customer,
                    amount: frm.doc.grand_total
                }
            });
            if (r.message && !r.message.approved) {
                frappe.throw(__('Credit limit exceeded. Limit: {0}, Outstanding: {1}',
                    [r.message.limit, r.message.outstanding]));
            }
        }
    }
});
```

## Example 6: Fetching Data on Field Change

```javascript
frappe.ui.form.on('Sales Order', {
    customer(frm) {
        if (!frm.doc.customer) return;

        frappe.db.get_value('Customer', frm.doc.customer,
            ['customer_group', 'territory', 'default_currency', 'credit_limit'])
            .then(r => {
                if (r.message) {
                    frm.set_value({
                        customer_group: r.message.customer_group,
                        territory: r.message.territory,
                        currency: r.message.default_currency
                    });

                    if (r.message.credit_limit) {
                        frappe.show_alert({
                            message: __('Credit limit: {0}',
                                [format_currency(r.message.credit_limit)]),
                            indicator: 'blue'
                        });
                    }
                }
            });
    }
});
```

## Example 7: Child Table Auto-Calculation

```javascript
frappe.ui.form.on('Sales Invoice Item', {
    qty(frm, cdt, cdn) {
        calculate_row_amount(frm, cdt, cdn);
    },
    rate(frm, cdt, cdn) {
        calculate_row_amount(frm, cdt, cdn);
    },
    discount_percentage(frm, cdt, cdn) {
        calculate_row_amount(frm, cdt, cdn);
    },
    items_remove(frm) {
        calculate_totals(frm);
    }
});

function calculate_row_amount(frm, cdt, cdn) {
    let row = frappe.get_doc(cdt, cdn);
    let amount = row.qty * row.rate;

    if (row.discount_percentage) {
        amount = amount * (1 - row.discount_percentage / 100);
    }

    frappe.model.set_value(cdt, cdn, 'amount', amount);
    calculate_totals(frm);
}

function calculate_totals(frm) {
    let total = 0;
    (frm.doc.items || []).forEach(item => {
        total += item.amount || 0;
    });
    frm.set_value('net_total', total);
    frm.set_value('grand_total', total * 1.21);  // Example: 21% VAT
}
```

## Example 8: Populating Child Table from Server

```javascript
frappe.ui.form.on('Project', {
    refresh(frm) {
        if (!frm.is_new()) {
            frm.add_custom_button(__('Add Standard Tasks'), async () => {
                let r = await frappe.call({
                    method: 'myapp.api.get_standard_tasks',
                    args: { project_type: frm.doc.project_type }
                });

                if (r.message && r.message.length) {
                    frm.clear_table('tasks');

                    r.message.forEach(task => {
                        frm.add_child('tasks', {
                            title: task.title,
                            description: task.description,
                            expected_time: task.expected_time,
                            status: 'Open'
                        });
                    });

                    frm.refresh_field('tasks');  // ALWAYS after child table changes
                    frm.dirty();                 // Mark form as modified

                    frappe.show_alert({
                        message: __('Added {0} tasks', [r.message.length]),
                        indicator: 'green'
                    });
                }
            });
        }
    }
});
```

## Example 9: Parallel API Calls with Promise.all

```javascript
frappe.ui.form.on('Customer', {
    async refresh(frm) {
        if (frm.is_new()) return;

        try {
            let [order_count, invoice_count, balance] = await Promise.all([
                frappe.db.count('Sales Order', { customer: frm.doc.name }),
                frappe.db.count('Sales Invoice', { customer: frm.doc.name }),
                frappe.call({
                    method: 'erpnext.accounts.utils.get_balance_on',
                    args: { party_type: 'Customer', party: frm.doc.name }
                })
            ]);

            frm.dashboard.add_indicator(__('Orders: {0}', [order_count]), 'blue');
            frm.dashboard.add_indicator(__('Invoices: {0}', [invoice_count]), 'green');
            frm.dashboard.add_indicator(
                __('Balance: {0}', [format_currency(balance.message)]),
                balance.message > 0 ? 'orange' : 'green'
            );
        } catch (e) {
            console.error('Dashboard load error:', e);
        }
    }
});
```

## Example 10: Confirmation Dialog Before Action

```javascript
frappe.ui.form.on('Sales Invoice', {
    refresh(frm) {
        if (frm.doc.docstatus === 1 && frm.doc.outstanding_amount > 0) {
            frm.add_custom_button(__('Write Off'), () => {
                frappe.confirm(
                    __('Write off {0}?',
                        [format_currency(frm.doc.outstanding_amount, frm.doc.currency)]),
                    async () => {
                        let r = await frappe.call({
                            method: 'myapp.api.write_off_invoice',
                            args: { invoice: frm.doc.name },
                            freeze: true
                        });
                        if (r.message) {
                            frappe.show_alert({
                                message: __('Invoice written off'),
                                indicator: 'green'
                            });
                            frm.reload_doc();
                        }
                    }
                );
            });
        }
    }
});
```

## Example 11: Dynamic Field Properties Based on Type

```javascript
frappe.ui.form.on('Purchase Order', {
    refresh(frm) {
        let is_draft = frm.doc.docstatus === 0;
        frm.toggle_enable('supplier', is_draft);
    },

    order_type(frm) {
        let is_shopping_cart = frm.doc.order_type === 'Shopping Cart';
        frm.set_df_property('shipping_rule', 'reqd', is_shopping_cart ? 1 : 0);
        frm.set_df_property('shipping_rule', 'hidden', is_shopping_cart ? 0 : 1);
    },

    supplier(frm) {
        if (!frm.doc.supplier) return;

        frappe.db.get_value('Supplier', frm.doc.supplier, 'payment_terms')
            .then(r => {
                if (r.message.payment_terms) {
                    frm.set_value('payment_terms_template', r.message.payment_terms);
                }
            });
    }
});
```

## Example 12: Custom Dialog with Table Field

```javascript
frappe.ui.form.on('Sales Order', {
    refresh(frm) {
        if (frm.doc.docstatus === 0 && !frm.is_new()) {
            frm.add_custom_button(__('Bulk Add Items'), () => {
                let d = new frappe.ui.Dialog({
                    title: __('Add Items'),
                    fields: [
                        {
                            fieldname: 'item_list',
                            fieldtype: 'Table',
                            label: __('Items'),
                            in_place_edit: true,
                            fields: [
                                { fieldname: 'item_code', fieldtype: 'Link',
                                  options: 'Item', label: __('Item'), in_list_view: 1, reqd: 1 },
                                { fieldname: 'qty', fieldtype: 'Float',
                                  label: __('Qty'), in_list_view: 1, default: 1 }
                            ]
                        }
                    ],
                    size: 'large',
                    primary_action_label: __('Add'),
                    primary_action(values) {
                        (values.item_list || []).forEach(row => {
                            if (row.item_code) {
                                frm.add_child('items', {
                                    item_code: row.item_code,
                                    qty: row.qty || 1
                                });
                            }
                        });
                        frm.refresh_field('items');
                        frm.dirty();
                        d.hide();
                    }
                });
                d.show();
            });
        }
    }
});
```

## Example 13: List View Customization

```javascript
// In {doctype}_list.js or via Client Script
frappe.listview_settings['Task'] = {
    add_fields: ['status', 'priority', 'assigned_to'],
    filters: [['status', '!=', 'Cancelled']],
    hide_name_column: true,

    get_indicator(doc) {
        const indicators = {
            'Open': [__('Open'), 'orange', 'status,=,Open'],
            'Working': [__('Working'), 'blue', 'status,=,Working'],
            'Completed': [__('Completed'), 'green', 'status,=,Completed'],
            'Cancelled': [__('Cancelled'), 'red', 'status,=,Cancelled']
        };
        return indicators[doc.status] || [__(doc.status), 'grey'];
    },

    button: {
        show(doc) { return doc.status === 'Open'; },
        get_label() { return __('Start'); },
        get_description(doc) { return __('Mark {0} as Working', [doc.name]); },
        action(doc) {
            frappe.call({
                method: 'frappe.client.set_value',
                args: { doctype: 'Task', name: doc.name, fieldname: 'status', value: 'Working' }
            });
        }
    },

    formatters: {
        priority(val) {
            const colors = { 'High': 'red', 'Medium': 'orange', 'Low': 'green' };
            return `<span class="indicator-pill ${colors[val] || 'grey'}">${val || ''}</span>`;
        }
    },

    onload(listview) {
        // Runs once when list view loads
    }
};
```

## Example 14: MultiSelectDialog for Linking Documents

```javascript
frappe.ui.form.on('Purchase Order', {
    refresh(frm) {
        if (frm.doc.docstatus === 0 && !frm.is_new()) {
            frm.add_custom_button(__('Get Items from MR'), () => {
                new frappe.ui.form.MultiSelectDialog({
                    doctype: 'Material Request',
                    target: frm,
                    setters: {
                        schedule_date: null,
                        status: 'Pending'
                    },
                    date_field: 'transaction_date',
                    get_query() {
                        return { filters: { docstatus: 1, status: ['!=', 'Stopped'] } };
                    },
                    action(selections) {
                        if (selections.length) {
                            frappe.call({
                                method: 'myapp.api.get_items_from_mr',
                                args: { material_requests: selections },
                                callback(r) {
                                    if (r.message) {
                                        r.message.forEach(item => {
                                            frm.add_child('items', item);
                                        });
                                        frm.refresh_field('items');
                                        frm.dirty();
                                    }
                                }
                            });
                        }
                    }
                });
            }, __('Get Items'));
        }
    }
});
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-clientscripts/references/examples.md*
