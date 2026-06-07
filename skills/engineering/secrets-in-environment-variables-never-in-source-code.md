---
title: Secrets in environment variables, never in source code
category: engineering
tags: ['security', 'process']
source: Frappe community
---

API keys, DB passwords, and tokens belong in environment config — committing them exposes them forever.

## When
Any time a value is secret: API keys, database passwords, JWT signing secrets, OAuth credentials, webhook tokens.

## Why
Git history is forever. Even if you delete the secret in the next commit, the original commit remains in history and can be recovered by anyone who has ever cloned the repository. GitHub and GitLab scanners find secrets in old commits routinely.

Secrets in code are also harder to rotate: you have to change code, test, and deploy just to update a credential. With environment variables, rotation is a config change with no code deploy required.

## How
```bash
# .env (gitignored)
DATABASE_URL=postgres://user:password@localhost/mydb
API_KEY=sk-...
JWT_SECRET=your-signing-secret
```

```js
// Load at startup
import 'dotenv/config';

// Access safely
const apiKey = process.env.API_KEY;
if (!apiKey) throw new Error('API_KEY is required');
```

Pre-commit check:
```bash
# Add to .git/hooks/pre-commit
if git diff --cached | grep -qE '(sk-[a-zA-Z0-9]{30,}|AIzaSy|ghp_|password\s*=\s*["'\''\`])'; then
  echo "STOP: possible secret in diff. Use env vars."
  exit 1
fi
```

If a secret was committed: rotate it immediately, then optionally clean history. Rotation first — history cleanup is optional. Unrotated secrets in history are active attack surface.

## Anti-pattern
`const API_KEY = 'sk-live-abc123...'` directly in source. Or `.env` in `.gitignore` but forgetting to add it after `git init`.
