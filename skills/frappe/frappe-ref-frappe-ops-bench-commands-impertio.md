---
title: Frappe Ref — Frappe Ops: Bench: Commands (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'ops', 'bench', 'reference', 'commands']
source: Impertio
---

# Bench Commands — Full Reference

## General Commands

| Command | Description |
|---------|-------------|
| `bench init [bench-name]` | Create new bench instance |
| `bench init [name] --frappe-branch version-15` | Create bench with specific Frappe version |
| `bench init [name] --python python3.11` | Create bench with specific Python |
| `bench --version` | Display bench version |
| `bench version` | Show version of all installed apps |
| `bench version -f` | Show version with branch and commit info |
| `bench src` | Display bench repo directory |
| `bench start` | Start development server (Procfile) |
| `bench serve` | Start web server only |
| `bench restart` | Restart production services (supervisor/systemd) |
| `bench update` | Pull, migrate, build, restart |
| `bench update --reset` | Reset to upstream (destroys local changes) |
| `bench update --no-build` | Skip asset build step |
| `bench update --no-backup` | Skip backup step |
| `bench update --pull --app [app]` | Update specific app only |
| `bench --help` | List all commands |
| `bench [command] --help` | Help for specific command |

## Site Commands

| Command | Description |
|---------|-------------|
| `bench new-site [site]` | Create new site |
| `bench new-site [site] --admin-password [pw]` | Create site with password |
| `bench new-site [site] --install-app [app]` | Create site and install app |
| `bench new-site [site] --db-name [name]` | Create site with specific database name |
| `bench new-site [site] --mariadb-root-password [pw]` | Provide MariaDB root password |
| `bench use [site]` | Set default site |
| `bench drop-site [site] --force` | Delete site and database |
| `bench --site [site] reinstall` | Fresh reinstall (deletes all data) |
| `bench --site [site] browse` | Open site in browser |
| `bench --site [site] browse --user [email]` | Open with auto-login |
| `bench --site [site] add-to-hosts` | Add site to /etc/hosts |

## App Commands

| Command | Description |
|---------|-------------|
| `bench get-app [app/url]` | Download app from repository |
| `bench get-app [url] --branch [branch]` | Download specific branch |
| `bench --site [site] install-app [app]` | Install app on site |
| `bench --site [site] uninstall-app [app]` | Remove app and its data |
| `bench --site [site] list-apps` | List installed apps |
| `bench remove-app [app]` | Remove app from bench |
| `bench new-app [app-name]` | Create new app scaffold |
| `bench switch-to-branch [branch] [apps...]` | Switch apps to branch |
| `bench exclude-app [app]` | Exclude from updates |
| `bench include-app [app]` | Re-include in updates |

## Migration and Build

| Command | Description |
|---------|-------------|
| `bench --site [site] migrate` | Run patches, sync schema, rebuild |
| `bench --site [site] ready-for-migration` | Check for pending jobs |
| `bench --site all migrate` | Migrate all sites |
| `bench build` | Build assets for all apps |
| `bench build --app [app]` | Build assets for specific app |
| `bench build --production` | Production build with minification |
| `bench watch` | Watch and rebuild on file changes |
| `bench --site [site] clear-cache` | Clear all caches |
| `bench clear-website-cache` | Clear website cache |

## Backup and Restore

| Command | Description |
|---------|-------------|
| `bench --site [site] backup` | Backup database + files |
| `bench --site [site] backup --backup-encryption-key [key]` | Encrypted backup |
| `bench backup-all-sites` | Backup all sites |
| `bench --site [site] restore [path]` | Restore from .sql.gz |
| `bench --site [site] restore [path] --with-public-files [tar]` | Restore with public files |
| `bench --site [site] restore [path] --with-private-files [tar]` | Restore with private files |
| `bench --site [site] partial-restore [path]` | Partial restore to existing site |

## Scheduler and Jobs

| Command | Description |
|---------|-------------|
| `bench --site [site] scheduler enable` | Enable scheduler |
| `bench --site [site] scheduler disable` | Disable scheduler |
| `bench --site [site] scheduler pause` | Pause scheduler |
| `bench --site [site] scheduler resume` | Resume scheduler |
| `bench doctor` | Scheduler diagnostics for all sites |
| `bench show-pending-jobs` | Display queued background jobs |
| `bench --site [site] purge-jobs` | Remove pending periodic tasks |
| `bench --site [site] purge-jobs --event [name]` | Purge specific event |
| `bench --site [site] trigger-scheduler-event [event]` | Manually trigger event |
| `bench --site [site] set-maintenance-mode [0/1]` | Toggle maintenance mode |
| `bench worker --queue [queue]` | Start specific queue worker |
| `bench schedule` | Start scheduler process |

## Console and Debugging

| Command | Description |
|---------|-------------|
| `bench --site [site] console` | IPython console with Frappe loaded |
| `bench --site [site] console --autoreload` | Console with auto-reload |
| `bench --site [site] mariadb` | MariaDB interactive console |
| `bench --site [site] postgres` | PostgreSQL interactive console |
| `bench db-console` | Database console (auto-detects DB type) |
| `bench --site [site] jupyter` | Start Jupyter Notebook |
| `bench execute [method]` | Execute Python method |
| `bench execute [method] --args '[...]'` | Execute with positional args |
| `bench execute [method] --kwargs '{...}'` | Execute with keyword args |
| `bench --site [site] request [method] [path]` | Authenticated HTTP request |
| `bench --site [site] start-recording` | Start Frappe Recorder |
| `bench --site [site] stop-recording` | Stop Frappe Recorder |
| `bench ngrok` | Create shareable URL for local site |

## Configuration

| Command | Description |
|---------|-------------|
| `bench --site [site] set-config [key] [value]` | Set site config value |
| `bench config set-common-config -c [key] [value]` | Set common config value |
| `bench config remove-common-config [key]` | Remove common config key |
| `bench --site [site] show-config` | Display site configuration |
| `bench config dns_multitenant on/off` | Toggle DNS multi-tenancy |
| `bench config restart_supervisor_on_update on/off` | Auto-restart supervisor |
| `bench config restart_systemd_on_update on/off` | Auto-restart systemd |
| `bench config http_timeout [seconds]` | Set HTTP timeout |
| `bench set-nginx-port [site] [port]` | Set site port for nginx |

## User Management

| Command | Description |
|---------|-------------|
| `bench --site [site] add-system-manager [email]` | Add system manager |
| `bench --site [site] add-user [email] --roles [roles]` | Add user with roles |
| `bench --site [site] disable-user [email]` | Disable user account |
| `bench --site [site] set-password [user] [password]` | Set user password |
| `bench --site [site] set-admin-password [password]` | Set Administrator password |
| `bench --site [site] destroy-all-sessions` | Force logout all users |

## Data Import/Export

| Command | Description |
|---------|-------------|
| `bench --site [site] data-import` | Import from CSV/XLSX |
| `bench --site [site] import-csv [path]` | Import via CSV |
| `bench --site [site] import-doc [path]` | Import from JSON files |
| `bench --site [site] export-csv [doctype]` | Export DocType data to CSV |
| `bench --site [site] export-doc [doctype] [name]` | Export single document |
| `bench --site [site] export-json [doctype] [name]` | Export as JSON |
| `bench --site [site] export-fixtures` | Export fixtures to app |
| `bench --site [site] export-fixtures --app [app]` | Export fixtures for specific app |
| `bench --site [site] bulk-rename [csv_path]` | Bulk rename from CSV |

## Setup Commands

| Command | Description |
|---------|-------------|
| `bench setup production [user]` | Full production setup |
| `bench setup nginx` | Generate nginx configuration |
| `bench setup supervisor` | Generate supervisor configuration |
| `bench setup redis` | Generate Redis configuration |
| `bench setup lets-encrypt [site]` | Configure SSL certificate |
| `bench setup env` | Configure Python virtual environment |
| `bench setup requirements` | Install Python and Node dependencies |
| `bench setup add-domain [domain] --site [site]` | Add custom domain |
| `bench disable-production` | Disable production setup |

## Database Maintenance

| Command | Description |
|---------|-------------|
| `bench --site [site] transform-database --tables [list]` | Modify table engine/format |
| `bench trim-tables` | Remove orphaned database columns |
| `bench trim-database` | Delete ghost tables from deleted DocTypes |
| `bench --site [site] reload-doctype [doctype]` | Reload DocType schema |
| `bench --site [site] reload-doc [doctype] [name]` | Reload specific document |
| `bench --site [site] reset-perms` | Reset permissions to defaults |
| `bench --site [site] build-search-index` | Rebuild full-text search index |
| `bench rebuild-global-search` | Rebuild global help search |

## Testing

| Command | Description |
|---------|-------------|
| `bench --site [site] run-tests` | Run Python unit tests |
| `bench --site [site] run-tests --app [app]` | Run tests for specific app |
| `bench --site [site] run-tests --doctype [doctype]` | Run tests for DocType |
| `bench run-parallel-tests` | Run tests in parallel (CI) |
| `bench run-ui-tests` | Run Cypress UI tests |

## Translation (Gettext/PO)

| Command | Description |
|---------|-------------|
| `bench generate-pot-file` | Create translation template |
| `bench create-po-file [app] [lang]` | Create new PO file |
| `bench update-po-files` | Sync PO files with POT |
| `bench compile-po-to-mo` | Compile PO to binary MO |
| `bench migrate-csv-to-po [app]` | Convert CSV to PO format |

## Misc

| Command | Description |
|---------|-------------|
| `bench --site [site] run-patch execute:[code]` | Run one-off Python patch |
| `bench --site [site] publish-realtime [event] [data]` | Publish realtime event |
| `bench migrate-to [provider]` | Migrate to hosting provider |
| `bench migrate-env [python-version]` | Migrate virtual environment |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/ops/frappe-ops-bench/references/commands.md*
