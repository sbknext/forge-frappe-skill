---
title: Phased deploys with human gates
category: engineering
tags: ['process', 'ops', 'safety']
source: Frappe community
---

Break multi-step work into named phases; no phase starts until the previous one is reported and approved.

## When
Any multi-step operation: refactors across multiple files, staged migrations, phased rollouts, complex deployments.

## Why
Bundling investigation, mutation, and verification into one "deploy" command means there is no checkpoint where a human can see the state mid-operation and abort. Issues found in phase 2 are easier to fix when phase 3 hasn't run yet.

A concrete win: structuring a deploy as (1) investigation, (2) backup, (3) migration dry-run, (4) human gate, (5) migration apply, (6) deploy, (7) smoke test — each with a named output and an explicit continue/stop decision — means that when the dry-run found unexpected rows, the entire operation stopped for human review before any data was changed.

## How
```
Phase 1 — Discovery (read-only)
  Output: current state captured, hypothesis confirmed or denied
  Gate: human reviews discovery output, decides to proceed

Phase 2 — Preparation (backup, dry-run)
  Output: backup created, dry-run plan reviewed
  Gate: human approves the plan

Phase 3 — Execution
  Output: changes applied
  Gate: automated health check passes

Phase 4 — Verification
  Output: smoke tests green, metrics nominal
  Gate: human confirms end state
```

Gate format — end of each phase:
```
[Phase N — COMPLETE | BLOCKED | FAILED]
Done: <what was accomplished>
Artifacts: <files created, commits made, rows affected>
Next: <what Phase N+1 requires>
Decision needed: <what the human needs to decide>
```

## Anti-pattern
One giant script that runs all phases without pausing. Fast when it works; catastrophic when phase 3 has a bug that phase 2 would have caught.
