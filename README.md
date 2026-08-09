# Agent Skills

An open library of portable **AI Agent Skills** in the open [Agent Skills](https://agentskills.io) `SKILL.md` format. One package works across **Kilo Code, Claude Code, Cursor, Kimi** and any other agent that implements the standard.

A *skill* is a folder with a `SKILL.md` (YAML frontmatter + instructions) plus optional `references/`, `scripts/` and `assets/`. Agents load the skill's name and description at all times, the body when it triggers, and bundled resources only when needed — specialized, on-demand expertise for the AI.

## Skill index

| Skill | Description |
|---|---|
| [esp32-portable-arduino](esp32-portable-arduino/) | ESP32 firmware in C++/Arduino framework (VS Code + pioarduino) architected in three layers (core/hal/app) for painless later migration to ESP-IDF. Covers BLE (NimBLE), motors/robot arms, power/ULP, on-device ML (TFLite Micro/ESP-DL), testing and the full migration playbook. |
| [agent-skills-contributor](agent-skills-contributor/) | Teaches an AI agent how to add a new skill to this repository: structure, frontmatter, quality bar, privacy/security checklist, PR conventions. |

## Installation

Each skill is a self-contained folder. Copy the folder of the skill you want into your agent's skills directory:

**Kilo Code**
```bash
# global (all projects)
cp -r <skill-name> ~/.kilo/skills/
# project-level (shared via git)
cp -r <skill-name> <your-repo>/.kilo/skills/
```
Older Kilo Code versions use `~/.kilocode/skills/` instead of `~/.kilo/skills/`.

**Claude Code**
```bash
cp -r <skill-name> ~/.claude/skills/        # personal
# or <your-repo>/.claude/skills/ for project skills
```

**Cursor**
```bash
cp -r <skill-name> <your-repo>/.cursor/skills/
```

**Kimi**

Download the packaged `<skill-name>.skill` from [Releases](../../releases) and import it, or copy the folder into your Kimi skills directory.

After copying, restart the agent session (Kilo Code: `/reload`) and ask the agent *"what skills do you have available?"* to verify.

## Repository structure

```
<skill-name>/
├── SKILL.md          # required: YAML frontmatter (name, description) + instructions
└── references/       # optional: detailed docs loaded by the agent on demand
```

One folder per skill at repository root. The folder name must match the `name` field in SKILL.md.

## Contributing

Contributions are welcome — read [CONTRIBUTING.md](CONTRIBUTING.md) first. It defines the structure requirements, the quality bar, and the **privacy & security rules** (no secrets, no company-confidential or personal data in skills — ever). The [agent-skills-contributor](agent-skills-contributor/) skill lets your AI agent do the whole workflow for you.

## License

[MIT](LICENSE) — use, fork and adapt freely.
