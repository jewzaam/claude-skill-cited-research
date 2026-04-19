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

## Agent Prompts

Agent prompt templates live in `references/sub-agent-prompts.md`. Keep
prompts lean and task-oriented — longer explicit reasoning instructions
degrade performance in research agents (Xu et al., "Search-R1",
arXiv:2602.19526, 2026). The current prompts are structured for "fast
thinking" (direct search/answer) over "slow thinking" (explicit reasoning
before each action).

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
(fetched page content, inter-agent files) are persisted under
`~/.local/share/cited-research-data/<topic-slug>/` via two wrappers:

- **`scripts/put_data.py <slug> <relative-path>`** — reads stdin and
  writes it verbatim to the target. Validates that the relative path
  stays inside the slug directory.
- **`scripts/reap_data.py <slug>`** — removes the slug's subtree
  recursively. Idempotent.

When adding new data-writing behavior to the skill, route it through
`put_data.py` rather than the Write tool. The wrappers are allowlisted
with a single Bash rule each; the Write tool prompts per path and the
pain compounds quickly during a research run. The skill directory itself
is never written to at runtime — only the data directory and the user's
project directory are mutable surfaces.

Shared helpers (path validation, DATA_ROOT) live in
`scripts/_data_paths.py`. Tests in `tests/test_data_wrappers.py`
monkeypatch `DATA_ROOT` onto `tmp_path`, so tests never touch
`~/.claude/`.

## Testing and Make Targets

Use the Makefile for all validation — `make check` runs format, lint,
typecheck, test, and coverage. The venv is created by the `$(PYTHON)`
target; do not run `python -m venv` or `pip install` manually.

Live-network tests are marked `@pytest.mark.live` and deselected by
default. Run them on demand with `make test-live` — they are a manual
sanity check, not a CI gate.
