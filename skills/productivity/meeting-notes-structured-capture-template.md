---
title: Meeting notes: structured capture template
category: productivity
tags: ['meetings', 'notes', 'communication']
source: Frappe community
---

Capture meeting context, decisions, and actions in a format that's useful to people who weren't there.

## When to use
Every non-trivial meeting (>2 people, >15 minutes). Takes 10 min to write after the meeting.

## Template
```markdown
## [Meeting name] — [Date] [Time] IST

**Attendees**: [Names or roles]
**Absent**: [Names + why, if relevant]
**Facilitator**: 
**Scribe**: 

---

### Context (2 sentences)
Why did this meeting exist? What was the input state?

### Decisions made
- [Decision 1] — owned by [Name]
- [Decision 2] — owned by [Name]

### Action items
| Action | Owner | Due date |
|--------|-------|----------|
| | | |

### Open questions / parking lot
- [Question] — to be resolved by [Person/date]

### Next meeting
[Date/time/agenda, or "not needed"]
```

## Example (good)
```
## API v2 Design Review — 2026-05-28 3 PM IST
Attendees: Arjun (BE), Priya (FE), Sam (Product)

Context: Reviewing the proposed pagination approach for /orders endpoint before FE build starts.

Decisions made:
- Use cursor-based pagination (not page+offset) — Arjun to update spec by tomorrow
- Rate limit at 100 req/min per API key — Sam to update API docs

Action items:
| Action | Owner | Due |
|--------|-------|-----|
| Update pagination spec | Arjun | 2026-05-29 |
| Update rate limit docs | Sam | 2026-05-30 |
| FE integration spike | Priya | 2026-06-02 |
```

## Tips
- Share the notes within 24 hours. After that, context decays.
- Decisions section is the most important — it's the record of why things are the way they are.
- Parking lot items must have an owner; ownerless questions get forgotten.
