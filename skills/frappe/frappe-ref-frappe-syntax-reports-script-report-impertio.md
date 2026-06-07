---
title: Frappe Ref — Frappe Syntax: Reports: Script Report (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'reports', 'reference', 'script-report']
source: Impertio
---

# Script Report JS API Reference

## Overview

Script Reports have two files:
- `{report_name}.py` — Server-side `execute()` function
- `{report_name}.js` — Client-side filters, hooks, and formatting

## JS File Structure

```javascript
frappe.query_reports["Report Name"] = {
    // Filters array (required)
    filters: [...],

    // Lifecycle hooks
    onload: function(report) { },
    after_datatable_render: function(datatable) { },

    // Display configuration
    formatter: function(value, row, column, data, default_formatter) { },
    get_datatable_options: function(options) { },

    // Tree mode
    tree: false,
    initial_depth: 1,
    parent_field: "parent_account",

    // Export
    export_hidden_cols: false
};
```

## Filter Configuration

### All Filter Properties

```javascript
{
    fieldname: "company",         // Required: maps to filters dict key
    label: __("Company"),         // Required: display label
    fieldtype: "Link",            // Required: input type
    options: "Company",           // Required for Link/Select
    default: "My Company",       // Optional: default value
    reqd: 1,                     // Optional: mandatory filter
    hidden: 0,                   // Optional: hide from UI
    read_only: 0,                // Optional: non-editable
    width: "80px",               // Optional: filter width
    get_query: function() {      // Optional: custom query for Link
        return {
            filters: { "is_group": 0 }
        };
    },
    on_change: function() {      // Optional: callback on value change
        frappe.query_report.refresh();
    },
    depends_on: 'eval:doc.company=="My Company"'  // Optional: conditional display
}
```

### Filter Fieldtype Examples

```javascript
// Link — autocomplete from DocType
{ fieldname: "customer", fieldtype: "Link", options: "Customer" }

// Select — dropdown
{ fieldname: "status", fieldtype: "Select",
  options: "\nDraft\nSubmitted\nCancelled" }  // leading \n for blank option

// Date
{ fieldname: "from_date", fieldtype: "Date",
  default: frappe.datetime.add_months(frappe.datetime.get_today(), -1) }

// DateRange — returns [from_date, to_date]
{ fieldname: "date_range", fieldtype: "DateRange",
  default: [frappe.datetime.add_months(frappe.datetime.get_today(), -1),
            frappe.datetime.get_today()] }

// Check — boolean
{ fieldname: "show_cancelled", fieldtype: "Check", default: 0 }

// Dynamic Link — depends on another filter
{ fieldname: "party_type", fieldtype: "Link", options: "DocType",
  get_query: function() {
      return { filters: { name: ["in", ["Customer", "Supplier"]] } };
  }
},
{ fieldname: "party", fieldtype: "Dynamic Link", options: "party_type" }

// MultiSelectList
{ fieldname: "warehouses", fieldtype: "MultiSelectList",
  get_data: function(txt) {
      return frappe.db.get_link_options("Warehouse", txt);
  }
}

// Int
{ fieldname: "limit", fieldtype: "Int", default: 20 }
```

## Lifecycle Hooks

### onload

Fires once when report loads. Use for dynamic filter setup:

```javascript
onload: function(report) {
    // Add custom button
    report.page.add_inner_button(__("Download PDF"), function() {
        // custom action
    });

    // Modify filters dynamically
    let company_filter = report.get_filter("company");
    company_filter.df.default = "Default Company";
}
```

### after_datatable_render

Fires after the DataTable renders. Use for post-render modifications:

```javascript
after_datatable_render: function(datatable) {
    // Highlight specific rows
    $(datatable.wrapper).find(".dt-row").each(function() {
        // custom styling
    });
}
```

## Formatter

Custom cell formatting. Return HTML string:

```javascript
formatter: function(value, row, column, data, default_formatter) {
    value = default_formatter(value, row, column, data);

    if (column.fieldname === "balance" && data && data.balance < 0) {
        value = "<span style='color:red'>" + value + "</span>";
    }

    if (data && data.bold) {
        value = "<b>" + value + "</b>";
    }

    return value;
}
```

## DataTable Options

Customize the DataTable rendering:

```javascript
get_datatable_options: function(options) {
    return Object.assign(options, {
        checkboxColumn: true,
        noDataMessage: __("No records found"),
        dynamicRowHeight: true
    });
}
```

## Tree Mode

Enable hierarchical display:

```javascript
frappe.query_reports["Account Balance"] = {
    tree: true,
    initial_depth: 3,          // levels expanded by default
    parent_field: "parent_account",  // field linking to parent row
    filters: [...]
};
```

For tree mode, `execute()` must return data with:
- An `indent` field (integer, 0 = root level)
- Rows ordered so children appear after their parent

## Accessing Report from JS

```javascript
// Refresh report
frappe.query_report.refresh();

// Get filter value
let company = frappe.query_report.get_filter_value("company");

// Set filter value
frappe.query_report.set_filter_value("company", "My Company");

// Get all filter values
let filters = frappe.query_report.get_filter_values();

// Toggle filter visibility
frappe.query_report.toggle_filter_display("status", true);  // hide
```

## Custom Buttons and Actions

```javascript
onload: function(report) {
    report.page.add_inner_button(__("Create Invoice"), function() {
        let checked = report.datatable.getCheckedRowIndexes();
        if (checked.length === 0) {
            frappe.throw(__("Select at least one row"));
            return;
        }
        // process checked rows
        let selected_data = checked.map(i => report.data[i]);
        // call server method
        frappe.call({
            method: "myapp.api.create_invoices",
            args: { rows: selected_data },
            callback: function(r) {
                frappe.msgprint(__("Invoices created"));
                report.refresh();
            }
        });
    });
}
```

## Critical Rules

- **ALWAYS** use `__()` for translatable strings in JS filters and labels
- **ALWAYS** call `default_formatter(value, row, column, data)` first in custom formatters — then modify the result
- **NEVER** mutate `data` directly in the formatter — it causes rendering bugs
- **ALWAYS** return a value from the formatter function — returning `undefined` blanks the cell
- **NEVER** use `frappe.query_reports` (plural) for accessing the current report instance — use `frappe.query_report` (singular)

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-reports/references/script-report.md*
