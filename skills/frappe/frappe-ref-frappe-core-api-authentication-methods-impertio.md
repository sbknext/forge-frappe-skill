---
title: Frappe Ref — Frappe Core: Api: Authentication Methods (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'api', 'reference', 'authentication-methods']
source: Impertio
---

# Authentication Methods Reference

> All authentication methods for Frappe API access.

---

## 1. Token Based Authentication (RECOMMENDED)

Most common method for server-to-server integrations. Available since Frappe v11.

### Generate API Keys

**Via UI:**
1. User list > Open user > Settings tab
2. Expand "API Access" section
3. Click "Generate Keys"
4. Copy API Secret immediately — shown only once

**Via CLI:**
```bash
bench execute frappe.core.doctype.user.user.generate_keys --args ['api_user@example.com']
```

**Via RPC:**
```bash
POST /api/method/frappe.core.doctype.user.user.generate_keys
{"user": "api_user@example.com"}
```

### Token Format

```
Authorization: token <api_key>:<api_secret>
```

### Python Example

```python
import requests
import os

API_KEY = os.environ.get('FRAPPE_API_KEY')
API_SECRET = os.environ.get('FRAPPE_API_SECRET')
BASE_URL = 'https://erp.example.com'

headers = {
    'Authorization': f'token {API_KEY}:{API_SECRET}',
    'Accept': 'application/json',
    'Content-Type': 'application/json'
}

response = requests.get(f'{BASE_URL}/api/resource/Customer', headers=headers)
```

### JavaScript (Node.js) Example

```javascript
const API_KEY = process.env.FRAPPE_API_KEY;
const API_SECRET = process.env.FRAPPE_API_SECRET;

const response = await fetch('https://erp.example.com/api/resource/Customer', {
    headers: {
        'Authorization': `token ${API_KEY}:${API_SECRET}`,
        'Accept': 'application/json',
        'Content-Type': 'application/json'
    }
});
```

### cURL Example

```bash
curl -X GET "https://erp.example.com/api/resource/Customer" \
  -H "Authorization: token api_key:api_secret" \
  -H "Accept: application/json"
```

---

## 2. Basic Authentication (Alternative Token Format)

Same credentials as Token auth but encoded differently.

```
Authorization: Basic base64(<api_key>:<api_secret>)
```

```python
import base64

credentials = base64.b64encode(f'{API_KEY}:{API_SECRET}'.encode()).decode()
headers = {'Authorization': f'Basic {credentials}'}
```

---

## 3. OAuth 2.0 (Third-Party Applications)

### Step 1: Register OAuth Client

OAuth Client DocType > New:
- App Name, Redirect URIs, Default Redirect URI
- Grant Type: Authorization Code
- Scopes: `openid all` (or specific scopes)
- Save > Get Client ID and Client Secret

### Step 2: Authorization Request

```
GET /api/method/frappe.integrations.oauth2.authorize
    ?client_id={client_id}
    &response_type=code
    &scope=openid all
    &redirect_uri={redirect_uri}
    &state={random_state}
```

User authenticates and approves. Redirected to: `{redirect_uri}?code={auth_code}&state={state}`

### Step 3: Exchange Code for Token

```bash
POST /api/method/frappe.integrations.oauth2.get_token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code={authorization_code}
&redirect_uri={redirect_uri}
&client_id={client_id}
```

Response:
```json
{
    "access_token": "...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "...",
    "scope": "openid all"
}
```

### Step 4: Use Bearer Token

```
Authorization: Bearer {access_token}
```

### Token Refresh

```bash
POST /api/method/frappe.integrations.oauth2.get_token
grant_type=refresh_token&refresh_token={token}&client_id={client_id}
```

### Token Revocation

```bash
POST /api/method/frappe.integrations.oauth2.revoke_token
token={access_token}
```

### Token Introspection

```bash
POST /api/method/frappe.integrations.oauth2.introspect_token
token={access_token}&token_type_hint=access_token
```

### PKCE Support [v15+]

Full PKCE (Proof Key for Code Exchange) support for SPA and mobile apps.

---

## 4. Session/Cookie Authentication

For browser-based applications.

### Login

```python
session = requests.Session()
response = session.post('https://erp.example.com/api/method/login', json={
    'usr': 'user@example.com',
    'pwd': 'password'
})
# Session cookie set automatically
```

### Subsequent Requests

```python
# Cookie sent automatically by session
data = session.get('https://erp.example.com/api/resource/Customer')
```

### Logout

```python
session.post('https://erp.example.com/api/method/logout')
```

### JavaScript (Browser)

```javascript
// Login
await fetch('/api/method/login', {
    method: 'POST',
    credentials: 'include',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({usr: 'user@example.com', pwd: 'password'})
});

// Subsequent — cookies sent automatically
const data = await fetch('/api/resource/Customer', {credentials: 'include'});
```

**WARNING**: Session cookies expire after ~3 days. NEVER use for long-running integrations.

**NOTE**: POST/PUT/DELETE requests via session auth require CSRF token (`X-Frappe-CSRF-Token` header).

---

## Authentication Decision Matrix

| Use Case | Method | Reason |
|----------|--------|--------|
| Server-to-server integration | Token Auth | Simple, no expiry |
| Third-party web app | OAuth 2.0 | Standard, delegated access |
| Mobile app | OAuth 2.0 + PKCE [v15+] | Secure, no client secret |
| SPA (single-page app) | OAuth 2.0 + PKCE [v15+] | No client secret exposure |
| Quick scripting/testing | Token Auth | Simplest setup |
| Browser session (short) | Session/Cookie | Built-in to Frappe desk |

---

## Security Best Practices

1. **ALWAYS** generate separate API keys per integration
2. **ALWAYS** store credentials in environment variables or `site_config.json`
3. **ALWAYS** use HTTPS in production
4. **ALWAYS** create dedicated API users with minimal required roles
5. **ALWAYS** rotate API secrets regularly
6. **NEVER** hardcode credentials in source code
7. **NEVER** put API secrets in version control
8. **NEVER** use Administrator credentials for API integrations
9. **NEVER** put credentials in URL query parameters

### Credential Storage Pattern

```python
# In site_config.json
# {"external_api_key": "abc123", "external_api_secret": "secret456"}

api_key = frappe.conf.get("external_api_key")
api_secret = frappe.conf.get("external_api_secret")
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Missing/wrong auth header | Check header format and credentials |
| 403 Forbidden | Valid auth but no permission | Check user roles and User Permissions |
| 403 + CSRF error | Session auth without CSRF | Add `X-Frappe-CSRF-Token` header |
| Token invalid | Wrong format or expired secret | Regenerate API keys |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-api/references/authentication-methods.md*
