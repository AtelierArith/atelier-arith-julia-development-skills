# atelier-arith-julia-development-skills

Plugin containing Julia development skills for both **Claude Code** and **Codex**.

## Included Skills

- `installing-julia` — install the Julia runtime
- `generating-julia-package` — create Julia project environments and package layouts
- `creating-julia-app` — create Julia command-line apps with `@main` and `[apps]`
- `creating-julia-test-env` — add a Julia package test environment
- `running-julia-test` — run standard and faster local Julia test workflows

## Installation

### Claude Code

```sh
/plugin marketplace add atelierarith/atelier-arith-julia-development-skills
/plugin install julia-dev-skills@atelier-arith-julia-development-skills
```

Session-only (no permanent install):

```sh
claude --plugin-dir /path/to/atelier-arith-julia-development-skills
# or
claude --plugin-url https://github.com/atelierarith/atelier-arith-julia-development-skills/archive/main.zip
```

### Codex

Add this repository URL in the Codex plugin settings. The manifest is at `.codex-plugin/plugin.json`.

## Repository Structure

```
.
├── .claude-plugin/
│   ├── plugin.json       # Claude Code plugin manifest
│   └── marketplace.json  # Claude Code marketplace manifest (references this repo)
├── .codex-plugin/
│   └── plugin.json       # Codex plugin manifest
├── AGENTS.md
├── CLAUDE.md
├── LICENSE
├── README.md
├── plans/
└── skills/
    ├── creating-julia-app/
    │   └── SKILL.md
    ├── creating-julia-test-env/
    │   └── SKILL.md
    ├── generating-julia-package/
    │   └── SKILL.md
    ├── installing-julia/
    │   └── SKILL.md
    └── running-julia-test/
        └── SKILL.md
```

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter:

   ```markdown
   ---
   name: <skill-name>
   description: Use when ...
   ---
   ```

2. Bump `version` in `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, and `.codex-plugin/plugin.json`.
