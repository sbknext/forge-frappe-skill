---
title: Frappe Ref — Frappe Impl: Integrations: Decision Tree (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'integrations', 'reference', 'decision-tree']
source: Impertio
---

# Integration Decision Trees

## Decision Tree 1: Webhook vs API vs Scheduled Sync

```
HOW OFTEN DOES DATA CHANGE?
│
├─► On every document event (real-time)
│   ├─► Push to external? → Webhook DocType
│   ├─► Receive from external? → Whitelisted API endpoint
│   └─► Both directions? → Webhook OUT + API endpoint IN
│
├─► Periodically (hourly, daily)
│   ├─► < 100 records per batch → Scheduler task (hourly)
│   ├─► 100-10,000 records → Scheduler task (daily_long)
│   └─► > 10,000 records → Split into chunks + frappe.enqueue()
│
└─► On demand (user-triggered)
    ├─► Single record → Button + frappe.call() to whitelisted method
    └─► Bulk operation → Background job via frappe.enqueue()
```

## Decision Tree 2: Authentication Method

```
WHO IS CALLING THE API?
│
├─► External service (server-to-server)
│   ├─► Needs user context? → OAuth 2.0 Bearer Token
│   └─► System-level access? → API Key + Secret
│
├─► External app on behalf of user
│   ├─► Web app (server-side) → OAuth Authorization Code
│   ├─► SPA (client-side) → OAuth Authorization Code + PKCE
│   └─► Trusted first-party → Skip Authorization enabled
│
├─► Your Frappe calling external service
│   ├─► Needs user consent (Google, etc.) → Connected App
│   ├─► Server API key → Store in site_config or Password field
│   └─► No auth needed → Direct requests call
│
└─► Browser/portal user
    └─► Session-based authentication (login endpoint)
```

## Decision Tree 3: Data Import Method

```
HOW MUCH DATA?
│
├─► 1-100 rows
│   ├─► One-time → Data Import UI (manual upload)
│   └─► Recurring → API calls from external system
│
├─► 100-5,000 rows
│   ├─► CSV/XLSX available → Data Import DocType
│   └─► From API → Programmatic import with batch commits
│
├─► 5,000-50,000 rows
│   ├─► Split into batches of 5,000
│   └─► Use frappe.enqueue() for each batch
│
└─► 50,000+ rows
    ├─► Direct database insert (advanced, skip validation)
    └─► ALWAYS backup before direct DB operations
```

## Decision Tree 4: Error Handling Strategy

```
WHAT FAILED?
│
├─► External API returned error
│   ├─► 4xx (client error) → Log error, do NOT retry
│   ├─► 5xx (server error) → Retry with exponential backoff
│   ├─► Timeout → Retry once, then log
│   └─► Connection refused → Log, alert admin, skip
│
├─► Webhook delivery failed
│   ├─► Check Webhook Request Log
│   ├─► Verify URL is accessible
│   ├─► Verify payload format matches receiver expectations
│   └─► No auto-retry — implement manual retry or scheduled recheck
│
├─► OAuth token expired
│   ├─► Connected App → Automatic refresh (if refresh_token exists)
│   ├─► No refresh token → User must re-authorize
│   └─► Provider revoked access → User must re-authorize
│
└─► Data Import failed
    ├─► Check Import Log for row-level errors
    ├─► Common: missing mandatory fields, duplicate names
    ├─► Fix CSV and re-import failed rows only
    └─► NEVER re-import successful rows (duplicates!)
```

## Decision Tree 5: Sync Direction

```
WHICH DIRECTION?
│
├─► Frappe → External (push)
│   ├─► Real-time → Webhook or doc_event + enqueue
│   ├─► Batch → Scheduler task
│   └─► On demand → Button + whitelisted method
│
├─► External → Frappe (pull/receive)
│   ├─► External pushes → Whitelisted API endpoint
│   ├─► Frappe pulls → Scheduler + external API calls
│   └─► File-based → Data Import (CSV upload)
│
└─► Bidirectional
    ├─► Use last_modified timestamps to detect changes
    ├─► Implement conflict resolution (last-write-wins or merge)
    └─► ALWAYS log sync direction per record for debugging
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-integrations/references/decision-tree.md*
