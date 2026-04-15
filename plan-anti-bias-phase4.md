# Phase 4 — Update Workflow: Fresh Re-fetch + Retracted-claim Ledger

**Status:** Not started. Tagged for future work. Independent of other phases, though Phase 1 (multi-engine search) would benefit the refresh-citations path.

## Why

The current update workflow trusts cached extractions. Over time, sources 404, drift, or get retracted — per `agentic-research-bias` Dim 3, 90% of retracted papers continue being cited. Refreshing research without re-fetching the underlying sources is citation laundering. This phase makes update runs fetch fresh and auto-remove unsupportable claims with a traceability log.

## Goal

On an update run, re-fetch every cited URL, reconcile with newly discovered URLs, auto-remove claims that lose all citation support, and write a `retraction-log.md` per topic documenting what was removed and why.

## Locked-in decisions

- **No manual retraction lookups.** Every citation has a URL; re-fetching it is the check.
- **Reconcile new + existing URLs.** During update, the discovery step produces new candidate URLs; combine with existing citation URLs, dedupe, fetch the full set. Expensive but correct.
- **Auto-remove unsupported claims.** The skill is citation-based. A claim with no surviving citation is removed in the updated deliverable. Do not attempt to re-research a citation for an existing claim — that reintroduces a "this claim must be true" bias.
- **Write a retraction ledger.** New file per topic (e.g., `retraction-log.md`) with entries: timestamp, removed claim text, citation that failed, reason (404 / content drift / explicit retraction). Git history is a secondary audit trail.

## Work

1. Update `references/update-workflow.md` — add the full re-fetch step. Sequence: identify update scope → run discovery for new sources → reconcile URL sets → fetch all URLs (new and existing) → run audit → remove unsupported claims → write `retraction-log.md`.
2. Update `SKILL.md` update-workflow section.
3. Define `retraction-log.md` format in `references/citation-format.md`. Example entry:
   ```markdown
   ## 2026-05-01T14:23:00Z
   - **Removed claim:** "LLMs amplify citation popularity by +1,326 median."
   - **Failed citation:** [6] Smith et al., 2024, https://example.com/paper
   - **Reason:** URL returns 404 as of re-fetch. No alternate source found in updated discovery pass.
   ```
4. Update audit agent instructions to flag content-drift cases distinctly from 404/retraction — the coordinator decides removal based on drift severity.
5. Document in `references/research-basis.md`.

## Open questions for this phase

- **Drift threshold.** Content drift is a spectrum. What level of drift triggers claim removal vs. a flag for user review? Start conservative: any measurable drift in the data point cited triggers user prompt; removal only on explicit user confirm or on 404/retraction.
- **Re-fetch scope on a partial update.** If the user chose `refresh-citations` for only some dimensions, does the re-fetch cover only those dimensions' citations, or all citations? Probably only the targeted dimensions — document this.
- **Expensive fetch budget.** A topic with 70 citations + 50 new candidates = 120 fetches per refresh. Acceptable or needs batching?

## Files to modify

- `references/update-workflow.md` — add re-fetch step
- `SKILL.md` — update-workflow section
- `references/citation-format.md` — retraction ledger format
- `references/sub-agent-prompts.md` — audit agent handling of drift vs. 404
- `references/research-basis.md` — new section

## Verification

- Pick an existing topic with citations known to be >1 year old. Run the update workflow. Confirm re-fetch happens, drift is detected, at least one `retraction-log.md` entry is written if anything is removed.
- Confirm git history shows the removed claims (secondary audit trail intact).
- Confirm the deliverable is internally consistent after claim removal (no dangling citation references, no orphan sentences).
