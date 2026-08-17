# Contributor Guidance for cited-research Skill

## Agent Prompt Framing

Do not use expert persona framing in agent prompts (e.g., "You are an expert
in...", "As a senior researcher..."). Use task-oriented framing instead (e.g.,
"You are researching [DIMENSION] for [PROJECT]").

Expert persona prompts reduce factual accuracy by 3.6 percentage points on
MMLU because persona activation competes with factual recall for processing
resources (Hu et al., "PRISM: Expert Personas Improve Alignment but Damage
Accuracy", USC, arXiv:2603.18507, 2026 —
<https://arxiv.org/html/2603.18507v1>). Research agents need maximum factual
recall, not role-playing.

## WebSearch Is Banned; Search Is Not a Source

Search runs only through `scripts/multi_search.py`. WebSearch must not be
used to find or support anything, because it returns a model-synthesized
answer above its links. That prose is
an interpretation of results, not source text. If it reaches an agent's
findings it becomes an upstream bias vector with a feedback loop: it shapes
which URLs get fetched, which claims get written, and then hides behind a
citation to a page nobody read.

Three places enforce the separation. Change them together or not at all:

- `SKILL.md` principle 1 — claims come from fetched page text; search
  contributes URLs and verbatim snippet quotes only.
- `agents/research-discovery.md` and `agents/research-counter-discovery.md`
  — "Snippet Quotes" (verbatim, quoted) rather than "Preliminary Findings",
  and an explicit ban on restating the synthesized answer.
- `agents/citation-audit.md` — a claim with no fetched file grades
  INACCESSIBLE, never VERIFIED.

Do not reintroduce a discovery-phase section that invites agents to state
conclusions. The phase exists to find pages, not to answer the question.

## Research Basis

Design decisions in this skill are documented with supporting evidence in
`references/research-basis.md`. When modifying a behavior that has a
corresponding section in that file, check whether the change is consistent
with the evidence. If new research supersedes an existing finding, update
both the skill instruction and the research basis entry.

## Agent Definitions

Agent definitions live in `agents/` as `.md` files with YAML frontmatter
(model, tools, background mode). Keep agent body prompts lean and
task-oriented — longer explicit reasoning instructions degrade performance
in research agents (Xu et al., "Search-R1", arXiv:2602.19526, 2026). The
current prompts are structured for "fast thinking" (direct search/answer)
over "slow thinking" (explicit reasoning before each action).

## Update Workflow

The update-research path (refresh stale topics, add dimensions, re-verify
citations) lives in `references/update-workflow.md`. `SKILL.md` only
delegates to it. Changes to update behavior go in that file, not inline
in `SKILL.md`.

## Multi-Engine Search Script

`scripts/multi_search.py` is the coordinator's augmentation tool for
the skill's only sanctioned search path. WebSearch is banned.

Two things about `ddgs` that are easy to get wrong:

- **It is a metasearch layer, not a DuckDuckGo client.** It fronts
  DuckDuckGo, Brave, Bing, Mojeek, Startpage, Yahoo, Google, and Yandex.
  Anything that allowlists hosts (a sandbox network policy) should name
  all of them. Not fatal if it does not: given a backend list, `ddgs`
  falls through to whichever respond, and a full result set comes back
  even from one working engine. Blocked backends cost diversity, not
  volume.
- **The point of this script is a corpus that did not come from
  a model's synthesis of results.** Cross-engine diversity within `ddgs`
  is a bonus, not the
  purpose. Do not add machinery to guarantee *all-engine* coverage. But
  **two** independent samples is the floor, not a bonus — that is what the
  wave split enforces, and it is why `--health` exits non-zero when fewer
  than two backends are alive.
- **Do not pass `backend="auto"`.** It picks engines at random per call,
  which makes coverage non-reproducible and can collapse to a single
  engine — the outcome this script exists to prevent.
- **Read the backend list from the library at runtime, never hardcode
  it.** `text_backends()` reads `ddgs.engines.ENGINES["text"]`. A
  hardcoded list silently narrows coverage when `ddgs` adds engines, and
  — worse — can name an engine that is not registered for the text
  category at all.
- **Unknown backend names are not a no-op.** `ddgs` silently falls back
  to auto-selection when a name is not registered for the requested
  category — verified directly: the bogus name `totallyfakeengine`
  returned the same result set as `bing` (registered for images/news,
  not text). `validate_backends()` rejects unregistered names before
  they reach `ddgs` for exactly this reason: an unnoticed typo or
  wrong-category name would otherwise return the full auto pool while
  looking like one isolated engine, destroying the two-wave split's
  guarantee without any error surfacing. Do not remove that validation,
  and do not "helpfully" pass unknown names through.
- **`--health` probes a deliberately unregistered sentinel name first**
  to establish what "no isolation" looks like (see the unknown-backend
  fallback above). Any backend whose results are identical to the
  sentinel's is reported `FALLBACK` rather than `OK` — without this
  check the probe gives false positives; it previously reported an
  unregistered `bing` as healthy.

**Do not add a flag that disables the two-wave split, and do not collapse
the waves into one call.** A single call fills `--limit` from whichever
backend answers first, so the split is the only thing that forces two
independent engine populations to contribute. Coverage is reported in the
JSON payload rather than only on stderr for the same reason: a caller must
not be able to overlook a run that collapsed to one sample. `unique_by_wave`
exists because a wave can answer successfully and still add zero new URLs —
measured at 0 of 6 on one live query and 2 of 6 on another.

To add a new engine:

1. Write a `search_<engine>(query, limit) -> dict` function returning
   `{"results": [{url, title, snippet, wave, backends}, ...],
   "coverage": {...}}`. If the engine has no wave concept of its own, still
   populate `coverage` so the caller's checks work uniformly.
2. Register the engine in both `SUPPORTED_ENGINES` and `ENGINE_FUNCTIONS`.
3. Declare any new library dependency in `pyproject.toml`.
4. Add unit tests in `tests/test_multi_search.py` that mock the client at
   `sys.modules` and cover field normalization + missing-field handling.
5. Update the `Engines:` section of the module docstring.

The module name must remain underscore-only (`multi_search`, not
`multi-search`) so `python -m scripts.multi_search` resolves.

## Fetch Service Script

`scripts/fetch_url.py` is the only page-reading path. `WebFetch` fails
inside a sandbox (`Socket is closed`) because it egresses from the
container and hits OpenShell's CONNECT proxy; the fetch service runs on
the host and performs the request over plain HTTP on the sandbox's
behalf. Search is a separate path entirely — `multi_search.py` egresses
directly, because the engines reject the fetch service's plain GET with an
anti-bot challenge.

When changing this script:

- Keep it stdlib-only, with exactly one exception. It runs before
  `make install-dev` might have been re-run, and adding a dependency to
  the fetch path makes the skill fail in exactly the situation where it is
  least debuggable. The exception is `pypdf`, and it is imported *inside*
  `pdf_to_text` rather than at module scope, so a missing install degrades
  to `FAILED (pypdf not installed …)` on PDF fetches only and leaves every
  HTML fetch working. The OCR extra follows the same rule and is optional
  on top of that. Any further dependency must meet the same bar: worth the
  cost, and import-guarded so its absence cannot take the script down.
- `scripts/explore/` is investigation. The test is whether acting on the
  output means editing code or running a query: probing search backends that
  are *not* integrated feeds a parser, so it is exploration; asking whether
  the configured backends are alive is `multi_search.py --health`, so it is
  runtime. A script that graduates to runtime leaves `explore/`, loses the
  coverage exemption and gains tests. Being copied into
  `~/.claude/skills/` is not that graduation and is not worth preventing —
  the deploy copies the repo, the files cost nothing there, and blocking it
  buys no safety.
- Shipped docs (`SKILL.md`, `references/`, `agents/`) must never give a
  command that assumes a cwd or a repo. The cwd during a run is the user's
  research repo, which has its own `scripts/` — a bare `scripts/fetch_url.py`
  silently resolves to a different project rather than erroring, and `make
  <target>` needs a cwd nobody is in. Invoke through
  `$SKILL/.venv/bin/python $SKILL/scripts/<name>.py`, and spell out `cd` for
  operator setup steps. Makefile targets are for this repo, not for a run.
- Every optional capability must appear in `CAPABILITIES` so `--check`
  reports it. The rule is that a dependency problem is discovered at
  preflight, never eighty URLs into a run — the cost of the late discovery
  is the whole run, and it presents as "the web is broken" rather than as
  "one pip install is missing".
- OCR text must keep its `OK (OCR …)` status marker. The citation-audit
  agent compares the deliverable against the fetched file; if OCR corrupts
  a digit, the deliverable inherits it and the audit *confirms* it, because
  both read the same corrupted text. The marker is the only thing telling a
  downstream reader that this body is a transcription rather than a source.
  `rapidocr-onnxruntime` resolves `opencv-python` by default, which needs
  `libGL.so.1` and fails on import in a headless container — the extra pins
  `opencv-python-headless` for that reason. Do not drop that pin.
- Keep the body as `bytes` from `fetch` through to `render`. Decoding in
  `fetch` is the obvious simplification and it destroys every PDF, which
  is the Tier 1 half of a typical source pool.
- Preserve the 403 disambiguation. A `403` whose body starts `refused:`
  is the service declining one URL (recoverable, recorded as `FAILED`);
  any other `403` is the proxy refusing the service address itself
  (exit 3, operator action). Collapsing these two makes a policy
  misconfiguration look like 100% of sources being unreachable.
- Preserve exit code 3 as distinct from 0. Exit 0 means a file was
  written, including a `FAILED` one; the run continues. Exit 3 means no
  file and no possible progress.
- HTML-to-text extraction is deliberately stdlib `HTMLParser`. It falls
  back to raw markup on a parse error rather than writing an empty file,
  because an empty file reads to an audit agent as "the source contained
  nothing" — a silent false negative.
- Encyclopedia reference mining is deterministic parsing, not a model
  instruction. A Wikipedia/Grokipedia URL gets its outbound citation
  links written to a `<name>.refs` sidecar, one URL per line. MediaWiki
  marks citations with `class="external"`; where that marker is absent
  (Grokipedia is not MediaWiki) it falls back to absolute cross-host
  links, and infrastructure hosts (wikimedia.org, wikidata.org,
  creativecommons.org, sister projects) are filtered because they
  appear on every page and are never the cited source. Do not move this
  back into prose instructions for the model — instructions cost tokens
  on every run and depend on the model complying; parsing is
  deterministic and free.
- A `Status: OK` file with no content is the failure mode to keep guarding
  against. Two paths produce it and both are now recorded as `FAILED`: a
  JS-rendered page whose HTML is a shell, and a scanned PDF with no text
  layer. Both grade to an audit agent as "the source said nothing", which
  is a different and wrong claim from "the source could not be read".

Behavior contract and operator-facing docs live in
`references/data-persistence.md`. The service itself is documented in
the openshell-sandbox repo — do not restate its contract here.

## Run Report

`scripts/run_report.py` computes the post-run validation report from
artifacts on disk. It exists so that "how did the run go" is counted, not
recalled — a model summarising its own run reports what it remembers doing,
which is how a run that skipped multi-engine search entirely still got
described as multi-engine.

When changing it:

- **Keep every number derived from a file.** If a figure cannot be counted
  from `agents.tsv`, `search/*.json`, `fetched/*.md`, `citations.md` or
  `audit/citation-audit.md`, it does not belong in the report.
- **Report missing inputs explicitly** ("no agents.tsv — …"). A silently
  omitted section reads as a clean result.
- **Ordinal scales print in scale order, not by frequency.** Tiers 1-4 and
  the grade severity order are fixed; engines and HTTP codes sort by count.
  Absent tiers still print as zero, because "Tier 1: 0" is a finding.
- **Gloss anything a reader would have to look up**, such as HTTP status
  codes.

`SKILL.md` requires the coordinator to paste the output verbatim into its
reply. That instruction exists because running the script does not show it
to anyone — command output reaches the model, not reliably the user.

The script also **writes `report.md` into the deliverable directory**, as a
peer of `citations.md`. Terminal output scrolls away and a pasted block lives
only in one conversation; the run's own measurements should travel with the
research. `--no-write` suppresses it for a dry run.

### Token accounting must stay local

`scripts/run_meta.py` reads token counts from the CLI's session transcripts at
`~/.claude/projects/<encoded-cwd>/*.jsonl` (`message.usage`), plus subagent
totals from the `<subagent_tokens>` marker in Agent tool results. **This is a
file read and must remain one.** A sandbox whose network policy deliberately
blocks the OTEL collector must not gain a back door to the same data through
the fetch service — do not add a telemetry query, a Prometheus call, or any
network path to this module.

Two consequences worth preserving:

- **Scope by timestamp window, not session id.** A run that outlives a
  container restart continues in a new session file; scoping by id truncates
  the report at the restart, which is the same blind spot the status line has.
  `render()` prints a `!` line when a run spans more than one session.
- **Never hardcode prices.** No local artifact contains cost. Rates are read
  from an operator-owned JSON file via `--rates` or `CITED_RESEARCH_RATES`,
  and with no rate file the report prints tokens and says money is unpriced.
  A stale constant would produce a confident wrong number, which is worse
  than no number.

Cache reads dominate the raw token total by an order of magnitude and are
priced differently everywhere, so the headline figure is **billable tokens**
(input + output + cache creation) with the components printed underneath.

## Data Persistence Wrappers

Transient artifacts the coordinator needs to hand off to sub-agents
(fetched page content, inter-agent files, audit reports) are persisted
under `./.tmp-cited-research/<topic-slug>/` — colocated with the user's
project root rather than a user-global cache. Three scripts manage the
tree, all behind single allowlisted Bash rules so the user never sees a
per-file approval prompt during a research run:

- **`scripts/bootstrap_tmp.sh <slug>`** — provisions the parent
  `./.tmp-cited-research/` and its `.gitignore` of `*`, then wipes-and-
  recreates the slug subdir. Per-slug isolation: sibling topic slugs in
  the same parent are untouched. Modeled on `bootstrap-tmp.sh` from
  `~/source/jewzaam-reviews/`.
- **`scripts/put_data.py <slug> <relative-path>`** — reads stdin and
  writes it verbatim to the target under the slug subdir. Validates
  that the relative path stays inside the slug directory.
- **`scripts/reap_data.py <slug>`** — removes the slug's subtree
  recursively, no parent setup. Idempotent. Use for cleanup between
  runs when full re-bootstrap is not desired.

When adding new data-writing behavior to the skill, route it through
`put_data.py` rather than the Write tool — including from sub-agents
(see `agents/citation-audit.md` and `agents/consistency-review.md` for
the heredoc invocation pattern). Reserve the Write tool for the
deliverable, references, and citation files in the user's project
directory, which is a separate permission surface.

Shared helpers (path validation, DATA_ROOT) live in
`scripts/_data_paths.py`. `DATA_ROOT = Path.cwd() / ".tmp-cited-research"`
resolves at import time, so each subprocess invocation picks up the
caller's working directory (the user's project root in normal use).
Tests in `tests/test_data_wrappers.py` monkeypatch `DATA_ROOT` onto
`tmp_path`, and tests in `tests/test_bootstrap_tmp.py` pass
`PROJECT_ROOT=tmp_path` via env — neither test suite ever touches the
real project directory.

## Testing and Make Targets

Use the Makefile for all validation — `make check` runs format, lint,
typecheck, test, and coverage. The venv is created by the `$(PYTHON)`
target; do not run `python -m venv` or `pip install` manually.

Live-network tests are marked `@pytest.mark.live` and deselected by
default. Run them on demand with `make test-live` — they are a manual
sanity check, not a CI gate.

`lint` runs flake8 with `-j 1`. In a rootless container without POSIX
semaphores, flake8's `multiprocessing.Pool` raises `PermissionError` and
fails the target before linting anything. The repo is ~11 files, so
serial linting costs nothing measurable and works in both sandboxes and
on a host — do not remove the flag to "restore parallelism".
