---
title: Frappe Ref — Frappe Impl: Website: Decision Tree (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'website', 'reference', 'decision-tree']
source: Impertio
---

# Website Page Type — Decision Tree

## Which Page Type Do I Need?

```
START: What is the use case?
│
├── Static informational content (About, Terms, FAQ)?
│   ├── Will it be edited by non-developers?
│   │   ├── YES → Web Page DocType (WYSIWYG editor, no code needed)
│   │   └── NO  → www/ portal page (.html + .py controller)
│   └── Does it need dynamic data from the database?
│       ├── YES → www/ portal page with get_context()
│       └── NO  → Web Page DocType or static .html in www/
│
├── External users need to submit data?
│   ├── Simple form (< 15 fields, no child tables)?
│   │   └── Web Form (built-in, no code needed)
│   ├── Complex form (child tables, multi-step, file uploads)?
│   │   └── Web Form + Custom Script OR custom www/ page with frappe.call
│   └── Need full CRUD (create, read, update, delete)?
│       └── has_web_view on DocType + portal page
│
├── List of records visible on website (articles, products, events)?
│   └── has_web_view on DocType
│       ├── Add `published` (Check) and `route` (Data) fields
│       ├── Create .html templates in doctype directory
│       └── Register in hooks.py: website_generators = ["DocType"]
│
├── Blog or news section?
│   └── Blog Post + Blog Category (built-in)
│       ├── NEVER build custom blog — use the built-in system
│       └── Customize via Blog Post web template override
│
├── Dynamic URL with parameters (/projects/<name>)?
│   └── website_route_rules in hooks.py
│       ├── Maps URL pattern to www/ page or DocType
│       └── Parameters available via frappe.form_dict
│
└── Full custom application page?
    └── www/ portal page with .html + .py + .js + .css
        ├── Use get_context() for server-side data
        ├── Use .js for client-side interactivity
        └── ALWAYS extend templates/web.html for consistent layout
```

## Web Form vs Portal Page vs Web Page

| Feature | Web Page | Web Form | Portal Page (www/) |
|---------|----------|----------|-------------------|
| Created by | UI (no code) | UI (no code) | Code (.html/.py) |
| Guest access | Yes | Configurable | Configurable |
| Data submission | No | Yes | Custom (frappe.call) |
| Custom logic | Limited | Events + server script | Full Python + JS |
| Child tables | N/A | Limited (v15+) | Full control |
| SEO meta tags | Built-in fields | Limited | Via context.metatags |
| Version control | No (in DB) | No (in DB) | Yes (in app code) |
| Multi-environment | Export as fixture | Export as fixture | Deployed with app |
| Best for | Marketing pages | Simple data collection | Complex portals |

## Static vs Dynamic Page Decision

```
Does the page content change per user or per request?
├── YES → Dynamic page
│   ├── ALWAYS set context.no_cache = 1
│   ├── ALWAYS use get_context() for data
│   └── Consider: will this scale? Cache where possible.
└── NO  → Static page
    ├── Let Frappe cache it (default behavior)
    ├── Use Web Page DocType for non-developer editing
    └── NEVER set no_cache on truly static pages (hurts performance)
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-website/references/decision-tree.md*
