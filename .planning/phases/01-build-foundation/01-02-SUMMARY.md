---
phase: 01-build-foundation
plan: 02
subsystem: infra
tags: [bash, jq, build-script, json, yaml, macos-compat]

# Dependency graph
requires:
  - 01-01 (catalog.yaml, prompts/coding/code-review-swift.md, schema/prompt.schema.json)
provides:
  - scripts/build.sh — executable bash 3.2-compatible build script
  - prompts.json — compiled catalog output at repo root
affects:
  - 01-03 (CI uses scripts/build.sh and validates prompts.json against schema)

# Tech tracking
tech-stack:
  added: [jq, bash-3.2-compat-patterns]
  patterns:
    - POSIX string accumulation for duplicate ID detection (sort|uniq -d, no declare -A)
    - jq --arg/--argjson for safe JSON encoding of all content
    - Two-pass build (collect IDs then build JSON)
    - || true guard on optional/possibly-absent grep fields under set -euo pipefail
    - sed '/./,$!d' to strip leading blank lines from body

key-files:
  created:
    - scripts/build.sh
    - prompts.json
  modified: []

key-decisions:
  - "|| true required on all grep field extractions under set -euo pipefail — grep exits non-zero when field absent"
  - "Comments must not contain literal 'declare -A' text — bash 3.2 safety grep catches it"
  - "printf '%s' (not echo) used for body piping to jq -Rs to avoid spurious trailing newline"

# Metrics
duration: 3min
completed: 2026-03-11
---

# Phase 1 Plan 02: Build Script Summary

**Zero-dependency bash 3.2-compatible build script (scripts/build.sh) that compiles prompts/*.md into prompts.json with full duplicate/validation error detection using POSIX string accumulation and jq for JSON encoding**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-11T21:50:13Z
- **Completed:** 2026-03-11T21:52:46Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- Created `scripts/build.sh` — executable, bash 3.2-compatible, zero external dependencies beyond jq
- Two-pass design: pass 1 collects IDs and detects duplicates via `sort | uniq -d` (no `declare -A`); pass 2 parses frontmatter and builds JSON
- jq `--arg`/`--argjson` for all JSON construction — handles newlines, quotes, backslashes safely
- Body stripping via `sed '/./,$!d'` removes leading blank lines
- `|| true` guards on title/version grep prevent premature exit under `set -euo pipefail`
- Category validation against catalog.yaml allowlist with frontmatter override support
- Colon-in-title preserved verbatim via two-stage sed: strip key prefix, then strip surrounding quotes
- `prompts.json` produced at repo root with version 1, one prompt (`code-review-swift`), exactly 5 fields

## Task Commits

1. **Task 1: Create scripts/build.sh** - `7c27709` (feat)
2. **Task 2: Validate error cases and edge cases** - `56068e2` (fix)

## Files Created/Modified

- `scripts/build.sh` — Executable build script; bash 3.2+, jq, sed, awk, grep; two-pass duplicate detection; full validation with named-file error messages
- `prompts.json` — Compiled catalog output: `{"version": 1, "prompts": [{"id": "code-review-swift", ...}]}`

## Decisions Made

- `|| true` required on `grep '^title:'` and `grep '^version:'` — `set -euo pipefail` causes script to silently exit if these greps find no match, suppressing the error message
- Removed literal `declare -A` from script comments — the bash 3.2 safety test (`grep -q 'declare -A'`) would false-positive on comment text
- `printf '%s'` (not `printf '%s\n'`) used when piping body to `jq -Rs '.'` — avoids capturing a spurious trailing newline in content field

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] || true guards missing on required-field grep extractions**
- **Found during:** Task 2 validation
- **Issue:** `set -euo pipefail` causes grep to exit non-zero when `title:` or `version:` field is absent in frontmatter, killing the script before the error message is printed — so `grep -q "title"` on the output found nothing
- **Fix:** Added `|| true` to title and version grep pipelines; same pattern already applied to optional `fm_category` field
- **Files modified:** `scripts/build.sh`
- **Commit:** `56068e2`

**2. [Rule 1 - Bug] Literal 'declare -A' in comment caused bash 3.2 safety check to fail**
- **Found during:** Task 2 validation (Step 1)
- **Issue:** Comment on pass 1 line read "no declare -A, bash 3.2 safe" — the safety grep `grep -q 'declare -A'` matched the comment text and reported FAIL
- **Fix:** Rewrote comment to "POSIX string accumulation — bash 3.2 safe" (no literal `declare -A` in comment text)
- **Files modified:** `scripts/build.sh`
- **Commit:** `56068e2`

## Issues Encountered

None beyond the two auto-fixed bugs above.

## User Setup Required

None.

## Next Phase Readiness

- `scripts/build.sh` is the core deliverable — all Phase 1 plans complete
- `prompts.json` at repo root is ready for schema validation in Plan 03 (CI)
- Error messages name the offending file — ready for CI failure reporting

---
*Phase: 01-build-foundation*
*Completed: 2026-03-11*
