---
title: One fix, one commit
category: engineering
tags: ['git', 'process', 'maintainability']
source: Frappe community
---

Keep commits small (≤5 files, ≤300 lines) and single-purpose so any single change can be reverted in isolation.

## When
Every commit, every time. Especially during refactors and hotfixes.

## Why
Small, single-purpose commits are the fundamental unit of reversibility. When a production incident occurs, the question is always: "which change caused this?" If the last 5 commits each touch 1 concept, bisection takes minutes. If the last commit touches 8 files and 3 features, bisection is a negotiation.

A concrete win: after a 22-commit refactor, two latent bugs were found in smoke testing. Because every commit was single-purpose and under 5 files, ruling out the entire refactor as the source took under 5 minutes via `git log -p`. The bugs turned out to be pre-existing — a conclusion reachable only because the commit history was clean.

## How
```
Soft cap: 3 files, 100 lines
Hard cap: 5 files, 300 lines
Beyond hard cap: STOP, split into two commits
```

Commit message format:
```
fix(auth): reject tokens missing userId claim

Why: sub-only JWT was silently accepted; add explicit null-check.
Tests: multiuser.test.js T15 now green.
```

Hard stop triggers — ask before committing:
- Touches middleware or auth
- Changes a test's expected output
- Touches two unrelated feature areas
- Over 300 lines of diff

## Anti-pattern
"Phase 3 complete" — 22 files, mixed concerns, commit that cannot be reverted without losing everything.
