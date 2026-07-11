---
description: "Use when exploring, searching, tracing, or analyzing project source code, architecture, dependencies, call chains, dead code, refactor candidates, or impact of changes. Hard rule: prefer codebase-memory graph tools before falling back to grep or manual file reading."
name: "Use Codebase Memory for Code Exploration"
---

# Codebase Memory First Rule

Whenever the task involves understanding, searching, tracing, or analyzing project code, you MUST load and use the `@codebase-memory` skill first.

## Trigger Scenarios

Load `@codebase-memory` when the user asks anything related to:

- Exploring or understanding the codebase architecture / structure
- Finding functions, classes, methods, variables, or symbols
- Tracing call chains: "who calls X", "what does X call"
- Finding callers, callees, definitions, implementations, or usages
- Impact analysis of changes or refactors
- Dead code / unused functions / high fan-out detection
- Code quality audit, dependency analysis, or cross-service communication
- Any task where reading multiple files or searching across the project would otherwise be needed

## Required Workflow

1. **Load the skill**: Invoke `@codebase-memory` before performing any code search or exploration.
2. **Check indexing**: Use `list_projects` and `index_status` to verify the current project is indexed. If not, run `index_repository` first.
3. **Prefer graph tools**: Use `search_graph`, `trace_path`, `search_code`, `get_architecture`, `get_code_snippet`, and `detect_changes` before falling back to `grep_search` or `file_search`.
4. **Fall back only when necessary**: Use grep/manual reads only when:
   - The project is not yet indexed and indexing fails, OR
   - The query is purely textual (e.g., finding a literal string in comments/logs), OR
   - The user explicitly asks for raw text search.

## Rationale

Graph tools return precise structural results in ~500 tokens versus ~80K for grep, and they preserve relationships (calls, imports, HTTP/async edges) that grep cannot express.
