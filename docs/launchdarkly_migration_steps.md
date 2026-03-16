# LaunchDarkly Migration Guide: Extract, Migrate & Revert

Compact guide for **exporting** source data, **migrating** to a destination project, and **reverting** when needed. Lists only the main YAML options to set per workflow.

---

## Table of contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Key YAML settings per file](#key-yaml-settings-per-file)
4. [Phase 1: Extract](#phase-1-extract)
5. [Phase 2: Migrate](#phase-2-migrate)
6. [Phase 3: Revert](#phase-3-revert)
7. [Include / exclude flags](#include--exclude-flags)
8. [Quick reference: which file to use](#quick-reference-which-file-to-use)

---

## Overview

| Step | What it does |
|------|----------------|
| **Extract** | Reads flags, environments, (optionally segments) from **source** → saves under `data/launchdarkly-migrations/source/`. |
| **Map members** | (Optional) Maps source → destination members for maintainer assignment on migrated flags. |
| **Migrate** | Creates/updates flags (and optionally segments) in the **destination** from extracted data. |
| **Revert** | Finds migrated flags in destination, archives/deletes them, optionally cleans up views. |

Run via **workflow** (one YAML, one command) or **individual tasks** (extract, map-members, migrate, revert).

---

## Prerequisites

- **Deno** installed.
- **API keys** in `config/api_keys.json`: `source_account_api_key`, `destination_account_api_key` (same key if one account).

---

## Key YAML settings per file

Set these **before** running. Other options (e.g. `environmentMapping`, `targetView`) are optional.

### `workflow-full.yaml` (initial: extract → map-members → migrate)

| Setting | Purpose |
|--------|---------|
| `source.projectKey` | Source project key. |
| `destination.projectKey` | Destination project key. |
| `source.domain` / `destination.domain` | e.g. `app.launchdarkly.com` or `app.eu.launchdarkly.com`. |
| `extraction.includeSegments` | `true` if you migrate segments. |
| `migration.assignMaintainerIds` | `true` to run map-members and assign maintainers. |
| `migration.migrateSegments` | `true` only if `extraction.includeSegments: true`. |
| `migration.environments` | Env keys to migrate; omit = all. |
| `migration.includeFlags` / `migration.excludeFlags` | Limit which flags; omit = all. |

### `workflow-extract-migrate-incremental.yaml` (later: extract + incremental migrate)

| Setting | Purpose |
|--------|---------|
| `source.projectKey` / `destination.projectKey` | Same as initial migration. |
| `source.domain` / `destination.domain` | As needed. |
| `extraction.includeSegments` | `true` if `migration.migrateSegments: true`. |
| `migration.incremental` | Keep `true` for incremental sync. |
| `migration.assignMaintainerIds` | Usually `false`. |
| `migration.includeFlags` / `migration.excludeFlags` | Optional; omit = all. |

**Do not** set `migration.conflictPrefix` for incremental—it creates new prefixed flags instead of updating.

### `workflow-migrate-only.yaml` (migrate only; extract already done)

| Setting | Purpose |
|--------|---------|
| `source.projectKey` / `destination.projectKey` | Must match the project you extracted. |
| `migration.assignMaintainerIds` | `true` only if map-members was run and mapping exists. |
| `migration.migrateSegments` | `true` only if segments were extracted. |
| `migration.includeFlags` / `migration.excludeFlags` | Optional. |
| `migration.environments` | Optional; omit = all. |

### Revert (`revert-migration-example.yaml` or `workflow-revert-and-migrate.yaml`)

| Setting | Purpose |
|--------|---------|
| `source.projectKey` / `destination.projectKey` | Same as the migration you are reverting. |
| `revert.dryRun` | `true` = preview; `false` = execute. |
| `migration.targetView` | If you used a target view during migration; revert unlinks from it. |

---

## Phase 1: Extract

Writes source project data to `data/launchdarkly-migrations/source/`. Required before migrate or revert.

**Full workflow (extract + map + migrate):**
```bash
deno task workflow examples/workflow-full.yaml
```

**Extract + incremental migrate:**
```bash
deno task workflow examples/workflow-extract-migrate-incremental.yaml
```

**Extract-only:** Use a workflow with only `extract-source`, or run the extract script (see README).

---

## Phase 2: Migrate

Creates/updates flags (and optionally segments) in the destination. Use **dry run** first: set `migration.dryRun: true`, then `false` to apply.

**Full (initial):**
```bash
deno task workflow examples/workflow-full.yaml
```

**Incremental:**
```bash
deno task workflow examples/workflow-extract-migrate-incremental.yaml
```

**Migrate only:**
```bash
deno task workflow examples/workflow-migrate-only.yaml
```

**Important:** Do **not** use `conflictPrefix` with incremental—it creates new flags instead of updating.

---

## Phase 3: Revert

Removes migrated flags from the destination (and optionally approval requests, view links, views).

**Revert only:**
```bash
deno task revert -f examples/revert-migration-example.yaml --dry-run   # preview
deno task revert -f examples/revert-migration-example.yaml             # execute
```

**Revert then re-migrate:**
```bash
deno task workflow -f examples/workflow-revert-and-migrate.yaml
```
Set `revert.dryRun` and `migration.dryRun` to `true` for preview, then `false` to run.

---

## Include / exclude flags

**Only migrate specific flags:**
```yaml
migration:
  includeFlags:
    - feature-one
    - feature-two
```

**Exclude flags:**
```yaml
migration:
  excludeFlags:
    - deprecated-flag
```

Omit both to migrate all extracted flags.

**Other useful options:** `migration.environments`, `migration.environmentMapping`, `migration.targetView`, `migration.conflictPrefix` (see examples or README).

---

## Quick reference: which file to use

| Goal | YAML file | Command |
|------|-----------|---------|
| Initial: extract + map + migrate | `examples/workflow-full.yaml` | `deno task workflow examples/workflow-full.yaml` |
| Later: extract + incremental migrate | `examples/workflow-extract-migrate-incremental.yaml` | `deno task workflow examples/workflow-extract-migrate-incremental.yaml` |
| Migrate only | `examples/workflow-migrate-only.yaml` | `deno task workflow examples/workflow-migrate-only.yaml` |
| Revert only | `examples/revert-migration-example.yaml` | `deno task revert -f examples/revert-migration-example.yaml` |
| Revert then re-migrate | `examples/workflow-revert-and-migrate.yaml` | `deno task workflow -f examples/workflow-revert-and-migrate.yaml` |

Positional path also works: `deno task workflow examples/workflow-full.yaml`.

For full parameter lists and CLI options, see the main [README](../README.md).
