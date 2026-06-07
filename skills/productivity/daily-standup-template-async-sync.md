---
title: Daily standup template (async + sync)
category: productivity
tags: ['standup', 'communication', 'daily']
source: Frappe community
---

A 3-field standup format that works for both live team calls and async Slack/chat updates.

## When to use
Every working day. Write async version before 10 AM; live version follows the same structure.

## Format
Three fields only. No preamble.

```
YESTERDAY
- [Outcome, not activity. "Shipped X" not "Worked on X"]
- [If nothing shipped, say what moved closer and by how much]

TODAY
- [2-3 specific outcomes you plan to complete]
- [Not tasks — outcomes]

BLOCKER
- [One line or "None"]
- [If blocked, name who/what can unblock you]
```

## Example (good)
```
YESTERDAY
- Merged PR #142 — login redirect fix, deployed to staging
- Completed design review for onboarding flow (feedback sent to Priya)

TODAY
- Write API spec for notification service (done by EOD)
- Review Raj's PR #148 (30 min)

BLOCKER
- Need database access to test #149 — waiting on Suresh
```

## Example (bad — avoid)
```
YESTERDAY
- Working on the login bug
- Had meetings

TODAY
- Continue login bug
- More meetings

BLOCKER
- None (but silently stuck)
```

## Tips
- "Outcome" = something that moved the product/project state. "Activity" = what you did. Standups should track outcomes.
- Blockers deserve a name. "Waiting on X" with no name creates invisible queues.
- Async standups: post by 10 AM. If you miss, post whenever — a late standup is still valuable.
