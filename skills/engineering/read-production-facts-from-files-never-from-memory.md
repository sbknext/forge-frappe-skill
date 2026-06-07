---
title: Read production facts from files, never from memory
category: engineering
tags: ['ops', 'safety', 'process']
source: Frappe community
---

Server IPs, paths, branches, service names — verify from version-controlled files every time.

## When
Before writing any command that targets production: deploy scripts, SSH commands, migration invocations, restart commands.

## Why
Memory — human or AI — is a lossy snapshot. Conventions (`main` branch, `ubuntu@` user, `pm2` for process management) are 90% correct on average, which means they are silently wrong 10% of the time. The consequences range from `ssh` connecting to a stranger's server to a migration running against the wrong database file.

The most treacherous failure mode: writing a rule about not trusting memory while simultaneously trusting memory to write the rule. Discipline must be applied at the keystroke level, not just as a principle.

## How
Before writing any production command, read the canonical source:

```bash
# Deploy host and paths
cat server.sh | grep -E 'HOST|PATH|SERVICE|PORT' | head -10

# Git branch
cat .git/HEAD

# Systemd unit names
ls systemd/

# DB file location
grep -rn 'DB_PATH\|database\|sqlite' config/ src/ | head -5

# Package layout
cat package.json | head -20
```

Conflict resolution protocol:
1. Memory says X. File says Y. X ≠ Y.
2. File wins. Use Y.
3. Update the memory/note to match Y.

## Anti-pattern
"I'm pretty sure the server is at `user@hostname` and the branch is `main`." Pretty sure is not verified.
