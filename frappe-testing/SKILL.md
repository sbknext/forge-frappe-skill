---
name: frappe-testing
description: >
  Write and run tests for Frappe/ERPNext apps — FrappeTestCase, test records, fixtures, mocking,
  permission tests, running with bench run-tests, and CI. Use when adding test coverage, doing TDD
  on a DocType or API, or setting up a test pipeline. Trigger on: "Frappe test", "FrappeTestCase",
  "bench run-tests", "unit test DocType", "test fixtures", "TDD frappe".
---

# Frappe Testing

Frappe ships a unittest-based harness with DB isolation. Tests run in a transaction that's rolled
back, so they don't pollute the site.

## Basic test

```python
# myapp/myapp/doctype/widget/test_widget.py
import frappe
from frappe.tests.utils import FrappeTestCase

class TestWidget(FrappeTestCase):
    def test_creates_with_defaults(self):
        w = frappe.get_doc({"doctype": "Widget", "title": "A"}).insert()
        self.assertEqual(w.status, "Open")

    def test_validation_blocks_bad_qty(self):
        with self.assertRaises(frappe.ValidationError):
            frappe.get_doc({"doctype": "Widget", "title": "B", "qty": -1}).insert()
```
`FrappeTestCase` wraps each test in a savepoint and rolls back — no manual cleanup needed.

## Test records (fixtures)

```python
# test_records.json next to the doctype → auto-loaded as dependencies
[ { "doctype": "Widget", "title": "Seed", "qty": 1 } ]
```
Or build inline with a helper:
```python
def make_widget(**kw):
    return frappe.get_doc({"doctype": "Widget", "title": "T", **kw}).insert()
```

## Permission tests

```python
def test_clerk_cannot_submit(self):
    frappe.set_user("clerk@example.com")
    self.assertFalse(frappe.has_permission("Widget", "submit"))
    frappe.set_user("Administrator")        # restore
```

## Mocking external calls

```python
from unittest.mock import patch

@patch("myapp.services.sync.requests.post")
def test_sync_calls_api(self, mock_post):
    mock_post.return_value.status_code = 200
    run_sync("W-0001")
    mock_post.assert_called_once()
```

## Running

```bash
bench --site <site> run-tests --app myapp
bench --site <site> run-tests --doctype "Widget"
bench --site <site> run-tests --module myapp.tests.test_sync
bench --site <site> run-tests --app myapp --coverage
bench --site <site> run-tests --skip-test-records --skip-before-tests   # fast iteration
bench --site <site> run-tests --junit-xml-output=/tmp/junit.xml         # CI output
```
Enable a test site: `bench --site <site> set-config allow_tests true`. Tests must run on a site whose name starts with `test_` to prevent accidental data loss on production sites.

## Rules

- Use `FrappeTestCase`, not bare `unittest.TestCase` — you get DB rollback + helpers.
- Never hit production data; `bench run-tests` uses the site's test mode.
- Make tests independent — don't rely on records another test created (savepoints roll back).
- For background jobs, call the method synchronously (`is_async=False`) or invoke the function directly.
- Assert behaviour and side-effects (DB rows, status), not implementation details.

Source: https://docs.frappe.io/framework/user/en/guides/automated-testing/unit-testing
