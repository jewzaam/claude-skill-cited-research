# Phase 1 — Multi-engine Search

**Status:** Not started. Tagged for future work.

## Why

Single-engine search is a structural bias. Per `agentic-research-bias` citation [45], 84.9% of results are unique to a single engine. The cited-research skill currently uses `WebSearch` only, so every discovery agent draws from one biased result pool.

## Goal

Give the discovery step access to 2+ search engines, deduplicate the URL pool before fetching, and record the change in the skill's research basis.

## Locked-in decisions

- **Script, not MCP.** A Python script using existing libraries. Adding an MCP server is out of scope.
- **Dedup before fetch.** Collect URLs from all engines, reduce to a distinct set, then hand off to the fetch step.
- **Sub-agents may invoke the script.** The whole point is engine diversity — forcing sub-agents to stay WebSearch-only defeats the purpose. Prompt-injection risk is acknowledged and must be mitigated at the script boundary (trusted path, no shell interpolation of fetched text).

## Work

1. Pick libraries. Candidates: DuckDuckGo (`duckduckgo-search`), Brave (`brave-search` API — requires key), Bing (API — requires key), SearXNG (self-hosted or public instance). Pick 1-2 to start; DuckDuckGo is the most frictionless.
2. Write `scripts/multi-search.py` exposing a unified CLI: `python scripts/multi-search.py --query "..." --engines ddg,websearch --limit 10` → JSON list of `{url, title, snippet, engine}`.
3. Decide fan-out granularity:
   - Per-query (simplest): each query runs on each engine.
   - Per-dimension: the dimension gets a pool from each engine.
   - Global: top-N from each engine, merged.
   Default to per-query unless cost becomes a blocker.
4. Update `references/sub-agent-prompts.md` Research-Discovery template — instruct the agent to invoke the script in addition to WebSearch.
5. Update `SKILL.md` Phase 1 section to document the multi-engine pattern.
6. Document the change in `references/research-basis.md` with citation [45] from `agentic-research-bias`.

## Open questions for this phase

- **Prompt-injection mitigation.** What does "trusted script path" mean operationally? Options: hardcode the script path in sub-agent prompts; have the coordinator invoke the script rather than the sub-agents; sanitize query strings.
- **API keys.** Brave/Bing require keys. DuckDuckGo is free but rate-limited. Pick based on the user's willingness to manage keys.
- **Cost.** Each extra engine = extra calls per dimension. Quantify before deciding fan-out granularity.

## Files to modify

- `scripts/multi-search.py` (new)
- `references/sub-agent-prompts.md` — Research-Discovery template
- `SKILL.md` — Phase 1 section
- `references/research-basis.md` — new section for multi-engine search

## Verification

- Run `scripts/multi-search.py` against a known query and confirm URLs from at least two engines appear.
- Run a discovery agent against a toy topic and confirm the URL pool is broader than a WebSearch-only baseline.
- Confirm dedup removes exact-URL duplicates across engines.
