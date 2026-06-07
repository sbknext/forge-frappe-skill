---
title: SSH keys, never password injection
category: engineering
tags: ['security', 'ops', 'ssh']
source: Frappe community
---

Use key-based SSH for all automation; never pipe a password into sshpass or a subprocess.

## When
Any automated or semi-automated process that needs to SSH into a remote host.

## Why
`sshpass -p 'secretpassword' ssh user@host` exposes the secret to `ps auxe` — any logged-in user on the machine can see it. The password also lands in shell history unless `HISTCONTROL=ignorespace` is set (and that has to be remembered every time). SSH keys have neither of these attack surfaces.

If you already have a working key-based setup, injecting a password is not "a fallback" — it is a regression in security posture for zero gain.

## How
```bash
# Canonical SSH command (key-based)
ssh -o StrictHostKeyChecking=no \
    -o PubkeyAuthentication=yes \
    user@hostname 'your command here'

# If key isn't loading:
ssh-add ~/.ssh/id_ed25519  # add to agent
ssh-add -l                  # verify it's loaded
ssh -v user@host            # diagnose what's being tried
```

For automation scripts, add the key once (`ssh-add`) or use `SSH_AUTH_SOCK` in the environment. Never embed the password string anywhere in the code.

If the key auth is broken, fix the key. Do not solve a key problem with a password workaround.

## Anti-pattern
```bash
# NEVER
sshpass -p "$PROD_PASSWORD" ssh root@server 'deploy.sh'
# Password visible in: ps output, shell history, log files, Docker inspect output
```
