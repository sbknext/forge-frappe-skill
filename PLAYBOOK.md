# Playbook — Using & Authoring Frappe Agent Skills Safely

Operational rules for AI assistants (and humans) **using** these skills to write
Frappe code, and for contributors **authoring** new skills. These are generic,
public, framework-level rules — no project- or company-specific assumptions.

Companion file: `MISTAKES.md` (the failure catalog these rules prevent).

---

## Part 1 — Using the skills

### R1. Route through `frappe-router` first

`frappe-router` is the entry point. Given a task ("create a DocType", "build an
API", "write a report"), it selects the right specialized skill. Don't guess
which skill to read — let the router map the task, then load that skill's
`SKILL.md` and only the `references/*.md` you need.

### R2. Triage the project before writing code

Run / follow `frappe-project-triage` before the first edit. Establish:

- **Frappe version** (`bench version` or `frappe.__version__`) — APIs differ
  across v13/v14/v15 (see MISTAKES #16). Target the version actually installed.
- **Installed apps** (`bench --site <site> list-apps`) — ERPNext present? Other
  custom apps? This tells you which DocTypes and tooling already exist.
- **Existing tooling** — package manager, lint/format config, test setup, CI.
  Match the project's conventions instead of importing your own.

Never start coding against an assumed version.

### R3. Verify before you write — never invent DocType fields

Before referencing any field, DocType, or method, confirm it exists:

- Read the DocType JSON: `<app>/<module>/doctype/<dt>/<dt>.json`.
- Or query meta: `frappe.get_meta("<DocType>").fields` in `bench console`.
- For methods/whitelisted endpoints, grep the app source.

If you cannot verify a field/DocType exists, **stop and say so** — do not emit
plausible-but-unverified field names. This is the most common AI-generated Frappe
bug (MISTAKES #10).

### R4. Prefer existing app tooling over reinventing

- Use `bench` commands (`new-doctype` scaffolding, `migrate`, `build`,
  `run-tests`) rather than hand-rolling equivalents.
- Reuse the project's existing utilities, service-layer modules, and base
  controllers before writing new ones.
- For UI, reuse Frappe UI / desk patterns the app already uses (see
  `frappe-ui-patterns`) rather than introducing a parallel stack.

### R5. Security is non-negotiable on every API path

When generating any `@frappe.whitelist()` method, the skill output MUST include:

- An explicit permission check (`frappe.has_permission` / `doc.check_permission`
  / `frappe.only_for`) — whitelist alone is not authorization (MISTAKES #2).
- Parameterized queries or Query Builder — never string-formatted SQL
  (MISTAKES #1).
- Field-whitelisting for any user-supplied update — no mass-assignment
  (MISTAKES #7).
- `allow_guest=True` only when the endpoint is genuinely public, and then with
  extra scrutiny.

No skill should ever emit secrets, real tokens, or credentials. Placeholders
only (`your_api_key`, `eyJ...`).

### R6. Respect the document lifecycle and transaction

- Use the correct hook for side effects (MISTAKES #4); keep `validate` pure.
- Don't `frappe.db.commit()` inside web requests (MISTAKES #8); use savepoints
  for partial rollback.
- Check `docstatus` before editing submitted docs (MISTAKES #12); don't paper
  over validation with `db_set` (MISTAKES #13).
- Offload long/external work to `frappe.enqueue` (MISTAKES #15).

### R7. Schema change ⇒ migrate

Any DocType/field/patch change must be followed by:

```bash
bench --site <site> migrate
bench build           # if JS/assets changed
bench --site <site> clear-cache
```

Write data migrations as idempotent entries in `patches.txt`, never as one-off
console edits (MISTAKES #19).

### R8. Test with real Frappe APIs

Generated tests use `FrappeTestCase` (v14+) / `unittest` on a test site with real
DocType fields and real lifecycle calls — not heavy mocks (MISTAKES #20). See
`frappe-testing`. Run via `bench --site <site> run-tests`.

### R9. Customize, don't fork core

Add fields via Custom Field / Customize Form; add behavior via your own app's
`hooks.py`; export customizations as fixtures. Never edit files under the
`frappe`/`erpnext` apps (MISTAKES #18).

### R10. Escalate instead of guessing

If a skill's procedure doesn't cover the case, or verification fails, or the
version is unsupported — stop and surface it. Link the relevant official Frappe
docs. A correct "I need to check X" beats confident wrong code.

---

## Part 2 — Authoring / contributing skills

### A1. Follow the standard SKILL.md format

Every skill's `SKILL.md` has these sections, in order:

- **When to use** — concrete trigger conditions.
- **Inputs required** — what the agent must know first (version, site, app).
- **Procedure** — numbered, executable steps.
- **Verification** — a checklist to confirm success.
- **Failure modes** — common breakages and their fixes.
- **Escalation** — when to read official docs / ask the user.

### A2. SKILL.md short and procedural; depth in `references/`

Keep `SKILL.md` lean (the agent reads it every time). Push code-heavy detail,
edge cases, and long examples into `references/*.md` and link to them. The agent
loads references on demand.

### A3. Version-check first, and label version-specific guidance

State which Frappe versions the skill targets (v13–v15). Where an API differs by
version, say so inline. Prefer APIs that exist across the supported range; flag
anything version-gated.

### A4. Never invent — cite or verify

Skill content must be Frappe-accurate. Don't document fields/APIs you haven't
verified against the framework or official docs. When unsure, link the canonical
Frappe documentation rather than asserting.

### A5. Public-safe content only

No secrets, no private hostnames, no company/personal data, no internal ledgers.
Examples use generic DocTypes (Sales Order, Customer, ToDo) and placeholder
values. This is an open-source community repo.

### A6. Bake in verification steps

Every procedure ends with how to confirm it worked — a `bench` command, a console
check, a test run. A skill without verification teaches the agent to declare
success blindly.

### A7. Document failure modes

For each skill, list what commonly goes wrong (migration not run, permission
missing, wrong version API) and the fix. The failure-modes section is where most
of the skill's real value lives.

### A8. Prefer existing skills over new ones

Before adding a skill, check whether `frappe-router` already covers the task via
an existing skill. Extend or refine first; add a new skill only for a genuinely
distinct task class, and register it in the router and README.

### A9. Preserve license and attribution

license and the upstream attribution intact in any contribution. See `LICENSE`

---

## Quick reference — the rules in one line each

1. Route via `frappe-router`.
2. Triage version + apps + tooling before coding.
3. Verify every field/DocType — never invent.
4. Reuse existing bench commands and app tooling.
5. Permission check + parameterized SQL on every API.
6. Right hook, no mid-request commit, respect docstatus.
7. Schema change → `bench migrate` (+ build, clear-cache).
8. Tests use real Frappe APIs on a test site.
9. Customize via Custom Field / hooks, never fork core.
10. Escalate on uncertainty; cite official docs.

## Attribution

These skills are curated from Impertio-Studio/frappe-skills (MIT), official Frappe documentation + community, and netchampfaris. Preserve the MIT license + per-file `source:` attribution (see LICENSE).
