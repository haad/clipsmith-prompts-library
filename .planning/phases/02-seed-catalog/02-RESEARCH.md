# Phase 2: Seed Catalog - Research

**Researched:** 2026-03-11
**Domain:** Markdown prompt authoring, Flycut PromptDTO content format, YAML frontmatter conventions
**Confidence:** HIGH

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SEED-01 | 8 coding prompts: code-review-swift, explain-code, fix-bug, write-tests, refactor-code, add-error-handling, convert-to-async, optimize-performance | code-review-swift.md already exists; 7 new files needed; exact filenames prescribed in REQUIREMENTS.md |
| SEED-02 | 6 writing prompts: summarize-text, rewrite-formal, fix-grammar, simplify-language, write-email-reply, expand-bullet-points | All 6 new files needed; exact filenames prescribed |
| SEED-03 | 5 analysis prompts: analyze-data, compare-options, extract-action-items, identify-risks, create-summary-table | All 5 new files needed; exact filenames prescribed |
| SEED-04 | 4 creative prompts: brainstorm, write-story, generate-names, create-outline | All 4 new files needed; exact filenames prescribed |
| SEED-05 | All 23 prompts have version 1, meaningful content, and correct frontmatter | Verified via `bash scripts/build.sh` producing 23 objects; build script validates required fields automatically |
</phase_requirements>

## Summary

Phase 2 is a content authoring phase, not a technical implementation phase. The build infrastructure (Phase 1) is complete and working. The task is to create 22 new Markdown prompt files (1 of 23 already exists as `prompts/coding/code-review-swift.md`) and verify that `bash scripts/build.sh` produces a `prompts.json` containing exactly 23 objects sorted alphabetically by `id`.

The exact filenames for all 23 prompts are prescribed in REQUIREMENTS.md and confirmed in `dist/PROMPTS-REPO-DESIGN.md`. The frontmatter format, content structure, and validation rules are all enforced by the existing build script — so creating a file with missing `title` or `version`, or placing it in a directory not in `catalog.yaml`, will cause the build to fail immediately with an informative error message.

The primary creative decision is the prompt body content itself. Each prompt must be "immediately useful" to a Flycut user — short enough to paste with one keystroke, targeted enough to produce good AI output without additional editing. The design document provides two sample bodies (`code-review-swift.md` already committed, and `write-email-reply.md` shown as an example). All other bodies must be authored fresh, following the established pattern: direct instruction + `{{clipboard}}` as the primary input + a few focused requirements.

**Primary recommendation:** Write each of the 22 new prompt files in a single wave. Use `{{clipboard}}` as the primary input variable in every prompt — this is the Flycut-native pattern. Verify with `bash scripts/build.sh` after the wave; the build output confirms count, catches frontmatter errors, and produces the final prompts.json.

## Standard Stack

### Core

| Tool | Version | Purpose | Why Standard |
|------|---------|---------|--------------|
| Markdown with YAML frontmatter | — | Prompt file format | Established by Phase 1; build script parses exactly this format |
| bash scripts/build.sh | already built | Compile .md → prompts.json | Phase 1 deliverable; zero-dep; macOS 3.2 compatible |
| jq | 1.6+ | Validate prompts.json output | Already required by build script |

### Prompt File Format (Locked by Phase 1)

```
---
title: "Human-Readable Title"
version: 1
description: "One-line description for README generation"
variables: ["clipboard"]
---

[Prompt body with {{clipboard}} as primary input]
```

Required frontmatter fields: `title` (string, may contain colons — use quotes), `version` (integer, must be 1 for all seed prompts).

Optional frontmatter fields (do not affect prompts.json output, used for README generation):
- `description` — one-line description
- `variables` — array of template variable names used (e.g., `["clipboard"]`, `["clipboard", "tone"]`)
- `tags` — array of tag strings (future use)
- `category` — override the directory-derived category (only needed if file is misplaced)

### Template Variables

Two variable types are supported by Flycut:
- `{{clipboard}}` — built-in; substituted with clipboard contents at paste time. This is the primary input pattern for all seed prompts.
- `{{variablename}}` — custom; substituted from user-configured key-value pairs in Flycut Settings. Use sparingly (e.g., `{{tone}}`, `{{language}}`).

Unknown variables are left as literal text — not an error, just displayed verbatim.

## Architecture Patterns

### Prompt File Distribution

```
prompts/
├── coding/          # 8 prompts (1 exists, 7 new)
│   ├── code-review-swift.md      [EXISTS]
│   ├── explain-code.md           [NEW]
│   ├── fix-bug.md                [NEW]
│   ├── write-tests.md            [NEW]
│   ├── refactor-code.md          [NEW]
│   ├── add-error-handling.md     [NEW]
│   ├── convert-to-async.md       [NEW]
│   └── optimize-performance.md   [NEW]
├── writing/         # 6 prompts (all new)
│   ├── summarize-text.md
│   ├── rewrite-formal.md
│   ├── fix-grammar.md
│   ├── simplify-language.md
│   ├── write-email-reply.md
│   └── expand-bullet-points.md
├── analysis/        # 5 prompts (all new)
│   ├── analyze-data.md
│   ├── compare-options.md
│   ├── extract-action-items.md
│   ├── identify-risks.md
│   └── create-summary-table.md
└── creative/        # 4 prompts (all new)
    ├── brainstorm.md
    ├── write-story.md
    ├── generate-names.md
    └── create-outline.md
```

Note: `.gitkeep` files in `writing/`, `analysis/`, `creative/` directories must be removed once real `.md` files are added (or they can be left — the build script uses `find ... -name '*.md' -type f` so `.gitkeep` files are ignored). Actually `.gitkeep` has no `.md` extension so they are harmlessly ignored by the build script.

### Pattern: Effective Flycut Prompt Body

A prompt body that works well in Flycut has these characteristics:

1. **Opens with a direct instruction** — tells the AI exactly what to do in the first sentence
2. **Uses `{{clipboard}}` as the primary input** — positions it prominently, usually after context setup
3. **Provides 2-5 focused requirements** — numbered list or bullet points of what matters most
4. **Closes with output expectations** — a single sentence specifying the desired output format

**Example structure (from existing `code-review-swift.md`):**
```
[Direct instruction — what to do]

[Numbered/bulleted requirements list]

[Prompt variable input:]

{{clipboard}}

[Output format expectation]
```

**Example structure (from design doc `write-email-reply.md`):**
```
[Direct instruction — what to do:]

{{clipboard}}

[Additional context variable:]
{{variable}}

[Output constraint]
```

### Reference Content: All 23 Prompts (Titles and Descriptions)

Sourced from `dist/PROMPTS-REPO-DESIGN.md` — these titles and descriptions are the authoritative names:

**coding/ (8 prompts)**

| Filename (= id) | Title | Description |
|-----------------|-------|-------------|
| `code-review-swift` | Swift Code Review | Review Swift code for correctness, safety, and style |
| `explain-code` | Explain Code | Step-by-step code explanation |
| `fix-bug` | Fix Bug | Find and fix bugs in code |
| `write-tests` | Write Tests | Generate comprehensive unit tests |
| `refactor-code` | Refactor Code | Improve code structure without changing behavior |
| `add-error-handling` | Add Error Handling | Add proper error handling to code |
| `convert-to-async` | Convert to Async/Await | Modernize callback/completion handler code |
| `optimize-performance` | Optimize Performance | Identify and fix performance issues |

**writing/ (6 prompts)**

| Filename (= id) | Title | Description |
|-----------------|-------|-------------|
| `summarize-text` | Summarize Text | Concise summary of text |
| `rewrite-formal` | Rewrite Formally | Professional tone rewrite |
| `fix-grammar` | Fix Grammar | Grammar and spelling correction |
| `simplify-language` | Simplify Language | Make text clearer and simpler |
| `write-email-reply` | Write Email Reply | Draft a professional email response |
| `expand-bullet-points` | Expand Bullet Points | Turn bullet points into full paragraphs |

**analysis/ (5 prompts)**

| Filename (= id) | Title | Description |
|-----------------|-------|-------------|
| `analyze-data` | Analyze Data | Extract key insights from data |
| `compare-options` | Compare Options | Pros/cons comparison |
| `extract-action-items` | Extract Action Items | Pull action items from meeting notes/text |
| `identify-risks` | Identify Risks | Risk analysis of a plan or proposal |
| `create-summary-table` | Create Summary Table | Structure text into a comparison table |

**creative/ (4 prompts)**

| Filename (= id) | Title | Description |
|-----------------|-------|-------------|
| `brainstorm` | Brainstorm Ideas | Creative brainstorming on a topic |
| `write-story` | Write Story | Short story from a prompt |
| `generate-names` | Generate Names | Name ideas for projects/products/features |
| `create-outline` | Create Outline | Structured outline from a topic |

### Anti-Patterns to Avoid

- **Vague instructions:** "Help me with this code" produces inconsistent results. "Find and fix the bug in this code, explain what was wrong and why the fix is correct" is specific.
- **Titles without quotes when containing colons:** `title: Swift Code Review: Advanced` will be parsed incorrectly. Use `title: "Swift Code Review: Advanced"`.
- **Missing `{{clipboard}}`:** The success criteria (SEED-05) requires at least one prompt to contain `{{clipboard}}` or `{{variable}}`. All coding, writing, and analysis prompts naturally use `{{clipboard}}`. Ensure it's present in the body, not just documented in `variables:`.
- **Empty content after frontmatter:** Forgetting the prompt body entirely — build script checks `minLength: 1` via schema, and validates content is non-empty.
- **Wrong version number:** All seed prompts must have `version: 1`. Using `version: 2` or higher on a new seed prompt violates SEED-05.
- **Filename with uppercase or underscores:** `write_tests.md` or `WriteTests.md` would produce IDs that fail the `^[a-z0-9-]+$` pattern. Use strictly lowercase kebab-case.
- **Creative prompts without `{{clipboard}}`:** Some creative prompts (brainstorm, generate-names, create-outline) take a topic as input — use `{{clipboard}}` for the topic text.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Frontmatter validation | Manually check required fields in each file | Run `bash scripts/build.sh` | Build script already validates title, version, category — exits with named error on any missing field |
| JSON structure verification | Manually inspect prompts.json | `jq '.prompts | length' prompts.json` | Exact count verification in one command |
| Duplicate ID detection | Manual cross-category comparison | Build script Pass 1 | Already detects duplicates via `sort | uniq -d` |
| Content quality review | No automation | Human review of each body | Prompt quality is subjective; automation can't judge usefulness |

**Key insight:** The entire authoring workflow reduces to: write `.md` files → run `bash scripts/build.sh` → if exit 0 and count is 23, phase is complete. The build script is the validator.

## Common Pitfalls

### Pitfall 1: `.gitkeep` Removal Confusion

**What goes wrong:** Implementer thinks `.gitkeep` files need to be explicitly removed before adding real prompts.

**Why it happens:** `.gitkeep` is a convention for committing empty directories. When real files are added, some workflows require removing the `.gitkeep`.

**How to avoid:** The build script uses `find ... -name '*.md' -type f` — it only processes `.md` files. `.gitkeep` files (no `.md` extension) are completely ignored. Adding real prompt files is sufficient; `.gitkeep` removal is optional. Removing `.gitkeep` files keeps the directory cleaner but is not required for the build to work.

**Warning signs:** Unnecessary git operations to remove `.gitkeep` before adding prompts.

### Pitfall 2: Title String With Colon Not Quoted

**What goes wrong:** `title: Fix Bug: Swift` in frontmatter is parsed as `Fix Bug` (truncated at second colon).

**Why it happens:** The build script's title parser strips the `title:` prefix using `sed 's/^title:[[:space:]]*//'` then optionally strips surrounding quotes. When the title value itself contains a colon and is not wrapped in quotes, the second colon becomes ambiguous.

**How to avoid:** Always quote titles that contain colons: `title: "Fix Bug: Swift"`. Convention: quote all titles unconditionally to be safe.

**Warning signs:** Titles in prompts.json are truncated compared to what was written in frontmatter.

### Pitfall 3: Missing `{{clipboard}}` in Body vs. Only in `variables:` Field

**What goes wrong:** Author documents `variables: ["clipboard"]` in frontmatter but forgets to put `{{clipboard}}` in the prompt body. The variable list in frontmatter is metadata only (not sent to Flycut) — the actual substitution happens on literal `{{clipboard}}` text in the body.

**Why it happens:** `variables:` in frontmatter looks authoritative. It is documentation only.

**How to avoid:** Always include `{{clipboard}}` literally in the prompt body where user input should appear. The success criterion (SEED-05 check 4) verifies at least one prompt's `content` field contains `{{clipboard}}` — this checks the built JSON, confirming the body passthrough.

**Warning signs:** `jq '.prompts[] | select(.content | contains("{{clipboard}}"))' prompts.json` returns an empty array.

### Pitfall 4: Wrong Final Count (22 vs 23)

**What goes wrong:** Implementer creates all 22 new files but forgets that `code-review-swift.md` already exists, resulting in 23 total. Or conversely, treats `code-review-swift.md` as not counting toward the 23 and creates 22 additional files expecting 22 total, then creates one more duplicate.

**Why it happens:** Phase 1 left exactly 1 of the 23 seed prompts (`code-review-swift.md`) as the pipeline verification prompt. Phase 2 needs exactly 22 more.

**How to avoid:** Target: 22 new `.md` files = 23 total including existing. After all files are created: `jq '.prompts | length' prompts.json` must output `23`.

**Warning signs:** Build output says "Built prompts.json: 22 prompts" or "24 prompts."

### Pitfall 5: Overwriting Existing `code-review-swift.md`

**What goes wrong:** Implementer creates all coding prompts from scratch and accidentally replaces `code-review-swift.md` with different content.

**Why it happens:** When processing all 8 coding prompts as a batch, the existing file may get overwritten if the implementer isn't paying attention.

**How to avoid:** Read the existing `prompts/coding/code-review-swift.md` before creating new files. It should NOT be modified in Phase 2 (it's already correct, version 1, with `{{clipboard}}`). Only create the 7 other coding prompts.

**Warning signs:** `git diff prompts/coding/code-review-swift.md` shows changes.

## Code Examples

Verified patterns from existing codebase and design document:

### Existing Prompt (Reference — Do Not Modify)

```markdown
---
title: "Swift Code Review"
version: 1
description: "Review Swift code for correctness, safety, and idiomatic style"
variables: ["clipboard"]
---

Review this Swift code for correctness, safety, and style. Focus on:

1. **Correctness** — logic errors, off-by-one, nil handling, race conditions
2. **Safety** — force unwraps, unowned references, unchecked casts
3. **Style** — naming conventions, Swift idioms, unnecessary complexity
4. **Performance** — obvious N+1 patterns, unnecessary allocations

Code to review:

{{clipboard}}

Provide specific line-by-line feedback with suggested fixes.
```

### Example New Coding Prompt: `explain-code.md`

```markdown
---
title: "Explain Code"
version: 1
description: "Step-by-step explanation of how code works"
variables: ["clipboard"]
---

Explain how this code works, step by step:

{{clipboard}}

Cover: what the code does, how it does it, and why each key decision was made. Assume the reader is a developer unfamiliar with this codebase.
```

### Example New Writing Prompt: `write-email-reply.md`

```markdown
---
title: "Write Email Reply"
version: 1
description: "Draft a professional email reply"
variables: ["clipboard"]
---

Write a professional reply to this email:

{{clipboard}}

Keep the reply concise, address all points raised, and end with a clear next step.
```

(Source: `dist/PROMPTS-REPO-DESIGN.md` — used verbatim as the design doc example)

### Validation Commands (Run After Creating All Files)

```bash
# Primary build and count check
bash scripts/build.sh
# Expected output: "Built prompts.json: 23 prompt(s), catalog version 1"

# Verify exact count
jq '.prompts | length' prompts.json
# Expected: 23

# Verify all 4 categories present
jq '[.prompts[].category] | unique | sort' prompts.json
# Expected: ["analysis","coding","creative","writing"]

# Verify category counts
jq '[.prompts[] | select(.category == "coding")] | length' prompts.json   # 8
jq '[.prompts[] | select(.category == "writing")] | length' prompts.json  # 6
jq '[.prompts[] | select(.category == "analysis")] | length' prompts.json # 5
jq '[.prompts[] | select(.category == "creative")] | length' prompts.json # 4

# Verify all version: 1
jq '[.prompts[] | select(.version != 1)] | length' prompts.json
# Expected: 0

# Verify at least one prompt has {{clipboard}} in content
jq '[.prompts[] | select(.content | contains("{{clipboard}}"))] | length' prompts.json
# Expected: >= 1 (should be most prompts)

# Verify all required fields non-empty
jq -e '.prompts | all(.id != "" and .title != "" and .category != "" and .content != "")' prompts.json
# Expected: true

# Verify alphabetical sort by id
jq '[.prompts[].id]' prompts.json
# Should list all 23 IDs in alphabetical order (a comes before b, etc.)

# Verify id pattern (lowercase kebab-case only)
jq -e '.prompts | all(.id | test("^[a-z0-9-]+$"))' prompts.json
# Expected: true

# Verify no leading blank lines in content
jq '[.prompts[] | select(.content | startswith("\n")) | .id] | length' prompts.json
# Expected: 0
```

### Expected Alphabetical ID Order in prompts.json

After all 23 files are created, IDs will appear in this order (alphabetical):
```
add-error-handling
analyze-data
brainstorm
code-review-swift
compare-options
convert-to-async
create-outline
create-summary-table
expand-bullet-points
explain-code
extract-action-items
fix-bug
fix-grammar
generate-names
identify-risks
optimize-performance
refactor-code
rewrite-formal
simplify-language
summarize-text
write-email-reply
write-story
write-tests
```

This is the expected order from `find ... | sort` (POSIX sort, lexicographic).

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Bundled JSON in app binary | External JSON synced at runtime | Phase 1 design | Users can get prompt updates without app updates |
| Single monolithic prompts.json | Individual .md source files compiled to JSON | Phase 1 design | Human-editable source with version control |
| No template variables | `{{clipboard}}` and custom `{{variable}}` | Flycut feature | Prompts adapt to current clipboard context without copy-paste pre-step |

## Open Questions

1. **Content depth for creative prompts**
   - What we know: `brainstorm`, `write-story`, `generate-names`, `create-outline` take a topic/concept as clipboard input
   - What's unclear: Should creative prompts use `{{clipboard}}` for the topic (user copies a concept, pastes the prompt), or should they be open-ended instructions that the user fills in manually?
   - Recommendation: Use `{{clipboard}}` for the topic — this is consistent with the Flycut-native pattern. User copies a topic, runs the prompt, gets output.

2. **Whether to bump `catalog.yaml` version from 1 to 2 after adding all 23 prompts**
   - What we know: The design doc says "Bump on every merge to main that changes prompt content." `catalog.yaml` is currently `version: 1`. SEED-05 says prompts have version 1 — this is per-prompt version, not catalog version.
   - What's unclear: Whether the Phase 2 completion should bump `catalog.yaml` version from 1 to 2 (since prompts.json content is changing significantly).
   - Recommendation: Bump `catalog.yaml` version to 2 when all 23 prompts are added and the final prompts.json is committed. This follows the design doc rule and is a good practice milestone.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | bash + jq assertions (build script is the test harness) |
| Config file | scripts/build.sh (already exists) |
| Quick run command | `bash scripts/build.sh` |
| Full suite command | `bash scripts/build.sh && jq '.prompts \| length' prompts.json` |
| Estimated runtime | ~2 seconds |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SEED-01 | 8 coding prompts present and valid | integration | `jq '[.prompts[] \| select(.category == "coding")] \| length' prompts.json` (expect 8) | ✅ build.sh exists |
| SEED-02 | 6 writing prompts present and valid | integration | `jq '[.prompts[] \| select(.category == "writing")] \| length' prompts.json` (expect 6) | ✅ build.sh exists |
| SEED-03 | 5 analysis prompts present and valid | integration | `jq '[.prompts[] \| select(.category == "analysis")] \| length' prompts.json` (expect 5) | ✅ build.sh exists |
| SEED-04 | 4 creative prompts present and valid | integration | `jq '[.prompts[] \| select(.category == "creative")] \| length' prompts.json` (expect 4) | ✅ build.sh exists |
| SEED-05 | All 23 prompts have version 1, non-empty content, correct frontmatter | integration | `jq '[.prompts[] \| select(.version != 1)] \| length' prompts.json` (expect 0) + `bash scripts/build.sh` exits 0 | ✅ build.sh exists |
| SEED-05 (template) | At least one prompt contains {{clipboard}} or {{variable}} | unit | `jq '[.prompts[] \| select(.content \| contains("{{clipboard}}"))] \| length' prompts.json` (expect ≥1) | ✅ build.sh exists |

### Sampling Rate

- **Per task commit (after adding each batch of prompts):** `bash scripts/build.sh`
- **Per wave merge:** Full suite: `bash scripts/build.sh && jq '.prompts | length' prompts.json`
- **Phase gate:** All validation commands from "Validation Commands" section above return expected values before `/gsd:verify-work`

### Wave 0 Gaps

None — existing test infrastructure (`scripts/build.sh` + `jq`) covers all phase requirements. No new test infrastructure needed.

## Sources

### Primary (HIGH confidence)

- `/Volumes/Devel/apple/prompt-library/dist/PROMPTS-REPO-DESIGN.md` — Complete specification; all 23 prompt filenames, titles, descriptions; two example prompt bodies; frontmatter format; validation workflow
- `/Volumes/Devel/apple/prompt-library/.planning/REQUIREMENTS.md` — Authoritative requirement IDs and exact filenames for all 23 prompts
- `/Volumes/Devel/apple/prompt-library/prompts/coding/code-review-swift.md` — Existing reference prompt; established body format and frontmatter pattern
- `/Volumes/Devel/apple/prompt-library/scripts/build.sh` — Implemented build pipeline; validation behavior is exactly what is enforced

### Secondary (MEDIUM confidence)

- `dist/PROMPTS-REPO-DESIGN.md` — `write-email-reply.md` body example; template variable behavior documentation
- Phase 1 RESEARCH.md — Established pitfalls (title colon truncation, leading blank lines, `|| true` for optional fields) fully apply to authoring

### Tertiary (LOW confidence)

- None for this phase — all findings are derived from committed code and the design document

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — format is locked by Phase 1; build script is the enforcer
- Architecture: HIGH — exact filenames prescribed in REQUIREMENTS.md and design doc; file structure already exists
- Pitfalls: HIGH — derived from analysis of existing code and design doc, not speculation

**Research date:** 2026-03-11
**Valid until:** 2026-06-11 (stable domain — prompt authoring format locked by Phase 1; build script behavior won't change)
