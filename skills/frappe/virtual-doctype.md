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

## Key concepts

- Check **Is Virtual DocType** on the DocType form.
- `frappe.db.*` calls only hit the site's MariaDB/Postgres — implement custom data access in the controller.
- Standard `/api/resource` CRUD works when you implement the required static/instance methods.

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

## Gotchas

- Virtual DocTypes cannot use `frappe.db.sql` against their own data — you own persistence.
- Test list, count, insert, update, and delete paths; the framework calls different methods per operation.
- `get_stats` is required for report/dashboard aggregations (can return `{}` if unused).

## Source

https://docs.frappe.io/framework/user/en/basics/doctypes/virtual-doctype
