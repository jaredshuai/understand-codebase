# zcode-skills

Agent skills for the [skills.sh](https://skills.sh) ecosystem, focused on real-world codebase understanding workflows.

## Skills

| Skill | Description |
|-------|-------------|
| [`understand-codebase`](./skills/understand-codebase/SKILL.md) | Codebase understanding, orientation, caller analysis, and refactor impact via `codegraph` + `codebase-memory` + `fast-context` MCPs. Includes cross-verification rules for ~15 known tool pitfalls (semantic_query full-library bug, is_test always false on Function, LSP public-service caller gap, Cypher subset limits, etc.). |

## Install

```bash
npx skills add jaredshuai/zcode-skills
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
