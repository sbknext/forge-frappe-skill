---
title: Sprint planning: 1-hour team template
category: productivity
tags: ['sprint', 'planning', 'agile']
source: Frappe community
---

Structure a sprint planning meeting in 60 minutes with clear capacity, goal, and committed stories.

## When to use
Start of each sprint (typically fortnightly). Requires backlog groomed in advance.

## Agenda
```
00–05 min: Recap — how did last sprint finish? Velocity? Carry-overs?
05–15 min: Sprint goal — one sentence, what does done look like?
15–45 min: Story selection — pick stories from groomed backlog until capacity is met
45–55 min: Capacity check — sum story points vs. team capacity (hours or points)
55–60 min: Commitments — who owns what, any dependencies flagged
```

## Template
```markdown
## Sprint [N] Planning — [Date]

**Sprint goal**: _______________________________________

**Capacity**
| Person | Days available | Capacity (pts or hrs) |
|--------|---------------|------------------------|
| | | |
Total team capacity: ___

**Committed stories**
| Story | Points | Owner | Dependency |
|-------|--------|-------|------------|
| | | | |
Total committed: ___

**Carry-overs from Sprint [N-1]**
- [Story] — reason for carry-over

**Risks / flags**
- 

**Definition of Done reminder**
- Code reviewed + merged
- Tests passing
- Deployed to staging
- PM sign-off on UX-impacting stories
```

## Tips
- Sprint goal should be a business outcome, not a list of features. "Users can complete signup" not "Implement signup flow".
- Leave 20% capacity unplanned for bugs, incidents, and review work.
- If stories aren't groomed (no AC, unclear scope), pull from backlog — don't plan ungroom stories into a sprint.
