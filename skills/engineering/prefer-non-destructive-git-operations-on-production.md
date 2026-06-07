---
title: Prefer non-destructive git operations on production
category: engineering
tags: ['git', 'ops', 'safety']
source: Frappe community
---

Use git merge --ff-only instead of git reset --hard when deploying — preserves working tree state you may not know about.

## When
Deploying code to a production server using git (bundle deploy, git pull, or similar).

## Why
`git reset --hard FETCH_HEAD` resets the working tree to exactly match the incoming commit. This is destructive: any files present on the server that aren't committed (legitimate hotfixes, environment-specific files, manually installed packages) are wiped without warning or backup.

`git merge --ff-only` adds new commits without touching the working tree. If the history has diverged (the only case where `--ff-only` would reject), it fails with a clear error and lets you decide — instead of silently destroying state.

A real case: a production server's `package-lock.json` had 8,964 lines of diff from the committed version (prototype packages installed manually). `git reset --hard` would have wiped this silently and broken the services depending on those packages.

## How
```bash
# 1. Create a backup branch first (see: Backup before destructive operations)
git branch "backup/pre-deploy-$(date +%Y%m%d-%H%M)"

# 2. Fetch the new commits
git fetch /tmp/deploy.bundle master:incoming

# 3. Merge non-destructively
git merge incoming --ff-only
# If this fails: history has diverged
# → check git status, decide consciously, don't force

# WRONG — potentially destructive
git fetch /tmp/deploy.bundle && git reset --hard FETCH_HEAD
```

## Anti-pattern
`git reset --hard` as the default deploy command, applied without first capturing the current working-tree state.
