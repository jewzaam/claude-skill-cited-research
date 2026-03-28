---
name: cited-research
description: >
  Produces citation-backed research documents with independent verification.
  Every claim traces to a web source visited in-session, and isolated sub-agents
  audit the output. Supports both new research and updating existing research
  topics (re-researching stale dimensions, adding new dimensions, refreshing
  citations). Use this skill whenever the user asks for research, analysis,
  comparisons, rubrics, updates to existing research, or any deliverable that
  must be grounded in cited sources. Trigger on phrases like "find out",
  "compare", "is X better than Y", "what are the tradeoffs", "give me the real
  numbers", "cite your sources", "verify this", "what does the research say",
  "update the research", "refresh this research", or any task where unsupported
  claims would undermine trust. Always use for non-code research requiring
  factual grounding, even if the user doesn't explicitly request citations.
---

# Cited Research

A methodology for producing factual, citation-backed documents where every claim
traces to a verifiable web source. The core principle: **nothing is true until a
source says it is, and a separate reviewer confirms the source actually says it.**

This skill exists because LLMs hallucinate. The methodology makes hallucination
structurally difficult by separating research, writing, and verification into
distinct phases — and by making the writing phase aware that verification follows.

## Output Location

Before starting any research, determine where the output will be written.

### Detecting the `cited-research` repo

Check whether the current working directory is inside a git repository whose
root directory is named `cited-research`. Use `git rev-parse --show-toplevel`
and inspect the basename of the result.

- **Inside `cited-research`**: This is the dedicated research monorepo. Use
  the repo root as the output location. Each research topic gets its own
  directory under `research/` (e.g., `research/queue-system-comparison/`,
  `research/clarinet-vs-french-horn/`). The topic directory name should be a
  short, kebab-case slug derived from the research subject.

- **Not inside `cited-research`**: Default to the current repo root as
  `<root>`. Use the same `research/<topic-slug>/` structure. If the user
  wants a different location, they will say so.

### Repo-level files (cited-research only)

The `cited-research` repo maintains two root-level files:

- **`README.md`** — Explains what the repository is (a monorepo of
  citation-backed research documents) and how it is organized. This is NOT an
  index of topics. Create it on the first research run if it does not exist.

- **`index.md`** — A searchable index of all research topics. Each entry is
  a record that both humans and LLMs can scan quickly. See
  [Phase 5: Index Maintenance](#phase-5-index-maintenance-cited-research-repo-only)
  for format and update rules.

## Updating Existing Research

When the user wants to update an existing research topic rather than create a new
one, read `references/update-workflow.md` and follow its steps. The update
workflow replaces Phase 0 and then feeds into Phases 1-5.

---

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

Once dimensions are approved, define the output file tree. The structure
depends on where the output is going.

The topic directory structure is the same regardless of output location:

```
research/<topic-slug>/
├── README.md                     # Short standalone summary (TL;DR + key tables)
├── <deliverable>.md              # Full analysis with methodology
├── citations.md                  # All sources, numbered
├── references/
│   ├── <dimension-1>.md          # One file per approved dimension
│   └── ...
└── audit/
    ├── citation-audit.md
    └── consistency-review.md
```

The README inside each topic directory is written last as a standalone
decision-making tool.

If a topic directory already exists (e.g., a prior research run on the same
subject), ask the user whether to revise the existing topic in place or
create a new directory with a disambiguating slug.

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

4. **Record everything immediately.** Research agents must include every URL,
   claim, and exact source wording in their structured response. The main
   thread cannot recover data that agents omit from their output.

5. **Acknowledge gaps.** If a data point cannot be found after 3+ distinct
   queries, state explicitly that this data point is unavailable. Do not invent
   a plausible number.

6. **Welcome bonus sources.** Research agents often discover relevant sources
   not in the plan. Encourage this — unanticipated sources frequently strengthen
   the deliverable.

### Coordinator Protocol

Sub-agents cannot use WebFetch (no approval prompt in background) or Write
(not allowlisted for arbitrary paths). The main thread handles all URL fetching
and file writes. This preserves the security boundary where the user sees every
outbound fetch before it happens.

The research phase uses an iterative coordinator loop, capped at 3 iterations:

```
for each iteration (max 3):
    1. Main thread dispatches agents with available context
    2. Agents return structured results (findings + fetch requests)
    3. Main thread WebFetches requested URLs (user approves)
    4. If agents reported confidence > 0.8 with no new URLs: stop
    5. Otherwise: feed fetched content back to agents for next iteration
```

**Iteration 1 — Discovery:**
- Dispatch one `general-purpose` sub-agent per dimension with
  `run_in_background: true`
- Agent tools: WebSearch only (already works in background)
- Agent returns: URL manifest, preliminary findings from search snippets,
  confidence score, open questions
- Use the **Research Agent — Discovery** template from
  `references/sub-agent-prompts.md`
- Main thread collects all URL manifests across agents

**Iteration 2 — Deep read:**
- Main thread batch-fetches all URLs from all manifests via WebFetch
- When fetching fails, attempt WebSearch fallbacks before passing results
  to agents — handle the 20-30% inaccessibility expectation at this layer
- Dispatch agents again with fetched page content injected into the prompt
- Use the **Research Agent — Analysis** template from
  `references/sub-agent-prompts.md`
- Agent returns: extracted data with citations, follow-up URL requests
  (if any), updated confidence score

**Iteration 3 — Gap-fill (conditional):**
- Only runs if any agent reported confidence < 0.8 or requested follow-up
  URLs after iteration 2
- Main thread fetches follow-up URLs
- Agents process remaining content, finalize findings
- No further iterations regardless of confidence

### Providing Fetched Content to Agents

When the main thread fetches URLs for iteration 2+ or for the citation audit,
write each page's extracted text to a temp directory at
`/tmp/cited-research/<topic-slug>/` and pass the directory path to the agent
prompt. Create the directory if it does not exist. This avoids bloating agent
prompts with raw page content and lets agents read selectively via the Read
tool. The temp directory is ephemeral — the OS handles cleanup.

Each fetched file should have a header identifying the source:

```
# Fetched: <URL>
# Date: <fetch timestamp>
# Status: OK | FAILED (<reason>)

<extracted text content>
```

For failed fetches, still create the file with the FAILED status so agents
can see which URLs were inaccessible and report accordingly.

### Convergence Criteria

Stop iterating when:
1. All agents report confidence > 0.8 **and** no agent requested follow-up
   URLs, OR
2. Iteration 3 completes (hard cap regardless of confidence)

If any agent reports confidence < 0.5 after iteration 2, flag the dimension
to the user as potentially under-sourced before proceeding to iteration 3.

### Structuring Research Agents

See `references/sub-agent-prompts.md` for two research agent prompt variants:
**Discovery** (iteration 1) and **Analysis** (iteration 2+).

The accountability line in both templates ("a citation audit agent will
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
content mismatches). The main thread handles WebSearch fallbacks when URLs fail
before passing results to agents. Plan for it:

- Provide 2-3 candidate sources per data point when possible
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

### Building `references/<topic>.md`

Each reference file should:
- State what dimension it covers
- Link to [`citations.md`](../citations.md) for source details
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

7. **Use relative links between files.** When referencing another file in the
   topic directory (e.g., citations.md, a reference file, the README), use a
   markdown link (`[citations](citations.md)`) rather than just naming the
   file. This makes the documents navigable in any markdown viewer or when
   rendered as HTML.

### Writing the README

The README is written last. It distills the deliverable into:

- One-paragraph summary of what the document answers
- The key table or result
- A quick decision framework (3-5 steps)
- Links to the supporting files for full methodology

It should stand alone — a reader who never opens the supporting files still gets an
actionable answer.

## Phase 4: Verification

After all files are written, launch **two independent review sub-agents in
parallel**. These agents receive NO context from the research conversation —
they read only the produced files. This isolation prevents confirmation bias.

### Pre-Fetch for Citation Audit

Before dispatching the Citation Audit agent, the main thread pre-fetches all
cited URLs so the audit agent does not need WebFetch:

1. Read `citations.md` and extract every cited URL
2. Batch-fetch all URLs via WebFetch (the user approves once per batch)
3. For URLs that fail, attempt WebSearch fallbacks from the main thread
4. Write fetched content to `/tmp/cited-research/<topic-slug>/` files (same format as Phase 1)
5. Dispatch the Citation Audit agent with the `/tmp/cited-research/<topic-slug>/` directory path —
   the agent reads files via Read tool, it does not fetch URLs itself

The audit agent receives **only** Read and Glob tools — do not give it WebFetch
or WebSearch. All web access happens in the main thread before the agent runs.
This ensures the audit runs against the same source content snapshot and avoids
the sub-agent permission boundary.

The Consistency Review agent also needs only Read and Glob.

See `references/sub-agent-prompts.md` for both sub-agent prompt templates
(Citation Audit and Consistency Review).

### Handling Verification Results

After both sub-agents complete:

1. For INACCURATE or NOT FOUND citations: correct the claim in all files to
   match what the source actually says, or remove the claim and note the gap.
2. For INACCESSIBLE sources: the main thread attempts alternative WebSearch
   queries and re-fetches. If unsuccessful, downgrade the claim to "unverified"
   with a note.
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

## Phase 5: Index Maintenance (cited-research repo only)

Skip this phase when output is not in the `cited-research` repo.

After Phase 4 verification is complete and all corrections are applied, update
`index.md` at the repo root. If the file does not exist, create it with a
brief header explaining its purpose.

### Index entry format

Each entry is a block that can be parsed by both humans scanning the page and
an LLM searching for relevant prior research:

```markdown
## <Topic Title>

- **Path:** `research/<topic-slug>/`
- **Summary:** <One or two sentences describing what was researched and the key finding or conclusion.>
- **Dimensions:** <comma-separated list of the research dimensions covered>
- **Last revised:** <YYYY-MM-DD>
- **Status:** Active
```

- **Topic Title** matches the README title inside the topic directory.
- **Summary** should be specific enough that someone can decide whether to
  read further without opening the topic directory.
- **Dimensions** are the approved dimensions from Phase 0. These are the
  primary search surface for discovering relevant prior work.
- **Last revised** is the date when this phase runs. Use the output of
  `date +%Y-%m-%d` — precision beyond the day is not important.
  Update this field on every revision to the topic.
- **Status** is `Active` for current research. If a topic is later superseded
  or abandoned, update to `Superseded by <other-slug>` or `Archived`.

Append new entries at the end of the file. Do not re-sort — chronological
order by creation is the simplest convention and git history tracks everything
else.
