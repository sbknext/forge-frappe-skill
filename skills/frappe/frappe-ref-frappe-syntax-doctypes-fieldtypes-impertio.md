---
title: Frappe Ref — Frappe Syntax: Doctypes: Fieldtypes (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'doctypes', 'reference', 'fieldtypes']
source: Impertio
---

# Fieldtype Reference

Complete reference for all Frappe fieldtypes. Organized by category.

## Data Entry Fields

| Fieldtype | Stores | DB Column | `options` Field |
|-----------|--------|-----------|-----------------|
| Data | Text, max 140 chars | VARCHAR(140) | Validation: "Name", "Email", "Phone", "URL", "Barcode", "IBAN" |
| Small Text | Short multi-line text | TEXT | _(none)_ |
| Text | Multi-line text | LONGTEXT | _(none)_ |
| Long Text | Unlimited text | LONGTEXT | _(none)_ |
| Text Editor | Rich text (WYSIWYG HTML) | LONGTEXT | _(none)_ |
| Markdown Editor | Markdown with preview | LONGTEXT | _(none)_ |
| HTML Editor | Raw HTML editing | LONGTEXT | _(none)_ |
| Code | Code with syntax highlighting | LONGTEXT | Language: "Python", "PythonExpression", "JavaScript", "HTML", "CSS", "JSON", "Jinja" |
| Password | Encrypted sensitive data | VARCHAR(140) | _(none)_ |
| Read Only | Non-editable display field | VARCHAR(140) | _(none)_ |
| Phone | Phone number input | VARCHAR(140) | _(none)_ -- Data subtype with phone formatting |
| Autocomplete | Text with autocomplete | VARCHAR(140) | _(none)_ |

### Code Field Extra Properties

| Property | Purpose |
|----------|---------|
| `options` | Syntax highlighting language |
| `wrap` | Enable text wrapping (bool) |
| `max_lines` | Maximum editor height |
| `min_lines` | Minimum editor height |

## Numeric Fields

| Fieldtype | Stores | DB Column | `options` Field |
|-----------|--------|-----------|-----------------|
| Int | Whole number | INT(11) | _(none)_ |
| Float | Decimal number (up to 9 places) | DECIMAL(21,9) | _(none)_ |
| Currency | Money value (up to 6 decimals) | DECIMAL(21,9) | Currency field name for symbol display |
| Percent | Percentage value | DECIMAL(21,9) | _(none)_ |
| Rating | Star rating (0-1 stored) | DECIMAL(21,9) | Number 3-10 for star count; supports half ratings |
| Duration | Time duration | DECIMAL(21,9) | "Hide Days", "Hide Seconds" toggles |

### Currency Field `options` Behavior

- If `options` points to another field (e.g. `currency`), the value of that field determines the currency symbol displayed.
- If `options` is empty, the system default currency is used.
- ALWAYS set `options` on Currency fields to display the correct symbol.

## Date and Time Fields

| Fieldtype | Stores | DB Column | `options` Field |
|-----------|--------|-----------|-----------------|
| Date | Calendar date | DATE | _(none)_ |
| Datetime | Date + time | DATETIME(6) | _(none)_ |
| Time | Time only | TIME(6) | _(none)_ |

## Relationship Fields

| Fieldtype | Stores | DB Column | `options` Field |
|-----------|--------|-----------|-----------------|
| Link | FK to another DocType | VARCHAR(140) | **Target DocType name** (REQUIRED) |
| Dynamic Link | FK to any DocType | VARCHAR(140) | **Fieldname** containing the target DocType |
| Table | Child table rows | _(separate table)_ | **Child DocType name** (REQUIRED, must have `istable=1`) |
| Table MultiSelect | Multi-select link rows | _(separate table)_ | **Child DocType name** (child must have a single Link field) |

### Link Field Behavior

- Displays a search input with autocomplete.
- `options` MUST be the exact DocType name (case-sensitive).
- ALWAYS ensure the target DocType exists before adding a Link field.

### Dynamic Link Behavior

- `options` points to ANOTHER field (Select or Data) in the same DocType.
- That field holds the DocType name at runtime.
- Example: field `party_type` (Select with "Customer\nSupplier") + field `party` (Dynamic Link with `options=party_type`).
- ALWAYS pair a Dynamic Link with a corresponding type-selector field.

### Table MultiSelect vs Table

- Table MultiSelect child DocType typically has ONE Link field.
- UI renders as a tag/pill selector instead of a full grid.
- NEVER use Table MultiSelect for child DocTypes with many editable fields.

## Selection Fields

| Fieldtype | Stores | DB Column | `options` Field |
|-----------|--------|-----------|-----------------|
| Select | One choice from dropdown | VARCHAR(140) | Newline-separated values (first line = default if no `default` set) |
| Check | Boolean (0 or 1) | TINYINT(1) | _(none)_ -- set `default=1` for checked by default |

### Select Field Options Format

```
Draft
Submitted
Cancelled
```

- First option is shown by default unless `default` is set.
- NEVER include blank line unless you want an empty option.
- Options are stored as the string value, not an index.

## File and Media Fields

| Fieldtype | Stores | DB Column | `options` Field |
|-----------|--------|-----------|-----------------|
| Attach | File path/URL | VARCHAR(140) | _(none)_ |
| Attach Image | Image file path/URL | VARCHAR(140) | _(none)_ |
| Signature | Base64 signature data | LONGTEXT | _(none)_ |
| Image | Display-only image | _(no storage)_ | Fieldname of Attach field to display |
| Barcode | Barcode data | LONGTEXT | _(none)_ |

### Image vs Attach Image

- `Attach Image` stores a file and shows upload widget.
- `Image` is DISPLAY ONLY -- it renders the image from another Attach field.
- `Image` field `options` = the fieldname of the Attach/Attach Image field.

## Special Data Fields

| Fieldtype | Stores | DB Column | `options` Field |
|-----------|--------|-----------|-----------------|
| Color | Hex color string | VARCHAR(140) | _(none)_ |
| Geolocation | GeoJSON feature collection | LONGTEXT | _(none)_ |
| JSON | Arbitrary JSON | JSON / LONGTEXT | _(none)_ |
| Icon | Icon name string | VARCHAR(140) | _(none)_ |

## Layout and UI Fields (No Data Storage)

| Fieldtype | Purpose | `options` Field |
|-----------|---------|-----------------|
| Section Break | Horizontal section divider | Label text (optional) |
| Column Break | Multi-column layout within section | _(none)_ |
| Tab Break | Form tab divider | Tab label |
| Heading | Display heading text | _(none)_ |
| HTML | Render static HTML content | HTML content string |
| Button | Actionable button | _(none)_ |

### Tab Break Rules

- If the first field is NOT a Tab Break, Frappe auto-creates a "Details" tab.
- ALWAYS start with a Tab Break if you want custom tab naming from the first tab.
- Each Tab Break starts a new tab; all fields until the next Tab Break belong to it.

### Button Field Properties

| Property | Purpose |
|----------|---------|
| `options` | _(none)_ |
| `btn_size` | "xs", "sm", "lg" |

- Button clicks are handled via client script (`frappe.ui.form.on`).
- Buttons store NO data in the database.

## Fieldtype Selection Guide

```
What kind of data?
├─ Text?
│  ├─ Short (< 140 chars) → Data
│  ├─ Formatted/Rich → Text Editor
│  ├─ Code → Code (set options to language)
│  ├─ Multi-line plain → Small Text or Text
│  └─ Very long → Long Text
├─ Number?
│  ├─ Whole number → Int
│  ├─ Decimal → Float
│  ├─ Money → Currency
│  └─ Percentage → Percent
├─ Date/Time?
│  ├─ Date only → Date
│  ├─ Date + Time → Datetime
│  └─ Time only → Time
├─ Reference to another record?
│  ├─ Fixed DocType → Link
│  ├─ Variable DocType → Dynamic Link
│  ├─ Multiple rows → Table
│  └─ Multiple selections → Table MultiSelect
├─ Yes/No → Check
├─ One of several options → Select
├─ File upload → Attach (or Attach Image for images)
└─ Layout only → Section Break / Column Break / Tab Break
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-doctypes/references/fieldtypes.md*
