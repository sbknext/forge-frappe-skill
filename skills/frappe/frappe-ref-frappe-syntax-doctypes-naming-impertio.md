---
title: Frappe Ref — Frappe Syntax: Doctypes: Naming (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'doctypes', 'reference', 'naming']
source: Impertio
---

# Naming Rules Reference

Complete reference for all DocType naming methods in Frappe.

## Overview

Every document in Frappe has a `name` field -- the primary key. The `naming_rule` property on the DocType controls how `name` is generated.

## All Naming Methods

### 1. Set by User

```json
{ "naming_rule": "Set by User" }
```

- User types the name manually on document creation.
- Name becomes the primary key and CANNOT be changed unless `allow_rename=1`.
- ALWAYS validate uniqueness -- Frappe raises `DuplicateEntryError` on collision.

### 2. Autoincrement

```json
{ "naming_rule": "Autoincrement" }
```

- Sequential integers: `1`, `2`, `3`, ...
- NEVER use in production -- deleted records leave gaps, names are not meaningful.
- NEVER switch naming scheme once documents exist (data corruption risk).

### 3. By Fieldname

```json
{
  "naming_rule": "By fieldname",
  "autoname": "field:employee_name"
}
```

- Uses the value of the specified field as the document name.
- The field value MUST be unique across all documents of this type.
- ALWAYS set `unique=1` on the source field.
- Good for: code-based records (item_code, currency abbreviation).

### 4. By Naming Series Field

```json
{
  "naming_rule": "By \"Naming Series\" field",
  "autoname": "naming_series:"
}
```

- Requires a `naming_series` field (Select type) on the DocType.
- Each option in the Select field is a pattern: `INV-.YYYY.-.#####`
- Users choose which series to use per document.
- ALWAYS include `.#####` (or more hashes) for the auto-increment portion.

**Series pattern syntax:**
| Token | Meaning | Example |
|-------|---------|---------|
| `.####` | Zero-padded counter | `0001`, `0002` |
| `.YYYY.` | 4-digit year | `2024` |
| `.YY.` | 2-digit year | `24` |
| `.MM.` | 2-digit month | `01`-`12` |
| `.DD.` | 2-digit day | `01`-`31` |
| `{fieldname}` | Field value | Dynamic prefix |

**Example series options:**
```
INV-.YYYY.-.#####
CN-.YYYY.-.#####
DN-.YYYY.-.#####
```
Output: `INV-2024-00001`, `CN-2024-00001`

### 5. Expression (Current Style)

```json
{
  "naming_rule": "Expression",
  "autoname": "PRE-.#####"
}
```

- Fixed prefix with auto-incrementing padded number.
- The `#` count determines zero-padding width.
- ALWAYS use at least 5 hashes (`#####`) for production systems.

**Examples:**
| autoname | Output |
|----------|--------|
| `PRE-.#####` | `PRE-00001` |
| `INV-.YYYY.-.######` | `INV-2024-000001` |
| `HR-EMP-.####` | `HR-EMP-0001` |

### 6. Expression (Old Style) -- Deprecated in v16

```json
{
  "naming_rule": "Expression (old style)",
  "autoname": "EXAMPLE-{MM}-{fieldname1}-{#####}"
}
```

**Supported tokens:**
| Token | Meaning |
|-------|---------|
| `{YYYY}` | 4-digit year |
| `{YY}` | 2-digit year |
| `{MM}` | Month (01-12) |
| `{DD}` | Day (01-31) |
| `{fieldname}` | Value of a document field |
| `{#####}` | Auto-increment counter |
| Static text | Included literally |

- NEVER use in new v15+ projects -- migrate to Expression or Naming Series.
- Will be removed in Frappe v16.

### 7. Random (Hash)

```json
{
  "naming_rule": "Random",
  "autoname": "hash"
}
```

- Generates a random 10-character alphanumeric string.
- Good for: records where the name has no business meaning.
- NEVER use if users need to reference records by name.

### 8. UUID

```json
{
  "naming_rule": "UUID"
}
```

- Standard UUID v4 format: `550e8400-e29b-41d4-a716-446655440000`.
- Available in Frappe v15+.
- Good for: API-first systems, external system integration.

### 9. By Script (Controller autoname)

```json
{
  "naming_rule": "By script"
}
```

Implement `autoname()` in the controller:

```python
# my_doctype.py
class MyDocType(Document):
    def autoname(self):
        prefix = f"P-{self.customer}-"
        self.name = make_autoname(prefix + ".#####")
```

**Utility functions:**
```python
from frappe.model.naming import make_autoname, getseries

# make_autoname: parse pattern and generate name
name = make_autoname("INV-.YYYY.-.#####")

# getseries: get next number in a series
name = getseries("INV-2024-", 5)  # Returns "INV-2024-00042" (next in sequence)
```

- The controller `autoname()` method takes PRIORITY over the DocType `naming_rule`.
- ALWAYS use `make_autoname()` or `getseries()` -- NEVER construct names with raw SQL counters.

## Document Naming Rules (Dynamic)

Frappe v14+ supports "Document Naming Rule" DocType for rule-based naming:

```python
# Managed via Document Naming Rule DocType, not code
# Fields: priority, conditions (filters), prefix, digits
```

- Higher `priority` value = applied first.
- Conditions filter which documents get which naming pattern.
- Overrides the DocType's default naming_rule.
- NEVER use for Child DocTypes -- they use random hash naming.

## Priority Order

1. **Document Naming Rule** (if conditions match)
2. **Controller `autoname()` method** (if defined)
3. **DocType `naming_rule` / `autoname`** (default)

## Amended Document Names

When a submitted document is amended:
- Original: `INV-2024-00001`
- First amendment: `INV-2024-00001-1`
- Second amendment: `INV-2024-00001-2`

NEVER override this behavior -- it maintains the audit trail for submittable documents.

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Using `Autoincrement` in production | Gaps after deletes, no meaning | Use Expression with prefix |
| Too few hashes (`##`) | Overflow at 100 records | Use at least `#####` |
| `field:` on non-unique field | `DuplicateEntryError` on insert | Add `unique=1` to the field |
| Changing naming scheme after data exists | Inconsistent names | Plan naming before go-live |
| Expression (Old Style) in v15+ | Will break in v16 | Migrate to Expression |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-doctypes/references/naming.md*
