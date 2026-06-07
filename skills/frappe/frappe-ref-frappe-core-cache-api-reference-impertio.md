---
title: Frappe Ref — Frappe Core: Cache: Api Reference (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'cache', 'reference', 'api-reference']
source: Impertio
---

# Cache API Reference

## frappe.cache — RedisWrapper

The `frappe.cache` object is an instance of `frappe.utils.redis_wrapper.RedisWrapper`, providing a Python interface to Redis with automatic key namespacing and pickle serialization.

### String Operations

```python
frappe.cache.set_value(
    key: str,                    # cache key
    val: Any,                    # value (pickled automatically)
    user: str = None,            # scope to specific user
    expires_in_sec: int = None,  # TTL in seconds (None = no expiry)
    shared: bool = False,        # True = omit site prefix
)

frappe.cache.get_value(
    key: str,                    # cache key
    generator: callable = None,  # called on miss, result is cached
    user: str = None,            # user scope
    expires: bool = False,       # deprecated — use expires_in_sec on set
    shared: bool = False,        # True = omit site prefix
    use_local_cache: bool = True,# check frappe.local.cache first
) -> Any                         # returns None if not found

frappe.cache.delete_value(
    keys: str | list[str],       # key or list of keys to delete
    user: str = None,            # user scope
    make_keys: bool = True,      # True = apply make_key() to keys
    shared: bool = False,        # True = omit site prefix
)

frappe.cache.delete_keys(
    key: str,                    # pattern with wildcard (e.g., "prefix*")
)                                # deletes all matching keys
```

### Hash Operations

```python
frappe.cache.hset(
    name: str,                   # hash name
    key: str,                    # field name
    value: Any,                  # field value (pickled)
    shared: bool = False,        # True = omit site prefix
)

frappe.cache.hget(
    name: str,                   # hash name
    key: str,                    # field name
    generator: callable = None,  # called on miss
    shared: bool = False,        # True = omit site prefix
) -> Any

frappe.cache.hgetall(
    name: str,                   # hash name
) -> dict                        # all fields deserialized

frappe.cache.hdel(
    name: str,                   # hash name
    keys: str | list[str],       # field(s) to delete
    shared: bool = False,
    pipeline = None,             # Redis pipeline for batching
)

frappe.cache.hdel_names(
    names: list[str],            # list of hash names
    key: str,                    # common field to delete from each
)

frappe.cache.hdel_keys(
    name_starts_with: str,       # hash name prefix (wildcard)
    key: str,                    # field to delete from matching hashes
)

frappe.cache.hexists(
    name: str,                   # hash name
    key: str,                    # field name
    shared: bool = False,
) -> bool
```

### Key Management

```python
frappe.cache.make_key(
    key: str,                    # raw key
    user: str = None,            # user scope
    shared: bool = False,        # True = omit site prefix
) -> str                         # returns namespaced key
```

Format: `{site_name}|{key}` (or `{site_name}|{user}|{key}` when user is set).

---

## @redis_cache Decorator

```python
from frappe.utils.caching import redis_cache

@redis_cache                      # no TTL — cached indefinitely
def my_function(arg1, arg2): ...

@redis_cache(ttl=300)             # expires after 300 seconds
def my_function(arg1, arg2): ...

# Clear cached results
my_function.clear_cache()         # removes all cached results for this function
```

**Availability:** v15+ only. Not available in v14.

Cache key is derived from: function module + function name + serialized arguments.

---

## Document Cache

```python
# Get document from cache (read-only, no permission check)
frappe.get_cached_doc(
    doctype: str,
    name: str = None,            # omit for Single DocTypes
) -> Document

# Invalidate cached document
frappe.clear_document_cache(
    doctype: str,
    name: str,
)

# Get single field with cache
frappe.db.get_value(
    doctype: str,
    name: str,
    fieldname: str,
    cache: bool = True,          # enable cache
) -> Any
```

---

## Distributed Locking

```python
# Context manager (recommended)
with frappe.lock(
    name: str,                   # lock name (must be unique per resource)
    timeout: int = None,         # seconds to wait for lock acquisition
):
    ...  # exclusive section

# Manual (use only when context manager is not possible)
frappe.lock(name: str, timeout: int = None)
frappe.unlock(name: str)
```

On timeout, raises `frappe.LockTimeoutError`.

---

## Site-Wide Cache Operations

```python
# Clear ALL cache for current site (use sparingly)
frappe.clear_cache()

# Clear cache for specific user
frappe.clear_cache(user="user@example.com")

# Clear cache for specific DocType
frappe.clear_cache(doctype="Item")
```

---

## frappe.local.cache

```python
# Per-request dict — not persisted to Redis
frappe.local.cache: dict

# Typical usage
if "my_key" not in frappe.local.cache:
    frappe.local.cache["my_key"] = expensive_computation()
result = frappe.local.cache["my_key"]
```

Lifetime: created at request start, garbage collected at request end. NEVER assume values persist across requests.

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-cache/references/api-reference.md*
