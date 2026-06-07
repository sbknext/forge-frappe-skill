---
title: Backup before destructive operations
category: engineering
tags: ['ops', 'safety', 'data']
source: Frappe community
---

Copy the DB (or create a git backup branch) before any operation that cannot be easily undone.

## When
Before: schema migrations, bulk data updates, file archival, production deploys, any `DELETE` or `UPDATE` without a narrow `WHERE` clause.

## Why
Recovery is priceless; backup storage is cheap. A backup that takes 2 seconds and costs 1 MB of disk space is the only thing standing between a mistake and permanent data loss. This is true even when you are confident the operation is safe — confidence is not a backup.

For code deploys specifically, a git backup branch on production means rollback is a single `git checkout` command instead of a code revert, test, and redeploy cycle.

## How
DB backup:
```bash
# SQLite
cp data/app.db data/app.db.$(date +%Y%m%d-%H%M).bak
ls -la data/app.db.*.bak  # verify file exists and is > 0 bytes

# PostgreSQL
pg_dump mydb > /tmp/mydb-$(date +%Y%m%d-%H%M).sql

# MySQL / MariaDB
mysqldump mydb > /tmp/mydb-$(date +%Y%m%d-%H%M).sql
```

Git backup branch (before prod deploy):
```bash
# On the production server, before merging incoming changes
BACKUP_BRANCH="backup/pre-deploy-$(date +%Y%m%d-%H%M)"
git branch "$BACKUP_BRANCH"
echo "Backup created: $BACKUP_BRANCH"
# Rollback: git checkout $BACKUP_BRANCH
```

Pre-migration script pattern:
```js
const bak = `${DB_PATH}.pre-${SCRIPT_NAME}-${Date.now()}.bak`;
fs.copyFileSync(DB_PATH, bak);
console.log(`[backup] ${bak}`);
// proceed with migration
```

## Anti-pattern
"It's a small change, I'll skip the backup." The smaller the change feels, the more dangerous skipping the backup becomes — because you lower your guard.
