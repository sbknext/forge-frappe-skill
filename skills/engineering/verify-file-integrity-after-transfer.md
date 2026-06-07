---
title: Verify file integrity after transfer
category: engineering
tags: ['ops', 'safety', 'data']
source: Frappe community
---

Checksum both sides after every scp/rsync; a corrupt transfer is silent without verification.

## When
After any file transfer: scp, rsync, git bundle deploy, S3 download, or any binary file move across a network boundary.

## Why
Network transfers fail silently in subtle ways. A file may land with correct byte count but corrupt content (partial write, decompression error, network bit flip). A git bundle with missing refs will only fail when you try to use it. Detecting corruption after the fact — when a service fails to start — is much harder than detecting it at transfer time.

## How
```bash
# Before transfer: checksum the source
md5sum myfile.tar.gz
# → abc123def456  myfile.tar.gz

# Transfer
scp myfile.tar.gz user@host:/tmp/

# After transfer: checksum the destination
ssh user@host 'md5sum /tmp/myfile.tar.gz'
# → abc123def456  /tmp/myfile.tar.gz  ← must match

# For git bundles specifically: verify contents
git bundle verify /tmp/deploy.bundle
git bundle list-heads /tmp/deploy.bundle
# Ensure the branch you need is listed before attempting git fetch
```

For large deploys, add a syntax check after transfer if applicable:
```bash
ssh host 'node --check /path/to/script.js'
ssh host 'python -m py_compile script.py'
```

## Anti-pattern
SCP → restart service immediately, assuming transfer was perfect. The service startup is not a substitute for transfer verification.
