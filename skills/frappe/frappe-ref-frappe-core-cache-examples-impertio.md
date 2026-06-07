---
title: Frappe Ref — Frappe Core: Cache: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'cache', 'reference', 'examples']
source: Impertio
---

# Cache Implementation Examples

## 1. Cached API Response with TTL

```python
import frappe
import requests

def get_exchange_rate(from_currency, to_currency):
    """Fetch exchange rate with 1-hour cache."""
    key = f"exchange_rate|{from_currency}|{to_currency}"

    rate = frappe.cache.get_value(key)
    if rate is not None:
        return rate

    # Cache miss — fetch from API
    response = requests.get(
        f"https://api.example.com/rates/{from_currency}/{to_currency}"
    )
    rate = response.json()["rate"]

    frappe.cache.set_value(key, rate, expires_in_sec=3600)
    return rate
```

## 2. @redis_cache for Expensive Query

```python
from frappe.utils.caching import redis_cache

@redis_cache(ttl=600)
def get_top_customers(company, limit=10):
    """Get top customers by revenue — cached for 10 minutes."""
    return frappe.db.sql("""
        SELECT customer, SUM(grand_total) as total
        FROM `tabSales Invoice`
        WHERE company = %s AND docstatus = 1
        GROUP BY customer
        ORDER BY total DESC
        LIMIT %s
    """, (company, limit), as_dict=True)

# Usage
top = get_top_customers("My Company")

# Invalidate when needed
get_top_customers.clear_cache()
```

## 3. Hash-Based Configuration Cache

```python
def get_app_settings():
    """Cache app settings as a Redis hash for fast field-level access."""
    settings = frappe.cache.hgetall("myapp|settings")
    if settings:
        return settings

    # Load from database
    doc = frappe.get_doc("My App Settings")
    settings = {
        "api_key": doc.api_key,
        "max_retries": doc.max_retries,
        "timeout": doc.timeout,
        "enabled": doc.enabled,
    }

    for key, value in settings.items():
        frappe.cache.hset("myapp|settings", key, value)

    return settings

def invalidate_app_settings(doc, method):
    """Hook: clear settings cache on update."""
    frappe.cache.delete_value("myapp|settings")
```

## 4. Request-Scoped Cache

```python
def get_current_fiscal_year():
    """Avoid repeated DB calls within a single request."""
    cache_key = "current_fiscal_year"

    if cache_key not in frappe.local.cache:
        frappe.local.cache[cache_key] = frappe.db.get_value(
            "Fiscal Year",
            {"year_start_date": ("<=", frappe.utils.today()),
             "year_end_date": (">=", frappe.utils.today())},
            "name",
        )

    return frappe.local.cache[cache_key]
```

## 5. Distributed Lock for Batch Processing

```python
def process_daily_reports():
    """Ensure only one worker processes reports at a time."""
    with frappe.lock("daily_report_processing"):
        pending = frappe.get_all("Report Queue",
            filters={"status": "Pending"},
            limit=50,
        )
        for report in pending:
            generate_report(report.name)
            frappe.db.set_value("Report Queue", report.name, "status", "Completed")
        frappe.db.commit()
```

## 6. Cache Warming on Startup

```python
# In hooks.py
after_migrate = ["my_app.cache.warm_cache"]

# In my_app/cache.py
def warm_cache():
    """Pre-populate cache after migration."""
    # Cache frequently accessed documents
    for dt in ["Currency", "Company", "Fiscal Year"]:
        for name in frappe.get_all(dt, pluck="name"):
            frappe.get_cached_doc(dt, name)

    frappe.logger().info("Cache warmed successfully")
```

## 7. Conditional Cache Invalidation

```python
def on_item_price_update(doc, method):
    """Invalidate only affected cache entries."""
    # Clear specific item price cache
    frappe.cache.delete_value(f"item_price|{doc.item_code}|{doc.price_list}")

    # Clear aggregated cache that includes this item
    frappe.cache.delete_keys(f"pricing_summary|{doc.price_list}*")

    # Clear function decorator cache
    from my_app.pricing import get_item_price
    get_item_price.clear_cache()
```

## 8. Cache with Fallback Chain

```python
def get_setting(key):
    """Three-tier lookup: request cache → Redis → database."""
    # Tier 1: Request-scoped cache
    local_key = f"settings|{key}"
    if local_key in frappe.local.cache:
        return frappe.local.cache[local_key]

    # Tier 2: Redis cache
    value = frappe.cache.get_value(local_key)
    if value is not None:
        frappe.local.cache[local_key] = value
        return value

    # Tier 3: Database
    value = frappe.db.get_single_value("My Settings", key)
    frappe.cache.set_value(local_key, value, expires_in_sec=300)
    frappe.local.cache[local_key] = value
    return value
```

## 9. Lock with Timeout Handling

```python
import frappe

def sync_external_data():
    """Sync with timeout protection."""
    try:
        with frappe.lock("external_sync", timeout=10):
            # If lock acquired within 10 seconds, proceed
            perform_sync()
    except frappe.LockTimeoutError:
        frappe.logger().warning("External sync skipped — another process holds the lock")
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-cache/references/examples.md*
