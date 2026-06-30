# Probe: uv-0-11-25-schema-features

## Purpose

Tests Mend SCA detection against three interlocking features
introduced in **uv 0.11.25**:

1. **Lockfile schema v2** — `version = 2` in `uv.lock`, with
   `[package.tool-receipt]` per package. The parser must not
   treat `tool-receipt` as a dependency entry.
2. **Environment factoring / environment-markers** — a new
   `environments` header in `uv.lock` declares two build
   environments (`linux`, `windows`). Per-package
   `environment-markers` restricts conditional packages to
   their environment. Mend must preserve these markers in the
   detected dependency tree.
3. **Centralized storage layout** — uv 0.11.25 changes the
   default project-environment location from `.venv/` inside
   each project to a centralized cache keyed by project-path
   hash. The UA filesystem collector follows the old path and
   will find nothing; the probe asserts that the
   lockfile-driven tree (not a flat pip-freeze fallback) is
   the correct output.

## Pattern catalog entry

`plugins/python-uv/skills/uv-core/references/feature-coverage-patterns.md`
entry: `uv-0-11-25-schema-features`

## Categories

`lockfile_format`, `tree_structure`, `storage_layout`

## Probe structure

```
uv-0-11-25-schema-features-20260630-191743/
├── pyproject.toml          # Virtual workspace root
├── .python-version         # "3.11" — Mend PIP-chain precedence
├── uv.lock                 # lockfile version 2, two environments
├── packages/
│   ├── app-linux/
│   │   ├── pyproject.toml  # httpx + h2 (linux-only)
│   │   └── src/app_linux/__init__.py
│   └── app-win/
│       ├── pyproject.toml  # httpx + pywin32 (windows-only)
│       └── src/app_win/__init__.py
├── README.md               # this file
└── expected-tree.json      # ground truth tree
```

## Dependency graph

```
[root: uv-0-11-25-schema-features]
├── app-linux (local/virtual, both envs)
│   ├── httpx 0.27.2 (registry, unconditional)
│   │   ├── anyio 4.4.0 -> idna, sniffio
│   │   ├── certifi 2024.8.30
│   │   ├── httpcore 1.0.5 -> certifi, h11
│   │   └── idna 3.7
│   └── h2 4.1.0 (registry, linux-only)
│       ├── hpack 4.0.0 (linux-only)
│       └── hyperframe 6.0.1 (linux-only)
└── app-win (local/virtual, both envs)
    ├── httpx 0.27.2 (shared, deduplicated)
    └── pywin32 306 (registry, windows-only)
```

## Package inventory

| Package        | Version     | Marker                       |
|----------------|-------------|------------------------------|
| httpx          | 0.27.2      | (none — unconditional)       |
| anyio          | 4.4.0       | (none)                       |
| certifi        | 2024.8.30   | (none)                       |
| httpcore       | 1.0.5       | (none)                       |
| h11            | 0.14.0      | (none)                       |
| idna           | 3.7         | (none)                       |
| sniffio        | 1.3.1       | (none)                       |
| h2             | 4.1.0       | `sys_platform == 'linux'`    |
| hpack          | 4.0.0       | `sys_platform == 'linux'`    |
| hyperframe     | 6.0.1       | `sys_platform == 'linux'`    |
| pywin32        | 306         | `sys_platform == 'win32'`    |

## Python version detection

Mend uses the `.python-version` file (PIP precedence chain, higher
priority than `requires-python` in `pyproject.toml`). This probe
emits `.python-version` containing `3.11`.

The workspace members also declare `requires-python = ">=3.11"` in
their `pyproject.toml` files. Both are compatible. Mend will read
`.python-version` and use Python 3.11.

## Mend config

**Bucket B — no `.whitesource` required.**

Python version is dynamically detected from `.python-version`
(PIP-chain highest precedence) and `requires-python = ">=3.11"`.
uv itself is not in the install-tool list and cannot be pinned via
`scanSettings.versioning`. No `.whitesource` is emitted.

If Mend's UA needs to be forced to lockfile-driven resolution:

```properties
# whitesource.config (optional, for lockfile-only mode)
python.resolveDependencies=true
python.resolveHierarchyTree=true
```

## Key Mend failure modes this probe targets

1. **Lockfile v2 parse failure** — if the parser exits on
   `version = 2` or `[package.tool-receipt]`, zero packages are
   reported. Expected: all 13 packages present.

2. **environment-markers stripped** — if Mend collapses all
   packages to unconditional, `h2`, `hpack`, `hyperframe`, and
   `pywin32` lose their `marker` field. Downstream platform-
   filtering becomes incorrect.

3. **Multi-environment packages dropped** — if Mend picks one
   branch (linux or windows) and discards the other, either
   `h2`/`hpack`/`hyperframe` or `pywin32` is missing entirely.

4. **Filesystem fallback on missing .venv** — centralized storage
   means `.venv/site-packages` is absent at the project root. If
   UA falls back to pip-resolver (flat list from manifest), the
   tree loses hierarchy and markers. Expected tree must not be a
   flat list.

5. **tool-receipt misidentified as dependency** — if the parser
   reads `[package.tool-receipt]` as a regular package entry, a
   spurious "tool-receipt" package appears in the tree.

## Schema version

`1.2` — uses `pm_version_under_test` for version-axis pinning.

## Resolver knowledge note

The upstream UA resolver (`python.md`) documents UV project
filtering via `MEND_SCA_UV_PROJECTS` env var but does not
specifically mention lockfile v2. The `environment-markers` and
`tool-receipt` behaviors are not yet covered in the resolver
knowledge file — this probe is **exploratory** for those features.
The centralized storage layout is covered implicitly by the
`UV Project Filtering` section (which removes `.venv` folder
manifests when `MEND_SCA_UV_PROJECTS` is set). Downstream
comparators should treat environment-markers and tool-receipt
deviation as exploratory rather than a known regression.

## Generated by

dispatcher → python-uv:project-creator + python-uv:tree-builder
Generated: 2026-06-30T19:17:43Z
Resolver fetched: 2026-06-30T19:16:02Z
Resolver SHA: 877d6d848391d838fb31b38dadf37d3ad0696cbc
