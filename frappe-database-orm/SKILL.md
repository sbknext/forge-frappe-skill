---
name: frappe-database-orm
description: >
  Query and mutate data in Frappe safely — get_doc/get_all/get_list, frappe.db.get_value/set_value,
  parameterized frappe.db.sql, Query Builder (frappe.qb), transactions, bulk ops, and performance.
  Use when reading or writing data, optimizing slow queries, or avoiding SQL injection. Trigger on:
  "frappe.db.sql", "frappe.get_all", "query builder", "frappe.qb", "bulk update", "N+1", "db transaction",
  "get_value set_value".
---

# Frappe Database / ORM

Prefer the ORM; drop to SQL only when needed, and only parameterized.

## Reads

```python
frappe.get_doc("Sales Order", "SO-0001")                    # full document (all children)
frappe.db.get_value("Sales Order", "SO-0001", "status")     # one field, fast
frappe.db.get_value("Sales Order", {"customer": "ACME"}, ["name", "status"], as_dict=True)
frappe.get_all("Sales Order",
    filters={"status": "Draft", "grand_total": [">", 1000]},
    fields=["name", "customer", "grand_total"],
    order_by="creation desc", limit=50)
frappe.get_list(...)   # same, but enforces the current user's permissions
```
`get_all` ignores user permissions (use server-side); `get_list` enforces them (use for user-facing).

## Writes

```python
frappe.db.set_value("Sales Order", "SO-0001", "status", "Closed")          # single field, no hooks
doc = frappe.get_doc("Sales Order", "SO-0001"); doc.status = "Closed"; doc.save()  # runs validate/hooks
frappe.db.set_value("Sales Order", {"customer": "ACME"}, "hold", 1)        # by filter
```
`db.set_value` skips controller hooks/validation — fast but bypasses business logic. `doc.save()`
runs the full lifecycle. Choose deliberately.

## Raw SQL — parameterized ONLY

```python
# RIGHT — %s placeholders
rows = frappe.db.sql("""
    SELECT customer, SUM(grand_total) AS total
    FROM `tabSales Order` WHERE status = %s AND company = %s
    GROUP BY customer
""", ("Draft", company), as_dict=True)

# WRONG — string interpolation = SQL injection
frappe.db.sql(f"... WHERE name = '{name}'")     # NEVER
```
Escape identifiers/values with `frappe.db.escape(value)` when you must build dynamically.

## Query Builder (frappe.qb)

```python
from frappe.query_builder import DocType
from frappe.query_builder.functions import Sum

SO = DocType("Sales Order")
q = (frappe.qb.from_(SO)
     .select(SO.customer, Sum(SO.grand_total).as_("total"))
     .where(SO.status == "Draft").groupby(SO.customer))
data = q.run(as_dict=True)
```
Composable and injection-safe; preferred over raw SQL for dynamic queries. All `frappe.qb` queries are parameterized by default — use `.run(as_dict=True)` to execute. Inspect SQL with `str(query)` or `.get_sql()`. Prefer passing query objects to `.run()` over raw `frappe.db.sql(query)` (the latter skips parameterization).

Source: https://docs.frappe.io/framework/user/en/api/query-builder

## Transactions

```python
frappe.db.savepoint("before_bulk")
try:
    ...
    frappe.db.commit()
except Exception:
    frappe.db.rollback(save_point="before_bulk")
```
Web requests auto-commit on success / rollback on exception. Background jobs must commit explicitly.

## Bulk update (v14+)

Update many documents in one SQL statement — skips controller hooks/validation:

```python
frappe.db.bulk_update(
    "Task",
    {
        "TASK-0001": {"status": "Closed"},
        "TASK-0002": {"status": "Open"},
    },
    chunk_size=200,
    update_modified=True,
)
```

Use when you need speed and know lifecycle hooks are not required.

## v16 `get_list` / `run=False` change

Prior to v16, `run=False` on `get_list` returned a SQL string. From v16 it returns a Query Builder object — call `.get_sql()` for the string or mutate before `.run()`.

v16 also prefers dict-style aggregate fields:

```python
frappe.db.get_list("Task",
    fields=[{"COUNT": "name", "as": "count"}, "status"],
    group_by="status")
```

## Transaction hooks (v15+)

Register callbacks tied to commit/rollback (useful for filesystem side-effects):

```python
frappe.db.after_commit.add(my_func)
frappe.db.after_rollback.add(cleanup_func)
```

Web `POST`/`PUT` auto-commit on success; background jobs must call `frappe.db.commit()` explicitly.

## frappe.qb.get_query (preferred for complex queries)

Modern query builder — auto-joins Link/child fields, supports aggregations and OR filters:

```python
query = frappe.qb.get_query("Sales Order",
    fields=["name", "customer.customer_name as customer_name",
            {"items": ["item_code", "qty"]}],
    filters={"docstatus": 1, "items.item_code": "ITEM-001"},
    limit=50)
rows = query.run(as_dict=True)
```

Use `ignore_permissions=False` when the query serves user-facing data. Use `for_update=True` for row locking.

## Performance

- Avoid **N+1**: fetch with one `get_all(filters={"parent": ["in", names]})` instead of looping `get_doc`.
- Select only the `fields` you need — never pull whole docs to read one column.
- Add DB indexes on hot filter/join columns (`search_index: 1` on the field, or a migration).
- For large bulk writes use `frappe.db.bulk_insert` / batched `set_value`, and commit periodically.

## Rules

- Never f-string/`%`-format values into `frappe.db.sql` — always `%s` params.
- `get_all` vs `get_list` — know which one applies permissions.
- `db.set_value`/`db.delete` skip hooks; use `doc.save()`/`doc.delete()` when lifecycle matters.
- Web requests auto-commit on successful POST/PUT; GET does not commit. Background jobs auto-commit on success.
- Rarely call `frappe.db.commit()` manually — exception: flush writes before `enqueue_after_commit` reads them.

## From Frappe docs

- Database API (`get_list`, `get_all`, `bulk_update`, transaction model): https://docs.frappe.io/framework/user/en/api/database
- Query Builder `frappe.qb.get_query`: https://docs.frappe.io/framework/get_query
