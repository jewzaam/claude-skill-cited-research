# Phase 5 — Dimension-picker Framing Challenge

**Status:** Not started. Tagged for future work. Independent of other phases.

## Why

The operator's question frames the entire research pipeline. Per `agentic-research-bias` Dim 6, expert anchoring has zero exceptions in tested LLMs — confirmation bias drops detection by up to 93.5pp when framing implies a conclusion. The skill currently proposes dimensions based on the user's prompt with no explicit challenge to the framing itself.

## Goal

Extend the Phase 0 dimension-proposal step so the coordinator surfaces framing assumptions for user review — not silently adjusted, not via a separate translation layer. Keep it inline.

## Locked-in decisions

- **Inline extension, not a sub-agent.** Phase 0 already asks "what dimensions haven't we considered?" Extend this to include a framing challenge: "what assumptions in the question are we accepting without examining?"
- **No distancing/translation layer.** Each abstraction adds bias. The user explicitly rejected a translator. Keep the challenge visible to the user as part of the normal dimension proposal.
- **Verbosity is tunable.** Start minimal; iterate based on how useful the output is.
- **Accepted limitation.** No mitigation eliminates expert-opinion anchoring. This phase is harm-reduction.

## Work

1. Update `SKILL.md` Phase 0 dimension-proposal prose to include a framing-challenge section. Example prompt to the coordinator's own reasoning: "Before finalizing the dimension list, state what the user's question assumes — the implicit conclusion, the frame of reference, the alternatives it excludes. Surface these to the user alongside the proposed dimensions."
2. Present the challenge output to the user as part of the Phase 0 approval gate. User can accept, reject, or modify the framing before research begins.
3. Document in `references/research-basis.md` with Dim 6 and citation [4] references.

## Open questions for this phase

- **How structured should the challenge be?** A one-line "the question assumes X" vs. a full "framing audit" section. Start one-line; expand only if a pilot run shows underwhelming results.
- **Does the challenge itself reflect the same bias?** Probably yes — the coordinator is an LLM reading the user's prompt. This phase is harm-reduction, not elimination. Document this in `research-basis.md`.

## Files to modify

- `SKILL.md` — Phase 0
- `references/research-basis.md` — new section

## Verification

- Run Phase 0 on a loaded question ("why is Rust better than C++?"). Confirm the framing challenge surfaces the implicit conclusion.
- Run Phase 0 on a neutral technical question ("how does async work in Python?"). Confirm the framing challenge is either empty or minimal — no false positives that waste user attention.
