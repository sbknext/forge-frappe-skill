---
title: Archive, never delete, in production
category: engineering
tags: ['ops', 'safety', 'data']
source: Frappe community
---

Move files to a timestamped archive directory with a README instead of rm — recovery beats regret.

## When
You need to remove stale files, dead code, or unused configs from a production host.

## Why
Deletion is permanent. Your hypothesis that a file is safe to delete is just that — a hypothesis. The cost of being wrong is infinite (data loss, broken deploy); the cost of archiving instead is a few KB. On production systems, the asymmetry mandates archive by default.

Even if grep proves a file is unreferenced, a running process may have it open, a supervisor may reference it, or a future rollback may need it.

## How
```bash
ARCHIVE=/opt/app/archive/pre-cleanup-$(date +%Y%m%d-%H%M)
mkdir -p "$ARCHIVE"
cat > "$ARCHIVE/README.md" << 'EOF'
# Archive: cleanup-YYYYMMDD-HHMM
# Files moved because: <reason>
# Verified unused via: <grep commands run>
# To restore: mv "$ARCHIVE"/* <original-path>
EOF
mv stale-file.js "$ARCHIVE/"
```

After the `mv`, run a health check immediately. If services degrade, restore in seconds: `mv "$ARCHIVE/stale-file.js" /original/path/`.

## Anti-pattern
`rm -f stale-file.js` — one command, permanent consequence. Even if you are 99% sure it is safe, the 1% case has no recovery path once the disk sector is overwritten.
