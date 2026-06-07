---
title: Frappe Ref — Frappe Impl: Ui Components: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'ui', 'components', 'reference', 'examples']
source: Impertio
---

# UI Components Examples — Complete Code

## Example 1: Confirmation Dialog Before Dangerous Action

```javascript
frappe.ui.form.on("Sales Order", {
    custom_cancel_all_items(frm) {
        frappe.confirm(
            __("This will cancel all {0} items. Continue?", [frm.doc.items.length]),
            () => {
                frappe.xcall("myapp.api.cancel_all_items", { order: frm.doc.name })
                    .then(() => {
                        frm.reload_doc();
                        frappe.show_alert({ message: "All items cancelled",
                            indicator: "green" });
                    });
            }
        );
    }
});
```

## Example 2: Progress Bar for Bulk Operation

```python
# Server
@frappe.whitelist()
def process_invoices(invoices):
    invoices = frappe.parse_json(invoices)
    total = len(invoices)

    for i, inv in enumerate(invoices):
        submit_invoice(inv)
        frappe.publish_progress(
            percent=int((i + 1) / total * 100),
            title="Submitting Invoices",
            description=f"Processing {inv} ({i+1}/{total})",
        )

    return {"processed": total}
```

```javascript
// Client
frappe.xcall("myapp.api.process_invoices", {
    invoices: selected_invoices
}).then(r => {
    frappe.msgprint(__("{0} invoices processed", [r.processed]));
});
// Progress bar appears automatically via frappe.publish_progress
```

## Example 3: Dynamic Dialog — Fields Change Based on Selection

```javascript
let d = new frappe.ui.Dialog({
    title: "New Entry",
    fields: [
        { label: "Type", fieldname: "type", fieldtype: "Select",
          options: "Expense\nIncome\nTransfer", reqd: 1,
          change() {
              let type = d.get_value("type");
              d.set_df_property("expense_account", "hidden", type !== "Expense");
              d.set_df_property("income_account", "hidden", type !== "Income");
              d.set_df_property("transfer_to", "hidden", type !== "Transfer");
          }
        },
        { label: "Amount", fieldname: "amount", fieldtype: "Currency", reqd: 1 },
        { label: "Expense Account", fieldname: "expense_account",
          fieldtype: "Link", options: "Account", hidden: 1 },
        { label: "Income Account", fieldname: "income_account",
          fieldtype: "Link", options: "Account", hidden: 1 },
        { label: "Transfer To", fieldname: "transfer_to",
          fieldtype: "Link", options: "Account", hidden: 1 },
    ],
    primary_action_label: "Save",
    primary_action(values) {
        frappe.xcall("myapp.api.create_entry", values).then(() => d.hide());
    }
});
d.show();
```

## Example 4: Scanner in Stock Entry

```javascript
frappe.ui.form.on("Stock Entry", {
    custom_scan_items(frm) {
        let scanner = new frappe.ui.Scanner({
            dialog: true,
            multiple: true,
            on_scan(data) {
                let barcode = data.decodedText;
                // Prevent duplicate scans
                let exists = frm.doc.items.find(d => d.barcode === barcode);
                if (exists) {
                    exists.qty += 1;
                    frm.refresh_field("items");
                    frappe.show_alert({ message: `${barcode}: qty +1`,
                        indicator: "blue" });
                    return;
                }

                frappe.xcall("erpnext.stock.utils.get_item_by_barcode",
                    { barcode }
                ).then(item => {
                    if (item) {
                        let row = frm.add_child("items");
                        frappe.model.set_value(row.doctype, row.name, {
                            item_code: item.item_code,
                            barcode: barcode,
                            qty: 1,
                        });
                        frm.refresh_field("items");
                        frappe.show_alert({ message: `Added: ${item.item_name}`,
                            indicator: "green" });
                    } else {
                        frappe.show_alert({ message: `Unknown barcode: ${barcode}`,
                            indicator: "red" });
                    }
                });
            }
        });
    }
});
```

## Example 5: Realtime Document Collaboration Indicator

```python
# Server: track who is viewing a document
@frappe.whitelist()
def register_viewer(doctype, docname):
    frappe.publish_realtime(
        "viewer_joined",
        {"user": frappe.session.user, "full_name": frappe.utils.get_fullname()},
        doctype=doctype,
        docname=docname,
        after_commit=True,
    )
```

```javascript
// Client: show active viewers
frappe.ui.form.on("Project", {
    onload(frm) {
        // Register self as viewer
        frappe.xcall("myapp.api.register_viewer", {
            doctype: frm.doctype, docname: frm.docname
        });

        // Listen for other viewers
        frappe.realtime.on("viewer_joined", (data) => {
            frappe.show_alert({
                message: __("{0} is also viewing this document", [data.full_name]),
                indicator: "blue"
            }, 5);
        });
    },

    before_unload(frm) {
        frappe.realtime.off("viewer_joined");
    }
});
```

## Example 6: Custom Formatter in List View

```javascript
frappe.listview_settings["Payment Entry"] = {
    add_fields: ["payment_type", "paid_amount", "status"],

    formatters: {
        paid_amount(val, df, doc) {
            // Color code by amount threshold
            let color = val > 10000 ? "red" : val > 1000 ? "orange" : "green";
            return `<span style="color: var(--${color})">${format_currency(val)}</span>`;
        },
        payment_type(val) {
            const icons = { Receive: "↓", Pay: "↑", "Internal Transfer": "↔" };
            return `${icons[val] || ""} ${val}`;
        }
    },

    get_indicator(doc) {
        return {
            Draft: ["Draft", "red", "docstatus,=,0"],
            Submitted: ["Submitted", "blue", "docstatus,=,1"],
            Cancelled: ["Cancelled", "darkgrey", "docstatus,=,2"],
        }[doc.status] || ["", "grey", ""];
    }
};
```

## Example 7: Tree View for Nested Categories

Tree View is automatic for DocTypes with `is_tree = 1`. Configuration:

```python
# In DocType JSON or via code
# Required fields (auto-added for tree DocTypes):
# - parent_{doctype_name} (Link to self)
# - is_group (Check)
# - lft, rgt (Int — Nested Set Model, managed by Frappe)
```

```javascript
// Optional: customize tree behavior via {doctype}_tree.js
frappe.treeview_settings["Department"] = {
    breadcrumb: "HR",
    title: "Department Tree",
    filters: [
        { fieldname: "company", fieldtype: "Link", options: "Company",
          label: "Company", default: frappe.defaults.get_default("company") }
    ],
    get_tree_root: false,  // Show root nodes
    root_label: "All Departments",
    onload(treeview) {
        treeview.page.add_inner_button("Expand All", () => {
            treeview.tree.load_children(treeview.tree.root_node);
        });
    }
};
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-ui-components/references/examples.md*
