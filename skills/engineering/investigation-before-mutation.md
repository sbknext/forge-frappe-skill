---
title: Investigation before mutation
category: engineering
tags: ['ops', 'safety', 'process']
source: Frappe community
---

Confirm your hypothesis with read-only evidence before changing any production state.

## When
Before any production state change: file archival, schema migration, config edit, service restart, content edit.

## Why
Local state and production state diverge — always. What is true on your laptop (that file is stale, that column is empty, that service depends on nothing) may not be true on the server. Investigation turns a belief into a fact before the point of no return.

A real example: four files believed to be identical copies were queued for archival. Investigation via md5 comparison found one was a different (older) version. Without investigation, that divergence would never have been discovered.

## How
Structure every production operation in three phases:

```
Phase X.A — Investigation (read-only)
  - Capture current state: ls, md5, grep, SELECT COUNT(*)
  - Does evidence confirm the hypothesis?

Phase X.B — Mutation (only if X.A is confirmed)
  - Execute the change
  - Health check immediately after

Phase X.C — Verification
  - Prove intended outcome achieved
  - Prove no collateral damage
```

Common investigation commands:
```bash
# File usage: is this code imported anywhere?
grep -rn "require.*filename\|from.*filename" src/ --include="*.js"

# DB state before migration
sqlite3 data/app.db "SELECT COUNT(*) FROM table WHERE condition"

# Service dependencies
systemctl status service-name --no-pager
```

## Anti-pattern
"It's obviously safe, let's just do it." Obvious is the most dangerous word in production work. Bundle investigation and mutation into a single approved step.
