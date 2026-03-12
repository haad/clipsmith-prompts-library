---
phase: 4
slug: documentation
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-03-12
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | bash (documentation validated by content checks + human review) |
| **Config file** | none |
| **Quick run command** | `bash scripts/build.sh` |
| **Full suite command** | `bash scripts/build.sh && grep 'gh-pages/prompts.json' README.md` |
| **Estimated runtime** | ~2 seconds |

---

## Sampling Rate

- **After every task commit:** Run `bash scripts/build.sh`
- **After every plan wave:** Verify README covers all DOCS requirements
- **Before `/gsd:verify-work`:** Human review of README against all 6 DOCS requirements
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 04-01-01 | 01 | 1 | DOCS-01 | semi-auto | `grep 'gh-pages/prompts.json' README.md` | ❌ W0 | ⬜ pending |
| 04-01-02 | 01 | 1 | DOCS-02 | semi-auto | `grep -c 'title.*version' README.md` | ❌ W0 | ⬜ pending |
| 04-01-03 | 01 | 1 | DOCS-03 | semi-auto | `grep -i 'version bump' README.md` | ❌ W0 | ⬜ pending |
| 04-01-04 | 01 | 1 | DOCS-04 | semi-auto | `grep '{{clipboard}}' README.md` | ❌ W0 | ⬜ pending |
| 04-01-05 | 01 | 1 | DOCS-05 | semi-auto | `grep -c 'add-error-handling\|code-review-swift' README.md` | ❌ W0 | ⬜ pending |
| 04-01-06 | 01 | 1 | DOCS-06 | semi-auto | `grep -i 'versioning rules' README.md` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `README.md` — the sole deliverable of this phase; does not exist yet

*Existing build infrastructure (`scripts/build.sh`) covers pipeline validation.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| README is clear to new contributors | DOCS-01 | Readability is subjective | Read README as if unfamiliar with the repo |
| Step-by-step guide is complete | DOCS-02 | Procedural correctness requires human judgment | Follow the guide to add a test prompt |
| Version bump warning is prominent | DOCS-03 | Emphasis/placement is a design decision | Check that warning stands out visually |
| Prompt catalog table is accurate | DOCS-05 | Cross-reference with actual prompt files | Compare table entries to `prompts/` directory |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 2s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
