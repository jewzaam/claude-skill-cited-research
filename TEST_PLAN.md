# Test Plan

This repo is a Claude Code skill, not a runtime application. The Python code that
needs automated testing is the set of helper scripts the skill coordinator
invokes: `scripts/multi_search.py`, `scripts/fetch_url.py`, and the data
wrappers.

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

`tests/test_fetch_url.py` covers the fetch path, where the failure modes are
subtle enough to be worth pinning:

1. **HTML-to-text extraction** — `script`/`style` content dropped, absolute
   link targets preserved inline (research agents are asked to report
   follow-up URLs found in content), relative links dropped, blank-line
   collapse, and the raw-markup fallback on a parser error. The fallback
   matters because an empty file reads to an audit agent as "the source
   contained nothing."
2. **Status classification** — the 403 disambiguation is the critical case:
   `refused:` bodies are per-URL service refusals recorded as `FAILED`, while
   any other 403 is the OpenShell proxy blocking the service address and
   exits 3. Also covers upstream 404, `502 upstream error:`, bad-request 400,
   and service-offline `URLError`.
3. **Header rendering** — `# Fetched:` / `# Date:` / `# Status:` format,
   HTML extraction on OK, plain-text passthrough, and binary content-types
   marked `FAILED` rather than written as mojibake.
4. **URL encoding** — an unencoded `?` or `&` would truncate the service's
   own query string, so the encoding is asserted directly.
5. **`main()`** — file written under the slug, bootstrap requirement enforced,
   and path-escape rejection.

`tests/test_data_wrappers.py` covers `put_data.py` and `reap_data.py`,
including the shared `atomic_write` and `require_slug_root` helpers in
`_data_paths.py` that `fetch_url.py` also uses.

These tests are deterministic, run offline, and gate `make check`.

## Live network tests

Tests marked `@pytest.mark.live` hit real services: one calls DuckDuckGo to
confirm the DDGS response shape we depend on (keys: `href`, `title`, `body`)
has not drifted, and one calls the fetch service `/healthz` to confirm it is
running and reachable. They are **deselected by default** via
`addopts = "-m 'not live'"` in `pyproject.toml`. Run on demand with
`make test-live`. Treat them as manual sanity checks, not CI gates — both
depend on services outside this repo, and a transient failure is not a
regression here. The fetch-service test in particular fails whenever the
operator simply has not started the service, which is its normal state.

## Why no broader test surface

The rest of the repo is markdown (skill instructions, references, contributor
guidance). Markdown content is reviewed by humans and validated by Claude when
the skill runs. Adding markdown linting or link checking is reasonable future
work but is not required to keep the scripts honest.

## What is **not** tested

- The skill behavior itself (Claude executing `SKILL.md`). The skill is
  validated qualitatively against fresh-research and update-workflow scenarios
  documented in `references/research-basis.md` §Validation Plan.
- The `duckduckgo-search` library internals. Third-party behavior is mocked
  at the seam, not retested.
