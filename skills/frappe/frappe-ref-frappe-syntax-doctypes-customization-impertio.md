---
title: Frappe Ref — Frappe Syntax: Doctypes: Customization (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'doctypes', 'reference', 'customization']
source: Impertio
---

# Customization API Reference

Programmatic APIs for customizing existing DocTypes without modifying their source JSON.

## Custom Fields

### create_custom_fields() -- Batch Creation

```python
from frappe.custom.doctype.custom_field.custom_field import create_custom_fields

create_custom_fields({
    "Sales Invoice": [
        dict(
            fieldname="custom_tracking_id",
            label="Tracking ID",
            fieldtype="Data",
            insert_after="naming_series",
            reqd=0,
            in_list_view=1
        ),
        dict(
            fieldname="custom_delivery_status",
            label="Delivery Status",
            fieldtype="Select",
            options="Pending\nShipped\nDelivered",
            insert_after="custom_tracking_id",
            default="Pending"
        )
    ],
    "Purchase Order": [
        dict(
            fieldname="custom_vendor_ref",
            label="Vendor Reference",
            fieldtype="Data",
            insert_after="supplier"
        )
    ]
}, ignore_validate=False, update=True)
```

**Signature:**
```python
def create_custom_fields(
    custom_fields: dict,   # {DocType: [field_dicts]}
    ignore_validate=False, # Skip field validation
    update=True            # Update existing fields if they exist
)
```

**Behavior:**
- Skips fields that already exist (no `DuplicateEntryError`).
- When `update=True`, updates existing custom fields with new properties.
- Clears DocType cache and rebuilds database schema after creation.
- Sets field owner to "Administrator".

**Field dict properties (most common):**

| Property | Required | Purpose |
|----------|----------|---------|
| `fieldname` | YES | Internal name (auto-generated from label if omitted) |
| `label` | YES | Display label |
| `fieldtype` | YES | Data type (Data, Link, Select, etc.) |
| `insert_after` | YES | Fieldname to position after |
| `options` | Depends | Target DocType (Link), choices (Select), etc. |
| `reqd` | No | Mandatory (0 or 1) |
| `default` | No | Default value |
| `depends_on` | No | Visibility condition |
| `in_list_view` | No | Show in list view (0 or 1) |
| `in_standard_filter` | No | Show as filter (0 or 1) |
| `read_only` | No | Non-editable (0 or 1) |
| `hidden` | No | Not visible (0 or 1) |
| `fetch_from` | No | Auto-populate source |
| `description` | No | Help text |

### create_custom_field() -- Single Field

```python
from frappe.custom.doctype.custom_field.custom_field import create_custom_field

create_custom_field(
    "Sales Invoice",
    dict(
        fieldname="custom_approval_status",
        label="Approval Status",
        fieldtype="Select",
        options="Pending\nApproved\nRejected",
        insert_after="status"
    ),
    ignore_validate=False,
    is_system_generated=True
)
```

**Signature:**
```python
def create_custom_field(
    doctype: str,              # Target DocType
    df: dict,                  # Field definition
    ignore_validate=False,     # Skip validation
    is_system_generated=True   # Mark as system-generated
)
```

### Tuple Keys for Shared Fields

Apply the same custom fields to multiple DocTypes:

```python
create_custom_fields({
    ("Sales Invoice", "Purchase Invoice"): [
        dict(
            fieldname="custom_external_ref",
            label="External Reference",
            fieldtype="Data",
            insert_after="naming_series"
        )
    ]
})
```

### Custom Fields via Fixtures (Recommended for Apps)

For app-distributed customizations, use the fixtures approach:

**Step 1:** Add fields via Frappe UI (Customize Form).

**Step 2:** Update `hooks.py`:
```python
fixtures = [
    {
        "dt": "Custom Field",
        "filters": [["module", "=", "My App"]]
    }
]
```

**Step 3:** Export:
```bash
bench --site mysite export-fixtures
```

This creates JSON files in your app's `fixtures/` directory, synced on `bench migrate`.

- ALWAYS use fixtures for app-distributed custom fields.
- ALWAYS use `create_custom_fields()` for programmatic setup during `after_install` hooks.

---

## Property Setters

### make_property_setter() -- Modify DocType/Field Properties

```python
from frappe.custom.doctype.property_setter.property_setter import make_property_setter

# Make a field mandatory
make_property_setter(
    "Sales Invoice",    # doctype
    "customer",         # fieldname
    "reqd",             # property
    1,                  # value
    "Check"             # property_type
)

# Change field default
make_property_setter(
    "Sales Invoice",
    "posting_date",
    "default",
    "Today",
    "Text"
)

# Change a DocType-level property (not a field)
make_property_setter(
    "Sales Invoice",
    "",                  # empty string for DocType-level
    "allow_rename",
    1,
    "Check",
    for_doctype=True     # IMPORTANT: set True for DocType properties
)
```

**Signature:**
```python
def make_property_setter(
    doctype: str,                       # Target DocType
    fieldname: str,                     # Field name (empty string for DocType-level)
    property: str,                      # Property to change
    value,                              # New value
    property_type: str,                 # Value data type ("Check", "Text", "Data", etc.)
    for_doctype=False,                  # True = DocType property, False = field property
    validate_fields_for_doctype=True,   # Validate after setting
    is_system_generated=True            # Mark as system-generated
)
```

**Common property_type values:**

| property_type | Use for |
|---------------|---------|
| "Check" | Boolean properties (reqd, hidden, read_only, etc.) |
| "Text" | String properties (default, description, options) |
| "Data" | Short string properties (label, fieldname) |
| "Int" | Integer properties (precision, columns) |
| "Select" | Select properties (fieldtype changes) |
| "Small Text" | Multi-line text properties |

### Common Property Setter Use Cases

```python
# Hide a field
make_property_setter("Sales Invoice", "tax_id", "hidden", 1, "Check")

# Change field label
make_property_setter("Sales Invoice", "customer", "label", "Client", "Data")

# Change select options
make_property_setter("Sales Invoice", "status", "options",
    "Draft\nUnpaid\nPaid\nCancelled", "Text")

# Set field as read-only
make_property_setter("Sales Invoice", "company", "read_only", 1, "Check")

# Change sort order for DocType
make_property_setter("Sales Invoice", "", "sort_field", "posting_date", "Data",
    for_doctype=True)

# Add description/help text
make_property_setter("Sales Invoice", "customer", "description",
    "Select the billing customer", "Small Text")
```

### Property Setter via Fixtures

```python
# hooks.py
fixtures = [
    {
        "dt": "Property Setter",
        "filters": [["module", "=", "My App"]]
    }
]
```

---

## Customize Form API (Web UI)

The Customize Form interface at `/app/customize-form` creates Custom Fields and Property Setters behind the scenes. Programmatic access:

```python
from frappe.custom.doctype.customize_form.customize_form import CustomizeForm

cf = CustomizeForm()
cf.doc_type = "Sales Invoice"
cf.run_method("fetch_to_customize")

# Modify properties
for field in cf.get("fields"):
    if field.fieldname == "customer":
        field.reqd = 0

cf.run_method("save_customization")
```

- NEVER use Customize Form API in production code -- use `create_custom_fields()` and `make_property_setter()` instead.
- Customize Form is designed for interactive (UI) customization.

---

## extend_doctype_class (v16+)

Add methods to existing DocType controllers without replacing them:

```python
# hooks.py
extend_doctype_class = {
    "Sales Invoice": "my_app.overrides.sales_invoice.CustomSalesInvoice"
}
```

```python
# my_app/overrides/sales_invoice.py
from erpnext.accounts.doctype.sales_invoice.sales_invoice import SalesInvoice

class CustomSalesInvoice(SalesInvoice):
    def custom_validation(self):
        # Additional validation logic
        if self.total < 0:
            frappe.throw("Total cannot be negative")
```

- Available in Frappe v16+.
- Multiple apps can extend the same DocType.
- ALWAYS inherit from the original controller class.

---

## Rules for Customization

1. ALWAYS prefix custom fieldnames with `custom_` to avoid conflicts with core fields.
2. ALWAYS use `insert_after` to control field position -- fields without it appear at the end.
3. NEVER modify core DocType JSON files directly -- use Custom Fields and Property Setters.
4. ALWAYS use fixtures for distributing customizations with your app.
5. ALWAYS test customizations with `bench migrate` to ensure they apply cleanly.
6. NEVER use `make_property_setter()` to change `fieldtype` on fields with existing data -- it may cause data loss.
7. Property Setters bypass permissions -- ALWAYS use them in controlled contexts (setup, hooks).

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-doctypes/references/customization.md*
