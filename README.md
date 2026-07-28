# Skills and Agents

OpenCode agent definitions and skill files — part of the [zsh-config](https://github.com/Wikijito7/zsh-config) OpenCode setup.

## Agents

Self-contained agent definitions with YAML frontmatter. No registration needed in `opencode.json` — OpenCode scans the `agents/` directory automatically.

### Coordinator (primary)
- **File:** `agents/coordinator.md`
- **Model:** opencode-go/deepseek-v4-pro
- **Role:** Task analysis, planning, and sub-agent coordination. Delegates to Developer, Testing, QA, and Reviewer agents.

### Developer (subagent)
- **File:** `agents/subagents/developer.md`
- **Model:** opencode-go/deepseek-v4-flash
- **Role:** Implements code and fixes bugs. Has write/edit/bash access.

### Testing (subagent)
- **File:** `agents/subagents/testing.md`
- **Model:** opencode-go/deepseek-v4-flash
- **Role:** Writes unit tests. Has write/edit/bash access.

### QA (subagent)
- **File:** `agents/subagents/qa.md`
- **Model:** opencode-go/deepseek-v4-flash
- **Role:** Verifies build, lint, and tests pass. Read-only + bash access.

### Reviewer (subagent)
- **File:** `agents/subagents/reviewer.md`
- **Model:** opencode-go/qwen3.7-plus
- **Role:** Reviews code quality and provides feedback. Read-only + bash access.

## Skills

Workflow-level skills that agents load for specific tasks:

| Skill | File | Purpose |
|---|---|---|
| `opencode-plugin` | `skills/opencode-plugin/SKILL.md` | Building server and TUI plugins |
| `code-review` | `skills/code-review/SKILL.md` | Code quality review (BLOCKER/ROAST/PRAISE) |
| `scan-project` | `skills/scan-project/SKILL.md` | Scan new projects, create memory bank |
| `bootstrapper` | `skills/bootstrapper/SKILL.md` | Onboard repos lacking documentation |
| `create-issue` | `skills/create-issue/SKILL.md` | GitHub issue creation with templates |
| `create-skill` | `skills/create-skill/SKILL.md` | Creating new skill files |
| `git-flow` | `skills/git-flow/SKILL.md` | Git commit/PR/branch conventions |
| `opencode-dialogs` | `skills/opencode-dialogs/SKILL.md` | TUI dialog UX patterns |
| `stress-test` | `skills/stress-test/SKILL.md` | Plan/design stress testing |

All skills follow the same format: YAML frontmatter with `name` and `description`, followed by actionable content (commands, checklists, tables).

## Structure

```
skills-and-agents/
├── agents/
│   ├── coordinator.md
│   └── subagents/
│       ├── developer.md
│       ├── qa.md
│       ├── reviewer.md
│       └── testing.md
└── skills/
    ├── bootstrapper/SKILL.md
    ├── code-review/SKILL.md
    ├── create-issue/SKILL.md
    ├── create-skill/SKILL.md
    ├── git-flow/SKILL.md
    ├── opencode-dialogs/SKILL.md
    ├── opencode-plugin/SKILL.md
    ├── scan-project/SKILL.md
    └── stress-test/SKILL.md
```
