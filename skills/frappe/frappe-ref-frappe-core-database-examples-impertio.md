---
title: Frappe Ref — Frappe Core: Database: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'database', 'reference', 'examples']
source: Impertio
---

# Database Examples

> Real-world patterns for common Frappe database operations. All examples use parameterized queries and follow best practices.

---

## Example 1: Document CRUD

### Create
```python
doc = frappe.get_doc({
    'doctype': 'Task',
    'subject': 'Review Sales Report',
    'status': 'Open',
    'priority': 'High',
    'description': 'Monthly sales review',
    'exp_start_date': frappe.utils.today(),
    'expected_time': 2
})
doc.insert()
# Framework auto-commits after request completes
```

### Read
```python
# Full document
doc = frappe.get_doc('Task', 'TASK-001')

# Specific fields only
subject, status = frappe.db.get_value('Task', 'TASK-001', ['subject', 'status'])

# Cached document (read-only, for stable data)
company = frappe.get_cached_doc('Company', 'My Company')
```

### Update
```python
# Via ORM — triggers validate, on_update hooks
doc = frappe.get_doc('Task', 'TASK-001')
doc.status = 'Working'
doc.save()

# Direct database — skips ALL validations
frappe.db.set_value('Task', 'TASK-001', 'status', 'Completed')
```

### Delete
```python
frappe.delete_doc('Task', 'TASK-001')
```

---

## Example 2: Filtered Lists with Pagination

```python
def get_open_tasks(page=0, page_size=50):
    """Fetch open tasks with pagination. ALWAYS paginate."""
    return frappe.get_all('Task',
        filters={
            'status': 'Open',
            'priority': ['in', ['High', 'Urgent']]
        },
        fields=['name', 'subject', 'assigned_to', 'exp_end_date'],
        order_by='exp_end_date asc',
        start=page * page_size,
        page_length=page_size
    )
```

### With OR Filters
```python
urgent_tasks = frappe.get_all('Task',
    filters={'docstatus': 0},
    or_filters={
        'priority': 'Urgent',
        'exp_end_date': ['<', frappe.utils.today()]
    },
    fields=['name', 'subject', 'priority', 'exp_end_date']
)
```

### Iterate All Records in Batches
```python
def process_all_invoices():
    """Process all submitted invoices in batches."""
    page = 0
    page_size = 100

    while True:
        batch = frappe.get_all('Sales Invoice',
            filters={'docstatus': 1},
            fields=['name', 'customer', 'grand_total'],
            start=page * page_size,
            page_length=page_size,
            order_by='creation asc'
        )
        if not batch:
            break

        for inv in batch:
            process_invoice(inv)

        page += 1
```

---

## Example 3: Aggregation with Query Builder

```python
from frappe.query_builder.functions import Count, Sum

Task = frappe.qb.DocType('Task')

stats = (
    frappe.qb.from_(Task)
    .select(
        Task.status,
        Count(Task.name).as_('count'),
        Sum(Task.expected_time).as_('total_hours')
    )
    .where(Task.docstatus == 0)
    .groupby(Task.status)
    .orderby(Count(Task.name), order='desc')
).run(as_dict=True)

for stat in stats:
    print(f"{stat.status}: {stat.count} tasks, {stat.total_hours}h total")
```

---

## Example 4: JOIN — Sales Report

### With Query Builder (preferred)
```python
from frappe.query_builder.functions import Sum, Count

SI = frappe.qb.DocType('Sales Invoice')
Customer = frappe.qb.DocType('Customer')

report = (
    frappe.qb.from_(SI)
    .inner_join(Customer).on(SI.customer == Customer.name)
    .select(
        Customer.customer_name,
        Customer.territory,
        Sum(SI.grand_total).as_('total_sales'),
        Count(SI.name).as_('invoice_count')
    )
    .where(SI.docstatus == 1)
    .where(SI.posting_date >= '2024-01-01')
    .groupby(Customer.name)
    .orderby(Sum(SI.grand_total), order='desc')
    .limit(10)
).run(as_dict=True)
```

### With Raw SQL (parameterized)
```python
results = frappe.db.sql("""
    SELECT
        c.customer_name,
        c.territory,
        SUM(si.grand_total) as total_sales,
        COUNT(si.name) as invoice_count
    FROM `tabSales Invoice` si
    INNER JOIN `tabCustomer` c ON si.customer = c.name
    WHERE si.docstatus = 1
    AND si.posting_date >= %(from_date)s
    GROUP BY c.name
    ORDER BY total_sales DESC
    LIMIT 10
""", {'from_date': '2024-01-01'}, as_dict=True)
```

---

## Example 5: Batch Processing — Avoid N+1

```python
def get_order_details(order_names):
    """Fetch order details with batch queries — NOT one query per order."""

    # One query for all orders
    orders = frappe.get_all('Sales Order',
        filters={'name': ['in', order_names]},
        fields=['name', 'customer', 'grand_total']
    )

    # One query for all customers
    customer_names = list(set(o.customer for o in orders))
    customers = {c.name: c for c in frappe.get_all('Customer',
        filters={'name': ['in', customer_names]},
        fields=['name', 'customer_name', 'territory']
    )}

    return [{
        'order': o.name,
        'total': o.grand_total,
        'customer': customers.get(o.customer, {}).get('customer_name', 'Unknown')
    } for o in orders]
```

### Bulk Update [v15+]
```python
def close_old_tasks():
    """Close all tasks older than 30 days."""
    old_tasks = frappe.get_all('Task',
        filters={
            'status': 'Open',
            'creation': ['<', frappe.utils.add_days(frappe.utils.today(), -30)]
        },
        pluck='name'
    )

    if old_tasks:
        updates = {name: {'status': 'Closed'} for name in old_tasks}
        frappe.db.bulk_update('Task', updates, chunk_size=100)
        frappe.db.commit()
```

---

## Example 6: Transaction with Savepoint

```python
def process_payment(invoice_name, amount):
    """Process payment with rollback on failure."""
    frappe.db.savepoint('before_payment')

    try:
        invoice = frappe.get_doc('Sales Invoice', invoice_name)

        pe = frappe.get_doc({
            'doctype': 'Payment Entry',
            'payment_type': 'Receive',
            'party_type': 'Customer',
            'party': invoice.customer,
            'paid_amount': amount,
            'received_amount': amount,
            'references': [{
                'reference_doctype': 'Sales Invoice',
                'reference_name': invoice_name,
                'allocated_amount': amount
            }]
        })
        pe.insert()
        pe.submit()

        return {'success': True, 'payment': pe.name}

    except Exception as e:
        frappe.db.rollback(save_point='before_payment')
        frappe.log_error(frappe.get_traceback(), f'Payment Error: {invoice_name}')
        return {'success': False, 'error': str(e)}
```

---

## Example 7: Cached Dashboard

```python
from frappe.utils.caching import redis_cache

@redis_cache(ttl=300)
def get_sales_dashboard(user):
    """Cached sales dashboard — refreshes every 5 minutes."""
    today = frappe.utils.today()
    month_start = frappe.utils.get_first_day(today)

    this_month = frappe.db.sql("""
        SELECT COALESCE(SUM(grand_total), 0) as total
        FROM `tabSales Invoice`
        WHERE docstatus = 1 AND posting_date >= %(month_start)s
    """, {'month_start': month_start}, as_dict=True)[0].total

    open_orders = frappe.db.count('Sales Order', {
        'docstatus': 1,
        'status': ['in', ['To Deliver and Bill', 'To Bill', 'To Deliver']]
    })

    return {
        'this_month_sales': this_month,
        'open_orders': open_orders
    }

# Invalidate when new invoice is submitted
class SalesInvoice(Document):
    def on_submit(self):
        get_sales_dashboard.clear_cache()
```

---

## Example 8: Background Job for Heavy Operations

```python
@frappe.whitelist()
def process_all_invoices():
    """Queue heavy processing as background job — NEVER block the request."""
    frappe.enqueue(
        'myapp.tasks.process_invoices_bg',
        queue='long',
        timeout=3600
    )
    return {'message': 'Processing started'}

def process_invoices_bg():
    """Background job: commit per batch to avoid losing progress."""
    invoices = frappe.get_all('Sales Invoice',
        filters={'status': 'Unpaid', 'docstatus': 1},
        fields=['name', 'customer'],
        page_length=0  # All records (OK in background job)
    )

    for i, inv in enumerate(invoices):
        try:
            send_reminder_email(inv.name)
        except Exception:
            frappe.log_error(frappe.get_traceback(), f'Reminder Error: {inv.name}')

        # Commit every 100 records in background jobs
        if (i + 1) % 100 == 0:
            frappe.db.commit()

    frappe.db.commit()
```

---

## Example 9: Existence Check and Conditional Create

```python
def ensure_task_exists(subject, project):
    """Create task only if it does not exist. ALWAYS use exists(), not try/except."""
    if not frappe.db.exists('Task', {'subject': subject, 'project': project}):
        doc = frappe.get_doc({
            'doctype': 'Task',
            'subject': subject,
            'project': project,
            'status': 'Open'
        })
        doc.insert(ignore_permissions=True)
        return doc.name

    return frappe.db.get_value('Task', {'subject': subject, 'project': project}, 'name')
```

---

## Example 10: Permission-Aware Data Access

```python
def get_user_invoices(user=None):
    """Fetch invoices respecting user permissions."""
    user = user or frappe.session.user

    if not frappe.has_permission('Sales Invoice', 'read', user=user):
        frappe.throw('No read permission for Sales Invoice', frappe.PermissionError)

    # get_list automatically applies User Permissions
    return frappe.db.get_list('Sales Invoice',
        fields=['name', 'customer', 'grand_total', 'status'],
        order_by='posting_date desc',
        page_length=100
    )
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-database/references/examples.md*
