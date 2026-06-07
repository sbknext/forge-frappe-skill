---
title: Frappe Ref — Frappe Syntax: Whitelisted: Parameter Handling (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'whitelisted', 'reference', 'parameter-handling']
source: Impertio
---

# Parameter Handling Reference

How request parameters are received, parsed, and validated in whitelisted methods.

## Parameter Sources

### Via Function Arguments (Recommended)

```python
@frappe.whitelist()
def process_order(customer, items=None, include_tax=False):
    """Parameters are mapped from request data to function arguments."""
    return {"customer": customer, "items": items}
```

### Via frappe.form_dict

```python
@frappe.whitelist()
def dynamic_handler():
    """Direct access to all request parameters."""
    all_params = frappe.form_dict           # frappe._dict object
    customer = frappe.form_dict.get("customer")
    page = frappe.form_dict.get("page", 1)
    return {"customer": customer}
```

Use `frappe.form_dict` when:
- Parameter names are dynamic or unknown at development time
- You need to forward all parameters to another function
- You need to log the complete request

---

## Type Coercion Rules

ALL HTTP parameters arrive as **strings**. ALWAYS convert explicitly in v14. In v15+ with type annotations, Frappe auto-coerces.

### Manual Conversion (v14 Compatible)

```python
@frappe.whitelist()
def calculate(amount, quantity, enabled):
    # ALWAYS cast explicitly
    amount = float(amount)
    quantity = int(quantity)
    enabled = enabled in ("true", "1", True, 1)
    return amount * quantity if enabled else 0
```

### Automatic Validation [v15+]

```python
@frappe.whitelist()
def calculate(amount: float, quantity: int, enabled: bool = True) -> dict:
    # Frappe auto-validates and coerces via Pydantic
    # "10" -> int(10), "true" -> True, "3.14" -> float(3.14)
    return {"result": amount * quantity if enabled else 0}
```

### Type Coercion Behavior

| Client Sends | Python Annotation | Result |
|-------------|-------------------|--------|
| `"123"` | `int` | `123` |
| `"3.14"` | `float` | `3.14` |
| `"true"` / `"1"` | `bool` | `True` |
| `"false"` / `"0"` | `bool` | `False` |
| `"[1, 2, 3]"` | `list` | `[1, 2, 3]` |
| `'{"k":"v"}'` | `dict` | `{"k": "v"}` |
| `"hello"` | `str` | `"hello"` |
| `"abc"` | `int` | `FrappeTypeError` [v15+] |

---

## JSON Data Parsing

### frappe.parse_json (Recommended)

```python
@frappe.whitelist(methods=["POST"])
def create_items(data):
    """ALWAYS check if string before parsing."""
    if isinstance(data, str):
        data = frappe.parse_json(data)

    if not isinstance(data, dict):
        frappe.throw(_("Data must be a JSON object"))

    for item in data.get("items", []):
        process_item(item)
    return {"processed": True}
```

### Complex Nested Data

```python
@frappe.whitelist(methods=["POST"])
def create_order(order_data):
    if isinstance(order_data, str):
        order_data = frappe.parse_json(order_data)

    customer = order_data.get("customer")
    items = order_data.get("items", [])
    shipping = order_data.get("shipping", {})
    return {"received": True}
```

**Client side** — ALWAYS use `JSON.stringify()` for complex arguments:

```javascript
frappe.call({
    method: 'myapp.api.create_order',
    args: {
        order_data: JSON.stringify({
            customer: 'CUST-001',
            items: [{item_code: 'ITEM-001', qty: 5}],
            shipping: {method: 'express'}
        })
    }
});
```

---

## Validation Patterns

### Required Parameters

```python
@frappe.whitelist()
def strict_endpoint(customer, amount):
    if not customer:
        frappe.throw(_("Customer is required"), frappe.ValidationError)
    if amount is None:
        frappe.throw(_("Amount is required"), frappe.ValidationError)
    return {"valid": True}
```

### Multi-Field Required Check

```python
@frappe.whitelist()
def check_required(**kwargs):
    required = ["customer", "item_code", "qty"]
    missing = [f for f in required if not kwargs.get(f)]
    if missing:
        frappe.throw(
            _("Missing required fields: {0}").format(", ".join(missing)),
            frappe.ValidationError
        )
```

### Numeric Validation

```python
@frappe.whitelist()
def set_price(item, price):
    try:
        price = float(price)
    except (TypeError, ValueError):
        frappe.throw(_("Price must be a number"), frappe.ValidationError)
    if price < 0:
        frappe.throw(_("Price cannot be negative"), frappe.ValidationError)
    return {"price": price}
```

### List Input with Limit

```python
@frappe.whitelist()
def process_items(items):
    if isinstance(items, str):
        items = frappe.parse_json(items)
    if not isinstance(items, list):
        frappe.throw(_("Items must be a list"))

    MAX_ITEMS = 100
    if len(items) > MAX_ITEMS:
        frappe.throw(_("Maximum {0} items allowed").format(MAX_ITEMS))
    return {"count": len(items)}
```

---

## force_types Enforcement [v15+]

### Per-Method

```python
@frappe.whitelist(force_types=True)
def strict_api(customer: str, limit: int = 10) -> dict:
    # ALL params MUST have annotations — FrappeTypeError if missing
    return {"data": []}
```

### Per-App (hooks.py)

```python
# In hooks.py — enforces type annotations on ALL whitelisted methods in this app
require_type_annotated_api_methods = 1
```

When `force_types` is active:
- Parameters without annotations raise `FrappeTypeError`
- `self`, `cls`, `*args`, `**kwargs` are exempt
- Pydantic `TypeAdapter` handles validation with LRU cache (maxsize=2048)
- Forward references and string annotations are skipped

---

## Common Problems

| Problem | Cause | Solution |
|---------|-------|----------|
| `None` for expected value | Parameter not sent by client | Add default value or validate |
| String instead of dict/list | JSON not parsed | Use `frappe.parse_json()` |
| `TypeError` on arithmetic | No type cast from string | Cast with `int()` / `float()` |
| `FrappeTypeError` [v15+] | Annotation mismatch | Fix type annotation or cast |
| Unicode errors | Encoding issue | Use `frappe.safe_decode()` |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-whitelisted/references/parameter-handling.md*
