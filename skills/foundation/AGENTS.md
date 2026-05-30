# Foundation Skill Guidelines

## Purpose

The `foundation` skill bootstraps preferred development tooling from language-specific JSON manifests.

## Owned Files

The `foundation` skill owns these files:

```text
skills/foundation/SKILL.md
skills/foundation/node.json
skills/foundation/python.json
skills/foundation/rust.json
skills/foundation/schema/language.schema.json
skills/foundation/assets/
```

## Manifest Policy

The language manifests are the source of truth for install behavior.

Do not hardcode package lists only in `SKILL.md`. Put dependencies, config files, scripts, hooks, and workspace placement in the relevant `{language}.json` manifest.

Each package operation in `packages.add` or `packages.remove` must define:

```json
{
  "name": "package-name",
  "type": "devDependencies",
  "placement": {
    "singlePackage": "target-package",
    "monorepo": "workspace-root",
    "allowWorkspaceRoot": true
  }
}
```

Do not infer placement from dependency type. Each package must carry its own placement policy.

Supported monorepo placement values are:

- `workspace-root`
- `ask-workspace-package`
- `target-package`
- `all-workspace-packages`
- `skip`

Unsupported ecosystems must have a manifest with `"supported": false`. The skill must report unsupported languages instead of installing tools opportunistically.

## Current Ecosystems

Node.js is supported by `node.json`.

Node.js currently installs:

- `oxlint` as `devDependencies`
- `oxfmt` as `devDependencies`
- `@commitlint/cli` as `devDependencies`
- `@commitlint/config-conventional` as `devDependencies`
- `husky` as `devDependencies`
- `ts-pattern` as `dependencies`

Node.js config assets currently include:

- `.oxfmtrc.json`
- `.oxlintrc.json`
- `commitlint.config.js`

Python is supported by `python.json`.

Python currently installs:

- `ruff` as `devDependencies`

Python config assets currently include:

- `ruff.toml`

Rust is currently unsupported by `rust.json`.

## Asset And Root Config Policy

The installable asset files live under `skills/foundation/assets/`.

Root config files are local convenience symlinks pointing to the assets:

```text
.oxfmtrc.json -> skills/foundation/assets/.oxfmtrc.json
.oxlintrc.json -> skills/foundation/assets/.oxlintrc.json
ruff.toml -> skills/foundation/assets/ruff.toml
commitlint.config.js -> skills/foundation/assets/commitlint.config.js
```

When editing a config, edit the asset file directly to preserve the root symlink.

Do not make `skills/foundation/assets/*` symlink to files outside the skill directory. Skill installation may copy only the skill directory, and outward symlinks can break after installation.

## Existing Config Choices

Oxfmt uses `.oxfmtrc.json`.

Oxlint uses `.oxlintrc.json`.

Ruff uses `ruff.toml` instead of `[tool.ruff]` in `pyproject.toml`.

Commitlint uses `commitlint.config.js` with `@commitlint/config-conventional`.

Husky uses a `commit-msg` hook with:

```bash
npx --no -- commitlint --edit "$1"
```

## Conflict Behavior

Do not overwrite an existing target project config silently.

If the target project already has a config file or hook declared by a manifest, ask whether to replace, merge, or keep it.

For monorepos, never add runtime dependencies to the workspace root when a package's placement has `allowWorkspaceRoot: false`. Ask the user which workspace package should receive the dependency.

## Verification

After changing foundation skill structure or metadata, verify discovery from the repository root:

```bash
pnpx skills add . --list
```

After changing JSON manifests or schemas, validate JSON syntax:

```bash
jq empty skills/foundation/node.json
jq empty skills/foundation/python.json
jq empty skills/foundation/rust.json
jq empty skills/foundation/schema/language.schema.json
```

After changing Ruff config, validate TOML syntax:

```bash
python3 -c 'import pathlib, tomllib; tomllib.loads(pathlib.Path("skills/foundation/assets/ruff.toml").read_text())'
```

After changing commitlint config, validate JavaScript syntax:

```bash
node --check skills/foundation/assets/commitlint.config.js
```
