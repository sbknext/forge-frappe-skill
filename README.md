# forge-frappe-skill

Production-grade **Agent Skills for Frappe / ERPNext** development, for use with Claude Code (and any
`npx skills`-compatible agent). Each skill is a focused, community-safe playbook of patterns,
guardrails, and copy-paste snippets — no framework lock-in, no proprietary references.

## Install

Add a single skill:

```bash
npx skills add https://github.com/sbknext/forge-frappe-skill/frappe-app-development
```

Or browse and add any of the skills below.

## Skills

| Skill | What it covers |
|---|---|
| **frappe-app-development** | Scaffold & architect custom apps — structure, hooks, jobs, service layer, APIs |
| **frappe-doctype-design** | Field types, naming series, child tables, links, Single doctypes, `fetch_from` |
| **frappe-hooks** | `hooks.py` — doc_events, scheduler_events, overrides, permission query conditions |
| **frappe-background-jobs** | `frappe.enqueue`, RQ, queues, timeouts, job dedup, progress, idempotency |
| **frappe-rest-api** | `@frappe.whitelist`, `/api/resource` & `/api/method`, token/session auth, uploads |
| **frappe-permissions** | Roles, perm levels, user permissions, row-level security, `has_permission` |
| **frappe-database-orm** | `get_all`/`get_list`, parameterized SQL, Query Builder, transactions, performance |
| **frappe-client-scripting** | `frappe.ui.form.on`, field events, `set_query`, custom buttons, list settings |
| **frappe-report-development** | Query & Script reports, filters, columns, charts, formatters |
| **frappe-bench-cli** | Site/app management, migrate, build, backup/restore, scheduler, console, config |

## Conventions

Every skill follows the same shape:

- **When to use** — the triggers that should pull the skill in
- **Core patterns** — tested, copy-paste snippets
- **Rules / guardrails** — the security and correctness lines you don't cross

## Contributing

PRs welcome. Keep skills **generic and self-contained** — no company-specific sites, client names,
credentials, or proprietary app references. Each skill lives in its own directory with a single
`SKILL.md` (frontmatter `name` + `description`, then the body).

## License

MIT
