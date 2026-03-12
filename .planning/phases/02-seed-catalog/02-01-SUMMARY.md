---
phase: 02-seed-catalog
plan: "01"
subsystem: content
tags: [prompts, markdown, yaml, coding, writing, catalog]

# Dependency graph
requires:
  - phase: 01-build-foundation
    provides: build.sh pipeline, prompt file format, catalog.yaml schema
provides:
  - 7 coding prompts (explain-code, fix-bug, write-tests, refactor-code, add-error-handling, convert-to-async, optimize-performance)
  - 6 writing prompts (summarize-text, rewrite-formal, fix-grammar, simplify-language, write-email-reply, expand-bullet-points)
  - 13 new prompt files totalling 14 compiled coding+writing prompts in prompts.json
affects:
  - 02-seed-catalog (remaining plans building analysis and creative categories)
  - 03-ci-cd (will validate all 23 prompts compile in CI)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Prompt file format: YAML frontmatter (title quoted, version integer, description quoted, variables array) + instruction body + {{clipboard}} variable + output expectation"

key-files:
  created:
    - prompts/coding/explain-code.md
    - prompts/coding/fix-bug.md
    - prompts/coding/write-tests.md
    - prompts/coding/refactor-code.md
    - prompts/coding/add-error-handling.md
    - prompts/coding/convert-to-async.md
    - prompts/coding/optimize-performance.md
    - prompts/writing/summarize-text.md
    - prompts/writing/rewrite-formal.md
    - prompts/writing/fix-grammar.md
    - prompts/writing/simplify-language.md
    - prompts/writing/write-email-reply.md
    - prompts/writing/expand-bullet-points.md
  modified:
    - prompts.json

key-decisions:
  - "Prompt bodies use numbered bold-label lists (matching code-review-swift.md reference) for consistency across all categories"
  - "Writing prompts include output format guidance (e.g., 'provide the reply text only, ready to send') to make prompts immediately actionable"

patterns-established:
  - "Prompt structure: direct instruction + numbered focus areas (2-5 items) + {{clipboard}} + output expectation"
  - "All titles are quoted strings to prevent YAML colon-truncation issues"

requirements-completed: [SEED-01, SEED-02, SEED-05]

# Metrics
duration: 2min
completed: 2026-03-12
---

# Phase 2 Plan 01: Seed Catalog — Coding and Writing Prompts Summary

**13 new prompt files (7 coding + 6 writing) authored and verified via build.sh, expanding prompts.json from 1 to 14 entries with consistent {{clipboard}} variable format**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-12T06:14:59Z
- **Completed:** 2026-03-12T06:17:19Z
- **Tasks:** 2
- **Files modified:** 14 (13 new prompt files + prompts.json)

## Accomplishments

- Created 7 coding prompts covering explain, fix-bug, write-tests, refactor, add-error-handling, convert-to-async, and optimize-performance
- Created 6 writing prompts covering summarize, rewrite-formally, fix-grammar, simplify, email-reply, and expand-bullets
- All 14 prompts (including pre-existing code-review-swift.md) compile to prompts.json with correct category, version 1, and {{clipboard}} variable

## Task Commits

Each task was committed atomically:

1. **Task 1: Create 7 coding prompt files** - `f6508bd` (feat)
2. **Task 2: Create 6 writing prompt files** - `6c54215` (feat)

## Files Created/Modified

- `prompts/coding/explain-code.md` - Step-by-step code explanation prompt
- `prompts/coding/fix-bug.md` - Bug finding and fixing prompt
- `prompts/coding/write-tests.md` - Comprehensive unit test generation prompt
- `prompts/coding/refactor-code.md` - Code structure improvement prompt
- `prompts/coding/add-error-handling.md` - Error handling addition prompt
- `prompts/coding/convert-to-async.md` - Callback to async/await conversion prompt
- `prompts/coding/optimize-performance.md` - Performance bottleneck identification prompt
- `prompts/writing/summarize-text.md` - Concise text summarization prompt
- `prompts/writing/rewrite-formal.md` - Professional tone rewrite prompt
- `prompts/writing/fix-grammar.md` - Grammar and spelling correction prompt
- `prompts/writing/simplify-language.md` - Clarity and simplicity rewrite prompt
- `prompts/writing/write-email-reply.md` - Professional email reply drafting prompt
- `prompts/writing/expand-bullet-points.md` - Bullet points to full prose prompt
- `prompts.json` - Rebuilt with 14 prompts (8 coding + 6 writing)

## Decisions Made

- Prompt bodies use numbered bold-label lists matching the `code-review-swift.md` reference style for visual consistency across all categories
- Writing prompts include explicit output format guidance (e.g., "provide the reply text only, ready to send") to make them immediately actionable without follow-up

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Coding (8 prompts) and Writing (6 prompts) categories complete
- Next plans will add Analysis and Creative categories (remaining 9 prompts to reach 23 total)
- prompts.json is consistent and build-verified; CI phase can validate this output

---
*Phase: 02-seed-catalog*
*Completed: 2026-03-12*

## Self-Check: PASSED

All 13 prompt files confirmed present on disk. Both task commits (f6508bd, 6c54215) confirmed in git log.
