---
name: foundation
description: Bootstrap preferred development tooling from language-specific JSON manifests. Currently supports Node.js with Oxlint, Oxfmt, commitlint, Husky, and ts-pattern, and Python with Ruff. Explicitly reports Rust or unknown languages as unsupported until their manifests are enabled. Use when asked to add foundation tooling, preferred tooling, dependencies, devDependencies, ruff, commitlint, oxlint, oxfmt, or ts-pattern.
license: MIT
compatibility: Requires package manager access for the detected ecosystem. Node.js requires npm, pnpm, yarn, or bun. Python requires uv, Poetry, PDM, or pip.
---

# Foundation

Use this skill to bootstrap a project with the user's preferred development tools. The source of truth is a language-specific JSON manifest in this skill directory.

## Manifest Files

Read the manifest for the target ecosystem before changing files:

- `node.json`: Node.js tooling and dependency policy.
- `python.json`: Python tooling and dependency policy.
- `rust.json`: Rust is currently unsupported.

If the requested language has no manifest or has `"supported": false`, stop and tell the user that the language is not supported by this skill yet. Do not install tools for unsupported languages. Rust is currently unsupported.

## Scope Rules

1. Apply only the ecosystem manifest that matches the target project or the user's explicit request.
2. Do not install tooling for an ecosystem that is not detected or requested.
3. If multiple ecosystems are present, ask which ecosystem should be bootstrapped before editing files.
4. The manifest controls dependencies, devDependencies, removals, config files, scripts, and workspace placement policy.
5. If the manifest and these written instructions disagree, follow the manifest and report the mismatch.

## Manifest Schema

Each `{language}.json` manifest may define:

```json
{
  "language": "node",
  "supported": true,
  "detect": {
    "files": ["package.json"]
  },
  "workspacePolicy": {
    "detectFiles": ["pnpm-workspace.yaml", "turbo.json"],
    "packageJsonWorkspaceField": "workspaces"
  },
  "packages": {
    "add": [
      {
        "name": "oxlint",
        "type": "devDependencies",
        "placement": {
          "singlePackage": "target-package",
          "monorepo": "workspace-root",
          "allowWorkspaceRoot": true
        }
      },
      {
        "name": "ts-pattern",
        "type": "dependencies",
        "placement": {
          "singlePackage": "target-package",
          "monorepo": "ask-workspace-package",
          "allowWorkspaceRoot": false
        }
      }
    ],
    "remove": []
  }
}
```

Supported `packages.add[].placement.monorepo` and `packages.remove[].placement.monorepo` values:

- `workspace-root`: apply the package at the monorepo root.
- `ask-workspace-package`: ask the user which workspace package should receive the package.
- `target-package`: apply the package to the package that is already the target.
- `all-workspace-packages`: apply the package to every workspace package.
- `skip`: do not apply the package in monorepos.

Every package in `packages.add` and `packages.remove` must define its own `placement`. Do not infer placement from its dependency type.

## Node.js Workflow

Use `node.json` for Node.js projects.

1. Confirm the target is a Node.js project by finding one of `node.json.detect.files`.
2. Detect the package manager from the lockfile or existing commands:
   - `pnpm-lock.yaml` -> `pnpm`
   - `yarn.lock` -> `yarn`
   - `bun.lock` or `bun.lockb` -> `bun`
   - `package-lock.json` -> `npm`
   - If no lockfile exists, follow the package manager already used in scripts or ask the user.
3. Detect whether the project is a monorepo using `node.json.workspacePolicy.detectFiles` and workspace configuration in `package.json`.
4. For each package in `node.json.packages.add`, install it according to that package's `type` and `placement`.
5. For each package in `node.json.packages.remove`, remove it according to that package's `type` and `placement`.
6. If a package uses `ask-workspace-package`, ask the user which workspace package should receive that specific package before changing package files.
7. Copy `node.json.configFiles` from the skill directory to the target project root. Do not overwrite an existing config silently; ask whether to replace, merge, or keep it.
8. Add `node.json.scripts` to the relevant root `package.json` when equivalent scripts do not already exist. Do not replace existing scripts without asking.
9. If `node.json.gitHooks` is present, configure those hooks after dependencies and scripts are in place. Do not overwrite an existing hook without asking.

## Node.js Command Guidance

Use the detected package manager and workspace-aware commands.

For a single-package pnpm app with the default `node.json`:

```bash
pnpm add -D oxlint oxfmt @commitlint/cli @commitlint/config-conventional husky
pnpm add ts-pattern
```

For a pnpm monorepo root with the default `node.json`:

```bash
pnpm add -Dw oxlint oxfmt @commitlint/cli @commitlint/config-conventional husky
```

After the user selects the workspace package that should receive default runtime dependencies:

```bash
pnpm --filter <package-name> add ts-pattern
```

Adapt commands for `npm`, `yarn`, or `bun` when those package managers are detected.

## Node.js Commitlint Guidance

The Node.js manifest follows commitlint local setup:

- Install `@commitlint/cli` and `@commitlint/config-conventional` as dev dependencies.
- Install `husky` as a dev dependency for the `commit-msg` hook.
- Copy `assets/commitlint.config.js` to `commitlint.config.js`.
- Add the `commitlint` package script from `node.json.scripts`.
- Create or update `.husky/commit-msg` from `node.json.gitHooks` with `npx --no -- commitlint --edit "$1"`.

If the project has no `.git` directory, skip Husky hook creation and report that the hook can be added after Git is initialized. If `.husky/commit-msg` already exists, ask before replacing or merging it.

## Python Workflow

Use `python.json` for Python projects.

1. Confirm the target is a Python project by finding one of `python.json.detect.files`.
2. Detect the package manager from lockfiles or project files:
   - `uv.lock` -> `uv`
   - `poetry.lock` -> `poetry`
   - `pdm.lock` -> `pdm`
   - `requirements.txt` -> `pip`
   - `pyproject.toml` without a known lockfile -> ask which package manager to use.
3. Detect whether the project is a monorepo using `python.json.workspacePolicy.detectFiles`.
4. For each package in `python.json.packages.add`, install it according to that package's `type` and `placement`.
5. Copy `python.json.configFiles` from the skill directory to the target project root.
6. If the target already has `ruff.toml`, `.ruff.toml`, or `[tool.ruff]` in `pyproject.toml`, do not add a conflicting Ruff config silently. Ask whether to replace, merge, or keep the existing configuration.
7. If the project Python version is clear, adjust `target-version` in the copied Ruff config to match it. Otherwise keep the bundled default.

## Python Command Guidance

Use the detected package manager.

For a uv Python project with the default `python.json`:

```bash
uv add --dev ruff
```

For Poetry:

```bash
poetry add --group dev ruff
```

For PDM:

```bash
pdm add -dG dev ruff
```

For pip-based projects, ask where the user wants to record development dependencies before editing files.

## Python Ruff Config

The bundled Ruff config is `assets/ruff.toml` and includes:

- `line-length = 120`
- `target-version = "py312"` as the default Python target
- `lint.select = ["E", "F", "I", "UP", "B", "SIM", "RUF"]`
- `lint.ignore = []`
- `format.quote-style = "double"`
- `format.indent-style = "space"`
- `format.line-ending = "auto"`
- `lint.isort.known-first-party = []`
- Generic excludes for virtualenvs, build output, migrations, and generated files

## Verification

After making changes, run the checks declared in the manifest when available. If a command fails because the project has pre-existing issues, report the failure and the first actionable error without broad refactors.

## Adding Future Language Support

To support Rust or another ecosystem later, update or add the corresponding `{language}.json` manifest and change `supported` to `true`. Keep ecosystem behavior data-driven in the manifest instead of hardcoding package lists in this file.
