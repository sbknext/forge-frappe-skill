---
title: Frappe Ref — Frappe Impl: Ui Components: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'ui', 'components', 'reference', 'anti-patterns']
source: Impertio
---

# UI Components Anti-Patterns

## AP-1: Using frappe.msgprint for Errors

**Wrong**:
```javascript
if (!frm.doc.customer) {
    frappe.msgprint("Customer is required");
    // Execution continues! Form may still save.
}
```

**Correct**: ALWAYS use `frappe.throw` for validation errors — it stops execution.
```javascript
if (!frm.doc.customer) {
    frappe.throw(__("Customer is required"));
    // Execution stops here
}
```

## AP-2: Hiding Dialog Before Async Call Completes

**Wrong**:
```javascript
primary_action(values) {
    d.hide();  // Dialog closes immediately
    frappe.call({ method: "myapp.api.create", args: values });
    // User has no feedback if the call fails
}
```

**Correct**: ALWAYS hide in the callback after success.
```javascript
primary_action(values) {
    d.disable_primary_action();  // Prevent double-click
    frappe.call({
        method: "myapp.api.create",
        args: values,
        callback(r) {
            if (!r.exc) d.hide();
        },
        always() {
            d.enable_primary_action();  // Re-enable on success or failure
        }
    });
}
```

## AP-3: Synchronous Calls in UI Components

**Wrong**:
```javascript
primary_action(values) {
    let result = frappe.call({ method: "myapp.api.check", args: values, async: false });
    // Freezes the entire browser tab
}
```

**Correct**: ALWAYS use async calls. Use `frappe.xcall` for promise-based flow.
```javascript
primary_action(values) {
    frappe.xcall("myapp.api.check", values).then(result => {
        // Handle result
    });
}
```

## AP-4: Missing in_list_view on Table Fields

**Wrong**:
```javascript
fields: [
    { fieldname: "item", label: "Item", fieldtype: "Link", options: "Item" },
    { fieldname: "qty", label: "Qty", fieldtype: "Int" },
]
// Both fields are hidden in the grid — user sees empty rows
```

**Correct**: ALWAYS set `in_list_view: 1` on fields you want visible in the table grid.
```javascript
fields: [
    { fieldname: "item", label: "Item", fieldtype: "Link", options: "Item",
      in_list_view: 1 },
    { fieldname: "qty", label: "Qty", fieldtype: "Int", in_list_view: 1 },
]
```

## AP-5: publish_realtime Without after_commit

**Wrong**:
```python
def on_update(self):
    frappe.publish_realtime("order_updated", {"name": self.name})
    # If the transaction rolls back, the event was already sent!
```

**Correct**: ALWAYS use `after_commit=True` in document lifecycle events.
```python
def on_update(self):
    frappe.publish_realtime("order_updated", {"name": self.name}, after_commit=True)
```

## AP-6: Kanban Board on DocType Without Select Field

**Wrong**: Creating a Kanban Board for a DocType that has no Select field — the board has no columns.

**Correct**: ALWAYS ensure the target DocType has a Select field with the status options. The Select field's options become the Kanban columns.

## AP-7: Missing Field Mapping in Calendar View

**Wrong**:
```javascript
frappe.views.calendar["Event"] = {
    field_map: {
        start: "starts_on",
        // Missing "end" — all events appear as zero-duration
    }
};
```

**Correct**: ALWAYS map both `start` and `end` fields.
```javascript
frappe.views.calendar["Event"] = {
    field_map: {
        start: "starts_on",
        end: "ends_on",
        id: "name",
        title: "subject",
    }
};
```

## AP-8: Two-Element Array from get_indicator

**Wrong**:
```javascript
get_indicator(doc) {
    return ["Active", "green"];  // Missing filter — click does nothing
}
```

**Correct**: ALWAYS return 3 elements. The third is the filter applied when clicking.
```javascript
get_indicator(doc) {
    return ["Active", "green", "status,=,Active"];
}
```

## AP-9: Not Cleaning Up Realtime Listeners

**Wrong**:
```javascript
onload(frm) {
    frappe.realtime.on("my_event", handler);
    // Every form load adds another listener — memory leak, duplicate handling
}
```

**Correct**: ALWAYS clean up listeners when leaving the form.
```javascript
onload(frm) {
    frappe.realtime.off("my_event");  // Remove previous listener first
    frappe.realtime.on("my_event", handler);
}
```

## AP-10: Blocking UI During Long Operations

**Wrong**: Running a long server call without any progress feedback.

**Correct**: Use `frappe.publish_progress` on the server and optionally `freeze: true` with a message on the client.
```javascript
frappe.call({
    method: "myapp.api.long_operation",
    args: { ... },
    freeze: true,
    freeze_message: __("Processing, please wait..."),
});
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-ui-components/references/anti-patterns.md*
