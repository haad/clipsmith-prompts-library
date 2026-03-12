# Phase 4: Documentation - Research

**Researched:** 2026-03-12
**Domain:** README authoring — contributor guide for a Markdown-to-JSON prompt pipeline
**Confidence:** HIGH

## Summary

Phase 4 is a pure documentation phase. All technical infrastructure is complete: the build script, 23 prompt files, GitHub Actions CI/CD pipeline, and landing page exist and are validated. The only deliverable is a `README.md` that makes the repository self-explanatory to a contributor who has never seen it.

The README must serve two audiences simultaneously. First, a Flycut user who wants to understand what the sync URL is and how to use prompts in the app. Second, a contributor who wants to add or update a prompt without accidentally breaking the Flycut sync contract. The critical knowledge gap for contributors is the version-bump requirement: omitting a version bump on a content change silently prevents all users from receiving the update — Flycut only upserts when `remote.version > local.version`.

The README content is fully deterministic from existing project artifacts. No new technical decisions are needed. Every section maps directly to a concrete, already-implemented system: the frontmatter format, the build script behavior, the sync URL, the variable interpolation system, and the 23-prompt catalog.

**Primary recommendation:** Write a single `README.md` at repo root covering all six DOCS requirements in order of how a reader encounters them — what it is, how to use it, how to contribute (add + update + variables), versioning rules, and the prompt table. Use the design document's content outline as the authoritative specification.

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| DOCS-01 | README explains what the repo is, how to use prompts in Flycut, and the sync URL | Sync URL and Flycut behavior fully documented in `dist/PROMPTS-REPO-DESIGN.md` and `PROJECT.md`; use `https://raw.githubusercontent.com/generalarcade/flycut-prompts/gh-pages/prompts.json` |
| DOCS-02 | README documents how to add a prompt (file format, frontmatter fields, PR workflow) | Frontmatter field table and filename rules are specified in PROMPTS-REPO-DESIGN.md; all fields verified against actual prompt files |
| DOCS-03 | README documents how to update a prompt (content change + version bump requirement) | Version-gated sync behavior documented in PROMPTS-REPO-DESIGN.md §Sync Behavior; the "omitting silently prevents updates" risk is explicitly called out |
| DOCS-04 | README explains template variables (`{{clipboard}}` and custom variables) | Variable interpolation behavior documented in PROMPTS-REPO-DESIGN.md §Template Variables; `{{clipboard}}` is built-in, others are user-defined key-value pairs in Flycut Settings |
| DOCS-05 | README includes table of all prompts grouped by category | All 23 prompts exist with titles and descriptions; table can be assembled directly from frontmatter metadata |
| DOCS-06 | README documents versioning rules prominently (never decrease, bump on update, catalog version) | Five versioning rules documented in PROMPTS-REPO-DESIGN.md §Versioning Rules; catalog version currently 2 |
</phase_requirements>

---

## Standard Stack

### Core

| Item | Value | Purpose | Why Standard |
|------|-------|---------|--------------|
| README.md | Markdown | Primary contributor documentation | GitHub renders it on the repo homepage automatically |
| GitHub-flavored Markdown | Standard | Tables, code blocks, headers | Native to GitHub; no rendering step needed |

### Supporting

| Item | Value | Purpose | When to Use |
|------|-------|---------|-------------|
| Code blocks with language hints | ` ```yaml `, ` ```bash `, ` ```json ` | Syntax-highlighted examples | All frontmatter examples, build commands, JSON output |
| Markdown tables | `| col | col |` syntax | Frontmatter field reference, prompt catalog listing | Structured data the reader scans rather than reads linearly |
| `>` blockquote | Markdown callout | Version bump warning | Call out the silent-failure risk prominently |

### No External Tools Needed

This phase has no library dependencies. The README is a single static Markdown file. No build step, no linting toolchain, no npm packages. The design document's content outline is complete and authoritative.

**Installation:** none required

---

## Architecture Patterns

### Recommended README Structure

```
README.md
├── Title + one-liner description
├── ## What is this?              ← DOCS-01: repo purpose + sync URL
├── ## Using prompts in Flycut    ← DOCS-01: how to use in app
├── ## Adding a prompt            ← DOCS-02: new file walkthrough
├── ## Updating a prompt          ← DOCS-03: content + version bump
├── ## Template variables         ← DOCS-04: {{clipboard}} and custom
├── ## Versioning rules           ← DOCS-06: the five rules
├── ## Building locally           ← contributor workflow
└── ## Prompt catalog             ← DOCS-05: 23-prompt table by category
```

### Pattern 1: Frontmatter Reference Table

All frontmatter fields are documented in `dist/PROMPTS-REPO-DESIGN.md`. The README table must match exactly:

```markdown
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | **yes** | Display title shown in Flycut's prompt list |
| `version` | integer | **yes** | Monotonically increasing. Bump to push updates. Start at 1. |
| `category` | string | no | Overrides directory-derived category. Rarely needed. |
| `description` | string | no | One-line description. Not sent to Flycut. |
| `tags` | string[] | no | For future use. Not sent to Flycut. |
| `variables` | string[] | no | Documents which `{{variables}}` the prompt uses. |
```

### Pattern 2: Add-a-Prompt Walkthrough

Step-by-step numbered list with a complete example file. Must include:
1. Choose the right category directory
2. Create `prompts/<category>/<kebab-case-id>.md`
3. Write the required frontmatter block (title + version: 1)
4. Write the prompt body
5. Run `bash scripts/build.sh` locally to verify
6. Submit a PR

Include a minimal working example:
```markdown
---
title: "Your Prompt Title"
version: 1
description: "One-line description"
---

Your prompt text here.

{{clipboard}}
```

### Pattern 3: Version Bump Warning (Prominent Callout)

This is the highest-risk gap for contributors. Use a blockquote or bold warning:

```markdown
> **Important:** Every content change to a prompt file **must** include a version bump
> (e.g., `version: 1` → `version: 2`). Without this, Flycut's sync system will see
> `remote.version == local.version` and skip the update silently. Users will not
> receive the change.
```

### Pattern 4: Prompt Catalog Table

Grouped by category, sorted alphabetically within each group (matching `prompts.json` sort order). Use the `description` frontmatter field from each file as the table's description column.

```markdown
### Coding (8 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `add-error-handling` | Add Error Handling | Add proper error handling to code |
| `code-review-swift` | Swift Code Review | Review Swift code for correctness, safety, and idiomatic style |
...
```

### Anti-Patterns to Avoid

- **Mixing audience concerns:** Don't interleave "how to use in Flycut" content with "how to contribute" content. Keep them in separate sections so each audience can skip what they don't need.
- **Buried version bump warning:** Do not mention the version bump rule only in a table footnote. It must appear as a prominent standalone section or callout where a contributor editing a prompt will encounter it.
- **Omitting the local build instructions:** Contributors must know how to verify their prompt before submitting a PR. `bash scripts/build.sh` is the verification step.
- **Hardcoding catalog version:** Do not document "current version is 2" in the README — the catalog version is metadata for Flycut debugging, not something contributors need to manually manage.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Prompt table data | Scrape from files at write time | Assemble directly from frontmatter metadata already read during research | All 23 descriptions are known; static table is correct for v1 |
| Versioning explainer | Write from scratch | Use the five rules in `dist/PROMPTS-REPO-DESIGN.md §Versioning Rules` verbatim | Already battle-tested phrasing that covers all edge cases |
| Sync behavior explanation | Write from scratch | Condense from `PROMPTS-REPO-DESIGN.md §Sync Behavior` | The five sync behaviors are already precisely documented |

**Key insight:** The design document (`dist/PROMPTS-REPO-DESIGN.md`) contains the authoritative phrasing for every section the README needs. The task is condensation and reorganization, not invention.

---

## Common Pitfalls

### Pitfall 1: Missing Version Bump Documentation
**What goes wrong:** Contributor edits a prompt, increments content but not `version`. Flycut users never receive the update. No error is thrown.
**Why it happens:** The failure is silent — build succeeds, CI passes, deploy succeeds. Only the sync consumer (Flycut) silently skips the update.
**How to avoid:** Document the version bump rule in both the "Updating a prompt" section AND a standalone "Versioning rules" section. Use a visual callout (blockquote or bold) so it stands out from surrounding prose.
**Warning signs:** A PR that changes prompt content but does not change `version` in the same file.

### Pitfall 2: Wrong Sync URL
**What goes wrong:** README documents the `main` branch URL instead of the `gh-pages` URL.
**Why it happens:** The design document notes two URLs: `main/prompts.json` (old default) and `gh-pages/prompts.json` (correct after CI/CD is live).
**How to avoid:** Use the `gh-pages` URL: `https://raw.githubusercontent.com/generalarcade/flycut-prompts/gh-pages/prompts.json`. This is confirmed by `PROJECT.md §Context`.
**Warning signs:** README references `/main/prompts.json`.

### Pitfall 3: Incomplete Frontmatter Field Coverage
**What goes wrong:** README omits optional fields (`description`, `tags`, `variables`) or incorrectly marks them as required.
**Why it happens:** The minimal working example only needs `title` and `version`, but contributors need to know about optional fields.
**How to avoid:** Include the complete field reference table even though most fields are optional. Note which fields are sent to Flycut (title, version, category, content via body) vs. which are metadata-only (description, tags, variables).
**Warning signs:** README only documents `title` and `version`.

### Pitfall 4: Incorrect Category Directory Instructions
**What goes wrong:** README says "add a `category` field" when the correct approach is to put the file in the right directory.
**Why it happens:** The `category` field exists but is an override mechanism, not the standard approach.
**How to avoid:** The "adding a prompt" walkthrough must lead with "create the file in `prompts/<category>/`". Mention the frontmatter override as a rare exception only.

### Pitfall 5: Undocumented GitHub Pages Setup
**What goes wrong:** A new maintainer deploys successfully but the gh-pages URL returns 404 because GitHub Pages is not enabled in repository Settings.
**Why it happens:** The CI/CD workflow pushes to the `gh-pages` branch but cannot enable GitHub Pages — that requires a manual step in repository Settings.
**How to avoid:** Include a one-time setup note: "After the first successful CI run, enable GitHub Pages in Settings → Pages → Source: Deploy from branch → `gh-pages` → `/ (root)`." This was called out as a blocker in `STATE.md §Blockers`.

---

## Code Examples

### Complete Example Prompt File (for README)

```markdown
---
title: "Summarize Text"
version: 1
description: "Concise summary of text"
variables: ["clipboard"]
---

Summarize this text concisely. Capture:

1. **Main points** — the central ideas or arguments presented
2. **Key arguments** — the supporting reasoning or evidence
3. **Conclusions** — what the text ultimately asserts or recommends

Text to summarize:

{{clipboard}}

Keep the summary to 3-5 sentences unless the source is very long.
```

### Local Build Command (for README)

```bash
bash scripts/build.sh
```

Expected output:
```
Built prompts.json: 23 prompt(s), catalog version 2
Validation passed: prompts.json structure is valid
```

### Sync URL (authoritative, from PROJECT.md)

```
https://raw.githubusercontent.com/generalarcade/flycut-prompts/gh-pages/prompts.json
```

---

## Complete Prompt Catalog Data

All 23 prompts for the DOCS-05 table, assembled from frontmatter:

### Coding (8)

| ID | Title | Description |
|----|-------|-------------|
| `add-error-handling` | Add Error Handling | Add proper error handling to code |
| `code-review-swift` | Swift Code Review | Review Swift code for correctness, safety, and idiomatic style |
| `convert-to-async` | Convert to Async/Await | Modernize callback/completion handler code |
| `explain-code` | Explain Code | Step-by-step explanation of how code works |
| `fix-bug` | Fix Bug | Find and fix bugs in code |
| `optimize-performance` | Optimize Performance | Identify and fix performance issues |
| `refactor-code` | Refactor Code | Improve code structure without changing behavior |
| `write-tests` | Write Tests | Generate comprehensive unit tests |

### Writing (6)

| ID | Title | Description |
|----|-------|-------------|
| `expand-bullet-points` | Expand Bullet Points | Turn bullet points into full paragraphs |
| `fix-grammar` | Fix Grammar | Grammar and spelling correction |
| `rewrite-formal` | Rewrite Formally | Professional tone rewrite |
| `simplify-language` | Simplify Language | Make text clearer and simpler |
| `summarize-text` | Summarize Text | Concise summary of text |
| `write-email-reply` | Write Email Reply | Draft a professional email reply |

### Analysis (5)

| ID | Title | Description |
|----|-------|-------------|
| `analyze-data` | Analyze Data | Extract key insights from data |
| `compare-options` | Compare Options | Pros/cons comparison |
| `create-summary-table` | Create Summary Table | Structure text into a comparison table |
| `extract-action-items` | Extract Action Items | Pull action items from meeting notes or text |
| `identify-risks` | Identify Risks | Risk analysis of a plan or proposal |

### Creative (4)

| ID | Title | Description |
|----|-------|-------------|
| `brainstorm` | Brainstorm Ideas | Creative brainstorming on a topic |
| `create-outline` | Create Outline | Structured outline from a topic |
| `generate-names` | Generate Names | Name ideas for projects, products, or features |
| `write-story` | Write Story | Short story from a prompt |

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Sync from `main` branch `prompts.json` | Sync from `gh-pages` branch `prompts.json` | Phase 3 complete | README must document the `gh-pages` URL, not the `main` branch URL |

**Deprecated/outdated:**
- `https://raw.githubusercontent.com/generalarcade/flycut-prompts/main/prompts.json`: Was the original default URL in Flycut before gh-pages deployment was established. README must document the `gh-pages` URL as the canonical sync URL.

---

## Open Questions

1. **PR workflow specifics**
   - What we know: The design doc says "submit a PR" but Phase 4 requirements don't specify a branch naming convention or PR checklist.
   - What's unclear: Whether to document a specific branch naming pattern (`feat/prompt-name` etc.)
   - Recommendation: Keep it simple — "fork the repo, create a branch, submit a PR." Detailed contribution guidelines are explicitly deferred to v2 (CTRB-01, CTRB-02).

2. **catalog.yaml version — should contributors bump it?**
   - What we know: The versioning rules say "Bump `catalog.yaml` version on every release." But catalog version is described as "for human tracking, not sync logic."
   - What's unclear: Whether external contributors are expected to bump catalog version in their PR.
   - Recommendation: Document the rule (DOCS-06 requires it) but note that maintainers handle catalog version bumps on merge. This avoids merge conflicts in PRs.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | bash (no test framework — documentation is validated by human review and build script) |
| Config file | none |
| Quick run command | `bash scripts/build.sh` |
| Full suite command | `bash scripts/build.sh` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DOCS-01 | README contains sync URL and repo description | manual | grep for URL: `grep 'gh-pages/prompts.json' README.md` | ❌ Wave 0 |
| DOCS-02 | README contains frontmatter field table | manual | `grep -c 'title.*version.*Required' README.md` | ❌ Wave 0 |
| DOCS-03 | README contains version bump documentation | manual | `grep -i 'version bump' README.md` | ❌ Wave 0 |
| DOCS-04 | README contains {{clipboard}} documentation | manual | `grep '{{clipboard}}' README.md` | ❌ Wave 0 |
| DOCS-05 | README contains all 23 prompt IDs in table | semi-auto | `grep -c 'add-error-handling\|code-review-swift\|...' README.md` | ❌ Wave 0 |
| DOCS-06 | README contains versioning rules section | manual | `grep -i 'versioning rules' README.md` | ❌ Wave 0 |

All DOCS requirements are documentation correctness requirements, not behavioral requirements. They are verified by reading the README, not by running code. The automated checks above confirm presence of key content strings but human review is the final gate.

### Sampling Rate
- **Per task commit:** `grep 'gh-pages/prompts.json' /Volumes/Devel/apple/prompt-library/README.md`
- **Per wave merge:** Read README.md and verify all 6 DOCS requirements are covered
- **Phase gate:** Human review of README against all 6 DOCS requirements before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `README.md` — the sole deliverable of this phase; does not exist yet

---

## Sources

### Primary (HIGH confidence)
- `dist/PROMPTS-REPO-DESIGN.md` — Complete specification including README content outline, versioning rules, frontmatter fields, sync behavior, and template variable behavior
- `PROJECT.md` — Canonical sync URL, key decisions, constraints, Flycut sync behavior summary
- `prompts.json` — Authoritative source for all 23 prompt IDs and content
- `prompts/**/*.md` — All 23 prompt files; frontmatter descriptions used directly in catalog table
- `scripts/build.sh` — Build command and expected output for "building locally" section
- `.planning/STATE.md §Blockers` — GitHub Pages manual setup requirement documented here

### Secondary (MEDIUM confidence)
- `REQUIREMENTS.md §Documentation` — Exact requirement text for DOCS-01 through DOCS-06
- `ROADMAP.md §Phase 4` — Success criteria that the README must satisfy

### Tertiary (LOW confidence)
- None — all findings are verifiable from project source files

---

## Metadata

**Confidence breakdown:**
- README structure: HIGH — derived directly from PROMPTS-REPO-DESIGN.md §README.md Content and REQUIREMENTS.md
- Frontmatter table: HIGH — verified against actual prompt files
- Sync URL: HIGH — confirmed in PROJECT.md §Context
- Versioning rules: HIGH — documented in PROMPTS-REPO-DESIGN.md §Versioning Rules
- Prompt catalog table: HIGH — assembled directly from all 23 `.md` file frontmatter fields
- Pitfalls: HIGH — derived from explicit warnings in design doc and STATE.md blockers

**Research date:** 2026-03-12
**Valid until:** Stable indefinitely — no external dependencies; all content is derived from this repository's own source files
