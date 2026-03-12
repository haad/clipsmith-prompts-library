# Phase 3: CI/CD and Deployment - Context

**Gathered:** 2026-03-12
**Status:** Ready for planning

<domain>
## Phase Boundary

GitHub Actions workflow that automatically builds, validates, and deploys prompts.json to the gh-pages branch on every qualifying push to main. Includes a static landing page (index.html) that reads prompts.json client-side and presents a browsable prompt catalog. The workflow also supports manual dispatch.

</domain>

<decisions>
## Implementation Decisions

### Deployment staging
- `_deploy/` staging directory pattern: CI copies prompts.json + index.html into `_deploy/`, then deploys that directory to gh-pages
- index.html is a **static file at repo root** that fetches prompts.json client-side via JavaScript — not regenerated on each build
- CI copies the static index.html + freshly-built prompts.json into the deploy staging area

### Landing page design
- **Category filter tabs** with counts: All (23) | Coding (8) | Writing (6) | Analysis (5) | Creative (4) — click to filter the grid
- **Search/filter bar** — text input that filters prompts by title/description as you type
- **Prompt cards with copy button** — each prompt shown as card with title, category badge, description, and full content visible (not collapsed). Copy-to-clipboard button on each card
- **Dark/light mode toggle** — theme switcher that respects system preference
- **Full prompt content visible** on cards — no expand/collapse, all content shown for scanability
- **Nicely styled** — designed as a foundation to build upon later (search capability, richer browsing in future phases)
- **Inspiration:** prompts.chat (card grid, copy buttons, modern aesthetic) and block.github.io/goose/prompt-library/ (category tabs, clean documentation style)

### Build validation
- Schema validation (prompts.json against schema/prompt.schema.json via jq) added **to build.sh itself** — works locally and in CI without duplication
- CI runs build.sh as a single step; build.sh handles both compilation and validation
- Validation failure causes build.sh to exit non-zero, failing the CI step

### Landing page generation
- index.html is a **static file checked into the repo root** — not generated during build
- It dynamically loads prompts.json via fetch() at runtime and renders the UI client-side
- Zero build-time dependencies for the page — pure HTML/CSS/JS

### Claude's Discretion
- Workflow trigger path filters (which file changes trigger the build)
- gh-pages branch management strategy (force-push vs incremental)
- Exact CSS styling approach and card layout details
- GitHub Actions runner version and action versions
- Error notification approach (default GitHub behavior)

</decisions>

<specifics>
## Specific Ideas

- Landing page inspired by prompts.chat (card-based grid, copy-to-clipboard, modern styling) and Goose prompt library (category tab filtering, clean documentation aesthetic)
- Page should feel polished enough to be a good foundation — "we can later build around it"
- Future additions like search within prompt content and expanded browsing are planned but not for this phase

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `scripts/build.sh`: Existing build pipeline — CI workflow calls this directly. Needs schema validation step added.
- `schema/prompt.schema.json`: JSON Schema for validating prompts.json structure — will be used by build.sh validation step
- `catalog.yaml`: Contains catalog version and valid categories list — referenced by build.sh

### Established Patterns
- Zero-dependency bash build: All tooling must work with bash 3.2+ and standard macOS/Ubuntu tools
- `jq` is the only external dependency (already required by build.sh)
- Deterministic builds: prompts sorted alphabetically by id

### Integration Points
- `prompts.json` at repo root: Build output that both index.html reads (client-side) and gh-pages serves
- No `.github/` directory exists yet — workflow file is entirely new
- GitHub Pages must be manually enabled in repository Settings after first gh-pages push

</code_context>

<deferred>
## Deferred Ideas

- Search within prompt content (full-text search) — future enhancement to landing page
- Prompt content preview/expand interaction — future UX improvement
- Richer browsing capabilities (tags, filtering by multiple criteria) — future phase
- PR-time validation workflow (build without deploying) — v2 requirement (EVAL-01)
- CHANGELOG generation — v2 requirement (EVAL-02)

</deferred>

---

*Phase: 03-ci-cd-and-deployment*
*Context gathered: 2026-03-12*
