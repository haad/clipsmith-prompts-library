# Architecture Research

**Domain:** Static prompt repository with Markdown-to-JSON build pipeline and GitHub Pages deployment
**Researched:** 2026-03-11
**Confidence:** HIGH — design document (`dist/PROMPTS-REPO-DESIGN.md`) provides complete, authoritative specification

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          SOURCE LAYER (main branch)                  │
├──────────────────┬────────────────────────┬─────────────────────────┤
│  Prompt Sources  │   Build Config         │   Build Infrastructure  │
│  ┌────────────┐  │  ┌──────────────────┐  │  ┌───────────────────┐  │
│  │ prompts/   │  │  │  catalog.yaml    │  │  │ scripts/build.sh  │  │
│  │ coding/    │  │  │  (catalog ver,   │  │  │ (bash, zero deps) │  │
│  │ writing/   │  │  │   valid cats)    │  │  └────────┬──────────┘  │
│  │ analysis/  │  │  └──────────────────┘  │           │             │
│  │ creative/  │  │  ┌──────────────────┐  │           │             │
│  │ *.md files │  │  │ schema/          │  │           │             │
│  └─────┬──────┘  │  │ prompt.schema.   │  │           │             │
│        │         │  │ json             │  │           │             │
│        │         │  └──────────────────┘  │           │             │
└────────┼─────────┴────────────────────────┴───────────┼─────────────┘
         │                                               │
         └──────────────────────┬────────────────────────┘
                                │
                         ┌──────▼──────┐
                         │  BUILD STEP  │
                         │ (GitHub CI)  │
                         └──────┬───────┘
                                │
         ┌──────────────────────┼────────────────────────┐
         │                      │                        │
    ┌────▼────┐           ┌──────▼──────┐         ┌──────▼──────┐
    │ Parse   │           │  Validate   │         │   Assemble  │
    │ .md     │           │  schema,    │         │  prompts.   │
    │ files   │           │  dups, cats │         │  json       │
    └─────────┘           └─────────────┘         └──────┬──────┘
                                                         │
┌────────────────────────────────────────────────────────┼────────────┐
│                  DISTRIBUTION LAYER (gh-pages branch)  │            │
├────────────────────────────────────────────────────────┼────────────┤
│                                                  ┌──────▼──────┐    │
│                                                  │ prompts.json│    │
│                                                  │ (static CDN)│    │
│                                                  └──────┬──────┘    │
└─────────────────────────────────────────────────────────┼───────────┘
                                                          │
                                                   ┌──────▼──────┐
                                                   │   Flycut     │
                                                   │  (iOS/macOS) │
                                                   │  sync client │
                                                   └─────────────┘
```

### Component Responsibilities

| Component | Responsibility | Boundaries |
|-----------|----------------|------------|
| `prompts/**/*.md` | Source of truth for prompt content. Markdown body = prompt text. YAML frontmatter = metadata. | Input only. Never modified by build. |
| `catalog.yaml` | Catalog-level version + valid category list. Single authority on what categories are legal. | Read by build script. Manually maintained. |
| `scripts/build.sh` | Orchestrates entire build: parse → validate → assemble → write output. Zero external deps. | Reads source files. Writes `prompts.json`. Exits non-zero on any error. |
| `schema/prompt.schema.json` | JSON Schema contract between repo and Flycut app. Validates built output matches expected structure. | Referenced by build script and CI validation step. Not read by Flycut. |
| `.github/workflows/build-prompts.yml` | CI/CD pipeline. Triggers on push to main, runs build script, deploys to gh-pages. | Orchestrates build script. Only publishes `prompts.json` to gh-pages. |
| `prompts.json` (main branch) | Build artifact committed to main for PR review visibility. Overwritten on every CI run. | Read by reviewers. Overwritten by CI. Not manually edited. |
| `prompts.json` (gh-pages branch) | Live distribution artifact. Served over HTTPS. Fetched by Flycut sync. | Deployed by CI only. Read by Flycut. |

## Recommended Project Structure

```
flycut-prompts/
├── prompts/                     # Source prompt files (category subdirs)
│   ├── coding/                  # Category directory — name = default category value
│   │   └── *.md                 # id derived from filename (strip .md)
│   ├── writing/
│   │   └── *.md
│   ├── analysis/
│   │   └── *.md
│   └── creative/
│       └── *.md
├── scripts/
│   └── build.sh                 # Build entrypoint; must be chmod +x
├── .github/
│   └── workflows/
│       └── build-prompts.yml    # CI trigger and gh-pages deploy
├── schema/
│   └── prompt.schema.json       # JSON Schema for output validation
├── catalog.yaml                 # Catalog version + valid categories
├── prompts.json                 # AUTO-GENERATED — committed for PR visibility
├── README.md                    # Contributor docs + prompt table
└── .gitignore
```

### Structure Rationale

- **`prompts/[category]/`:** Directory name carries category semantics. Eliminates required `category:` frontmatter field in 99% of cases. Convention over configuration.
- **`scripts/`:** Isolates build tooling from prompt content. Makes the build step obvious and easy to invoke locally (`bash scripts/build.sh`).
- **`schema/`:** Separates the contract definition from the tool that uses it. Schema is readable by humans and reusable by other validators.
- **`catalog.yaml` at root:** Single place to bump catalog version and add/remove categories. Kept simple and visible.
- **`prompts.json` committed to main:** Intentional design decision. Lets PR reviewers see the generated diff without running the build. CI overwrites it, so it stays current.

## Architectural Patterns

### Pattern 1: Directory-Derived Metadata

**What:** The subdirectory a file lives in determines the `category` field in the output JSON. Frontmatter `category:` field is an optional override for edge cases.

**When to use:** Always. This is the primary categorization mechanism.

**Trade-offs:** Simple for contributors (no category field needed), slightly magic for newcomers (they must know "directory = category"). The override valve handles legitimate edge cases without complicating the common case.

**Example:**
```
prompts/coding/fix-bug.md  →  { "category": "coding" }
prompts/writing/fix-bug.md →  ERROR: duplicate id "fix-bug"
```

### Pattern 2: Filename-as-ID

**What:** The `.md` filename without extension is the stable `id` in `prompts.json`. IDs must be unique across ALL categories, not just within one directory.

**When to use:** Always. IDs are the sync key in Flycut; changing a filename changes the ID and breaks upsert continuity.

**Trade-offs:** Simple to derive, but makes renaming a breaking change. Uniqueness constraint is global, which prevents accidental ID collisions across categories.

**Example:**
```bash
# filename → id mapping
prompts/coding/explain-code.md    →  "id": "explain-code"
prompts/creative/brainstorm.md    →  "id": "brainstorm"

# duplicate detection
prompts/coding/fix-bug.md
prompts/analysis/fix-bug.md       →  BUILD ERROR: Duplicate id 'fix-bug'
```

### Pattern 3: Version-Gated Sync

**What:** Each prompt has a monotonically increasing integer `version`. Flycut only applies a remote prompt update when `remote.version > local.version`. Bumping version is the explicit publish mechanism.

**When to use:** Every time prompt content changes. Version must be bumped manually in frontmatter.

**Trade-offs:** Requires contributor discipline (easy to forget). No automation possible in a zero-dependency bash build. The constraint is well-documented in README but enforcement is social, not technical.

**Example:**
```yaml
# prompts/coding/fix-bug.md
---
title: "Fix Bug"
version: 2          # was 1, bumped because content changed
---
...updated prompt content...
```

### Pattern 4: Separation of Source Branch and Distribution Branch

**What:** Prompt source files live on `main`. The built artifact (`prompts.json`) is deployed to `gh-pages`. Flycut syncs from `gh-pages`, never from `main`.

**When to use:** Always for this repo. This pattern prevents accidental serving of partial builds, broken JSON, or source files to the Flycut sync endpoint.

**Trade-offs:** Two-branch model adds a small conceptual overhead. The benefit is that `gh-pages` always reflects a complete, validated, CI-built artifact.

## Data Flow

### Build Flow (Markdown → JSON)

```
prompts/**/*.md  ──┐
                   │
catalog.yaml  ─────┼──→  scripts/build.sh
                   │         │
schema/*.json ─────┘         │
                             │
                     ┌───────▼────────────────────┐
                     │  1. find all .md files      │
                     │     sort alphabetically     │
                     ├────────────────────────────┤
                     │  2. for each file:          │
                     │     - derive id (filename)  │
                     │     - parse frontmatter     │
                     │       (sed/awk/grep)        │
                     │     - extract body (awk)    │
                     │     - resolve category      │
                     │       (dir or override)     │
                     │     - JSON-escape content   │
                     │       (jq -Rs)              │
                     ├────────────────────────────┤
                     │  3. validate:               │
                     │     - no duplicate ids      │
                     │     - title present         │
                     │     - version present       │
                     │     - category in allow-    │
                     │       list (catalog.yaml)   │
                     ├────────────────────────────┤
                     │  4. assemble output:        │
                     │     read catalog version    │
                     │     sort by id              │
                     │     wrap in {version,       │
                     │       prompts: [...]}       │
                     └───────────┬────────────────┘
                                 │
                          prompts.json
                          (written to repo root)
```

### CI/CD Deploy Flow

```
Push to main
    │
    ├── paths filter:
    │   prompts/**  OR  catalog.yaml  OR  scripts/build.sh
    │
    ▼
GitHub Actions: build-and-deploy job
    │
    ├── checkout@v4 (main branch)
    ├── apt-get install jq
    ├── bash scripts/build.sh  ──→  prompts.json
    ├── jq validation checks:
    │     .version present
    │     .prompts length > 0
    │     all fields present
    │     no duplicate ids
    │
    └── peaceiris/actions-gh-pages@v4
            publish_dir: ./_deploy
            publish_branch: gh-pages
            (only prompts.json + optional index.html)
                │
                ▼
        gh-pages branch
        raw.githubusercontent.com/.../gh-pages/prompts.json
                │
                ▼
        Flycut sync client (iOS/macOS)
        URLSession fetch → JSONDecoder → CoreData upsert
```

### Sync Flow (Flycut consuming prompts.json)

```
prompts.json (gh-pages)
    │
    ▼
Flycut sync triggered (manual or periodic)
    │
    ├── fetch URL → decode PromptCatalog
    │
    └── for each PromptDTO:
            id exists locally?
            ├── NO  → insert
            └── YES → is user-customized?
                       ├── YES → skip (protect user edit)
                       └── NO  → remote.version > local.version?
                                  ├── YES → update
                                  └── NO  → skip (already current)
```

### Key Data Flows

1. **New prompt publish:** Author creates `.md` → opens PR → CI validates build → merge to main → CI deploys `prompts.json` to gh-pages → Flycut syncs on next check.

2. **Prompt update:** Author edits `.md` content AND bumps `version:` → merge → CI deploys → Flycut upserts the prompt (version gate passes).

3. **Broken build:** Any validation failure in `build.sh` → CI job fails → `prompts.json` on gh-pages is NOT updated → Flycut continues serving previous working version.

4. **Local build:** Contributor runs `bash scripts/build.sh` → `prompts.json` updated in working tree → inspect with `jq . prompts.json` → not committed unless desired.

## Build Order (What to Create First)

The following order minimizes rework and enables progressive testing:

1. **`catalog.yaml`** — Must exist before `build.sh` can run. Defines the category allowlist and initial catalog version.

2. **`scripts/build.sh`** — The core artifact. All other pieces depend on it working correctly. Create and test with a single placeholder `.md` file before adding all 23 prompts.

3. **`schema/prompt.schema.json`** — Create alongside the build script. Serves as the contract specification and enables `jq` schema validation in the build.

4. **`prompts/**/*.md`** — Add all 23 prompt files after the build pipeline is verified working. Build script can then be run end-to-end with the real corpus.

5. **`.github/workflows/build-prompts.yml`** — Add CI after local build is confirmed working. Push to main to verify CI build and gh-pages deployment.

6. **`README.md`** — Add last, after the pipeline is complete. README references the sync URL, contributor workflow, and prompt table — all of which depend on earlier components being final.

7. **`.gitignore`** — Can be added at any step; no dependencies.

## Anti-Patterns

### Anti-Pattern 1: Using `prompts.json` from main as the Flycut Sync URL

**What people do:** Configure Flycut to fetch `https://raw.githubusercontent.com/.../main/prompts.json` instead of the gh-pages URL.

**Why it's wrong:** The `main` branch copy of `prompts.json` is a build artifact committed for PR review convenience. It may lag behind CI (if someone pushed source files but CI hasn't run yet), or it could be out of date if the file was manually edited. The `gh-pages` copy is always the result of a successful CI build.

**Do this instead:** Use the gh-pages URL (`/gh-pages/prompts.json` or GitHub Pages domain). This is the only URL that guarantees a validated, CI-built artifact.

### Anti-Pattern 2: Forgetting to Bump `version` on Content Changes

**What people do:** Edit prompt content, open a PR, merge — but leave `version: 1` unchanged.

**Why it's wrong:** Flycut's sync is version-gated. `remote.version > local.version` is the update trigger. If version stays the same, users who already have the prompt will never receive the content update.

**Do this instead:** Every content change to a `.md` file must increment the `version` field in its frontmatter. Document this prominently in the README and PR template.

### Anti-Pattern 3: Reusing IDs Across Categories

**What people do:** Create `prompts/coding/review.md` and `prompts/writing/review.md` thinking they are distinct because they're in different directories.

**Why it's wrong:** The `id` is derived from filename only (not path). Both would produce `"id": "review"`. The build script must reject this as a duplicate — both can't coexist in `prompts.json`.

**Do this instead:** Use category-qualified filenames where ambiguity exists: `code-review.md`, `writing-review.md`.

### Anti-Pattern 4: Introducing a Dependency in the Build Script

**What people do:** Add `npm install` or `pip install` to parse YAML properly, or use `python3 -c "import yaml"`.

**Why it's wrong:** The build script must work on vanilla Ubuntu (GitHub Actions runner) and macOS without any package installation beyond `jq`. External dependency managers introduce versioning problems, slower CI, and setup complexity.

**Do this instead:** Parse YAML frontmatter with `sed`, `awk`, and `grep`. These tools are always available. The frontmatter format is simple enough that a full YAML parser is unnecessary.

### Anti-Pattern 5: Publishing Source Files to gh-pages

**What people do:** Set `publish_dir: .` without filtering, deploying the entire repo (including `prompts/`, `scripts/`, `.github/`) to gh-pages.

**Why it's wrong:** Flycut fetches `prompts.json` — the other files serve no purpose on gh-pages and may confuse browsers. Worse, if someone navigates to the repo URL they see raw source files.

**Do this instead:** Use a `_deploy/` staging directory. Copy only `prompts.json` (and optionally an `index.html`) into it. Set `publish_dir: ./_deploy`.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| GitHub Actions | Push event triggers workflow via `on.push.paths` filter | Only runs when relevant files change; `workflow_dispatch` for manual trigger |
| GitHub Pages / raw.githubusercontent.com | Static file serving from `gh-pages` branch | No configuration beyond enabling Pages on the repo; raw URL available even without Pages enabled |
| `peaceiris/actions-gh-pages@v4` | Third-party action handles force-push to gh-pages branch | Requires `contents: write` permission; uses `GITHUB_TOKEN` |
| Flycut app (iOS/macOS) | HTTP GET to JSON URL; no auth; URLSession + JSONDecoder | Sync is pull-only; repo has no knowledge of clients |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| `prompts/*.md` to `build.sh` | Filesystem read via `find` + `awk`/`sed` | Build script owns parsing logic; source files are dumb data |
| `catalog.yaml` to `build.sh` | Filesystem read via `grep`/`awk` | Single read at build start; catalog version and category list extracted |
| `build.sh` to `prompts.json` | Filesystem write (stdout redirect) | Atomic replacement; build either succeeds fully or exits non-zero |
| `build.sh` to `schema/prompt.schema.json` | Used by `jq` for optional schema validation | Validation is advisory in local builds; mandatory in CI |
| GitHub Actions to gh-pages | `peaceiris/actions-gh-pages` force-pushes `_deploy/` contents | Replaces gh-pages branch on every successful build; no history retained |

## Scaling Considerations

This is a static content distribution system. Scaling properties are determined by GitHub's infrastructure.

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 23 prompts (initial) | Single `build.sh` pass, single JSON file. No optimization needed. Build time < 1 second. |
| 100-500 prompts | Same architecture. Build time still trivial for bash + jq. Consider splitting into category-scoped JSON files if payload size becomes a concern for mobile. |
| 1000+ prompts | JSON payload size (~500 bytes/prompt uncompressed) approaches 500KB. GitHub Pages serves gzip automatically. Flycut would need conditional fetch (ETag/Last-Modified) to avoid re-parsing on every sync. Consider splitting by category. |

## Sources

- `dist/PROMPTS-REPO-DESIGN.md` — Complete specification; **primary authoritative source** for this architecture (HIGH confidence)
- `dist/PROMPTS-REPO-DESIGN.md` — GitHub Action workflow specification with exact YAML (HIGH confidence)
- `.planning/PROJECT.md` — Project constraints and out-of-scope boundaries (HIGH confidence)
- `peaceiris/actions-gh-pages` — Standard GitHub Pages deploy action, widely used pattern (HIGH confidence — pattern is well-established)

---
*Architecture research for: Flycut Prompts Repository (Markdown-to-JSON build pipeline)*
*Researched: 2026-03-11*
