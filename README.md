# forge-frappe-skill

Production-grade **Agent Skills for Frappe / ERPNext** — for Claude Code, any `npx skills`-compatible
agent, Forge MCP, or plain markdown use. Two layers:

1. **Core skills** (repo root) — 11 focused, community-safe, *installable* playbooks. Hand-authored, generic, no lock-in.
2. **Reference corpus** (`skills/`) — 471 granular, curated skills (Frappe + Engineering + everyday-ops) aggregated from public OSS sources for deep lookups.

> Curation + aggregation, not original authorship of the corpus — see **Credit** below.

---

## Install a core skill

```bash
npx skills add https://github.com/sbknext/forge-frappe-skill/frappe-app-development
```

### Core skills (root, installable)

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
| **frappe-testing** | Unit + integration tests, fixtures, CI |

These 11 are original to this repo (community-safe, no proprietary references) — the clean, installable face.

---

## Reference corpus (`skills/`)

A deeper, granular library for lookups — usable **via Forge MCP** (or any MCP host that serves a skills
directory) **and standalone** (drop the `.md` files into any markdown-skill agent).

```
skills/
├── frappe/        (365)  DocTypes, controllers, hooks, APIs, Query Builder, permissions,
│                          reports, print formats, Vue/Frappe UI, web forms, testing, ops
├── engineering/   (28)   language-agnostic engineering safety + patterns
├── health-yoga/   (48)   generic Ayurveda / wellness / yoga routines
├── productivity/  (15)   work productivity routines
└── life-ops/      (15)   personal life-admin checklists
```

Full catalog with tags: **[INDEX.md](INDEX.md)**. Each corpus file:
```
---
title: <human title>
category: <folder>
tags: [..]
source: <Impertio | lubus | Frappe community | netchampfaris>
---
<markdown body>
```

The core skills (root) are curated overviews; the corpus (`skills/`) is the deep reference layer — complementary, not duplicates.

---

## Credit & attribution — please support upstream

The **reference corpus** was assembled from public, openly-licensed sources, then curated + used in real
Frappe work on a Forge MCP community hub over recent months:

| Source | Skills | What |
|---|---|---|
| **[Impertio-Studio/frappe-skills](https://github.com/Impertio-Studio/frappe-skills)** (MIT) | 333 | Largest source — deterministic Frappe/ERPNext dev + ops skills. **Star the upstream.** |
| **Frappe community + [official Frappe docs](https://docs.frappe.io)** | 124 | Public Frappe docs + community knowledge (incl. the official [frappe/frappe-agent-skills](https://github.com/frappe/frappe-agent-skills) set). |
| **[lubusIN/frappe-skills](https://github.com/lubusIN/frappe-skills)** (MIT, © lubus) | 13 | Net-new depth — Frappe Manager, frappe-ui patterns, SLA/enterprise architecture, rate-limiting. **Star the upstream.** |
| **[netchampfaris](https://github.com/netchampfaris)** | 1 | Frappe core contributor — entry-point skill. |

The **11 core skills** (root) are original to this repo. If your work is represented here and you'd like
different attribution (or removal), open an issue — happy to fix.

---

## Contributing

PRs welcome. Keep skills **generic + self-contained** — no company-specific sites, client names,
credentials, or proprietary app references. Core skills live in their own root dir with a single
`SKILL.md` (frontmatter `name` + `description`, then body). Corpus skills go under `skills/<category>/`
with the frontmatter shape above. See **[PLAYBOOK.md](PLAYBOOK.md)** + **[MISTAKES.md](MISTAKES.md)**.

Rules: no secrets ever · no private/internal data · Frappe-accurate (v13–v15) · credit your source.

---

## License

**MIT** — see [LICENSE](LICENSE). Preserves upstream licensing (Impertio-Studio/frappe-skills +
lubusIN/frappe-skills are MIT; Frappe docs/community under their original terms). If any attribution is
incorrect, open an issue.
