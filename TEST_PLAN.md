# Test Plan

This repo is a Claude Code skill, not a runtime application. The only Python code
that needs automated testing is `scripts/multi_search.py`, the multi-engine search
helper invoked by the skill coordinator.

## What is tested

`tests/test_multi_search.py` exercises three concerns:

1. **`deduplicate()`** — pure logic. Empty input, duplicate URL collapse,
   first-occurrence preservation, drop-empty-URL behavior.
2. **`search_ddg()`** — DDGS interaction with the `duckduckgo_search` library
   mocked. Field normalization (`href` → `url`, `body` → `snippet`), missing
   field handling, and the import-error path that fires when the dependency
   is not installed.
3. **`main()`** — CLI contract. Argparse failures (unknown engine, missing
   `--query`), JSON output shape, and per-engine failure isolation.

These tests are deterministic, run offline, and gate `make check`.

## Live network test

A single test marked `@pytest.mark.live` calls real DuckDuckGo to confirm the
DDGS response shape we depend on (keys: `href`, `title`, `body`) has not
drifted. It is **deselected by default** via `addopts = "-m 'not live'"` in
`pyproject.toml`. Run on demand with `make test-live`. Treat it as a manual
sanity check, not a CI gate — DDG is a third-party service and a transient
failure is not a regression in this repo.

## Why no broader test surface

The rest of the repo is markdown (skill instructions, references, contributor
guidance). Markdown content is reviewed by humans and validated by Claude when
the skill runs. Adding markdown linting or link checking is reasonable future
work but is not required to keep the script honest.

## What is **not** tested

- The skill behavior itself (Claude executing `SKILL.md`). The skill is
  validated qualitatively against fresh-research and update-workflow scenarios
  documented in `references/research-basis.md` §Validation Plan.
- The `duckduckgo-search` library internals. Third-party behavior is mocked
  at the seam, not retested.
