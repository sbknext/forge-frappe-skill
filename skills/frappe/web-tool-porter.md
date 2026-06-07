---
title: Frappe Ref — Integrating External Web Tools (frappe-apps-manager)
category: frappe
tags: [frappe, desk, page, frappe.require, assets, jquery, third-party]
source: vyogotech/frappe-apps-manager
---

# Integrating External Web Tools

## When to use

Trigger when porting a **standalone HTML/JS tool** (jQuery plugins, Gantt charts, legacy editors, 3D viewers) into a Frappe **Page** or DocType view — asset migration, path fixes, and lifecycle hooks.

## Procedure

### 1. Migrate assets

```
my_app/
  public/
    js/libs/gantt/
      ganttMaster.js
      gantt.css
      images/
```

Run `bench build --app my_app` after adding files.

### 2. Fix relative paths

External libs often use relative URLs. Update to absolute Frappe asset paths:

- Before: `url('../images/icon.png')`
- After: `url('/assets/my_app/js/libs/gantt/images/icon.png')`

### 3. Page wrapper with frappe.require

```javascript
frappe.pages["my-external-tool"].on_page_load = function (wrapper) {
    const page = frappe.ui.make_app_page({
        parent: wrapper,
        title: __("External Tool"),
        single_column: true,
    });

    frappe.require([
        "/assets/my_app/css/tool_style.css",
        "/assets/my_app/js/libs/tool_dependency.js",
        "/assets/my_app/js/libs/main_tool_logic.js",
    ]).then(() => {
        new MyExternalTool(wrapper, page);
    });
};
```

### 4. Inject DOM + init plugin

```javascript
class MyExternalTool {
    constructor(wrapper, page) {
        this.wrapper = $(wrapper);
        this.page = page;
        this.setup_dom();
        this.init_plugin();
    }

    setup_dom() {
        this.wrapper.find(".layout-main-section").append(`
            <div id="tool-container" style="height: 600px;">
                <div class="tool-toolbar"></div>
                <div class="tool-workspace"></div>
            </div>
        `);
    }

    init_plugin() {
        $("#tool-container").myPlugin({
            data: [],
            onSave: (data) => this.save_to_frappe(data),
        });
    }

    save_to_frappe(data) {
        frappe.call({
            method: "my_app.api.save_data",
            args: { data },
            callback: () => frappe.show_alert({ message: __("Saved"), indicator: "green" }),
        });
    }
}
```

### 5. Hot reload and resize

```javascript
$(wrapper).bind("show", () => {
    // Re-render when user navigates back to page
});

$(window).on("resize", () => {
    // Adjust tool container height for navbar/footer
});
```

### 6. hooks.py registration

Prefer **`frappe.require` on the Page** over global `app_include_js` — keeps initial Desk load light. Use `app_include_js` only when the library is needed on most forms.

## Gotchas

- **CSP / eval** — some legacy plugins use `eval`; may conflict with strict headers.
- **jQuery conflicts** — Frappe ships jQuery; test plugin compatibility with Desk version.
- **Multiple Page loads** — re-init plugins on `show` or guard with a singleton flag.
- **Whitelisted save API** — validate and permission-check all data from external tools server-side.

## Version notes (v11–v16)

`frappe.require` and Frappe Pages are stable. v14+ uses ES modules in some Desk areas — test legacy IIFE plugins in target version.

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-web-tool-porter/SKILL.md
