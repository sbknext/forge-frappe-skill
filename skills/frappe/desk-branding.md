---
title: Frappe Ref — Desk Branding & Theme (frappe-apps-manager)
category: frappe
tags: [frappe, desk, branding, theme, hooks, extend_bootinfo, navbar, css]
source: vyogotech/frappe-apps-manager
---

# Desk Branding & Theme

## When to use

Trigger when customizing **Frappe Desk** visuals — logo, primary color, navbar, login/splash, boot payload for client scripts, or exporting **Navbar Settings** as fixtures. For website-only theming without Desk, use Website Theme / web SCSS hooks instead.

## Implementation checklist

1. **Assets** — logo(s) under `your_app/public/images/` (SVG/PNG; optional dark-navbar variant).
2. **`hooks.py` CSS/JS** — `app_include_css` / `app_include_js` for Desk; optional `web_include_css` for website reuse.
3. **Built-in branding** — `app_logo_url`, `app_color`; optional `website_context` for `brand_html` on website templates.
4. **`extend_bootinfo`** — register hook; set `app_name`, `app_logo_url`, `brand_html`, optional `bootinfo.theme` dict.
5. **Navbar Settings** — set in Desk or ship filtered fixture JSON.
6. **Theme CSS** — CSS variables on `:root`; scope to Desk selectors (navbar, `.btn-primary`, form focus, list/workspace, `.for-login`, splash).
7. **Optional client JS** — `frappe.ready`, `startup`, `frappe.router.on('change', ...)` to re-apply navbar after navigation.
8. **Verify** — hard refresh; test light/dark if Theme Switcher enabled; confirm router navigation does not revert brand.

## When not to use

- **Website-only theming** without Desk — use Website Theme / web SCSS hooks instead.
- **Deep UX redesigns** — this covers branding surfaces, not wholesale layout rewrites.

## Patterns

### 1. Static assets + hooks.py

```python
# hooks.py
app_include_css = "/assets/your_app/css/desk-branding.css"
app_include_js = "/assets/your_app/js/desk-branding.js"

app_logo_url = "/assets/your_app/images/your-logo.svg"
app_color = "#2563eb"

extend_bootinfo = "your_app.boot.extend_bootinfo"

fixtures = [
    {"doctype": "Navbar Settings", "filters": [["name", "=", "Navbar Settings"]]},
]
```

Place logos under `your_app/public/images/`; run `bench build` (or `bench build --app your_app`) after asset changes.

### 2. extend_bootinfo (server)

```python
def extend_bootinfo(bootinfo):
    bootinfo.app_name = "YOUR_SHORT_LABEL"
    bootinfo.app_logo_url = "/assets/your_app/images/your-logo.svg"
    bootinfo.brand_html = (
        '<img src="/assets/your_app/images/your-logo.svg" '
        'alt="YOUR_PRODUCT" class="app-logo" style="height: 28px;">'
    )
    bootinfo.theme = {
        "primary_color": "#2563eb",
        "navbar_bg": "#1e293b",
    }
    return bootinfo
```

Client scripts read `frappe.boot.theme` and `frappe.boot.brand_html` at runtime.

### 3. CSS variables (Desk)

```css
:root {
    --brand-primary: #2563eb;
    --brand-navbar-bg: #1e293b;
    --brand-navbar-fg: #f8fafc;
    --primary-color: var(--brand-primary);
    --navbar-bg: var(--brand-navbar-bg);
    --navbar-color: var(--brand-navbar-fg);
}
```

Scope overrides to Desk selectors (navbar, `.btn-primary`, form focus, list/workspace shells, `.for-login`, splash `#startup_div`). Prefer variables over copying large upstream bundles.

### 4. Optional client JS

Run after boot with `frappe.ready` or `$(document).on('startup', ...)`. Re-apply on `frappe.router.on('change', ...)` if navbar re-renders. Typical tasks: webfont link, `document.title`, replace `.navbar-home a` from `frappe.boot.brand_html`, set CSS variables on `document.documentElement`.

### 5. Navbar Settings (database)

Set **Navbar Settings → App Logo** in Desk **or** ship via fixture JSON (see hooks above) so all sites stay consistent after `bench migrate`.

## Version notes (v11–v16 + develop)

| Area | Notes |
|------|--------|
| v13+ | Theme Switcher / dark mode — test light and dark; variables may need both themes |
| v14–v16 | Workspace sidebar uses CSS vars; avoid `!important` unless fighting specificity briefly |
| All | Hard refresh / clear cache after asset or boot changes |

## Gotchas

- Do not commit license-restricted artwork; use org-approved assets only.
- `extend_bootinfo` runs server-side every session boot — keep it lightweight.
- Router navigation can revert navbar HTML — re-apply in router change handler if needed.
- Website branding (`website_context`, `web_include_css`) is separate from Desk branding.

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-desk-branding/SKILL.md
