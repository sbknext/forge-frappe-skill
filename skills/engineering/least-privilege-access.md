---
title: Least-privilege access
category: engineering
tags: ['security', 'process']
source: Frappe community
---

Grant only the permissions actually needed — every excess permission is an attack surface that never gets used.

## When
Creating API keys, DB users, service accounts, IAM roles, OAuth scopes, file system permissions.

## Why
The blast radius of any compromise is bounded by the permissions of the compromised credential. A read-only DB user, even when fully compromised, cannot delete your data. A read-write user with no admin privileges cannot drop your tables. An OAuth token with only `email` scope cannot post on a user's behalf.

Over-permissioned credentials are also harder to audit: when a credential has access to everything, a log showing it called a sensitive endpoint is ambiguous — is that normal use or abuse?

## How
For database users:
```sql
-- Application user: read + write on app tables only
GRANT SELECT, INSERT, UPDATE, DELETE ON app_schema.* TO 'app_user'@'localhost';
-- Never: GRANT ALL PRIVILEGES

-- Read replica access
CREATE USER 'reporting'@'%' IDENTIFIED BY '...';
GRANT SELECT ON app_schema.* TO 'reporting'@'%';
```

For API keys:
```
// Scope at creation time
const key = await createApiKey(userId, {
  scopes: ['memories:read', 'todos:write'],  // explicit allowlist
  // Not: scopes: ['*']  or no scopes at all
});
```

For IAM / service accounts: create one role per service, with only the policies that service actually calls. Review quarterly and remove unused policies.

## Anti-pattern
`GRANT ALL PRIVILEGES ON *.* TO 'app'@'%'` — entire database cluster accessible via any compromised app credential.
