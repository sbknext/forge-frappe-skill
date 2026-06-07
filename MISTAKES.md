# Frappe Development Mistakes — and the Correct Pattern

A field guide to the mistakes that bite Frappe developers (and the AI assistants
generating Frappe code) most often. Each entry: **the mistake → why it bites →
the fix**. Patterns are written against Frappe Framework v13–v15.

These are generic, framework-level mistakes drawn from public Frappe knowledge.
They are not specific to any private codebase. When in doubt, prefer the version
of an API documented for the target Frappe version — see the version-drift notes
throughout.

---

## 1. Raw SQL with string formatting (SQL injection)

**Mistake**

```python
# DANGER: user-controlled value spliced straight into SQL
status = frappe.form_dict.get("status")
rows = frappe.db.sql(f"SELECT name FROM `tabSales Order` WHERE status = '{status}'")
```

**Why it bites**

f-strings / `%`-formatting / `.format()` build the query text from untrusted
input, so a value like `' OR '1'='1` rewrites the query. This is the single most
common security defect in Frappe apps.

**The fix**

Use parameterized queries, or — better — the ORM / Query Builder.

```python
# Parameterized (values are bound, never interpolated)
rows = frappe.db.sql(
    "SELECT name FROM `tabSales Order` WHERE status = %(status)s",
    {"status": status},
    as_dict=True,
)

# Better: ORM
rows = frappe.get_all("Sales Order", filters={"status": status}, pluck="name")

# Best for complex joins: Query Builder (frappe.qb), which always parameterizes
so = frappe.qb.DocType("Sales Order")
rows = frappe.qb.from_(so).select(so.name).where(so.status == status).run(as_dict=True)
```

Never concatenate identifiers either — if a table/column name comes from input,
validate it against a whitelist of known names before use.

---

## 2. Missing permission checks in whitelisted endpoints

**Mistake**

```python
@frappe.whitelist()
def get_customer_balance(customer):
    return frappe.get_doc("Customer", customer).outstanding_amount
```

**Why it bites**

`@frappe.whitelist()` only makes a method *callable* over HTTP — it does **not**
enforce DocType permissions. Any authenticated user (and with
`allow_guest=True`, any anonymous user) can pass any `customer` name and read
data they should not see. This is an Insecure Direct Object Reference.

**The fix**

Enforce permissions explicitly. `frappe.get_doc(...)` does *not* check read
permission on its own; use `frappe.has_permission` or `.check_permission()`.

```python
@frappe.whitelist()
def get_customer_balance(customer):
    if not frappe.has_permission("Customer", "read", doc=customer):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    doc = frappe.get_doc("Customer", customer)
    return doc.outstanding_amount
```

For role-gated endpoints add `frappe.only_for("Accounts Manager")`. For methods
that must never be public, do **not** set `allow_guest=True`.

---

## 3. Module-level globals instead of `frappe.cache()`

**Mistake**

```python
# module scope — computed once per worker process, never refreshed
SETTINGS = frappe.get_single("My Settings")

def do_work():
    if SETTINGS.feature_enabled:
        ...
```

**Why it bites**

Frappe runs multiple gunicorn workers and background workers. A module-level
global is evaluated at import time, is per-process, is shared across *all sites*
in a multi-tenant bench (a serious data-leak / wrong-site bug), and never
reflects config changes until the worker restarts.

**The fix**

Read config per request, and cache hot values in the request/redis cache, which
is site-scoped and invalidatable.

```python
def get_settings():
    return frappe.get_cached_doc("My Settings")  # site-scoped, cleared on save

def do_work():
    if get_settings().feature_enabled:
        ...
```

For arbitrary values use `frappe.cache().get_value(...)` / `set_value(...)`, and
clear with `frappe.cache().delete_value(...)` (e.g. from an `on_update` hook).

---

## 4. Wrong hook for the job (`scheduler_events`, `doc_events`)

**Mistake**

- Putting business logic in `after_insert` that needs the *submitted* state.
- Registering a cron job by editing crontab instead of `scheduler_events`.
- Hooking `validate` to send emails (re-runs on every save, including drafts).

**Why it bites**

Hooks fire at specific lifecycle points. `validate` runs every save (and can run
multiple times). `on_submit` runs only on docstatus 0→1. Side effects in
`validate` (emails, external API calls, row inserts) get duplicated or fire for
documents that are never saved. External crontab entries bypass the scheduler,
don't respect site context, and break on multi-site benches.

**The fix**

Pick the correct event and keep `validate` pure (validation only):

| Need | Hook |
|------|------|
| Mutate/validate fields before write | `validate` / `before_save` |
| Act after a *new* doc is created | `after_insert` |
| Act when a doc is submitted | `on_submit` |
| Act when a submitted doc is cancelled | `on_cancel` |
| Periodic jobs | `scheduler_events` (`daily`, `hourly`, `cron`) |

```python
# hooks.py
scheduler_events = {
    "daily": ["my_app.tasks.send_daily_digest"],
    "cron": {"0 */6 * * *": ["my_app.tasks.sync_inventory"]},
}
doc_events = {
    "Sales Invoice": {"on_submit": "my_app.events.notify_finance"},
}
```

Make scheduled jobs idempotent — the scheduler can re-run them after a crash.

---

## 5. Swallowing `frappe.ValidationError` / bare `except`

**Mistake**

```python
try:
    doc.submit()
except Exception:
    pass  # "ignore errors"
```

**Why it bites**

A bare `except` swallows `frappe.ValidationError`, `frappe.PermissionError`,
*and* `frappe.db` rollback signals. The transaction may be left half-applied,
the user gets a success response for a failed operation, and the real cause is
hidden. It also catches `frappe.does_not_exist` style errors you wanted to see.

**The fix**

Catch the specific exception, let validation/permission errors propagate to
Frappe's handler (it produces the right HTTP status + user message), and log
anything you genuinely recover from.

```python
try:
    doc.submit()
except frappe.ValidationError:
    raise  # let Frappe format the 417 + message
except SomeExternalAPIError as e:
    frappe.log_error(message=frappe.get_traceback(), title="Inventory sync failed")
    frappe.throw(_("Could not reach inventory service. Try again."))
```

In an API, raising `frappe.throw(msg)` is the idiomatic way to return an error —
don't catch and `return {"error": ...}` by hand unless you also set the status.

---

## 6. Client Script vs Server Script confusion

**Mistake**

Putting security or data-integrity logic in a **Client Script** (form JS),
assuming it protects the data. Or expecting `frappe.db.set_value` calls in client
JS to respect server-side validation.

**Why it bites**

Client Scripts run in the browser. They are for UX — field visibility, fetching
related values, client-side hints. A malicious user can bypass them entirely by
calling the REST API directly. Validation that only exists client-side is not
validation.

**The fix**

- **Client Script**: UX only — `frm.set_df_property`, `frm.add_fetch`, dynamic
  field display, friendly warnings.
- **Server Script / controller `validate`**: every rule that must hold — amount
  limits, mandatory combinations, permission gates.

Mirror client hints with a server-side check. The server is the source of truth.
(Note: enabling untrusted Server Scripts has its own risk surface — only allow
them from trusted authors, since they run server-side Python.)

---

## 7. Trusting `frappe.form_dict` types / mass-assigning user input

**Mistake**

```python
@frappe.whitelist()
def update_order(order, data):
    doc = frappe.get_doc("Sales Order", order)
    doc.update(frappe.parse_json(data))   # user controls every field
    doc.save()
```

**Why it bites**

All HTTP params arrive as strings, and `doc.update(user_dict)` lets the caller
set *any* field — `owner`, `docstatus`, `status`, price fields, child rows. This
is mass-assignment. `"0"` is truthy in Python, so `if doc.disabled:` on a string
misbehaves too.

**The fix**

Whitelist the fields you accept and coerce types via `frappe.utils`.

```python
ALLOWED = {"delivery_date", "customer_notes"}

@frappe.whitelist()
def update_order(order, data):
    data = frappe.parse_json(data)
    doc = frappe.get_doc("Sales Order", order)
    doc.check_permission("write")
    for field in ALLOWED & data.keys():
        doc.set(field, data[field])
    doc.save()
```

Use `cint`, `flt`, `cstr`, `getdate` from `frappe.utils` to coerce, never raw
`int()`/`float()` (they throw on empty strings; the Frappe helpers handle None).

---

## 8. Committing inside request handlers / fighting the transaction

**Mistake**

```python
@frappe.whitelist()
def process():
    for row in rows:
        frappe.get_doc(...).insert()
        frappe.db.commit()   # commit per row "to be safe"
```

**Why it bites**

Frappe wraps each web request in a transaction and commits on success / rolls
back on exception automatically. Manual `frappe.db.commit()` mid-request defeats
that: a later failure can no longer roll back the already-committed rows, leaving
partial data. It also hurts performance and can deadlock.

**The fix**

Let Frappe manage the request transaction — do **not** call `commit()` in a web
request. For nested partial rollback use savepoints:

```python
for row in rows:
    sp = f"row_{row.idx}"
    frappe.db.savepoint(sp)
    try:
        frappe.get_doc(...).insert()
    except Exception:
        frappe.db.rollback(save_point=sp)
        frappe.log_error(title="Row failed")
```

Explicit `commit()` is appropriate in long-running background jobs / patches that
intentionally checkpoint — not in HTTP handlers.

---

## 9. N+1 queries with `frappe.get_doc` in loops

**Mistake**

```python
names = frappe.get_all("Sales Order", pluck="name")
total = 0
for name in names:
    total += frappe.get_doc("Sales Order", name).grand_total  # one query each
```

**Why it bites**

`frappe.get_doc` loads the full document (parent + all child tables) per call.
Over hundreds of rows this is thousands of queries and a slow request.

**The fix**

Fetch the fields you need in one query.

```python
rows = frappe.get_all("Sales Order", fields=["name", "grand_total"])
total = sum(r.grand_total for r in rows)

# Or aggregate in the DB
total = frappe.db.get_value("Sales Order", filters={}, fieldname="sum(grand_total)")
```

Only `get_doc` when you need the controller methods or child rows. Use
`frappe.get_all(..., fields=[...])` / Query Builder for read-heavy paths.

---

## 10. Hardcoding fieldnames / inventing DocType fields

**Mistake**

Generating code that references `doc.customer_name`, `doc.total_amount`,
`doc.is_active` without verifying those fields exist on the DocType. AI
assistants are especially prone to inventing plausible-but-nonexistent fields.

**Why it bites**

Frappe raises at runtime (`AttributeError` for controller access, or silently
returns `None` for `db.get_value` of an unknown field in some versions). The bug
surfaces only when the code path runs, often in production.

**The fix**

Verify the schema before writing field access. Read the DocType JSON
(`<app>/<module>/doctype/<dt>/<dt>.json`) or query meta:

```python
meta = frappe.get_meta("Sales Order")
assert meta.has_field("grand_total")
# In bench console:
#   frappe.get_meta("Sales Order").fields  -> list of fieldnames
```

Prefer `bench --site <site> console` or reading the `.json` over guessing. Never
assume a field exists because it "should."

---

## 11. `frappe.db.get_value` vs `get_all` vs `get_doc` misuse

**Mistake**

Using `frappe.db.get_value("DocType", name)` (no fieldname) expecting a dict, or
using `get_doc` just to read one scalar, or using `get_all` without `filters`
and accidentally scanning the whole table.

**Why it bites**

- `get_value(dt, name)` with no fieldname returns the *name*, not a dict.
- `get_doc` to read one field loads the entire document (see #9).
- `get_all` ignores user permissions by design; `get_list` respects them.

**The fix**

| Goal | API |
|------|-----|
| One or a few scalars | `frappe.db.get_value(dt, name, ["f1", "f2"], as_dict=True)` |
| Permission-filtered list (user-facing) | `frappe.get_list(...)` |
| Full data, ignore permissions (internal/system) | `frappe.get_all(...)` |
| Full document + controller methods | `frappe.get_doc(...)` |

Know that `get_all` bypasses permissions — never expose its raw output to a user
without a permission check, or use `get_list` instead.

---

## 12. Not respecting docstatus / editing submitted documents

**Mistake**

```python
doc = frappe.get_doc("Sales Invoice", name)
doc.due_date = new_date
doc.save()   # but the invoice is submitted (docstatus 1)
```

**Why it bites**

Submitted documents are immutable except for fields marked
`allow_on_submit`. A plain `save()` on a submitted doc raises
`UpdateAfterSubmitError`. Newer code that ignores `docstatus` breaks on real
data where most documents are submitted.

**The fix**

Check `docstatus` and use the right mechanism:

```python
if doc.docstatus == 1:
    # only allow_on_submit fields can change this way
    doc.db_set("remarks", new_remarks)   # if 'remarks' is allow_on_submit
else:
    doc.due_date = new_date
    doc.save()
```

To change a submitted document's core data, the correct flow is cancel → amend →
resubmit, not a silent `db_set` that skips validation.

---

## 13. `db_set` to skip validation (and the data corruption that follows)

**Mistake**

```python
doc.db_set("status", "Completed")   # used to "avoid validation hassles"
```

**Why it bites**

`db_set` writes straight to the DB, bypassing `validate`, controller logic,
dependent-field recalculation, and child-table totals. It is the right tool for
narrow, deliberate updates — but used as a general save replacement it silently
corrupts derived state (totals out of sync, status inconsistent with linked
docs).

**The fix**

Use `doc.save()` (runs full lifecycle) for normal edits. Reserve `db_set` for:
single denormalized status flags, `allow_on_submit` fields, or hot-path counters
where you've reasoned about the skipped validation. Document *why* each `db_set`
is safe.

---

## 14. Reading secrets from DocType fields / committing secrets

**Mistake**

Storing API keys, passwords, or tokens in plain `Data` fields, or hardcoding
them in source / `hooks.py`.

**Why it bites**

Plain fields are world-readable to anyone with DocType read permission, appear in
exports/backups, and end up in version control. Hardcoded secrets leak the moment
the repo is shared.

**The fix**

- Use the `Password` fieldtype — Frappe encrypts it at rest; read it with
  `doc.get_password("fieldname")`.
- For app/site config, use `site_config.json` and read via
  `frappe.conf.get("my_api_key")`.
- Never log full secrets; never put them in tracebacks or chat. Use placeholders
  (`your_api_key`, `eyJ...`) in docs only.

---

## 15. Blocking the web worker with long / external work

**Mistake**

```python
@frappe.whitelist()
def export_everything():
    for row in million_rows:
        build_pdf(row)        # 5 minutes, in the request
    return file_url
```

**Why it bites**

Web workers have request timeouts (gunicorn / proxy). Long loops, big exports,
and synchronous external API calls block a worker, time out for the user, and
starve other requests.

**The fix**

Offload to the background queue and report progress.

```python
@frappe.whitelist()
def export_everything():
    frappe.enqueue(
        "my_app.tasks.do_export",
        queue="long",
        timeout=1500,
        user=frappe.session.user,
    )
    return {"status": "queued"}
```

Use `queue="short" | "default" | "long"` deliberately. Make the job idempotent
and use `frappe.publish_progress` / `frappe.publish_realtime` to update the UI.

---

## 16. Version/API drift across v13 → v14 → v15

**Mistake**

Generating code for the wrong Frappe version: assuming Python 3.6, using removed
APIs, or using a v15-only signature on a v13 site.

**Why it bites**

Significant changes across versions break "looks-correct" code:

- **Python baseline**: v13 ran on older Python; v14 requires Python 3.10+, v15
  raised it further. f-strings/walrus assumptions can fail on old sites.
- **Query Builder** (`frappe.qb`) was added in v13 and matured later — older code
  used raw `frappe.db.sql`.
- **Frontend**: v13 desk was jQuery-heavy; modern frontends (CRM/Helpdesk) use
  Vue 3 + Frappe UI + the `/api/method/frappe.client` and resource layer.
- **`bench` / build**: asset bundling moved to esbuild in v14; `bench build`
  flags changed.
- Various helper signatures (`get_list` permission handling, `frappe.db` methods)
  gained/changed keyword args.

**The fix**

Detect the version first and target it:

```bash
bench version            # bench + frappe + app versions
# or in console:
frappe.__version__
```

Read the changelog / release notes for the target major version before using a
helper you're unsure about, and prefer APIs that exist across the supported range.

---

## 17. Ignoring multi-tenancy / site context in background code

**Mistake**

Caching site-specific data in a process global (see #3), or running a script /
patch without an active site so `frappe.local.site` is unset.

**Why it bites**

A bench hosts many sites in one process pool. Code that assumes a single site
leaks data between tenants, and `frappe.db` calls without `frappe.init_site` /
`frappe.connect` fail or hit the wrong DB.

**The fix**

- Run scripts inside site context: `bench --site <site> execute <path>` or
  `bench --site <site> console`.
- In standalone scripts: `frappe.init(site=site); frappe.connect()` and
  `frappe.destroy()` at the end.
- Never cache one site's data where another site's worker could read it — use the
  site-scoped `frappe.cache()`, not module globals.

---

## 18. Editing standard DocTypes / customizing the wrong way

**Mistake**

Directly editing a standard (framework/ERPNext) DocType's JSON, or adding fields
by hacking core, instead of using Customize Form / Custom Fields.

**Why it bites**

Core changes are overwritten on `bench update` / app upgrade, and they put the
site into a "customized standard doctype" state that fights migrations. The
customization vanishes or causes migrate conflicts.

**The fix**

- Add fields via **Custom Field** (DocType: Custom Field) or **Customize Form**.
- Add behavior via your **own app's** `hooks.py` (`doc_events`, `override_doctype_class`).
- Keep customizations exportable as fixtures so they're versioned and reproducible:

```python
# hooks.py
fixtures = ["Custom Field", {"dt": "Property Setter", "filters": [["module", "=", "My App"]]}]
```

Never edit files under the `frappe` or `erpnext` app directories.

---

## 19. Forgetting `bench migrate` after schema changes

**Mistake**

Adding a field/DocType in JSON (or pulling a branch that did) and running the app
without migrating, then debugging "unknown column" errors.

**Why it bites**

DocType JSON describes schema, but the database table only changes on
`bench migrate` (which runs `frappe.model.sync` + patches). Until then the DB and
the code disagree.

**The fix**

After any schema change or `git pull`:

```bash
bench --site <site> migrate
bench build            # if assets/JS changed
bench --site <site> clear-cache
```

In dev, `bench --site <site> migrate` is also what runs your `patches.txt`
entries — write data migrations as idempotent patches, never as one-off console
edits.

---

## 20. Not writing tests that use real Frappe APIs / fixtures

**Mistake**

Mocking `frappe.get_doc` heavily, or testing controllers without `FrappeTestCase`
and a test site, so tests pass but real behavior breaks.

**Why it bites**

Over-mocking hides integration bugs (permission checks, validation, child-table
recalculation) — exactly the things Frappe code gets wrong. Tests that never
touch the DB give false confidence.

**The fix**

Use `frappe.tests.utils.FrappeTestCase` (v14+) / `unittest` on a test site, build
real docs, and let the framework run.

```python
from frappe.tests.utils import FrappeTestCase
import frappe

class TestSalesFlow(FrappeTestCase):
    def test_submit_sets_status(self):
        doc = frappe.get_doc({"doctype": "Sales Order", ...}).insert()
        doc.submit()
        self.assertEqual(doc.status, "To Deliver and Bill")
```

Run with `bench --site <site> run-tests --module my_app.tests.test_sales`.
`FrappeTestCase` auto-rolls-back per test, so tests stay isolated. Use real
DocType fields (see #10) — never invent fields in test setup either.
