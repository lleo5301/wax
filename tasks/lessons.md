## Release Script

- Use single backslashes in `perl` regexes inside shell single-quoted strings. Over-escaping `\s` caused the `waxmcp` release script to print the version bump step without actually mutating `package.json` or `Sources/WaxMCPServer/main.swift`.

## Feature Scoping

- When the user says a feature is "for the MCP tool", do not add a parallel CLI surface by default. Put the workflow behind MCP schemas, handlers, and MCP-focused tests first, and only add CLI affordances if the user asks for them explicitly.

## MCP Tool Reviews

- Every new MCP tool needs two explicit checks before considering it done:
  - extend `validateArgumentSurface` so typoed top-level keys fail fast instead of silently falling back to defaults
  - audit mode-specific resource policy so `mode=text` paths do not load embedders or rebuild vector data unnecessarily

## Regression Tests

- When removing expensive setup from a regression test, preserve the original contract being asserted. If the test is about protocol conformance, switch it to a type-level conformance assertion instead of weakening it to "can be constructed".

## CLI / MCP Contracts

- If a broker-backed path cannot honor a CLI flag, do not silently ignore the flag. Either bypass the broker for that code path or fail with a clear error.
- For PATH-launched process tests, keep stdin open until the MCP `tools/list` response arrives. Closing early can make a healthy server look broken because the response never flushes.
- When asserting on CLI JSON output, parse the JSON or match the exact pretty-printed form. Do not assume compact formatting.
- When migrating MCP/CLI behavior behind the broker, keep broker-backed regression coverage for lifecycle, reserved metadata keys, and renamed tool aliases. Compatibility-only tests are not enough to protect the production path.
- Any tool that fabricates or transforms memory identifiers must have a round-trip test through the paired read path (`memory_search` -> `memory_get`, `compact_context` -> `memory_get`).
- Retrieval accounting must canonicalize chunk hits to document IDs before persisting signals; otherwise promotion/explanation logic silently diverges from what agent-facing tools expose.
- Broker/process shutdown is not complete just because the UNIX socket disappears. For shared-store tests and one-shot broker calls, wait until the store lock is reusable before treating shutdown as finished.
- Deterministic process-harness temp roots must preserve persisted session artifacts across same-store restart tests. Do not delete the harness root in generic cleanup if later harness instances need to reopen the same broker-managed sessions.
- When broker client pathing depends on defaults, provide an explicit environment override for test harness isolation; overriding only the socket root is not enough if session stores still resolve under the real home directory.

## Delivery Scope

- When the user asks for an end-to-end completion, do not stop at core code changes. Finish the proof surface too: verification scripts, deployment docs, benchmark harnesses, and task-log updates.
- When the user explicitly says to keep going until everything is finished, do not hand back control after a verified subtask. Treat each green slice as an internal checkpoint and continue automatically unless the full requested scope is complete or a real blocker stops progress.
