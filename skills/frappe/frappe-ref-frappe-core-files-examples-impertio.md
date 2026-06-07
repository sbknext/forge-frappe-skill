---
title: Frappe Ref — Frappe Core: Files: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'files', 'reference', 'examples']
source: Impertio
---

# File Management Examples

## 1. Upload and Attach File in Server Script

```python
import frappe
from frappe.utils.file_manager import save_file

def attach_generated_report(doctype, docname):
    """Generate a CSV report and attach it to a document."""
    rows = frappe.get_all("Sales Invoice Item",
        filters={"parent": docname},
        fields=["item_code", "qty", "amount"],
    )

    import csv
    import io

    output = io.StringIO()
    writer = csv.DictWriter(output, fieldnames=["item_code", "qty", "amount"])
    writer.writeheader()
    writer.writerows(rows)

    file_doc = save_file(
        fname=f"{docname}-items.csv",
        content=output.getvalue().encode("utf-8"),
        dt=doctype,
        dn=docname,
        is_private=1,
    )
    return file_doc.file_url
```

## 2. Bulk File Cleanup

```python
def remove_orphaned_files():
    """Delete private files not attached to any document."""
    orphans = frappe.get_all("File",
        filters={
            "is_private": 1,
            "attached_to_doctype": ("is", "not set"),
            "attached_to_name": ("is", "not set"),
            "is_folder": 0,
        },
        fields=["name"],
        limit=100,
    )
    for f in orphans:
        frappe.delete_doc("File", f.name, ignore_permissions=True)
    frappe.db.commit()
```

## 3. Copy Attachment Between Documents

```python
def copy_attachments(source_dt, source_dn, target_dt, target_dn):
    """Copy all attachments from one document to another."""
    files = frappe.get_all("File",
        filters={
            "attached_to_doctype": source_dt,
            "attached_to_name": source_dn,
        },
        fields=["file_name", "file_url", "is_private"],
    )

    for f in files:
        # Create new File doc pointing to same physical file
        new_file = frappe.get_doc({
            "doctype": "File",
            "file_name": f.file_name,
            "file_url": f.file_url,
            "attached_to_doctype": target_dt,
            "attached_to_name": target_dn,
            "is_private": f.is_private,
        }).insert(ignore_permissions=True)
```

## 4. Validate File Extension Before Processing

```python
ALLOWED_EXTENSIONS = {"pdf", "xlsx", "csv", "docx"}

def validate_attachment(doc, method):
    """Reject uploads with disallowed extensions."""
    if doc.doctype != "File":
        return

    if doc.file_type and doc.file_type.lower() not in ALLOWED_EXTENSIONS:
        frappe.throw(
            f"File type '{doc.file_type}' is not allowed. "
            f"Allowed: {', '.join(sorted(ALLOWED_EXTENSIONS))}"
        )
```

Hook in `hooks.py`:
```python
doc_events = {
    "File": {
        "before_insert": "my_app.validators.validate_attachment"
    }
}
```

## 5. Serve Private File via Whitelisted Method

```python
@frappe.whitelist()
def download_report(report_name):
    """Serve a private file with custom permission check."""
    file_doc = frappe.get_doc("File", {"file_name": report_name, "is_private": 1})

    # Custom permission check
    if not frappe.has_permission("Sales Report", "read"):
        frappe.throw("Not permitted", frappe.PermissionError)

    filename, content = frappe.get_file(report_name)
    frappe.response["filename"] = filename
    frappe.response["filecontent"] = content
    frappe.response["type"] = "download"
```

## 6. Image Thumbnail Access

```python
file_doc = frappe.get_doc("File", "FILE-00001")

# Thumbnail is auto-generated for images
if file_doc.thumbnail_url:
    print(f"Thumbnail: {file_doc.thumbnail_url}")
    # e.g., "/files/report_small.jpg"

# Force thumbnail generation
file_doc.make_thumbnail()
```

## 7. Attach File to Email

```python
# Get file content for email attachment
file_doc = frappe.get_doc("File", {"file_name": "contract.pdf"})
content = file_doc.get_content()

frappe.sendmail(
    recipients=["client@example.com"],
    subject="Contract",
    message="<p>Please find the contract attached.</p>",
    attachments=[{
        "fname": file_doc.file_name,
        "fcontent": content,
    }],
    reference_doctype="Contract",
    reference_name="CONTRACT-001",
)
```

## 8. Check File Size Before Save

```python
def before_save(doc, method):
    """Enforce 5MB limit on specific DocType attachments."""
    if doc.doctype != "File" or not doc.attached_to_doctype:
        return

    if doc.attached_to_doctype == "Employee" and doc.file_size > 5 * 1024 * 1024:
        frappe.throw("Employee attachments must be under 5 MB")
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-files/references/examples.md*
