---
title: Frappe Ref — Virtual DocType (Frappe docs)
category: frappe
tags: [frappe, virtual-doctype, controller, custom-datasource]
source: Frappe docs
---

# Virtual DocType

## When to use

- Expose external data (API, JSON/CSV, secondary DB) through standard Desk forms, list views, and `/api/resource` without creating a MariaDB table.
- Added in Frappe v13 — indistinguishable from normal DocTypes in the frontend.
- Aggregate read-only operational metrics or multi-source views where you still want permissions and list/form UX.

## Key concepts

- Set `"is_virtual": 1` in DocType JSON (or **Is Virtual DocType** on the DocType form).
- `frappe.db.*` calls only hit the site's MariaDB/Postgres — implement custom data access in the controller.
- Standard `/api/resource` CRUD works when you implement the required static/instance methods.
- Custom fields in JSON still define the schema mapping to your external payload shape.
- Keys in dicts returned by `get_list` must match DocType `fieldname`s exactly.

## DocType JSON (minimal)

```json
{
  "doctype": "DocType",
  "name": "External User",
  "module": "Integration",
  "is_virtual": 1,
  "fields": [
    {"fieldname": "id", "fieldtype": "Data", "in_list_view": 1},
    {"fieldname": "username", "fieldtype": "Data", "in_list_view": 1},
    {"fieldname": "email", "fieldtype": "Data", "in_list_view": 1}
  ],
  "permissions": [{"role": "System Manager", "read": 1, "write": 1, "create": 1, "delete": 1}]
}
```

For read-only external data, remove write/create/delete permissions or raise in mutation methods.

## Procedure / patterns

Override these controller methods for a custom backend:

```python
class VirtualDoctype(Document):
    @staticmethod
    def get_list(args):
        return [frappe._dict(doc) for doc in fetch_all()]

    @staticmethod
    def get_count(args):
        return len(fetch_all())

    def db_insert(self, *args, **kwargs):
        data = self.get_valid_dict(convert_dates_to_str=True)
        persist(data)

    def load_from_db(self):
        d = fetch_one(self.name)
        super(Document, self).__init__(d)

    def db_update(self, *args, **kwargs):
        self.db_insert(*args, **kwargs)

    def delete(self):
        remove(self.name)
```

Do not override `db_insert`/`db_update` on normal (non-virtual) DocTypes unless you know the implications.

## External API backing (pattern)

```python
import frappe
from frappe.model.document import Document

class ExternalUser(Document):
    def load_from_db(self):
        row = fetch_from_api(self.name)  # your HTTP/JSON client
        super(Document, self).__init__(row)

    def db_insert(self, *args, **kwargs):
        data = self.get_valid_dict(convert_dates_to_str=True)
        created = post_to_api(data)
        self.name = created["id"]

    def db_update(self, *args, **kwargs):
        put_to_api(self.name, self.get_valid_dict(convert_dates_to_str=True))

    def delete(self):
        delete_from_api(self.name)

    @staticmethod
    def get_list(args):
        return [frappe._dict(r) for r in list_from_api(args)]

    @staticmethod
    def get_count(args):
        return count_from_api(args)

    @staticmethod
    def get_value(fields, filters, **kwargs):
        """Called by frappe.db.get_value — return one field or dict for one record."""
        name = filters.get("name") if isinstance(filters, dict) else None
        row = fetch_one(name)
        if isinstance(fields, str):
            return row.get(fields)
        return {f: row.get(f) for f in fields}
```

`get_list(args)` receives pagination (`start`, `page_length`), `filters`, `fields`, and `order_by` — translate Frappe filter tuples to external API query params.

Use `frappe.make_get_request` / `make_post_request` in app code; in Server Scripts the same helpers are sandbox-safe.

## Best practices

- **Caching:** network calls in `get_list` slow list views — cache with `frappe.cache()` when data is not highly volatile.
- **Filter handling:** map Frappe filters like `['username', 'like', '%john%']` to API params; don't ignore filters silently.
- **Secrets:** store API keys in a separate Settings DocType using **Password** fieldtype — never hardcode in the controller.
- **Config:** store base URLs in a Settings Single, not inline in controller code.
- **Read-only backends:** leave `db_insert`/`db_update`/`delete` as `pass` or raise `frappe.PermissionError`.

## Version notes (v11–v16)

| Version | Notes |
|---------|--------|
| v13+ | Virtual DocType feature available |
| v14–v16 | `/api/resource` and Desk list/form parity maintained; test `get_list` filters/sort args |
| develop | Same controller contract — verify `get_stats` if using dashboard number cards |

## Gotchas

- Virtual DocTypes cannot use `frappe.db.sql` against their own data — you own persistence.
- Test list, count, insert, update, and delete paths; the framework calls different methods per operation.
- `get_stats` is required for report/dashboard aggregations (can return `{}` if unused).
- Handle API timeouts and map external errors to `frappe.throw` with clear messages.
- Permissions still apply — implement checks in controller methods if external ACL differs from Frappe roles.

## Source

- https://docs.frappe.io/framework/user/en/basics/doctypes/virtual-doctype
- https://github.com/vyogotech/frappe-apps-manager/blob/main/.cursor/skills/frappe-virtual-doctype/SKILL.md
