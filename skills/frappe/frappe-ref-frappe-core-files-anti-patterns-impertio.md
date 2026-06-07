---
title: Frappe Ref — Frappe Core: Files: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'files', 'reference', 'anti-patterns']
source: Impertio
---

# File Management Anti-Patterns

## AP-1: Storing Sensitive Files as Public

**Wrong:**
```python
save_file("payslip.pdf", content, "Salary Slip", "SAL-001", is_private=0)
# File accessible at /files/payslip.pdf — NO authentication required
```

**Correct:**
```python
save_file("payslip.pdf", content, "Salary Slip", "SAL-001", is_private=1)
# File at /private/files/payslip.pdf — requires authentication + permission
```

NEVER store sensitive documents (payslips, contracts, ID copies, financial records) as public files. ALWAYS use `is_private=1`.

---

## AP-2: Path Traversal in File Operations

**Wrong:**
```python
# User-supplied filename — potential path traversal attack
file_path = f"/home/frappe/site/private/files/{user_input}"
with open(file_path, "rb") as f:
    content = f.read()
```

**Correct:**
```python
# Use Frappe's safe file resolution
from frappe.utils.file_manager import get_file_path
file_path = get_file_path(user_input)  # validates against "../" traversal
```

ALWAYS use `get_file_path()` or `frappe.get_file()` to resolve file paths. NEVER construct file paths from user input with string concatenation.

---

## AP-3: Not Setting attached_to Fields

**Wrong:**
```python
frappe.get_doc({
    "doctype": "File",
    "file_name": "report.pdf",
    "content": pdf_bytes,
    "is_private": 1,
    # Missing attached_to_doctype and attached_to_name
}).insert()
# File is orphaned — no document link, no inherited permissions
```

**Correct:**
```python
frappe.get_doc({
    "doctype": "File",
    "file_name": "report.pdf",
    "content": pdf_bytes,
    "is_private": 1,
    "attached_to_doctype": "Sales Invoice",
    "attached_to_name": "SINV-00001",
}).insert()
```

ALWAYS set `attached_to_doctype` and `attached_to_name` for private files. Without them, the file has no permission inheritance and becomes an orphaned record.

---

## AP-4: Ignoring Max File Size

**Wrong:**
```python
# Accepting arbitrary file sizes without validation
file_doc = save_file("huge-file.zip", large_content, dt, dn)
# May exhaust disk space or cause timeout
```

**Correct:**
```python
from frappe.utils.file_manager import check_max_file_size, save_file

check_max_file_size(large_content)  # raises if exceeds limit
file_doc = save_file("data.zip", large_content, dt, dn)
```

ALWAYS call `check_max_file_size()` before saving programmatically generated files that could exceed the configured limit.

---

## AP-5: Deleting File Doc Without Filesystem Cleanup

**Wrong:**
```python
# Direct database deletion — leaves file on disk
frappe.db.delete("File", {"name": file_name})
```

**Correct:**
```python
# Use delete_doc — triggers on_trash which cleans up filesystem
frappe.delete_doc("File", file_name, ignore_permissions=True)
```

ALWAYS use `frappe.delete_doc()` to delete File records. Direct database deletion leaves orphaned files on the filesystem.

---

## AP-6: Hardcoding File Paths

**Wrong:**
```python
with open("/home/frappe/frappe-bench/sites/mysite/private/files/report.pdf", "rb") as f:
    content = f.read()
```

**Correct:**
```python
# Use Frappe's site resolution
import os
site_path = frappe.get_site_path("private", "files", "report.pdf")
with open(site_path, "rb") as f:
    content = f.read()

# Or better — use the File API
filename, content = frappe.get_file("report.pdf")
```

NEVER hardcode bench or site paths. ALWAYS use `frappe.get_site_path()` or the File API.

---

## AP-7: Not Handling Duplicate Files

**Symptom:** Multiple File records pointing to the same physical file (same `content_hash`).

Frappe deduplicates automatically via `content_hash` — multiple File docs can reference the same physical file. This is by design. However:

- NEVER delete the physical file manually — other File docs may reference it
- ALWAYS delete via `frappe.delete_doc("File", ...)` — Frappe checks for other references before removing the physical file

---

## AP-8: Using Attach Field Without Private Flag

**Wrong:** Using `Attach` field type for sensitive documents without setting `is_private` in the upload handler. By default, uploaded files may be public depending on the upload context.

**Correct:** Set `is_private=1` in the file upload call or use a `before_insert` hook on File to enforce privacy for specific DocTypes:

```python
def enforce_private_files(doc, method):
    """Force all Employee attachments to be private."""
    if doc.attached_to_doctype == "Employee":
        doc.is_private = 1
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-files/references/anti-patterns.md*
