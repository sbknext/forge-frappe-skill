---
title: Frappe Ref — Frappe Syntax: Serverscripts: Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'serverscripts', 'reference', 'methods']
source: Impertio
---

# Server Script Sandbox — Complete Method Reference

All methods available inside the RestrictedPython sandbox. NEVER use import
statements — everything listed here is pre-loaded.

---

## doc Object (Document Event Scripts Only)

### Properties

```python
doc.name              # str — Document ID (NOT available in Before Insert)
doc.doctype           # str — DocType name
doc.docstatus         # int — 0=Draft, 1=Submitted, 2=Cancelled
doc.owner             # str — Creator email
doc.modified_by       # str — Last modifier email
doc.creation          # datetime — Creation timestamp
doc.modified          # datetime — Last modification timestamp
doc.flags             # _dict — Transient flags (not persisted)

# Every DocType field is a direct attribute:
doc.customer          # Link field value
doc.grand_total       # Currency field value
doc.items             # Child table (list of child doc objects)
```

### Methods

```python
doc.get("fieldname")                    # Safe access — returns None if missing
doc.get("fieldname", "default")         # With default value
doc.update({"field1": val1, ...})       # Set multiple fields at once
doc.append("child_table", {...})        # Add row to child table
doc.as_dict()                           # Convert to dictionary
doc.db_set("field", value)              # Direct DB update (After Save only)
doc.add_comment("Info", "text")         # Add comment to document
doc.add_tag("tag_name")                 # Add tag
doc.get_tags()                          # Get tags list

# Child table iteration
for item in doc.items:
    item.item_code      # Child field
    item.qty            # Child field
    item.idx            # Row number (1-based)
    item.parent         # Parent document name
    item.parenttype     # Parent DocType name
```

---

## frappe.db — Database Operations

### Single Value

```python
# Get one field from one document
val = frappe.db.get_value("Customer", "CUST-001", "customer_name")

# Get multiple fields as dict
vals = frappe.db.get_value("Customer", "CUST-001",
    ["customer_name", "territory"], as_dict=True)

# Get value with filters (returns first match)
email = frappe.db.get_value("User", {"first_name": "John"}, "email")

# Get value from Singles DocType
company = frappe.db.get_single_value("Global Defaults", "default_company")

# Get default value
default = frappe.db.get_default("currency")
```

### Set Value

```python
# Single field
frappe.db.set_value("Customer", "CUST-001", "status", "Active")

# Multiple fields
frappe.db.set_value("Customer", "CUST-001", {
    "status": "Active",
    "last_contact": frappe.utils.today()
})
# WARNING: set_value bypasses validate hooks — use only for simple updates
```

### Multiple Records

```python
# get_all — NO permission filtering
orders = frappe.get_all("Sales Order",
    filters={"customer": "CUST-001", "docstatus": 1},
    fields=["name", "grand_total", "status"],
    order_by="creation desc",
    limit=20
)
# Returns: [{"name": "SO-001", "grand_total": 5000, "status": "Submitted"}, ...]

# get_list — WITH permission filtering (respects user permissions)
orders = frappe.db.get_list("Sales Order",
    filters={"docstatus": 1},
    fields=["name", "grand_total"],
    limit=20
)

# Filter operators
filters = {
    "grand_total": [">", 1000],
    "status": ["in", ["Open", "Active"]],
    "due_date": ["<", frappe.utils.today()],
    "name": ["like", "SO-%"],
    "customer": ["is", "set"],           # IS NOT NULL
    "note": ["is", "not set"],           # IS NULL
    "creation": ["between", [start, end]]
}

# pluck — returns flat list of one field
names = frappe.get_all("Customer", filters={...}, pluck="name")
# Returns: ["CUST-001", "CUST-002", ...]
```

### Count / Exists

```python
count = frappe.db.count("Sales Invoice",
    filters={"status": "Unpaid", "docstatus": 1})

if frappe.db.exists("Customer", "CUST-001"):
    pass  # Document exists

if frappe.db.exists("Sales Order", {"customer": doc.customer, "docstatus": 0}):
    pass  # Matching document exists
```

### Raw SQL

```python
# ALWAYS use parameterized queries
results = frappe.db.sql("""
    SELECT name, grand_total
    FROM `tabSales Invoice`
    WHERE customer = %(customer)s AND docstatus = 1
""", {"customer": doc.customer}, as_dict=True)

# NEVER use f-strings or string concatenation in SQL:
# frappe.db.sql(f"... WHERE name = '{user_input}'")  ← SQL INJECTION
```

### Query Builder (frappe.qb)

```python
SI = frappe.qb.DocType("Sales Invoice")
query = (
    frappe.qb.from_(SI)
    .select(SI.name, SI.grand_total)
    .where(SI.customer == "CUST-001")
    .where(SI.docstatus == 1)
    .orderby(SI.creation, order=frappe.qb.desc)
    .limit(10)
)
results = query.run(as_dict=True)
```

### Transaction Control

```python
frappe.db.commit()    # ONLY in Scheduler scripts — NEVER in Document Events
frappe.db.rollback()  # ONLY in Scheduler scripts — NEVER in Document Events
frappe.db.escape(val) # Escape value for SQL WHERE clause
```

---

## frappe Document Methods

```python
# Fetch existing document
customer = frappe.get_doc("Customer", "CUST-001")
customer.customer_name   # Read field
customer.save()          # Save changes (triggers hooks)

# Cached fetch (faster, read-only — NEVER modify and save)
customer = frappe.get_cached_doc("Customer", "CUST-001")

# Create new document (method 1)
todo = frappe.get_doc({
    "doctype": "ToDo",
    "description": "Follow up",
    "reference_type": doc.doctype,
    "reference_name": doc.name
})
todo.insert(ignore_permissions=True)

# Create new document (method 2)
todo = frappe.new_doc("ToDo")
todo.description = "Follow up"
todo.insert()

# Get most recent document
last = frappe.get_last_doc("Sales Order",
    filters={"customer": doc.customer})

# Delete document
frappe.delete_doc("ToDo", "TODO-001")

# Rename document
frappe.rename_doc("Customer", "Old Name", "New Name")

# Get DocType metadata
meta = frappe.get_meta("Sales Invoice")
meta.get_field("grand_total")  # Field metadata
```

---

## frappe.utils — Utilities

### Date / Time

```python
frappe.utils.today()                    # "2024-01-15"
frappe.utils.nowdate()                  # Same as today()
frappe.utils.now()                      # "2024-01-15 10:30:00"
frappe.utils.now_datetime()             # datetime object
frappe.utils.nowtime()                  # "10:30:00"

frappe.utils.add_days(date, 7)          # +7 days
frappe.utils.add_months(date, 1)        # +1 month
frappe.utils.add_years(date, 1)         # +1 year
frappe.utils.date_diff(date1, date2)    # Days between (int)
frappe.utils.get_first_day(date)        # First day of month
frappe.utils.get_last_day(date)         # Last day of month
frappe.utils.getdate(string)            # String → date object
frappe.utils.get_datetime(string)       # String → datetime object

frappe.utils.formatdate(date, "dd-MM-yyyy")
frappe.utils.format_datetime(datetime)
frappe.format_date(date)                # Human-readable date
frappe.date_format                      # System date format string
```

### Number / String

```python
frappe.utils.flt(value)                 # → float (None → 0.0)
frappe.utils.flt(value, precision=2)    # With decimal precision
frappe.utils.cint(value)                # → int (None → 0)
frappe.utils.cstr(value)                # → str (None → "")
frappe.utils.rounded(123.456, 2)        # → 123.46
frappe.utils.fmt_money(1234.56, currency="EUR")

frappe.utils.strip_html(html)           # Remove HTML tags
frappe.utils.escape_html(text)          # Escape HTML entities
frappe.utils.random_string(8)           # Random alphanumeric string
frappe.utils.get_url()                  # Site base URL
frappe.utils.get_fullname(user)         # User's full name
```

### JSON (Instead of import json)

```python
data = frappe.parse_json(json_string)   # JSON string → Python dict/list
text = frappe.as_json(python_obj)       # Python dict/list → JSON string
```

---

## Messaging / Errors

```python
# Stop execution and show error to user
frappe.throw("Amount cannot be negative")
frappe.throw("Access denied", frappe.PermissionError)
frappe.throw("Invalid amount", title="Validation Error")

# Show notification (does NOT stop execution)
frappe.msgprint("Record updated successfully")
frappe.msgprint(msg="Created", title="Success", indicator="green")

# Log to Error Log list (background — no user notification)
frappe.log_error(message="Details here", title="Sync Failed")
frappe.log_error(frappe.get_traceback(), "Unhandled Error")
```

---

## HTTP Requests (Available in Sandbox)

```python
# GET request
response = frappe.make_get_request(
    "https://api.example.com/data",
    params={"key": "value"},
    headers={"Authorization": "Bearer token"}
)

# POST request
response = frappe.make_post_request(
    "https://api.example.com/submit",
    data={"field": "value"},
    headers={"Content-Type": "application/json"}
)

# PUT request
response = frappe.make_put_request(
    "https://api.example.com/update/1",
    data={"field": "new_value"}
)
```

---

## Email

```python
frappe.sendmail(
    recipients=["user@example.com"],
    sender="noreply@example.com",
    subject="Invoice Overdue",
    message="Your invoice SI-001 is overdue."
)
```

---

## Session / Permissions

```python
frappe.session.user                     # "user@example.com" or "Guest"
frappe.session.csrf_token               # CSRF token
frappe.user                             # Same as frappe.session.user
frappe.full_name                        # Current user's full name

frappe.get_roles()                      # Current user's roles
frappe.get_roles("user@email.com")      # Specific user's roles
frappe.get_fullname()                   # Current user's full name
frappe.get_fullname("user@email.com")   # Specific user's full name
frappe.get_gravatar()                   # User avatar URL

frappe.has_permission("Sales Invoice", "read")
frappe.has_permission("Sales Invoice", "write", "SINV-001")
# Permission types: read, write, create, delete, submit, cancel, amend
```

---

## Miscellaneous

```python
# Translation
_("Translatable string")

# Template rendering
html = frappe.render_template("Hello {{ name }}", {"name": "World"})

# Call another Server Script as a library (v13+)
result = run_script("My Library Script", arg1="value1")

# Read hooks from apps
hooks = frappe.get_hooks("doc_events")

# Format value by field type
frappe.format_value(1234.5, {"fieldtype": "Currency"})

# System settings
settings = frappe.get_system_settings()
```

---

## What is BLOCKED

| Category | Examples | Error |
|---|---|---|
| All imports | `import json`, `from datetime import date` | `ImportError: __import__ not found` |
| File I/O | `open()`, `file()` | `NameError` |
| Code execution | `eval()`, `exec()`, `compile()` | Blocked by sandbox |
| OS access | `os.system()`, `subprocess.run()` | Not available |
| Scope introspection | `globals()`, `locals()`, `vars()` | Blocked |
| Module access | `__import__`, `importlib` | Blocked |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-serverscripts/references/methods.md*
