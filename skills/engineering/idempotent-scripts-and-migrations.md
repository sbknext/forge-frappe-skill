---
title: Idempotent scripts and migrations
category: engineering
tags: ['data', 'reliability', 'migrations']
source: Frappe community
---

Any script that modifies state should produce the same result whether run once or ten times.

## When
Database migrations, seed scripts, setup scripts, deploy hooks, or any automation that writes state.

## Why
Non-idempotent scripts fail in predictable ways on the second run: duplicate key errors, double-charged events, double-inserted rows. In incident response, you often need to re-run a script to recover — if it's not idempotent, re-running makes things worse.

Idempotency also makes scripts safe to run as part of automated deploys: no need to track "has this migration been applied?" state separately.

## How
For SQL:
```sql
-- Non-idempotent (fails on second run)
INSERT INTO settings (key, value) VALUES ('theme', 'dark');

-- Idempotent
INSERT INTO settings (key, value) VALUES ('theme', 'dark')
  ON CONFLICT(key) DO UPDATE SET value = excluded.value;

-- Table creation
CREATE TABLE IF NOT EXISTS users (id TEXT PRIMARY KEY, ...);

-- Index creation
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email ON users(email);
```

For script logic:
```js
// Check before acting
const existing = db.prepare('SELECT id FROM config WHERE key = ?').get('version');
if (!existing) {
  db.prepare('INSERT INTO config (key, value) VALUES (?, ?)').run('version', '1.0');
  console.log('Config seeded');
} else {
  console.log('Config already present, skipping');
}
```

## Anti-pattern
A migration that runs `ALTER TABLE ... ADD COLUMN` without `IF NOT EXISTS` — fails on every re-run and blocks the migration runner on all subsequent migrations.
