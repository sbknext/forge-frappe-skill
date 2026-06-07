---
title: Frappe Ref — Frappe Core: Notifications: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'notifications', 'reference', 'examples']
source: Impertio
---

# Notification Examples

## 1. Send Email with Attachment on Submit

```python
def on_submit(doc, method):
    """Send invoice PDF to customer on submission."""
    pdf = frappe.attach_print(
        doc.doctype, doc.name, print_format="Standard"
    )
    frappe.sendmail(
        recipients=[doc.customer_email],
        subject=f"Invoice {doc.name}",
        message=f"<p>Dear {doc.customer_name}, please find your invoice attached.</p>",
        attachments=[pdf],
        reference_doctype=doc.doctype,
        reference_name=doc.name,
    )
```

## 2. System Notification with Progress Bar

```python
def process_batch(items):
    """Process items with real-time progress updates."""
    total = len(items)
    for i, item in enumerate(items):
        process_item(item)
        frappe.publish_realtime(
            event="progress",
            message={
                "progress": [i + 1, total],
                "title": "Processing batch...",
            },
            user=frappe.session.user,
            after_commit=False,  # OK here — not in a write transaction
        )
```

## 3. Conditional Email Notification (No Code)

Create a Notification DocType record:

| Field | Value |
|-------|-------|
| Name | Overdue Invoice Alert |
| Document Type | Sales Invoice |
| Event | Days After |
| Days After | 7 |
| Date Field | due_date |
| Condition | `doc.outstanding_amount > 0` |
| Channel | Email |
| Recipients | Document Field: customer_email |
| Subject | `Overdue: {{ doc.name }}` |
| Message | `<p>Invoice {{ doc.name }} is overdue by {{ frappe.utils.date_diff(nowdate(), doc.due_date) }} days.</p>` |

## 4. Assignment with Notification

```python
from frappe.desk.form.assign_to import add

def auto_assign_task(doc, method):
    """Assign new tasks to the team lead."""
    if doc.is_new():
        add({
            "assign_to": ["teamlead@example.com"],
            "doctype": doc.doctype,
            "name": doc.name,
            "description": f"New task created: {doc.subject}",
            "priority": doc.priority or "Medium",
        })
```

## 5. Auto Repeat Setup (Programmatic)

```python
auto_repeat = frappe.get_doc({
    "doctype": "Auto Repeat",
    "reference_doctype": "Journal Entry",
    "reference_document": "JV-00001",
    "frequency": "Monthly",
    "start_date": "2025-01-01",
    "end_date": "2025-12-31",
    "notify_by_email": 1,
    "recipients": "accountant@example.com",
}).insert()
```

## 6. Custom Realtime Event with Client Handler

Server-side:
```python
frappe.publish_realtime(
    event="custom_alert",
    message={"title": "Stock Low", "item": "ITEM-001", "qty": 5},
    user="warehouse@example.com",
    after_commit=True,
)
```

Client-side (JS):
```javascript
frappe.realtime.on("custom_alert", (data) => {
    frappe.show_alert({
        message: `${data.title}: ${data.item} (${data.qty} remaining)`,
        indicator: "orange",
    });
});
```

## 7. Bulk Comment and Tag

```python
def mark_reviewed(doc_names):
    """Add review comment and tag to multiple documents."""
    for name in doc_names:
        doc = frappe.get_doc("Task", name)
        doc.add_comment("Comment", text="Reviewed in batch process")
        doc.add_tag("reviewed")
```

## 8. Email Template with Jinja

Template stored in Email Template DocType:

```html
<h2>Welcome, {{ doc.employee_name }}!</h2>

<p>Your onboarding is scheduled for {{ frappe.utils.format_date(doc.date_of_joining) }}.</p>

<h3>Checklist:</h3>
<ul>
{% for item in doc.onboarding_items %}
  <li>{{ item.activity }} — Due: {{ frappe.utils.format_date(item.due_date) }}</li>
{% endfor %}
</ul>

<p>Contact HR at hr@example.com for questions.</p>
```

## 9. Notification with Attach Print and Set Property After Alert

Create a Notification to mark invoices as "Reminded":

| Field | Value |
|-------|-------|
| Event | Days After |
| Days After | 3 |
| Date Field | due_date |
| Condition | `doc.outstanding_amount > 0 and doc.status != "Reminded"` |
| Attach Print | Yes |
| Print Format | Invoice Reminder |
| Set Property After Alert | status = "Reminded" |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-notifications/references/examples.md*
