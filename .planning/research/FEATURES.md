# Feature Research

**Domain:** Curated prompt library repository with static JSON sync for mobile app
**Researched:** 2026-03-11
**Confidence:** HIGH (project spec is detailed; ecosystem patterns are well-understood)

## Feature Landscape

### Table Stakes (Users Expect These)

These are non-negotiable. Missing any of these makes the repo non-functional as a
sync-based prompt distribution system.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Markdown prompt files with YAML frontmatter | Every comparable prompt repo uses this format; human-readable, git-diffable, easy to contribute | LOW | `title`, `version`, `category` are minimum required fields per PROJECT.md spec |
| Category-based directory organization | Standard convention in all prompt library repos; enables discoverability and validation | LOW | Directories `coding/`, `writing/`, `analysis/`, `creative/` — category derived from dirname |
| Unique, stable prompt IDs | Required for upsert-based sync — ID is the primary key Flycut uses to match remote vs local prompts | LOW | Derived from filename; kebab-case, a-z/0-9/- only; unique across all categories |
| Build script that produces prompts.json | The entire sync chain depends on a well-formed JSON output that Flycut can decode via `PromptCatalog`/`PromptDTO` Swift types | MEDIUM | Bash, zero dependencies; sed/awk for YAML parsing; jq for validation only |
| catalog.yaml for catalog-level metadata | Version tracking at the catalog level (separate from per-prompt versions); category allowlist for build-time validation | LOW | Enforces valid category names; holds `catalog_version` bumped on each release |
| JSON Schema for output validation | Build pipelines need machine-verifiable contracts; prevents silent regressions in JSON shape when Flycut types evolve | LOW | Schema validates `PromptCatalog`/`PromptDTO` structure; run by CI after build |
| GitHub Actions CI workflow | Prompt repos consumed by apps require automated, reproducible builds — manual builds are unreliable | MEDIUM | Triggers on push to main; runs build + validate + deploy to gh-pages |
| gh-pages deployment of prompts.json | Flycut syncs from a stable, public URL (`raw.githubusercontent.com` or gh-pages); GitHub Pages is zero-cost and fits the static-JSON pattern | LOW | Deploys built `prompts.json` to `gh-pages` branch; URL is the canonical sync endpoint |
| Template variable support (`{{variable}}`) | Flycut's sync behavior and UI depend on `{{clipboard}}` and user-defined `{{variable}}` patterns; prompts without this are less useful | LOW | `{{clipboard}}` is built-in; other `{{variable}}` tokens resolved from Flycut Settings key-value pairs |
| README with contributor documentation | Any public repo receiving contributions needs clear authoring instructions; without it, contributors produce malformed prompts | LOW | Documents frontmatter format, filename rules, category rules, variable syntax, and build/test instructions |
| .gitignore for standard exclusions | Table stakes for any git repo | LOW | Standard macOS, Linux, editor exclusions; no build artifacts in source |

### Differentiators (Competitive Advantage)

Features that are not universally present in prompt library repos but provide meaningful
value for this specific use case: a sync-based, app-integrated catalog.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Strict build-time validation with explicit failure modes | Most prompt repos have loose or no validation; strict CI that fails on duplicate IDs, missing fields, or invalid categories protects Flycut from corrupt sync payloads | MEDIUM | Build script exits non-zero on: duplicate prompt IDs across categories, missing required frontmatter fields, invalid category names, malformed JSON output |
| Deterministic, alphabetically-sorted JSON output | Reproducible builds reduce noise in diffs and make PR reviews useful (reviewers see actual content changes, not sort-order churn) | LOW | Prompts sorted by `id` in output; same input always produces identical JSON |
| Per-prompt monotonic integer versioning | Enables Flycut's version-gated upsert: `remote.version > local.version` triggers update, preserving user customizations | LOW | Version is an integer in frontmatter; must be manually incremented on each edit; CI can warn if version was not bumped on modified files |
| Prompt body preserved verbatim (no reformatting) | Some build tools normalize whitespace or line endings; verbatim extraction preserves intentional formatting in prompts (lists, code blocks, structured instructions) | LOW | Build script extracts everything after frontmatter delimiter as-is |
| prompts.json committed to main branch alongside source | Enables PR reviewers to see the compiled output diff alongside source changes — catches build script bugs and JSON shape regressions without running build locally | LOW | CI overwrites on each merge; the committed file is for review, not for consumption |
| Category-as-directory convention (no redundant frontmatter) | Reduces authoring friction; category is inferred from path, not declared in frontmatter — impossible to create a category mismatch | LOW | Convention-over-configuration; new categories require only a new directory and catalog.yaml addition |
| 23-prompt seed catalog across 4 domains | Ships with enough content to be immediately useful to Flycut users; comparable repos often launch empty or with only 2-3 prompts | MEDIUM | coding/8, writing/6, analysis/5, creative/4 — meaningful coverage without padding |
| Issue/PR templates tuned for prompt contributions | Generic repo templates don't ask for prompt-specific information (what model was it tested with, what task does it solve); prompt-specific templates improve contribution quality | LOW | PR template includes: prompt purpose, tested use cases, variable documentation, category justification |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem like obvious additions but are explicitly out of scope for this project.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Web UI / browse interface | "Users want to discover prompts visually" | Adds Node.js/frontend build complexity, hosting concerns, and maintenance surface; this repo is a *distribution mechanism*, not a discovery product — Flycut's UI is the browse interface | Flycut app provides the browse/search UX; the repo README documents categories and lists all prompts |
| Node.js or Python build pipeline | "Proper parsers, better YAML support, richer tooling" | Violates zero-dependency constraint; adds setup steps on CI and contributor machines; makes the build fragile across OS versions | bash + sed/awk + jq; tools available on macOS and Ubuntu without any install step |
| Custom YAML parser | "More robust frontmatter extraction" | Building a correct YAML parser in bash is error-prone; edge cases (multiline values, special chars) cause build failures | Use deterministic, constrained frontmatter structure (simple key: value lines only); document the constraints in CONTRIBUTING.md |
| Prompt analytics / usage tracking | "Learn which prompts are popular" | Requires backend infrastructure, privacy considerations, user consent; entirely outside the repo's responsibility boundary | Flycut app can track usage locally; analytics belong in the app, not the distribution repo |
| User authentication or private prompts | "Enterprise teams want private catalogs" | Public repo + public JSON endpoint; adding auth requires an API backend — no longer a static site | Users who want private catalogs fork the repo; the design is intentionally forkable |
| Automated prompt quality scoring | "AI-generated prompts should be validated for quality" | Subjectivity, API costs, non-determinism in CI; a low-quality prompt that passes CI is better than a CI step that fails intermittently | Quality is enforced through human PR review; CONTRIBUTING.md sets expectations |
| Prompt deletion propagating to Flycut | "Removing a prompt from the repo should remove it from the app" | Flycut's sync behavior explicitly does NOT delete local prompts when removed from JSON; building deletion support would require app changes, which are out of scope | Document the no-deletion behavior clearly; version prompts to `0` as a convention to signal deprecation |
| Git tags as version source | "Use git tags instead of per-prompt version integers" | Git tags are catalog-level, not prompt-level; Flycut's upsert needs per-prompt version comparison | Keep per-prompt integer versions in frontmatter; bump catalog version in catalog.yaml independently |

## Feature Dependencies

```
[catalog.yaml]
    └──required by──> [Build Script]
                          └──produces──> [prompts.json]
                                             └──consumed by──> [GitHub Actions CI]
                                                                   └──deploys to──> [gh-pages / sync URL]

[Prompt Markdown Files]
    └──required by──> [Build Script]

[JSON Schema]
    └──validates──> [prompts.json]
    └──run by──> [GitHub Actions CI]

[Per-prompt integer versions]
    └──enables──> [Flycut version-gated upsert]

[Unique stable IDs]
    └──enables──> [Flycut upsert-by-id sync]
    └──enforced by──> [Build Script duplicate detection]

[Template variables ({{var}})]
    └──resolved by──> [Flycut at runtime — not by build script]

[prompts.json on main branch]
    └──enhances──> [PR review experience]
    └──overwritten by──> [GitHub Actions CI on merge]
```

### Dependency Notes

- **Build Script requires catalog.yaml:** Category allowlist is in catalog.yaml; without it, the build cannot validate that a prompt's directory name is a known category.
- **GitHub Actions CI requires Build Script + JSON Schema:** CI orchestrates the full pipeline: run build script, validate output against JSON Schema, deploy to gh-pages. The schema and script must exist before CI is useful.
- **Flycut version-gated upsert requires per-prompt integer versions:** The sync logic `remote.version > local.version` only works if versions are monotonically increasing integers — not hashes, dates, or semver strings.
- **Flycut upsert-by-id requires unique stable IDs:** If a prompt is renamed (ID changes), Flycut treats it as a new prompt. ID stability is load-bearing for the sync contract.
- **Template variables are Flycut runtime, not build time:** The build script passes `{{variable}}` tokens through verbatim. Resolution happens in Flycut. Build script only validates that prompt text was extracted correctly, not that variables are defined.

## MVP Definition

### Launch With (v1)

The minimum set needed to make the sync URL functional and the catalog useful to Flycut users.

- [ ] Prompt Markdown files with YAML frontmatter (`title`, `version`, `category`) — without these, there is nothing to build
- [ ] Category directories (`coding/`, `writing/`, `analysis/`, `creative/`) with 23 seed prompts — provides immediate value to users on first sync
- [ ] catalog.yaml with category allowlist and catalog version — enables build-time category validation
- [ ] Build script (bash) that produces sorted, validated prompts.json — the core artifact
- [ ] JSON Schema for prompts.json output — catches format regressions before they break Flycut
- [ ] GitHub Actions workflow: build + validate + deploy to gh-pages on push to main — automation is what makes the sync URL reliable
- [ ] README with authoring guide — necessary for external contributors to produce valid prompts
- [ ] .gitignore — table stakes for any repository

### Add After Validation (v1.x)

Add once v1 is stable and receiving contributions.

- [ ] PR template for prompt contributions — add when first external PR arrives and reveals what information reviewers actually need
- [ ] CI check that warns if a modified prompt's version was not incremented — add when the first version-bump mistake happens in a PR
- [ ] CHANGELOG or release notes format — add when catalog version starts being referenced by Flycut users

### Future Consideration (v2+)

Defer until product-market fit for the catalog is established.

- [ ] Multi-catalog / namespace support — defer; adds architectural complexity and Flycut app changes are required
- [ ] Prompt deprecation convention (version set to 0 or `deprecated: true` frontmatter field) — defer; only needed once content churn is high enough to matter
- [ ] Automated CI check for frontmatter completeness beyond required fields (e.g., `description`, `tags`) — defer; adds friction to contributions before the catalog has momentum

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Prompt Markdown files + YAML frontmatter | HIGH | LOW | P1 |
| Category directories + 23 seed prompts | HIGH | MEDIUM | P1 |
| Bash build script → prompts.json | HIGH | MEDIUM | P1 |
| GitHub Actions CI + gh-pages deploy | HIGH | MEDIUM | P1 |
| catalog.yaml + category validation | HIGH | LOW | P1 |
| JSON Schema validation | MEDIUM | LOW | P1 |
| README / contributor docs | MEDIUM | LOW | P1 |
| Deterministic sorted output | MEDIUM | LOW | P1 |
| Per-prompt integer versioning | HIGH | LOW | P1 |
| Unique stable IDs + duplicate detection | HIGH | LOW | P1 |
| Template variable pass-through | MEDIUM | LOW | P1 |
| prompts.json committed to main | MEDIUM | LOW | P2 |
| PR template for prompt contributions | LOW | LOW | P2 |
| CI version-bump warning | MEDIUM | LOW | P2 |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | thibaultyou/prompt-library | awesome-copilot (github) | Flycut Prompts (this project) |
|---------|----------------------------|--------------------------|-------------------------------|
| File format | Markdown + separate metadata.yml | Markdown with YAML frontmatter | Markdown with YAML frontmatter |
| Build tooling | Node.js / npm CLI | GitHub Actions + marketplace actions | Bash only (zero dependencies) |
| Sync mechanism | Optional git sync via CLI | Not applicable (IDE integration) | Static JSON on gh-pages, Flycut polls |
| Versioning | AI-generated metadata, no integer versioning | No per-prompt versioning observed | Per-prompt monotonic integers (Flycut-required) |
| Category system | Directory-based categories | prompts/, agents/, instructions/ | Directory-based, validated against allowlist |
| Validation | AI analysis | Schema via GitHub Actions | JSON Schema + bash build-time checks |
| Contributor workflow | CLI-first (not git-PR-first) | PR to public repo | PR to public repo |
| Mobile app integration | No | No | Yes — purpose-built for Flycut sync |
| Template variables | Global + prompt-specific env vars | Not applicable | `{{clipboard}}` + user-defined key-value |

## Sources

- [thibaultyou/prompt-library — GitHub](https://github.com/thibaultyou/prompt-library) — feature inspection (LOW confidence, fetched via search summary)
- [github/awesome-copilot repository structure — DeepWiki](https://deepwiki.com/github/awesome-copilot/1.2-repository-structure-and-content-types) — content type patterns (MEDIUM confidence)
- [Prompt file format and guidelines — DeepWiki](https://deepwiki.com/github/awesome-copilot/5.2-prompt-file-format-and-guidelines) — YAML frontmatter conventions (MEDIUM confidence)
- [Understanding Prompt Management in GitHub Repositories — arXiv](https://arxiv.org/html/2509.12421v1) — best practices survey (MEDIUM confidence)
- [Hosting a JSON API on GitHub Pages — Medium](https://victorscholz.medium.com/hosting-a-json-api-on-github-pages-47b402f72603) — static JSON on gh-pages pattern (HIGH confidence)
- [Prompt Versioning Best Practices — Braintrust](https://www.braintrust.dev/articles/what-is-prompt-versioning) — versioning patterns (MEDIUM confidence)
- [YAML frontmatter format convention — jlevy/frontmatter-format](https://github.com/jlevy/frontmatter-format) — YAML as metadata standard (HIGH confidence)
- PROJECT.md — primary source for Flycut-specific requirements and constraints (HIGH confidence)

---
*Feature research for: Flycut Prompts Repository — curated prompt catalog with static JSON sync*
*Researched: 2026-03-11*
