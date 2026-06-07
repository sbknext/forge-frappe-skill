---
title: Refactor cleanup in the same session
category: engineering
tags: ['git', 'maintainability', 'process']
source: Frappe community
---

When you replace function A with function B, delete A in the same PR — later never comes.

## When
Any refactor that introduces a replacement for an existing function, module, or file.

## Why
Every incomplete refactor leaves behind a "shadow" of the old code. Two weeks later, nobody remembers which version is canonical. The next developer edits the old one. The new one silently drifts. Eventually both exist in production and differ.

Shadow route files, duplicate middleware, stale config keys — all of these follow the same pattern: someone was careful enough to add the new version but did not want to risk deleting the old one in the same PR.

## How
Two-step pattern (two commits, same PR):

```
Commit 1: introduce B, migrate all call sites to B
  Message: "refactor(X): introduce B as canonical, migrate all references"

Commit 2: delete A, verify zero references
  Message: "refactor(X): remove A — all call sites migrated (grep verified)"
```

Pre-delete verification:
```bash
# Confirm zero references remain
grep -rn "from.*old-module\|require.*old-module" src/ --include="*.js"
# Expected: no output

# Run tests with old module deleted
npm test  # must still be green
```

If deleting on production instead of in source control, use the archive pattern (see: Archive, never delete, in production).

## Anti-pattern
Commit 1 merged. "I'll clean up A in a follow-up PR." The follow-up PR never gets prioritized. Shadow code accumulates for years.
