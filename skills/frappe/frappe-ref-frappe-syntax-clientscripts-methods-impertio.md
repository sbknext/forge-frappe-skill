---
title: Frappe Ref — Frappe Syntax: Clientscripts: Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'clientscripts', 'reference', 'methods']
source: Impertio
---

# Client Script Methods Reference

## frm Object Methods

### Value Manipulation

#### frm.set_value(fieldname, value)
Sets field value. Triggers change events, dirty flag, and UI refresh. Returns Promise.

```javascript
// Single field
frm.set_value('status', 'Approved');

// Multiple fields at once
frm.set_value({
    status: 'Approved',
    priority: 'High',
    due_date: frappe.datetime.add_days(frappe.datetime.now_date(), 7)
});

// With Promise handling
frm.set_value('status', 'Approved').then(() => {
    console.log('Value set and change events fired');
});

// Await pattern
await frm.set_value('status', 'Approved');
```

#### frm.doc.fieldname
Direct read access to field values. **NEVER write to frm.doc directly.**

```javascript
let customer = frm.doc.customer;           // String field
let items = frm.doc.items;                 // Child table (array of objects)
let total = frm.doc.grand_total;           // Currency field
let status = frm.doc.docstatus;            // 0=Draft, 1=Submitted, 2=Cancelled
```

### Field Display Properties

#### frm.toggle_display(fieldname, show)
Shows or hides fields. Accepts string or array.

```javascript
frm.toggle_display('priority', frm.doc.status === 'Open');
frm.toggle_display(['priority', 'due_date', 'assigned_to'], condition);
```

#### frm.toggle_reqd(fieldname, required)
Makes fields mandatory or optional. Accepts string or array.

```javascript
frm.toggle_reqd('due_date', true);
frm.toggle_reqd(['email', 'phone'], frm.doc.customer_type === 'Company');
```

#### frm.toggle_enable(fieldname, enable)
Makes fields editable or read-only. Accepts string or array.

```javascript
frm.toggle_enable('amount', false);   // Read-only
frm.toggle_enable('amount', true);    // Editable
```

#### frm.set_df_property(fieldname, property, value)
Sets any docfield property dynamically.

```javascript
// Common properties
frm.set_df_property('status', 'options', ['New', 'Open', 'Closed']);
frm.set_df_property('amount', 'read_only', 1);
frm.set_df_property('description', 'hidden', 1);
frm.set_df_property('priority', 'reqd', 1);
frm.set_df_property('rate', 'precision', 4);
frm.set_df_property('notes', 'label', 'Internal Notes');
frm.set_df_property('description', 'description', 'Enter detailed notes here');
```

#### frm.set_intro(message, color)
Displays intro text at the top of the form.

```javascript
frm.set_intro('This document requires approval', 'orange');
// Colors: 'blue', 'red', 'orange', 'green', 'yellow'
frm.set_intro('');  // Clear intro
```

### Link Field Queries

#### frm.set_query(fieldname, [tablename], callback)
Filters options in Link fields. ALWAYS call in `setup` event.

```javascript
// Form-level Link filter
frm.set_query('customer', () => ({
    filters: { disabled: 0, customer_type: 'Company' }
}));

// With dynamic filters (callback reads frm.doc at execution time)
frm.set_query('customer', () => ({
    filters: { territory: frm.doc.territory }
}));

// Child table Link filter
frm.set_query('item_code', 'items', (doc, cdt, cdn) => {
    let row = locals[cdt][cdn];
    return {
        filters: {
            is_sales_item: 1,
            item_group: row.item_group || undefined
        }
    };
});

// Server-side query (for complex filtering logic)
frm.set_query('customer', () => ({
    query: 'myapp.queries.get_customers_by_region',
    filters: { region: frm.doc.region }
}));
```

### Child Table Methods

#### frm.add_child(tablename, values)
Adds a row to child table. Returns the new row object.

```javascript
let row = frm.add_child('items', {
    item_code: 'ITEM-001',
    qty: 5,
    rate: 100
});
frm.refresh_field('items');  // ALWAYS required after modification
```

#### frm.clear_table(tablename)
Removes all rows from a child table.

```javascript
frm.clear_table('items');
frm.refresh_field('items');  // ALWAYS required
```

#### frm.refresh_field(fieldname)
Redraws field in the UI. **ALWAYS call after child table modifications.**

```javascript
frm.refresh_field('items');       // After child table change
frm.refresh_field('grand_total'); // After computed field update
```

#### frm.get_selected()
Returns selected rows in table fields.

```javascript
let selected = frm.get_selected();
// Returns: { items: ['row-id-1', 'row-id-2'] }
```

### Form State Methods

#### frm.is_new()
Returns `true` if document has never been saved.

```javascript
if (frm.is_new()) {
    frm.set_value('status', 'Draft');
}
```

#### frm.is_dirty()
Returns `true` if form has unsaved changes.

```javascript
if (frm.is_dirty()) {
    frappe.warn(__('Warning'), __('You have unsaved changes'));
}
```

#### frm.dirty()
Marks form as modified. Triggers "Not Saved" indicator. Use after programmatic changes.

```javascript
frm.doc.items.forEach(row => { row.discount = 5; });
frm.refresh_field('items');
frm.dirty();  // Show unsaved indicator
```

#### frm.enable_save() / frm.disable_save()
Toggles save button availability.

```javascript
frm.disable_save();  // Prevent saving
frm.enable_save();   // Allow saving
```

### Navigation and Refresh

#### frm.reload_doc()
Reloads document from server and calls `frm.refresh()`.

```javascript
await frm.reload_doc();
```

#### frm.refresh()
Re-renders form with current data. Triggers: before_load > onload > refresh > onload_post_render.

#### frm.save(action)
Saves document. Optional action: `'Submit'`, `'Cancel'`, `'Update'`.

```javascript
frm.save().then(() => frappe.show_alert(__('Saved!')));
frm.save('Submit');  // Submit the document
```

#### frm.trigger(event)
Programmatically triggers a form event handler.

```javascript
frm.trigger('refresh');     // Re-run refresh handler
frm.trigger('customer');    // Re-run customer change handler
```

### Custom Buttons

#### frm.add_custom_button(label, action, [group])
Adds button to inner toolbar. Returns jQuery element.

```javascript
// Standalone button
frm.add_custom_button(__('Generate Report'), () => {
    frappe.call({ method: 'myapp.api.generate_report', args: { name: frm.doc.name } });
});

// Grouped button (dropdown)
frm.add_custom_button(__('Sales Invoice'), () => { /* create invoice */ }, __('Create'));
frm.add_custom_button(__('Delivery Note'), () => { /* create DN */ }, __('Create'));
```

#### frm.remove_custom_button(label, [group])
Removes a custom button.

```javascript
frm.remove_custom_button(__('Generate Report'));
frm.remove_custom_button(__('Sales Invoice'), __('Create'));
```

#### frm.clear_custom_buttons()
Removes all custom buttons.

#### frm.change_custom_button_type(label, group, type)
Changes button styling.

```javascript
frm.change_custom_button_type(__('Approve'), null, 'primary');
```

#### frm.page.set_primary_action(label, action)
Sets the primary action button (blue, top-right).

```javascript
frm.page.set_primary_action(__('Process'), () => {
    frm.call('process').then(() => frm.reload_doc());
});
```

### Communication

#### frm.call(method, args)
Calls a whitelisted method on the document controller. Returns Promise.

```javascript
let r = await frm.call('calculate_taxes', { include_shipping: true });
console.log(r.message);
```

#### frm.email_doc(message)
Opens email dialog for this document.

```javascript
frm.email_doc('Please review this document.');
```

---

## frappe.* Client-Side Methods

### Server Communication

#### frappe.call(options)
Calls a whitelisted Python method via AJAX. Returns Promise.

```javascript
let r = await frappe.call({
    method: 'myapp.api.get_customer_data',   // Full dotted path
    args: { customer: frm.doc.customer },     // Arguments
    freeze: true,                              // Show loading overlay
    freeze_message: __('Loading...'),          // Custom message
    // Or use callback style:
    callback: (r) => { if (r.message) { /* use */ } },
    error: (r) => { frappe.msgprint(__('Error')); }
});
```

**NEVER use `async: false`** — it blocks the browser thread.

#### frappe.db.get_value(doctype, name, fieldname)
Gets field value(s) from a document.

```javascript
// Single field
let r = await frappe.db.get_value('Customer', frm.doc.customer, 'credit_limit');
console.log(r.message.credit_limit);

// Multiple fields
let r = await frappe.db.get_value('Customer', frm.doc.customer,
    ['credit_limit', 'territory']);

// With filters instead of name
let r = await frappe.db.get_value('Customer',
    { customer_name: 'Acme Corp' }, 'name');
```

#### frappe.db.get_list(doctype, args)
Gets a list of documents matching criteria.

```javascript
let orders = await frappe.db.get_list('Sales Order', {
    filters: { customer: frm.doc.customer, docstatus: 1 },
    fields: ['name', 'grand_total', 'status'],
    order_by: 'creation desc',
    limit: 10
});
// Returns array of objects
```

#### frappe.db.count(doctype, filters)
Counts documents matching filters.

```javascript
let count = await frappe.db.count('Sales Order', { customer: frm.doc.customer });
```

### User Interface

#### frappe.msgprint(message | options)
Shows modal message dialog.

```javascript
frappe.msgprint(__('Operation completed'));

frappe.msgprint({
    title: __('Success'),
    message: __('Invoice created successfully'),
    indicator: 'green'
});

// With primary action
frappe.msgprint({
    title: __('Confirm'),
    message: __('Proceed with operation?'),
    primary_action: {
        label: __('Proceed'),
        server_action: 'myapp.api.do_action',
        args: { name: frm.doc.name }
    }
});
```

#### frappe.throw(message)
Shows error modal AND throws exception. Halts execution.

```javascript
if (frm.doc.amount < 0) {
    frappe.throw(__('Amount cannot be negative'));
}
```

#### frappe.show_alert(message | options, seconds)
Non-blocking notification toast. Default 7 seconds.

```javascript
frappe.show_alert(__('Saved!'), 3);
frappe.show_alert({ message: __('Done'), indicator: 'green' }, 5);
```

#### frappe.confirm(message, if_yes, if_no)
Modal confirmation dialog.

```javascript
frappe.confirm(
    __('Are you sure you want to delete?'),
    () => { /* Yes */ },
    () => { /* No */ }
);
```

#### frappe.warn(title, message, proceed_action, primary_label, is_minimizable)
Warning modal with optional minimize.

```javascript
frappe.warn(__('Warning'), __('Unsaved changes exist'),
    () => { /* proceed */ }, __('Continue'), true);
```

#### frappe.prompt(fields, callback, title, primary_label)
Quick input dialog.

```javascript
// Single field
frappe.prompt('Reason', (values) => { console.log(values.value); });

// Multiple fields
frappe.prompt([
    { label: 'Name', fieldname: 'name', fieldtype: 'Data', reqd: 1 },
    { label: 'Date', fieldname: 'date', fieldtype: 'Date' }
], (values) => { console.log(values); }, __('Enter Details'));
```

#### frappe.show_progress(title, count, total, description)
Progress bar indicator.

```javascript
frappe.show_progress(__('Importing'), 45, 100, __('Please wait'));
```

#### new frappe.ui.Dialog(options)
Full-featured dialog with custom fields.

```javascript
let d = new frappe.ui.Dialog({
    title: __('Enter Details'),
    fields: [
        { label: 'Name', fieldname: 'name', fieldtype: 'Data', reqd: 1 },
        { label: 'Type', fieldname: 'type', fieldtype: 'Select',
          options: 'Type A\nType B\nType C' }
    ],
    size: 'small',  // 'small', 'large', 'extra-large'
    primary_action_label: __('Submit'),
    primary_action(values) {
        console.log(values);
        d.hide();
    }
});
d.show();
```

### Child Table Utilities

#### frappe.get_doc(cdt, cdn)
Gets child row data object. Use inside child table event handlers.

```javascript
frappe.ui.form.on('Sales Invoice Item', {
    qty(frm, cdt, cdn) {
        let row = frappe.get_doc(cdt, cdn);
        // row.qty, row.rate, row.amount, etc.
    }
});
```

#### frappe.model.set_value(cdt, cdn, fieldname, value)
Sets value in child row. Triggers change events. Returns Promise.

```javascript
frappe.model.set_value(cdt, cdn, 'amount', row.qty * row.rate);

// Multiple values
frappe.model.set_value(cdt, cdn, {
    amount: row.qty * row.rate,
    net_amount: row.qty * row.rate * (1 - row.discount / 100)
});
```

#### frappe.model.open_mapped_doc(options)
Opens a new document pre-filled from a source document.

```javascript
frappe.model.open_mapped_doc({
    method: 'erpnext.selling.doctype.sales_order.sales_order.make_sales_invoice',
    frm: frm
});
```

### Translation

#### __(message, replace, context)
Translates a string. ALWAYS wrap user-facing text.

```javascript
__('Hello World')
__('Hello {0}', [user_name])
__('Total: {0}', [frm.doc.grand_total])
__('Row {0}: Quantity must be positive', [idx + 1])
```

### Date/Time Utilities

```javascript
frappe.datetime.now_date()                          // 'YYYY-MM-DD'
frappe.datetime.now_datetime()                      // 'YYYY-MM-DD HH:mm:ss'
frappe.datetime.add_days('2024-01-01', 7)           // '2024-01-08'
frappe.datetime.add_months('2024-01-01', 1)         // '2024-02-01'
frappe.datetime.str_to_user('2024-01-15')           // User date format
frappe.datetime.get_diff(date1, date2)              // Difference in days
```

### Formatting Utilities

```javascript
frappe.format(value, { fieldtype: 'Currency' })     // Formatted currency
frappe.format('2024-09-08', { fieldtype: 'Date' })  // User date format
format_currency(1234.50, 'USD')                     // '$1,234.50'
```

### Routing

```javascript
frappe.set_route('Form', 'Sales Order', 'SO-0001');
frappe.set_route('List', 'Sales Order');
frappe.new_doc('Sales Order');
frappe.new_doc('Task', { subject: 'New Task' });    // With defaults
let route = frappe.get_route();                      // ['Form', 'Sales Order', ...]
```

### Asset Loading

```javascript
frappe.require('/assets/myapp/js/library.js', () => {
    // Library is now available
});
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-clientscripts/references/methods.md*
