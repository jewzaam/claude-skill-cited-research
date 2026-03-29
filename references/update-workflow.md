# Updating Existing Research

When the user wants to update an existing research topic rather than create a new
one, follow this workflow. It replaces Phase 0 and then feeds into Phases 1-5.

## Step 1: Identify the Target Topic

Determine whether you are in the `cited-research` repo using the detection
method from SKILL.md (last path component of `git rev-parse --show-toplevel`
equals exactly `cited-research`).

If inside the `cited-research` repo, read `index.md` to get the list of
existing topics. Ask the user which topic to update — they can specify by:
- Directory path (e.g., `research/security-skills/`)
- Topic name as it appears in `index.md`
- A description that you match against index entries

If not in the `cited-research` repo, ask the user for the path to the research
directory to update.

## Step 2: Read Existing Research State

Read the target topic's key files to understand what exists:
1. `README.md` — current summary and decision framework
2. `<deliverable>.md` — full analysis
3. `citations.md` — all existing sources
4. `references/*.md` — dimension files (scan filenames and read headers)
5. `audit/` — prior verification results

Build a mental model of: what dimensions were covered, what conclusions were
reached, how old the sources are, and what the audit found last time. Unresolved
audit findings (issues without `**Status: RESOLVED**`) are strong candidates for
re-research in Step 4.

## Step 3: Scan for Adjacent Research

Read `index.md` and identify topics whose dimensions overlap with or relate to
the target topic. For each adjacent topic, read its `README.md` to understand
whether it contains context that could inform the update.

Present adjacent topics to the user with a brief explanation of what
cross-pollination each might offer. Examples:
- "The security-skills research covers dependency risk tooling that may have
  updated capabilities relevant to otel's dependency surface"
- "The engineering-estimate research has updated RCT data on AI productivity
  that could refine the mono-vs-micro agent capability assumptions"

This step provides **input context only** — the adjacent research informs what
dimensions to consider for the update but does not get modified during this run.

## Step 4: Propose Update Scope

Present the user with:

1. **Existing dimensions** — list each with a recommendation:
   - **Re-research**: Sources are stale, landscape has changed, or prior audit
     found issues
   - **Refresh citations**: Spot-check existing sources for accessibility and
     accuracy; add new sources if the field has evolved
   - **Keep as-is**: Still accurate and current
2. **New dimensions** — informed by adjacent research or by developments since
   the original research was conducted. Explain why each adds value.
3. **Dimensions to retire** — if any original dimensions are no longer relevant

Wait for user approval before proceeding.

## Step 5: Execute Update

For approved dimensions, proceed through Phases 1-5 with these modifications:

- **Phase 1 (Research):** Use the coordinator protocol (discovery → deep read
  → gap-fill iterations) for all approved dimensions. For re-research and new
  dimensions, use the standard discovery/analysis agent prompts. For
  refresh-citations dimensions, scope the discovery agent to: verify
  accessibility of existing sources, check whether cited data has been updated
  or superseded, and search for recent developments.
- **Phase 2 (Organization):** Update `citations.md` incrementally — append new
  sources with the next sequential number. Do not renumber existing citations.
  Update affected `references/*.md` files. Create new reference files for new
  dimensions.
- **Phase 3 (Writing):** Rewrite the deliverable and README to incorporate
  updated findings. Existing conclusions that are still supported by evidence
  should be retained, not rewritten for the sake of rewriting.
- **Phase 4 (Verification):** Pre-fetch all cited URLs and run both audit
  sub-agents against the full updated topic directory, not just the changed
  files. The auditors need the complete picture to check consistency.
- **Phase 5 (Index):** Update the `Last revised` date and `Dimensions` list
  in `index.md`. Update the `Summary` if conclusions changed.

## Step 6: Flag Adjacent Impact

After the update is complete, review whether any findings materially affect
adjacent research topics identified in Step 3. If so, report this to the user:

- Which adjacent topic is affected
- What specific finding creates the impact
- Whether the impact changes conclusions or just adds supporting data

**Do not update adjacent topics in this run.** Each adjacent topic update
should be a separate invocation of the cited-research skill so that it receives
full attention, full verification, and a clean context window. The user
decides whether and when to run those updates.
