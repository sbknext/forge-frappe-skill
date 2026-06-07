---
title: Frappe Ref — Form Layout & Field Grouping (frappe-apps-manager)
category: frappe
tags: [frappe, doctype, form, layout, section-break, column-break, ux]
source: vyogotech/frappe-apps-manager
---

# Form Layout & Field Grouping

## When to use

Trigger when **designing or refactoring DocType form layouts** — section/column breaks, field order, density, child table placement, or moving calculated fields into a Totals block. Use during DocType JSON authoring or Customize Form planning.

## Patterns

### Optimal structure (JSON excerpt)

```json
{
  "fields": [
    {"fieldname": "title_section", "fieldtype": "Section Break", "label": "Basic Info"},
    {"fieldname": "column_break_1", "fieldtype": "Column Break"},
    {"fieldname": "customer", "fieldtype": "Link", "label": "Customer", "options": "Customer"},
    {"fieldname": "customer_name", "fieldtype": "Data", "read_only": 1, "fetch_from": "customer.customer_name"},
    {"fieldname": "posting_date", "fieldtype": "Date"},
    {"fieldname": "column_break_2", "fieldtype": "Column Break"},
    {"fieldname": "company", "fieldtype": "Link", "options": "Company"},
    {"fieldname": "items_section", "fieldtype": "Section Break", "label": "Items"},
    {"fieldname": "items", "fieldtype": "Table", "options": "Sales Invoice Item"},
    {"fieldname": "totals_section", "fieldtype": "Section Break", "label": "Totals", "collapsible": 1},
    {"fieldname": "grand_total", "fieldtype": "Currency", "read_only": 1}
  ]
}
```

### Layout principles

| Principle | Guidance |
|-----------|----------|
| Section Breaks | Group related fields; use clear labels (Basic Info, Items, Totals, System Info) |
| Column Breaks | Two columns max for readability; put paired fields side-by-side (date/time, from/to) |
| Primary info first | Mandatory and high-frequency fields in the first section |
| Calculated fields | Read-only totals, timestamps, audit fields in a collapsible Totals or System section |
| Child tables | Always full-width in their own Section Break — grids need horizontal space |
| Collapsible sections | `collapsible: 1` on advanced/rarely-edited blocks |

### Field order checklist

1. Identity / links (customer, company, naming series)
2. Transaction dates and status
3. Line items (child table)
4. Totals and tax
5. Terms, notes, attachments
6. System / audit (owner, creation — often hidden via Property Setter)

## Gotchas

- **Column Break without Section Break** — first column break after section start creates two columns; a third break starts a new row in some themes — test in Desk.
- **depends_on on section fields** — hiding a Section Break can leave orphan fields; group dependent fields in the same section.
- **List view fields** — `in_list_view` on too many fields clutters lists; keep 4–6 columns max.
- **Mobile Desk** — two-column layouts stack on narrow screens; critical fields must appear before column breaks.

## Version notes (v11–v16)

Form layout JSON is stable across versions. v14+ Workspace and v15+ form sidebar do not replace Section/Column Break semantics.

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-form-layout-optimizer/SKILL.md
