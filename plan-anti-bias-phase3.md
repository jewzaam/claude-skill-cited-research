# Phase 3 — Self-reflection Intervention

**Status:** Not started. Tagged for future work. Independent of other phases.

## Why

LLM self-correction fails 64.5% of the time, but a simple reflection prompt ("wait, reconsider") reduces that blind-spot rate by 89.3% per `agentic-research-bias` citation [22]. It's a cheap, prompt-level change with large evidence for impact.

## Goal

Add an explicit reconsider step at the tail end of writing and verification sub-agents. Keep it to a single reflection turn — preserve the "fast thinking" design basis (citation [14] of the skill's research basis, Search-R1).

## Locked-in decisions

- **Placement: end of work.** Shape: "do initial work → reflect on output → revise based on reflection → finalize."
- **Exact wording is tunable.** Iterate once a draft is in place.
- **One pass.** Not a chain-of-thought expansion. A single reconsider step.

## Work

1. Update `SKILL.md` Phase 3 writing instructions — add reflection step at the end of draft assembly.
2. Update Citation-Audit template in `references/sub-agent-prompts.md` — reflection before returning grades.
3. Update Consistency-Review template in `references/sub-agent-prompts.md` — reflection before returning findings.
4. Consider adding reflection to Research-Analysis too — lower priority, only if the writing/audit reflection lands cleanly.
5. Document in `references/research-basis.md` with citation [22].

## Open questions for this phase

- **Wording.** A draft phrasing to start with: "Before finalizing, reconsider your output. Is there anything you overlooked, a claim you stated with more confidence than the source supports, or an alternative interpretation you dismissed? Revise accordingly, then return the final output." Iterate.
- **Does reflection double the token budget?** If so, quantify and decide whether it's acceptable per agent role.

## Files to modify

- `SKILL.md` — Phase 3 writing instructions
- `references/sub-agent-prompts.md` — Citation-Audit, Consistency-Review (and possibly Research-Analysis)
- `references/research-basis.md` — new section

## Verification

- Run an audit agent with and without reflection on the same `/tmp/cited-research/<slug>/` directory. Compare the grade distribution — reflection should catch more PARTIAL cases that a single-pass agent missed.
- Confirm the reflection adds one turn per agent, not a full CoT expansion.
