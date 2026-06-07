---
title: Retry with exponential backoff
category: engineering
tags: ['reliability', 'integrations']
source: Frappe community
---

When calling external services, retry transiently-failed requests with increasing delays and a cap.

## When
Any call to an external service: HTTP APIs, databases under load, message queues, object storage.

## Why
Transient failures — network blips, brief overload, cold-start latency — are normal in distributed systems. Retrying immediately hammers an already-struggling service. Retrying with backoff gives the service time to recover while still eventually succeeding from the caller's perspective.

Without backoff, a thundering herd of retries is often what turns a 30-second blip into a 30-minute outage.

## How
```js
async function withRetry(fn, options = {}) {
  const {
    maxAttempts = 3,
    baseDelayMs = 200,
    maxDelayMs = 10_000,
    shouldRetry = (err) => err.status >= 500 || err.code === 'ECONNRESET',
  } = options;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts || !shouldRetry(error)) throw error;

      const jitter = Math.random() * baseDelayMs;
      const delay = Math.min(baseDelayMs * 2 ** (attempt - 1) + jitter, maxDelayMs);

      console.warn(`Attempt ${attempt} failed, retrying in ${Math.round(delay)}ms`, {
        error: error.message
      });
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

// Usage
const data = await withRetry(() => fetch('https://api.example.com/data'));
```

**Never retry:** 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found. These are caller errors — retrying wastes time and inflates billing.

## Anti-pattern
Retry immediately in a tight loop: `while (tries < 3) { try { ... } catch { tries++ } }` — equivalent to multiplying the load by 3 at the worst possible moment.
