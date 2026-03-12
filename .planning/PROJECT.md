# Flycut Prompts Repository

## What This Is

A prompts repository for Flycut's Prompt Library feature. Prompts are authored as individual Markdown files with YAML frontmatter, built into a single `prompts.json` via a bash build script, and deployed to GitHub Pages (`gh-pages` branch) for Flycut to sync from. The repository serves as the canonical source of prompts that Flycut users receive through the app's sync mechanism.

## Core Value

Flycut users receive a curated, versioned set of useful prompts that auto-sync to their app, with a contributor-friendly authoring workflow (Markdown files) and a reliable build pipeline that produces the exact JSON format Flycut expects.

## Requirements

### Validated

- ✓ Design document specifies complete repository structure — existing
- ✓ Output JSON format defined (PromptCatalog/PromptDTO Swift types) — existing
- ✓ Sync behavior documented (upsert by id, version-gated, user customization protection) — existing
- ✓ Repository structure with prompts organized by category directories — Phase 1
- ✓ YAML frontmatter format with title, version, category fields — Phase 1
- ✓ Build script (bash, zero dependencies) that compiles .md files to prompts.json — Phase 1
- ✓ catalog.yaml for catalog-level metadata and category validation — Phase 1
- ✓ JSON Schema for validating built output — Phase 1
- ✓ Template variable support ({{clipboard}} and custom variables) — Phase 1
- ✓ .gitignore for standard exclusions — Phase 1
- ✓ 23 prompt Markdown files across 4 categories (coding/8, writing/6, analysis/5, creative/4) — Phase 2

### Active

- [ ] GitHub Action workflow for automatic build and gh-pages deployment
- [ ] README with contributor documentation

### Out of Scope

- Flycut app changes — this project is the prompts repo only
- Custom YAML parser — build script uses sed/awk/grep
- Node.js or Python dependencies — zero-dependency bash build
- Database or API backend — static JSON file served via GitHub Pages
- User authentication — public repository, public JSON endpoint
- Prompt analytics or usage tracking — not part of repo scope

## Context

- **Target platform:** GitHub repository (`generalarcade/flycut-prompts`)
- **Sync URL:** `https://raw.githubusercontent.com/generalarcade/flycut-prompts/gh-pages/prompts.json`
- **Flycut sync behavior:** Upsert by `id`, version-gated updates (`remote.version > local.version`), user customization protection (skips customized prompts), no deletion (removing from JSON doesn't delete locally)
- **Template variables:** `{{clipboard}}` is built-in; other `{{variable}}` placeholders looked up from user-defined key-value pairs in Flycut Settings
- **Valid categories:** coding, writing, analysis, creative
- **Filename rules:** Lowercase kebab-case, a-z/0-9/- only, unique across all categories
- **Existing reference:** `dist/PROMPTS-REPO-DESIGN.md` contains the complete specification

## Constraints

- **Zero dependencies:** Build script must work with standard macOS/Ubuntu tools (bash, sed, awk, jq for validation only)
- **Deterministic builds:** Prompts sorted alphabetically by id in output JSON
- **Strict validation:** Build fails on duplicate IDs, missing required fields, or invalid categories
- **JSON compatibility:** Output must match Flycut's `PromptCatalog`/`PromptDTO` Swift Decodable types exactly
- **Version discipline:** Per-prompt versions are monotonically increasing integers; catalog version bumps on every release

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Markdown + YAML frontmatter for prompts | Human-readable, git-friendly, easy to contribute | Validated Phase 1 |
| Bash build script (no Node/Python) | Zero dependencies, works on CI without setup | Validated Phase 1 |
| gh-pages deployment via GitHub Action | Free hosting, automatic on push to main | — Pending |
| Category derived from directory name | Convention over configuration, less frontmatter boilerplate | Validated Phase 1 |
| prompts.json committed to main branch | Useful for PR review, overwritten by CI on each build | Validated Phase 1 |
| Numbered bold-label prompt body format | Visual consistency across all categories | Validated Phase 2 |
| catalog.yaml version bump on content release | Signals content-significant changes to sync clients | Validated Phase 2 |

---
*Last updated: 2026-03-12 after Phase 2*
