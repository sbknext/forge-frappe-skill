# forge-frappe-skill

**A curated, categorized library of 471 agent-skills** — Frappe-focused, plus Engineering, Health/Yoga, Productivity, and Life-Ops. Each skill is a self-contained Markdown file (frontmatter + body) that any AI coding assistant can load to do real work — write idiomatic Frappe code (DocTypes, hooks, Query Builder, permissions, REST/RPC APIs, reports, Vue frontends, testing, ops) or follow a practical routine — instead of guessing.

Usable **via Brain MCP** (or any MCP host that serves a skills directory) **and standalone** — drop the `.md` files into any agent that reads Markdown skills (Claude Code, Cursor, or a plain prompt).

> **This repo is a curation + aggregation, not original authorship.** The bulk of the technical knowledge here was written by others (see **Credit** below). What this project adds is selection, a clean domain taxonomy, consistent frontmatter, de-duplication, and packaging for agent consumption.

---

## Credit & attribution — please support upstream

This library was assembled from public, openly-licensed sources, then curated and used in real Frappe work on a Brain MCP community hub over recent months. The real provenance (from each skill's `source:` frontmatter):

| Source | Skills | What |
|---|---|---|
| **[Impertio-Studio/frappe-skills](https://github.com/Impertio-Studio/frappe-skills)** (MIT) | 333 | The largest source — a deterministic set of Frappe Framework & ERPNext development/operations skills. **Thank you, Impertio Studio — go star the upstream.** |
| **Frappe community + [official Frappe docs](https://docs.frappe.io)** | 124 | Public Frappe Framework documentation + community knowledge, distilled into skills. |
| **[lubusIN/frappe-skills](https://github.com/lubusIN/frappe-skills)** (MIT, © lubus) | 13 | Net-new depth — Frappe Manager (fm), frappe-ui SPA/UX patterns, SLA + enterprise app architecture, custom API rate-limiting. **Thank you, lubus — go star the upstream.** |
| **[netchampfaris](https://github.com/netchampfaris)** | 1 | Frappe core contributor — entry-point skill. |

If your work is represented here and you'd like different attribution (or removal), open an issue — happy to fix.

---

## Categories

```
skills/
├── frappe/        (365)  DocTypes, controllers, hooks, APIs, Query Builder, permissions,
│                          reports, print formats, Vue/Frappe UI, web forms, testing, ops
├── engineering/   (28)   language-agnostic engineering safety + patterns
├── health-yoga/   (48)   generic Ayurveda / wellness / yoga routines
├── productivity/  (15)   work productivity routines
└── life-ops/      (15)   personal life-admin checklists
```

Frappe + Engineering are the primary focus; the rest are bonus everyday-ops skills. Full catalog with tags + descriptions: see **[INDEX.md](INDEX.md)**.

Each skill file:
```
---
title: <human title>
category: <folder>
tags: [..]
source: <Impertio | Frappe community | lubus | netchampfaris>
---
<markdown body — when to use, procedure, references>
```

---

## How to use

**Standalone (no MCP):** clone the repo and point your agent at `skills/`. For Claude Code / Cursor, copy the relevant `skills/<category>/*.md` into your agent's skills directory, or paste a skill body into context for the task at hand.

**Via Brain MCP / any MCP host:** serve the `skills/` directory through your MCP skills endpoint; the assistant discovers a skill by its `title` + `tags` and loads the body on demand. Start a Frappe task by letting the agent pick the matching `frappe/` skill. (Brain MCP is one consumer — these skills are MCP-agnostic.)

---

## Contributing

PRs welcome — add a `.md` skill under the right `skills/<category>/` folder using the frontmatter shape above. Rules:

- **No secrets, ever** — no API keys/tokens/passwords/real JWTs. Documentation placeholders only.
- **No private/internal data** — no company hostnames, client names, real personal info.
- **Frappe-accurate** — match the framework (v13–v15); note version-specific behavior.
- **Credit your source** in the `source:` field.

See **[PLAYBOOK.md](PLAYBOOK.md)** (authoring rules) and **[MISTAKES.md](MISTAKES.md)** (common Frappe pitfalls).

---

## License

**MIT** — see [LICENSE](LICENSE). This curation preserves the licensing of its upstream sources (Impertio-Studio/frappe-skills + lubusIN/frappe-skills are MIT). Skills sourced from Frappe documentation/community remain under their original terms. If any attribution is incorrect, open an issue.
