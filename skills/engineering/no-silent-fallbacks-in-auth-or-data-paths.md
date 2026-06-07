---
title: No silent fallbacks in auth or data paths
category: engineering
tags: ['security', 'auth', 'safety']
source: Frappe community
---

When a required identity value is missing, reject loudly — never fall through to a guess.

## When
JWT middleware, session validation, any DB query scoped by user identity.

## Why
Chained fallback operators in auth code (`a || b || c`) are bugs waiting for a token shape inconsistency. When `userId` is missing, falling back to `sub` or `id` accepts a structurally different token without telling anyone. When `userId` resolves to `undefined`, passing it into a SQL query produces `WHERE user_id = undefined`, which SQLite and most databases handle silently (empty result set with 200 OK — no error raised).

Every silent fallback in an auth path is a potential cross-user data leak. The cost of an explicit rejection (one 401) is always lower than the cost of serving the wrong user's data.

## How
```js
// WRONG — silent fallback
req.userId = payload.userId || payload.sub || payload.id;

// WRONG — no null-check, undefined flows into SQL
req.userId = payload.userId;

// CORRECT — explicit reject
if (!payload.userId) {
  return res.status(401).json({
    error: 'invalid_token',
    reason: 'missing_userId_claim'
  });
}
req.userId = payload.userId;
```

Same discipline for DB handlers:
```js
// WRONG — unconditional success
const result = await db.update(id, userId);
res.json({ success: true });

// CORRECT — honor the DB result
const result = await db.update(id, userId); // returns { changes: N }
if (result.changes === 0) {
  return res.status(404).json({ error: 'not_found_or_forbidden' });
}
res.json({ success: true });
```

## Anti-pattern
`payload.userId || payload.sub || payload.id` — every `||` here is a potential impersonation vector.
