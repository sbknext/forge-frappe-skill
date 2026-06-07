---
title: Feature flags for risky changes
category: engineering
tags: ['reliability', 'ops', 'process']
source: Frappe community
---

Wrap risky new behavior in a feature flag so you can disable it instantly without a redeploy.

## When
Any change that: (a) alters visible user behavior, (b) touches auth or data access patterns, (c) cannot be safely rolled back with a single revert commit, or (d) you want to test with a subset of users first.

## Why
Code deploy and feature activation are two separate concerns. Bundling them means the only way to turn off a problematic feature is to revert code, redeploy, and wait for the rollout — typically 5-15 minutes minimum. A feature flag reduces that to a config change or DB update: seconds.

Feature flags also enable gradual rollout (% of users), A/B testing, and emergency kill switches without any code changes.

## How
Simplest implementation — DB-backed flag:
```sql
CREATE TABLE feature_flags (
  name TEXT PRIMARY KEY,
  enabled INTEGER DEFAULT 0,
  rollout_pct INTEGER DEFAULT 100,  -- 0-100
  updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

```js
async function isEnabled(flagName, userId) {
  const flag = db.prepare('SELECT * FROM feature_flags WHERE name = ?').get(flagName);
  if (!flag || !flag.enabled) return false;
  if (flag.rollout_pct >= 100) return true;
  // Stable hash-based rollout
  const hash = crypto.createHash('md5').update(userId + flagName).digest('hex');
  return (parseInt(hash.slice(0, 8), 16) % 100) < flag.rollout_pct;
}

// Usage
if (await isEnabled('new_checkout_flow', req.userId)) {
  return newCheckoutHandler(req, res);
}
return legacyCheckoutHandler(req, res);
```

## Anti-pattern
Shipping directly to 100% of users with no kill switch. When it breaks at 2 AM, you're doing a code revert and redeploy instead of flipping one flag.
