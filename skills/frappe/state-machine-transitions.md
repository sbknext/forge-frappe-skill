---
title: Frappe Ref — Status State Machine Transitions (frappe-apps-manager)
category: frappe
tags: [frappe, workflow, status, state-machine, validate, controller]
source: vyogotech/frappe-apps-manager
---

# Status State Machine Transitions

## When to use

Trigger when implementing **custom status lifecycles** with explicit allowed transitions — beyond Frappe's Workflow doctype UI, or when status is a Select field and you need code-level guards (`Draft → Pending → Confirmed`, terminal states, role-independent rules). For approval gates with role permissions, consider the **Workflow** doctype first; use this pattern for lightweight, code-owned state machines.

## Patterns

### Allowed transitions in validate

```python
class SalesOrder(Document):
    def validate(self):
        self.validate_state_transition()

    def validate_state_transition(self):
        if self.is_new():
            return

        old_status = frappe.db.get_value(self.doctype, self.name, "status")
        if old_status == self.status:
            return

        allowed = {
            "Draft": ["Pending", "Cancelled"],
            "Pending": ["Confirmed", "Cancelled"],
            "Confirmed": ["In Progress", "Cancelled"],
            "In Progress": ["Completed", "On Hold"],
            "On Hold": ["In Progress", "Cancelled"],
            "Completed": [],
            "Cancelled": [],
        }.get(old_status, [])

        if self.status not in allowed:
            frappe.throw(
                _("Cannot transition from {0} to {1}").format(old_status, self.status)
            )
```

### Hook transitions to doc lifecycle

```python
def on_submit(self):
    self.status = "Confirmed"

def on_cancel(self):
    if self.status == "Submitted":
        self.reverse_side_effects()
    self.status = "Cancelled"
```

### State-dependent validation

```python
def validate(self):
    if self.status == "Draft":
        self.validate_draft_entry()
    elif self.status == "Submitted":
        self.validate_submitted_entry()
```

### Configurable status master (alternative)

When admins must add statuses, use `Link` → status master with a `category` field (Open/Resolved) instead of hardcoded `Select`. Transition rules can read the master's category:

```python
old_cat = frappe.db.get_value("HD Ticket Status", old_status, "category")
new_cat = frappe.db.get_value("HD Ticket Status", self.status, "category")
if old_cat == "Resolved" and new_cat == "Open":
    frappe.throw(_("Reopening resolved tickets is not allowed"))
```

## Client script mirror (UX only)

Show/hide actions by status in `refresh`; **never** rely on client-only checks — duplicate every transition rule in `validate`.

## Gotchas

- `frappe.db.get_value` in `validate` compares against **persisted** status — correct for edits; for new docs set initial status in `before_insert`.
- Terminal states (`Completed`, `Cancelled`) should have empty allowed lists to prevent drift.
- Status changes via **Data Import** or API bypass Desk UI — server `validate` is mandatory.
- Do not mix **docstatus** and custom status without clear rules — e.g. `on_submit` setting status while docstatus moves 0→1.
- Frappe **Workflow** doctype enforces transitions in UI; controller validation still needed for API/import paths.

## Version notes

Works identically on v13–v16. Workflow + server validation can coexist — keep rules in sync or disable Workflow actions that contradict code.

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-state-machine-helper/SKILL.md
