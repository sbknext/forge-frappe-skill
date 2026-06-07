---
name: frappe-doctype-design
description: >
  Design well-structured Frappe DocTypes — field types, naming series, child tables, links,
  Single doctypes, dynamic links, fetch_from, depends_on, and JSON schema. Use when modelling
  data in Frappe/ERPNext, adding fields, building master/child relationships, or choosing a
  naming strategy. Trigger on: "design a DocType", "child table", "naming series", "Single doctype",
  "link field", "fetch_from", "DocType JSON".
---

# Frappe DocType Design

Model data correctly the first time. A DocType is both a DB table and a form; design choices here
ripple through APIs, permissions, and reports.

## Field type cheat sheet

| Need | Fieldtype |
|---|---|
| Short text | `Data` |
| Long text | `Small Text` / `Text` / `Long Text` |
| Rich HTML | `Text Editor` |
| Number | `Int` / `Float` |
| Money | `Currency` (respects company currency) |
| Yes/No | `Check` |
| Fixed choices | `Select` (newline-separated options) |
| Reference to another DocType | `Link` |
| Polymorphic reference | `Dynamic Link` (+ a `Link` to "DocType") |
| One-to-many rows | `Table` (child DocType, `istable: 1`) |
| Computed/read-only mirror | `Data`/`Read Only` + `fetch_from` |
| File/image | `Attach` / `Attach Image` |
| Date / datetime | `Date` / `Datetime` |

## Naming strategies

```text
# In DocType settings → Naming
field:<fieldname>           # name = value of a field
naming_series:              # SO-.YYYY.-  → SO-2026-00001
autoname = "hash"           # random
autoname = "format:PROJ-{####}"
```
Prefer human-meaningful series for documents users reference; `hash` for internal/link-only records.

## Child tables

```python
# Parent doc with child rows
doc = frappe.get_doc({
    "doctype": "Sales Order",
    "customer": "ACME",
    "items": [
        {"item_code": "WIDGET", "qty": 2},
        {"item_code": "GADGET", "qty": 1},
    ],
})
doc.insert()

# Iterate / mutate child rows
for row in doc.get("items"):
    row.amount = row.qty * row.rate
```
Child DocType has `istable: 1`, no standalone permissions — it inherits the parent's.

## fetch_from + depends_on

```json
{ "fieldname": "customer_name", "fieldtype": "Data", "fetch_from": "customer.customer_name", "read_only": 1 }
{ "fieldname": "reason", "fieldtype": "Small Text", "depends_on": "eval:doc.status=='Rejected'" }
{ "fieldname": "approve_btn", "fieldtype": "Button", "mandatory_depends_on": "eval:doc.amount>10000" }
```

## Single DocType (settings)

```python
# istable:0, issingle:1 — exactly one record, ideal for app config
settings = frappe.get_single("My App Settings")
threshold = settings.threshold
```

## DocField visibility and validation (v12+)

Beyond `depends_on` (show/hide), Frappe supports conditional mandatory and read-only:

```json
{ "fieldname": "reason", "depends_on": "eval:doc.status=='Rejected'" }
{ "fieldname": "reason", "mandatory_depends_on": "eval:doc.status=='Rejected'" }
{ "fieldname": "approved_by", "read_only_depends_on": "eval:doc.docstatus==1" }
```

Use `eval:` expressions referencing `doc` — same pattern as client-side `frm.toggle_reqd`, but defined in DocType JSON.

## Form layout (sections & columns)

Group fields with **Section Break** labels (Basic Info, Items, Totals). Use **Column Break** for
two-column density — limit to two columns. Put child tables in a full-width section; put read-only
calculated fields in a collapsible Totals block (`collapsible: 1`). Primary/mandatory fields first.

Deep reference: `skills/frappe/form-layout-optimizer.md`.

## Design rules

- Don't over-Select; if options change at runtime, use a `Link` to a master DocType.
- Add `index` on fields you filter/join on heavily (or `search_index: 1`).
- Keep child tables lean — denormalize only fields the row truly needs.
- Set `track_changes: 1` for audit on important DocTypes.
- Use `Dynamic Link` only when the target DocType genuinely varies; otherwise a plain `Link`.
- Mark mirror fields `read_only` + `fetch_from` instead of copying in code.

## From Frappe docs

- DocType conventions (singular names, `tab` prefix, meta-as-data): https://docs.frappe.io/framework/user/en/basics/doctypes
- DocField properties (`depends_on`, `mandatory_depends_on`, `read_only_depends_on`): https://docs.frappe.io/framework/user/en/basics/doctypes/docfield
- Child table properties (`parent`, `parenttype`, `parentfield`, `idx`): https://docs.frappe.io/framework/user/en/basics/doctypes/child-doctype
- Single DocTypes stored in `tabSingles`: https://docs.frappe.io/framework/user/en/basics/doctypes/single-doctype
- Developer mode + DocType boilerplate export: https://docs.frappe.io/framework/user/en/tutorial/create-a-doctype
