---
title: Tests use the same libraries as production
category: engineering
tags: ['testing', 'safety', 'process']
source: Frappe community
---

Mismatched libraries between tests and production code mean your tests are proving nothing about your app.

## When
Setting up or auditing a test suite. Whenever a production library is upgraded or replaced.

## Why
If production uses library A for JWT signing and your tests use library B, the tests may pass while hiding real bugs. Library B may have different default algorithms, different error shapes, or different edge-case behavior than A. The tests give you confidence that B works correctly — not that A works correctly in production.

A concrete example: an app migrated JWT signing from `jsonwebtoken` to `jose`. Tests continued using `jsonwebtoken`. The 11 existing tests kept passing. When the tests were migrated to `jose`, three new red tests were written — and they surfaced real production auth bugs that had existed undetected.

## How
Audit libraries on each axis:
```bash
# What does production use?
grep -rn "from 'jose'\|from 'jsonwebtoken'" src/ --include="*.js"

# What do tests use?
grep -rn "from 'jose'\|from 'jsonwebtoken'" tests/ --include="*.js"
```

If they differ → migrate tests to match production. Not the other way around.

Apply this to every dependency class: JWT library, DB client, HTTP client, crypto, date handling. Same version, same configuration.

## Anti-pattern
"The test uses a simpler library because it's easier to set up." Easier to set up means easier to diverge. Divergence means your test suite is not testing your production code.
