---
title: Decision log: record why you chose what you chose
category: productivity
tags: ['decisions', 'documentation', 'communication']
source: Frappe community
---

Capture decisions and their reasoning so future team members understand context, not just outcomes.

## When to use
For any non-trivial decision: architecture choices, process changes, vendor selections, product direction pivots.

## Why it matters
Most teams make good decisions but document zero reasoning. 6 months later, someone asks "why did we do it this way?" and nobody knows. The decision log is institutional memory.

## Template (per entry)
```markdown
## Decision: [Short title]
**Date**: 
**Decider(s)**: 
**Status**: Decided / Superseded / Under review

### Context
[What situation forced this decision? What constraints existed?]

### Options considered
1. [Option A] — pros: ___ / cons: ___
2. [Option B] — pros: ___ / cons: ___
3. [Option C — often "do nothing"] — pros: ___ / cons: ___

### Decision
[What was chosen and the single most important reason.]

### Trade-offs accepted
[What are we knowingly giving up?]

### Review trigger
[When should this decision be revisited? E.g. "If user volume exceeds 10k/day" or "Q4 2026"]
```

## Tips
- Keep it in a shared doc (Notion, Confluence, or a `DECISIONS.md` in the repo).
- "Status: Superseded" is important — when a decision is changed, don't delete the old one. Add a link to the new one.
- One file per major domain (engineering, product, operations) keeps it navigable.
- The review trigger is often skipped and is often the most valuable field.
