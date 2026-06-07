---
title: Separate file transfer from directory sync
category: engineering
tags: ['ops', 'safety']
source: Frappe community
---

scp a specific file rather than rsync an entire directory when you know exactly what changed — avoids unintended overwrites.

## When
Deploying a specific file (a config change, a single script, a single binary) to a production server.

## Why
`rsync -av src/ user@host:dest/` syncs the entire source directory tree including deletions (with `--delete`), hidden files, node_modules, `.env` files, and any file that is present locally but should not be on the server. It is the right tool when you want exact mirror-of-source semantics. It is the wrong tool when you want to deploy one file.

`scp src/config.json user@host:dest/config.json` transfers exactly that one file. There are no side effects, no deletions, no surprises. For known-targeted deploys, the specificity is a feature.

## How
```bash
# Deploy a single config file
scp config/nginx.conf user@host:/etc/nginx/sites-available/myapp

# Deploy a single script update
scp scripts/migrate.js user@host:/opt/app/scripts/migrate.js

# After transfer, verify integrity
md5sum scripts/migrate.js
ssh user@host 'md5sum /opt/app/scripts/migrate.js'
# Both should match

# If you genuinely want directory sync (with full awareness of what it does)
rsync -av --exclude='.env' --exclude='node_modules/' --dry-run src/ user@host:dest/
# Review dry-run output before removing --dry-run
```

## Anti-pattern
`rsync -av --delete . user@host:/opt/app/` from a dev machine — this will delete files that exist on the server but not locally (logs, .env, locally-installed packages, user-uploaded files).
