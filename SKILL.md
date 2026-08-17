---
name: cited-research
model: opus
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
# No `allowed-tools:` here. This skill has no `!`-injection auto-fetch
# in SKILL.md, so the frontmatter does not need to pre-approve anything
# at skill-load time. Runtime tools (Read/Glob on ./.tmp-cited-research,
# Bash invocations of bootstrap_tmp.sh / put_data.py / reap_data.py /
# multi_search.py, Write to the deliverable directory) are streamlined
# by user-level settings — see README.md §Streamlining Permissions for
# the exact entries to add to ~/.claude/settings.json.
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

Run `git rev-parse --show-toplevel`, then `basename` on the result
(two separate commands — no `$()` or backticks; command substitution
triggers extra approval prompts). If the basename equals exactly
`cited-research`, you are inside the dedicated research monorepo —
use its repo root as the output location. Otherwise default to the
current repo root. Either way, each topic lives at
`research/<topic-slug>/` using a kebab-case slug derived from the
subject. Do not use `pwd`, the remote URL, or file presence as
detection signals — only the basename check is correct.

### Repo-level files (cited-research only)

- **`README.md`** — explains the repository (a monorepo of
  citation-backed research documents). NOT a topic index. Create on
  first run if missing.
- **`index.md`** — searchable topic index. See
  [Phase 5: Index Maintenance](#phase-5-index-maintenance-cited-research-repo-only).

## Updating Existing Research

When the user wants to update an existing research topic rather than create a new
one, read `references/update-workflow.md` and follow its steps. The update
workflow replaces Phase 0 and then feeds into Phases 1-5.

---

## Phase 0: Plan Mode (Required)

Enter plan mode before doing any research. The plan is the contract between you
and the user about what will be researched and how.

### Step 1: Dimension Discovery

Decompose the user's request into research dimensions before running any searches.
Present these to the user and wait for approval.

Include both directly requested dimensions and recommended additions that would
strengthen the analysis. Explain why each recommended dimension adds value. The
user may add dimensions you didn't consider or remove ones they find out of scope.

### Step 1b: Framing Challenge

When the user's request carries opinion or preference markers
("better", "best", "should we", "worth it", "recommend", "compare
and pick", "is X right for…"), read
`references/framing-challenge.md` and surface the embedded
assumptions to the user alongside the proposed dimensions. For
neutral how-does-X-work technical questions, skip this step — the
output would be empty anyway.

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

For each key data point, identify **2-3 candidate sources** rather than relying
on a single source. This is not redundancy for its own sake — 20-30% of web
sources are inaccessible at any given time due to link rot, paywalls, and AI
crawler blocking (see `references/research-basis.md` §Source Inaccessibility).
Planning for multiple candidates per data point shifts inaccessibility handling
from reactive fallback to proactive coverage.

### Step 4: Counter-perspective Handling

Before finalizing the plan, ask the user how to handle counter-perspectives
during research. Present three options:

1. **Find and include** (default) — search for counter-perspectives alongside
   supporting evidence. Whatever is found merges into the citation pool.
2. **Find and gate** — search for counter-perspectives; if few or none are
   found, pause and surface this to the user before proceeding.
3. **Skip** — do not search for counter-perspectives. Appropriate for purely
   technical topics (e.g., "how does X work?") where counter-perspectives
   are unlikely to exist.

The user's choice governs whether Counter-Discovery agents are dispatched
in Phase 1 and how null results are handled.

### Step 5: Get Plan Approval

Include in the plan: dimensions, file structure, the deliverable's
intended structure, counter-perspective handling choice, and a note
that two independent review agents will audit the output.
Exit plan mode only after the user approves.

## Phase 1: Research

### Principles

1. **Web sources only, and search is not a source.** Every number,
   measurement, date, or factual statement must come from the text of a page
   fetched in-session — via `WebFetch`, or `scripts/fetch_url.py` when
   `WebFetch` is unavailable (see §Preflight under Coordinator Protocol).

   Search is for *finding* pages, not for sourcing claims. Search results may
   contribute a URL and a verbatim snippet quote. They may never contribute a
   claim.

2. **Do not use WebSearch. Search runs through `scripts/multi_search.py`.**
   This is a hard rule, not a preference.

   WebSearch returns a model-synthesized answer alongside its links. That
   prose is an interpretation of results, not source text, and it is a bias
   vector with a feedback loop: it shapes which URLs get fetched and which
   claims get written, then hides inside a citation to a page nobody read.
   It has also been observed to return *no* per-URL verbatim snippets at all,
   which leaves an agent nothing quotable and invites paraphrase.

   `multi_search.py` returns raw `{url, title, snippet, engine}` records
   scraped from engine result pages — the snippet is the engine's extract of
   the page, not a model's reading of it — and it carries no per-session call
   budget. It is the only sanctioned search path.

   If a run is somehow forced onto WebSearch, every claim touching those
   results must be disclosed as search-derived in the deliverable's
   Limitations section.

3. **Parallel where possible.** Launch one research sub-agent per dimension (or
   group of related dimensions). Four agents in parallel take the same wall-clock
   time as one.

4. **Primary sources preferred.** Assign quality tiers to sources and prefer
   higher tiers when conflicts arise:
   - **Tier 1:** Peer-reviewed papers, government/institutional reports
   - **Tier 2:** Manufacturer specs, established reference sites, university
     publications
   - **Tier 3:** Industry blogs, conference talks, well-known practitioners
   - **Tier 4:** Forums, personal blogs, GitHub discussions, social media

   When a secondary source quotes a number, try to find the original study.
   See `references/research-basis.md` §Source Quality and Weighting for the
   evidence behind these tiers.

5. **Consider source recency relative to topic.** For fast-moving domains
   (for example AI, cloud infrastructure, security), prefer sources published within the
   last 2 years — the landscape changes quickly enough that older findings may
   be superseded. For stable domains (for example physics, music theory, established
   engineering, mathematics), older sources are often authoritative.
   When mixing source ages, note the publication year alongside each claim so
   readers can assess currency.

6. **Record everything immediately.** Research agents must include every URL,
   claim, and exact source wording in their structured response. The main
   thread cannot recover data that agents omit from their output.

7. **Acknowledge gaps.** If a data point cannot be found after 3+ distinct
   queries, state explicitly that this data point is unavailable. Do not invent
   a plausible number.

8. **Welcome bonus sources.** Research agents often discover relevant sources
   not in the plan. Encourage this — unanticipated sources frequently strengthen
   the deliverable.

### Coordinator Protocol

Main thread owns every fetch and every file write — that's the security
boundary (the user approves the outbound fetch rule once, and every
requested URL is logged host-side). Sub-agents return structured results;
the main thread acts on them. Iterative loop, capped at 3:

```
for each iteration (max 3):
    1. Main thread dispatches agents with available context
    2. Agents return structured results (findings + fetch requests)
    3. Main thread fetches requested URLs (WebFetch, or fetch_url.py)
    4. If agents reported confidence > 0.8 with no new URLs: stop
    5. Otherwise: feed fetched content back to agents for next iteration
```

**Log every agent dispatch.** Immediately after each batch of agents
returns, append one tab-separated line per agent to the run's `agents.tsv`
via `put_data.py`:

```
<name>	<mode>	ok
<name>	<mode>	error: <short reason>
```

An agent that returned nothing, timed out, or hit a tool budget is a loss of
signal, and it is invisible afterwards unless recorded here. The run report
counts this file.

**Every script path in this file is relative to the installed skill, never
to the working directory.** The cwd during a run is the user's research
repo, which has its own `scripts/` — a bare `scripts/fetch_url.py` resolves
there and finds a different project's files. Always invoke through the
skill's own interpreter, which is also what makes `from scripts...` imports
resolve:

```
SKILL=~/.claude/skills/cited-research
$SKILL/.venv/bin/python $SKILL/scripts/<name>.py ...     # git-bash: .venv/Scripts/python.exe
```

`ModuleNotFoundError: No module named 'scripts'` means that venv exists but
the package was never installed into it — see §Prerequisite below.

**Preflight.** Before the first page read of a run, establish which
retrieval path works:

1. Try `WebFetch` on the first URL. If it succeeds, use it for the whole
   run and persist each page with `put_data.py`.
2. If it fails with `Socket is closed`, this is a sandbox. Check the
   host-side fetch service with `$SKILL/scripts/fetch_url.py --check`.
3. If that exits non-zero, stop and tell the operator. No pages can be
   read, and that must surface before agents are dispatched rather than
   as a wall of failures afterward. Do not route around it — there is no
   other egress for arbitrary hosts.
4. `--check` also reports every capability. `MISSING pypdf` exits non-zero
   too, and means the same thing in practice: PDFs are the Tier 1 half of
   most source pools, so proceeding would bias the run toward the very tier
   this method exists to avoid. Fix it before dispatching, not after. An
   `absent` optional line (OCR) is a reduced capability, not a stop — but
   report it, because scanned sources will fail.

Invocation and exit codes are in
[`references/data-persistence.md`](references/data-persistence.md); the
service itself is documented at
<https://github.com/jewzaam/openshell-sandbox/blob/main/docs/fetch-service.md>.

### Model Assignment

Each agent's frontmatter (`agents/*.md`) declares its model. The
assignments balance task type against cost — mechanical search and
verification on `sonnet`, deep extraction and synthesis on `opus`.
See `references/research-basis.md` §Model Assignment by Agent Role
for the evidence-backed rationale per role.

**Iteration 1 — Discovery (three steps, coordinator executes all search):**

Sub-agents plan queries and triage results. The coordinator runs every
query. No agent has a search tool.

*Step 1 — agents propose queries.* Dispatch one `research-discovery` agent
per dimension in MODE: propose, with DIMENSION, PROJECT_DESCRIPTION and
KNOWN_GAPS. Each returns 8–15 prioritized queries. Unless the user chose
"Skip" in Phase 0 Step 4, dispatch a `research-counter-discovery` agent per
dimension in MODE: propose alongside it, with RESEARCH_QUESTION added.

*Step 2 — coordinator runs the queries.*

First, once per session, probe which backends are alive:

```
~/.claude/skills/cited-research/.venv/bin/python \
    ~/.claude/skills/cited-research/scripts/multi_search.py --health
```

It reports each backend and prints a `suggested --backends` line. Engines
block intermittently and in clusters — a probe in August 2026 found 3 of 8
alive — so passing only the live ones avoids spending every query on dead
backends. If fewer than two are alive it exits non-zero: two independent
waves cannot be formed, and that is a stop condition (see below).

Then run each query, serially, with the live backend list:

```
~/.claude/skills/cited-research/.venv/bin/python \
    ~/.claude/skills/cited-research/scripts/multi_search.py \
    --query "..." --limit 10 --backends <live list>
```

Windows (git-bash): swap `.venv/bin/python` for `.venv/Scripts/python.exe`.

Run queries serially, not in parallel. Concurrent hits on the same engines
are the traffic shape that trips anti-bot challenges, and a challenged
engine returns a CAPTCHA page instead of results.

The script splits the backends into **two disjoint waves and queries each
separately**. This is not optional and cannot be disabled. A single combined
call fills `--limit` from whichever backend answers first — measured, one
backend alone and all eight together both returned 10 of 10 requested
results — so one call buys far less diversity than the backend list
suggests. Two waves force two independent engine populations to contribute.

Write each query's JSON output into the topic's `search/` directory via
`put_data.py`.

**Read the `coverage` block in the output.** The script returns
`{"results": [...], "coverage": {...}}`, and coverage carries:

- `single_wave: true` — only one wave answered. Usable, but not two
  independent samples. Record it in the deliverable's Limitations section.
- `returned_by_wave` vs `unique_by_wave` — how many results each wave
  returned against how many survived dedup. A wave can answer successfully
  and add nothing new. Observed: two healthy waves where the second
  contributed 0 unique URLs on one query and 2 of 6 on another.
- `waves_adding_unique` — if this has fewer than two entries, the engines
  overlapped completely and effective coverage is single-sample despite two
  waves running. The script warns on stderr; the flag is in the payload so
  it survives into agent context either way.

Do not report a run as multi-engine when coverage says otherwise.

*Step 2a — scholarly APIs (optional, per dimension).*

```
... multi_search.py --query "..." --engines ddg,scholarly --backends <live list>
```

The `scholarly` engine queries OpenAlex, Crossref and Wikipedia full-text as
keyless JSON APIs through the fetch service. They have no anti-bot layer, so
they answer on days when every ddgs scraper is blocked, and they are Tier 1-2
by construction. OpenAlex snippets are the publisher's own abstract; Crossref
often carries none at all.

Two limits, both load-bearing:

- **It is not a web index.** It finds peer-reviewed analysis, treaty text and
  statistics. It will not find a government fee schedule or a tax authority's
  rate page. Adding it does not make a dimension web-sourced.
- **It does not lift the stop condition.** `scholarly` waves count toward
  `independent_samples`, so a run with one live scraper plus `scholarly` will
  report two or more samples. That is honest about sampling and silent about
  the web: claims about current regulation still need a web engine behind
  them. Judge the ddgs backend count separately.

*Step 2b — encyclopedia sweep (optional, once per dimension).*

```
... multi_search.py --query "..." --backends wikipedia,grokipedia
```

Keep these out of the main `--backends` list: both are lookups, not indexes
(each returns one result), so in a wave they add almost nothing while
distorting the coverage metrics.

Fetch any hit as normal. `fetch_url.py` detects an encyclopedia URL and
writes its outbound citation links to a `<name>.refs` sidecar — one URL per
line, ready to feed straight back into the fetch queue:

```
... fetch_url.py --batch <slug> < .tmp-cited-research/<slug>/fetched/<name>.md.refs
```

Those references are the sources to cite. The article itself is tertiary —
see `references/citation-format.md` §Tertiary Sources.

*Step 3 — agents triage.* Re-dispatch the same agents in MODE: triage with
RESULTS_DIR pointing at the JSON. Each returns a URL manifest with tiers,
verbatim snippet quotes, follow-up queries, and a confidence score.
Counter-discovery URLs merge into the same pool — no tagging distinguishes
them. If the user chose "Find and gate" and a counter agent returns
confidence < 0.3 with no URLs, surface that before iteration 2.

The coordinator then merges all manifests and deduplicates by exact URL
before fetching.

**When search fails entirely — stop.**

`multi_search.py` exits non-zero when every backend fails. There is no
WebSearch fallback; that path is banned (Principle 2). A run that cannot
search cannot discover sources, so:

1. Report which backends failed and the error. A
   `ProxyError: 403 Forbidden` or `tunnel error` inside a sandbox means the
   search engines are missing from the network policy — a fixable
   configuration gap, not an inherent limitation. The repo README lists the
   hosts to allow.
2. **Stop and tell the operator.** Do not proceed on a thin pool of
   whatever URLs happen to be at hand, and do not silently fall back to
   WebSearch.
3. `--health` reporting fewer than two live backends is also a stop
   condition: two independent waves cannot be formed from one engine, so
   the run cannot deliver the property the methodology depends on.
4. Partial failure is different: `ddgs` fills `--limit` from whichever
   backends respond, so some engines being blocked costs cross-engine
   diversity but not result volume. That degrades; it does not halt —
   record it in Limitations.

Prerequisite: the skill's `.venv` is populated once, and this is operator
setup — do it in the skill directory, not the research repo:

```
cd ~/.claude/skills/cited-research
python3 -m venv .venv && .venv/bin/python -m pip install -e .
.venv/bin/python -m pip install -e ".[ocr]"   # optional, scanned PDFs
```

`pip install -e .` rather than a plain dependency install because the
scripts import each other as a package. Without the venv there is no search
at all — treat a missing venv the same as total backend failure.

84.9% of search results are unique to a single engine [§Multi-Engine Search
Diversity in `references/research-basis.md`], which is why the multi-engine
path is the only sanctioned one rather than a nice-to-have.

**Iteration 2 — Deep read:**
- Main thread batch-fetches all URLs from all manifests using the path
  established at preflight, writing each page into the topic's `fetched/`
  directory. See
  [`references/data-persistence.md`](references/data-persistence.md) for
  the invocation and exit-code contract
- When a fetch fails, run a targeted `multi_search.py` query for an
  alternative source before passing results to agents — handle the 20-30%
  inaccessibility expectation at this layer.
  Keep the `FAILED` file either way; do not delete it to tidy the directory
- Dispatch one `research-analysis` agent per dimension. Provide
  DIMENSION, PROJECT_DESCRIPTION, FETCHED_DIR, and DATA_TYPE in the
  invocation prompt
- Agent returns: extracted data with citations, follow-up URL requests
  (if any), updated confidence score

**Source triage (between iteration 2 and 3):**

After iteration 2, review the fetch results for high-priority sources that
failed. If any Tier 1-2 sources (peer-reviewed, institutional, government)
were inaccessible and the data they were expected to provide feeds into key
claims or calculations, present them to the user before proceeding. Users
often have institutional access, cached copies, or bookmarks that resolve
sources agents cannot reach. If the user provides content, write it to the
temp directory and include it in the next iteration's agent prompts.

See `references/research-basis.md` §Source Triage as Human Gate for evidence.

**Iteration 3 — Gap-fill (conditional):**
- Only runs if any agent reported confidence < 0.8 or requested follow-up
  URLs after iteration 2
- Main thread fetches follow-up URLs
- Agents process remaining content, finalize findings
- No further iterations regardless of confidence

### Providing Fetched Content to Agents

When the main thread fetches URLs for iteration 2+ or for the
citation audit, each page's extracted text lands under
`./.tmp-cited-research/<topic-slug>/fetched/` — written by
`fetch_url.py` directly, or by `put_data.py` when the run is using
`WebFetch`. Pass that directory to the agent prompt. Agents read the
files selectively via the Read tool — never paste page content into
a prompt.

**Bootstrap first.** Before the first fetch or `put_data.py` call
for a topic, run
`bash ~/.claude/skills/cited-research/scripts/bootstrap_tmp.sh
<topic-slug>`. The bootstrap script provisions the parent
`./.tmp-cited-research/` and its `.gitignore` of `*`, then wipes
and recreates the slug subdir. Both `fetch_url.py` and
`put_data.py` refuse to create the slug root themselves — running
either without the bootstrap returns a fail-fast error pointing at
this step. The hard failure exists so the parent `.gitignore`
protection cannot be silently bypassed, which would risk
accidentally committing fetched URLs.

Read `references/data-persistence.md` for the fetch invocation and
exit codes, the heredoc pattern used by `WebFetch`-mode pages,
operator-supplied content, and audit reports, the fetched-file header
format, and the rationale for routing every other write through
`put_data.py` rather than the Write tool.

### Convergence Criteria

Stop iterating when:
1. All agents report confidence > 0.8 **and** no agent requested follow-up
   URLs, OR
2. Iteration 3 completes (hard cap regardless of confidence)

If any agent reports confidence < 0.5 after iteration 2, flag the dimension
to the user as potentially under-sourced before proceeding to iteration 3.

### Structuring Research Agents

Agent definitions live in `agents/` — one `.md` file per role with YAML
frontmatter specifying model, tools, and background mode. The coordinator
invokes agents by name and provides dimension-specific values in the
invocation prompt.

All research agent definitions include an accountability line ("a citation audit agent will
independently verify every claim you report"). The behavioral effect of this
line on LLM output is plausible but unvalidated by published research. The
real enforcement mechanism is structural: requiring inline citations forces
a model to fabricate both a false claim AND a false citation simultaneously
(the "dual-error" principle), making fabrication harder. The accountability
line reinforces the inline citation requirement. See
`references/research-basis.md` §Accountability Clause for evidence.

### Capture Provenance, Not Just Data

Look for the chain of attribution — who originally claimed what, and through
whom. "Source X reports that Person A claimed Y, validated by Z" is far more
useful than "Source X says Y." Instruct research and verification agents to
report the full attribution chain.

### Expect Source Failures

Expect **20-30% of sources to be inaccessible** (403 errors, permission denials,
content mismatches, AI crawler blocking). The main thread runs targeted
`multi_search.py` queries for alternatives when URLs fail, before passing
results to agents. A `FAILED` fetched
file is a normal outcome, not a run error — the citation-audit agent grades
those citations `INACCESSIBLE`, which is truthful. The 2-3 candidate
sources per data point planned in Phase 0 Step 3 provide the redundancy needed
to absorb this failure rate. Above 50% inaccessibility may indicate the topic
lacks accessible web sources — adjust the scope.

PDFs are extracted like any other page, so a PDF is not itself a failure.
A scanned PDF falls back to OCR when the extra is installed; the status then
reads `OK (OCR — transcription, not verbatim source)`. **Do not quote from an
OCR body and do not lift a figure out of one** — a misread digit is a
plausible wrong number, and the citation audit reads the same file, so it
confirms the error rather than catching it. Use it to establish that a claim
is supported, then cite the original and mark the entry OCR-derived.

`FAILED (no text layer …)` means scanned with no OCR available. That source
exists and is unread — search for its title plus the specific data point
needed, check PubMed abstracts, or look for citing secondary sources. Never
silently drop a source — record the gap visibly in the citation entry.

Two statuses that look like failures but are the system working, and must be
read literally rather than retried:

- `FAILED (empty after extraction, N chars …)` — the page rendered its
  content in JavaScript. Refetching gets the same shell. Find the underlying
  document, often a PDF or an API endpoint linked from that page.
- `FAILED (403 …)` — some hosts block the fetch service by policy rather
  than intermittently. Two failures on the same host means stop spending
  retries on it; `run_report.py` ranks repeat-failing hosts for exactly this.

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

Two independent review agents will audit this document — one checks every
cited URL against source content, the other checks numerical and logical
consistency. The real protection is structural: every claim requires an inline
citation, and fabricating both a false claim and a matching false citation is
harder than fabricating either alone (the "dual-error" principle). The review
step catches what slips through.

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

5. **Surface contradictions, don't suppress them.** When sources disagree,
   state the disagreement explicitly with citations to both sides. Do not
   silently select one interpretation — the reader needs to see the conflict
   to assess which source to trust. See `references/research-basis.md`
   §Contradiction Transparency.

6. **State limitations prominently.** The user trusts you more when you show
   what you don't know.

7. **Cross-file consistency is your responsibility.** Verify every number in
   the deliverable matches the corresponding reference file before finalizing.

8. **Use relative links between files.** When referencing another file in the
   topic directory (e.g., citations.md, a reference file, the README), use a
   markdown link (`[citations](citations.md)`) rather than just naming the
   file. This makes the documents navigable in any markdown viewer or when
   rendered as HTML.

9. **Review cross-source synthesis carefully.** Claims that draw on multiple
   sources for a single conclusion are where LLMs are weakest — individual
   document analysis is strong, but narrative integration across papers is a
   documented limitation (see `references/research-basis.md` §Cross-Source
   Synthesis Limitation). Flag cross-document claims for operator review when
   they underpin key conclusions.

### Reflection Before Finalizing

After assembling the draft deliverable and before writing the README,
perform one reflection pass. Ask yourself:

> Is there anything I overlooked, a claim I stated with more confidence
> than the source supports, an alternative interpretation I dismissed, or
> a contradiction I suppressed? Revise accordingly.

This is a single pass, not a chain-of-thought expansion. LLM self-correction
fails 64.5% of the time, but a single reflection prompt reduces that
blind-spot rate by 89.3% [§Self-reflection Intervention in
`references/research-basis.md`]. Keep it to one turn to preserve the
"fast thinking" design principle.

### Writing the README

The README is written last. It distills the deliverable into:

- One-paragraph summary of what the document answers
- The key table or result
- A quick decision framework (3-5 steps)
- Links to the supporting files for full methodology

It should stand alone — a reader who never opens the supporting files still gets an
actionable answer.

## Phase 4: Verification

After all files are written, launch the `citation-audit` and
`consistency-review` agents in parallel. These agents receive NO context
from the research conversation — they read only the produced files. This
isolation prevents confirmation bias. Agent frontmatter defines model and
tools (see `references/research-basis.md` §Model Assignment by Agent Role).

### Pre-Fetch for Citation Audit

Before dispatching the Citation Audit agent, the main thread pre-fetches all
cited URLs so the audit agent needs no network access of its own:

1. Read `citations.md` and extract every cited URL
2. Batch-fetch all URLs via the preflight-established path, writing each
   page into the topic's `fetched/` directory
3. For URLs that fail, run a targeted `multi_search.py` query from the main
   thread to find an alternative source
4. Leave every `FAILED` file in place — the audit agent needs to see which
   sources were unreachable to grade them `INACCESSIBLE`
5. Dispatch the `citation-audit` agent with DELIVERABLE_DIR, FETCHED_DIR,
   and SLUG in the invocation prompt — the agent reads files via Read
   tool, does not fetch URLs itself, and persists its report via
   `put_data.py` to `./.tmp-cited-research/<SLUG>/audit/citation-audit.md`

All web access happens in the main thread before the agents run. This
ensures the audit runs against the same source content snapshot and
avoids the sub-agent permission boundary. Agent tool restrictions (Read,
Glob, and Bash — Bash scoped to the `put_data.py` allowlist rule) are
enforced by agent definition frontmatter.

Dispatch the `consistency-review` agent with DELIVERABLE_DIR and SLUG in
the invocation prompt.

### Promoting Audit Reports to the Deliverable Directory

After both audit agents complete, read each report back from
`./.tmp-cited-research/<topic-slug>/audit/` and write it via the Write
tool to its final home under `<deliverable-dir>/audit/`:

- `<deliverable-dir>/audit/citation-audit.md`
- `<deliverable-dir>/audit/consistency-review.md`

The tmp copy is the agent's write surface; the deliverable copy is the
canonical artifact that ships with the research directory. This two-step
keeps the audit agents on a single allowlisted Bash rule
(`put_data.py`) while preserving the deliverable layout.

### Handling Verification Results

After both sub-agents complete:

1. For INACCURATE or NOT FOUND citations: correct the claim in all files to
   match what the source actually says, or remove the claim and note the gap.
2. For INACCESSIBLE sources: the main thread runs alternative
   `multi_search.py` queries and re-fetches. If unsuccessful, downgrade the claim to "unverified"
   with a note.
3. For FAIL consistency checks: reconcile across all files — fix every file
   that references the incorrect value.
4. If corrections are significant (>3 items), re-run the affected sub-agent.
5. **Update audit reports after fixes.** Add `**Status: RESOLVED**` to each
   fixed issue. Audit reports describing fixed issues as still open confuse
   future readers.

Present the verification summary to the user along with the deliverable.

### Resolving Inaccessible Sources — Ask the User

High-priority inaccessible sources (Tier 1-2) should already have been
triaged with the user between iterations 2 and 3 (see Phase 1). For any
remaining inaccessible sources discovered during verification, ask the user
for help before marking claims as permanently unverified. Prioritize sources
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

### Post-Run Report

After the audits are promoted, print the run report:

```
~/.claude/skills/cited-research/.venv/bin/python \
    ~/.claude/skills/cited-research/scripts/run_report.py <topic-slug>
```

It counts agent losses, engines used, fetch outcomes, citation tiers,
verified-vs-not, token cost and wall clock from the artifacts on disk, and
**writes `report.md` into the deliverable directory** beside `citations.md` so
the run's own measurements ship with the research.

Token accounting reads the CLI's session transcript on disk — it never queries
a telemetry endpoint, which is what lets it work in a sandbox that blocks one.
Currency needs a rate file you supply (`--rates`, or `CITED_RESEARCH_RATES`);
without one the report prints tokens and says money is unpriced rather than
guessing a price.

**Paste the report into your reply, inside a fenced code block, in full.**
(Writing `report.md` does not discharge this — a file in the repo is not
something the reader has seen.)
Running the command is not showing it. Command output goes to the model, not
reliably to the user — a user reading the conversation sees nothing unless
the text is in the response body. Copy every line, including the sections
that report missing inputs; a section reading "no agents.tsv" is a finding,
not filler to trim.

Do not summarise it, restate its numbers in prose, or recompute any of them
yourself. The script exists so the same run always reports the same figures,
and so the reported figures are not the model's recollection of the run.

Anything it flags with `!` is a finding to address or disclose:

- agent signal loss above zero
- queries that returned a single wave
- citations defined but never cited
- citations never audited

## Phase 5: Index Maintenance (cited-research repo only)

Skip this phase when output is not in the `cited-research` repo.
Otherwise, after Phase 4 verification and corrections, update
`index.md` at the repo root. Read
`references/index-maintenance.md` for the entry format and update
rules.
