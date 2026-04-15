# Phase 6 — Validation & Regression

**Status:** Not started. Blocked on Phases 1-5 landing.

## Why

Each phase changes skill behavior. Without a validation pass, we don't know whether the changes produce the intended anti-bias effects on real research tasks — or whether they introduce new problems (e.g., the framing challenge being noisy, the counter-discovery pass finding junk sources, the reflection step inflating token costs).

## Goal

Run the updated skill on at least one new topic and at least one refresh of an existing topic. Compare outputs before and after the phase 1-5 changes along measurable axes. Document findings.

## Work

1. **Fresh-research test.** Pick a small topic that invites counter-perspective (e.g., "should we use microservices?"). Run the updated skill end-to-end. Observe:
   - Multi-engine search URL diversity (Phase 1).
   - Counter-perspective citations in the pool (Phase 2).
   - Reflection step behavior in writing and audit (Phase 3).
   - Framing challenge output (Phase 5).
2. **Update-workflow test.** Pick 1-2 existing topics from `~/source/cited-research/research/` — NOT `agentic-research-bias` (that's the source). Candidates: any topic >6 months old with ~30+ citations. Run the refresh path. Observe:
   - Fresh re-fetch happens (Phase 4).
   - `retraction-log.md` is written if anything is removed.
   - Drift detection triggers user prompts where appropriate.
3. **Compare:** for the refresh test, compare old deliverable vs. new deliverable. Any removed claims should have log entries. Any new claims should have fresh citations.
4. **Document findings** in `references/research-basis.md` — behavioral changes, unexpected outcomes, tuning recommendations for future phases.

## Open questions for this phase

- **Pick the test topics.** User decides at execution time — depends on which existing topics are stale enough to make a refresh meaningful.
- **Measurement rigor.** This is a qualitative review, not a controlled experiment. Document observations, don't over-claim effect sizes.

## Files to modify

- `references/research-basis.md` — validation findings section
- Possibly `SKILL.md` and sub-agent prompts if validation surfaces issues that need tuning

## Verification

- Both test runs complete without coordinator errors.
- Findings are documented.
- If any phase underperformed, the tuning required is captured as a follow-up plan — do not mix tuning into this validation phase.
