# Requirements: Flycut Prompts Repository

**Defined:** 2026-03-11
**Core Value:** Flycut users receive a curated, versioned set of useful prompts via auto-sync, with a contributor-friendly Markdown authoring workflow and reliable build pipeline.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Repository Structure

- [ ] **REPO-01**: Repository has `prompts/` directory with `coding/`, `writing/`, `analysis/`, `creative/` subdirectories
- [ ] **REPO-02**: `catalog.yaml` defines catalog version and valid categories list
- [ ] **REPO-03**: `.gitignore` excludes .DS_Store, swap files, and node_modules

### Prompt Authoring

- [ ] **PRMT-01**: Each prompt is a Markdown file with YAML frontmatter containing required `title` (string) and `version` (integer) fields
- [ ] **PRMT-02**: Optional frontmatter fields supported: `category` (override), `description`, `tags`, `variables`
- [ ] **PRMT-03**: Prompt `id` derived from filename (without `.md`), lowercase kebab-case, unique across all categories
- [ ] **PRMT-04**: Category resolved from parent directory name, overridable via frontmatter `category` field
- [ ] **PRMT-05**: Prompt content supports `{{clipboard}}` and custom `{{variable}}` template placeholders

### Seed Prompts

- [ ] **SEED-01**: 8 coding prompts: code-review-swift, explain-code, fix-bug, write-tests, refactor-code, add-error-handling, convert-to-async, optimize-performance
- [ ] **SEED-02**: 6 writing prompts: summarize-text, rewrite-formal, fix-grammar, simplify-language, write-email-reply, expand-bullet-points
- [ ] **SEED-03**: 5 analysis prompts: analyze-data, compare-options, extract-action-items, identify-risks, create-summary-table
- [ ] **SEED-04**: 4 creative prompts: brainstorm, write-story, generate-names, create-outline
- [ ] **SEED-05**: All 23 prompts have version 1, meaningful content, and correct frontmatter

### Build Pipeline

- [ ] **BILD-01**: `scripts/build.sh` compiles all `.md` files under `prompts/` into `prompts.json` at repo root
- [ ] **BILD-02**: Build script is zero-dependency (bash + sed/awk/grep + jq for JSON assembly)
- [ ] **BILD-03**: Build script reads catalog version from `catalog.yaml`
- [ ] **BILD-04**: Build script derives `id` from filename and `category` from directory (with frontmatter override)
- [ ] **BILD-05**: Build script fails with error on duplicate IDs across categories
- [ ] **BILD-06**: Build script fails with error on missing required frontmatter fields (title, version)
- [ ] **BILD-07**: Build script fails with error on invalid category (not in catalog.yaml list)
- [ ] **BILD-08**: Build script outputs prompts sorted alphabetically by `id` for deterministic builds
- [ ] **BILD-09**: Build script properly JSON-escapes multiline Markdown content (newlines, quotes, backslashes)
- [ ] **BILD-10**: Build script works on macOS bash 3.2 (no `declare -A` or bash 4+ features)
- [ ] **BILD-11**: Build script strips leading blank lines from extracted prompt body content

### Output Format

- [ ] **JSON-01**: `prompts.json` matches Flycut's `PromptCatalog` structure: `{version: Int, prompts: [PromptDTO]}`
- [ ] **JSON-02**: Each prompt object has exactly 5 fields: `id`, `title`, `category`, `version`, `content` (no extras)
- [ ] **JSON-03**: `prompts.json` validates against `schema/prompt.schema.json`

### Validation Schema

- [ ] **SCHM-01**: `schema/prompt.schema.json` validates the complete prompts.json structure with JSON Schema draft 2020-12
- [ ] **SCHM-02**: Schema enforces id pattern (`^[a-z0-9-]+$`), valid category enum, minimum version 1, non-empty strings

### CI/CD Pipeline

- [ ] **CICD-01**: GitHub Actions workflow triggers on push to `main` when `prompts/**`, `catalog.yaml`, or `scripts/build.sh` change
- [ ] **CICD-02**: Workflow supports manual trigger via `workflow_dispatch`
- [ ] **CICD-03**: Workflow runs `build.sh`, validates output JSON, and deploys to `gh-pages` branch
- [ ] **CICD-04**: Deployment uses `_deploy/` staging directory containing only `prompts.json` and a simple `index.html`
- [ ] **CICD-05**: Workflow validates no duplicate IDs, all required fields present, and valid JSON structure

### Documentation

- [ ] **DOCS-01**: README explains what the repo is, how to use prompts in Flycut, and the sync URL
- [ ] **DOCS-02**: README documents how to add a prompt (file format, frontmatter fields, PR workflow)
- [ ] **DOCS-03**: README documents how to update a prompt (content change + version bump requirement)
- [ ] **DOCS-04**: README explains template variables (`{{clipboard}}` and custom variables)
- [ ] **DOCS-05**: README includes table of all prompts grouped by category
- [ ] **DOCS-06**: README documents versioning rules prominently (never decrease, bump on update, catalog version)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Contributor Experience

- **CTRB-01**: CONTRIBUTING.md with detailed contribution guidelines
- **CTRB-02**: PR template with version-bump checklist
- **CTRB-03**: CI version-bump enforcement check on PRs that modify prompt files

### Enhanced Validation

- **EVAL-01**: PR-time validation workflow (build + check without deploying)
- **EVAL-02**: CHANGELOG generation from git history

## Out of Scope

| Feature | Reason |
|---------|--------|
| Web UI for prompt browsing | Repo is a build pipeline, not a web app |
| Node.js/Python build tools | Zero-dependency bash constraint |
| Custom YAML parser library | sed/awk/grep sufficient for simple frontmatter |
| Prompt analytics/usage tracking | Not part of repo responsibility |
| User authentication | Public repo, public JSON endpoint |
| Flycut app code changes | Separate codebase |
| Database or API backend | Static JSON file served via GitHub Pages |
| Automated prompt generation via AI | Content is human-curated |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| REPO-01 | Phase 1 | Pending |
| REPO-02 | Phase 1 | Pending |
| REPO-03 | Phase 1 | Pending |
| PRMT-01 | Phase 1 | Pending |
| PRMT-02 | Phase 1 | Pending |
| PRMT-03 | Phase 1 | Pending |
| PRMT-04 | Phase 1 | Pending |
| PRMT-05 | Phase 1 | Pending |
| BILD-01 | Phase 1 | Pending |
| BILD-02 | Phase 1 | Pending |
| BILD-03 | Phase 1 | Pending |
| BILD-04 | Phase 1 | Pending |
| BILD-05 | Phase 1 | Pending |
| BILD-06 | Phase 1 | Pending |
| BILD-07 | Phase 1 | Pending |
| BILD-08 | Phase 1 | Pending |
| BILD-09 | Phase 1 | Pending |
| BILD-10 | Phase 1 | Pending |
| BILD-11 | Phase 1 | Pending |
| JSON-01 | Phase 1 | Pending |
| JSON-02 | Phase 1 | Pending |
| JSON-03 | Phase 1 | Pending |
| SCHM-01 | Phase 1 | Pending |
| SCHM-02 | Phase 1 | Pending |
| SEED-01 | Phase 2 | Pending |
| SEED-02 | Phase 2 | Pending |
| SEED-03 | Phase 2 | Pending |
| SEED-04 | Phase 2 | Pending |
| SEED-05 | Phase 2 | Pending |
| CICD-01 | Phase 3 | Pending |
| CICD-02 | Phase 3 | Pending |
| CICD-03 | Phase 3 | Pending |
| CICD-04 | Phase 3 | Pending |
| CICD-05 | Phase 3 | Pending |
| DOCS-01 | Phase 4 | Pending |
| DOCS-02 | Phase 4 | Pending |
| DOCS-03 | Phase 4 | Pending |
| DOCS-04 | Phase 4 | Pending |
| DOCS-05 | Phase 4 | Pending |
| DOCS-06 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 40 total
- Mapped to phases: 40
- Unmapped: 0

---
*Requirements defined: 2026-03-11*
*Last updated: 2026-03-11 after roadmap creation*
