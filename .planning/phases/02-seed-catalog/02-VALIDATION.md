---
phase: 2
slug: seed-catalog
status: draft
nyquist_compliant: false
wave_0_complete: true
created: 2026-03-11
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | bash + jq assertions (build script is the test harness) |
| **Config file** | scripts/build.sh (already exists) |
| **Quick run command** | `bash scripts/build.sh` |
| **Full suite command** | `bash scripts/build.sh && jq '.prompts | length' prompts.json` |
| **Estimated runtime** | ~2 seconds |

---

## Sampling Rate

- **After every task commit:** Run `bash scripts/build.sh`
- **After every plan wave:** Run `bash scripts/build.sh && jq '.prompts | length' prompts.json`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 02-01-01 | 01 | 1 | SEED-01 | integration | `jq '[.prompts[] | select(.category == "coding")] | length' prompts.json` (expect 8) | ✅ | ⬜ pending |
| 02-01-02 | 01 | 1 | SEED-02 | integration | `jq '[.prompts[] | select(.category == "writing")] | length' prompts.json` (expect 6) | ✅ | ⬜ pending |
| 02-01-03 | 01 | 1 | SEED-03 | integration | `jq '[.prompts[] | select(.category == "analysis")] | length' prompts.json` (expect 5) | ✅ | ⬜ pending |
| 02-01-04 | 01 | 1 | SEED-04 | integration | `jq '[.prompts[] | select(.category == "creative")] | length' prompts.json` (expect 4) | ✅ | ⬜ pending |
| 02-01-05 | 01 | 1 | SEED-05 | integration | `bash scripts/build.sh && jq '[.prompts[] | select(.version != 1)] | length' prompts.json` (expect 0) | ✅ | ⬜ pending |
| 02-01-06 | 01 | 1 | SEED-05 | unit | `jq '[.prompts[] | select(.content | contains("{{clipboard}}"))] | length' prompts.json` (expect ≥1) | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No new test infrastructure needed.

- `scripts/build.sh` validates frontmatter, rejects duplicates, checks categories
- `jq` available on macOS for JSON assertions
- `schema/prompt.schema.json` validates output structure

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Prompt content is "immediately useful" | SEED-05 | Quality is subjective | Review each prompt body for clarity, specificity, and utility |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
