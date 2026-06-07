---
title: Immutable infrastructure snapshots
category: engineering
tags: ['ops', 'safety', 'reliability']
source: Frappe community
---

Before major infrastructure changes, snapshot the full machine image — rollback becomes boot from snapshot.

## When
Before: OS upgrades, kernel updates, major dependency upgrades (Node version, Python version, database engine), or any change to the base system rather than the application layer.

## Why
Application-layer backups (DB dumps, git backup branches) don't help when the OS upgrade breaks a critical system library, the Node.js version upgrade breaks a native module, or a kernel update breaks a hardware driver. Machine-level snapshots give you a restore point for the entire system state.

Cloud providers offer this natively and cheaply. A snapshot of a 20 GB disk takes seconds to create and costs cents per month to retain. Rebuilding a broken server from scratch costs hours.

## How
Cloud (AWS, GCP, DigitalOcean, etc.):
```bash
# DigitalOcean (doctl CLI)
doctl compute droplet-action snapshot <droplet-id> \
  --snapshot-name "pre-upgrade-$(date +%Y%m%d)"

# AWS EC2
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "pre-upgrade-$(date +%Y%m%d)" \
  --no-reboot
```

For on-premise or when cloud snapshots aren't available:
```bash
# LVM snapshot (if root volume is LVM)
lvcreate -L10G -s -n snap-pre-upgrade /dev/vg0/root

# Document the snapshot in your runbook
echo "$(date): Created snapshot pre-upgrade-$(date +%Y%m%d) before Node.js upgrade" \
  >> /var/log/infrastructure-changes.log
```

## Anti-pattern
"The upgrade should be reversible with apt-get downgrade." Package manager downgrade is theoretically possible and practically unreliable for major version changes.
