# Contributing to agent-skills

Thanks for contributing! This file defines how skills are added to this library. AI agents: the [agent-skills-contributor](agent-skills-contributor/) skill encodes these same rules as an executable workflow.

## 1. Structure requirements

```
<skill-name>/
├── SKILL.md          # REQUIRED
└── references/       # optional: domain guides, API details, playbooks
```

- One skill per folder, at repository root.
- Folder name = `name` field in SKILL.md frontmatter = **kebab-case** (e.g. `esp32-portable-arduino`).
- `SKILL.md` frontmatter contains exactly two fields:
  - `name` — matches the folder name.
  - `description` — the triggering mechanism. Write what the skill does **and** when to use it, with concrete trigger keywords. Max ~1024 characters.
- `SKILL.md` body: under 500 lines, imperative form ("Write X", not "You should write X").
- Details that are not always needed go into `references/<domain>.md`, each linked from SKILL.md with a one-line "when to read" note. Reference files >100 lines start with a table of contents.
- No README/docs/changelog files inside skill folders — the skill is the documentation.
- Do not commit packaged `.skill` archives; they are built for GitHub Releases only.

## 2. Quality bar

- Teach what the model does **not** already know: current toolchain state, version-specific API changes, hard-won pitfalls, exact procedures. Skip generic programming advice.
- Prefer concise, tested code examples over prose.
- Every claim about versions/compatibility must be true **as of the commit date** — verify against primary sources before contributing.
- The skill must be generic and reusable: parameterized knowledge, not a transcript of one private project.

## 3. Privacy & security rules (mandatory)

Skills have a **double exposure surface**: they are public on GitHub *and* their full content is loaded into an AI model's context (sent to the model provider) whenever the skill triggers. Treat every line as published twice.

1. **No secrets, ever.** No API keys, tokens, passwords, certificates, private keys, connection strings, or credentials of any kind. Reference them as placeholders only: `${API_KEY}`, `<YOUR_TOKEN>`, `os.environ[...]`.
2. **No company-confidential information.** No internal hostnames/URLs, employer system details, unreleased product information, customer data, or anything under NDA. Contributing knowledge gained at work requires your employer's explicit permission; generic industry knowledge does not.
3. **No personal data.** No real names, addresses, emails, IDs, or any data about identifiable people (yours included) beyond your GitHub identity. GDPR applies — this library is served worldwide.
4. **Original content only.** Paraphrase vendor documentation; never paste large copyrighted excerpts. Credit sources with links.
5. **Pre-publish checklist** — before opening a PR, read every added file as a stranger would and answer:
   - [ ] Contains no secret or credential, even an expired one?
   - [ ] Reveals nothing about any employer, client, or private project?
   - [ ] Contains no personal data?
   - [ ] All content original or properly attributed?

Enforcement: a gitleaks workflow scans every push and PR, and GitHub push protection is enabled. A passing scan does **not** transfer responsibility away from you — scanners catch key-shaped strings, not confidential prose.

If you committed a secret by mistake: rotate/revoke it immediately, then open an issue — force-purging git history is done by the maintainer.

## 4. Pull request process

1. Fork and branch: `add/<skill-name>` or `fix/<skill-name>-<topic>`.
2. One skill per PR (or one focused fix).
3. Update the skill index table in `README.md` when adding a skill.
4. PR description: what the skill teaches, sources used for version-sensitive claims, confirmation of the section 3 checklist.
5. Maintainer review focuses on: description quality (triggering), structure, factual accuracy, privacy.

## 5. License

By contributing you agree your contribution is licensed under the repository's [MIT License](LICENSE).
