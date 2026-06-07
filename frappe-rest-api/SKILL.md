---
name: frappe-rest-api
description: >
  Build and consume Frappe REST APIs — @frappe.whitelist endpoints, the auto /api/resource and
  /api/method routes, token vs session auth, request/response handling, file uploads, and CORS.
  Use when exposing app functionality over HTTP or integrating an external system with Frappe.
  Trigger on: "whitelist API", "/api/resource", "/api/method", "Frappe REST", "token auth",
  "api_key api_secret", "call Frappe from outside".
---

# Frappe REST API

Frappe exposes two API surfaces: auto CRUD over DocTypes, and custom RPC via whitelisted methods.

## Auto CRUD — /api/resource

```
GET    /api/resource/Sales Order                 # list
GET    /api/resource/Sales Order/SO-0001          # one doc
POST   /api/resource/Sales Order                  # create (JSON body)
PUT    /api/resource/Sales Order/SO-0001          # update
DELETE /api/resource/Sales Order/SO-0001          # delete

# filters / fields / paging
GET /api/resource/Sales Order?filters=[["status","=","Draft"]]&fields=["name","customer"]&limit_page_length=20

# expand linked fields in list/read responses
GET /api/resource/Sales Order?expand=["customer"]
GET /api/resource/Sales Order/SO-0001?expand_links=True

# debug: returns executed SQL under exc in payload
GET /api/resource/Sales Order?debug=True
```

## Custom RPC — /api/method

```python
import frappe

@frappe.whitelist()                 # logged-in users
def order_summary(customer: str) -> dict:
    frappe.has_permission("Sales Order", "read", throw=True)
    rows = frappe.get_all("Sales Order",
        filters={"customer": customer}, fields=["name", "grand_total"])
    return {"count": len(rows), "rows": rows}

@frappe.whitelist(allow_guest=True) # public — validate everything yourself
def public_status(token: str): ...
```
Call: `GET/POST /api/method/myapp.api.order_summary?customer=ACME`.

## Auth

```bash
# Token auth (server-to-server) — header on every call:
Authorization: token <api_key>:<api_secret>

# Generate per-user in: User → API Access → Generate Keys
curl -H "Authorization: token KEY:SECRET" \
     "https://site/api/method/myapp.api.order_summary?customer=ACME"
```
Session auth (browser): login at `/api/method/login` → cookie `sid` + `X-Frappe-CSRF-Token` for writes.

OAuth: pass `Authorization: Bearer <access_token>` (see OAuth setup docs).

Remote methods: use `GET` when the method only reads; use `POST` when it mutates DB (framework auto-commits on successful POST).

## File upload

```bash
curl -H "Authorization: token KEY:SECRET" \
     -F "file=@/path/inv.pdf" -F "doctype=Sales Order" -F "docname=SO-0001" \
     https://site/api/method/upload_file
```

## Response shape

- Method calls return `{"message": <your return value>}`.
- Errors return an HTTP status + `exc_type` / `_server_messages`. Raise `frappe.PermissionError`,
  `frappe.ValidationError`, `frappe.DoesNotExistError` to map to 403/417/404.

## Rules

- **Permission-check inside every whitelisted method** — whitelisting only exposes the route, it does not authorize.
- `allow_guest=True` = the open internet; validate inputs, rate-limit, never trust the caller.
- Return plain JSON-serializable dicts/lists, not `Document` objects.
- Use `frappe.get_all` (ignores user perms, fast) vs `frappe.get_list` (applies user perms) deliberately.
- Never accept a raw SQL fragment or `fields`/`filters` you don't validate from untrusted callers.

Source: https://docs.frappe.io/framework/user/en/api/rest
