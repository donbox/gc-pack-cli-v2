---
title: "Pack And Registry CLI Surface"
---

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-04-16 |
| Author(s) | D. Box |
| Issue | — |
| Supersedes | The `gc pack`/`gc export` split introduced in 0.15.0 |

This document builds on the **PackV2** work from the 0.15.0 release and makes no change to package format or loader semantics.

It does however do two things:
1. Introduce the notion of a *registry* where Gas City packs can be published and discovered.
2. Propose a coherent CLI interface across all package operations (discovery, import, upgrade, et al.)

The proposed `gc pack` CLI is functionally the same as `gc import`. The primary difference is more precise targeting of which entity's imports are being impacted (in-scope pack vs. city pack vs. targeted rig). A secondary difference is that `add` accepts the result of a registry lookup, so you can add an import without ever typing a `/`.

This proposal would also subsume the two legacy `gc pack` commands to land on a single coherent surface area.

All of this said, the main new concept in this proposal is a registry, so let's begin there.

## Registries
A Gas City registry is simply a `registry.toml` file that is typically fetched over HTTP.

A `registry.toml` file is a list of packages with a name, version info, description, and the URL of the source. Registries do not store packs; they are an index of packs. Once the source URL is read from `registry.toml`, the registry is out of the loop.

The registry entry for a pack lists one or more versioned releases of the pack. Here is an example `registry.toml` file.


```toml
schema = 1

[[pack]]
name = "lighthouse"
description = "Harbor-watch checks and incident response workflows."
source = "https://packages.example/main/lighthouse"

  [[pack.release]]
  version = "1.2.0"
  commit = "abc123"
  hash = "sha256:..."
  description = "Adds dock patrol checks and improves incident triage."

[[pack]]
name = "weatherglass"
description = "Forecasting and telemetry helpers for harbor operations."
source = "https://packages.example/main/weatherglass"

  [[pack.release]]
  version = "0.4.0"
  commit = "def456"
  hash = "sha256:..."
  description = "First public release."
```

A registry can advertise multiple versions of the same pack, with distinct notes on each version:

```toml
[[pack]]
name = "lighthouse"
description = "Harbor-watch checks and incident response workflows."
source = "https://packages.example/main/lighthouse"

  [[pack.release]]
  version = "1.2.0"
  commit = "abc123"
  hash = "sha256:..."
  description = "Adds dock patrol checks and improves incident triage."

  [[pack.release]]
  version = "1.1.0"
  commit = "def456"
  hash = "sha256:..."
  description = "Stabilizes patrol behavior and wakeup handling."
```

To facilitate discovery, a Gas City installation maintains a list of registries that are consulted when searching for or enumerating available packs. That list is a system-managed file, `~/.gc/registries.toml`, and has the standard `add` / `list` / `remove` CLI operations. A fresh Gas City installation would normally include a `main` entry pointing at the Gas City-managed registry. A fuller example with two configured registries looks like this:

```toml
schema = 1

[[registry]]
name = "main"
source = "https://github.com/gastownhall/gascity-packs"

[[registry]]
name = "acme"
source = "https://github.com/acme/gascity-packs"
```

The `registries.toml` file can point to multiple registries. Each configured registry name must be unique locally; those local names are what disambiguate packs when two or more registries publish the same pack name.


## CLI Command Trees

The operations one wants to do when managing imports from one package to another have *some* overlap with the operations one wants to do on a registry; however, there are enough differences to warrant two command trees:
* `gc pack` which is focused exclusively on managing package-to-package import graphs
* `gc registry` which is focused on discovery of packages based on name, description, or version.

The two work in tandem: the result of a registry search is a qualified name that can be passed directly to the add command that creates the import.

This design is silent with respect to *where* packages are stored:
- Registries are index only
- Caches are largely opaque and contain fetched content only based on a pack's `[import]` declarations.


### `gc registry`

```text
gc registry list
gc registry add <registry-name> <source>
gc registry remove <registry-name>
gc registry search [query] [--registry <name>]
gc registry show <qualified-pack-name>
```


### `gc pack`

```text
gc pack add <source-or-name> [--name <import-name>] [--version <constraint>] [--pack <path>] [--rig <name-or-path>]
gc pack remove <import-name> [--pack <path>] [--rig <name-or-path>]
gc pack list [--transitive] [--pack <path>] [--rig <name-or-path>]
gc pack show <import-name> [--pack <path>] [--rig <name-or-path>]
gc pack fetch [<import-name>] [--pack <path>] [--rig <name-or-path>]
gc pack outdated [<import-name>] [--pack <path>] [--rig <name-or-path>]
gc pack upgrade [<import-name>] [--pack <path>] [--rig <name-or-path>]
```

## `gc pack`

`gc pack` owns imports, fetched state, and upgrade flow for the selected pack.

### Scope model

The baseline target is the ambient pack, when one can be discovered from the
working directory.

Working rules:

- if the current directory is inside a pack definition, the ambient pack is
  that surrounding pack
- if the current directory is inside a rigged directory but not inside a
  different pack definition, the ambient pack is the pack that defines the
  owning city
- if the current directory is in neither a pack directory nor a rigged
  directory, there is no ambient pack and the user must pass `--pack` or
  `--rig`

  
  There are two command arguments to most `gc pack commands`
- `--pack <path>` explicitly targets the pack the import will be added to 
- `--rig <name-or-path>` opts into rig-scoped import behavior within the city
  associated with the ambient or explicitly targeted pack
- rig-scoped imports only happen when `--rig` is explicitly passed
- `--rig` never creates an ambient rig scope on its own; it is always an
  explicit refinement

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
gc pack add <source-or-name> [--name <import-name>] [--version <constraint>] [--pack <path>] [--rig <name-or-path>]
```

- adds an import to the selected scope
- fetches the resolved content into cache by default
- accepts:
  - qualified registry names like `main:lighthouse`
  - unqualified registry names when resolution is unambiguous
  - direct source URLs
  - local paths
- `--name` gives an explicit local import name when needed
- `--version` records a version constraint in the import when the add is
  registry-backed
- if `--version` is omitted, registry-backed adds follow the registry's default
  resolution behavior
- this proposal does not freeze the resolver's default selection policy; it
  only requires that an explicit `--version` constraint, when present, be
  recorded in the import
- a qualified registry name is just a convenience handle; when `add` succeeds,
  the resulting import resolves to the underlying source or path rather than
  retaining the registry label as a separate identity


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
- respects version constraints already recorded on imports

#### `gc pack upgrade`

```text
gc pack upgrade [<import-name>] [--pack <path>] [--rig <name-or-path>]
```

- with no target, upgrades all imports in scope
- with a target, upgrades one imported pack
- is transitive as needed for coherent re-resolution
- fetches the new resolved result into cache
- respects version constraints already recorded on imports

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
- this is the full configured-registry view for this proposal

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
- there are intentionally no commands here for adding or removing packs from a
  registry itself; each registry owns and manages its own published
  `registry.toml`

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
main      lighthouse   1.2.0   Harbor-watch checks and incident response workflows.
acme      lighthouse   2.0.1   Acme-flavored harbor patrol and response tooling.
```

#### `gc registry show`

```text
gc registry show <qualified-pack-name>
```

- exact-address lookup for one pack catalog entry
- requires a qualified name like `main:lighthouse`
- unqualified names are intentionally not accepted here

Expected output:

```text
Pack:         acme:lighthouse
Registry:     acme
Name:         lighthouse
Latest:       2.0.1
Description:  Acme-flavored harbor patrol and response tooling.
Source:       https://packages.example/acme/lighthouse

Releases:
- 2.0.1  abc123  Adds patrol upgrades and doctor fixes.
- 1.9.0  def456  Stabilizes maintenance workflows.
```

### Registry resolution rules

- there is no `default` registry
- first-party registry name is `main`
- unqualified names only resolve when exactly one registry matches in contexts
  that allow unqualified resolution
- collisions require qualified names like `acme:lighthouse`

## File Formats

These are the file-format rules.

### `registries.toml`

This is the machine-known registry config.

- lives under `~/.gc/registries.toml`
- is edited by `gc registry add` / `gc registry remove`
- does not carry pack descriptions
- does not define a default registry

Example:

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
- `pack.toml` does not need a required description field for this proposal

Example:

```toml
schema = 1

[[pack]]
name = "lighthouse"
description = "Harbor-watch checks and incident response workflows."
source = "https://packages.example/main/lighthouse"

  [[pack.release]]
  version = "1.2.0"
  commit = "abc123"
  hash = "sha256:..."
  description = "Adds dock patrol checks and improves incident triage."
```


## Cache

The storage model is intentionally light:

| Thing | Role |
|---|---|
| registry | what exists |
| import | what this scope wants |
| cache | fetched bytes already available |

Working rules:

- keep a machine-global cache under `~/.gc`
- do not introduce a first-class machine-wide store
- do not assume `.gc` should be kept wholesale in source control
- treat `fetch` as cache-oriented, not as full materialization
- any later runtime materialization is explicitly outside the scope of this CLI
  proposal

## Parked Questions

These are intentionally parked to the side for now. The current design pass is
primarily trying to settle command names, signatures, and output shapes.

1. This design has two top-level command trees corresponding to the two nouns
   in play (packages and registries). If desired, we could embed the registry
   commands under pack, but it's unclear whether creating a secondary/scoped
   noun is better than having two peer nouns.
2. The current design relies on implicit caching of a lowered or processed form
   of a package, all stored under `.gc`. We lack an explicit mechanism for
   embedding any form of the imported packages into package content (a.k.a.
   vendoring), making the transitive closure of imported packages into a single
   deployable unit. Our import mechanism supports the scenario by embedding
   pack content under `assets/` and using path references in the `import`
   directive in `pack.toml`. But we don't provide CLI surface area to manage it.
