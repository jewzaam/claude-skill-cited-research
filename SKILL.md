---
name: cited-research
description: >
  Produces citation-backed research documents with independent verification.
  Every claim traces to a web source visited in-session, and isolated sub-agents
  audit the output. Use this skill whenever the user asks for research, analysis,
  comparisons, rubrics, or any deliverable that must be grounded in cited sources.
  Trigger on phrases like "find out", "compare", "is X better than Y", "what are
  the tradeoffs", "give me the real numbers", "cite your sources", "verify this",
  "what does the research say", or any task where unsupported claims would
  undermine trust. Always use for non-code research requiring factual grounding,
  even if the user doesn't explicitly request citations.
---

# Cited Research

A methodology for producing factual, citation-backed documents where every claim
traces to a verifiable web source. The core principle: **nothing is true until a
source says it is, and a separate reviewer confirms the source actually says it.**

This skill exists because LLMs hallucinate. The methodology makes hallucination
structurally difficult by separating research, writing, and verification into
distinct phases — and by making the writing phase aware that verification follows.

## Phase 0: Plan Mode (Required)

Enter plan mode before doing any research. The plan is the contract between you
and the user about what will be researched and how.

### Step 1: Dimension Discovery

Decompose the user's request into research dimensions before any WebSearch calls.
Present these to the user and wait for approval.

Include both directly requested dimensions and recommended additions that would
strengthen the analysis. Explain why each recommended dimension adds value. The
user may add dimensions you didn't consider or remove ones they find out of scope.

### Step 2: Plan the File Structure

Once dimensions are approved, define the output file tree:

```
<project-root>/
├── README.md                     # Short standalone summary (TL;DR + key tables)
└── docs/
    ├── <deliverable>.md          # Full analysis with methodology
    ├── citations.md              # All sources, numbered
    ├── reference/
    │   ├── <dimension-1>.md      # One file per approved dimension
    │   └── ...
    └── audit/
        ├── citation-audit.md
        └── consistency-review.md
```

Use a `docs/` directory from the start — projects outgrow flat structures.
The README is written last as a standalone decision-making tool.

### Step 3: Plan Data Points as Hypotheses

List expected data points and candidate sources, but treat them as **hypotheses
to verify**, not commitments. Research agents must report what they actually find,
even when it contradicts the plan. Dropped or corrected claims are a sign the
methodology is working, not failing.

### Step 4: Get Plan Approval

The plan should include: dimensions, file structure, the deliverable's intended
structure, and a note that two independent review agents will audit the output.
Exit plan mode only after the user approves.

## Phase 1: Research

### Principles

1. **Web sources only.** Every number, measurement, date, or factual statement
   must come from a URL visited in-session via WebSearch or WebFetch.

2. **Parallel where possible.** Launch one research sub-agent per dimension (or
   group of related dimensions). Four agents in parallel take the same wall-clock
   time as one.

3. **Primary sources preferred.** Peer-reviewed papers > manufacturer specs >
   established reference sites > blogs/forums. When a secondary source quotes a
   number, try to find the original study.

4. **Record everything immediately.** Note the URL, the specific claim, and the
   exact wording from the source. Do not defer this.

5. **Acknowledge gaps.** If a data point cannot be found after 3+ distinct
   queries, state explicitly that this data point is unavailable. Do not invent
   a plausible number.

6. **Welcome bonus sources.** Research agents often discover relevant sources
   not in the plan. Encourage this — unanticipated sources frequently strengthen
   the deliverable.

### Structuring Research Agents

Launch one `general-purpose` sub-agent per dimension, all in parallel via
`run_in_background: true`. See `references/sub-agent-prompts.md` for the
research agent prompt template.

The accountability line in the template ("a citation audit agent will
independently verify every claim you report") is functional, not decorative.
Research agents that receive this instruction proactively flag source quality
concerns, note when data comes from secondary citations, and explicitly report
when expected data points are absent.

### Capture Provenance, Not Just Data

Look for the chain of attribution — who originally claimed what, and through
whom. "Source X reports that Person A claimed Y, validated by Z" is far more
useful than "Source X says Y." Instruct research and verification agents to
report the full attribution chain.

### Expect Source Failures

Expect **20-30% of sources to be inaccessible** (403 errors, permission denials,
content mismatches). Plan for it:

- Provide 2-3 candidate sources per data point when possible
- Instruct research agents to attempt WebSearch fallbacks when URLs fail
- Above 50% inaccessibility may indicate the topic lacks accessible web sources
  and the scope should be adjusted

When PDFs fail to extract, search for the paper's title plus the specific data
point needed, check PubMed abstracts, or look for citing secondary sources.
Never silently drop a source — record the gap visibly in the citation entry.

## Phase 2: Organization

Before writing the deliverable, organize all research into the reference files
and citations file. This forces you to confront what you actually found (vs.
what you think you found) and creates the audit trail reviewers will check.

Write `citations.md` and all reference files simultaneously — they're
independent. The deliverable depends on them, so it comes after.

### Building `citations.md`

See `references/citation-format.md` for the entry format and rules. Key
principles: number sequentially, include the specific data extracted (not just
"useful article"), flag source quality concerns, and mark retracted sources
rather than deleting them (keeps citation numbers stable).

### Building `reference/<topic>.md`

Each reference file should:
- State what dimension it covers
- Link to `../citations.md` for source details
- Present data in tables where possible (easier to audit)
- Quote sources directly when precision matters
- Cite every fact with `[N]`
- End with a "Gaps and Limitations" section
- Flag interpolated or estimated values with "(est.)"

### Calculation Discipline

- Carry at least 2 significant digits through calculations
- Show your math: "2.7 kg ÷ 0.73 kg = 3.70×"
- Round consistently toward the conservative/safe side
- Mark every interpolated or derived value with "(est.)"

## Phase 3: Writing the Deliverable

### The Accountability Clause

Before writing, internalize this: **Two independent review agents will audit
this document.** One visits every cited URL. The other checks numerical and
logical consistency across all files. This is not a threat — it is the
methodology. The review agents catch errors before the user relies on the output.

### Writing Rules

1. **Every factual claim gets a citation number.** If you cannot cite it, find
   a source or explicitly mark it as inference/estimate with reasoning shown.

2. **Distinguish data from inference.** Show calculations and cite both inputs
   for derived values. Label clearly: "Calculated from [3] and [7]."

3. **Qualify uncertainty precisely:**
   - "Source [3] reports X" — verified single-source fact
   - "Based on [3] and [7], approximately X" — derived estimate
   - "No published data found; [3] suggests Y as a proxy" — acknowledged gap
   - "~3.3 kPa (est.)" — interpolated, flagged

4. **Do not round aggressively.** If the source says 2.7, write 2.7, not
   "about 3." Rounding obscures precision and makes audit harder.

5. **State limitations prominently.** The user trusts you more when you show
   what you don't know.

6. **Cross-file consistency is your responsibility.** Verify every number in
   the deliverable matches the corresponding reference file before finalizing.

### Writing the README

The README is written last. It distills the deliverable into:

- One-paragraph summary of what the document answers
- The key table or result
- A quick decision framework (3-5 steps)
- Links to `docs/` for full methodology

It should stand alone — a reader who never opens `docs/` still gets an
actionable answer.

## Phase 4: Verification

After all files are written, launch **two independent review sub-agents in
parallel**. These agents receive NO context from the research conversation —
they read only the produced files. This isolation prevents confirmation bias.

See `references/sub-agent-prompts.md` for both sub-agent prompt templates
(Citation Audit and Consistency Review).

### Handling Verification Results

After both sub-agents complete:

1. For INACCURATE or NOT FOUND citations: correct the claim in all files to
   match what the source actually says, or remove the claim and note the gap.
2. For INACCESSIBLE sources: attempt alternative searches. If unsuccessful,
   downgrade the claim to "unverified" with a note.
3. For FAIL consistency checks: reconcile across all files — fix every file
   that references the incorrect value.
4. If corrections are significant (>3 items), re-run the affected sub-agent.
5. **Update audit reports after fixes.** Add `**Status: RESOLVED**` to each
   fixed issue. Audit reports describing fixed issues as still open confuse
   future readers.

Present the verification summary to the user along with the deliverable.

### Resolving Inaccessible Sources — Ask the User

When sources remain inaccessible after fallbacks, ask the user for help before
marking claims as permanently unverified. Users often have cached copies,
institutional access, or browser bookmarks. Prioritize asking about sources
with quantitative claims that feed into calculations or conclusions.

### Re-Verification on Revisit

Source accessibility changes over time. When revisiting a project, re-check all
INACCESSIBLE and UNVERIFIED sources. Update citation entries, audit reports,
and summary counts for any newly accessible sources. Propagate any new
discoveries to the relevant reference files.

## Adapting to Scale

Scale the methodology proportionally to the task:

- **Medium research (3-5 dimensions):** Full methodology. Sub-agents can
  spot-check rather than exhaustively audit every citation.
- **Major research (6+ dimensions, decision-critical):** Full methodology with
  exhaustive audit. Consider having the user review reference files before
  writing the summary.

The non-negotiable elements at every scale:
1. Claims come from web sources visited in-session
2. URLs are recorded
3. Writing and verification are separate steps
4. The writer knows verification will happen
