---
title: Frappe Ref — Frappe Impl: Ui Components: Workflows (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'ui', 'components', 'reference', 'workflows']
source: Impertio
---

# UI Components Workflows — Extended

## Complete Dialog with Validation and Async Submit

```javascript
let d = new frappe.ui.Dialog({
    title: "Create Invoice",
    size: "large",
    fields: [
        { label: "Customer", fieldname: "customer", fieldtype: "Link",
          options: "Customer", reqd: 1,
          change() {
              // Dynamic filtering when customer changes
              let customer = d.get_value("customer");
              if (customer) {
                  d.fields_dict.items.grid.get_field("item").get_query = () => ({
                      filters: { "customer": customer }
                  });
              }
          }
        },
        { fieldtype: "Section Break", label: "Items" },
        { label: "Items", fieldname: "items", fieldtype: "Table",
          in_place_edit: true, reqd: 1,
          fields: [
              { fieldname: "item", label: "Item", fieldtype: "Link",
                options: "Item", in_list_view: 1, reqd: 1 },
              { fieldname: "qty", label: "Qty", fieldtype: "Int",
                in_list_view: 1, default: 1, reqd: 1 },
              { fieldname: "rate", label: "Rate", fieldtype: "Currency",
                in_list_view: 1 },
          ]
        },
        { fieldtype: "Section Break" },
        { label: "Notes", fieldname: "notes", fieldtype: "Small Text" },
    ],
    primary_action_label: "Create Invoice",
    primary_action(values) {
        // Disable button to prevent double-click
        d.disable_primary_action();

        frappe.xcall("myapp.api.create_invoice", {
            customer: values.customer,
            items: values.items,
            notes: values.notes,
        }).then((invoice_name) => {
            d.hide();
            frappe.show_alert({ message: __("Invoice {0} created", [invoice_name]),
                indicator: "green" });
            frappe.set_route("Form", "Sales Invoice", invoice_name);
        }).catch(() => {
            // Re-enable on error so user can retry
            d.enable_primary_action();
        });
    }
});
d.show();
```

## MultiSelectDialog for Record Selection

```javascript
new frappe.ui.form.MultiSelectDialog({
    doctype: "Item",
    target: cur_frm,
    setters: {
        item_group: null,
        brand: null,
    },
    add_filters_group: 1,
    primary_action_label: "Add Items",
    columns: ["item_name", "item_group", "brand", "stock_uom"],
    action(selections) {
        // selections = array of selected document names
        selections.forEach(item_name => {
            let row = cur_frm.add_child("items");
            frappe.model.set_value(row.doctype, row.name, "item_code", item_name);
        });
        cur_frm.refresh_field("items");
    }
});
```

## Complete Custom Page with Data Table

```javascript
frappe.pages["inventory-dashboard"].on_page_load = function(wrapper) {
    let page = frappe.ui.make_app_page({
        parent: wrapper,
        title: "Inventory Dashboard",
        single_column: true,
    });

    // Toolbar filters
    let warehouse = page.add_field({
        label: "Warehouse",
        fieldtype: "Link",
        fieldname: "warehouse",
        options: "Warehouse",
        change() { refresh(page); }
    });

    let item_group = page.add_field({
        label: "Item Group",
        fieldtype: "Link",
        fieldname: "item_group",
        options: "Item Group",
        change() { refresh(page); }
    });

    // Action buttons
    page.set_primary_action("Export", () => export_data(page));
    page.add_menu_item("Print", () => window.print());

    // Content area
    $(page.body).html('<div id="inventory-table"></div>');

    refresh(page);
};

function refresh(page) {
    let filters = page.get_form_values();
    page.set_indicator("Loading...", "orange");

    frappe.xcall("myapp.api.get_inventory", filters).then(data => {
        render_table(data);
        page.set_indicator("Updated", "green");
        page.set_title_sub(`${data.length} items`);
    });
}

function render_table(data) {
    let container = document.getElementById("inventory-table");
    // Use frappe.DataTable or custom HTML
    new frappe.DataTable(container, {
        columns: [
            { name: "Item", width: 200 },
            { name: "Warehouse", width: 150 },
            { name: "Qty", width: 100 },
            { name: "Value", width: 120 },
        ],
        data: data.map(d => [d.item_name, d.warehouse, d.qty, d.value]),
    });
}
```

## List View with Multiple Buttons (Dropdown)

```javascript
frappe.listview_settings["Sales Order"] = {
    add_fields: ["status", "grand_total", "customer", "delivery_status"],

    get_indicator(doc) {
        const map = {
            "Draft": ["Draft", "red", "status,=,Draft"],
            "To Deliver and Bill": ["To Deliver", "orange",
                "status,=,To Deliver and Bill"],
            "Completed": ["Completed", "green", "status,=,Completed"],
            "Cancelled": ["Cancelled", "darkgrey", "status,=,Cancelled"],
        };
        return map[doc.status] || ["Unknown", "grey", ""];
    },

    // Dropdown with multiple actions per row
    dropdown_button: {
        buttons: [
            {
                show(doc) { return doc.status === "Draft"; },
                get_label() { return __("Submit"); },
                get_description(doc) { return __("Submit {0}", [doc.name]); },
                action(doc) {
                    frappe.xcall("frappe.client.submit", { doc: doc.name })
                        .then(() => cur_list.refresh());
                }
            },
            {
                show(doc) { return doc.docstatus === 1; },
                get_label() { return __("Make Invoice"); },
                get_description(doc) { return __("Create invoice for {0}", [doc.name]); },
                action(doc) {
                    frappe.set_route("Form", "Sales Invoice", {
                        sales_order: doc.name
                    });
                }
            },
        ]
    },

    onload(listview) {
        listview.page.add_inner_button("Bulk Update", () => {
            frappe.prompt(
                { label: "Status", fieldname: "status", fieldtype: "Select",
                  options: "Open\nClosed", reqd: 1 },
                (values) => {
                    let names = listview.get_checked_items().map(d => d.name);
                    frappe.xcall("myapp.api.bulk_update_status", {
                        orders: names, status: values.status
                    }).then(() => listview.refresh());
                },
                "Set Status"
            );
        });
    }
};
```

## Realtime Chat-Style Updates

```python
# Server: myapp/api.py
import frappe

@frappe.whitelist()
def send_message(room, message):
    frappe.publish_realtime(
        "new_message",
        {"room": room, "message": message, "sender": frappe.session.user,
         "timestamp": frappe.utils.now()},
        after_commit=True
    )
    return "ok"
```

```javascript
// Client: listen and render
frappe.realtime.on("new_message", (data) => {
    append_message(data.room, data.message, data.sender, data.timestamp);
    frappe.show_alert({ message: `New message from ${data.sender}`,
        indicator: "blue" }, 3);
});
```

## Calendar View with Custom Events

```javascript
// myapp/doctype/appointment/appointment_calendar.js
frappe.views.calendar["Appointment"] = {
    field_map: {
        start: "scheduled_date",
        end: "end_date",
        id: "name",
        title: "patient_name",
        allDay: "all_day",
        color: "color",
    },
    gantt: false,
    filters: [
        { fieldtype: "Link", fieldname: "department", label: "Department",
          options: "Medical Department" },
    ],
    get_events_method: "myapp.api.get_appointments",
};
```

```python
# myapp/api.py
@frappe.whitelist()
def get_appointments(start, end, filters=None):
    conditions = {"scheduled_date": ("between", [start, end])}
    if filters:
        import json
        filters = json.loads(filters) if isinstance(filters, str) else filters
        conditions.update(filters)

    return frappe.get_all("Appointment",
        filters=conditions,
        fields=["name", "patient_name", "scheduled_date", "end_date",
                "all_day", "color", "department"],
    )
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-ui-components/references/workflows.md*
