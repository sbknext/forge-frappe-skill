---
title: Frappe Ref — Frappe Core: Notifications: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'notifications', 'reference', 'anti-patterns']
source: Impertio
---

# Notification Anti-Patterns

## AP-1: Using `now=True` in Production

**Wrong:**
```python
frappe.sendmail(
    recipients=["user@example.com"],
    subject="Alert",
    message="<p>Important!</p>",
    now=True,  # BLOCKS the HTTP request until email is sent
)
```

**Correct:**
```python
frappe.sendmail(
    recipients=["user@example.com"],
    subject="Alert",
    message="<p>Important!</p>",
    # delayed=True is the default — emails go via Email Queue
)
```

NEVER use `now=True` in production code. It blocks the current request until the SMTP transaction completes, causing timeouts for users.

---

## AP-2: Missing Email Account Configuration

**Symptom:** Emails silently fail — no error raised, but nothing is delivered.

**Fix:** ALWAYS verify that at least one Email Account with "Enable Outgoing" is configured before deploying notification code. Check via:

```python
has_outgoing = frappe.db.exists("Email Account", {"enable_outgoing": 1})
if not has_outgoing:
    frappe.throw("No outgoing Email Account configured")
```

---

## AP-3: publish_realtime Without after_commit

**Wrong:**
```python
def on_update(doc, method):
    doc.db_set("processed", 1)
    frappe.publish_realtime("doc_updated", {"name": doc.name}, user=doc.owner)
    # Client receives event BEFORE the transaction is committed
    # Client fetches doc — gets OLD data
```

**Correct:**
```python
def on_update(doc, method):
    doc.db_set("processed", 1)
    frappe.publish_realtime(
        "doc_updated", {"name": doc.name},
        user=doc.owner,
        after_commit=True,  # event fires AFTER transaction commits
    )
```

ALWAYS use `after_commit=True` when publishing realtime events inside database transactions.

---

## AP-4: Missing reference_doctype on sendmail

**Wrong:**
```python
frappe.sendmail(
    recipients=[doc.email],
    subject="Update",
    message="<p>Your document was updated.</p>",
    # No reference_doctype/reference_name — email not linked to document
)
```

**Correct:**
```python
frappe.sendmail(
    recipients=[doc.email],
    subject="Update",
    message="<p>Your document was updated.</p>",
    reference_doctype=doc.doctype,
    reference_name=doc.name,
)
```

ALWAYS set `reference_doctype` and `reference_name` — this links the email to the document timeline, making it traceable.

---

## AP-5: Notification Without Condition Guard

**Wrong:** Notification DocType with no Condition set on a high-frequency DocType like Communication or Email Queue.

**Result:** Notification fires on EVERY save, flooding recipients.

**Fix:** ALWAYS set a Condition on Notification DocType records. At minimum, use `doc.docstatus == 0` or a status check.

NEVER create Notifications on Email Queue or Communication DocTypes — this causes infinite notification loops.

---

## AP-6: Closing Assignment as Wrong User

**Wrong:**
```python
# Called as Administrator but trying to close user's assignment
from frappe.desk.form.assign_to import close
close("Task", "TASK-001", "user@example.com")
# Raises: "Only user@example.com can complete this to-do"
```

**Correct:**
```python
# Either run as the assignee:
frappe.set_user("user@example.com")
close("Task", "TASK-001", "user@example.com")
frappe.set_user("Administrator")

# Or cancel instead of close:
from frappe.desk.form.assign_to import remove
remove("Task", "TASK-001", "user@example.com")
```

---

## AP-7: Auto Repeat Without End Date

**Wrong:**
```python
frappe.get_doc({
    "doctype": "Auto Repeat",
    "reference_doctype": "Journal Entry",
    "reference_document": "JV-001",
    "frequency": "Daily",
    "start_date": "2025-01-01",
    # No end_date — creates documents FOREVER
}).insert()
```

**Correct:** ALWAYS set an `end_date` on Auto Repeat records. Open-ended schedules create documents indefinitely and are hard to debug after accumulating thousands of records.

---

## AP-8: Hardcoded Recipients in Notification Templates

**Wrong:** Putting email addresses directly in the Notification message or subject.

**Correct:** Use the Recipients table with Document Field or Role-based resolution. This ensures recipients update when user data changes and respects the permission model.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-notifications/references/anti-patterns.md*
