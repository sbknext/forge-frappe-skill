---
name: frappe-bench-cli
description: >
  Operate a Frappe bench from the command line — site creation, app install, migrate, build,
  backup/restore, scheduler, console, set-config, and multi-site management. Use when running or
  troubleshooting bench commands, deploying changes, or managing sites. Trigger on: "bench command",
  "bench migrate", "bench new-site", "bench backup", "bench console", "set-config", "bench build",
  "scheduler", "bench restore".
---

# Frappe Bench CLI

`bench` is the command-line tool that manages Frappe sites and apps. On a multi-site bench, **always
pass `--site <site>`**.

## Bench init & directory layout

```bash
bench init frappe-bench    # creates env/, apps/, sites/, config/, logs/
cd frappe-bench && bench start   # web :8000 + redis + workers + scheduler
```

## Sites & apps

```bash
bench new-site mysite.localhost                       # create a site
bench --site mysite.localhost add-to-hosts            # map site hostname → 127.0.0.1
bench --site mysite.localhost install-app myapp       # install an app on it
bench --site mysite.localhost uninstall-app myapp
bench get-app https://github.com/org/myapp            # fetch an app into the bench
bench --site mysite.localhost list-apps
bench use mysite.localhost                            # set the default site
```

## The everyday loop

```bash
bench --site <site> migrate          # apply schema/patches after any DocType or hooks change
bench build --app myapp              # rebuild JS/CSS assets
bench --site <site> clear-cache
bench restart                        # restart web + workers (production/supervisor)
bench start                          # dev: run web, workers, scheduler in the foreground
```

## Backup & restore

```bash
bench --site <site> backup                      # DB only
bench --site <site> backup --with-files         # DB + public/private files
bench --site <site> restore /path/to/db.sql.gz  # restore a backup
bench --site <site> --force restore <db> --with-public-files <tar> --with-private-files <tar>
```

## Scheduler

```bash
bench --site <site> enable-scheduler
bench --site <site> scheduler status
bench --site <site> trigger-scheduler-event daily     # run a hook on demand
bench --site <site> doctor                            # health check (workers/redis/scheduler)
```

## Console & config

```bash
bench --site <site> console            # interactive Python shell with frappe loaded
bench --site <site> execute myapp.tasks.daily.run     # run one function
bench --site <site> set-config key value               # writes to site_config.json
bench --site <site> set-config -g key value            # global common_site_config.json
bench --site <site> show-config
bench --site <site> mariadb            # open a DB shell
```

## Users & maintenance

```bash
bench --site <site> set-admin-password <pwd>
bench --site <site> add-system-manager user@x --first-name A
bench --site <site> set-maintenance-mode on
bench update --patch            # pull + migrate (be careful in prod; backup first)
```

## Rules

- **Backup before** `migrate`, `restore`, `update`, or anything destructive in production.
- Always `--site` on multi-site benches — a missing flag can hit the wrong site.
- Store secrets via `set-config` (→ `site_config.json` / `frappe.conf`), never in DocType fields or code.
- After editing `hooks.py`, DocType JSON, or patches: `bench --site <site> migrate`.
- After editing JS/CSS: `bench build` (use `bench build --app myapp` to scope it).
- Don't run `bench update` blindly on prod — pin app versions and test on staging first.

Source: https://docs.frappe.io/framework/user/en/tutorial , https://docs.frappe.io/framework/user/en/tutorial/create-a-site
