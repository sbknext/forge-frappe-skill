---
title: Frappe Ref — Frappe Core: Api: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'core', 'api', 'reference', 'examples']
source: Impertio
---

# API Examples

> Complete working examples for common integration scenarios.

---

## 1. Python API Client

```python
"""Frappe API Client — production-ready implementation."""
import requests
import json
import os
from typing import Optional, Dict, List, Any

class FrappeClient:
    def __init__(self, url: str, api_key: str, api_secret: str, timeout: int = 30):
        self.url = url.rstrip('/')
        self.timeout = timeout
        self.session = requests.Session()
        self.session.headers.update({
            'Authorization': f'token {api_key}:{api_secret}',
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        })

    def _request(self, method: str, endpoint: str, **kwargs) -> Dict:
        kwargs.setdefault('timeout', self.timeout)
        response = self.session.request(method, f'{self.url}{endpoint}', **kwargs)
        if not response.ok:
            error = response.json()
            raise Exception(error.get('_server_messages') or
                            error.get('message') or response.text)
        return response.json()

    # Resource API
    def get_list(self, doctype: str, fields: Optional[List[str]] = None,
                 filters: Optional[List] = None, order_by: Optional[str] = None,
                 limit_start: int = 0, limit_page_length: int = 20) -> List[Dict]:
        params = {'limit_start': limit_start, 'limit_page_length': limit_page_length}
        if fields:
            params['fields'] = json.dumps(fields)
        if filters:
            params['filters'] = json.dumps(filters)
        if order_by:
            params['order_by'] = order_by
        return self._request('GET', f'/api/resource/{doctype}', params=params).get('data', [])

    def get_doc(self, doctype: str, name: str) -> Dict:
        return self._request('GET', f'/api/resource/{doctype}/{name}').get('data', {})

    def create_doc(self, doctype: str, data: Dict) -> Dict:
        return self._request('POST', f'/api/resource/{doctype}', json=data).get('data', {})

    def update_doc(self, doctype: str, name: str, data: Dict) -> Dict:
        return self._request('PUT', f'/api/resource/{doctype}/{name}', json=data).get('data', {})

    def delete_doc(self, doctype: str, name: str) -> bool:
        self._request('DELETE', f'/api/resource/{doctype}/{name}')
        return True

    # Method API
    def call_method(self, method: str, **kwargs) -> Any:
        return self._request('POST', f'/api/method/{method}', json=kwargs).get('message')

    # Convenience
    def get_count(self, doctype: str, filters: Optional[Dict] = None) -> int:
        return self.call_method('frappe.client.get_count',
                                doctype=doctype, filters=filters or {})

    def submit_doc(self, doctype: str, name: str) -> Dict:
        return self.call_method('frappe.client.submit',
                                doc={'doctype': doctype, 'name': name})

# Usage
client = FrappeClient(
    url='https://erp.example.com',
    api_key=os.environ['FRAPPE_API_KEY'],
    api_secret=os.environ['FRAPPE_API_SECRET']
)

customers = client.get_list('Customer',
    fields=['name', 'customer_name', 'outstanding_amount'],
    filters=[['outstanding_amount', '>', 0]],
    order_by='outstanding_amount desc',
    limit_page_length=10)
```

---

## 2. JavaScript/Node.js Client

```javascript
class FrappeClient {
    constructor(url, apiKey, apiSecret) {
        this.url = url.replace(/\/$/, '');
        this.auth = `token ${apiKey}:${apiSecret}`;
    }

    async _request(method, endpoint, options = {}) {
        const response = await fetch(`${this.url}${endpoint}`, {
            method,
            headers: {
                'Authorization': this.auth,
                'Accept': 'application/json',
                'Content-Type': 'application/json',
                ...options.headers
            },
            body: options.body ? JSON.stringify(options.body) : undefined
        });
        const data = await response.json();
        if (!response.ok) {
            throw new Error(data._server_messages || data.message || response.statusText);
        }
        return data;
    }

    async getList(doctype, options = {}) {
        const params = new URLSearchParams();
        if (options.fields) params.set('fields', JSON.stringify(options.fields));
        if (options.filters) params.set('filters', JSON.stringify(options.filters));
        if (options.orderBy) params.set('order_by', options.orderBy);
        params.set('limit_start', options.limitStart || 0);
        params.set('limit_page_length', options.limitPageLength || 20);
        return (await this._request('GET', `/api/resource/${doctype}?${params}`)).data || [];
    }

    async getDoc(doctype, name) {
        return (await this._request('GET', `/api/resource/${doctype}/${name}`)).data || {};
    }

    async createDoc(doctype, data) {
        return (await this._request('POST', `/api/resource/${doctype}`, {body: data})).data;
    }

    async updateDoc(doctype, name, data) {
        return (await this._request('PUT', `/api/resource/${doctype}/${name}`, {body: data})).data;
    }

    async deleteDoc(doctype, name) {
        await this._request('DELETE', `/api/resource/${doctype}/${name}`);
        return true;
    }

    async callMethod(method, args = {}) {
        return (await this._request('POST', `/api/method/${method}`, {body: args})).message;
    }
}
```

---

## 3. cURL Examples

```bash
#!/bin/bash
BASE_URL="https://erp.example.com"
AUTH="Authorization: token ${FRAPPE_API_KEY}:${FRAPPE_API_SECRET}"

# List with filters
curl -s -X GET "${BASE_URL}/api/resource/Customer" \
  -H "${AUTH}" -H "Accept: application/json" -G \
  --data-urlencode 'fields=["name","customer_name"]' \
  --data-urlencode 'filters=[["customer_type","=","Company"]]' \
  --data-urlencode 'limit_page_length=10'

# Create
curl -s -X POST "${BASE_URL}/api/resource/Customer" \
  -H "${AUTH}" -H "Content-Type: application/json" \
  -d '{"customer_name":"New Corp","customer_type":"Company"}'

# Update
curl -s -X PUT "${BASE_URL}/api/resource/Customer/CUST-001" \
  -H "${AUTH}" -H "Content-Type: application/json" \
  -d '{"customer_group":"Premium"}'

# Delete
curl -s -X DELETE "${BASE_URL}/api/resource/Customer/CUST-001" \
  -H "${AUTH}" -H "Accept: application/json"

# Call method
curl -s -X POST "${BASE_URL}/api/method/frappe.client.get_count" \
  -H "${AUTH}" -H "Content-Type: application/json" \
  -d '{"doctype":"Sales Order","filters":{"status":"Draft"}}'
```

---

## 4. Pagination Helper

```python
def fetch_all_documents(client, doctype, filters=None, fields=None, batch_size=100):
    """Retrieve all documents with automatic pagination."""
    all_docs, offset = [], 0
    while True:
        batch = client.get_list(doctype, filters=filters, fields=fields,
                                limit_start=offset, limit_page_length=batch_size)
        if not batch:
            break
        all_docs.extend(batch)
        if len(batch) < batch_size:
            break
        offset += batch_size
    return all_docs
```

---

## 5. Batch Operations

```python
def batch_create(client, doctype, documents, batch_size=50):
    """Create multiple documents with error tracking."""
    results = []
    for i in range(0, len(documents), batch_size):
        for doc in documents[i:i + batch_size]:
            try:
                result = client.create_doc(doctype, doc)
                results.append({'success': True, 'name': result.get('name')})
            except Exception as e:
                results.append({'success': False, 'error': str(e), 'data': doc})
    return results

# Usage
results = batch_create(client, 'Customer', [
    {'customer_name': 'Corp A', 'customer_type': 'Company'},
    {'customer_name': 'Corp B', 'customer_type': 'Company'},
])
failed = [r for r in results if not r['success']]
```

---

## 6. Error Handling Classes

```python
class FrappeAPIError(Exception):
    pass

class ValidationError(FrappeAPIError):
    pass

class PermissionError(FrappeAPIError):
    pass

class NotFoundError(FrappeAPIError):
    pass

def handle_response(response):
    if response.status_code == 200:
        return response.json()
    try:
        error = response.json()
    except Exception:
        raise FrappeAPIError(f"HTTP {response.status_code}: {response.text}")

    exc_type = error.get('exc_type', '')
    msg = error.get('_server_messages', '')

    if 'ValidationError' in exc_type:
        raise ValidationError(msg)
    elif 'PermissionError' in exc_type or response.status_code == 403:
        raise PermissionError(msg)
    elif 'DoesNotExistError' in exc_type or response.status_code == 404:
        raise NotFoundError(msg)
    else:
        raise FrappeAPIError(msg or error.get('message'))
```

---

## 7. Webhook Receiver (Flask)

```python
from flask import Flask, request, jsonify
import hmac, hashlib, base64, os

app = Flask(__name__)
SECRET = os.environ.get('WEBHOOK_SECRET')

@app.route('/webhook/order', methods=['POST'])
def handle_webhook():
    sig = request.headers.get('X-Frappe-Webhook-Signature')
    if sig:
        expected = base64.b64encode(
            hmac.new(SECRET.encode(), request.data, hashlib.sha256).digest()
        ).decode()
        if not hmac.compare_digest(expected, sig):
            return jsonify({'error': 'Invalid signature'}), 401

    data = request.json
    # Process asynchronously for production
    return jsonify({'status': 'received'}), 200
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/core/frappe-core-api/references/examples.md*
