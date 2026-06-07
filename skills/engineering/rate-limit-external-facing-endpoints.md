---
title: Rate limit external-facing endpoints
category: engineering
tags: ['security', 'reliability', 'ops']
source: Frappe community
---

Cap requests per API key before you ship any public endpoint — one runaway client should not drain your budget.

## When
Any endpoint reachable by external clients: public APIs, webhook receivers, AI inference proxies, developer-facing REST endpoints.

## Why
A single misbehaving script — a customer's bug, a misconfigured retry loop, a deliberate abuse attempt — can exhaust your external API quota or inference budget in minutes if there are no per-key limits. The cost is real money (LLM tokens, API calls) and real downtime (upstream provider rate-limiting your account).

Rate limiting is cheapest to add before launch. Retrofitting it after an incident involves an outage and an emergency deploy.

## How
Simplest implementation using an in-memory sliding window:
```js
const buckets = new Map();

function rateLimit(key, maxRequests, windowMs) {
  const now = Date.now();
  const bucket = buckets.get(key) || { count: 0, resetAt: now + windowMs };

  if (now > bucket.resetAt) {
    bucket.count = 0;
    bucket.resetAt = now + windowMs;
  }

  bucket.count++;
  buckets.set(key, bucket);

  if (bucket.count > maxRequests) {
    const retryAfter = Math.ceil((bucket.resetAt - now) / 1000);
    throw new RateLimitError(429, `Rate limit exceeded. Retry after ${retryAfter}s.`);
  }
}
```

For production, use Redis-backed sliding window (handles multiple instances). Return standard headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1714567890
Retry-After: 43  (only on 429)
```

## Anti-pattern
Shipping a public endpoint with no rate limiting because "we'll add it later." The abuse comes before the "later."
