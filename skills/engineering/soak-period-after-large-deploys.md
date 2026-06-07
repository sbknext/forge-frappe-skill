---
title: Soak period after large deploys
category: engineering
tags: ['ops', 'process', 'reliability']
source: Frappe community
---

Wait 24 hours before the next deploy after any large change — let scheduled jobs run a full cycle.

## When
After any deploy that includes more than 10 commits, 500+ line net diff, or any auth/DB schema change.

## Why
Unit tests and smoke tests cannot prove scheduled job stability. A cron that fires at 06:30 IST, a digest that generates daily, a reminder that fires every minute — all of these need a real cycle to prove they still work after a big deploy. Stacking another refactor before seeing that cycle makes it impossible to attribute a regression to the right deploy.

The soak period is also when real users surface edge cases that synthetic tests miss.

## How
Post-deploy observation checklist (next 24 hours):
```
Hour 1:   /health endpoint green, no error spikes in logs
Hour 4:   any scheduled job that should have fired did fire
Hour 12:  async workers processed their queues

Next morning:
  [ ] Scheduled batch jobs completed
  [ ] No overnight error accumulation in logs
  [ ] Latency/throughput metrics nominal
```

During soak — OK to do:
- Documentation, read-only audits, planning
- Hotfix only if an actual production bug surfaces (not pre-existing issues)

During soak — NOT OK:
- Starting the next refactor
- Pushing non-critical code to prod
- Changing scheduler timing or cron config

## Anti-pattern
Deploying phase N+1 the same afternoon as phase N because "tests passed."
