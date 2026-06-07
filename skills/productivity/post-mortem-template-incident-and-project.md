---
title: Post-mortem template (incident and project)
category: productivity
tags: ['post-mortem', 'incident', 'learning']
source: Frappe community
---

Run a blame-free post-mortem that surfaces systemic causes and produces durable fixes.

## When to use
After any significant incident (outage, missed deadline, major bug) or at the end of a project.

## Principles
- **Blame-free**: actions of well-meaning people in broken systems. Fix the system.
- **Factual timeline**: what happened, in order, with timestamps.
- **5 Whys**: keep asking why until you find a systemic cause, not a person.

## Template
```markdown
## Post-Mortem: [Incident/Project name]
**Date**: 
**Severity**: P0/P1/P2
**Duration**: [Start] → [End]
**Impact**: [Users affected / revenue impacted / SLA breach]
**Authors**: 

---

### Timeline
| Time | Event |
|------|-------|
| HH:MM | [What happened] |
| HH:MM | [Detection] |
| HH:MM | [Response action] |
| HH:MM | [Resolution] |

### Root cause analysis (5 Whys)
Q: Why did the incident happen? A: 
Q: Why did [A]? A: 
Q: Why did [A2]? A: 
...(continue until systemic cause)
Root cause: 

### Contributing factors
- [Factor 1]
- [Factor 2]

### What went well
- 

### Action items (each must have owner + date)
| Action | Type | Owner | Due |
|--------|------|-------|-----|
| | Prevention/Detection/Mitigation | | |

### Lessons learned
[1–3 sentences: what would we do differently?]
```

## Tips
- 5 Whys stops when the answer is a system/process failure, not a person's failure.
- Action types matter: Prevention (stop it happening), Detection (know sooner), Mitigation (reduce impact).
- Share the post-mortem within 48 hours — memory degrades fast and it signals organisational learning culture.
