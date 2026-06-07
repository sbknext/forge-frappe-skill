---
title: Frappe Ref — Frappe Impl: Integrations: Workflows (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'integrations', 'reference', 'workflows']
source: Impertio
---

# Integration Workflows — Extended Reference

## OAuth 2.0 Provider: Complete Authorization Code Flow

```
1. External app redirects user to:
   GET /api/method/frappe.integrations.oauth2.authorize
     ?client_id=<client_id>
     &redirect_uri=<callback_url>
     &response_type=code
     &scope=openid all

2. User authenticates with Frappe credentials

3. User grants consent (unless Skip Authorization is enabled)

4. Frappe redirects to callback with authorization code:
   GET <callback_url>?code=<authorization_code>

5. External app exchanges code for token:
   POST /api/method/frappe.integrations.oauth2.get_token
   Body: {
     grant_type: authorization_code,
     code: <authorization_code>,
     redirect_uri: <callback_url>,
     client_id: <client_id>,
     client_secret: <client_secret>
   }

6. Frappe returns access_token + refresh_token

7. External app uses token for API calls:
   GET /api/resource/Sales Invoice
   Authorization: Bearer <access_token>
```

## Connected App: Token Refresh Flow

```
1. Access token expires (typically 1 hour)
2. Connected App detects expired token
3. Automatic refresh using stored refresh_token
4. New access_token issued
5. API call retried with new token

If refresh_token also expired:
→ User must re-authorize via "Connect to..." button
```

## Webhook Delivery Flow

```
1. Document event fires (e.g., on_submit on Sales Invoice)
2. Frappe checks active Webhooks for matching DocType + Event
3. Conditions evaluated (Jinja expression)
4. If conditions pass:
   a. Payload constructed (Form or JSON format)
   b. HMAC signature computed (if secret configured)
   c. HTTP request sent to Request URL
   d. Webhook Request Log created with response
5. If delivery fails:
   a. Error logged in Webhook Request Log
   b. No automatic retry (implement retry externally)
```

## Data Import Flow

```
1. Create Data Import document
2. Select Reference DocType
3. Choose Import Type:
   - Insert New Records: creates new documents
   - Update Existing Records: updates by name/ID
4. Download Template (preserves column order)
5. Fill data in template
6. Upload file
7. Preview imported data
8. Start Import (background job)
9. Check Import Log for results:
   - Success: document created/updated
   - Warning: partial success
   - Error: row skipped with error message
```

## Payment Request Flow

```
1. Create Payment Request (linked to Sales Invoice/Order)
2. Payment Request selects Payment Gateway
3. User redirected to gateway payment page
4. User completes payment
5. Gateway sends callback to Frappe
6. Payment controller processes callback:
   a. Validates payment signature
   b. Creates Payment Entry
   c. Updates Payment Request status
   d. Optionally submits linked document
```

## Scheduled Sync Pattern

```python
# hooks.py
scheduler_events = {
    "hourly": ["myapp.integrations.sync.hourly_sync"],
    "daily": ["myapp.integrations.sync.daily_full_sync"]
}

# myapp/integrations/sync.py
import frappe

def hourly_sync():
    """Sync recent changes only."""
    last_sync = frappe.db.get_single_value("My Integration Settings", "last_sync_time")
    records = frappe.get_all("Sales Invoice",
        filters={"modified": [">", last_sync]},
        fields=["name", "customer", "grand_total"]
    )
    for record in records:
        try:
            push_to_external(record)
        except Exception as e:
            frappe.log_error(f"Sync failed for {record.name}: {e}")

    frappe.db.set_single_value("My Integration Settings", "last_sync_time",
                                frappe.utils.now_datetime())
    frappe.db.commit()

def daily_full_sync():
    """Full reconciliation — runs as daily_long."""
    # Use daily_long in hooks.py if > 5 minutes
    pass
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-integrations/references/workflows.md*
