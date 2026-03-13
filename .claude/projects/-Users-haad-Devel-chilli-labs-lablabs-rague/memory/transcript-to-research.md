# Transcript → Company Research Gap (2026-03-09)

## Problem
Transcript signals (budget, timeline, pain points, next steps, participants) extracted and approved but NOT visible in company research.

### Evidence: Lead ee4a9fa1 (Kardi AI)
- Transcript `cdc548e4` status=approved, has rich extracted_signals
- Company research `challenges` field is **null** — merge didn't persist
- Budget: "4,000 USD monthly AWS spend; ~2,000 EUR monthly for Lara platform"
- Timeline: "Decision on audit by start of next week; 1-year commitment"
- 4 pain points, 9 tech mentions, 4 next steps, 3 participants — all lost

### Root Causes

1. **Merge may have failed silently** — `merge_transcript_into_research()` in `src/workflows/sales_transcript.py` has broad exception handling. Check server logs for this lead's approval job.

2. **`challenges` bypasses the store** — Written via raw SQL (lines 187-205 in sales_transcript.py) instead of through `create_or_update_research()`, which doesn't accept a `challenges` param. This means no change tracking.

3. **`next_steps` and `participants` are dropped entirely** — The merge prompt (`prompts/sales/transcript/merge_research.jinja2`) only maps pain_points→challenges and tech_mentions→tech_stack. Next steps and participants aren't in the output schema.

## Fix Plan

### Phase 1: Add missing columns to CompanyResearch
- Add `next_steps_json` (Text, nullable) to `src/server/db/models.py`
- Add `participants_json` (Text, nullable) to `src/server/db/models.py`
- Add get/set helpers like existing `get_challenges()`/`set_challenges()`
- Include in `to_api_dict()`
- Create Alembic migration

### Phase 2: Update store
- Add `challenges`, `next_steps`, `participants` params to `create_or_update_research()` in `src/server/company_research_store.py`
- Remove raw SQL workaround from `merge_transcript_into_research()`

### Phase 3: Update merge prompt + workflow
- Update `prompts/sales/transcript/merge_research.jinja2` to output `next_steps` and `participants`
- Update `merge_transcript_into_research()` to pass all three to store

### Phase 4: Update UI
- Add `next_steps` and `participants` to TypeScript types (`web-ui/src/types.ts`)
- Add sections in `CompanyPanel.tsx` for next steps (action, owner, due date) and participants (name, role, company)

### Key Files
- `src/server/db/models.py` — CompanyResearch model (line ~756)
- `src/server/company_research_store.py` — store CRUD
- `src/workflows/sales_transcript.py` — merge logic
- `prompts/sales/transcript/merge_research.jinja2` — merge prompt
- `web-ui/src/types.ts` — TypeScript interfaces
- `web-ui/src/components/sales/CompanyPanel.tsx` — UI rendering
