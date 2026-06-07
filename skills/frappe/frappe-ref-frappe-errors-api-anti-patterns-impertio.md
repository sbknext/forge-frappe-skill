---
title: Frappe Ref — Frappe Errors: Api: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'api', 'reference', 'anti-patterns']
source: Impertio
---

# Anti-Patterns — API Error Handling

Common mistakes to avoid. Each entry follows: WRONG -> CORRECT -> WHY.

---

## 1. No Input Validation

```python
# WRONG — Uses inputs directly, crashes on bad data
@frappe.whitelist()
def process_order(customer, amount):
    order = frappe.get_doc({"doctype": "Sales Order", "customer": customer,
        "items": [{"item_code": "ITEM", "qty": 1, "rate": amount}]})
    order.insert()

# CORRECT — Validate everything first
@frappe.whitelist()
def process_order(customer, amount):
    if not customer:
        frappe.throw(_("Customer required"), exc=frappe.ValidationError)
    if not frappe.db.exists("Customer", customer):
        frappe.throw(_("Customer not found"), exc=frappe.DoesNotExistError)
    try:
        amount = float(amount)
        if amount <= 0:
            frappe.throw(_("Amount must be positive"), exc=frappe.ValidationError)
    except (ValueError, TypeError):
        frappe.throw(_("Invalid amount"), exc=frappe.ValidationError)
    # Now safe to proceed
```

**Why:** Unvalidated inputs cause cryptic errors and potential security issues.

---

## 2. Missing Error Callback in frappe.call

```javascript
// WRONG — No feedback on failure
frappe.call({
    method: "myapp.api.process",
    args: {data: data},
    callback: function(r) { frappe.show_alert("Done!"); }
    // No error handler!
});

// CORRECT — ALWAYS handle errors
frappe.call({
    method: "myapp.api.process",
    args: {data: data},
    callback: function(r) {
        if (r.message) frappe.show_alert({message: "Done!", indicator: "green"});
    },
    error: function(r) {
        frappe.msgprint({title: __("Error"),
            message: r._server_messages || __("Operation failed"), indicator: "red"});
    }
});
```

**Why:** Without error callback, users see no feedback when API calls fail.

---

## 3. Exposing Internal Errors to Users

```python
# WRONG — Leaks stack traces and internal details
@frappe.whitelist()
def calculate(item_code):
    try:
        return get_price(item_code)
    except Exception as e:
        frappe.throw(str(e))  # Exposes internals!

# CORRECT — Log internally, show friendly message
@frappe.whitelist()
def calculate(item_code):
    try:
        return get_price(item_code)
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Price Calculation Error")
        frappe.throw(_("Unable to calculate price. Please try again."))
```

**Why:** Internal errors may expose database structure, file paths, or credentials.

---

## 4. No Permission Check in Whitelisted Method

```python
# WRONG — Any logged-in user can delete any record
@frappe.whitelist()
def delete_record(doctype, name):
    frappe.delete_doc(doctype, name)

# CORRECT — Check permission first
@frappe.whitelist()
def delete_record(doctype, name):
    if not doctype or not name:
        frappe.throw(_("DocType and name required"), exc=frappe.ValidationError)
    frappe.has_permission(doctype, "delete", name, throw=True)
    frappe.delete_doc(doctype, name)
```

**Why:** `@frappe.whitelist()` makes the function callable by ANY logged-in user. Permission checks are mandatory.

---

## 5. Retrying 4xx Client Errors

```python
# WRONG — Retries all errors including 400, 401, 403
def call_api(url, data):
    for attempt in range(3):
        response = requests.post(url, json=data)
        if response.status_code != 200:
            time.sleep(2 ** attempt)
            continue
        return response.json()

# CORRECT — Only retry 5xx and 429, NEVER retry other 4xx
def call_api(url, data):
    for attempt in range(3):
        response = requests.post(url, json=data, timeout=30)
        if response.status_code == 200:
            return response.json()
        if response.status_code == 429:
            time.sleep(int(response.headers.get("Retry-After", 60)))
            continue
        if 400 <= response.status_code < 500:
            frappe.throw(f"Client error: {response.status_code}")
        if response.status_code >= 500:
            time.sleep(2 ** attempt)
            continue
```

**Why:** 4xx errors indicate client bugs — retrying them wastes resources and never succeeds.

---

## 6. Swallowing Errors Silently

```python
# WRONG — Silent failure, impossible to debug
@frappe.whitelist()
def sync_data():
    try:
        perform_sync()
    except Exception:
        pass  # No logging!

# CORRECT — ALWAYS log before handling
@frappe.whitelist()
def sync_data():
    try:
        perform_sync()
        return {"status": "success"}
    except Exception:
        frappe.log_error(frappe.get_traceback(), "Sync Error")
        frappe.throw(_("Sync failed. Please try again."))
```

**Why:** Silent failures make production debugging impossible. ALWAYS log with `frappe.log_error()`.

---

## 7. Hardcoded API Credentials

```python
# WRONG — Credentials in source code
def get_data():
    headers = {"Authorization": "Bearer sk_live_abc123"}
    return requests.get(url, headers=headers)

# CORRECT — Store in encrypted DocType field
def get_data():
    settings = frappe.get_single("API Settings")
    headers = {"Authorization": f"Bearer {settings.get_password('api_key')}"}
    return requests.get(url, headers=headers, timeout=30)
```

**Why:** Hardcoded credentials end up in version control and cannot be rotated.

---

## 8. No Timeout on External Requests

```python
# WRONG — Can hang forever
response = requests.get("https://api.example.com/data")

# CORRECT — ALWAYS set timeout
response = requests.get("https://api.example.com/data", timeout=30)
```

**Why:** Requests without timeout can hang indefinitely, blocking the worker process.

---

## 9. Wrong HTTP Status for Error Type

```python
# WRONG — Returns 200 with error in body
@frappe.whitelist()
def get_item(name):
    if not frappe.db.exists("Item", name):
        return {"error": "Not found"}  # Client gets 200!

# CORRECT — Use proper exception for 404
@frappe.whitelist()
def get_item(name):
    if not frappe.db.exists("Item", name):
        frappe.throw(_("Item not found"), exc=frappe.DoesNotExistError)
    return frappe.get_doc("Item", name)
```

**Why:** Proper HTTP status codes let clients handle different error types correctly.

---

## 10. Not Parsing JSON Input

```python
# WRONG — Crashes if items arrives as JSON string
@frappe.whitelist()
def update_items(items):
    for item in items:  # TypeError if items is a string
        update_item(item)

# CORRECT — Handle both string and parsed input
@frappe.whitelist()
def update_items(items):
    if isinstance(items, str):
        try:
            items = frappe.parse_json(items)
        except Exception:
            frappe.throw(_("Invalid JSON"), exc=frappe.ValidationError)
    if not isinstance(items, list):
        frappe.throw(_("Items must be a list"), exc=frappe.ValidationError)
    for item in items:
        update_item(item)
```

**Why:** frappe.call sends complex args as JSON strings. ALWAYS handle both formats.

---

## 11. No Loading Indicator for Long Operations

```javascript
// WRONG — UI appears frozen
async function processLarge() {
    const result = await frappe.xcall("myapp.api.process_large");
    console.log(result);
}

// CORRECT — Show feedback during processing
async function processLarge() {
    try {
        frappe.freeze(__("Processing..."));
        const result = await frappe.xcall("myapp.api.process_large");
        frappe.show_alert({message: __("Complete!"), indicator: "green"});
        return result;
    } catch (e) {
        frappe.msgprint({title: __("Error"), message: e.message, indicator: "red"});
    } finally {
        frappe.unfreeze();
    }
}
```

**Why:** Users need visual feedback during operations longer than 1 second.

---

## 12. No Rate Limit Handling

```python
# WRONG — Rapid-fire requests hit rate limits
for record in records:
    call_external_api(record)  # Will get 429 eventually

# CORRECT — Throttle and handle 429
for i, record in enumerate(records):
    try:
        call_external_api(record)
    except RateLimitError as e:
        time.sleep(e.retry_after or 60)
        call_external_api(record)
    if i % 10 == 0:
        time.sleep(1)  # Throttle proactively
```

**Why:** All APIs have rate limits. Exceeding them causes cascading failures.

---

## 13. Inconsistent Error Response Format

```python
# WRONG — Every endpoint returns errors differently
def ep1(): return {"error": True, "msg": "Failed"}
def ep2(): return {"success": False, "message": "Error"}
def ep3(): frappe.throw("Something wrong")

# CORRECT — Consistent: use frappe.throw with exception class
def ep1(): frappe.throw(_("Failed"), exc=frappe.ValidationError)
def ep2(): frappe.throw(_("Error"), exc=frappe.ValidationError)
def ep3(): frappe.throw(_("Something wrong"), exc=frappe.ValidationError)
```

**Why:** Consistent error format simplifies client-side error handling across all endpoints.

---

## 14. Mixing Authentication Token Formats

```python
# WRONG — Bearer with API key:secret
headers = {"Authorization": f"Bearer {api_key}:{api_secret}"}

# CORRECT — "token" keyword for API key:secret
headers = {"Authorization": f"token {api_key}:{api_secret}"}

# CORRECT — "Bearer" for OAuth tokens only
headers = {"Authorization": f"Bearer {oauth_access_token}"}
```

**Why:** Frappe uses `token` prefix for API keys and `Bearer` for OAuth. Mixing them gives 401.

---

## 15. Processing Webhooks Synchronously

```python
# WRONG — Slow processing causes sender timeouts and retries
@frappe.whitelist(allow_guest=True)
def webhook():
    data = frappe.parse_json(frappe.request.data)
    process_heavy_operation(data)  # Takes 30+ seconds
    return {"status": "ok"}

# CORRECT — Return 200 immediately, process in background
@frappe.whitelist(allow_guest=True)
def webhook():
    data = frappe.parse_json(frappe.request.data)
    frappe.enqueue("myapp.tasks.process_webhook", queue="short", data=data)
    return {"status": "accepted"}
```

**Why:** Webhook senders expect fast responses (< 5s). Slow responses trigger retries.

---

## Checklist Before Deploying API Endpoints

- [ ] All inputs validated before use
- [ ] Permission checks in every whitelisted method
- [ ] Specific exception types (ValidationError, PermissionError, DoesNotExistError)
- [ ] Error callback in every frappe.call
- [ ] Internal errors logged, not exposed to users
- [ ] No hardcoded credentials
- [ ] Timeouts on all external requests
- [ ] Rate limiting handled (respect Retry-After)
- [ ] JSON inputs parsed safely (string or parsed)
- [ ] Network errors handled separately from server errors
- [ ] Loading indicators for long operations
- [ ] Consistent error response format
- [ ] Token format correct (token vs Bearer)
- [ ] CSRF token included for session-based auth
- [ ] Webhooks processed asynchronously

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-api/references/anti-patterns.md*
