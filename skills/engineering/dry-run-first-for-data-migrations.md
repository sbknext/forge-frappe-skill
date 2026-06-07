---
title: Dry-run first for data migrations
category: engineering
tags: ['data', 'safety', 'migrations']
source: Frappe community
---

Default script behavior is read-only preview; --apply is a deliberate second step.

## When
Writing or running any script that modifies database state: row updates, schema changes, data backfills, record deletes.

## Why
Prod DB state and local DB state diverge — always. A migration that touches zero rows in dev may touch thousands in prod. The human gate between "here is what will change" and "apply it" is the only thing that separates a controlled migration from a surprise.

A dry-run that reports zero rows is also valuable: it proves the WHERE clause is right, not that you happened to be lucky.

## How
```js
// Every migration script skeleton
const APPLY = process.argv.includes('--apply');

const plan = computePlan(db);
console.log(`Plan: ${plan.rows} rows affected`);
plan.entries.forEach(e => console.log(`  ${e.id}: ${e.before} → ${e.after}`));

if (!APPLY) {
  console.log('\nDry-run only. Re-run with --apply to execute.');
  process.exit(0);
}

// Backup before any write
const bak = `${DB_PATH}.pre-migration-${Date.now()}.bak`;
fs.copyFileSync(DB_PATH, bak);
console.log(`[backup] ${bak}`);

// Wrap in transaction
db.transaction(() => plan.entries.forEach(e => apply(db, e)))();
console.log(`[done] ${plan.rows} rows migrated. Backup at: ${bak}`);
```

Workflow:
1. `node migrate.js` — review plan, sample rows, count
2. Get human approval if any rows appear
3. `node migrate.js --apply` — execute with backup

## Anti-pattern
`node migrate.js --apply` as the first command, or auto-promoting dry-run to apply when "the count is small."
