---
title: "Pack And Registry CLI Surface"
---

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-04-16 |
| Author(s) | Nora, Noreen |
| Issue | — |
| Supersedes | earlier `gc pack`-only and `pack`/`store` explorations |

Design document for the user-facing `gc pack` and `gc registry` surface.

## Command Trees

These are the working command trees reviewers should look at first.

### `gc pack`

```text
gc pack add <source-or-name> [--name <import-name>] [--pack <path>] [--rig <name-or-path>]
gc pack remove <import-name> [--pack <path>] [--rig <name-or-path>]
gc pack list [--transitive] [--pack <path>] [--rig <name-or-path>]
gc pack show <import-name> [--pack <path>] [--rig <name-or-path>]
gc pack fetch [<import-name>] [--pack <path>] [--rig <name-or-path>]
gc pack outdated [<import-name>] [--pack <path>] [--rig <name-or-path>]
gc pack upgrade [<import-name>] [--pack <path>] [--rig <name-or-path>]
```

### `gc registry`

```text
gc registry list
gc registry add <registry-name> <source>
gc registry remove <registry-name>
gc registry search [query] [--registry <name>]
gc registry show <qualified-pack-name>
```

## Current Proposal

This is the cohesive model the mock and the rest of this note now reflect.

- `gc pack` is the local workflow noun
- `gc registry` is the machine-known registry config and catalog-browsing noun
- registries are index only
- caches are fetched content only
- imports declare intent
- full materialization is optional and demand-driven

The overall design should follow npm/pub-like user expectations for pack
workflow, while keeping registry configuration and discovery as a separate
supporting surface.

## `gc pack`

`gc pack` owns imports, fetched state, and upgrade flow for the selected pack.

### Scope model

The baseline target is always the ambient pack discovered from the current
working directory.

Working rules:

- ambient behavior always targets the current pack
- `--pack <path>` targets another pack explicitly
- `--rig <name-or-path>` opts into rig-scoped import behavior
- rig-scoped imports only happen when `--rig` is passed
- `--rig` refines pack behavior; it is never ambient

### Verb set

| Verb | Meaning |
|---|---|
| `add` | Add an import to the selected scope and fetch it by default. |
| `remove` | Remove an import from the selected scope. |
| `list` | List imports in the selected scope. |
| `show` | Show one imported pack in the selected scope. |
| `fetch` | Fetch resolved pack content into cache for the selected scope. |
| `outdated` | Show which imported packs could be upgraded. |
| `upgrade` | Upgrade imported packs in scope and fetch the new resolved result. |

### Signatures and semantics

#### `gc pack add`

```text
gc pack add <source-or-name> [--name <import-name>] [--pack <path>] [--rig <name-or-path>]
```

- adds an import to the selected scope
- fetches the resolved content into cache by default
- accepts:
  - qualified registry names like `main:maintenance`
  - unqualified registry names when resolution is unambiguous
  - direct source URLs
  - local paths
- `--name` gives an explicit local import name when needed

#### `gc pack remove`

```text
gc pack remove <import-name> [--pack <path>] [--rig <name-or-path>]
```

- removes an import from the selected scope
- does not imply eager cache deletion
- remains strict about inbound reference blockers

#### `gc pack list`

```text
gc pack list [--transitive] [--pack <path>] [--rig <name-or-path>]
```

- with no flags, lists direct imports in scope
- with `--transitive`, lists the full resolved transitive set
- `--transitive` is the POR simplification in place of an initial separate
  `deps` verb

#### `gc pack show`

```text
gc pack show <import-name> [--pack <path>] [--rig <name-or-path>]
```

- shows one imported pack in the selected scope
- local inspection only
- does not reach out to registry catalog views

Expected output shape:

- imported name
- source
- resolved version or release
- fetched status
- scope

#### `gc pack fetch`

```text
gc pack fetch [<import-name>] [--pack <path>] [--rig <name-or-path>]
```

- with no target, fetches all imports in scope
- with a target, fetches one imported pack in scope
- does not edit imports
- is the explicit warm-cache and reconcile command

#### `gc pack outdated`

```text
gc pack outdated [<import-name>] [--pack <path>] [--rig <name-or-path>]
```

- shows what `upgrade` could move
- does not mutate imports or cache
- with no target, reports all outdated imports in scope
- with a target, reports one imported pack

#### `gc pack upgrade`

```text
gc pack upgrade [<import-name>] [--pack <path>] [--rig <name-or-path>]
```

- with no target, upgrades all imports in scope
- with a target, upgrades one imported pack
- is transitive as needed for coherent re-resolution
- fetches the new resolved result into cache

## `gc registry`

`gc registry` owns machine-known registry configuration and catalog browsing.

### Surface area

| Command | Meaning |
|---|---|
| `gc registry list` | List configured registries from `~/.gc/registries.toml`. |
| `gc registry add <registry-name> <source>` | Add one configured registry entry. |
| `gc registry remove <registry-name>` | Remove one configured registry entry. |
| `gc registry search [query] [--registry <name>]` | Search pack entries across all registries by default; with no query, return everything. |
| `gc registry show <qualified-pack-name>` | Show one exact pack catalog entry from a registry. |

### Signatures and semantics

#### `gc registry list`

```text
gc registry list
```

- lists configured registries from `~/.gc/registries.toml`
- this is the full configured-registry view for POR

Expected output:

```text
Name   Source
main   https://github.com/gastownhall/gascity-packs
acme   https://github.com/acme/gascity-packs
```

#### `gc registry add`

```text
gc registry add <registry-name> <source>
```

- adds one configured registry entry
- edits `~/.gc/registries.toml`

#### `gc registry remove`

```text
gc registry remove <registry-name>
```

- removes one configured registry entry
- edits `~/.gc/registries.toml`

#### `gc registry search`

```text
gc registry search [query] [--registry <name>]
```

- uses a plain text query, not regex
- with no query, returns all available pack entries
- searches across all configured registries by default
- `--registry <name>` narrows the search to one registry
- returns multiple results

Expected output:

```text
Registry  Name         Latest  Description
main      maintenance  1.2.0   Health checks and baseline operational tooling.
acme      maintenance  2.0.1   Acme-flavored maintenance tasks and patrol tooling.
```

#### `gc registry show`

```text
gc registry show <qualified-pack-name>
```

- exact-address lookup for one pack catalog entry
- requires a qualified name like `main:maintenance`
- unqualified names are intentionally not accepted here

Expected output:

```text
Pack:         acme:maintenance
Registry:     acme
Name:         maintenance
Latest:       2.0.1
Description:  Acme-flavored maintenance tasks and patrol tooling.
Source:       https://github.com/acme/maintenance

Releases:
- 2.0.1  abc123  Adds patrol upgrades and doctor fixes.
- 1.9.0  def456  Stabilizes maintenance workflows.
```

### Registry resolution rules

- there is no `default` registry
- first-party registry name is `main`
- unqualified names only resolve when exactly one registry matches in contexts
  that allow unqualified resolution
- collisions require qualified names like `acme:maintenance`

## File Formats

These are the POR file-format rules.

### `registries.toml`

This is the machine-known registry config.

- lives under `~/.gc/registries.toml`
- is edited by `gc registry add` / `gc registry remove`
- does not carry pack descriptions
- does not define a default registry

POR example:

```toml
schema = 1

[[registry]]
name = "main"
source = "https://github.com/gastownhall/gascity-packs"

[[registry]]
name = "acme"
source = "https://github.com/acme/gascity-packs"
```

### `registry.toml`

This is the published registry catalog file.

- each `[[pack]]` entry has a required `description`
- each `[[pack.release]]` entry has a required `description`
- `pack.toml` does not need a required description field for POR

POR example:

```toml
schema = 1

[[pack]]
name = "maintenance"
description = "Health checks and baseline operational tooling."
source = "https://github.com/gastownhall/maintenance"

  [[pack.release]]
  version = "1.2.0"
  commit = "abc123"
  hash = "sha256:..."
  description = "Adds doctor checks and improves stale-db handling."
```

### Provenance note

This registry model comes from earlier design work that was paused while
PackV2 moved forward. It should be treated as a well thought out and cohesive
design for machine-known registry configuration and catalog shape.

## Cache And Materialization

The storage model is intentionally light:

| Thing | Role |
|---|---|
| registry | what exists |
| import | what this scope wants |
| cache | fetched bytes already available |
| materialization | optional realized local tree when runtime behavior needs it |

Working rules:

- keep a machine-global cache under `~/.gc`
- do not introduce a first-class machine-wide store
- do not assume `.gc` should be kept wholesale in source control
- treat `fetch` as cache-oriented, not as full materialization

If materialization happens:

- it is a runtime concern rather than the primary pack CLI story
- rig behavior is still explicit through `--rig`

## Deferred Questions

The following are intentionally deferred:

- exact reverse-dependency or blocker-reporting flags beyond `list --transitive`
- whether pack output should later surface more explicit reason/provenance data
- whether a future richer registry admin surface needs extra flags or status metadata
- whether any materialized subset of `.gc` should later become explicitly persistent/shareable
