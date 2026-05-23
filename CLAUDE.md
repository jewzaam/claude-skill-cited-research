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
broadening the URL pool beyond Claude's built-in WebSearch. To add a new
engine:

1. Write a `search_<engine>(query, limit) -> list[dict]` function that
   returns `{url, title, snippet, engine}` dicts.
2. Register the engine in both `SUPPORTED_ENGINES` and `ENGINE_FUNCTIONS`.
3. Declare any new library dependency in `pyproject.toml`.
4. Add unit tests in `tests/test_multi_search.py` that mock the client at
   `sys.modules` and cover field normalization + missing-field handling.
5. Update the `Engines:` section of the module docstring.

The module name must remain underscore-only (`multi_search`, not
`multi-search`) so `python -m scripts.multi_search` resolves.

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
