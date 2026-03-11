# Project Research Summary

**Project:** Flycut Prompts Repository (prompt-library)
**Domain:** Static prompt catalog with Markdown-to-JSON build pipeline and GitHub Pages distribution
**Researched:** 2026-03-11
**Confidence:** HIGH

## Executive Summary

This project is a curated prompt library designed as a distribution mechanism for the Flycut iOS/macOS app. The correct mental model is a static content pipeline, not a web application: Markdown source files on `main` are compiled by a bash build script into a single `prompts.json` artifact, deployed to a `gh-pages` branch, and fetched by Flycut clients via a stable HTTPS URL. Research across all four domains confirms that the entire system is buildable with zero external dependencies beyond `jq` — no Node.js, no Python packages, no package managers. This is not a trade-off but a hard constraint baked into the project design.

The recommended approach is a four-layer architecture: source files (`prompts/**/*.md`) → build script (`scripts/build.sh`) → CI/CD pipeline (GitHub Actions) → distribution (`gh-pages` branch). The build script handles all transformation: YAML frontmatter parsing with `sed`/`awk`/`grep`, JSON construction and escaping with `jq`, and validation with `jq -e` assertions. GitHub Actions orchestrates the pipeline on push to `main`, triggering only when relevant files change. The `peaceiris/actions-gh-pages@v4` action deploys a staging directory (`_deploy/`) containing only `prompts.json` and a minimal `index.html`.

The key risks are implementation-level, not architectural. The most dangerous: macOS bash 3.2 incompatibility with `declare -A` will break local developer builds if the build script uses associative arrays; the fix is a one-line POSIX-compatible alternative using string concatenation and `grep -w`. Second risk: version field discipline. Flycut's sync is version-gated — if a contributor edits prompt content without bumping the integer `version` field, deployed users silently receive no update. This must be enforced by CI, not README documentation alone. Third risk: filename immutability — renaming a `.md` file changes the prompt's `id`, orphaning it in every user's local Flycut store permanently. All three risks are preventable with upfront investment in the right guard rails.

## Key Findings

### Recommended Stack

The stack is intentionally minimal. `bash 5.x` (available on Ubuntu CI runners) is the build runtime — no install required. `jq 1.6` is the only non-system dependency and is pre-installed on `ubuntu-latest` via apt. YAML frontmatter parsing uses `sed`, `awk`, and `grep` — POSIX tools available on macOS and Ubuntu without any setup. GitHub Actions CI uses `actions/checkout@v4` (Node 20, current) and `peaceiris/actions-gh-pages@v4` for gh-pages deployment. The `include_files` parameter on `peaceiris/actions-gh-pages` is NOT implemented — deployment must use a `_deploy/` staging directory.

One non-obvious macOS constraint: the system bash at `/bin/bash` is version 3.2 (Apple's GPLv2 freeze). Any bash 4+ feature (`declare -A`, `mapfile`) will silently break local builds for contributors on standard macOS. Use POSIX-compatible patterns throughout.

**Core technologies:**
- `bash 5.x`: Build script runtime — zero-dependency, pre-installed on all CI environments
- `jq 1.6`: JSON construction and validation — the only safe way to escape prompt content containing quotes, backslashes, and newlines
- `sed`/`awk`/`grep`: YAML frontmatter parsing — POSIX tools sufficient for flat key-value frontmatter; no real YAML parser needed
- `actions/checkout@v4`: Git checkout in CI — Node 20 runtime, v3 is EOL
- `peaceiris/actions-gh-pages@v4`: gh-pages branch deployment — simpler than official `actions/deploy-pages` for single-file use case

### Expected Features

The feature set is well-defined by the project spec. Every table-stakes feature is load-bearing for the Flycut sync contract: stable unique IDs (filename-derived), per-prompt monotonic integer versions, and a correctly structured `prompts.json` at a stable gh-pages URL. These are not negotiable — the Flycut app's sync logic depends on them.

**Must have (v1 launch):**
- Markdown prompt files with YAML frontmatter (`title`, `version`) — source of truth
- Category directories (`coding/`, `writing/`, `analysis/`, `creative/`) with 23 seed prompts — immediate user value
- `catalog.yaml` with category allowlist and catalog version — enables build-time validation
- Bash build script producing sorted, validated `prompts.json` — the core artifact
- JSON Schema for output validation — catches format regressions before Flycut breaks
- GitHub Actions workflow: build + validate + deploy to gh-pages on push — makes sync URL reliable
- README with authoring guide — prerequisite for external contributions
- Unique stable IDs + duplicate detection — Flycut upsert-by-id requires this
- Per-prompt integer versioning — Flycut version-gated upsert requires this
- Template variable pass-through (`{{clipboard}}`, `{{variable}}`) — verbatim extraction

**Should have (v1.x after validation):**
- PR template for prompt contributions — add when first external PR reveals gaps
- CI version-bump check — add when first missed version bump happens in a PR
- CHANGELOG format — add when catalog version is referenced by users

**Defer (v2+):**
- Multi-catalog / namespace support — requires Flycut app changes
- Prompt deprecation convention — only needed when content churn is high
- Web browse interface — Flycut app is the browse UI; this repo is a distribution mechanism

**Explicit anti-features (reject on sight):**
- Node.js or Python build pipeline — violates zero-dependency constraint
- Automated prompt quality scoring — API costs, non-determinism in CI
- Prompt deletion propagating to Flycut — sync protocol has no deletion; app changes required

### Architecture Approach

The architecture follows a clean separation of source, build, and distribution layers across two git branches. Source files live on `main`; the validated build artifact (`prompts.json`) is deployed to `gh-pages`. Flycut syncs from `gh-pages` only — never from `main`. The `prompts.json` committed to `main` is a review artifact, not the sync endpoint. This two-branch model prevents partial or broken builds from reaching sync clients.

The build script implements four sequential operations: (1) find and sort all `.md` files, (2) parse frontmatter and extract body per file, (3) validate IDs, titles, versions, and categories, (4) assemble the final JSON with catalog-level version wrapper. Category is derived from the containing directory name — convention over configuration. ID is derived from filename — stable and unique across all categories, not just within one directory.

**Major components:**
1. `prompts/**/*.md` — Source truth; never modified by build; YAML frontmatter + Markdown body
2. `catalog.yaml` — Category allowlist and catalog version; single authority on legal categories
3. `scripts/build.sh` — Orchestrates parse → validate → assemble → write; exits non-zero on any error
4. `schema/prompt.schema.json` — JSON contract between repo and Flycut; validated by CI after build
5. `.github/workflows/build-prompts.yml` — CI/CD pipeline; paths-filtered trigger; deploys `_deploy/` to gh-pages
6. `prompts.json` (gh-pages) — Live distribution artifact at stable HTTPS URL; consumed by Flycut

**Build order (minimize rework):** `catalog.yaml` → `scripts/build.sh` → `schema/prompt.schema.json` → `prompts/**/*.md` → `.github/workflows/build-prompts.yml` → `README.md`

### Critical Pitfalls

1. **macOS bash 3.2 `declare -A` failure** — The build script pseudocode uses associative arrays for duplicate ID detection. macOS ships bash 3.2 which does not support `declare -A`. Replace with POSIX-compatible string + `grep -w` membership test or `sort | uniq -d` post-collection approach. Test on macOS before declaring build script complete.

2. **Version not bumped on content edit — silent sync failure** — Flycut's update check is `remote.version > local.version`. An unchanged version means users never receive content updates, with no error. README documentation is insufficient; CI must enforce this with a `git diff HEAD~1` comparison step that fails the build when content changes without a version increment.

3. **Leading newline in extracted prompt body** — The `awk` body extractor captures the blank line authors place after the closing `---` delimiter. The `content` field then starts with `\n`, which Flycut pastes verbatim. Fix by stripping leading blank lines from extracted body before JSON-encoding. Verify with `jq '.prompts[] | select(.content | startswith("\n"))' prompts.json`.

4. **Quoted YAML title with colon silently truncated** — `sed`/`grep` parsers using `cut -d: -f2` or naive patterns drop everything after the first `:` in a title. Use a sed pattern that strips only the `title: ` prefix (not splitting on colons), then pass the full value to `jq --arg` for safe encoding. Include a test fixture with a colon in its title.

5. **Prompt file rename orphans all synced users permanently** — Renaming `fix-bug.md` to `debug-code.md` changes the `id`. Every user who synced `fix-bug` retains the old entry forever; the new `debug-code` appears as a new prompt. There is no tombstone mechanism. Enforce via CONTRIBUTING.md rule: filenames are immutable once deployed. Add a CI alert (not block) when an existing ID disappears from the output.

6. **`peaceiris/actions-gh-pages` `include_files` parameter does not exist** — Using this parameter deploys the entire repo to gh-pages. Always use a `_deploy/` staging directory with `publish_dir: ./_deploy`.

7. **GitHub Pages not enabled in repository settings** — CI can create and push the `gh-pages` branch without the `github.io` URL being active. The `raw.githubusercontent.com` URL works immediately; the `github.io` domain requires manual activation in Settings → Pages. Document this as a one-time post-deploy step.

## Implications for Roadmap

Research establishes clear dependencies that dictate phase ordering. The build script cannot run without `catalog.yaml`. CI cannot validate without the build script. The 23 seed prompts cannot be processed without a working build script. Deployment cannot be verified without content. This creates a strict linear dependency chain.

### Phase 1: Repository Skeleton and Build Foundation

**Rationale:** Everything else depends on the build pipeline working. Establish the directory structure, `catalog.yaml`, and a working `build.sh` before adding any content. A placeholder `.md` file is sufficient to verify the pipeline end-to-end.
**Delivers:** Runnable `scripts/build.sh` producing valid `prompts.json` from a single test prompt; `catalog.yaml` with category allowlist; `schema/prompt.schema.json`; `.gitignore`
**Addresses features:** Build script, catalog.yaml, JSON Schema, category validation, unique ID enforcement
**Avoids pitfalls:** macOS bash 3.2 `declare -A` (fix during initial build script authoring, not after all 23 prompts are added); leading newline in content (test immediately with placeholder); colon-in-title parsing (include test fixture in placeholder set)

### Phase 2: Seed Prompt Catalog

**Rationale:** Build pipeline is verified working. Add all 23 seed prompts across four categories. This is the content value that makes the repository immediately useful to Flycut users on first sync.
**Delivers:** 23 prompt files (`coding/8`, `writing/6`, `analysis/5`, `creative/4`); `prompts.json` committed to main for review visibility
**Addresses features:** Category directories with seed content, per-prompt integer versioning, template variable pass-through, stable filename-as-ID convention
**Avoids pitfalls:** Filename immutability (establish names carefully now — renaming post-deployment orphans users); duplicate ID detection runs on each build

### Phase 3: CI/CD Pipeline and gh-pages Deployment

**Rationale:** Content exists and builds locally. Automate the pipeline and establish the canonical sync URL. This is what makes the repository a reliable distribution endpoint rather than a manual process.
**Delivers:** `.github/workflows/build-prompts.yml` with paths filter, build + validate + deploy steps; `_deploy/` staging directory pattern; `index.html` on gh-pages; confirmed `raw.githubusercontent.com` sync URL
**Addresses features:** GitHub Actions CI workflow, gh-pages deployment, JSON validation in CI
**Avoids pitfalls:** `include_files` non-existence (use `_deploy/` from the start); `contents: write` permissions (explicit declaration); gh-pages Settings activation (document as post-deploy step); full-repo publish to gh-pages (staging directory prevents this)

### Phase 4: Contributor Documentation and Guard Rails

**Rationale:** Repository is functional. Make it contribution-safe. The version-bump enforcement and filename immutability rules protect sync integrity at scale.
**Delivers:** `README.md` with authoring guide, frontmatter format reference, variable syntax, and build instructions; `CONTRIBUTING.md` with filename immutability rule; CI version-bump check step; PR template
**Addresses features:** README, PR template, CI version-bump warning
**Avoids pitfalls:** Version-not-bumped silent failure (CI enforcement is the goal of this phase); prompt rename orphaning (CONTRIBUTING.md rule established here); catalog.yaml version bump checklist

### Phase Ordering Rationale

- Phase 1 before Phase 2: Build script must be verified before adding 23 prompts — catching `declare -A` or content extraction bugs with 1 file is far less painful than debugging with 23.
- Phase 2 before Phase 3: Having real content in CI validates the full pipeline end-to-end, including edge cases like prompts with colons in titles or backslashes in content.
- Phase 3 before Phase 4: Documentation references the sync URL and CI workflow — both must exist before README can be accurate.
- Phase 4 is last: Guard rails and contributor docs assume the core system works. Adding these too early creates friction during initial development.

### Research Flags

Phases with well-documented patterns (skip `research-phase`):
- **Phase 1:** Build script implementation is a well-understood bash scripting problem. Architecture research provides the exact build order, data flow, and POSIX-compatible patterns. Pitfalls research provides specific fixes for all known failure modes.
- **Phase 3:** GitHub Actions workflow is specified down to the exact YAML in STACK.md and ARCHITECTURE.md. No additional research needed.

Phases that may benefit from targeted validation during planning:
- **Phase 2 (prompt content):** The 23 prompts themselves need to be written. Research does not cover content quality or which specific prompts to include — this is an editorial decision outside the technical scope of this research. Recommend listing the 23 prompts explicitly in requirements before implementation.
- **Phase 4 (CI version-bump check):** The git-diff approach for version bump enforcement uses `git show HEAD~1` which has edge cases on the initial commit (no parent) and squash-merged PRs. The CI snippet in PITFALLS.md handles the common case; verify the `2>/dev/null || echo 0` fallback is sufficient for the first-ever build.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Primary source is project design document + verified against peaceiris/actions-gh-pages repo and GitHub Actions docs; all tool behaviors confirmed |
| Features | HIGH | Project spec (PROJECT.md) is the primary source; Flycut sync contract is well-understood; competitor analysis confirms table-stakes choices |
| Architecture | HIGH | Design document (`dist/PROMPTS-REPO-DESIGN.md`) provides complete authoritative specification; architecture has no ambiguous areas |
| Pitfalls | HIGH | Pitfalls drawn from design pseudocode analysis + confirmed community sources; macOS bash 3.2, jq escaping, and Pages setup all verified against real issue threads |

**Overall confidence:** HIGH

### Gaps to Address

- **Prompt content list:** Research does not specify the 23 individual prompts by name, title, or content. Requirements phase must enumerate all 23 prompts explicitly — what they are called, what categories they belong to, and what their initial content will be. This is editorial, not technical.
- **Flycut sync URL format:** Research confirms `raw.githubusercontent.com/[owner]/[repo]/gh-pages/prompts.json` works without GitHub Pages enabled. However, if the canonical URL is intended to be the `github.io` domain, the one-time Pages Settings activation must be part of the deployment checklist. Confirm which URL format Flycut expects before deployment.
- **Catalog version semantics:** `catalog.yaml` has a `catalog_version` field, but research does not specify the initial value or when/how it is incremented relative to prompt versions. Clarify during requirements: is `catalog_version` a monotonic integer, semver, or date-based? Who bumps it?
- **`prompts.json` committed to main:** Research identifies this as a deliberate design decision (PR review visibility), but it creates a perpetual uncommitted-file situation in contributors' working trees after a local build. Document the expected contributor workflow clearly in README.

## Sources

### Primary (HIGH confidence)

- `dist/PROMPTS-REPO-DESIGN.md` — Complete system specification; primary source for architecture, CI workflow, and build script design
- `.planning/PROJECT.md` — Project constraints, zero-dependency requirement, out-of-scope boundaries
- [peaceiris/actions-gh-pages GitHub repo](https://github.com/peaceiris/actions-gh-pages) — Action inputs, `include_files` non-existence confirmed, `publish_dir` behavior
- [GitHub Actions workflow syntax docs](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-pages) — `paths:` filter, `permissions`, `workflow_dispatch` behavior
- [GitHub Pages — Configuring a publishing source](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site) — Manual Pages activation requirement
- [Build a JSON String With Bash Variables — Baeldung](https://www.baeldung.com/linux/bash-variables-create-json-string) — `jq --arg`/`--argjson` as safe escaping pattern

### Secondary (MEDIUM confidence)

- [Hosting a JSON API on GitHub Pages — Medium](https://victorscholz.medium.com/hosting-a-json-api-on-github-pages-47b402f72603) — Static JSON on gh-pages pattern
- [YAML frontmatter format convention — jlevy/frontmatter-format](https://github.com/jlevy/frontmatter-format) — YAML as metadata standard
- [Prompt Versioning Best Practices — Braintrust](https://www.braintrust.dev/articles/what-is-prompt-versioning) — Versioning patterns
- [github/awesome-copilot — DeepWiki](https://deepwiki.com/github/awesome-copilot/1.2-repository-structure-and-content-types) — Content type and frontmatter patterns
- [Associative array error on macOS for bash `declare -A`](https://dipeshmajumdar.medium.com/associative-array-error-on-macos-for-bash-declare-a-invalid-option-16466534e145) — macOS bash 3.2 limitation
- [Escaping Characters in YAML Front Matter](https://inspirnathan.com/posts/134-escape-characters-in-yaml-frontmatter/) — Colon-in-title parsing issues
- [BSD/macOS sed vs GNU sed](https://riptutorial.com/sed/topic/9436/bsd-macos-sed-vs--gnu-sed-vs--the-posix-sed-specification) — sed portability differences

### Tertiary (LOW confidence)

- [thibaultyou/prompt-library — GitHub](https://github.com/thibaultyou/prompt-library) — Competitor feature inspection (fetched via search summary)
- [Understanding Prompt Management in GitHub Repositories — arXiv](https://arxiv.org/html/2509.12421v1) — Best practices survey

---
*Research completed: 2026-03-11*
*Ready for roadmap: yes*
