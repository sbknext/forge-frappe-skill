---
title: Frappe Ref — Frappe Core: Database: Caching Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'database', 'reference', 'caching-patterns']
source: Impertio
---

# Caching Patterns Reference

> Redis cache, @redis_cache decorator, document caching, and invalidation strategies. Verified against Frappe v14-v16 docs.

---

## Document Caching

### frappe.get_cached_doc(doctype, name)
Returns cached Document object. Falls back to database if not in cache.
```python
# ALWAYS use for read-only access to rarely-changing documents
company = frappe.get_cached_doc('Company', 'My Company')
settings = frappe.get_cached_doc('Selling Settings')

# NEVER use when:
# - You need to modify the document
# - You need guaranteed up-to-date data
# - You just saved changes to this document
```

### frappe.db.get_value with cache=True
Caches single field lookups.
```python
country = frappe.db.get_value('Company', 'My Company', 'country', cache=True)
```

---

## Redis Cache — Basic Operations

### Set / Get / Delete
```python
# Simple value
frappe.cache.set_value('my_key', 'my_value')
value = frappe.cache.get_value('my_key')

# Dict or list
frappe.cache.set_value('user_data', {'name': 'Admin', 'role': 'System Manager'})
data = frappe.cache.get_value('user_data')

# With expiry (seconds)
frappe.cache.set_value('temp_key', 'value', expires_in_sec=3600)   # 1 hour
frappe.cache.set_value('short_lived', 'value', expires_in_sec=300) # 5 min

# Delete
frappe.cache.delete_value('my_key')
```

### Site-Specific Keys
Frappe automatically prefixes all cache keys with the site name:
```python
# Site: site1.example.com
frappe.cache.set_value('key', 'value')
# Actual Redis key: site1.example.com|key
```
This means the same key on different sites stores separate values.

### Local Request Cache
Within a single request, repeated `get_value` calls return from in-memory cache without hitting Redis:
```python
frappe.cache.get_value('key')  # Redis call
frappe.cache.get_value('key')  # In-memory (no Redis call)
```

---

## Hash Operations

Use for complex objects where individual fields need independent updates.

```python
# Set individual fields
frappe.cache.hset('user|admin', 'name', 'Administrator')
frappe.cache.hset('user|admin', 'email', 'admin@example.com')

# Get single field
name = frappe.cache.hget('user|admin', 'name')

# Get all fields
user_data = frappe.cache.hgetall('user|admin')
# {'name': 'Administrator', 'email': 'admin@example.com'}

# Delete field
frappe.cache.hdel('user|admin', 'name')
```

---

## @redis_cache Decorator

### Basic Usage
```python
from frappe.utils.caching import redis_cache

@redis_cache
def expensive_calculation(param1, param2):
    # Heavy computation
    return param1 + param2

# First call: executes function
result = expensive_calculation(10, 20)

# Subsequent calls with same args: returns from cache
result = expensive_calculation(10, 20)
```

### With TTL (Time To Live)
```python
@redis_cache(ttl=300)    # 5 minutes
def get_dashboard_data(user):
    return calculate_dashboard(user)

@redis_cache(ttl=3600)   # 1 hour
def get_monthly_report(month, year):
    return generate_report(month, year)
```

### Cache Invalidation
```python
@redis_cache(ttl=300)
def get_user_stats(user):
    return calculate_stats(user)

# Manually clear cache
get_user_stats.clear_cache()
```

---

## TTL Guidelines

| Data type | TTL | Example |
|-----------|-----|---------|
| Static reference data | No TTL (manual invalidation) | Country list, currency codes |
| Configuration | 3600s (1 hour) | System settings, company defaults |
| Dashboard data | 300s (5 minutes) | Sales totals, task counts |
| Active session data | 60s (1 minute) | Online users, active sessions |

**RULE**: ALWAYS set a TTL unless you have explicit invalidation logic. Unbounded caches cause stale data bugs.

---

## Cache Invalidation Patterns

### Pattern 1: Invalidate on Document Update
```python
@redis_cache(ttl=3600)
def get_company_settings(company):
    return frappe.get_doc('Company', company)

class Company(Document):
    def on_update(self):
        get_company_settings.clear_cache()
```

### Pattern 2: Key-Based Invalidation
```python
def get_dashboard_data(user):
    cache_key = f"dashboard_{user}"
    data = frappe.cache.get_value(cache_key)
    if data:
        return data

    data = compute_dashboard(user)
    frappe.cache.set_value(cache_key, data, expires_in_sec=300)
    return data

class SalesInvoice(Document):
    def on_submit(self):
        # Invalidate dashboard for the invoice owner
        frappe.cache.delete_value(f"dashboard_{self.owner}")
```

### Pattern 3: Bulk Cache with Hash
```python
def cache_all_companies():
    companies = frappe.get_all('Company', fields=['name', 'country', 'default_currency'])
    for company in companies:
        frappe.cache.hset('companies', company.name, company)

def get_company_cached(name):
    data = frappe.cache.hget('companies', name)
    if not data:
        data = frappe.db.get_value('Company', name, ['name', 'country', 'default_currency'], as_dict=True)
        frappe.cache.hset('companies', name, data)
    return data
```

### Pattern 4: Graceful Degradation
```python
def get_data_with_fallback(key):
    try:
        data = frappe.cache.get_value(key)
        if data:
            return data
    except Exception:
        pass  # Redis down — fall back to database

    return fetch_from_database(key)
```

---

## Anti-Patterns

### NEVER: Cache Without Invalidation
```python
# ❌ Cache never cleared — stale data guaranteed
@redis_cache
def get_settings():
    return frappe.get_doc('My Settings')
```

### NEVER: Generic Cache Keys
```python
# ❌ Collisions and confusion
frappe.cache.set_value('data', result)

# ✅ Specific, namespaced keys
frappe.cache.set_value(f"sales_report_{user}_{month}", result)
```

### NEVER: Cache Large Objects Unnecessarily
```python
# ❌ Caching full document with all children
frappe.cache.set_value('invoice', frappe.get_doc('Sales Invoice', 'SINV-001'))

# ✅ Cache only what you need
frappe.cache.set_value('invoice_total', {'name': 'SINV-001', 'total': 50000})
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-database/references/caching-patterns.md*
