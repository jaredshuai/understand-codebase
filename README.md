# understand-codebase

An agent skill for [skills.sh](https://skills.sh) that provides a codebase-understanding workflow via `codegraph` + `codebase-memory` + `fast-context` MCPs, with cross-verification rules for ~15 known tool pitfalls.

## Install

```bash
npx skills add jaredshuai/understand-codebase
```

## Prerequisites for `understand-codebase`

This skill requires three MCP servers installed and the target repo indexed:

- **fast-context** — semantic search (`fast_context_search`)
- **codegraph** — call graph + symbol index (`codegraph_search`, `codegraph_callers`, `codegraph_node`, `codegraph_explore`)
- **codebase-memory** — graph database + static analysis (`get_architecture`, `get_code_snippet`, `search_graph`, `query_graph`, `trace_path`, `search_code`, `detect_changes`)

For a new repo, run `index_repository(repo_path, mode:"moderate")` once before using any codebase-memory tool.

## Background

The `understand-codebase` skill was distilled from ~15 rounds of evaluation across four codebases (Rust / Python / TypeScript / mixed), totaling roughly 25,000 graph nodes and 90,000 edges. The pitfalls it documents are tool-level, not project-specific — they reproduce on any repo indexed with the three MCPs above.

## License

MIT
