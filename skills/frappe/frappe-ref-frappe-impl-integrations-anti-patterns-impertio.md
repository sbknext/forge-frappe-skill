---
title: Frappe Ref — Frappe Impl: Integrations: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'impl', 'integrations', 'reference', 'anti-patterns']
source: Impertio
---

# Integration Anti-Patterns

## Anti-Pattern 1: External API Call in validate

```python
# WRONG — blocks save, causes timeout on slow APIs
def validate(doc, method=None):
    response = requests.post("https://api.example.com/check", json={"id": doc.name})
    if response.json()["status"] != "valid":
        frappe.throw("External validation failed")
```

**Fix**: Move external calls to `on_update` with `frappe.enqueue()`:

```python
def on_update(doc, method=None):
    frappe.enqueue(
        "myapp.integrations.validate_external",
        doc_name=doc.name,
        queue="short",
        enqueue_after_commit=True
    )
```

## Anti-Pattern 2: No Timeout on External Requests

```python
# WRONG — hangs indefinitely if external service is down
response = requests.get("https://api.example.com/data")
```

**Fix**: ALWAYS set timeout:

```python
response = requests.get("https://api.example.com/data", timeout=30)
```

## Anti-Pattern 3: No Error Logging for Failed API Calls

```python
# WRONG — silent failure, impossible to debug
try:
    response = requests.post(url, json=data)
except:
    pass
```

**Fix**: Log ALL integration errors:

```python
try:
    response = requests.post(url, json=data, timeout=30)
    response.raise_for_status()
except requests.exceptions.RequestException as e:
    frappe.log_error(
        f"API call to {url} failed: {e}\nPayload: {data}",
        "Integration Error"
    )
    raise
```

## Anti-Pattern 4: Webhook Without HMAC Verification

```python
# WRONG — anyone can send fake webhooks to your endpoint
@frappe.whitelist(allow_guest=True)
def webhook_handler():
    data = frappe.parse_json(frappe.request.get_data(as_text=True))
    process(data)  # No verification!
```

**Fix**: ALWAYS verify webhook signatures:

```python
@frappe.whitelist(allow_guest=True)
def webhook_handler():
    verify_hmac_signature(frappe.request)  # Verify FIRST
    data = frappe.parse_json(frappe.request.get_data(as_text=True))
    process(data)
```

## Anti-Pattern 5: No Idempotency on Webhook Receiver

```python
# WRONG — duplicate webhooks create duplicate records
@frappe.whitelist(allow_guest=True)
def handle_payment():
    data = frappe.parse_json(frappe.request.get_data(as_text=True))
    frappe.get_doc({"doctype": "Payment Entry", ...}).insert()
```

**Fix**: Check for duplicates using external transaction ID:

```python
@frappe.whitelist(allow_guest=True)
def handle_payment():
    data = frappe.parse_json(frappe.request.get_data(as_text=True))
    if frappe.db.exists("Payment Entry", {"custom_external_id": data["txn_id"]}):
        return {"status": "already_processed"}
    frappe.get_doc({"doctype": "Payment Entry", "custom_external_id": data["txn_id"], ...}).insert()
```

## Anti-Pattern 6: Storing Secrets in Code

```python
# WRONG — secrets visible in version control
API_KEY = "sk_live_abc123"
response = requests.get(url, headers={"Authorization": f"Bearer {API_KEY}"})
```

**Fix**: Store secrets in site_config or use Frappe's password field:

```python
api_key = frappe.utils.password.get_decrypted_password(
    "My Integration Settings", "My Integration Settings", "api_key"
)
# Or from site_config:
api_key = frappe.conf.get("my_integration_api_key")
```

## Anti-Pattern 7: Importing Too Many Rows at Once

```python
# WRONG — memory exhaustion, timeout
# Uploading 50,000 row CSV into Data Import
```

**Fix**: Split into batches of 5,000 or fewer. For programmatic imports, commit every 100 rows:

```python
for i, row in enumerate(rows):
    doc = frappe.get_doc({"doctype": "Item", **row})
    doc.insert()
    if i % 100 == 0:
        frappe.db.commit()
frappe.db.commit()
```

## Anti-Pattern 8: OAuth Client Secret in Frontend

```javascript
// WRONG — client_secret exposed to all users
fetch("/api/method/frappe.integrations.oauth2.get_token", {
    body: JSON.stringify({
        client_secret: "my_secret_123"  // Visible in DevTools!
    })
});
```

**Fix**: Token exchange MUST happen server-side. Use a whitelisted method as proxy.

## Anti-Pattern 9: Not Using enqueue_after_commit

```python
# WRONG — enqueued job may run before transaction commits
def on_update(doc, method=None):
    frappe.enqueue("myapp.sync.push", doc_name=doc.name)
    # Job might start before doc changes are committed!
```

**Fix**: Use `enqueue_after_commit=True`:

```python
def on_update(doc, method=None):
    frappe.enqueue("myapp.sync.push", doc_name=doc.name, enqueue_after_commit=True)
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/impl/frappe-impl-integrations/references/anti-patterns.md*
