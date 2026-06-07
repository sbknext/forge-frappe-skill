---
title: Frappe Ref — Frappe Errors: Permissions: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'permissions', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns — Permission Error Handling

Common mistakes to avoid. Each entry follows: WRONG -> CORRECT -> WHY.

---

## 1. Throwing in has_permission Hook

```python
# WRONG — Breaks ALL document access
def has_permission(doc, ptype, user):
    if doc.status == "Locked":
        frappe.throw("Document is locked")

# CORRECT — Return False to deny silently
def has_permission(doc, ptype, user):
    try:
        if doc.get("status") == "Locked" and ptype != "read":
            return False
        return None
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Permission Error")
        return None
```

**Why:** `frappe.throw()` inside `has_permission` raises an exception that breaks ALL access to the DocType, including read.

---

## 2. Returning True in has_permission

```python
# WRONG — True has no effect
def has_permission(doc, ptype, user):
    if user == doc.owner:
        return True  # Does nothing!

# CORRECT — Return None to defer to standard system
def has_permission(doc, ptype, user):
    if user == doc.get("owner"):
        return None  # Let standard system handle
    if not meets_criteria(doc, user):
        return False
    return None
```

**Why:** `has_permission` hooks can only **deny** (return False). Returning True is ignored — hooks NEVER grant access.

---

## 3. SQL Injection in permission_query_conditions

```python
# WRONG — SQL injection vulnerability
def query_conditions(user):
    return f"owner = '{user}'"

# CORRECT — ALWAYS escape
def query_conditions(user):
    return f"owner = {frappe.db.escape(user)}"
```

**Why:** Unescaped input allows attackers to bypass all permission controls via SQL injection.

---

## 4. Throwing in permission_query_conditions

```python
# WRONG — Breaks list view for ALL users
def query_conditions(user):
    if not user:
        frappe.throw("User required")

# CORRECT — Handle gracefully
def query_conditions(user):
    try:
        user = user or frappe.session.user
        return f"owner = {frappe.db.escape(user)}"
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Query Error")
        return f"owner = {frappe.db.escape(frappe.session.user)}"
```

**Why:** Exceptions in query conditions crash the entire list view for all users of that DocType.

---

## 5. Using get_all Instead of get_list

```python
# WRONG — Bypasses ALL permission checks
@frappe.whitelist()
def get_user_orders():
    return frappe.get_all("Sales Order", filters={"customer": customer})

# CORRECT — Respects user permissions and permission_query_conditions
@frappe.whitelist()
def get_user_orders():
    return frappe.get_list("Sales Order", filters={"customer": customer})
```

**Why:** `frappe.get_all()` bypasses user permissions AND `permission_query_conditions`. ALWAYS use `get_list()` for user-facing queries.

---

## 6. No try/except in Permission Hooks

```python
# WRONG — Crash breaks all document access
def has_permission(doc, ptype, user):
    territories = frappe.get_all("User Permission", filters={"user": user})
    if doc.territory not in territories:
        return False

# CORRECT — Wrapped with safe fallback
def has_permission(doc, ptype, user):
    try:
        territories = frappe.get_all("User Permission",
            filters={"user": user, "allow": "Territory"}, pluck="for_value") or []
        territory = doc.get("territory")
        if territory and territories and territory not in territories:
            return False
        return None
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Permission Error")
        return None
```

**Why:** Any unhandled exception in a permission hook completely breaks access to that DocType.

---

## 7. Not Checking System Manager in Custom Hook

```python
# WRONG — System Manager blocked by custom logic
def has_permission(doc, ptype, user):
    if doc.get("department") != get_user_department(user):
        return False

# CORRECT — ALWAYS bypass for System Manager / Administrator
def has_permission(doc, ptype, user):
    try:
        user = user or frappe.session.user
        if user == "Administrator":
            return None
        if "System Manager" in frappe.get_roles(user):
            return None
        if doc.get("department") != get_user_department(user):
            return False
        return None
    except Exception:
        return None
```

**Why:** System Managers expect full access. Blocking them causes confusion and support tickets.

---

## 8. Missing Permission Check in API Endpoint

```python
# WRONG — Any logged-in user can update salary
@frappe.whitelist()
def update_salary(employee, new_salary):
    frappe.db.set_value("Employee", employee, "salary", new_salary)

# CORRECT — Check permission first
@frappe.whitelist()
def update_salary(employee, new_salary):
    frappe.has_permission("Employee", "write", employee, throw=True)
    frappe.only_for(["HR Manager"])
    frappe.db.set_value("Employee", employee, "salary", new_salary)
```

**Why:** `@frappe.whitelist()` makes the function callable by ANY logged-in user. Permission checks are your responsibility.

---

## 9. Blocking Read When Only Write Should Be Blocked

```python
# WRONG — Can't even view locked documents
def has_permission(doc, ptype, user):
    if doc.get("status") == "Locked":
        return False  # Blocks read too!

# CORRECT — Only block modifications
def has_permission(doc, ptype, user):
    if doc.get("status") == "Locked":
        if ptype in ("write", "delete", "submit", "cancel"):
            return False
    return None
```

**Why:** Users ALWAYS need to read documents they cannot modify. Blocking read makes locked documents invisible.

---

## 10. Not Handling None Values Safely

```python
# WRONG — Crashes if territory is None
def has_permission(doc, ptype, user):
    if doc.territory not in allowed_territories:
        return False

# CORRECT — Guard against None
def has_permission(doc, ptype, user):
    territory = doc.get("territory") if hasattr(doc, "get") else getattr(doc, "territory", None)
    if territory and allowed_territories and territory not in allowed_territories:
        return False
    return None
```

**Why:** Document fields can be None. Permission hooks receive both Document objects and dicts, requiring safe access.

---

## 11. Using ignore_permissions Without Documentation

```python
# WRONG — No justification for bypassing permissions
def process():
    docs = frappe.get_all("Confidential Doc")
    for doc in docs:
        d = frappe.get_doc("Confidential Doc", doc.name)
        d.flags.ignore_permissions = True
        d.save()

# CORRECT — Document WHY and restrict WHO
def process():
    """System cleanup task — runs as Administrator in scheduler only."""
    if frappe.session.user != "Administrator":
        frappe.throw(_("System function only"))
    docs = frappe.get_all("Doc", filters={"status": "Pending"})
    for doc in docs:
        d = frappe.get_doc("Doc", doc.name)
        d.flags.ignore_permissions = True  # Scheduler task — no user context
        d.save()
```

**Why:** Undocumented `ignore_permissions` creates security audit gaps and makes code reviews harder.

---

## 12. Checking Permissions After Action

```python
# WRONG — Deletes first, checks later
@frappe.whitelist()
def delete_document(name):
    frappe.delete_doc("Important Doc", name)
    if not frappe.has_permission("Important Doc", "delete"):
        frappe.throw("No permission")  # Too late!

# CORRECT — ALWAYS check before action
@frappe.whitelist()
def delete_document(name):
    frappe.has_permission("Important Doc", "delete", name, throw=True)
    frappe.delete_doc("Important Doc", name)
```

**Why:** The action is already performed. Permission checks MUST happen before any state change.

---

## 13. Exposing Sensitive Info in Permission Errors

```python
# WRONG — Leaks internal user list
def has_permission(doc, ptype, user):
    allowed = ["admin@company.com", "ceo@company.com"]
    if user not in allowed:
        frappe.throw(f"Only {allowed} can access")  # Exposes emails!

# CORRECT — Silent deny, no information leak
def has_permission(doc, ptype, user):
    if user not in get_allowed_users():
        return False
    return None
```

**Why:** Error messages can leak internal system configuration to unauthorized users.

---

## 14. Assuming Sharing Replaces Role Permissions

```python
# WRONG — Sharing alone does not grant access
frappe.share.add("Sales Order", "SO-001", "john@example.com", read=1, write=1)
# John still gets PermissionError if he has no role with Sales Order access

# CORRECT — User MUST have base role permission first
# 1. Ensure user has a role with at least read on Sales Order
# 2. Then share for document-level access
frappe.share.add("Sales Order", "SO-001", "john@example.com", read=1, write=1)
```

**Why:** Sharing supplements role permissions — it never replaces them. The user MUST have at least one role with read access to the DocType.

---

## 15. Not Logging Permission Denials

```python
# WRONG — No audit trail
def has_permission(doc, ptype, user):
    if doc.get("is_confidential") and user not in allowed:
        return False  # Silent, no record

# CORRECT — Log for security audit
def has_permission(doc, ptype, user):
    if doc.get("is_confidential") and user not in get_allowed(doc):
        try:
            frappe.log_error(
                f"Access denied: {user} attempted {ptype} on {getattr(doc, 'name', 'unknown')}",
                "Permission Audit")
        except Exception:
            pass  # NEVER break permission check for logging
        return False
    return None
```

**Why:** Security compliance requires logging denied access attempts, especially for sensitive data.

---

## Checklist Before Deploying Permission Code

- [ ] No `frappe.throw()` in has_permission hooks
- [ ] No `frappe.throw()` in permission_query_conditions
- [ ] NEVER return True in has_permission (only False or None)
- [ ] All SQL uses `frappe.db.escape()`
- [ ] All hooks wrapped in try/except with safe fallback
- [ ] `frappe.get_list()` used for user queries (not `get_all`)
- [ ] System Manager / Administrator checked first in custom hooks
- [ ] Permission checks BEFORE actions (not after)
- [ ] `ignore_permissions` usage documented with justification
- [ ] Access denials logged for audit
- [ ] None values handled safely (doc may be dict or object)
- [ ] Read permission preserved when blocking write

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-permissions/references/anti-patterns.md*
