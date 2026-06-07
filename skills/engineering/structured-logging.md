---
title: Structured logging
category: engineering
tags: ['observability', 'reliability']
source: Frappe community
---

Log JSON objects with consistent fields — free-text logs don't aggregate, can't be queried, and age poorly.

## When
Any application logging: errors, business events, performance measurements, request/response.

## Why
Free-text logs work until you have more than one server, more than one service, or more than one week of logs. At that point, `grep` stops working and you need to search, aggregate, and correlate across machines and time. Structured logs (JSON) are queryable from day one.

Structured logging also prevents a common pattern: "I added a log line but the format is subtly different from the other log line so my grep doesn't match and I miss the error."

## How
```js
// Minimal structured log shape
const log = {
  ts: new Date().toISOString(),  // ISO 8601, always UTC
  level: 'error',                 // debug | info | warn | error
  service: 'auth',                // which module/service
  event: 'jwt_verify_failed',     // machine-readable event name
  userId: req.userId,             // always include if available
  requestId: req.id,              // trace correlation
  error: error.message,           // human readable
  // Never include: full JWTs, passwords, raw secrets
};
console.log(JSON.stringify(log));
```

For Node.js, `pino` is lightweight and outputs NDJSON by default:
```js
import pino from 'pino';
const logger = pino({ level: 'info' });
logger.error({ userId, event: 'login_failed', reason: 'bad_password' });
```

## Anti-pattern
```js
console.log('User ' + userId + ' failed auth at ' + new Date());
// Can't query, can't aggregate, can't set alert thresholds
```
