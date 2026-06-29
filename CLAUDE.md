# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

`sentio-ai-kit` is a **Claude Code plugin** — there is no application code, build step, test suite, or `package.json`. The entire deliverable is the **Markdown skill content** under `skills/`. "Working in this repo" means authoring and editing skill instructions, not writing software. Treat the prose as the product: accuracy of CLI flags, code snippets, and API field names is the whole point.

The skills teach Claude Code how to use the [Sentio](https://docs.sentio.xyz/) blockchain indexing platform. Content currently targets **Sentio SDK 4** (`@sentio/sdk@^4`, `@sentio/cli@^4`) — keep version-specific guidance (e.g. upstream `ethers` 6.x, not the dropped `@sentio/ethers` fork) consistent across all files when updating.

## Repository Layout

```
.claude-plugin/
  plugin.json          # Plugin manifest (name, version, metadata)
  marketplace.json     # Marketplace listing — also carries a version field
skills/
  sentio-processor/    # Skill: building/testing/deploying processors (SDK-side)
    SKILL.md           # Entry point — loaded when skill triggers
    references/        # Progressive-disclosure deep dives, loaded on demand
  sentio-platform/     # Skill: SQL, alerts, dashboards, endpoints (CLI/platform-side)
    SKILL.md
    references/openapi.swagger.json   # Synced artifact — see below
.github/workflows/publish-clawhub.yml # CI: auto-publishes skills to ClawHub
```

The two skills split by surface: **sentio-processor** = writing/deploying processor code (`@sentio/sdk` handlers, `sentio.yaml`, `sentio gen/build/upload`); **sentio-platform** = operating an existing project from the CLI (`sentio data sql`, `alert`, `dashboard`, `endpoint`, `processor` management). When adding guidance, put it in the skill matching that boundary.

## Dual Distribution (the key architectural fact)

The same `skills/` directory is shipped through **two independent channels**, so changes propagate to both:

1. **Claude Code plugin** — installed via `/plugin marketplace add sentioxyz/sentio-ai-kit` then `/plugin install sentio-ai-kit`. Driven by `.claude-plugin/plugin.json` + `marketplace.json`. The whole plugin (both skills) installs together.

2. **ClawHub** — each skill is published and installed **individually** (`npx clawhub@latest install sentio-processor`). `.github/workflows/publish-clawhub.yml` runs `clawhub sync --all` on every push to `main` that touches `skills/**` (or the workflow file), bumping the patch version by default. Manual runs (`workflow_dispatch`) allow choosing `patch`/`minor`/`major` and a changelog message. Publishing requires the `CLAWHUB_TOKEN` repo secret.

Implication: **any merge to `main` that touches `skills/**` auto-publishes a new ClawHub version.** Don't merge half-finished skill edits.

## Authoring Conventions

### SKILL.md frontmatter — the `description` is the trigger
Each `SKILL.md` starts with YAML frontmatter (`name`, `description`). The `description` is what Claude Code matches against to decide whether to load the skill, so it must enumerate concrete trigger conditions (CLI commands, task types, chains). Edit it deliberately when the skill's scope changes — vague descriptions cause the skill to mis-trigger or never load.

### Progressive disclosure
`SKILL.md` holds the high-frequency core (CLI lifecycle, all handler types, metrics/store overview, best practices, common-mistakes table). Lower-frequency depth lives in `references/*.md`, linked from a "Progressive Disclosure" list in `SKILL.md`. **When you add a reference file, add it to that list** (with a one-line description) so it's discoverable — an unlinked reference is effectively dead. Keep `SKILL.md` lean; push long examples and protocol-specific templates into `references/`.

### Code snippets must be SDK-4-correct
Snippets are copied verbatim by users. Use ESM import paths with `.js` extensions for generated types (`./types/{chain}/{name}.js`), real package names (`@sentio/sdk`, `@sentio/chain`, `@sentio/sdk/solana`), and respect the documented constraints already encoded in the skill (e.g. metric names must not end in `_count/_sum/_avg/_min/_max/_last`; enable `GLOBAL_CONFIG.execution = { sequential: true }` for store usage). If you change a pattern, grep the other skill files and references for the old form and update all copies.

### `openapi.swagger.json` is a synced artifact — do not hand-edit
`skills/sentio-platform/references/openapi.swagger.json` (~10k lines) is the Sentio REST API spec, updated by `chore: update sentio platform openapi spec` commits. Treat it as generated: regenerate/re-sync it from upstream rather than editing by hand. It's the schema source for dashboard-import JSON and CLI debugging referenced from `sentio-platform/SKILL.md`.

### Keep version fields in sync
`plugin.json` and `marketplace.json` each carry a `version`. When bumping the plugin version, update both. (ClawHub's per-skill versions are managed separately by the publish workflow's bump input.)

## Contribution Workflow

Per the workspace `CLAUDE.md`, this repo uses **pull requests — never commit directly to `main`**. Create a branch, open a PR, wait for approval before merging. Use **conventional commits with scopes**; the established scopes here are `skill`/`skills` and `claude`:

```
docs(skill): drop @sentio/ethers fork guidance, bump typescript to 6
refactor(skills): fold Sui SDK4 gRPC guide into sentio-processor
chore: update sentio platform openapi spec
```

This is a **public repository** — never reference the private `sentio/` or `production/` repos in commits, PRs, or skill content.
