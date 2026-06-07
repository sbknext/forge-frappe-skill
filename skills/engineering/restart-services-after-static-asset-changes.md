---
title: Restart services after static asset changes
category: engineering
tags: ['ops', 'reliability']
source: Frappe community
---

Servers that cache static assets or compiled code on startup need a restart to pick up new files.

## When
After adding, replacing, or modifying: static assets (images, CSS, JS bundles), compiled templates, config files read at startup, or any file that the server reads once and caches in memory.

## Why
Many servers load static assets or compiled resources into memory at startup and never re-read them during normal operation. Deploying new files to disk has no effect until the server restarts and re-reads them. The symptom — "I deployed but nothing changed" — is confusing and wastes time diagnosing the wrong thing.

This is especially common with: nginx (config reload required), Node.js servers with in-memory template caches, Python with `.pyc` bytecode, and any CDN-backed service.

## How
```bash
# After deploying static assets
nginx -t && systemctl reload nginx  # config reload (no connection drop)

# After deploying new app code
systemctl restart app-name  # full restart required for in-memory cache invalidation

# After changing environment variables
systemctl restart app-name  # env is read at startup only

# Verify the new version is running
curl -s https://example.com/api/version | jq .version
```

For zero-downtime restarts:
```bash
# Node.js with pm2
pm2 reload app-name  # graceful rolling restart, no connection drop

# systemd with socket activation
systemctl reload-or-restart app-name
```

## Anti-pattern
Deploying new files → verifying on the server that files exist → concluding "deploy worked" without restarting the service and confirming the running process actually serves the new assets.
