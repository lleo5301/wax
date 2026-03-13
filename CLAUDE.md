# Wax (macOS) - Claude Code Memory

## Install

### MCP Server (Claude Code)

```bash
npx -y waxmcp@latest mcp install --scope user
```

## Rules

1. **Session start** — call `wax_session_start`, then `wax_handoff_latest` to resume prior context (use `session_id` for scoped calls)
2. **Before answering** — call `wax_recall` (or `wax_search` if you need raw ranked hits)
3. **When you learn something durable** — call `wax_remember` (batch writes, then `wax_flush` to persist)
4. **When corrected** — store the correction via `wax_remember` (and retract structured facts via `wax_fact_retract` when applicable), then `wax_flush`
5. **Session end** — call `wax_handoff` (`content`, optional `project`, `pending_tasks`), then `wax_flush`, then `wax_session_end`

## Tools

| Tool | When |
|------|------|
| `wax_session_start` | Start a new memory session; use returned `session_id` to scope reads/writes |
| `wax_session_end` | End the active session |
| `wax_remember` | Store durable info (preferences, decisions, facts, key notes) |
| `wax_recall` | Build RAG context for a query (recommended default read path) |
| `wax_search` | Direct search when you need raw ranked hits (hybrid/text modes) |
| `wax_flush` | Persist pending writes so they’re searchable and safe on disk |
| `wax_stats` | Quick health check (store stats, embedder identity, vector search enabled) |
| `wax_handoff` | End-of-session summary (`content`) + optional `pending_tasks` for continuity |
| `wax_handoff_latest` | Start-of-session: load the most recent handoff (optionally by `project`) |
| `wax_entity_upsert` | Create/update a structured entity (knowledge graph) |
| `wax_entity_resolve` | Resolve entities by alias |
| `wax_fact_assert` | Assert a structured fact |
| `wax_fact_retract` | Retract (soft-delete) a structured fact by id |
| `wax_facts_query` | Query structured facts |
