---
phase: 03
slug: ci-cd-and-deployment
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-12
---

# Phase 03 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | bash + manual curl/verification (no unit test framework — zero-dependency constraint) |
| **Config file** | none — tests are workflow verification steps |
| **Quick run command** | `bash scripts/build.sh && echo "Build OK"` |
| **Full suite command** | `bash scripts/build.sh && jq -e '.version and (.prompts | length > 0)' prompts.json` |
| **Estimated runtime** | ~3 seconds |

---

## Sampling Rate

- **After every task commit:** Run `bash scripts/build.sh && echo "Build OK"`
- **After every plan wave:** Run `bash scripts/build.sh && jq -e '.version and (.prompts | length > 0)' prompts.json`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 3 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 03-01-01 | 01 | 1 | CICD-03, CICD-05 | smoke | `bash scripts/build.sh` | ✅ | ⬜ pending |
| 03-01-02 | 01 | 1 | CICD-01, CICD-02, CICD-04 | integration | `ls .github/workflows/deploy.yml` | ❌ W0 | ⬜ pending |
| 03-02-01 | 02 | 2 | CICD-04 | smoke | `test -f index.html` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `.github/workflows/deploy.yml` — the workflow file (primary deliverable)
- [ ] `index.html` — static landing page at repo root (primary deliverable)
- [ ] Manual verification: GitHub Pages enabled in repository Settings after first push

*build.sh and schema/prompt.schema.json already exist — no framework install needed.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Push to main triggers workflow | CICD-01 | Requires actual GitHub push | Push a change to prompts/, verify Actions tab shows running workflow |
| workflow_dispatch triggers workflow | CICD-02 | Requires GitHub UI interaction | Go to Actions > deploy workflow > Run workflow button |
| gh-pages branch deployed and fetchable | CICD-03 | Requires deployed environment | `curl -s https://raw.githubusercontent.com/generalarcade/flycut-prompts/gh-pages/prompts.json \| jq .version` |
| GitHub Pages manually enabled | CICD-03 | Repository Settings change | Settings > Pages > Source: Deploy from branch > gh-pages |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 3s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
