---
title: Frappe Ref — Frappe Core: Permissions: Permission Types Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'permissions', 'reference', 'permission-types-reference']
source: Impertio
---

# Permission Types Reference

> Complete reference for all Frappe permission types, options, and levels.

---

## Document-Level Permissions

| Permission | API Check | Description | Applies To |
|------------|-----------|-------------|------------|
| `read` | `frappe.has_permission(dt, "read")` | View document | All DocTypes |
| `write` | `frappe.has_permission(dt, "write")` | Edit/update document | All DocTypes |
| `create` | `frappe.has_permission(dt, "create")` | Create new document | All DocTypes |
| `delete` | `frappe.has_permission(dt, "delete")` | Delete document | All DocTypes |
| `select` | `frappe.has_permission(dt, "select")` | Select in Link field [v14+] | All DocTypes |

## Workflow Permissions (Submittable DocTypes Only)

| Permission | API Check | Description | Prerequisite |
|------------|-----------|-------------|--------------|
| `submit` | `frappe.has_permission(dt, "submit")` | Submit document | `is_submittable = 1` |
| `cancel` | `frappe.has_permission(dt, "cancel")` | Cancel submitted doc | `is_submittable = 1` |
| `amend` | `frappe.has_permission(dt, "amend")` | Amend cancelled doc | `is_submittable = 1` |

## Action Permissions

| Permission | Description | Notes |
|------------|-------------|-------|
| `report` | Access Report Builder for DocType | UI-level only |
| `export` | Export records to Excel/CSV | UI-level only |
| `import` | Import records via Data Import | UI-level only |
| `share` | Share document with other users | UI-level only |
| `print` | Print document or generate PDF | UI-level only |
| `email` | Send email for document | UI-level only |

## Data Masking Permission [v16+]

| Permission | Description | Notes |
|------------|-------------|-------|
| `mask` | View unmasked field values | Only for fields with `mask=1` |

---

## Permission Options

### if_owner

Restricts permission to documents created by the user.

```json
{
  "role": "Sales User",
  "permlevel": 0,
  "read": 1,
  "write": 1,
  "if_owner": 1
}
```

**Effect**: Sales User can only read/write documents they created (owner field matches).

### set_user_permissions

Allows the user to create User Permission records for other users on this DocType.

```json
{
  "role": "Sales Manager",
  "permlevel": 0,
  "set_user_permissions": 1
}
```

---

## Permission Levels (Perm Levels)

### Concept

Group fields into levels (0-9) for separate permission control per role.

### Configuration

```json
// Field definition
{"fieldname": "salary", "fieldtype": "Currency", "permlevel": 1}

// Role permission
{"role": "HR Manager", "permlevel": 1, "read": 1, "write": 1}
```

### Rules

1. Level 0 MUST be granted before higher levels — ALWAYS
2. Levels do NOT imply hierarchy (level 2 is NOT "higher" than level 1)
3. Levels group fields; roles grant access to groups
4. A Section Break with permlevel affects all fields in that section

### Example

```
Field: employee_name    permlevel: 0  → All roles with level 0 read
Field: phone           permlevel: 0  → All roles with level 0 read
Field: salary          permlevel: 1  → Only roles with level 1 read
Field: bank_account    permlevel: 1  → Only roles with level 1 read
Field: performance     permlevel: 2  → Only roles with level 2 read
```

---

## Automatic Roles

| Role | Assigned To | Use Case |
|------|-------------|----------|
| `Guest` | All users (including anonymous) | Public website pages |
| `All` | All registered users (including website users) | Basic authenticated access |
| `Administrator` | Only the Administrator user | Full system control |
| `Desk User` | System Users only [v15+] | Desk/backend access |

These roles are hidden in the Role list and CANNOT be manually assigned or removed.

---

## Custom Permission Types [v16+]

### Creating Custom Permission

1. Enable Developer Mode
2. Create a Permission Type record (e.g., `approve`)
3. Export as fixture for deployment
4. Use in Role Permission Manager

### Checking

```python
if frappe.has_permission("Sales Order", "approve", doc):
    approve_document(doc)
```

---

## Permission Precedence Order

1. **Administrator** — ALWAYS has all permissions (cannot be restricted)
2. **Role Permissions** — Based on assigned roles
3. **User Permissions** — Restricts to specific document values
4. **has_permission hook** — Can only deny (any `False` = denied)
5. **Sharing** — Grants access to individual shared documents
6. **if_owner** — Further restricts to owned documents

---

## Quick Decision Table

| I want to... | Use |
|--------------|-----|
| Control who can create/read/write/delete | Role Permissions |
| Let users only edit their own docs | `if_owner` option |
| Hide salary field from most users | Perm Level 1+ |
| Show masked phone numbers | Data Masking [v16+] |
| Restrict access to specific company | User Permission |
| Add custom "approve" action | Custom Permission Type [v16+] |
| Programmatically deny access | `has_permission` hook |
| Share one doc with one user | `frappe.share.add()` |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-permissions/references/permission-types-reference.md*
