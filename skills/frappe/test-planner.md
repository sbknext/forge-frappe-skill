---
title: Frappe Ref — Test Planning & Manual Test Cases (frappe-apps-manager)
category: frappe
tags: [frappe, testing, test-plan, manual-testing, regression, qa]
source: vyogotech/frappe-apps-manager
---

# Test Planning & Manual Test Cases

## When to use

Trigger when creating **structured testing deliverables** for a Frappe app — test plans, manual test cases, regression/smoke suites, or bug report templates before or alongside automated `FrappeTestCase` coverage.

## Deliverables

### Test plan sections

| Section | Content |
|---------|---------|
| Scope | DocTypes, APIs, hooks in/out of scope |
| Strategy | Manual vs automated mix (testing pyramid) |
| Entry/exit | e.g. "controllers at 80% coverage" or "all P0 manual cases pass" |
| Risks | Probability × impact for high-risk flows |
| Environments | Frappe/ERPNext version, site config, apps installed |

### Manual test case format

```markdown
## TC-001: Submit Sales Order with valid items
**Priority**: High
**Preconditions**: User with Sales User role; Customer and Item exist
**Steps**:
1. Create Sales Order, add item qty 2 → **Expected**: rate × qty in amount
2. Save and Submit → **Expected**: docstatus 1, status Submitted
**Test data**: Customer CUST-001, Item ITEM-001
```

### Regression suites

- **Smoke (5–10 tests)**: login, core dashboard, create/submit one primary DocType
- **Critical path**: end-to-end flows (Lead → Quotation → Order → Invoice, Issue → Resolution)

### Bug report template

Include: Frappe/ERPNext version, branch, site config, reproduction steps, expected vs actual, traceback/console logs, screenshots.

## Frappe-specific verification checklist

**Desk / client scripts**

- [ ] Field layout matches spec
- [ ] Client Script show/hide and `set_query` behave on refresh and field change
- [ ] Client validation mirrored on server

**Data integrity**

- [ ] `before_insert` / `on_update` handle null/edge values
- [ ] Role Permissions and User Permissions enforced (not only UI)
- [ ] Naming series generates expected IDs
- [ ] State transition rules enforced via API/import (not only Desk)

**Hooks & jobs**

- [ ] `doc_events` and `scheduler_events` tested after `bench migrate`
- [ ] Background jobs: test with `is_async=False` or invoke worker function directly

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Happy-path only | Edge cases: cancel, amend, permission denied, empty child table |
| Vague steps ("check it works") | Action → expected result per step |
| No cleanup | Delete test records or use dedicated test site (`test_*`) |
| Ignoring hooks | Test feature + global `hooks.py` side effects |

## Version notes (v11–v16)

Align test matrix with target version — v15+ Server Scripts disabled by default; v16+ data masking affects permission tests.

## Source

- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-test-planner/SKILL.md
