---
title: Frappe Ref — Frappe Syntax: Doctypes: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'doctypes', 'reference', 'anti-patterns']
source: Impertio
---

# DocType Anti-Patterns

Common schema design mistakes and how to avoid them.

## Naming Anti-Patterns

### Using Autoincrement in Production

**Problem:** `naming_rule: "Autoincrement"` creates gaps when records are deleted. Names like `1`, `2`, `5` confuse users and are meaningless.

**Fix:** ALWAYS use Expression (`PRE-.#####`) or Naming Series for production DocTypes.

### Too Few Hash Characters

**Problem:** `autoname: "INV-##"` overflows at 100 records.

**Fix:** ALWAYS use at least 5 hashes: `INV-.#####`. For high-volume DocTypes, use 6-7.

### Field-Based Naming on Non-Unique Fields

**Problem:** `autoname: "field:customer_name"` causes `DuplicateEntryError` when two customers share a name.

**Fix:** ALWAYS set `unique=1` on the source field. Prefer fields with natural uniqueness (codes, IDs).

### Changing Naming Scheme After Data Exists

**Problem:** Switching from `INV-.####` to `SINV-.#####` creates inconsistent names. Old records keep old format.

**Fix:** NEVER change naming scheme after go-live. Plan naming during DocType design.

---

## Field Design Anti-Patterns

### Missing search_index on Filtered Fields

**Problem:** Fields used in `frappe.get_list()` filters or `get_all()` without `search_index=1` cause slow full-table scans.

**Fix:** ALWAYS set `search_index=1` on fields used in filters, especially Link fields with many records.

### Using Data Instead of Link

**Problem:** Storing a reference as plain text (`fieldtype: "Data"`) instead of Link. No referential integrity, no validation, no autocomplete.

**Fix:** ALWAYS use Link fieldtype when referencing another DocType. Use Dynamic Link when the target DocType varies.

### Currency Without options

**Problem:** `fieldtype: "Currency"` without `options` pointing to a currency field. Amount displays in default system currency, even for multi-currency documents.

**Fix:** ALWAYS set `options` on Currency fields to a field containing the currency code:
```json
{
  "fieldname": "amount",
  "fieldtype": "Currency",
  "options": "currency"
}
```

### Overusing allow_on_submit

**Problem:** Setting `allow_on_submit=1` on fields that affect calculations (amounts, quantities) without triggering recalculation.

**Fix:** NEVER use `allow_on_submit` on calculation-critical fields unless the controller recalculates totals in `on_update_after_submit`.

### fetch_from Without fetch_if_empty

**Problem:** `fetch_from: "customer.customer_name"` overwrites user edits every time the Link field changes.

**Fix:** ALWAYS set `fetch_if_empty=1` unless you explicitly want forced overwriting.

### Dynamic Link Without Type Selector

**Problem:** Creating a Dynamic Link field but forgetting the corresponding type-selector field.

**Fix:** ALWAYS create a pair:
```json
[
  {
    "fieldname": "party_type",
    "fieldtype": "Select",
    "options": "\nCustomer\nSupplier"
  },
  {
    "fieldname": "party",
    "fieldtype": "Dynamic Link",
    "options": "party_type"
  }
]
```

### Using depends_on with Raw Python

**Problem:** `depends_on: "doc.status == 'Active'"` -- missing the `eval:` prefix.

**Fix:** ALWAYS use `eval:` prefix: `depends_on: "eval:doc.status == 'Active'"`.

---

## Child Table Anti-Patterns

### Child DocType Without istable=1

**Problem:** Using a regular DocType in a Table field. The relationship breaks; child records have no parent linkage.

**Fix:** ALWAYS set `istable=1` on DocTypes used in Table/Table MultiSelect fields.

### Too Many Fields in Table MultiSelect Child

**Problem:** Using Table MultiSelect with a child DocType that has 10+ editable fields. The UI becomes unusable.

**Fix:** NEVER use Table MultiSelect for complex child records. Use regular Table for editable multi-field rows.

### Missing in_list_view on Child Fields

**Problem:** Child table fields without `in_list_view=1` are hidden from the grid and only visible when expanding a row.

**Fix:** ALWAYS set `in_list_view=1` on the 3-5 most important child table fields.

### Orphaned Child Records

**Problem:** Deleting parent records via raw SQL without cleaning up child tables.

**Fix:** ALWAYS use `frappe.delete_doc()` which handles cascade deletion. NEVER delete parents via `frappe.db.sql("DELETE ...")`.

---

## Structure Anti-Patterns

### Single DocType for Multi-Record Data

**Problem:** Using `issingle=1` when you need multiple records. Single DocTypes store ONE instance in `tabSingles`.

**Fix:** Use `issingle=1` ONLY for app-wide settings/configuration. For multi-record data, use a standard DocType.

### Tree DocType Without nsm_parent_field

**Problem:** Setting `is_tree=1` but not defining `nsm_parent_field`. The NestedSet model cannot build the hierarchy.

**Fix:** ALWAYS set `nsm_parent_field` to the self-referencing Link field name.

### Manual lft/rgt Manipulation

**Problem:** Directly updating `lft`/`rgt` values via SQL. Corrupts the entire tree structure.

**Fix:** NEVER touch `lft`/`rgt` directly. Use `frappe.utils.nestedset.rebuild_tree()` to fix corruption.

### Virtual DocType Using frappe.db

**Problem:** Using `frappe.db.get_list()` or `frappe.db.sql()` inside Virtual DocType methods. These only query the site database, not your custom backend.

**Fix:** ALWAYS implement custom data access in every Virtual DocType method. The `frappe.db.*` API is for site-database-backed DocTypes only.

---

## Permission Anti-Patterns

### No Permissions Defined

**Problem:** Creating a DocType without any permission rules. No one can access it except Administrator.

**Fix:** ALWAYS define at least one permission entry in the DocType JSON.

### Permissions on Child DocType

**Problem:** Adding permission rules to a child DocType (`istable=1`). Child tables inherit permissions from their parent.

**Fix:** NEVER define permissions on child DocTypes. Control access via the parent DocType's permissions.

---

## Performance Anti-Patterns

### Too Many Fields in One DocType

**Problem:** DocTypes with 100+ fields cause slow form loads and large database rows.

**Fix:** Split into parent + child tables, or use separate linked DocTypes. Keep parent DocTypes under 50 data fields.

### Missing Indexes on High-Volume DocTypes

**Problem:** High-volume DocTypes (100k+ records) without `search_index` on filtered fields.

**Fix:** ALWAYS add `search_index=1` to:
- All Link fields used in filters
- Date fields used in date-range queries
- Status/Select fields used in list filters
- Any field in `search_fields`

### Unnecessary track_changes

**Problem:** Enabling `track_changes` on high-volume, frequently-updated DocTypes creates massive `tabVersion` records.

**Fix:** ONLY enable `track_changes` on business-critical documents where audit history is required.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-doctypes/references/anti-patterns.md*
