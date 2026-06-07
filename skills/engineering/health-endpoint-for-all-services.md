---
title: Health endpoint for all services
category: engineering
tags: ['observability', 'reliability', 'ops']
source: Frappe community
---

Every service should expose /health returning its dependency status — silent failure is the hardest class of outage to detect.

## When
Every server-side service you build or operate.

## Why
A service that returns 200 on the root route but has a broken database connection, a failed scheduler, or a misconfigured external dependency is silently degraded. The first person to notice is usually a user who gets wrong data or a failed request. A health endpoint externalizes the internal state so monitoring can catch it first.

This also enables load balancer health checks, automated recovery (restart on health failure), and deployment verification ("deploy complete" = health endpoint green after deploy).

## How
```js
// Express example
app.get('/health', async (req, res) => {
  const checks = {};
  let healthy = true;

  // DB check
  try {
    db.prepare('SELECT 1').get();
    checks.db = 'ok';
  } catch (e) {
    checks.db = 'error: ' + e.message;
    healthy = false;
  }

  // External dependency check
  try {
    const resp = await fetch('https://api.example.com/ping', { signal: AbortSignal.timeout(2000) });
    checks.externalApi = resp.ok ? 'ok' : `error: ${resp.status}`;
    if (!resp.ok) healthy = false;
  } catch (e) {
    checks.externalApi = 'error: ' + e.message;
    healthy = false;
  }

  // Scheduler check (if applicable)
  const lastRun = db.prepare('SELECT ran_at FROM scheduler_log ORDER BY ran_at DESC LIMIT 1').get();
  const minutesSinceRun = lastRun ? (Date.now() - new Date(lastRun.ran_at)) / 60000 : Infinity;
  checks.scheduler = minutesSinceRun < 5 ? 'ok' : `stale: ${Math.round(minutesSinceRun)}m ago`;
  if (minutesSinceRun > 10) healthy = false;

  res.status(healthy ? 200 : 503).json({ status: healthy ? 'ok' : 'degraded', checks });
});
```

## Anti-pattern
`app.get('/health', (req, res) => res.json({ status: 'ok' }))` — always-200 health endpoint defeats the purpose and gives false confidence to monitoring.
