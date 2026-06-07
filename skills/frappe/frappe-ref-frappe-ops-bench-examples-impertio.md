---
title: Frappe Ref — Frappe Ops: Bench: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'ops', 'bench', 'reference', 'examples']
source: Impertio
---

# Bench Examples — Common Workflows

## Example 1: Fresh Development Setup

```bash
# Install bench (one-time)
pip install frappe-bench

# Create bench with Frappe v15
bench init my-erp --frappe-branch version-15
cd my-erp

# Get ERPNext
bench get-app erpnext --branch version-15

# Create site
bench new-site dev.localhost --admin-password admin --install-app erpnext

# Set as default
bench use dev.localhost

# Enable developer mode
bench --site dev.localhost set-config developer_mode 1

# Start development server
bench start
# Site available at http://dev.localhost:8000
```

## Example 2: Adding a Custom App

```bash
# Create new app
bench new-app custom_erp

# Install on site
bench --site dev.localhost install-app custom_erp

# After making changes to DocTypes/hooks:
bench --site dev.localhost migrate
bench build --app custom_erp
```

## Example 3: Production Deployment

```bash
# Create bench
bench init /opt/frappe-bench --frappe-branch version-15
cd /opt/frappe-bench

# Get apps
bench get-app erpnext --branch version-15

# Create production site
bench new-site erp.example.com --admin-password SecurePass123 --install-app erpnext

# Setup production (as root or sudo)
sudo bench setup production frappe

# Setup SSL
bench setup lets-encrypt erp.example.com

# Enable scheduler
bench --site erp.example.com scheduler enable
```

## Example 4: Multi-Tenant Setup

```bash
# Enable DNS multi-tenancy
bench config dns_multitenant on

# Create multiple sites
bench new-site company-a.example.com --admin-password pass1 --install-app erpnext
bench new-site company-b.example.com --admin-password pass2 --install-app erpnext

# Regenerate nginx config (includes all sites)
bench setup nginx
sudo service nginx reload

# Each site is now accessible via its hostname
```

## Example 5: Backup and Restore to New Site

```bash
# Create backup
bench --site production.example.com backup
# Output:
# Database: sites/production.example.com/private/backups/20240115_120000-production-database.sql.gz
# Files: sites/production.example.com/private/backups/20240115_120000-production-files.tar
# Private: sites/production.example.com/private/backups/20240115_120000-production-private-files.tar

# Create new site for restore
bench new-site staging.example.com --admin-password admin

# Restore
bench --site staging.example.com restore \
  sites/production.example.com/private/backups/20240115_120000-production-database.sql.gz \
  --with-public-files sites/production.example.com/private/backups/20240115_120000-production-files.tar \
  --with-private-files sites/production.example.com/private/backups/20240115_120000-production-private-files.tar

# Run migrations (in case versions differ)
bench --site staging.example.com migrate
```

## Example 6: Debugging with Console

```bash
bench --site dev.localhost console

# In console:
>>> import frappe

# Query documents
>>> frappe.get_all("Sales Invoice", filters={"status": "Unpaid"}, limit=5)

# Get specific document
>>> doc = frappe.get_doc("Sales Invoice", "INV-001")
>>> doc.grand_total

# Direct SQL
>>> frappe.db.sql("SELECT COUNT(*) FROM `tabSales Invoice` WHERE status='Paid'")

# Test a method
>>> from myapp.api import process_invoice
>>> process_invoice("INV-001")

# Commit changes (if needed)
>>> frappe.db.commit()
```

## Example 7: Updating Production

```bash
# Put site in maintenance (optional)
bench --site erp.example.com set-maintenance-mode 1

# Check no pending jobs
bench --site erp.example.com ready-for-migration

# Update
bench update

# Verify
bench --site erp.example.com list-apps
bench doctor

# Remove maintenance mode
bench --site erp.example.com set-maintenance-mode 0
```

## Example 8: common_site_config.json for Production

```json
{
  "background_workers": 4,
  "dns_multitenant": true,
  "gunicorn_workers": 9,
  "gunicorn_max_requests": 5000,
  "redis_cache": "redis://localhost:13000",
  "redis_queue": "redis://localhost:11000",
  "redis_socketio": "redis://localhost:13000",
  "restart_supervisor_on_update": true,
  "serve_default_site": true,
  "scheduler_tick_interval": 60,
  "socketio_port": 9000,
  "webserver_port": 8000
}
```

**Rule of thumb for gunicorn_workers**: `(2 * CPU_cores) + 1`

## Example 9: Scheduler Troubleshooting

```bash
# Check overall health
bench doctor
# Output shows: scheduler status, active workers, queue lengths

# View pending jobs
bench show-pending-jobs

# If scheduler is stuck:
bench --site mysite scheduler disable
bench --site mysite purge-jobs
bench --site mysite scheduler enable

# Test a scheduler task manually
bench --site mysite trigger-scheduler-event hourly

# Or execute directly
bench --site mysite execute myapp.tasks.daily_cleanup
```

## Example 10: Database Maintenance

```bash
# Remove orphaned columns (safe — only removes columns for deleted fields)
bench --site mysite trim-tables

# Remove ghost tables from deleted DocTypes
bench --site mysite trim-database

# Reset all permissions to default
bench --site mysite reset-perms

# Rebuild search index
bench --site mysite build-search-index
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/ops/frappe-ops-bench/references/examples.md*
