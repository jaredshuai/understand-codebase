---
name: understand-codebase
description: Use when doing codebase understanding, orientation, discovery, caller analysis, or refactor impact analysis via codegraph / codebase-memory / fast-context MCPs, with cross-verification rules for known tool pitfalls. Requires these three MCPs installed.
---

# Understand Codebase

## Overview

This skill is the authoritative workflow for **codebase understanding** tasks via three MCPs:

- **fast-context** — semantic search (concept → file, Chinese/English, cross-layer)
- **codegraph** — call graph + symbol index (LSP-backed, precise)
- **codebase-memory** — graph database + 25+ static metrics (Cypher, blast radius)

The rules below are self-contained and capture ~15 rounds of cross-project evaluation (Rust / Python / TypeScript / mixed) across four codebases. If your repo has a local `docs/mcp-cookbook.md` with project-specific overrides, that cookbook wins for that repo. Otherwise treat this skill as the source of truth.

The main failure mode in practice is **single-source trust**:

- a public service function shows 0 callers (LSP gap) and the agent concludes it is dead code
- `is_test` reports `false` for all `test_*` functions and the agent filters them out
- `semantic_query` returns the entire library and the agent treats it as "broad match"
- `codegraph_explore` claims "20 callers incl. tests" and the agent believes the tests exist without counting

Design around those failures first. A codebase-understanding answer is not reliable until at least two independent sources agree, or a single source has been verified against `rg`.

## Prerequisites

This skill assumes the following MCP servers are installed and the repo is indexed:

- `fast-context` (semantic search)
- `codegraph` (call graph + symbol index)
- `codebase-memory` (graph database + static analysis)

For a new repo, run `index_repository(repo_path, mode:"moderate")` once before using any codebase-memory tool.

## Core Defaults

Use these defaults unless there is a specific reason not to:

- For **discovery / orientation** (no known file path), start with `fast_context_search` before `Grep` or blind file walks.
- For **caller analysis**, use `codegraph_callers` for precise file:line, and `trace_path(direction:"inbound", include_tests:true)` for impact analysis. They disagree more often than they agree on test callers.
- For **refactor impact**, use `codegraph_explore` as the primary source — it is source-heuristic and does not depend on the CALLS edge index.
- For **per-symbol metrics**, use `get_code_snippet` with `include_neighbors=true` to get `caller_names` / `callee_names` arrays.
- For **complex-function hunting**, use `query_graph` on `Function` (not `Method`) — React components and many Python business methods are indexed as `Function`.
- For **mixed-language projects** (e.g. Python backend + TSX frontend), run Cypher on **both** `Function` and `Method` and union the results.
- For **full-text search**, use `search_code` with `regex=true` whenever the pattern is not a pure literal; use `context:N` to get source lines around each match.
- For **exact counts**, never trust `search_code` `total_results` (dedup up to 8.3x); use `rg -c`.
- When **two tools disagree**, prefer the more reliable one for that specific question (see Reliability Tiers below), then investigate the discrepancy rather than picking the smaller answer.

## Reliability Tiers

Tier 1 — trust directly:

- `fast_context_search` semantic file localization
- `get_architecture` overview / hotspots / clusters / routes
- `get_code_snippet` static metrics (complexity, param_count, lines, callers/callees counts, fp/sp/bt)
- `codegraph_explore` blast radius
- `codegraph_node` source code and Trail

Tier 2 — trust after one cross-check:

- `codegraph_callers` (cross-check with `trace_path inbound` for test callers; with `codegraph_explore` for public-service LSP gap)
- `trace_path data_flow` (may include same-name-different-semantics paths — inspect qualified_name)
- `search_code` total_results (cross-check with `rg -c` for counts)

Tier 3 — do not use as a primary source:

- `search_graph` `semantic_query` (results field returns the entire library; use `query` BM25 instead)
- `is_test` field on `Function` nodes (always false; filter by `file_path` or `name` prefix instead)
- `codegraph_explore` test-coverage annotations (name-match heuristic, false positives)
- `codegraph_search` with `kind: route | component | type` (use `search_graph(label:"Route"|"Type")` or `get_architecture` routes instead)
- `ingest_traces` (accepted but not implemented)

## Default Workflow

```
1. get_architecture(aspects=['all'])             # or ["hotspots"] etc. on large repos
2. fast_context_search("natural-language description")  # concept → file
3. codegraph_callers("core_function") OR trace_path("core_function", risk_labels:true)
4. query_graph("MATCH (n:Function) WHERE n.complexity > 10 ...")   # refactor candidates
5. get_code_snippet(qualified_name, include_neighbors:true)
6. search_code("keyword", regex:true) OR rg fallback
```

## Hard Rules (known pitfalls — do not violate)

- **NEVER** use `search_graph` `semantic_query`. Its `results` field returns the entire library regardless of keywords. Use `query` (BM25).
- **NEVER** trust the `is_test` field on `Function` nodes — it is `false` even for obvious `test_*` functions. Filter test code by `file_path CONTAINS 'tests/'` or `name STARTS WITH 'test_'`.
- **NEVER** trust `codegraph_explore` test-coverage annotations. Count tests yourself with `rg -c "def test_"` or `rg -c "#\[test\]"`.
- **NEVER** trust `codegraph_callers` returning 0 for a **public service function** (no `_` prefix). Always cross-verify with `codegraph_explore`. Large projects have an LSP resolver gap that drops CALLS edges for module-level imports.
- **ALWAYS** pass `regex=true` to `search_code` for any non-literal pattern (`.*`, `\w+`, `|`, `^...`, `[a-z]`). Default is literal search.
- **ALWAYS** pass `include_neighbors=true` to `get_code_snippet` when you need caller/callee **names**, not just counts.
- `codegraph_node` returns 0 symbols for `__init__.py` / `mod.rs` / `index.ts` — use `rg` for those.
- `codegraph_callers` may underreport test callers. For impact analysis prefer `trace_path(direction:"inbound", include_tests:true)`.
- `trace_path mode:"data_flow"` may mix in same-name-different-semantics paths (e.g. `commit` → `PluginImportService.commit` instead of `Session.commit`). Inspect `qualified_name` before drawing conclusions.
- `search_code` dedups up to 8.3x; `total_grep_matches` is capped at 500. Use `rg -c` for exact counts.

## Cypher (`query_graph`) Rules

Supported:

- `DISTINCT`, `CONTAINS`, `STARTS WITH`, `ENDS WITH`, `toUpper()`, `IN [...]`
- `count()`, `sum()`, `max()`, `min()`, `collect()` (output may be truncated)
- Implicit `GROUP BY`, `SKIP ... LIMIT`, `ORDER BY`, `AND`, range `>= <=`
- Variable-length `*1..2`, chained `(a)->(b)->(c)`
- `type(r)` when not combined with aggregation
- Boolean comparison `WHERE n.x = true` / `= false`

Not supported / project-dependent:

- `MATCH path=...` (variable paths) — always fails with `expected token type 66`
- `WITH` clauses — silently drop results
- Boolean shorthand `WHERE n.x` — must be `WHERE n.x = true`
- Reverse traversal `<-[:TYPE]-` — project-dependent; rewrite as forward if it errors
- `!=` / `<>` field-vs-field — project-dependent; field-vs-literal works
- `type(r)` combined with aggregation — returns numbers instead of strings

Most valuable Cypher query (find refactor candidates):

```cypher
MATCH (n:Function) WHERE n.complexity > 10
RETURN n.name, n.complexity, n.file_path
ORDER BY n.complexity DESC LIMIT 10
```

## Common Decision Trees

### "Is function X dead code?"

1. `codegraph_callers(X)` — note the count and the file paths
2. `trace_path(X, direction:"inbound", include_tests:true)` — note test vs non-test callers
3. If step 1 returned 0 and X is a **public service function** (no `_` prefix), run `codegraph_explore(X)` — this is the LSP-gap case
4. Only declare X dead if **all three** agree there are no non-test callers
5. Confirm with `rg -n "X\("` across the repo

### "Who calls X, including tests?"

1. Primary: `trace_path(X, direction:"inbound", include_tests:true)`
2. Precision layer: `codegraph_callers(X)` for file:line references
3. If the two disagree on test coverage, trust `trace_path` for coverage and `codegraph_callers` for line numbers

### "What's safe to rename / refactor?"

1. Primary: `codegraph_explore("X")` for blast radius (source-heuristic, most reliable)
2. Cross-check: `trace_path(X, direction:"inbound", risk_labels:true)` for CRITICAL-hop callers
3. Complex callees: `query_graph("MATCH (n:Function) WHERE n.complexity > 10 ...")`
4. For each caller that is itself complex, repeat steps 1–3 before committing

### "Filter out test code in a Cypher query"

Use:

```cypher
MATCH (n:Function)
WHERE n.name STARTS WITH 'test_' OR n.file_path CONTAINS 'tests/'
RETURN ...
```

Do **not** use `WHERE n.is_test = true` — it returns 0 even when there are hundreds of `test_*` functions.

### "Which React components are most complex?"

```cypher
MATCH (n:Function)
WHERE n.file_path ENDS WITH '.tsx' AND n.complexity > 15
RETURN n.name, n.complexity, n.file_path
ORDER BY n.complexity DESC LIMIT 10
```

React components are indexed as `Function`, not `Method`.

## Project Setup (first run on a new repo)

```
index_repository(repo_path: "<absolute path>", mode: "moderate")
index_status(project: "<auto-generated project name>")
get_graph_schema(project: "<project name>")
get_architecture(project: "<project name>", aspects: ["all"])
```

Confirm:

- node / edge counts are reasonable for the repo size
- the expected languages appear in the distribution
- routes appear in `get_architecture` (do not rely on `codegraph_search kind:route`)

## Common Mistakes

- trusting `codegraph_callers = 0` for a public service function and deleting it
- filtering test code with `WHERE n.is_test = true` and concluding there are no tests
- using `search_code` with a regex pattern but no `regex:true`, getting 0 results, and concluding the pattern does not exist
- using `semantic_query` and treating 15,000 results as "broad match"
- using `search_code` `total_results` for counting (dedup) instead of `rg -c`
- running Cypher on `Method` only in a React/TSX project and missing all components
- running Cypher on `Function` only in a mixed project and missing backend methods
- trusting `codegraph_explore` "tests: <file>" annotation without `rg -c` verifying the tests exist
- trusting `trace_path data_flow` paths without inspecting `qualified_name` (same-name-different-semantics)
- using `codegraph_node` on `__init__.py` / `mod.rs` and concluding the file is empty
- attempting `MATCH path=...` in Cypher and retrying variations instead of rewriting to forward chains
- using `WHERE n.x` boolean shorthand in Cypher and treating the error as "no results"
- using `ingest_traces` and treating the `accepted` response as evidence that traces were recorded

## Quick Reference

| Situation | Default |
|---|---|
| discovery / orientation (no known file) | `fast_context_search` |
| project overview | `get_architecture(aspects=['all'])` (or subset on large repos) |
| who calls X (precise file:line) | `codegraph_callers(X)` |
| who calls X (impact, incl. tests) | `trace_path(X, direction:'inbound', include_tests:true, risk_labels:true)` |
| refactor blast radius | `codegraph_explore(X)` |
| per-symbol metrics | `get_code_snippet(qualified_name, include_neighbors:true)` |
| complex functions | `query_graph` on `Function` (both `Function` + `Method` in mixed projects) |
| similar/parallel code | Cypher `MATCH (a)-[:SIMILAR_TO]->(b)` (jaccard ≥ 0.95) |
| route inventory | `get_architecture` routes array, or `search_graph(label:'Route')` with `WHERE source='decorator'` |
| full-text search | `search_code(pattern, regex:true, context:N)` |
| exact match counts | `rg -c` (search_code dedups) |
| test code filtering | `file_path CONTAINS 'tests/'` or `name STARTS WITH 'test_'` |
| `__init__.py` / `mod.rs` contents | `rg` (codegraph returns 0 symbols) |
| Rust `#[test]` / `derive(` | `rg` (search_code fails on Rust special syntax) |
| large file symbol list | `codegraph_node(file, symbolsOnly:true)` |

## Core Principle

A codebase-understanding answer is reliable only when **at least two independent sources agree**, or one source is Tier 1 and has been verified against `rg`. Single-source answers — especially negative answers ("no callers", "no tests", "pattern not found") — must be cross-verified before any irreversible action (rename, delete, migrate).
