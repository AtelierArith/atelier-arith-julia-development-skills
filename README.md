# atelier-arith-julia-development-skills

Plugin containing Julia development skills for **Claude Code**, **Codex**, and **OpenCode**.

## Included Skills

- `finding-latest-julia-version` — resolve the latest stable Julia version number from official metadata
- `installing-julia` — point the user to the official Julia installation instructions (does not run install scripts)
- `generating-julia-package` — create Julia project environments and package layouts
- `developing-julia-package` — write Julia packages idiomatically
- `precompiling-julia-package` — reduce compile latency (TTFX ≥ 3 s, TTFL ≥ 1 s) with PrecompileTools.jl
- `profiling-julia-performance` — diagnose runtime slowness, type instability, and allocations with Profile, JET.jl, and AllocCheck.jl
- `debugging-julia` — find the cause of exceptions and wrong results non-interactively (stack traces, @code_typed, JET)
- `creating-julia-app` — create Julia command-line apps with `@main` and `[apps]`
- `creating-julia-test-env` — add a Julia package test environment
- `running-julia-test` — run standard, targeted, and parallel (ParallelTestRunner.jl) Julia test workflows

## Installation

Each skill is a self-contained `<skill-name>/SKILL.md` folder, and that format
is read natively by Claude Code, Codex, and OpenCode. Install it with whichever
client you use; you do not need the others.

### Claude Code

Add this repository as a plugin marketplace, then install the `aa-jl` plugin
from it:

```sh
/plugin marketplace add AtelierArith/atelier-arith-julia-development-skills
/plugin install aa-jl@atelier-arith-julia-development-skills
```

Session-only, without a permanent install:

```sh
claude --plugin-dir /path/to/atelier-arith-julia-development-skills
# or
claude --plugin-url https://github.com/AtelierArith/atelier-arith-julia-development-skills/archive/main.zip
```

### Codex

Codex loads local skills directly from `.agents/skills` directories — no plugin
or marketplace registration needed. Install globally so the skills are
available in every repository:

```sh
git clone https://github.com/AtelierArith/atelier-arith-julia-development-skills.git ~/.local/share/aa-jl
mkdir -p ~/.agents/skills
cp -R ~/.local/share/aa-jl/skills/. ~/.agents/skills/
```

To install only for one project, copy into that project's `.agents/skills`:

```sh
mkdir -p .agents/skills
cp -R /path/to/atelier-arith-julia-development-skills/skills/. .agents/skills/
```

Codex follows symlinked skill folders, so you can track upstream instead of
copying:

```sh
for d in ~/.local/share/aa-jl/skills/*/; do
  ln -sfn "$d" ~/.agents/skills/"$(basename "$d")"
done
```

Notes:

- Codex scans `.agents/skills` from the current directory up to the repository
  root, then `$HOME/.agents/skills`. Installed skills appear under `$`, `/skills`,
  or the Skills sidebar.
- Codex detects skill changes automatically; restart Codex if an update does not
  appear.
- `~/.agents/skills` is also one of OpenCode's skill locations, so this single
  install works for both Codex and OpenCode. OpenCode additionally reads the
  Claude-compatible locations (see below).

For distribution as an installable plugin, the repository also ships a Codex
plugin manifest at `.codex-plugin/plugin.json`; see the
[Codex plugin docs](https://developers.openai.com/plugins) to package and
publish it. The direct `.agents/skills` install above is simpler for personal
use.

### OpenCode

OpenCode has no plugin marketplace for skills — it discovers them directly from
a few well-known directories, so you install by placing the skill folders where
OpenCode looks.

The simplest option is the Codex install above: OpenCode reads the same
`~/.agents/skills` directory, so one install covers both. If you prefer an
OpenCode-only global location, use `~/.config/opencode/skills`:

```sh
git clone https://github.com/AtelierArith/atelier-arith-julia-development-skills.git ~/.local/share/aa-jl
mkdir -p ~/.config/opencode/skills
cp -R ~/.local/share/aa-jl/skills/. ~/.config/opencode/skills/
```

Project-scoped install (vendored into the repository you are working on). Both
`.opencode/skills` and `.agents/skills` are read here; use `.agents/skills` if
you also want Codex to see them:

```sh
mkdir -p .opencode/skills
cp -R /path/to/atelier-arith-julia-development-skills/skills/. .opencode/skills/
```

To track upstream instead of copying, symlink each skill folder (OpenCode
follows symlinks and a clone update is then picked up on restart):

```sh
for d in ~/.local/share/aa-jl/skills/*/; do
  ln -sfn "$d" ~/.config/opencode/skills/"$(basename "$d")"
done
```

Notes:

- OpenCode also reads `~/.claude/skills/` and `.claude/skills/` for
  Claude-code compatibility. If you already installed these skills for Claude
  Code, OpenCode picks them up automatically — do not copy them into
  `~/.config/opencode/skills/` as well, since duplicate skill names across
  locations can make discovery ambiguous.
- A skill's directory name must match its `name` frontmatter field and be
  unique across all discoverable locations.
- Restart OpenCode after installing so it reloads the skill list.

## Updating

New versions are published by bumping the plugin `version` in the manifests. How
you receive them depends on how you installed.

### Claude Code

Auto-update is **off by default** for third-party marketplaces like this one.
Turn it on in `/plugin` → **Marketplaces** → select this marketplace →
**Enable auto-update**, or update on demand:

```sh
claude plugin update aa-jl@atelier-arith-julia-development-skills
```

Refresh the marketplace's catalog separately with:

```sh
claude plugin marketplace update atelier-arith-julia-development-skills
```

Run `/reload-plugins`, or start a new session, to load the update.

### Codex and OpenCode

These installs are plain files, so pull the repository and re-place them.

For a copied install, re-copy after pulling:

```sh
git -C ~/.local/share/aa-jl pull
cp -R ~/.local/share/aa-jl/skills/. ~/.agents/skills/   # shared Codex + OpenCode location
# for an OpenCode-only install instead:
cp -R ~/.local/share/aa-jl/skills/. ~/.config/opencode/skills/
```

For a symlinked install, the links follow the clone, so a pull is enough:

```sh
git -C ~/.local/share/aa-jl pull
```

Restart Codex / OpenCode so they reload the skill list. If you vendored the
skills into a project (`.agents/skills` or `.opencode/skills`), pull and re-copy
there, then commit the change.

## Repository Structure

```
.
├── .claude-plugin/
│   ├── plugin.json       # Claude Code plugin manifest
│   └── marketplace.json  # Claude Code marketplace manifest (references this repo)
├── .codex-plugin/
│   └── plugin.json       # Codex plugin manifest
├── AGENTS.md
├── LICENSE
├── README.md
└── skills/
    ├── creating-julia-app/
    │   └── SKILL.md
    ├── creating-julia-test-env/
    │   └── SKILL.md
    ├── debugging-julia/
    │   └── SKILL.md
    ├── developing-julia-package/
    │   └── SKILL.md
    ├── finding-latest-julia-version/
    │   └── SKILL.md
    ├── generating-julia-package/
    │   └── SKILL.md
    ├── installing-julia/
    │   └── SKILL.md
    ├── precompiling-julia-package/
    │   └── SKILL.md
    ├── profiling-julia-performance/
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
