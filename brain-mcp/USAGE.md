# Using Frappe Skills via Brain MCP (and any MCP)

This guide explains how to consume the **Frappe Agent Skills** in this repo through
an MCP-based AI coding assistant. Brain MCP is used here as a concrete example, but
the skills are **MCP-agnostic** — nothing in `skills/` depends on Brain MCP. Any host
that can read a directory of Markdown files (Claude Code, Cursor, a custom MCP server,
or even a plain prompt) can use them.

> Attribution: the skills in this repository originate from

---

## 1. The SKILL.md format

Every skill is a directory under `skills/<skill-name>/` containing:

```
skills/
  frappe-doctype-development/
    SKILL.md              # required — entry document for the skill
    references/           # optional — deep-dive Markdown the skill links to
      controllers.md
      child-tables.md
```

`SKILL.md` begins with YAML frontmatter and is followed by Markdown body content:

```markdown
---
name: frappe-doctype-development
description: Create and modify DocTypes, controllers, and child tables in Frappe.
              Use when working on data models, field definitions, or document
              lifecycle hooks (validate, before_save, on_submit).
---

# Frappe DocType Development

## When to use
- Creating a new DocType or adding fields to an existing one
- Writing controller logic (validate / before_save / on_submit / on_cancel)
- Modeling parent-child relationships with child tables

## Procedure
1. ...
2. ...

## References
- See `references/controllers.md` for controller hook patterns.
```

Frontmatter fields:

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | yes | Stable identifier. Must match the directory name. Used for routing and loading. |
| `description` | yes | One- to three-sentence summary of **what the skill does and when to use it**. This is the text an assistant matches against the user's task. Write it trigger-first: name the situations that should invoke it. |

The body is free-form Markdown. The convention used across this repo is
`When to use` → `Procedure` → `References`, with detailed material pushed into
`references/*.md` so the top-level `SKILL.md` stays short and routable.

---

## 2. How an MCP-based assistant discovers and loads a skill

The flow has three stages: **discover → route → load**.

### a) Discover (catalog of names + descriptions)

On startup the host scans the skills directory and reads only the frontmatter
(`name` + `description`) of every `SKILL.md`. This gives a lightweight catalog —
just identifiers and trigger descriptions — without loading full skill bodies into
the context window.

### b) Route (pick the right skill via `frappe-router`)

`frappe-router` is the designated **entry point**. Instead of matching the user's
task against all skills blindly, the assistant loads `frappe-router` first. It
contains a decision table that maps task types to specialized skills, e.g.:

| Task | Skill |
|------|-------|
| Create/modify DocTypes, fields, controllers | `frappe-doctype-development` |
| Build REST/RPC APIs, webhooks, integrations | `frappe-api-development` |
| Customize Desk UI, form scripts, list views | `frappe-desk-customization` |
| Write or run tests | `frappe-testing` |

Routing is **by `name` and `description`**: the router (or the host's own
description-matching) selects the skill whose description best fits the task, then
the assistant loads that skill's full `SKILL.md`. Complex tasks may resolve to
several skills (e.g. a new feature = `frappe-doctype-development` +
`frappe-api-development` + `frappe-testing`).

Recommended order for any Frappe task:

1. Load `frappe-router` → choose the specialized skill(s).
2. (Recommended) Load `frappe-project-triage` → detect bench/FM, Frappe version,
   installed apps, available tooling.
3. Load the specialized skill(s) and follow its procedure.

### c) Load (pull body + referenced files on demand)

Once routed, the assistant loads the chosen `SKILL.md` body into context, and
follows links into `references/*.md` only when the procedure calls for that depth.
This keeps token usage proportional to the task.

> Brain MCP's role: Brain MCP is simply one host that performs the
> discover → route → load loop above and exposes the resulting skill text to the
> model. It does not modify or own the skills. Swapping Brain MCP for any other
> MCP host that reads the same `skills/` directory yields identical behavior.

---

## 3. Adding a new skill so the community can use it

Anyone can contribute a Frappe skill. Because skills are plain directories, adding
one is a documentation contribution, not a code change.

1. **Create the directory** under `skills/`:

   ```
   skills/frappe-<your-topic>/
     SKILL.md
     references/        # optional
   ```

2. **Write `SKILL.md`** with valid frontmatter. The `name` must equal the directory
   name. The `description` must be trigger-first so routing can find it — describe
   the tasks/situations that should invoke the skill, not just the topic.

3. **Keep it generic and version-accurate.** Skills target Frappe Framework
   v13–v15. Use current patterns: Query Builder (`frappe.qb`) over raw SQL,
   `@frappe.whitelist()` with explicit permission checks, `frappe.get_doc` /
   `frappe.get_all`, hook-based controller logic. Do **not** include secrets, API
   keys, real hostnames, private business logic, or anything client-specific —
   documentation placeholders (`your_api_key`, `eyJ...`) are fine.

4. **Register it in the router.** Add a row to the decision table in
   `skills/frappe-router/SKILL.md` so other developers' assistants can route to it:

   ```markdown
   | <when this task applies> | `frappe-<your-topic>` |
   ```

5. **List it in the catalog.** Add a row to the "Available Skills" table in
   `skills/README.md`.

6. **Open a pull request.** Once merged, every assistant pointed at this repo (Brain
   MCP or any other MCP host) picks up the new skill automatically on its next
   directory scan — no per-user install step. That is how a skill becomes usable
   "through the community": it lives in the shared repo, the router knows its name,
   and any consumer that syncs the repo can route to it.

Contribution checklist:

- [ ] `name` matches directory name
- [ ] `description` is trigger-first and routable
- [ ] No secrets / no private or client-specific data
- [ ] Frappe v13–v15 accurate (Query Builder, whitelist + permission checks)
- [ ] Router table updated
- [ ] README catalog updated

---

## 4. Generic MCP config snippet

This repo ships **skills only** — it does not ship an MCP server. To consume the
skills you point your existing MCP host at the `skills/` directory. The exact config
shape depends on your host; below is a generic illustration using placeholders.
**Do not put real keys or hostnames in a public config.**

A filesystem-style host that serves a skills directory:

```jsonc
{
  "mcpServers": {
    "frappe-skills": {
      // however your host launches a server that reads a skills directory
      "command": "<your-skill-server-command>",
      "args": ["--skills-dir", "<path-to>/forge-frappe-skill/skills"],
      "env": {
        // only if your host needs an endpoint/token — use a placeholder in
        // committed files and inject the real value from your environment
        "MCP_ENDPOINT": "<your-mcp-endpoint>",
        "MCP_API_KEY": "<your_api_key>"
      }
    }
  }
}
```

A remote/HTTP host that already serves these skills:

```jsonc
{
  "mcpServers": {
    "frappe-skills": {
      "url": "<your-mcp-endpoint>/mcp",
      "headers": {
        "Authorization": "Bearer <your_api_key>"
      }
    }
  }
}
```

Notes:

- Replace `<your-mcp-endpoint>`, `<your_api_key>`, and `<path-to>` with values from
  **your own** environment. Keep placeholders in anything you commit.
- No key, token, or real host is required to *read* the skills — they are public
  Markdown. Credentials only matter if your specific host gates access.
- Brain MCP is one such host; substitute any MCP server that can expose a directory
  of `SKILL.md` files and the skills behave the same.

---

## 5. Using without MCP at all

The skills are just Markdown. If you have no MCP host, you can still:

- Paste `skills/frappe-router/SKILL.md` into your assistant, let it choose a skill,
  then paste that skill's `SKILL.md`.
- Drop the `skills/` directory into a tool that auto-loads project skill files
  (e.g. an assistant that reads a local skills folder directly).

MCP is a convenience for automatic discovery and routing — not a requirement.
