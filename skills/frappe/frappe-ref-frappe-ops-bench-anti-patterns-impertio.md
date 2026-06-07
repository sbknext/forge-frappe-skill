---
title: Frappe Ref — Frappe Ops: Bench: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'ops', 'bench', 'reference', 'anti-patterns']
source: Impertio
---

# Bench Anti-Patterns

## Anti-Pattern 1: Running Bench as Root

```bash
# WRONG — creates permission issues, security risk
sudo bench start
sudo bench new-site mysite.localhost
```

**Fix**: ALWAYS run bench as a dedicated user (e.g., `frappe`):

```bash
# Create dedicated user
sudo adduser frappe
# Run all bench commands as frappe user
su - frappe
bench init myproject
```

## Anti-Pattern 2: Skipping Migrate After Code Changes

```bash
# WRONG — schema out of sync, runtime errors
git pull origin main
bench build  # builds assets but does NOT sync schema
# Missing: bench --site mysite migrate
```

**Fix**: ALWAYS migrate after pulling code:

```bash
git pull origin main
bench --site mysite migrate  # Sync schema + run patches
bench build --app myapp      # Rebuild assets
```

## Anti-Pattern 3: Using bench update --reset in Production

```bash
# WRONG — destroys ALL local changes without warning
bench update --reset
```

**Fix**: Understand what `--reset` does (`git reset --hard`). In production:

```bash
# Check for local changes first
cd apps/erpnext && git status

# If you have intentional patches, stash them
git stash

# Then update
bench update

# Re-apply patches
git stash pop
```

## Anti-Pattern 4: Not Setting --frappe-branch on bench init

```bash
# WRONG — gets latest develop branch (unstable)
bench init myproject
```

**Fix**: ALWAYS specify the Frappe version:

```bash
bench init myproject --frappe-branch version-15
```

## Anti-Pattern 5: Manually Deleting App Directories

```bash
# WRONG — leaves orphaned data in sites, broken references
rm -rf apps/custom_app
```

**Fix**: Use proper uninstall sequence:

```bash
# First uninstall from all sites
bench --site mysite uninstall-app custom_app
# Then remove from bench
bench remove-app custom_app
```

## Anti-Pattern 6: Editing site_config.json by Hand Without Validation

```bash
# WRONG — typos, invalid JSON, missing commas
vim sites/mysite/site_config.json
```

**Fix**: Use bench commands for config changes:

```bash
bench --site mysite set-config developer_mode 1
bench --site mysite show-config  # Verify
```

If you must edit manually, ALWAYS validate JSON before saving.

## Anti-Pattern 7: No Backup Before Destructive Operations

```bash
# WRONG — no recovery if something goes wrong
bench --site production.example.com reinstall
```

**Fix**: ALWAYS backup first:

```bash
bench --site production.example.com backup
bench --site production.example.com reinstall  # Now safe to proceed
```

## Anti-Pattern 8: Running bench update During Business Hours

```bash
# WRONG — causes downtime, locks database during migration
bench update  # at 2:00 PM on a Monday
```

**Fix**: Schedule updates during off-hours:

```bash
# 1. Put in maintenance mode
bench --site mysite set-maintenance-mode 1

# 2. Wait for pending jobs to complete
bench --site mysite ready-for-migration

# 3. Update
bench update

# 4. Verify
bench doctor

# 5. Remove maintenance mode
bench --site mysite set-maintenance-mode 0
```

## Anti-Pattern 9: Too Few Gunicorn Workers

```json
// WRONG — single worker blocks all requests
{ "gunicorn_workers": 1 }
```

**Fix**: Set workers based on CPU cores:

```json
// Formula: (2 * CPU_cores) + 1
// For 4-core server:
{ "gunicorn_workers": 9 }
```

Minimum is ALWAYS 2. NEVER set to 1 in production.

## Anti-Pattern 10: Ignoring bench doctor Warnings

```bash
bench doctor
# Output: "scheduler disabled" or "workers not running"
# Ignored...
```

**Fix**: ALWAYS investigate `bench doctor` output:

```bash
bench doctor
# If scheduler disabled:
bench --site mysite scheduler enable

# If workers not running:
sudo supervisorctl status
sudo supervisorctl restart all

# If jobs stuck:
bench show-pending-jobs
bench --site mysite purge-jobs
```

## Anti-Pattern 11: Not Testing Restore

```bash
# WRONG — assuming backups work without testing
bench --site mysite backup  # and never testing restore
```

**Fix**: Periodically test restore on a staging site:

```bash
# Create test site
bench new-site test-restore.localhost --admin-password admin

# Restore production backup
bench --site test-restore.localhost restore /path/to/backup.sql.gz

# Verify data integrity
bench --site test-restore.localhost console
# >>> frappe.db.count("Sales Invoice")

# Clean up
bench drop-site test-restore.localhost --force
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/ops/frappe-ops-bench/references/anti-patterns.md*
