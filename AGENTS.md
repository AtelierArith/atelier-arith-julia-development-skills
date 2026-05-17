# Repository Guidelines

## Project Structure & Module Organization

This repository contains Codex skill documentation for Julia development workflows. Each skill lives in its own directory under `skills/` and uses a single `SKILL.md` file:

- `skills/installing-julia/SKILL.md`
- `skills/generating-julia-package/SKILL.md`
- `skills/creating-julia-app/SKILL.md`
- `skills/creating-julia-test-env/SKILL.md`
- `skills/running-julia-test/SKILL.md`

Keep skill directories lowercase and hyphen-separated. Add supporting files only when a skill genuinely needs reusable assets, scripts, or references.

## Build, Test, and Development Commands

There is no application build step in this repository. Use lightweight checks before submitting changes:

```sh
rg --files
```

Lists tracked project files and helps confirm new skill paths.

```sh
sed -n '1,120p' skills/<skill-name>/SKILL.md
```

Reviews front matter and opening instructions for a skill.

```sh
git diff --check
```

Detects trailing whitespace and common patch formatting issues.

If you add Markdown tooling later, document the exact command here and prefer commands that run without global machine-specific setup.

## Coding Style & Naming Conventions

Write skill content in Markdown. Each `SKILL.md` must start with YAML front matter containing `name` and `description`, followed by an H1 title. Use fenced code blocks with language tags such as `sh`, `julia`, `toml`, or `powershell`.

Prefer direct, imperative instructions. Keep examples copy-pasteable and mark placeholder values clearly, for example `<UUID for MyPkg goes here>`. Use two-space indentation for nested Markdown list content when needed.

## Testing Guidelines

Validate documentation changes by reading the rendered Markdown and checking all commands for accuracy. For Julia examples, prefer commands that can be run from a clean package environment, such as:

```sh
julia --project -e 'using Pkg; Pkg.test()'
```

When changing a skill about test execution, include both the standard `Pkg.test()` path and any faster local workflow caveats.

## Version Update Workflow

When bumping the plugin version, update every plugin metadata file in the same change:

- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `.codex-plugin/plugin.json`

Keep all version fields synchronized to the same semantic version, for example `0.1.1`. Verify the update with:

```sh
rg -n '"version"|"metadata"' .claude-plugin .codex-plugin
git diff --check
```

If the version bump accompanies skill documentation changes, include both the metadata updates and the related skill/report changes in the same commit.

## Commit & Pull Request Guidelines

The current history uses short, plain commit messages such as `Init` and `first commit`. Continue with concise imperative summaries, for example `Add Julia app skill notes`.

Pull requests should describe the skill changed, the user scenario it supports, and any commands or links verified. Include screenshots only if adding rendered documentation or visual assets.
