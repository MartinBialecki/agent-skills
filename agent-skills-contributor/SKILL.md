---
name: agent-skills-contributor
description: "Add a new Agent Skill (or update an existing one) in the MartinBialecki/agent-skills GitHub repository — an open library of portable SKILL.md-format skills for Kilo Code, Claude Code, Cursor, Kimi and other agents. Use whenever the user wants to: contribute a skill to the agent-skills repo/library, publish a skill they created, open a skill PR, check whether a skill folder meets the repo's structure/quality/privacy rules, update the repo README skill index, or prepare a skill release. Trigger keywords: agent-skills repo, contribute skill, publish skill, skill library, add skill to GitHub, skill PR, SKILL.md contribution."
---

# agent-skills-contributor

Contribute skills to the public repo **MartinBialecki/agent-skills** correctly on the first try. The authoritative, possibly newer rules live in the repo — **always fetch `CONTRIBUTING.md` and `README.md` from the repo's main branch before starting** and treat them as the source of truth over this file.

## Hard rules (violating any of these = rejected PR)

**Structure:**
- One folder per skill at repo root; folder name = `name` in frontmatter = kebab-case.
- `SKILL.md` required; frontmatter has exactly two fields: `name` and `description` (max ~1024 chars, must say what the skill does AND when to use it, with trigger keywords).
- Body < 500 lines, imperative form. Conditional detail goes to `references/<domain>.md`, each linked from SKILL.md with a when-to-read note; files >100 lines start with a table of contents.
- No extra docs inside the skill folder (no README/CHANGELOG/etc.). No packaged `.skill` files in the repo (releases only).

**Quality:**
- Teach what models don't already know: current toolchain state, version-specific changes, real pitfalls, exact procedures. No generic programming advice.
- Version-sensitive claims verified against primary sources as of commit date.
- Generic and reusable — never a transcript of one private project.

**Privacy & security (mandatory):**
1. No secrets or credentials of any kind — placeholders only (`${API_KEY}`, `<YOUR_TOKEN>`).
2. No company-confidential info: internal hostnames/URLs, employer systems, unreleased products, customer data, NDA material.
3. No personal data about any identifiable person (GDPR applies).
4. Original content only; paraphrase vendor docs, credit with links.
5. Before submitting, re-read every added file as a stranger hunting for exploitable info and confirm the checklist in CONTRIBUTING.md §3.

## Workflow: adding a new skill

1. **Fetch latest rules**: GET `CONTRIBUTING.md`, `README.md` and (if it exists) the repo's `.github/workflows/` from `main`. Note any differences from this file; the repo wins.
2. **Design the skill** with the user: what does it teach, what triggers it, what are the reference domains. Check the README index for name clashes; on clash propose a distinct kebab-case name.
3. **Author locally**: create `<skill-name>/SKILL.md` (+ `references/` if needed). Write the `description` last, once content is stable.
4. **Validate**:
   - YAML frontmatter parses; only `name` + `description` fields.
   - Folder name == `name` field.
   - All links from SKILL.md resolve to files that exist.
   - Privacy sweep: grep every file for key-shaped strings AND read for confidential prose (scanners miss prose). Patterns to grep: `api[_-]?key`, `token`, `secret`, `password`, `BEGIN .* PRIVATE KEY`, `-----BEGIN`, `http[s]://[a-z0-9.-]*internal`, email addresses, phone numbers.
5. **Update README.md skill index** — add one table row, keep alphabetical order.
6. **Submit**:
   - If the user owns the repo or has write access: branch `add/<skill-name>` from `main`, commit files (`Add <skill-name> skill`), push, open PR to `main`.
   - External contribution: fork first, same branch naming, PR from fork.
   - PR body: what the skill teaches, sources for version-sensitive claims, explicit confirmation of the privacy checklist.
7. **Report** the PR URL to the user and summarize what review will check (description quality, structure, accuracy, privacy).

## Workflow: updating an existing skill

- Branch `fix/<skill-name>-<topic>`; change only what the fix needs; keep the folder's existing style.
- Bump nothing (no version fields exist); let git history speak.
- If facts changed (e.g. a toolchain moved), state the as-of date in the reference file when relevant.

## Common rejection causes (check before submitting)

- `description` is a vague one-liner ("helps with ESP32") — agents won't trigger on it; rewrite with concrete use cases and keywords.
- SKILL.md is a documentation dump, not instructions — convert to imperative procedures.
- Duplicated content between SKILL.md and references — SKILL.md keeps only the core workflow; move detail out.
- Missing README index row.
- Anything that looks like it came from a private project: real device names, client contexts, internal paths, "at my company we...".
