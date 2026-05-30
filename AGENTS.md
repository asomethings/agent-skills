# Repository Guidelines

## Purpose

This repository is an Agent Skills collection intended to be installable with `pnpx skills add` and discoverable on skills-compatible clients.

## Skill Layout

Keep skills under `skills/<skill-name>/SKILL.md`.

The `name` frontmatter must match the directory name exactly.

For example:

```text
skills/foundation/SKILL.md
```

Skill-specific implementation rules belong in that skill's own `AGENTS.md`.

For example, foundation-specific rules belong in:

```text
skills/foundation/AGENTS.md
```

## Verification

After changing skill structure or metadata, verify discovery from the repository root:

```bash
pnpx skills add . --list
```
