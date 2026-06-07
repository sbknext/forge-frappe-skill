---
title: Frappe Ref — Frappe Core: Notifications: Api Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'notifications', 'reference', 'api-reference']
source: Impertio
---

# Notification API Reference

## frappe.sendmail()

```python
frappe.sendmail(
    recipients: list[str],           # required — list of email addresses
    sender: str = None,              # override default sender
    subject: str = "",               # email subject line
    message: str = "",               # HTML body content
    template: str = None,            # Jinja template name (from Email Template DocType)
    args: dict = None,               # context variables for template rendering
    attachments: list[dict] = None,  # [{"fname": "file.pdf", "fcontent": bytes}]
    reference_doctype: str = None,   # link email to this DocType
    reference_name: str = None,      # link email to this document
    cc: list[str] = None,            # carbon copy recipients
    bcc: list[str] = None,           # blind carbon copy recipients
    reply_to: str = None,            # reply-to address
    expose_recipients: str = None,   # "header" to show recipients
    delayed: bool = True,            # True = queue via Email Queue (default)
    now: bool = False,               # True = send immediately (NEVER in production)
    unsubscribe_message: str = None, # custom unsubscribe text
    inline_images: list = None,      # inline image attachments
    header: list = None,             # [title, indicator_color] for email header
)
```

## frappe.publish_realtime()

```python
frappe.publish_realtime(
    event: str = None,          # event name (e.g., "msgprint", "progress", custom)
    message: dict = None,       # JSON-serializable payload
    room: str = None,           # explicit room name
    user: str = None,           # target user email → room "user:{email}"
    doctype: str = None,        # target doctype → room "doctype:{dt}"
    docname: str = None,        # target document → room "doc:{dt}/{dn}"
    task_id: str = None,        # task progress tracking
    after_commit: bool = False, # True = emit after DB transaction commits
)
```

### Room Resolution Priority
1. If `task_id` → room = `task_progress:{task_id}`
2. If `user` → room = `user:{user}`
3. If `doctype` + `docname` → room = `doc:{doctype}/{docname}`
4. If `doctype` only → room = `doctype:{doctype}`
5. Else → room = `all` (site-wide)

## frappe.desk.form.assign_to

```python
from frappe.desk.form.assign_to import add, remove, close, clear, get

# Create assignment (creates ToDo)
add(args={
    "assign_to": list[str],        # required — user emails
    "doctype": str,                # required — reference DocType
    "name": str,                   # required — reference document name
    "description": str = "",       # task description
    "priority": str = "Medium",    # Low, Medium, High
    "date": str = None,            # due date (YYYY-MM-DD)
    "assignment_rule": str = None, # linked Assignment Rule name
}, ignore_permissions=False)

# Get all active assignments for a document
get(args={"doctype": str, "name": str})
# Returns: [{"owner": "user@example.com", "name": "ToDo-ID"}, ...]

# Cancel specific assignment
remove(doctype: str, name: str, assign_to: str, ignore_permissions=False)

# Close assignment (ONLY callable by the assignee)
close(doctype: str, name: str, assign_to: str, ignore_permissions=False)

# Cancel all assignments for a document
clear(doctype: str, name: str, ignore_permissions=False)
```

## Document Methods

```python
# Comments
doc.add_comment(
    comment_type: str,  # "Comment", "Edit", "Shared", "Like", etc.
    text: str = None,   # comment content
)

# Tags
doc.add_tag(tag: str)       # add tag to document
doc.get_tags() -> list      # get all tags on document

# Realtime update notification
doc.notify_update()         # publish socket.io event for document change
```

## Notification DocType Fields

| Field | Type | Description |
|-------|------|-------------|
| name | Data | Unique notification name |
| document_type | Link → DocType | Target DocType |
| event | Select | Trigger event type |
| channel | Select | Email, Slack, System Notification, SMS |
| condition | Code | Python expression |
| subject | Data | Jinja-enabled subject line |
| message | Text Editor | Jinja-enabled HTML body |
| attach_print | Check | Attach PDF of document |
| print_format | Link → Print Format | Custom print format |
| set_property_after_alert | Table | Update doc fields after sending |

## ToDo DocType Fields

| Field | Type | Description |
|-------|------|-------------|
| allocated_to | Link → User | Assigned user |
| assigned_by | Link → User | Assigning user |
| status | Select | Open, Closed, Cancelled |
| priority | Select | Low, Medium, High |
| date | Date | Due date |
| description | Text Editor | Task description |
| reference_type | Link → DocType | Linked DocType |
| reference_name | Dynamic Link | Linked document |

## Email Account — Required Fields for Outgoing

| Field | Description |
|-------|-------------|
| email_id | Email address |
| enable_outgoing | Must be checked |
| smtp_server | SMTP host (e.g., smtp.gmail.com) |
| smtp_port | Port (587 for TLS, 465 for SSL) |
| use_tls | Enable TLS encryption |
| password | App-specific password |
| default_outgoing | Set as default sender |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-notifications/references/api-reference.md*
