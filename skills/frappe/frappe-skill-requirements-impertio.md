---
title: Frappe Skill — Requirements (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'requirements', 'spec']
source: Impertio
---

# REQUIREMENTS - ERPNext Skills Package

> Quality criteria and acceptance standards for all skills in this package.

---

## Skill Content Requirements

### YAML Frontmatter

| Requirement | Standard |
|-------------|----------|
| `name` field | kebab-case, max 64 characters, pattern: `erpnext-{type}-{topic}` |
| `description` field | Folded block scalar (`>`), max 1024 characters |
| Description content | Must contain: "Use when [trigger]", anti-pattern warning, scope, Keywords line |
| `license` field | LGPL-3.0 |
| `compatibility` field | Must mention ERPNext/Frappe version range (v14/v15/v16) |

### SKILL.md Structure

| Requirement | Standard |
|-------------|----------|
| Max length | < 500 lines |
| Language | English only |
| Tone | Deterministic — ALWAYS/NEVER, not "you might/consider/should" |
| Quick Reference | Must include critical warning and decision tree |
| API Overview | Table with function, purpose, version |
| Anti-Patterns | WRONG vs CORRECT with explanation |
| Reference Links | Links to references/ subfolder files |

### Reference Files

| Requirement | Standard |
|-------------|----------|
| Location | `references/` subfolder within skill directory |
| Required files | methods.md, examples.md, anti-patterns.md (minimum) |
| Code examples | Must include version annotation (v14/v15/v16) |
| Content | Verified against SOURCES.md approved sources |

---

## Version Coverage Requirements

| Version | Requirement |
|---------|-------------|
| Frappe/ERPNext v14 | Full coverage — production baseline |
| Frappe/ERPNext v15 | Full coverage — migration patterns documented |
| Frappe/ERPNext v16 | Full coverage — breaking changes annotated |

### V16 Breaking Changes (must be documented where applicable)

- `extend_doctype_class` hook (new)
- Data masking field-level option
- UUID naming rule for globally unique IDs
- Chrome-based PDF rendering option
- Scheduler tick interval change (4 min → 60 sec)

---

## Quality Gates

### Per Skill

- [ ] YAML frontmatter valid (description uses `>` folded scalar)
- [ ] SKILL.md < 500 lines
- [ ] English only — no Dutch content in skills
- [ ] Deterministic language — no vague suggestions
- [ ] All code examples include version annotation
- [ ] Anti-patterns section present with WRONG/CORRECT pairs
- [ ] Reference files exist and are linked
- [ ] Server Script examples use `frappe` namespace only (no imports)

### Per Batch (3 skills)

- [ ] All skills in batch pass individual quality gates
- [ ] No duplicate content across skills in batch
- [ ] Cross-references between related skills correct

### Per Package

- [ ] Skill count matches masterplan (28 skills)
- [ ] All categories represented (syntax, core, impl, errors, agents)
- [ ] INDEX.md catalogues all skills
- [ ] CHANGELOG.md has versioned release entry
- [ ] All governance files present and consistent

---

## Validation Tools

| Tool | Purpose | Pass Criteria |
|------|---------|---------------|
| `quick_validate.py` | Structure validation | All checks pass |
| `package_skill.py` | Package generation | .skill file generated |
| Manual review | Content accuracy | Verified against sources |

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/REQUIREMENTS.md*
