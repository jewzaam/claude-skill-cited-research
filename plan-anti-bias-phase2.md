# Phase 2 — Counter-perspective Discovery

**Status:** Not started. Tagged for future work. Depends on Phase 1 being useful but not strictly blocked by it.

## Why

LLM-based discovery tends to confirm the operator's framing. Asking "are apples healthy?" returns supporting evidence; apple allergies, sugar content, and pesticide residue barely surface. Per `agentic-research-bias` Dim 3, LLM-selected citations skew to popular, agreeable sources. A dedicated counter-perspective pass during citation gathering broadens the pool before any synthesis happens.

## Goal

Dispatch a counter-discovery sub-agent alongside each Research-Discovery agent during Phase 1. Counter-sources feed into the same unified citation pool the writer sees — no tagging, no writer-side branching.

## Locked-in decisions

- **No tagging to the writer.** Counter-sources and supporting sources go into one pool. Tagging would let the writer weight them differently and reintroduce bias.
- **No hardcoded minimum threshold.** Some topics have no counter-perspective (e.g., "how does a language feature work"). The skill attempts the search; empty result is a valid outcome.
- **Null counter-result is silent** unless the agent visibly gave up mid-task.
- **User pre-flight option.** During Phase 0 dimension review, ask the user a three-option question about counter-perspective handling:
  1. Continue with whatever counter-perspectives are found (default).
  2. Stop if few/no counter-perspectives are found and surface to the user.
  3. Don't look for counter-perspectives (purely technical topics).

## Work

1. Add Counter-Discovery sub-agent template in `references/sub-agent-prompts.md`. Task-oriented framing only (no expert persona — see existing skill rule). Prompt should instruct the agent to seek: contradicting evidence, alternative interpretations, failure cases, edge-case exceptions, minority viewpoints.
2. Update `SKILL.md` Phase 1 dispatch logic — one Counter-Discovery agent per dimension alongside the existing Research-Discovery agents.
3. Update `SKILL.md` Phase 0 to include the three-option user question during dimension review.
4. Confirm the writer instructions in `SKILL.md` Phase 3 make no distinction between counter and supporting citations — all sources are equivalent in the pool.
5. Document in `references/research-basis.md` with citation [45] and Dim 3 references.

## Open questions for this phase

- **Dispatch timing.** Same iteration as Research-Discovery, or a follow-up iteration after initial sources land?
- **"Gave up" detection.** What signal does the coordinator use to decide the agent gave up vs. genuinely found nothing? Length of output? Explicit null marker in the agent's return format?

## Files to modify

- `references/sub-agent-prompts.md` — new Counter-Discovery template
- `SKILL.md` — Phase 0 user question; Phase 1 dispatch logic
- `references/research-basis.md` — new section

## Verification

- Run a Phase 0-1 pass on a loaded question ("why is X better than Y?"). Confirm the citation pool includes sources that push back on the premise.
- Run on a neutral technical topic ("how does X work?"). Confirm no counter-perspectives surface and the skill continues cleanly.
- Confirm the writer's output surfaces dissenting views as part of normal synthesis, not as a separate "counter" section.
