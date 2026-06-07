---
title: Frappe Ref — Frappe Core: Cache: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'cache', 'reference', 'anti-patterns']
source: Impertio
---

# Cache Anti-Patterns

## AP-1: No TTL on External Data

**Wrong:**
```python
frappe.cache.set_value("exchange_rate_USD", fetch_rate("USD"))
# Cached forever — rate becomes stale after minutes
```

**Correct:**
```python
frappe.cache.set_value("exchange_rate_USD", fetch_rate("USD"), expires_in_sec=3600)
```

ALWAYS set `expires_in_sec` on cached values derived from external sources. Without TTL, stale data persists until manual deletion or Redis eviction.

---

## AP-2: Modifying Cached Documents

**Wrong:**
```python
settings = frappe.get_cached_doc("System Settings")
settings.enable_telemetry = 0  # CORRUPTS the cache for all readers
settings.save()
```

**Correct:**
```python
# Get a fresh, mutable copy
settings = frappe.get_doc("System Settings")
settings.enable_telemetry = 0
settings.save()
```

NEVER modify a document returned by `frappe.get_cached_doc()`. It returns a shared reference — mutations affect all subsequent cache reads.

---

## AP-3: Using frappe.clear_cache() as Routine Invalidation

**Wrong:**
```python
def on_item_update(doc, method):
    frappe.clear_cache()  # Nukes ALL cache for the entire site
```

**Correct:**
```python
def on_item_update(doc, method):
    frappe.cache.delete_value(f"item_data|{doc.name}")
    frappe.clear_document_cache("Item", doc.name)
```

NEVER use `frappe.clear_cache()` for targeted invalidation. It clears ALL cache keys (user permissions, boot info, document cache), causing a performance cliff as everything rebuilds.

---

## AP-4: Cache Stampede

**Wrong:**
```python
def get_report_data():
    data = frappe.cache.get_value("expensive_report")
    if data is None:
        # 100 concurrent requests all see cache miss
        # All 100 execute the expensive query simultaneously
        data = run_expensive_query()
        frappe.cache.set_value("expensive_report", data)
    return data
```

**Correct:**
```python
def get_report_data():
    # Use generator — only one caller computes, others wait
    return frappe.cache.get_value(
        "expensive_report",
        generator=run_expensive_query,
        expires_in_sec=300,
    )

# Or use locking for critical sections
def get_report_data():
    data = frappe.cache.get_value("expensive_report")
    if data is None:
        with frappe.lock("compute_report"):
            # Double-check after acquiring lock
            data = frappe.cache.get_value("expensive_report")
            if data is None:
                data = run_expensive_query()
                frappe.cache.set_value("expensive_report", data, expires_in_sec=300)
    return data
```

---

## AP-5: Unbounded Cache Keys

**Wrong:**
```python
def cache_user_data(user, page, filters):
    key = f"user_data|{user}|{page}|{filters}"
    # Creates a unique key for every filter combination
    # Keys accumulate indefinitely — memory bloat
    frappe.cache.set_value(key, compute_data(user, page, filters))
```

**Correct:**
```python
def cache_user_data(user, page, filters):
    key = f"user_data|{user}|{page}|{hash(str(sorted(filters.items())))}"
    frappe.cache.set_value(key, compute_data(user, page, filters), expires_in_sec=300)
    # TTL ensures keys expire even if never explicitly deleted
```

ALWAYS set TTL on dynamically generated cache keys. Without TTL, keys from old filter combinations accumulate and consume Redis memory indefinitely.

---

## AP-6: Caching Functions with Side Effects

**Wrong:**
```python
@redis_cache
def create_and_cache_report(report_name):
    doc = frappe.get_doc({"doctype": "Report Log", "report": report_name}).insert()
    return doc.name
# Second call returns cached name — does NOT create a new Report Log
# But caller expects a new document each time
```

**Correct:** NEVER use `@redis_cache` on functions that create documents, send emails, or perform any side effects. The decorator skips function execution on cache hits.

---

## AP-7: Forgetting to Release Locks

**Wrong:**
```python
frappe.lock("critical_section")
process_data()  # If this raises, lock is never released
frappe.unlock("critical_section")
```

**Correct:**
```python
with frappe.lock("critical_section"):
    process_data()  # Lock released even if exception occurs
```

ALWAYS use the `with` context manager for distributed locks. Manual `lock()`/`unlock()` pairs leak locks on exceptions.

---

## AP-8: Non-Hashable Arguments to @redis_cache

**Wrong:**
```python
@redis_cache
def get_filtered_items(filters):
    return frappe.get_all("Item", filters=filters)

# This fails — dicts are not hashable
get_filtered_items({"item_group": "Products"})
```

**Correct:**
```python
@redis_cache
def get_filtered_items(item_group, item_type=None):
    filters = {"item_group": item_group}
    if item_type:
        filters["item_type"] = item_type
    return frappe.get_all("Item", filters=filters)
```

ALWAYS use hashable arguments (strings, numbers, tuples) with `@redis_cache`. NEVER pass dicts, lists, or mutable objects.

---

## AP-9: Writing to Internal Cache Keys

**Wrong:**
```python
frappe.cache.set_value("doctype::meta::Item", custom_meta)
# Overwrites Frappe's internal DocType metadata cache
# Causes unpredictable behavior across the entire site
```

**Correct:** NEVER write to cache keys matching Frappe's internal patterns (`doctype::*`, `user_permissions::*`, `bootinfo::*`). ALWAYS prefix custom keys with your app name.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-cache/references/anti-patterns.md*
