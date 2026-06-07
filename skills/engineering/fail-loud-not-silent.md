---
title: Fail loud, not silent
category: engineering
tags: ['reliability', 'observability', 'safety']
source: Frappe community
---

When something goes wrong, throw or log an error — never return a success response for a failed operation.

## When
Every error handler, every operation that can fail, every place where "success" is a possible response.

## Why
Silent failures are the hardest class of bug to diagnose. When an operation fails silently and returns success, the caller has no way to know something went wrong. Problems compound over time. By the time someone notices the data is wrong, the root cause is days or weeks old and the audit trail is cold.

Silent success also corrupts user trust in a specific way: the user thinks the action worked, makes downstream decisions based on that belief, and discovers much later that nothing actually happened.

## How
```js
// WRONG — silent success regardless of outcome
async function updateRecord(id, data) {
  await db.update(id, data);
  return { success: true };  // always true
}

// CORRECT — surface the actual outcome
async function updateRecord(id, data) {
  const result = await db.update(id, data);
  if (result.rowsAffected === 0) {
    throw new NotFoundError(`Record ${id} not found or access denied`);
  }
  return { success: true, id, rowsAffected: result.rowsAffected };
}
```

For background jobs:
```js
try {
  await processJob(job);
} catch (error) {
  logger.error('Job failed', { jobId: job.id, error: error.message, stack: error.stack });
  await markJobFailed(job.id, error.message);
  // Do NOT mark as succeeded
  throw error;  // Let the job runner handle retry/alert
}
```

## Anti-pattern
`catch (e) { return { success: true }; }` — catching an exception and returning success is the canonical silent failure.
