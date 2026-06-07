---
title: Frappe Ref — Frappe Core: Permissions: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'permissions', 'reference', 'examples']
source: Impertio
---

# Permission Examples

> Working implementation examples for common permission scenarios.

---

## Example 1: Permission Check Before Action

```python
@frappe.whitelist()
def update_order_status(order_name, new_status):
    doc = frappe.get_doc("Sales Order", order_name)
    doc.check_permission("write")  # Raises PermissionError if denied

    doc.status = new_status
    doc.save()
    return {"status": "success"}
```

---

## Example 2: Owner-Only Edit Configuration

```json
{
  "permissions": [
    {
      "role": "Sales User", "permlevel": 0,
      "read": 1, "write": 1, "create": 1, "if_owner": 1
    },
    {
      "role": "Sales Manager", "permlevel": 0,
      "read": 1, "write": 1, "create": 1, "delete": 1, "if_owner": 0
    }
  ]
}
```

- Sales User: read/write/create only their own documents
- Sales Manager: full CRUD on all documents

---

## Example 3: Field-Level Permissions (Perm Levels)

Hide salary from regular users, show to HR only:

```json
{
  "fields": [
    {"fieldname": "employee_name", "permlevel": 0},
    {"fieldname": "department",    "permlevel": 0},
    {"fieldname": "salary",        "permlevel": 1},
    {"fieldname": "bank_account",  "permlevel": 1}
  ],
  "permissions": [
    {"role": "Employee Self Service", "permlevel": 0, "read": 1},
    {"role": "HR Manager", "permlevel": 0, "read": 1, "write": 1},
    {"role": "HR Manager", "permlevel": 1, "read": 1, "write": 1}
  ]
}
```

---

## Example 4: Multi-Company User Restrictions

```python
def setup_user_company_restriction(user, company):
    from frappe.permissions import add_user_permission

    add_user_permission(
        doctype="Company",
        name=company,
        user=user,
        ignore_permissions=True,
        is_default=1
    )

# Bulk setup
for user, company in [("john@ex.com", "Company A"), ("jane@ex.com", "Company B")]:
    setup_user_company_restriction(user, company)
```

---

## Example 5: Custom Permission Hook — Invoice Age Lock

Deny editing invoices older than 30 days for non-accountants:

```python
# hooks.py
has_permission = {
    "Sales Invoice": "myapp.permissions.invoice_permission"
}
```

```python
# myapp/permissions.py
from frappe.utils import date_diff, today

def invoice_permission(doc, ptype, user):
    if ptype not in ("write", "cancel"):
        return None
    if "Accounts Manager" in frappe.get_roles(user):
        return None
    if doc.posting_date and date_diff(today(), doc.posting_date) > 30:
        return False
    return None
```

---

## Example 6: Territory-Based Query Conditions

```python
# hooks.py
permission_query_conditions = {
    "Customer": "myapp.permissions.customer_territory_query"
}
```

```python
def customer_territory_query(user):
    if not user:
        user = frappe.session.user
    if "Sales Manager" in frappe.get_roles(user):
        return ""

    territories = frappe.get_all(
        "User Permission",
        filters={"user": user, "allow": "Territory", "apply_to_all_doctypes": 1},
        pluck="for_value"
    )
    if not territories:
        return ""

    escaped = [frappe.db.escape(t) for t in territories]
    return f"`tabCustomer`.territory IN ({', '.join(escaped)})"
```

---

## Example 7: Document Sharing

```python
from frappe.share import add as add_share, remove as remove_share

def share_for_review(order_name, reviewer_email):
    add_share("Sales Order", order_name, reviewer_email,
              read=1, write=0, share=0, notify=1)

def revoke_access(order_name, reviewer_email):
    remove_share("Sales Order", order_name, reviewer_email)
```

---

## Example 8: Controller Permission Validation

```python
class MyDocType(Document):
    def validate(self):
        if self.has_value_changed("status") and self.status == "Approved":
            if not frappe.has_permission(self.doctype, "write", self):
                frappe.throw(_("No permission to approve"), frappe.PermissionError)
            if self.amount > 50000 and "Finance Manager" not in frappe.get_roles():
                frappe.throw(_("Finance Manager required for amounts > 50,000"))
```

---

## Example 9: Programmatic Role Setup (App Install)

```python
def after_install():
    if not frappe.db.exists("Role", "Custom Reviewer"):
        role = frappe.new_doc("Role")
        role.role_name = "Custom Reviewer"
        role.desk_access = 1
        role.insert(ignore_permissions=True)

    from frappe.permissions import add_permission, update_permission_property

    add_permission("Sales Order", "Custom Reviewer", permlevel=0)
    update_permission_property("Sales Order", "Custom Reviewer", 0, "read", 1)
    update_permission_property("Sales Order", "Custom Reviewer", 0, "report", 1)
    update_permission_property("Sales Order", "Custom Reviewer", 0, "export", 1)
    frappe.clear_cache()
```

---

## Example 10: Role-Restricted API Endpoint

```python
@frappe.whitelist()
def get_confidential_report():
    frappe.only_for(["Sales Manager", "System Manager"])
    return generate_report()

@frappe.whitelist()
def approve_all_pending():
    frappe.only_for(["Approver", "System Manager"])

    pending = frappe.get_all("Sales Order",
                             filters={"status": "Pending Approval"}, pluck="name")
    for name in pending:
        doc = frappe.get_doc("Sales Order", name)
        doc.add_comment("Info", "System: Bulk approved")
        doc.flags.ignore_permissions = True  # System action — bulk approval
        doc.status = "Approved"
        doc.save()
    return {"approved": len(pending)}
```

---

## Example 11: Combined Hook System

```python
# hooks.py
has_permission = {"Project": "myapp.permissions.project_permission"}
permission_query_conditions = {"Project": "myapp.permissions.project_query"}
```

```python
# myapp/permissions.py
def project_permission(doc, ptype, user):
    if not user:
        user = frappe.session.user
    if "Projects Manager" in frappe.get_roles(user):
        return None
    if not frappe.db.exists("Project User", {"parent": doc.name, "user": user}):
        return False
    if ptype in ("read", "write"):
        return None
    return False

def project_query(user):
    if not user:
        user = frappe.session.user
    if "Projects Manager" in frappe.get_roles(user):
        return ""
    return """EXISTS (
        SELECT 1 FROM `tabProject User`
        WHERE `tabProject User`.parent = `tabProject`.name
        AND `tabProject User`.user = {user}
    )""".format(user=frappe.db.escape(user))
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-permissions/references/examples.md*
