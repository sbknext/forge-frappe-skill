---
title: Frappe Ref — Frappe Syntax: Doctypes: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'doctypes', 'reference', 'examples']
source: Impertio
---

# DocType JSON Examples

Real-world examples of DocType JSON definitions for all DocType types.

## Standard DocType (Multi-Record)

```json
{
  "name": "Project Task",
  "module": "Projects",
  "naming_rule": "Expression",
  "autoname": "PTASK-.#####",
  "title_field": "subject",
  "search_fields": "subject, project",
  "show_title_field_in_link": 1,
  "is_submittable": 0,
  "track_changes": 1,
  "allow_rename": 0,
  "allow_import": 1,
  "sort_field": "modified",
  "sort_order": "DESC",
  "fields": [
    {
      "fieldname": "subject",
      "fieldtype": "Data",
      "label": "Subject",
      "reqd": 1,
      "in_list_view": 1,
      "in_standard_filter": 1,
      "search_index": 1
    },
    {
      "fieldname": "project",
      "fieldtype": "Link",
      "label": "Project",
      "options": "Project",
      "reqd": 1,
      "in_list_view": 1,
      "in_standard_filter": 1,
      "search_index": 1
    },
    {
      "fieldname": "status",
      "fieldtype": "Select",
      "label": "Status",
      "options": "Open\nWorking\nCompleted\nCancelled",
      "default": "Open",
      "in_list_view": 1,
      "in_standard_filter": 1
    },
    {
      "fieldname": "column_break_1",
      "fieldtype": "Column Break"
    },
    {
      "fieldname": "assigned_to",
      "fieldtype": "Link",
      "label": "Assigned To",
      "options": "User",
      "in_standard_filter": 1
    },
    {
      "fieldname": "due_date",
      "fieldtype": "Date",
      "label": "Due Date"
    },
    {
      "fieldname": "section_details",
      "fieldtype": "Section Break",
      "label": "Details"
    },
    {
      "fieldname": "description",
      "fieldtype": "Text Editor",
      "label": "Description"
    }
  ],
  "permissions": [
    {
      "role": "Projects User",
      "read": 1,
      "write": 1,
      "create": 1,
      "delete": 1
    },
    {
      "role": "Projects Manager",
      "read": 1,
      "write": 1,
      "create": 1,
      "delete": 1,
      "export": 1,
      "import": 1
    }
  ]
}
```

## Submittable DocType

```json
{
  "name": "Payment Entry",
  "module": "Accounts",
  "naming_rule": "By \"Naming Series\" field",
  "autoname": "naming_series:",
  "is_submittable": 1,
  "title_field": "title",
  "search_fields": "party, paid_amount, payment_type",
  "track_changes": 1,
  "fields": [
    {
      "fieldname": "naming_series",
      "fieldtype": "Select",
      "label": "Series",
      "options": "PAY-.YYYY.-.#####\nREC-.YYYY.-.#####",
      "reqd": 1,
      "default": "PAY-.YYYY.-.#####"
    },
    {
      "fieldname": "payment_type",
      "fieldtype": "Select",
      "label": "Payment Type",
      "options": "Receive\nPay\nInternal Transfer",
      "reqd": 1,
      "in_standard_filter": 1
    },
    {
      "fieldname": "posting_date",
      "fieldtype": "Date",
      "label": "Posting Date",
      "reqd": 1,
      "default": "Today"
    },
    {
      "fieldname": "section_party",
      "fieldtype": "Section Break",
      "label": "Party"
    },
    {
      "fieldname": "party_type",
      "fieldtype": "Select",
      "label": "Party Type",
      "options": "\nCustomer\nSupplier\nEmployee",
      "in_standard_filter": 1
    },
    {
      "fieldname": "party",
      "fieldtype": "Dynamic Link",
      "label": "Party",
      "options": "party_type",
      "in_standard_filter": 1,
      "search_index": 1
    },
    {
      "fieldname": "party_name",
      "fieldtype": "Data",
      "label": "Party Name",
      "fetch_from": "party.name",
      "read_only": 1
    },
    {
      "fieldname": "column_break_party",
      "fieldtype": "Column Break"
    },
    {
      "fieldname": "paid_amount",
      "fieldtype": "Currency",
      "label": "Paid Amount",
      "options": "paid_currency",
      "reqd": 1,
      "in_list_view": 1
    },
    {
      "fieldname": "paid_currency",
      "fieldtype": "Link",
      "label": "Currency",
      "options": "Currency",
      "reqd": 1
    },
    {
      "fieldname": "section_references",
      "fieldtype": "Section Break",
      "label": "References"
    },
    {
      "fieldname": "references",
      "fieldtype": "Table",
      "label": "Payment References",
      "options": "Payment Entry Reference",
      "allow_on_submit": 1
    },
    {
      "fieldname": "amended_from",
      "fieldtype": "Link",
      "label": "Amended From",
      "options": "Payment Entry",
      "read_only": 1,
      "no_copy": 1
    }
  ]
}
```

**Key points for submittable DocTypes:**
- ALWAYS include `amended_from` field (Link to self, read_only, no_copy).
- Use `allow_on_submit=1` on fields that should be editable after submission.
- The `docstatus` field is auto-managed: 0=Draft, 1=Submitted, 2=Cancelled.

## Child DocType (istable=1)

```json
{
  "name": "Payment Entry Reference",
  "module": "Accounts",
  "istable": 1,
  "fields": [
    {
      "fieldname": "reference_doctype",
      "fieldtype": "Link",
      "label": "Type",
      "options": "DocType",
      "reqd": 1,
      "in_list_view": 1
    },
    {
      "fieldname": "reference_name",
      "fieldtype": "Dynamic Link",
      "label": "Name",
      "options": "reference_doctype",
      "reqd": 1,
      "in_list_view": 1
    },
    {
      "fieldname": "allocated_amount",
      "fieldtype": "Currency",
      "label": "Allocated",
      "reqd": 1,
      "in_list_view": 1,
      "columns": 2
    }
  ]
}
```

**Key points for child DocTypes:**
- ALWAYS set `istable=1`.
- NEVER add naming_rule -- child docs use hash naming.
- NEVER add permissions -- they inherit from the parent.
- Set `in_list_view=1` on fields to show in the grid.
- Use `columns` to control grid column width (1-10).

## Single DocType (Settings)

```json
{
  "name": "Notification Settings",
  "module": "Core",
  "issingle": 1,
  "fields": [
    {
      "fieldname": "enable_email",
      "fieldtype": "Check",
      "label": "Enable Email Notifications",
      "default": 1
    },
    {
      "fieldname": "sender_email",
      "fieldtype": "Data",
      "label": "Sender Email",
      "options": "Email",
      "depends_on": "eval:doc.enable_email",
      "mandatory_depends_on": "eval:doc.enable_email"
    },
    {
      "fieldname": "section_defaults",
      "fieldtype": "Section Break",
      "label": "Defaults",
      "collapsible": 1
    },
    {
      "fieldname": "default_currency",
      "fieldtype": "Link",
      "label": "Default Currency",
      "options": "Currency"
    }
  ],
  "permissions": [
    {
      "role": "System Manager",
      "read": 1,
      "write": 1,
      "create": 1
    }
  ]
}
```

**Key points for Single DocTypes:**
- ALWAYS set `issingle=1`.
- Data stored in `tabSingles` as key-value rows, NOT a dedicated table.
- No list view -- only a form view at `/app/{doctype-slug}`.
- ALWAYS restrict permissions to admin roles (System Manager).

## Tree DocType

```json
{
  "name": "Department",
  "module": "HR",
  "is_tree": 1,
  "nsm_parent_field": "parent_department",
  "naming_rule": "Set by User",
  "allow_rename": 1,
  "title_field": "name",
  "fields": [
    {
      "fieldname": "parent_department",
      "fieldtype": "Link",
      "label": "Parent Department",
      "options": "Department",
      "in_standard_filter": 1
    },
    {
      "fieldname": "is_group",
      "fieldtype": "Check",
      "label": "Is Group",
      "default": 0
    },
    {
      "fieldname": "company",
      "fieldtype": "Link",
      "label": "Company",
      "options": "Company",
      "reqd": 1,
      "in_standard_filter": 1
    },
    {
      "fieldname": "lft",
      "fieldtype": "Int",
      "label": "Left",
      "hidden": 1,
      "search_index": 1
    },
    {
      "fieldname": "rgt",
      "fieldtype": "Int",
      "label": "Right",
      "hidden": 1,
      "search_index": 1
    },
    {
      "fieldname": "old_parent",
      "fieldtype": "Data",
      "label": "Old Parent",
      "hidden": 1
    }
  ]
}
```

**Key points for Tree DocTypes:**
- ALWAYS set `is_tree=1`.
- ALWAYS define `nsm_parent_field` pointing to the self-referencing Link field.
- The `lft`, `rgt`, `old_parent` fields are auto-managed -- include them hidden.
- `is_group` field distinguishes leaf nodes from group nodes.
- NEVER manually modify `lft`/`rgt` values.

## Virtual DocType

```json
{
  "name": "External API Record",
  "module": "Integrations",
  "is_virtual": 1,
  "fields": [
    {
      "fieldname": "external_id",
      "fieldtype": "Data",
      "label": "External ID",
      "in_list_view": 1,
      "reqd": 1
    },
    {
      "fieldname": "title",
      "fieldtype": "Data",
      "label": "Title",
      "in_list_view": 1
    },
    {
      "fieldname": "status",
      "fieldtype": "Select",
      "label": "Status",
      "options": "Active\nInactive",
      "in_list_view": 1,
      "in_standard_filter": 1
    },
    {
      "fieldname": "payload",
      "fieldtype": "JSON",
      "label": "Raw Data"
    }
  ]
}
```

Corresponding controller (REQUIRED):

```python
# external_api_record.py
import frappe
from frappe.model.document import Document

class ExternalAPIRecord(Document):
    @staticmethod
    def get_list(args):
        # Fetch from external API
        response = call_external_api("/records", params=args)
        return [
            frappe._dict(
                name=r["id"],
                external_id=r["id"],
                title=r["title"],
                status=r["status"]
            )
            for r in response["data"]
        ]

    @staticmethod
    def get_count(args):
        response = call_external_api("/records/count", params=args)
        return response["count"]

    @staticmethod
    def get_stats(args):
        return {}

    def db_insert(self, *args, **kwargs):
        call_external_api("/records", method="POST", data=self.as_dict())

    def load_from_db(self):
        response = call_external_api(f"/records/{self.name}")
        for key, value in response.items():
            self.set(key, value)

    def db_update(self, *args, **kwargs):
        call_external_api(f"/records/{self.name}", method="PUT", data=self.as_dict())

    def delete(self):
        call_external_api(f"/records/{self.name}", method="DELETE")
```

**Key points for Virtual DocTypes:**
- ALWAYS implement ALL 7 methods (get_list, get_count, get_stats, db_insert, load_from_db, db_update, delete).
- NEVER use `frappe.db.*` for Virtual DocType data queries.
- The `/api/resource` endpoints work automatically with Virtual DocTypes.

## Table MultiSelect Example

Child DocType:
```json
{
  "name": "Project User",
  "module": "Projects",
  "istable": 1,
  "fields": [
    {
      "fieldname": "user",
      "fieldtype": "Link",
      "label": "User",
      "options": "User",
      "in_list_view": 1,
      "reqd": 1
    }
  ]
}
```

Parent field:
```json
{
  "fieldname": "users",
  "fieldtype": "Table MultiSelect",
  "label": "Project Members",
  "options": "Project User"
}
```

- The child DocType for Table MultiSelect ALWAYS has exactly one Link field.
- UI renders as a tag/pill selector instead of a grid.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-doctypes/references/examples.md*
