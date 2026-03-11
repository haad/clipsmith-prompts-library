---
phase: 1
slug: build-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-11
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | bash + jq (build script is the test harness) |
| **Config file** | scripts/build.sh |
| **Quick run command** | `bash scripts/build.sh` |
| **Full suite command** | `bash scripts/build.sh && jq empty prompts.json && jq -e '.prompts | length > 0' prompts.json > /dev/null` |
| **Estimated runtime** | ~2 seconds |

---

## Sampling Rate

- **After every task commit:** Run `bash scripts/build.sh`
- **After every plan wave:** Run full suite command
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01-01 | 01 | 1 | REPO-01,02,03 | integration | `ls prompts/{coding,writing,analysis,creative}` | ❌ W0 | ⬜ pending |
| 1-01-02 | 01 | 1 | BILD-01,02,03 | integration | `bash scripts/build.sh` | ❌ W0 | ⬜ pending |
| 1-01-03 | 01 | 1 | PRMT-01,03,04 | integration | `bash scripts/build.sh && jq '.prompts[0].id' prompts.json` | ❌ W0 | ⬜ pending |
| 1-01-04 | 01 | 1 | BILD-05 | integration | duplicate ID test | ❌ W0 | ⬜ pending |
| 1-01-05 | 01 | 1 | BILD-06 | integration | missing field test | ❌ W0 | ⬜ pending |
| 1-01-06 | 01 | 1 | BILD-07 | integration | invalid category test | ❌ W0 | ⬜ pending |
| 1-01-07 | 01 | 1 | BILD-09,10 | integration | `bash scripts/build.sh` with edge-case prompt | ❌ W0 | ⬜ pending |
| 1-01-08 | 01 | 1 | JSON-01,02,03 | integration | `jq -e '.version and .prompts' prompts.json` | ❌ W0 | ⬜ pending |
| 1-01-09 | 01 | 1 | SCHM-01,02 | manual | review schema file | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `prompts/coding/code-review-swift.md` — test prompt with edge cases (colons, multiline)
- [ ] `catalog.yaml` — catalog metadata
- [ ] `scripts/build.sh` — build script (executable)

*Build script IS the test infrastructure — no separate test framework needed.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Schema correctness | SCHM-01,02 | Schema review requires human judgment | Inspect schema/prompt.schema.json for correct draft, patterns, enums |
| macOS bash compat | BILD-10 | Requires macOS environment | Run `bash scripts/build.sh` on macOS with default /bin/bash |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
