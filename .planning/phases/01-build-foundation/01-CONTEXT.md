# Phase 1: Build Foundation - Context

**Gathered:** 2026-03-11
**Status:** Ready for planning

<domain>
## Phase Boundary

Establish the complete build infrastructure: repository structure with category directories, catalog.yaml metadata, the zero-dependency bash build script that compiles Markdown+YAML prompts into prompts.json, JSON Schema for validation, and a single test prompt to verify the pipeline end-to-end. Content authorship (23 seed prompts) is Phase 2.

</domain>

<decisions>
## Implementation Decisions

### Repository Structure
- `prompts/` directory with four subdirectories: `coding/`, `writing/`, `analysis/`, `creative/`
- `catalog.yaml` at repo root with `version: 1` and categories list
- `scripts/build.sh` at `scripts/build.sh` (executable)
- `schema/prompt.schema.json` at `schema/prompt.schema.json`
- `.gitignore` with standard exclusions (.DS_Store, *.swp, *.swo, *~, node_modules/)
- `prompts.json` committed to main (not gitignored) — useful for PR review, overwritten by CI

### Build Script Behavior
- Zero dependencies: bash + sed/awk/grep + jq (jq required for JSON assembly, not optional)
- Uses `jq -Rs '.'` for safe multiline content escaping (handles quotes, backslashes, newlines)
- No `declare -A` or bash 4+ features — must work on macOS bash 3.2
- Duplicate ID detection via `sort | uniq -d` (POSIX-compatible)
- Prompts sorted alphabetically by `id` in output JSON for deterministic builds
- Exits non-zero with descriptive error on: duplicate IDs, missing title/version, invalid category
- Category resolution: frontmatter `category:` field overrides directory-derived category
- Body extraction: everything after second `---`, with leading blank lines stripped
- Frontmatter parsing: sed to extract between first and second `---`, grep/awk for key-value pairs
- Title parsing must handle quoted values and values containing colons

### Output Format (Exact)
- Top-level: `{"version": <catalog_version>, "prompts": [...]}`
- Each prompt: `{"id": "<filename>", "title": "<title>", "category": "<category>", "version": <int>, "content": "<body>"}`
- No additional fields beyond these 5 per prompt — Flycut's Decodable types reject extras
- `additionalProperties: false` in JSON Schema enforces this

### Validation Schema
- JSON Schema draft 2020-12
- id pattern: `^[a-z0-9-]+$`
- category enum: `["coding", "writing", "analysis", "creative"]`
- version minimum: 1
- Non-empty strings for title and content

### Test Prompt (Pipeline Verification)
- Include one test prompt (e.g., `prompts/coding/code-review-swift.md`) to validate the full pipeline
- This prompt will remain in the final catalog (it's one of the 23 seed prompts)
- Must include `{{clipboard}}` placeholder to verify template variable passthrough

### Claude's Discretion
- Exact sed/awk patterns for frontmatter extraction (as long as they handle edge cases)
- Build script summary output format
- Whether to add a `--help` flag to build.sh
- Internal variable naming in build script

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `dist/PROMPTS-REPO-DESIGN.md`: Complete specification with pseudocode for build script, exact JSON structure, GitHub Actions YAML, and schema definition

### Established Patterns
- No existing code patterns — this is greenfield
- Design document pseudocode provides the reference implementation pattern

### Integration Points
- `catalog.yaml` → build script reads catalog version and valid categories
- `prompts/**/*.md` → build script finds and processes all prompt files
- `schema/prompt.schema.json` → build script optionally validates output (CI always validates)
- Build output: `prompts.json` at repo root

</code_context>

<specifics>
## Specific Ideas

- Build script pseudocode in design doc is the reference implementation — follow its structure but fix the macOS bash 3.2 incompatibility (`declare -A` → POSIX alternative)
- The `jq -Rs '.'` approach for content escaping is non-negotiable — hand-rolled string concatenation will produce corrupt JSON on edge cases
- Error messages should name the offending file, not just report the error type

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-build-foundation*
*Context gathered: 2026-03-11*
