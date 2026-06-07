---
name: frappe-client-scripting
description: >
  Customize Frappe desk forms and lists with client scripts — frappe.ui.form.on handlers, field
  events, set_query link filters, set_df_property, custom buttons, frappe.call, and list view
  settings. Use when adding form interactivity, dynamic filters, validation, or buttons in the desk
  UI. Trigger on: "client script", "frappe.ui.form.on", "form event", "set_query", "custom button",
  "frappe.call", "list view settings", "cur_frm".
---

# Frappe Client Scripting

Client scripts run in the browser on desk forms/lists. Use them for UX; never for security (enforce
on the server).

## Form events

```javascript
frappe.ui.form.on("Sales Order", {
    refresh(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button("Sync Now", () => run_sync(frm));
        }
    },
    customer(frm) {                         // fires when 'customer' changes
        frm.set_value("currency", "");
    },
    validate(frm) {
        if (frm.doc.grand_total < 0) {
            frappe.throw(__("Total cannot be negative"));
        }
    },
});

// Child table events
frappe.ui.form.on("Sales Order Item", {
    qty(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        frappe.model.set_value(cdt, cdn, "amount", row.qty * row.rate);
    },
});
```

## Dynamic link filters

```javascript
frm.set_query("item_code", "items", () => ({
    filters: { is_sales_item: 1, disabled: 0 },
}));

// Server-side query method (for complex filters)
frm.set_query("item_code", "items", () => ({
    query: "myapp.queries.item_query",
    filters: { is_sales_item: 1 },
}));
```

Server query methods must be `@frappe.whitelist()` and decorated with `@frappe.validate_and_sanitize_search_inputs` to prevent SQL injection:

```python
@frappe.whitelist()
@frappe.validate_and_sanitize_search_inputs
def item_query(doctype, txt, searchfield, start, page_len, filters):
    return frappe.db.sql("""SELECT name FROM `tabItem`
        WHERE name LIKE %(txt)s LIMIT %(page_len)s""",
        {"txt": f"%{txt}%", "page_len": page_len})
```

## Field control at runtime

```javascript
frm.set_df_property("discount", "hidden", frm.doc.status !== "Draft");
frm.toggle_reqd("reason", frm.doc.status === "Rejected");
frm.set_value("approved_by", frappe.session.user);
```

## Calling the server

```javascript
frappe.call({
    method: "myapp.api.order_summary",
    args: { customer: frm.doc.customer },
    freeze: true,
    callback: (r) => {
        if (!r.exc) frappe.msgprint(`${r.message.count} orders`);
    },
});
```

## List view settings

```javascript
frappe.listview_settings["Sales Order"] = {
    add_fields: ["status"],
    get_indicator(doc) {
        return doc.status === "Closed"
            ? [__("Closed"), "green", "status,=,Closed"]
            : [__("Open"), "orange", "status,=,Open"];
    },
};
```

## Rules

- Client scripts are **UX only** — duplicate every guard in a server `validate`/permission check.
- Use `__("...")` for all user-facing strings (translatable).
- Prefer `frappe.model.set_value` for child rows; mutating `locals` directly won't refresh the grid.
- Don't put secrets or privileged logic in client scripts — they're shipped to the browser.
- Ship reusable logic as an app JS file via `doctype_js` hook, not as DB "Client Script" records, for versioning.

## From Frappe docs

- Server-side link query override (`set_query` + custom method): https://docs.frappe.io/framework/user/en/guides/app-development/overriding-link-query-by-custom-script
