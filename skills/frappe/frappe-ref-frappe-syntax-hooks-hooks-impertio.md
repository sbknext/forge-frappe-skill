---
title: Frappe Ref — Frappe Syntax: Hooks: Hooks (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'reference', 'hooks']
source: Impertio
---

# Complete Hooks Catalog

All Frappe configuration hooks organized by category. For document lifecycle
events (`doc_events`), see the **frappe-syntax-hooks-events** skill.

---

## App Metadata

| Hook | Type | Description |
|------|------|-------------|
| `app_name` | string | Slugified app identifier (REQUIRED) |
| `app_title` | string | Human-readable name (REQUIRED) |
| `app_publisher` | string | Publisher name |
| `app_description` | string | App description |
| `app_email` | string | Contact email |
| `app_license` | string | License identifier |
| `app_version` | string | Semantic version |
| `app_icon` | string | Icon reference |
| `app_color` | string | Brand color |
| `required_apps` | list | App dependencies |

---

## Frontend Asset Injection

| Hook | Type | Scope |
|------|------|-------|
| `app_include_js` | string/list | Desk (backend) JavaScript |
| `app_include_css` | string/list | Desk (backend) CSS |
| `web_include_js` | string/list | Website/portal JavaScript |
| `web_include_css` | string/list | Website/portal CSS |
| `webform_include_js` | dict | Per-webform JavaScript |
| `webform_include_css` | dict | Per-webform CSS |
| `page_js` | dict | Per-page desk scripts |
| `doctype_js` | dict | Form script extensions |
| `doctype_list_js` | dict | List view script extensions |
| `sounds` | list | Custom audio notifications |

---

## Installation & Migration Lifecycle

| Hook | Type | When Called |
|------|------|------------|
| `before_install` | string | Before app installation |
| `after_install` | string | After app installation |
| `after_sync` | string | After fixture sync |
| `before_migrate` | string | Before `bench migrate` |
| `after_migrate` | string | After `bench migrate` |
| `before_uninstall` | string | Before app removal [v15+] |
| `after_uninstall` | string | After app removal [v15+] |
| `before_tests` | string | Before test suite runs |

All receive NO arguments. Access context via `frappe.local`.

---

## Scheduler Events

| Hook | Type | Description |
|------|------|-------------|
| `scheduler_events` | dict | Periodic background tasks |

Frequency keys inside `scheduler_events`:

| Key | Queue | Timeout | Frequency |
|-----|-------|---------|-----------|
| `all` | default | 300s | ~60 seconds |
| `hourly` | default | 300s | Every hour at :00 |
| `daily` | default | 300s | Every day at 00:00 |
| `weekly` | default | 300s | Every Sunday 00:00 |
| `monthly` | default | 300s | 1st of month 00:00 |
| `hourly_long` | long | 1500s | Every hour |
| `daily_long` | long | 1500s | Every day |
| `weekly_long` | long | 1500s | Every week |
| `monthly_long` | long | 1500s | Every month |
| `cron` | default | 300s | Custom cron expression |

---

## Session & Authentication

| Hook | Type | Signature |
|------|------|-----------|
| `on_login` | string | `def handler(login_manager):` |
| `on_logout` | string | `def handler():` |
| `on_session_creation` | string | `def handler():` |
| `auth_hooks` | list | `def handler():` — request validators |

---

## Request/Response Middleware

| Hook | Type | When Called |
|------|------|------------|
| `before_request` | list | Before every HTTP request |
| `after_request` | list | After every HTTP response |
| `before_job` | list | Before background job execution |
| `after_job` | list | After background job execution |

---

## Permission Hooks

| Hook | Type | Signature |
|------|------|-----------|
| `permission_query_conditions` | dict | `def handler(user): -> str` |
| `has_permission` | dict | `def handler(doc, user, permission_type): -> bool/None` |

---

## DocType Extensions & Overrides

| Hook | Type | Version | Description |
|------|------|---------|-------------|
| `override_doctype_class` | dict | v14+ | Replace controller class (last app wins) |
| `extend_doctype_class` | dict | v16+ | Add mixin to controller (all coexist) |
| `doctype_js` | dict | v14+ | Extend form scripts |
| `doctype_list_js` | dict | v14+ | Extend list view scripts |
| `override_whitelisted_methods` | dict | v14+ | Replace API endpoints |
| `standard_queries` | dict | v14+ | Replace link field search |
| `additional_timeline_content` | dict | v14+ | Add timeline entries |

---

## Website & Portal

| Hook | Type | Description |
|------|------|-------------|
| `website_route_rules` | list | URL-to-controller mapping |
| `website_redirects` | list | URL redirects (regex supported) |
| `website_catch_all` | string | Custom 404 handler |
| `website_path_resolver` | string | Custom route resolution [v15+] |
| `get_web_pages_with_dynamic_routes` | string | Dynamic route definitions |
| `homepage` | string | Default homepage route |
| `role_home_page` | dict | Role-based homepages |
| `get_website_user_home_page` | string | Custom homepage function |
| `portal_menu_items` | list | Hardcoded sidebar items |
| `standard_portal_menu_items` | list | DB-synced sidebar items |
| `base_template` | string | Override web base template |
| `base_template_map` | dict | Regex-based template routing |
| `website_context` | dict | Static portal context vars |
| `update_website_context` | string | Dynamic context function |
| `extend_website_page_controller_context` | dict | Per-page context extension |
| `brand_html` | string | Custom navbar brand markup |

---

## File Handling

| Hook | Type | Description |
|------|------|-------------|
| `before_write_file` | string | Pre-save hook |
| `write_file` | string | Replace file storage (S3, CDN) |
| `delete_file_data_content` | string | Replace file deletion |

---

## Email

| Hook | Type | Description |
|------|------|-------------|
| `override_email_send` | string | Replace email sending backend |
| `get_sender_details` | string | Override From address/name |
| `default_mail_footer` | string | HTML footer for all emails |

---

## PDF

| Hook | Type | Description |
|------|------|-------------|
| `pdf_header_html` | string | Custom PDF header HTML |
| `pdf_body_html` | string | Custom PDF body wrapper |
| `pdf_footer_html` | string | Custom PDF footer HTML |

---

## Jinja

| Hook | Type | Description |
|------|------|-------------|
| `jinja.methods` | list | Custom functions for Jinja templates |
| `jinja.filters` | list | Custom filters for Jinja templates |

Configured via the `jinja` dict:
```python
jinja = {
    "methods": ["myapp.jinja_utils.my_method"],
    "filters": ["myapp.jinja_utils.my_filter"]
}
```

---

## Boot & Client Data

| Hook | Type | Description |
|------|------|-------------|
| `extend_bootinfo` | string | Inject data into `frappe.boot` |
| `notification_config` | string | Client notification configuration |

---

## Data & Fixtures

| Hook | Type | Description |
|------|------|-------------|
| `fixtures` | list | DB records to export/sync as JSON |
| `global_search_doctypes` | dict | DocTypes for global search indexing |
| `ignore_links_on_delete` | list | Skip link validation on delete |
| `calendars` | list | DocTypes with calendar view |
| `clear_cache` | string | App-specific cache clearing |

---

## Hook Resolution Order

When multiple apps define the same hook:

- **Override hooks** (`override_doctype_class`, `override_whitelisted_methods`): Last installed app wins
- **Extend hooks** (`doc_events`, `extend_bootinfo`, `scheduler_events`): ALL handlers run in installation order
- **List hooks** (`app_include_js`, `fixtures`): Values are merged from all apps

Adjust installation order via: Setup > Installed Applications > Update Hooks Resolution Order

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks/references/hooks.md*
