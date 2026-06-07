---
title: Pre-commit secret scanning
category: engineering
tags: ['security', 'git', 'process']
source: Frappe community
---

Run a secret-detection check before every commit — once a key is in git history, it requires rotation to fix.

## When
Every commit, without exception. Especially before pushing to remote.

## Why
Git history is essentially permanent. Even if you delete the secret in a follow-up commit, the original commit remains reachable via `git log`, `git show`, reflog, and any clone made before the cleanup. The only reliable response to a committed secret is immediate rotation — not history rewriting.

Secret scanning at commit time catches the leak before it ever reaches the remote, when the cost of recovery is zero.

## How
`.git/hooks/pre-commit`:
```bash
#!/bin/bash
# Secret patterns to catch
if git diff --cached | grep -qE \
  '(sk-ant-[a-zA-Z0-9-]{20,}|sk-[a-zA-Z0-9]{30,}|AIzaSy[a-zA-Z0-9_-]{33}|ghp_[a-zA-Z0-9]{36}|xai-[a-zA-Z0-9]{30,}|eyJ[a-zA-Z0-9_-]{20,}\.[a-zA-Z0-9_-]{20,})'; then
  echo "STOP: possible secret detected in staged diff."
  echo "Run: git diff --cached | grep -E '(sk-|AIzaSy|ghp_|xai-)'"
  echo "Replace secrets with environment variable references."
  exit 1
fi
```

Make the hook executable:
```bash
chmod +x .git/hooks/pre-commit
```

For teams (hooks that are version-controlled):
```bash
npm install --save-dev husky
npx husky init
# Put the scan logic in .husky/pre-commit
```

If a secret was already committed:
1. Rotate the secret immediately (generate a new one, revoke the old)
2. Update all places using it
3. Optionally clean git history (`git filter-branch` or BFG Repo Cleaner)
4. Step 1 is mandatory; step 3 is nice-to-have

## Anti-pattern
Running secret scans only in CI. By the time CI sees the commit, it has already been pushed and potentially cloned.
