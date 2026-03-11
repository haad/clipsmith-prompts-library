# Roadmap: Flycut Prompts Repository

## Overview

Four phases deliver a complete static prompt distribution pipeline. Phase 1 builds the foundation: directory structure, catalog metadata, the bash build script, and JSON schema. Phase 2 populates the catalog with all 23 seed prompts across four categories. Phase 3 automates the pipeline with GitHub Actions and establishes the live gh-pages sync URL. Phase 4 completes the contributor surface with README documentation and versioning guard rails. Each phase is a prerequisite for the next — nothing in Phase 2 can be validated without a working build script, and nothing in Phase 4 can be accurate without a live deployment.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Build Foundation** - Directory skeleton, catalog.yaml, build script, and JSON schema — the prerequisite for all other work
- [ ] **Phase 2: Seed Catalog** - All 23 prompt Markdown files across four categories with correct frontmatter and content
- [ ] **Phase 3: CI/CD and Deployment** - GitHub Actions workflow that builds, validates, and deploys prompts.json to gh-pages on push
- [ ] **Phase 4: Documentation** - README authoring guide and versioning rules that make the repository contribution-safe

## Phase Details

### Phase 1: Build Foundation
**Goal**: A contributor can run `scripts/build.sh` locally and produce a valid `prompts.json` from any `.md` file under `prompts/`
**Depends on**: Nothing (first phase)
**Requirements**: REPO-01, REPO-02, REPO-03, PRMT-01, PRMT-02, PRMT-03, PRMT-04, PRMT-05, BILD-01, BILD-02, BILD-03, BILD-04, BILD-05, BILD-06, BILD-07, BILD-08, BILD-09, BILD-10, BILD-11, JSON-01, JSON-02, JSON-03, SCHM-01, SCHM-02
**Success Criteria** (what must be TRUE):
  1. Running `bash scripts/build.sh` on a machine with only standard macOS tools produces a `prompts.json` at the repo root without errors
  2. The produced `prompts.json` validates against `schema/prompt.schema.json` using `jq`
  3. Adding a prompt `.md` file with a duplicate `id` causes the build script to exit non-zero with an error message identifying the conflict
  4. Adding a `.md` file missing `title` or `version` frontmatter causes the build script to exit non-zero with an error message
  5. A prompt with a colon in its title, backslashes in its content, or multiline content builds correctly with properly JSON-escaped output
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md — Repository scaffold: directory structure, catalog.yaml, .gitignore, test prompt, JSON schema
- [ ] 01-02-PLAN.md — Build script: scripts/build.sh with error detection, edge case validation, produces prompts.json

### Phase 2: Seed Catalog
**Goal**: Flycut users syncing for the first time receive 23 immediately useful prompts across coding, writing, analysis, and creative categories
**Depends on**: Phase 1
**Requirements**: SEED-01, SEED-02, SEED-03, SEED-04, SEED-05
**Success Criteria** (what must be TRUE):
  1. Running `bash scripts/build.sh` produces a `prompts.json` containing exactly 23 prompt objects sorted alphabetically by `id`
  2. Each of the four category directories (`coding/`, `writing/`, `analysis/`, `creative/`) contains the correct count of `.md` files (8, 6, 5, 4)
  3. Every prompt object in `prompts.json` has all five required fields (`id`, `title`, `category`, `version`, `content`) with version 1 and non-empty content
  4. At least one prompt in `prompts.json` contains a `{{clipboard}}` or `{{variable}}` template placeholder in its `content` field
**Plans**: TBD

### Phase 3: CI/CD and Deployment
**Goal**: Every push to `main` that touches prompt source files automatically builds, validates, and publishes an updated `prompts.json` to the Flycut sync URL
**Depends on**: Phase 2
**Requirements**: CICD-01, CICD-02, CICD-03, CICD-04, CICD-05
**Success Criteria** (what must be TRUE):
  1. Pushing a change to any file under `prompts/`, `catalog.yaml`, or `scripts/build.sh` triggers the GitHub Actions workflow automatically
  2. The workflow completes successfully and the `gh-pages` branch contains a `prompts.json` that is fetchable at the `raw.githubusercontent.com` sync URL
  3. Triggering the workflow manually via the GitHub Actions UI (`workflow_dispatch`) runs the same build and deploy steps
  4. A push that introduces a duplicate prompt ID or missing required field causes the workflow to fail before deploying to `gh-pages`
**Plans**: TBD

### Phase 4: Documentation
**Goal**: A contributor who has never seen the repository can add or update a prompt correctly without breaking the Flycut sync contract
**Depends on**: Phase 3
**Requirements**: DOCS-01, DOCS-02, DOCS-03, DOCS-04, DOCS-05, DOCS-06
**Success Criteria** (what must be TRUE):
  1. README explains what the repository is, what Flycut's sync URL is, and how to use prompts in the app
  2. README provides a complete step-by-step guide for adding a new prompt file with correct frontmatter
  3. README documents the version bump requirement and explains that omitting it silently prevents users from receiving updates
  4. README includes a table listing all 23 prompts grouped by category with their names and descriptions
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Build Foundation | 0/2 | Not started | - |
| 2. Seed Catalog | 0/TBD | Not started | - |
| 3. CI/CD and Deployment | 0/TBD | Not started | - |
| 4. Documentation | 0/TBD | Not started | - |
