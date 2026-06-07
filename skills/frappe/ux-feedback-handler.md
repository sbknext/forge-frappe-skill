---
title: Frappe Ref — Desk UX Feedback (Alerts, Progress, Dialogs) (frappe-apps-manager)
category: frappe
tags: [frappe, desk, ux, show_alert, show_progress, frappe.confirm, dialog]
source: vyogotech/frappe-apps-manager
---

# Desk UX Feedback (Alerts, Progress, Dialogs)

## When to use

Trigger when improving **user feedback during long tasks or state changes** — success/error toasts, progress bars for imports/syncs, confirmation before destructive actions, or pairing with `frappe.publish_realtime` progress events.

## Patterns

### Alerts (toasts)

```javascript
// Success — auto-dismiss after 5s
frappe.show_alert({
    message: __("Action completed successfully"),
    indicator: "green",
}, 5);

// Error — longer persistence
frappe.show_alert({
    message: __("Something went wrong"),
    indicator: "red",
}, 10);

// Indicators: green | red | orange | blue | yellow
```

### Progress bar

```javascript
// Manual progress (client-only loop)
frappe.show_progress(__("Importing Data…"), 45, 100, __("Processing row 45/100"));

// Hide when done (if not auto-closed)
frappe.hide_progress();
```

Pair with server `frappe.publish_realtime("task_progress", { progress, total, message })` and listen in client — see `frappe-reference-realtime.md`.

### Confirmation dialog

```javascript
frappe.confirm(
    __("Are you sure you want to proceed with this bulk action?"),
    () => { /* Yes */ run_bulk_action(); },
    () => { /* No — optional */ }
);
```

### Blocking freeze during server call

```javascript
frappe.call({
    method: "myapp.api.heavy_sync",
    freeze: true,
    freeze_message: __("Syncing…"),
    callback(r) {
        if (!r.exc) frappe.show_alert({ message: __("Done"), indicator: "green" });
    },
});
```

### msgprint vs show_alert

| API | Use when |
|-----|----------|
| `frappe.show_alert` | Brief non-blocking feedback |
| `frappe.msgprint` | User must read details; supports HTML, multiple buttons |
| `frappe.throw` | Block save / action with validation error (server or client `validate`) |

## Gotchas

- **`frappe.show_progress` without hide** — can leave a stale bar; always hide or complete to 100/100.
- **Duplicate realtime handlers** — registering `frappe.realtime.on` on every `refresh` stacks handlers; register once in `onload` or off previous handler.
- **Translated strings** — wrap user-visible text in `__("…")`.
- **Guest / portal** — some Desk APIs differ on website portal; test both contexts if applicable.

## Version notes (v11–v16)

`show_alert`, `show_progress`, and `frappe.confirm` are stable Desk APIs. v15+ toast styling uses indicator colors consistently across themes.

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-ux-feedback-handler/SKILL.md
