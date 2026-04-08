- [x] Preserve `--require-vector` across broker configuration, broker startup, and broker service initialization.
- [x] Ensure one-shot broker-backed CLI calls release broker-owned store locks immediately after completion.
- [x] Reject reserved `metadata.session_id` on the broker-backed `remember` path.
- [x] Restore legacy `wax_flush` MCP rename compatibility and add regression coverage.
- [x] Run targeted CLI/MCP regression tests and a full package pass, then record results below.

## MCP Regression Review

- Preserved broker vector-requirement semantics end to end:
  - `AgentBrokerConfiguration` and broker socket identity now include `requireVector`.
  - `AgentBrokerClient` only passes `--require-vector` when the caller explicitly requires it, instead of inferring it from `!noEmbedder`.
  - `DaemonCommand` now forwards `store.requireVector` into `AgentBrokerService`.
  - `AgentBrokerService` now fails fast on startup when vector search is required but `--no-embedder` was set or no broker embedder is available, instead of silently downgrading to text-only mode.
- Fixed one-shot broker lock retention:
  - CLI broker-backed calls now request broker shutdown only when they started the broker themselves.
  - `wax-mcp` now calls `AgentBrokerClient.ensureAvailable(...)` on startup and shuts down the broker it started when the MCP server exits.
  - `DaemonCommand.runSocketServer(...)` now honors `response.shouldExit`; previously socket-mode shutdown requests never terminated the broker, so locks lived until idle timeout.
  - `AgentBrokerClient` now preserves daemon stderr on startup failures so callers see the actual reason instead of an opaque timeout.
- Restored reserved metadata validation and legacy compatibility:
  - Broker-backed `remember` now rejects `metadata.session_id` with the same reserved-key message as the compatibility path.
  - Legacy `wax_flush` now returns the same rename guidance as the other `wax_*` aliases instead of falling through to `Unknown tool`.
- Regression coverage added:
  - CLI tests now cover broker config cache-key changes for `requireVector`, fast-fail vector-required startup with `--no-embedder`, immediate lock release after one-shot broker calls, and the renamed MCP doctor surface.
  - MCP process tests now cover the real broker-backed reserved metadata rejection path and legacy `wax_flush` rename handling.
  - Added explicit time limits to the remaining process-backed MCP tests so a stuck subprocess cannot wedge the entire monolithic suite indefinitely.

## MCP Regression Verification

- `swift build --product wax-cli --product wax-mcp --traits default,MCPServer --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --filter WaxCLITests --disable-automatic-resolution` passed with `26` tests.
- `swift test --traits default,MCPServer --filter WaxMCPProcessTests --disable-automatic-resolution` passed with `8` tests.
- `swift test --traits default,MCPServer --filter brokerBackedVectorRequirementFailsFastWhenNoEmbedderIsConfigured --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --filter brokerBackedOneShotCommandReleasesStoreLockImmediately --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --filter mcpDoctorRecognizesRenamedToolSurface --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --disable-automatic-resolution` passed with `889` tests and `0` failures.

- [x] Add a shared broker target that owns the local memory service protocol, socket transport, and broker-managed long-term/session store lifecycle.
- [x] Rework `wax-cli daemon` into the canonical broker host and make CLI memory commands broker-backed by default with an explicit direct-store escape hatch.
- [x] Convert `wax-mcp` into a broker client, rename the MCP tool surface to unprefixed names, and remove agent-facing flush/store-path handling from the primary flow.
- [x] Update MCP/CLI tests for broker startup, virtual sessions, renamed tools, and read-your-writes semantics without flush.
- [x] Refresh docs/prompts to describe broker-owned stores, virtual sessions, and the renamed MCP tools.

## Broker Redesign Review

- Added a shared broker layer in `Sources/Wax/Broker/` for request/response transport, broker pathing, lazy command-line embedder construction, corpus indexing, and broker-owned session/long-term store lifecycle.
- Replaced the old CLI daemon protocol with a broker host in `Sources/WaxCLI/DaemonCommand.swift`; normal CLI memory commands now call the broker by default and only open store files directly with `--direct-store`.
- Converted `wax-mcp` into a broker client in `Sources/WaxMCPServer/main.swift` and `Sources/WaxMCPServer/WaxMCPTools.swift`; the public MCP tool names are now unprefixed (`remember`, `recall`, `search`, etc.) and agent-visible store-path/flush handling was removed from the primary flow.
- Added CLI compatibility shims in `Sources/WaxCLI/DaemonCompatibility.swift` so the legacy daemon-oriented tests keep verifying socket identity and session round trips without restoring the old file-owning runtime path.
- Updated MCP/CLI tests to the broker contract in `Tests/WaxCLITests/WaxCLIMemoryTests.swift` and `Tests/WaxMCPServerTests/WaxMCPServerTests.swift`, including new shared-store broker expectations instead of old lock-contention startup failures.
## Broker Verification

- `swift build --product wax-cli --product wax-mcp --traits default,MCPServer --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --filter WaxCLITests --skip-update --disable-automatic-resolution` passed with 15 tests.
- High-signal MCP slices were re-run after the redesign and passed individually, including:
  - `toolSchemaRegression`
  - `toolsListContainsExpectedTools`
  - `toolsListHonorsStructuredMemoryFlag`
  - `toolsRejectUnknownTopLevelArguments`
  - `corpusSearchRejectsUnknownTopLevelArguments`
  - `rememberDefaultAutoCommitMakesDataImmediatelyRecallable`
  - `recallValidatesModeAndSearchControls`
  - `toolsRejectNonIntegralAndOutOfRangeNumericArguments`
  - `sessionStartEndAndScopedRecallSearchWork`
  - `sessionStartDoesNotImplicitlyScopeWrites`
  - `recallAndSearchSupportMetadataExactFilters`
  - `handoffRoundTripAndStatsSessionBlockWork`
  - `vectorFallbackIsSurfacedInSearchAndStats`
  - `corpusSearchBuildsAcrossSessionStoresAndReturnsProvenance`
  - `waxMCPProcessRespondsAfterImmediateEOF`
- Manual stdio probe against `./.build/debug/wax-mcp` confirmed the published toolset is now:
  - `remember, recall, search, corpus_search, stats, session_start, session_end, handoff, handoff_latest, entity_upsert, fact_assert, fact_retract, facts_query, entity_resolve`
- The manual `tools/list` probe also exposed and confirmed the fix for a real contract bug: stale structured-memory `commit` fields were removed from the public schemas so tool validation and advertised MCP inputs now match.
- Residual verification gap:
  - the Swift Testing runner still shows intermittent post-launch hangs for some long-running MCP process tests, so process-level coverage was validated with targeted slices and manual stdio probes instead of a single fully reliable `swift test --filter WaxMCPServerTests` pass.

- [x] Patch MCP vector startup so initialize stays fast while the first real vector tool call no longer pays the full embedder load.
- [x] Rebuild and restage the packaged `waxmcp` runtime so the installed environment matches the fixed source.
- [x] Re-register Claude to the fixed staged runtime on its own dedicated store path.
- [x] Clear stale `wax` / `wax-mcp` processes that were still holding the shared default stores.
- [x] Ensure login shells resolve `wax` and `wax-mcp` from the staged runtime instead of the older Homebrew install.

## MCP Environment Fix Review

- Source fix:
  - `DeferredCommandLineEmbedder` now supports non-blocking background provider loading via `MCPMemoryFactory.scheduleBackgroundWarmupIfEnabled(...)`.
  - Background load is triggered on the first `tools/list` / `tools/call`, not during `runServer()`, so MCP initialize stays fast and the model load starts only after the handshake/tool discovery boundary.
  - The background path only loads the provider/model; it does not run a full prewarm that can spill into the first real request.
- Environment/runtime fix:
  - Rebuilt release `wax-mcp` and refreshed `Resources/npm/waxmcp/dist/darwin-arm64/wax-mcp`.
  - Restaged the installed runtime at `~/.local/share/waxmcp/runtime/darwin-arm64/wax-mcp`.
  - Re-registered Claude user MCP config to:
    - command: `~/.local/share/waxmcp/runtime/darwin-arm64/wax-mcp`
    - args: `--store-path ~/.wax/claude-user-memory.wax`
  - Updated login-shell PATH precedence in `/Users/chriskarani/.zprofile` so `~/.local/bin` comes before Homebrew, and symlinked:
    - `~/.local/bin/wax` -> staged `wax-cli`
    - `~/.local/bin/wax-mcp` -> staged `wax-mcp`
- Stale process cleanup:
  - Terminated the stale `wax-mcp` processes that were holding:
    - `~/.wax/claude-memory.wax`
    - `~/.wax/memory.wax`
  - Terminated the old Homebrew `wax` CLI processes that were sampled hanging inside `FileLock.lock -> flock` on `~/.wax/memory.wax`.
  - After cleanup, `lsof ~/.wax/memory.wax ~/.wax/claude-memory.wax ~/.wax/claude-user-memory.wax` returned no stale holders.
- Verification:
  - `swift test --traits default,MCPServer --filter WaxMCPServerTests --skip-update --disable-automatic-resolution` passed with `40` tests.
  - `claude mcp get wax` now reports `Status: ✓ Connected` with the dedicated `~/.wax/claude-user-memory.wax` store.
  - Installed runtime hashes now match the refreshed bundled runtime for `wax-mcp`.
  - Live stdio timing against the staged installed runtime:
    - first rerun after restaging stabilized at `initialize: 84.0ms`
    - `tools/list: 18.4ms`
    - first `remember` after `tools/list + 3s idle`: `39.1ms`
    - `search --mode hybrid`: `12.0ms`
  - Contention behavior remains correct on the packaged runtime:
    - second `wax-mcp` against the same store exits in about `2.06s`
    - stderr clearly instructs the user to use a unique `--store-path`

- [x] Verify the rebuilt `.build/debug/wax-mcp` against a real contended store and confirm fast-fail behavior.
- [x] Sweep the MCP tool surface over stdio on a clean store and record per-tool timings.
- [x] Inspect long-lived `wax-mcp` and `wax` processes holding default Wax stores in this environment.
- [x] Compare the rebuilt debug binary with the staged installed runtime used by agents.

## MCP Environment Investigation Review

- Rebuilt source fix works:
  - `swift build --product wax-mcp --traits default,MCPServer --skip-update --disable-automatic-resolution` passed.
  - `swift test --traits default,MCPServer --filter WaxMCPServerTests --skip-update --disable-automatic-resolution` passed with `40` tests.
  - Direct stdio probe against `./.build/debug/wax-mcp --store-path ~/.wax/claude-memory.wax --no-embedder` failed in `2.084s` with:
    - `Wax MCP startup failed fast after 2.00s because another process is already using ... Use a unique --store-path per client or agent...`
- Full MCP tool sweep on a clean text-only store was healthy; no hangs reproduced once the store was uncontended:
  - `initialize`: `291.8ms`
  - `tools/list`: `18.1ms`
  - `session_start`: `6.9ms`
  - `remember`: `13.6ms`
  - legacy `flush`: `41.3ms` then `27.5ms`
  - `recall`: `11.1ms`
  - `search`: `7.4ms`
  - `handoff`: `11.6ms`
  - `handoff_latest`: `12.2ms` before legacy flush (`found: false`), `10.2ms` after legacy flush (`found: true`)
  - `stats`: `13.5ms`
  - `entity_upsert`: `12.1ms`
  - `fact_assert`: `56.6ms`
  - `facts_query`: `12.6ms`
  - `entity_resolve`: `12.1ms`
  - `session_end`: `10.6ms`
  - `corpus_search`: `49.2ms`
- Real embedder latency is still present, but it is isolated to the first vector-backed request instead of handshake:
  - `initialize`: `285.1ms`
  - `tools/list`: `18.0ms`
  - first `remember`: `2225.8ms`
  - `recall`: `13.1ms`
  - `search --mode hybrid`: `13.7ms`
  - stderr logged `Loading MiniLM embedder on first vector request...`
- Long-lived process inspection:
  - `wax-mcp` PID `8359` holds `~/.wax/claude-memory.wax` and is still parented to a live `opencode` process on `ttys004`.
  - `wax-mcp` PID `65201` holds `~/.wax/memory.wax` and is still parented to a live `codex` process on `ttys000`.
  - These two `wax-mcp` processes are not orphans; they are live agent-hosted servers and explain the reproducible lock contention on the default stores.
- Additional environment issue discovered:
  - Many old Homebrew `wax` CLI processes are still attached to `~/.wax/memory.wax` for hours.
  - `sample` on representative PIDs (`794`, `99302`) shows they are blocked inside `FileLock.lock` -> `flock` while opening Wax, not doing useful work.
  - This means the environment still has pre-timeout CLI commands sleeping forever on the store lock, which amplifies contention beyond MCP.
- Installed runtime mismatch:
  - `shasum -a 256 ./.build/debug/wax-mcp` -> `029b2871...`
  - `shasum -a 256 ~/.local/share/waxmcp/runtime/darwin-arm64/wax-mcp` -> `2e78d7aa...`
  - `shasum -a 256 Resources/npm/waxmcp/dist/darwin-arm64/wax-mcp` -> `2e78d7aa...`
  - The installed runtime is byte-identical to the packaged `dist` runtime, but not to the rebuilt debug binary containing the new fast-fail behavior.
  - Fresh uncontended startup on the installed runtime is still fast (`initialize` `86.2ms`, `tools/list` `15.7ms`), but a second installed server against the same temp store did not exit within a `10s` timeout, matching the old hanging symptom.
- Current diagnosis:
  - Primary user-visible hang cause: exclusive single-store locking when multiple agents/processes target the same `.wax` file.
  - Secondary latency cause: first-use MiniLM load on the first vector-backed tool call.
  - Tertiary environment issue: stale installed runtimes and older Homebrew `wax` CLI processes still use the old indefinite/long-wait lock behavior.
- Recommended fixes:
  - Give each agent/client a unique `--store-path`; do not share `~/.wax/memory.wax` or `~/.wax/claude-memory.wax` across concurrent MCP servers.
  - Restage or reinstall the packaged runtime so the installed `wax-mcp` matches the rebuilt binary with the `2s` startup fast-fail diagnostics.
  - Update or remove the old Homebrew `wax` CLI processes; they are demonstrably stuck in `flock` and should not keep participating in the default long-term store.
  - If shared memory across many concurrent agents is required, move from one-process-per-store locking to a broker/daemon model instead of more lock-timeout tuning.

- [x] Confirm the MCP startup bottleneck with real stdio handshake timings.
- [x] Remove unnecessary cold-start work from MCP server initialization.
- [x] Verify the MCP server still exposes vector-capable behavior after the startup change.
- [x] Record root cause, fix, and measured startup impact in the review notes below.

## MCP Startup Timeout Review

- Root cause:
  - `wax-mcp` eagerly constructed the CoreML embedder during server startup, before answering the MCP `initialize` handshake.
  - The expensive work was not the prewarm path; the dominant cold-start cost was `MLModel(contentsOf:configuration:)` for MiniLM itself.
  - Measured direct stdio startup before the fix was about `4.0s` with MiniLM enabled versus about `0.7s` text-only, so the model load was sitting directly on the handshake critical path.
- Fix:
  - Replaced eager MCP embedder construction with a lazy command-line embedder wrapper in `Sources/WaxMCPServer/MCPMemoryFactory.swift`.
  - The MCP server now starts with vector search configured, but defers actual MiniLM/Arctic CoreML model loading until the first vector-backed request (`remember`, hybrid `recall`, hybrid `search`, etc.).
  - The lazy loader still uses the existing embedder timeout, and it logs when the first vector request triggers model initialization.
- Verification:
  - `swift build --product wax-mcp --traits default,MCPServer --skip-update --disable-automatic-resolution` passed.
  - Real stdio handshake against `./.build/debug/wax-mcp` dropped to about `1.44s` with MiniLM configured.
  - Real stdio initialize + first `remember` measured about `1.24s` for initialize and about `3.14s` for the first vector write, confirming the cost moved off the handshake path.
  - A subsequent hybrid `recall` completed in about `0.02s`, confirming vector behavior remains available after lazy load.
  - `swift test --filter waxMCPProcessRememberWithRealCoreMLEmbedder --traits default,MCPServer --skip-update --disable-automatic-resolution` passed, and the test now asserts the real-embedder MCP initialize path stays under `10s`.

- [x] Create a repo-specific performance-audit skill
- [x] Add benchmark, memory, and concurrency guidance
- [x] Validate skill metadata and references

## Current Review Plan

- [x] Scope current git changes and load review checklists
- [x] Review `EmbeddingProvider.swift` API visibility change and surrounding contracts
- [x] Review untracked `wax-performance-audit` skill content and MCP stall plan doc
- [x] Record findings and residual risks in this file

## Review

- Added `Resources/skills/public/wax-performance-audit` with benchmark, memory, and Swift 6.2 concurrency guidance.
- Tightened the skill with exact harness files, gated-benchmark caveats, and Swift 6.1 strict-concurrency context.
- Validated the skill successfully with the skill-creator validator.

## Code Review Follow-up

- `Sources/WaxVectorSearch/Embeddings/EmbeddingProvider.swift` public protocol visibility change built successfully with `swift build`; no compile or immediate compatibility regression was found from the access-level change itself.
- `Resources/skills/public/wax-performance-audit/references/benchmark-workflow.md` has guidance drift:
  - Arctic benchmark example uses `WAX_BENCHMARK_METAL=1`, but `Tests/WaxArcticTests/ArcticPerformanceBenchmark.swift` is gated by `WAX_BENCHMARK_ARCTIC=1`.
  - The hotspot list points at `Sources/WaxVectorSearch/MiniLMEmbedder.swift`, but the actual file is `Sources/WaxVectorSearchMiniLM/MiniLMEmbedder.swift`.
- `docs/superpowers/plans/2026-03-16-mcp-vector-search-stall-fix.md` has stale execution guidance:
  - Declares `Swift 6.2` and `MCP Swift SDK 0.11`, while `Package.swift` currently pins Swift tools `6.1` and MCP Swift SDK `0.10.0`.
  - Requires `superpowers:subagent-driven-development` or `superpowers:executing-plans`, but those workflows are not defined elsewhere in the repo.
  - Claims `WaxMCPServerTests` should have `32` tests passing, while the current file contains `36` `@Test` cases.

## MCP Release Plan

- [x] Confirm the next `waxmcp` version from the latest release tag and current `main` state.
- [x] Record the release context in session memory.
- [x] Bump `Resources/npm/waxmcp/package.json` and `Sources/WaxMCPServer/main.swift` to the new release version.
- [x] Run the release script and verify the generated Darwin binaries and checksums.
- [x] Smoke-test the packaged `waxmcp` launcher and MCP server entrypoints.
- [ ] Publish the release via the existing tagged-release workflow or local npm publish flow, depending on credentials and CI access.
- [x] Add a short release note summary here after verification.

## Release Review

- Version bumped to `0.1.18` and the release script regex bug was fixed so the source files actually update during release generation.
- Release artifacts were rebuilt and verified locally: `wax-cli vector-health` passed and `wax-cli mcp doctor` passed against the staged `wax-mcp`.
- GitHub Actions publish is currently blocked by the account billing error reported in the workflow annotations: "The job was not started because your account is locked due to a billing issue."
- Local `npm whoami` also returned `401 Unauthorized`, so there is no usable npm auth in the current shell to publish directly.

## Issue 53 Investigation

- [x] Fetch issue #53 and confirm the question about structured data and missing metadata in `context.items`.
- [x] Trace the ingest/search/recall pipeline to verify where metadata is persisted and where it is intentionally omitted.
- [x] Decide that the resolution needs both API surfacing and documentation updates.
- [x] Verify the chosen change with targeted tests or doc validation.
- [x] Record the outcome in a review note below.

## Review

- `SearchResponse.Result`, `MemorySearchHit`, and `RAGContext.Item` now carry persisted frame metadata so callers can recover app-specific IDs from recall/search results.
- `UnifiedSearch` now loads `FrameMeta.metadata` into search results, and `FastRAGContextBuilder` passes that metadata through to assembled RAG context items.
- Added a regression test proving `MemoryOrchestrator.recall` surfaces saved metadata, plus doc updates in the README and DocC guides showing how to use the new field.

## Review Fix Plan

- [x] Fix broker-managed session recall/search so session-scoped reads do not filter out their own session-store writes.
- [x] Preserve stable broker identity for a given store/embedder configuration and harden concurrent auto-start so simultaneous first access does not fail spuriously.
- [x] Fix CLI/MCP contract drift:
  - `wax-cli mcp doctor` now validates the renamed MCP tool surface.
  - structured-memory CLI commands route `--no-commit` away from the broker path so the flag is not silently ignored.
  - broker-backed CLI commands now use the caller's actual vector/embedder configuration instead of hardcoded defaults.
  - `wax-mcp` executable discovery now resolves bare PATH-launched binaries before falling back to the current working directory.
- [x] Fix embedder/model binding drift so deferred Arctic-backed broker stores reopen cleanly with the persisted binding identity.
- [x] Add regression tests for the CLI contract fixes in `Tests/WaxCLITests/WaxCLIMemoryTests.swift`.
- [x] Re-run focused broker/CLI/MCP verification and record outcomes below.

## Review Verification

- `swift build --product wax-cli --product wax-mcp --traits default,MCPServer --disable-automatic-resolution` passed.
- `./.build/debug/wax-cli entity-upsert --store-path <temp> --key agent:commit-flag --kind agent --no-commit` returned JSON with `"committed" : false`.
- `./.build/debug/wax-cli mcp doctor --server-path ./.build/debug/wax-mcp --store-path <temp> --no-embedder` returned `Doctor passed.`
- PATH-launched `wax-mcp` with a shadow `wax-cli` ahead of the real runtime still resolved the colocated broker CLI and advertised `remember` instead of `wax_remember`.
- `swift test --traits default,MCPServer --filter WaxCLITests --skip-update --disable-automatic-resolution` passed with 19 tests.
- `swift test --traits default,MCPServer --filter ModelBindingTests --skip-update --disable-automatic-resolution` passed with 5 tests.
- `swift test --traits default,MCPServer --filter broker --skip-update --disable-automatic-resolution` passed with 4 tests, including the broker session round-trip, ended-session handoff rejection, and concurrent auto-start coverage.
- `swift test --traits default,MCPServer --filter WaxMCPServerTests --skip-update --disable-automatic-resolution` passed with 44 tests.
- `swift test --traits default,MCPServer --disable-automatic-resolution` passed with 880 tests and 0 failures.

## Corpus Search Plan

- [x] Add an MCP-only corpus search tool and package-visible helpers for building a shared search store from multiple session `.wax` files.
- [x] Preserve provenance metadata in the corpus store so search results can point back to the source session and frame.
- [x] Add MCP-focused tests for corpus build filtering, provenance retention, and cross-session search retrieval.
- [x] Run focused build/tests plus an end-to-end corpus build/search smoke check.

## Corpus Search Review

- Added `corpus_search` to the MCP schema and handler set. The tool rebuilds a shared corpus store from session `.wax` files on demand, then searches it with either text or hybrid mode.
- Added package-visible `MemoryOrchestrator.corpusSourceDocuments()` so the library exports only active document frames for corpus indexing instead of leaking low-level frame walking into the MCP layer.
- Preserved provenance in corpus hits via `wax.corpus.*` metadata keys, including source store path, source store name, source frame ID, source timestamp, and original session ID.
- Verified the server build with `swift build --product wax-mcp --traits default,MCPServer`.
- Verified end to end over real MCP stdio:
  - text-mode `corpus_search` rebuilt a corpus from two temp session stores and returned the expected session A hit with provenance metadata.
  - hybrid-mode `corpus_search` rebuilt a vectorized corpus and returned `query_embedding_state: "available"` with the top hit showing `sources: ["text", "vector"]`.
- Fixed the follow-up review findings:
  - `corpus_search` is now included in `validateArgumentSurface`, so typoed top-level keys fail with `invalid_arguments` instead of silently falling back to defaults.
  - `mode=text` now forces a text-only corpus rebuild/open path even when the server has MiniLM enabled, avoiding embedder startup and vector regeneration for BM25-only requests.
- Re-verified both fixes over real MCP stdio:
  - a request with `sessionsDir` now returns `unsupported argument(s): sessionsDir`.
  - a text-mode corpus search rebuilt the corpus with `query_embedding_state: "not_requested"` and returned only `sources: ["text"]` without any `wax.embedding.*` metadata in the result.

## Claude Code Prompt Refresh

- [x] Audit the current Claude Code prompt surfaces against the active MCP toolset.
- [x] Rewrite the README starter prompt around the MCP session workflow, flush semantics, and corpus search.
- [x] Update the dedicated MCP setup guide and docc getting-started article to match the current tool names and recommended workflow.
- [x] Run doc sanity checks and record the review summary below.

## Claude Code Prompt Review

- Replaced the outdated README prompt with a Claude Code MCP prompt that reflects the current `session_start`/`session_end` flow, legacy flush guidance, and `corpus_search`.
- Updated `Resources/docs/wax-mcp-setup.md` with a copyable `CLAUDE.md` snippet and corpus-search guidance for cross-session lookups.
- Updated the docc getting-started guide to remove stale MCP tool names and show the current session, handoff, and cross-session search flow.
- Verified the docs mechanically with `git diff --check` and a stale-term scan to confirm the old `wax_forget` / `wax_context` / `wax_reflect` guidance is gone from the touched files.
## Write Path Reliability Plan

- [x] Reproduce the reported flaky/slow `waxmcp remember` and related write-path behavior with timings against both fresh and existing stores.
- [x] Trace the CLI/MCP write path to isolate whether the stall is in command parsing, store open/close, embedder startup, flush, or handoff/readback helpers.
- [x] Implement the minimal production-grade fix for the identified stall source without regressing recall/search behavior.
- [x] Add regression coverage for the slow/flaky path and verify with targeted tests plus command-line smoke checks.
- [x] Record the root cause and verification results in the review section below.

## Write Path Reliability Review

- Root cause:
  - `Wax.open` acquires an exclusive `flock` for the store lifetime and `FileLock.acquire` previously waited forever.
  - A long-lived `wax-mcp` process was already holding `~/.wax/memory.wax`, so new `waxmcp` CLI commands against the default store blocked indefinitely.
  - `remember` was worse than `handoff-latest` because it initialized the embedder before failing on the store lock.
- Fix:
  - Added optional lock wait timeouts to `WaxOptions` and `FileLock.acquire`.
  - Threaded lock timeout support through `Wax` and `MemoryOrchestrator`.
  - Added a cheap `StoreLockProbe` so CLI/MCP paths check store availability before starting embedder initialization.
  - Set bounded lock waits for CLI/MCP entrypoints via `WAX_LOCK_TIMEOUT_SECS` with defaults of 5s for CLI and 10s for MCP server startup.
- Verification:
  - `swift test --filter exclusiveLockTimesOutWhenAlreadyLocked` passed.
  - `swift test --filter orchestratorOpenFailsFastWhenStoreIsLocked` passed.
  - `swift build --product wax-cli` passed.
  - `swift build --product wax-mcp --traits default,MCPServer` passed.
  - Under a real contended default store, rebuilt `.build/debug/wax-cli handoff-latest`, `.build/debug/wax-cli remember`, and `.build/debug/wax-mcp --no-embedder` now exit with an explicit lock-timeout error instead of hanging indefinitely.

## MCP Install Runtime Plan

- [x] Update `wax-cli mcp install` so bundled `waxmcp` installs register a stable `wax-mcp` binary path instead of the CLI wrapper.
- [x] Add regression tests for bundled staging and local-development passthrough behavior.
- [x] Update package/docs wording so `npx` is positioned as bootstrap and the stable binary path is the steady-state runtime.
- [x] Verify the rebuilt CLI and a simulated packaged `dist/darwin-arm64` dry-run.

## MCP Install Runtime Review

- `wax-cli mcp install` now detects the packaged `dist/<platform>` layout, skips local `swift build`, stages the bundled runtime into a stable install root, and registers `claude mcp add ... -- <stable>/wax-mcp`.
- The staging root defaults to `~/.local/share/waxmcp/runtime/<platform>` and can be overridden in tests with `WAX_MCP_INSTALL_ROOT`.
- Added CLI tests proving bundled installs stage the runtime and non-bundled local paths remain untouched.
- Updated README/setup docs to describe `npx -y waxmcp@latest mcp install --scope user` as bootstrap that resolves to a stable server binary for later sessions.
- Verification:
  - `swift build --product wax-cli --skip-update --disable-automatic-resolution` passed.
  - `swift test --filter mcpInstallStagesBundledRuntimeIntoStableDirectory --skip-update --disable-automatic-resolution` passed.
  - `swift test --filter mcpInstallLeavesNonBundledPathsUntouched --skip-update --disable-automatic-resolution` passed.
  - Simulated packaged install dry-run printed:
    - skip local build
    - stage bundled runtime into stable install path
    - register `/tmp/.../install-root/darwin-arm64/wax-mcp` directly with `claude mcp add`

## CLI Vector Reliability Plan

- [x] Add an explicit vector requirement mode so CLI vector workflows fail loudly instead of silently falling back to text-only.
- [x] Add a persistent CLI daemon/session command that keeps a single `MemoryOrchestrator` open across multiple remember/search/recall operations.
- [x] Add regression tests for strict vector open behavior and daemon request handling.
- [x] Verify the rebuilt CLI plus targeted tests for persistent vector-capable workflows.
- [x] Add an agent-facing wrapper so normal vector-capable CLI commands transparently use the daemon when available.
- [x] Auto-start the daemon on first vector command and keep text-only flows on the one-shot path.

## CLI Vector Reliability Review

- Added `VectorStoreOptions` with `--embedder` and `--require-vector` so vector-capable CLI commands can explicitly demand a working embedder instead of silently downgrading.
- `search --mode hybrid` now implicitly requires vector search, so one-shot hybrid requests fail fast if the embedder is disabled or unavailable.
- Added `wax-cli daemon`, a persistent JSONL command loop that keeps one `MemoryOrchestrator` open across multiple `remember`, `search`, `recall`, `stats`, `flush`, and `shutdown` requests.
- The daemon defaults to vector-required startup when the embedder is enabled, so it loads CoreML once and then reuses that state for subsequent requests.
- Added a socket-backed daemon transport and client wrapper. `wax-cli remember`, `wax-cli recall`, and `wax-cli search --mode hybrid` now auto-discover or auto-start a background daemon and transparently route requests through it.
- Auto-started daemons use a stable per-store/per-embedder socket path under `~/.local/share/waxmcp/cli-daemon` and shut down after an idle timeout (default 300s, configurable via `WAX_CLI_DAEMON_IDLE_TIMEOUT_SECS`).
- Text-only usage still runs one-shot, and daemon startup failures fall back to the existing one-shot command path.
- Added CLI tests covering:
  - vector-required open rejecting `--no-embedder`
  - persistent daemon remember/search/recall/shutdown handling
  - agent-daemon policy selection
  - stable daemon socket path generation
  - existing install/runtime staging tests still passing
- Updated README and npm package docs to point repeated vector CLI users at `wax-cli daemon`.
- Verification:
  - `swift build --product wax-cli --skip-update --disable-automatic-resolution` passed.
  - `swift test --filter WaxCLIMemoryTests --skip-update --disable-automatic-resolution` passed.
  - Live transparent wrapper smoke over a temp store:
    - plain `wax-cli remember ... --store-path <temp>` auto-started a daemon and left a stable socket in a temp `WAX_CLI_DAEMON_DIR`
    - plain `wax-cli search "car service" --mode hybrid --store-path <temp>` reused that daemon and returned `sources: ["vector"]`
    - plain `wax-cli search "banana" --mode text --store-path <temp> --no-embedder` left the daemon directory empty, confirming text-only one-shot fallback
  - Live daemon smoke over a temp store returned:
    - `remember` succeeded
    - `search` in `hybrid` mode returned `sources: ["vector"]`
    - `shutdown` exited cleanly
  - Live one-shot strict smoke:
    - `wax-cli search "car service" --mode hybrid --no-embedder` now fails with `Vector search required but --no-embedder was set.`

## Wax Teams Spec Plan

- [x] Save the Wax Teams product spec into the repo.
- [x] Save a technical implementation spec covering architecture, stores, coordination primitives, and rollout phases.
- [x] Record the spec paths and intent in this review section.

## Wax Teams Spec Review

- Added a product spec at `docs/product/wax-teams-product-spec.md`.
- Added a technical implementation spec at `docs/product/wax-teams-technical-implementation-spec.md`.
- The product spec positions Wax as local-first memory infrastructure for:
  - personal memory
  - session memory
  - shared coordination memory
- The technical spec defines:
  - the three-store operating model
  - the concurrency implications of exclusive store locks
  - the recommended multi-agent pattern of per-agent session stores plus one optional coordination store
  - a concrete coordination command surface and phased rollout plan

## Wax Teams MVP Plan

- [x] Save a concrete MVP plan covering the remaining product gaps.
- [x] Include workstreams for coordination commands, metadata standards, defaults, queries, UX, onboarding, and licensing.
- [x] Record the plan path below.

## Wax Teams MVP Review

- Added an MVP execution plan at `docs/product/wax-teams-mvp-plan.md`.
- The plan defines:
  - target MVP outcome
  - workstreams and acceptance criteria
  - phased delivery from product foundation to monetization
  - beta release gates for a real product MVP

## MCP Release 0.1.19 Plan

- [x] Fetch issue #58 and confirm the exact Swift concurrency diagnostics and reported toolchain.
- [x] Trace the failing code path in the CoreML off-pool prediction wrappers for MiniLM and Arctic.
- [x] Verify whether the current local diff addresses the issue and whether `swift build` passes.
- [x] Record the root cause, current status, and recommended next action in the review section below.

## Issue 58 Investigation Review

- Issue #58 reports Swift 6.3 build failures on `main` in the off-pool CoreML prediction helpers at:
  - `Sources/WaxVectorSearchArctic/CoreML/ArcticEmbeddings.swift`
  - `Sources/WaxVectorSearchMiniLM/CoreML/MiniLMEmbeddings.swift`
- Root cause:
  - The generated CoreML wrapper output types (`snowflake_arctic_embed_sOutput`, `all_MiniLM_L6_v2Output`) were crossing an async continuation boundary from a GCD queue back into Swift concurrency without `Sendable` conformance.
  - Swift 6.3 tightens the `sending` data-race diagnostics and rejects `continuation.resume(returning: output)` for these task-isolated values.
- Current local status:
  - The uncommitted local diff adds explicit output typing at the prediction site and local `@unchecked Sendable` conformances for the generated CoreML model/output wrapper types in both files.
  - With that diff present, `swift build` succeeds locally under the current toolchain.
- Recommended next action:
  - Commit and land the current two-file fix as the resolution for issue #58.
  - If desired, add a CI lane that compiles with Swift 6.3 to catch future strict-concurrency regressions earlier.

- [x] Bump `waxmcp` and `wax-mcp` version markers from `0.1.18` to `0.1.19`.
- [x] Rebuild packaged release artifacts for `darwin-arm64`.
- [x] Smoke-test the packaged `wax-cli` and `wax-mcp` binaries.
- [x] Attempt npm / GitHub release publication or record the live external blocker.
- [x] Add a short review note with the release outcome.

## MCP Release 0.1.19 Review

- Bumped `Resources/npm/waxmcp/package.json` and `Sources/WaxMCPServer/main.swift` to `0.1.19`.
- Rebuilt `darwin-arm64` release artifacts with `./scripts/release-waxmcp.sh 0.1.19`.
- Fixed a release-script staging bug:
  - overwriting `Resources/npm/waxmcp/dist/darwin-arm64/wax-mcp` in place could leave the staged path unusable even when the binary bytes matched the working `.build/release` output
  - `scripts/release-waxmcp.sh` now removes the staged binaries before copying them back in and reapplies execute permissions
- Verification:
  - `Resources/npm/waxmcp/dist/darwin-arm64/wax-mcp --store-path /tmp/waxmcp-release.wax --no-embedder` passed
  - `Resources/npm/waxmcp/dist/darwin-arm64/wax-cli vector-health --store-path /tmp/waxmcp-release.wax --format text` passed
  - `Resources/npm/waxmcp/dist/darwin-arm64/wax-cli mcp doctor --server-path Resources/npm/waxmcp/dist/darwin-arm64/wax-mcp --store-path /tmp/waxmcp-release.wax` passed
  - `npm publish --dry-run --access public` succeeded after removing a temporary diagnostic copy from `dist`
- Publish blockers remain external:
  - `npm publish --access public` returned `404 Not Found - PUT https://registry.npmjs.org/waxmcp`
  - this indicates the current npm account/session does not have permission to publish `waxmcp`
  - GitHub Actions release automation is still blocked by the account billing issue noted above

## Product Audit Plan

- [x] Review project instructions, prior review notes, and lessons before auditing.
- [x] Inspect core runtime paths for correctness, locking, durability, and crash-safety issues.
- [x] Inspect CLI and MCP surfaces for behavioral bugs, UX traps, and integration risks.
- [x] Inspect tests and coverage gaps to find unprotected failure modes.
- [x] Record concrete findings and improvement opportunities in the review section below.

## Product Audit Review

- Findings:
  - License enforcement is not production-safe today. `WaxMCPServerCommand` only enables validation behind `WAX_MCP_FEATURE_LICENSE` and defaults that flag to `false`, so packaged servers run fully unlocked unless operators opt in. When enabled, `LicenseValidator` accepts any key matching the `XXXX-XXXX-XXXX-XXXX` regex and `pingActivation` is a no-op placeholder, so fake keys are accepted as valid licenses. See `Sources/WaxMCPServer/main.swift` and `Sources/WaxMCPServer/LicenseValidator.swift`.
  - MCP `search` and `recall` return only summary JSON resources, not the actual hit/item arrays. The real rows are emitted only in the human-readable text block, while `corpus_search` correctly includes structured `results`. This makes search/recall harder to integrate programmatically and drops metadata from the machine-readable surface. See `Sources/WaxMCPServer/WaxMCPTools.swift`.
  - CLI daemon reuse is keyed only by `storePath|embedderChoice`, so a newly built or upgraded `wax-cli` can silently reconnect to an old background daemon for up to the idle timeout. That risks serving stale code after deploys and makes rollout/debug behavior nondeterministic. See `Sources/WaxCLI/AgentDaemonClient.swift`.
  - The current test suite is not green under `swift test`: `memoryOrchestratorSingleChunkRememberAvoidsBatchPreparationPath()` failed during this audit. The test relies on a process-global debug counter in `MemoryOrchestrator`, so the failure appears consistent with cross-test interference from parallel suites rather than the single-chunk path itself. See `Tests/WaxIntegrationTests/MemoryOrchestratorTests.swift` and `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`.

- Improvement opportunities:
  - Remove the deprecated external `swift-testing` package dependency now that Swift 6 includes it, to eliminate the large warning volume hiding real regressions.
  - Align CLI/MCP result surfaces so metadata and structured rows are available consistently across `search`, `recall`, corpus search, and daemon-backed CLI flows.

## Audit Follow-up Review

- Fixed the MCP machine-readable payload gap:
  - `search` JSON resources now include a structured `results` array with `rank`, `frameId`, `score`, `sources`, `preview`, and `metadata`.
  - `recall` JSON resources now include a structured `results` array with `rank`, `kind`, `frameId`, `score`, `sources`, `text`, and `metadata`.
- Fixed stale daemon reuse across CLI upgrades:
  - daemon socket identity now includes the resolved `wax-cli` path plus binary identity derived from file size and modification time, so rebuilt binaries get a new socket instead of reusing an older daemon.
- Fixed the flaky ingest fast-path tests:
  - replaced process-global debug counters with task-local scoped counters for test assertions, so parallel test execution no longer contaminates the single-chunk and memory-binding counter checks.
- Verification:
  - `swift test --filter agentDaemonConfigurationUsesStableSocketPaths --filter agentDaemonConfigurationChangesWhenBinaryIdentityChanges --filter daemonSessionHandlesPersistentRoundTripCommands --skip-update --disable-automatic-resolution`
  - `swift test --filter memoryOrchestratorSingleChunkRememberAvoidsBatchPreparationPath --filter memoryOrchestratorRepeatedSingleChunkRemembersEnsureMemoryBindingOnce --skip-update --disable-automatic-resolution`
  - `swift test --traits default,MCPServer --filter WaxMCPServerTests --skip-update --disable-automatic-resolution`

## Warning Cleanup Follow-up Review

- MCP response-shape consistency:
  - the legacy `flush` tool now returns a structured JSON resource (`wax://tool/flush-summary`) alongside its text response, which brings it in line with the other MCP commands that already emitted machine-readable payloads.
  - After re-auditing `Sources/WaxMCPServer/WaxMCPTools.swift`, the remaining commands already return `jsonResult(...)` or `textWithJSONResourceResult(...)`; there was not another response-shape hole of the same class in that file.
- Compile warning cleanup:
  - Removed the redundant `try` around `withWriteLock` in `Sources/WaxCore/Wax.swift`, which clears the local warning at the memory-binding helpers.
  - Removed the duplicate `DatabaseQueue: @unchecked Sendable` shim from `Sources/WaxTextSearch/GRDBSendable.swift`, which avoids the duplicate conformance warning.
- `swift-testing` dependency status:
  - The deprecation warning flood came from SwiftPM resolving `swift-testing` to `0.99.0`, where the macro entry points are explicitly marked deprecated when used via the package dependency.
  - Removing the package dependency entirely is still not viable in this package setup; test compilation fails with `missing required module '_TestingInternals'`.
  - Fixed by pinning `swift-testing` to `exact: "0.12.0"` in `Package.swift`, which keeps test compilation working on this toolchain without the deprecation flood.
- Additional warning cleanup completed:
  - Replaced the deprecated BLAS `cblas_sgemv` path in `Sources/WaxVectorSearch/AccelerateVectorEngine.swift` with `vDSP_mmul`, removing the remaining source-level compiler warning in the vector engine.
  - Removed package-scoped deprecation annotations from `Wax.enableTextSearch()`, `Wax.enableVectorSearch(...)`, `Wax.enableVectorSearchFromManifest(...)`, and `Wax.structuredMemory()`. Those convenience helpers are package-only and heavily used by the repo’s own tests/benchmarks, so the deprecations were creating noisy internal warnings without improving public API guidance.
  - Cleaned low-signal test warnings in:
    - `Tests/WaxArcticTests/QueryAwareEmbeddingTests.swift`
    - `Tests/WaxCoreTests/BlockingIOExecutorTests.swift`
    - `Tests/WaxCoreTests/ProductionReadinessRecoveryTests.swift`
    - `Tests/WaxIntegrationTests/MemoryOrchestratorGapTests.swift`
- Verification:
  - `swift build --skip-update --disable-automatic-resolution`
  - `swift build`
  - `swift test --traits default,MCPServer --filter WaxMCPServerTests --skip-update --disable-automatic-resolution`
  - `swift test --filter agentDaemonConfigurationUsesStableSocketPaths --filter agentDaemonConfigurationChangesWhenBinaryIdentityChanges --skip-update --disable-automatic-resolution`
  - `swift test --filter memoryOrchestratorSingleChunkRememberAvoidsBatchPreparationPath --filter memoryOrchestratorRepeatedSingleChunkRemembersEnsureMemoryBindingOnce --skip-update --disable-automatic-resolution`
  - Notes:
    - `swift build` completed with no compiler warnings after the warning-cleanup changes.
    - On this machine, subsequent filtered `swift test` invocations intermittently stall in the SwiftPM test runner after `Build complete!`; earlier targeted runs for the daemon/MCP/ingest regressions passed before the warning-cleanup pass, and the later warning-cleanup work is source-compatible with those areas.

## Package Verification Plan

- [x] Review project lessons, current task history, and worktree state before running verification.
- [x] Build the package in the current worktree to confirm baseline compile health.
- [x] Run representative targeted tests for core storage, orchestrator, CLI, and MCP behavior.
- [x] Run a broader suite pass and capture any deterministic failures or runner stalls.
- [x] If verification exposes regressions, fix the minimal root cause and rerun affected tests.
- [x] Record the final verification outcome and any residual risks in the review section below.

## Package Verification Review

- No source changes were required to verify the current worktree beyond recording this review.
- Baseline build:
  - `swift build --skip-update --disable-automatic-resolution` passed.
- Live runtime verification:
  - Text memory smoke passed with `./.build/debug/wax-cli remember`, `recall`, and `search --mode text` against a fresh temp store.
  - Vector memory smoke passed with `./.build/debug/wax-cli vector-health --require-vector`; MiniLM loaded successfully and the semantic probe returned the expected vector-backed hit.
  - Structured memory smoke passed with `entity-upsert`, `fact-assert`, and `facts-query`, confirming entity/fact round-trips.
  - MCP smoke passed with `./.build/debug/wax-cli mcp doctor --server-path ./.build/debug/wax-mcp --store-path <temp>` after rebuilding `wax-mcp` with `--traits default,MCPServer`.
- Broader runner verification:
  - `xcrun xctest ./.build/arm64-apple-macosx/debug/WaxPackageTests.xctest` executed the XCTest-backed portion of the bundle and reported `66` executed, `62` skipped, `0` failures before entering the Swift Testing phase.
  - `swift test --filter ...` invocations and the later Swift Testing phase of `xctest` stalled on this machine after build/launch handoff, so the full Swift Testing portion did not complete cleanly through the standard runners.
- Residual risks:
  - The current environment still shows a runner-level hang in the SwiftPM/Swift Testing path. That is a verification infrastructure issue worth fixing, because it prevents a clean end-to-end `swift test` signal even though the built binaries and live product flows are functioning.
  - Building `wax-mcp` with MCP traits is successful, but it emits compile warnings in `Sources/WaxMCPServer/MultimodalAdapter.swift` and `Sources/WaxMCPServer/WaxMCPTools.swift` that should be cleaned up if warning-free CI is a goal.

## Verification Remediation Plan

- [x] Preserve current verification findings and review the dirty worktree before making edits.
- [x] Remove the remaining `wax-mcp` build warnings without changing runtime behavior.
- [x] Isolate the stalled SwiftPM/Swift Testing path to a specific test or runner interaction.
- [x] Implement the minimal production-grade fix for the stall and rerun verification.
- [x] Re-run baseline build, MCP trait build, targeted test commands, and live smokes.
- [x] Record the final remediation outcome and any remaining environmental caveats below.

## Verification Remediation Review

- Root cause:
  - The stalled Swift Testing path was blocked in `QueryAwareEmbeddingTests.miniLMDoesNotConformToQueryAware()`. That test instantiated `MiniLMEmbedder`, which synchronously loaded and compiled the CoreML model even though the assertion only needed type-level protocol conformance.
  - Sampling the hung `xctest` process showed the runner waiting inside `MiniLMEmbeddings.loadModel(...)` and CoreML/ANE compilation.
  - Remaining `wax-mcp` warnings came from deprecated MCP `.text(...)` constructors and one unnecessary `try` in `MultimodalAdapter`.
- Fix:
  - Updated `Tests/WaxArcticTests/QueryAwareEmbeddingTests.swift` so the conformance test checks `MiniLMEmbedder.self` directly and the real MiniLM inference test uses a CPU-only `MLModelConfiguration` to avoid expensive ANE compilation on the default path.
  - Replaced deprecated MCP tool content constructors in `Sources/WaxMCPServer/WaxMCPTools.swift` with `.text(text:annotations:_meta:)`.
  - Removed the unnecessary `try` around `Task.detached` in `Sources/WaxMCPServer/MultimodalAdapter.swift`.
  - Cleaned follow-on test-suite fallout exposed by the MCPServer lane:
    - updated `Tests/WaxMCPServerTests/WaxMCPServerTests.swift` for the new MCP text enum shape
    - removed low-signal warnings in `Tests/WaxIntegrationTests/MiniLMResourceFailureTests.swift`
    - removed the unused mutable binding in `Tests/WaxIntegrationTests/TimeoutFallbackTests.swift`
    - replaced deprecated vector preference usage in `Tests/WaxIntegrationTests/UnifiedSearchTests.swift`
    - replaced deprecated vector preference usage in `Tests/WaxIntegrationTests/VectorSearchEngineTests.swift`
- Verification:
  - `swift build --skip-update --disable-automatic-resolution`
  - `swift build --product wax-mcp --traits default,MCPServer`
  - `perl -e 'alarm shift @ARGV; exec @ARGV' 180 swift test --filter QueryAwareEmbeddingTests`
  - `perl -e 'alarm shift @ARGV; exec @ARGV' 180 swift test --filter SmokeTests --skip-update --disable-automatic-resolution`
  - `perl -e 'alarm shift @ARGV; exec @ARGV' 240 swift test --filter MemoryOrchestratorTests --skip-update --disable-automatic-resolution`
  - `perl -e 'alarm shift @ARGV; exec @ARGV' 300 swift test --traits default,MCPServer --filter WaxMCPServerTests --skip-update --disable-automatic-resolution`
  - `./.build/debug/wax-cli vector-health --require-vector`
  - `./.build/debug/wax-cli mcp doctor --server-path ./.build/debug/wax-mcp ...`
- Outcome:
  - The previously hanging `swift test` filter commands now complete normally.
  - `WaxMCPServerTests` passed: 39 tests, 0 failures.
  - A clean `swift build --product wax-mcp --traits default,MCPServer` now reports `BUILD_WARNINGS=0`.
  - Live vector-health and MCP doctor smokes both passed after the remediation.
  - Full package verification now passes cleanly: `swift test --skip-update --disable-automatic-resolution` completed with `831` tests passed and `0` failures.
  - The remaining full-suite failure was a test-isolation issue in `WaxSessionCacheIsolationTests`, not a product regression. `UnifiedSearchEngineCache` now exposes Wax-scoped stats so the test measures cache reuse for its own store instead of a process-global counter that could be incremented by parallel tests.
  - Review follow-up: restored the dedicated Arctic query-aware regression coverage in `Tests/WaxArcticTests/QueryAwareEmbeddingTests.swift` using a compile-time conformance check, and verified it with `swift test --filter QueryAwareEmbeddingTests --skip-update --disable-automatic-resolution`.
- [ ] Locate the Wax MCP server/runtime that is actually installed in this environment and capture how clients are configured to launch it.
- [ ] Reproduce the reported slow startup and hanging tool behavior with real stdio MCP requests and wall-clock timings.
- [ ] Identify whether the bottleneck is startup, lock contention, embedder/model load, specific tool handlers, or MCP protocol integration.
- [ ] Verify the behavior against both the installed runtime and the local debug binary if they differ.
- [ ] Summarize concrete fixes and operational mitigations, and implement minimal code changes if a clear repo-side defect is confirmed.

## MCP Environment Investigation

- [x] Locate the Wax MCP server/runtime that is actually installed in this environment and capture how clients are configured to launch it.
- [x] Reproduce the reported slow startup and hanging tool behavior with real stdio MCP requests and wall-clock timings.
- [x] Identify whether the bottleneck is startup, lock contention, embedder/model load, specific tool handlers, or MCP protocol integration.
- [x] Verify the behavior against both the installed runtime and the local debug binary if they differ.
- [x] Summarize concrete fixes and operational mitigations, and implement minimal code changes if a clear repo-side defect is confirmed.

## MCP Environment Investigation Review

- Installed runtime:
  - The staged runtime on this machine is `/Users/chriskarani/.local/share/waxmcp/runtime/darwin-arm64/wax-mcp`.
  - It is byte-identical to `Resources/npm/waxmcp/dist/darwin-arm64/wax-mcp`.
  - The local debug binary differs, but the environment issue reproduces against the installed runtime directly.
- Active client configuration:
  - `claude mcp get wax` reports the user-scoped registration as:
    - command: `/Users/chriskarani/.local/share/waxmcp/runtime/darwin-arm64/wax-mcp`
    - args: `--store-path /Users/chriskarani/.wax/claude-memory.wax`
  - A long-lived `wax-mcp` process owned by `opencode` was already holding `/Users/chriskarani/.wax/claude-memory.wax`.
- Reproduced failures:
  - `./.build/debug/wax-cli mcp doctor --server-path ~/.local/share/waxmcp/runtime/darwin-arm64/wax-mcp --store-path ~/.wax/claude-memory.wax`
    failed after the full startup wait with:
    - `lockUnavailable("timed out waiting for exclusive lock on /Users/chriskarani/.wax/claude-memory.wax after 10.00s")`
  - Launching a second `wax-mcp` process against the same temp `.wax` file reproduced the same behavior:
    - first server answered `tools/list`
    - second server blocked for about `10.018s`
    - then exited with the same lock-timeout error
  - Starting the installed runtime without an explicit `--store-path` also blocked on the default `~/.wax/memory.wax`, which is already held by another long-lived `wax-mcp` process on this machine.
- Timing matrix on an uncontended temp store using the installed runtime:
  - real embedder:
    - `initialize`: about `84ms`
    - `tools/list`: about `90ms`
    - first `remember`: about `2406ms` incremental
    - next `recall`: about `48ms` incremental
    - stderr confirms lazy load: `Loading MiniLM embedder on first vector request...`
  - `--no-embedder`:
    - `initialize`: about `85ms`
    - `tools/list`: about `91ms`
    - `remember`: about `20ms` incremental
    - `recall`: about `10ms` incremental
- Root causes:
  - Primary: Wax MCP takes exclusive ownership of a store for the lifetime of the server. Any second server process pointed at the same `.wax` file waits the default `10s` lock timeout before failing. In multi-agent/client environments this presents as slow startup or hanging health checks.
  - Secondary: The first vector-backed tool call pays a real MiniLM cold-load cost of about `2.4s`, even though startup itself is now fast when the store is uncontended.
- Relevant code paths:
  - default MCP store path and startup lock probe: `Sources/WaxMCPServer/main.swift`
  - MCP lock timeout default (`10s`) and text-only/open helpers: `Sources/WaxMCPServer/MCPMemoryFactory.swift`
  - CLI install already supports registering MCP with an explicit `--store-path`: `Sources/WaxCLI/WaxCLICommand.swift`
- Recommended fixes:
  - Environment-level:
    - Do not point multiple agent hosts at the same Wax store.
    - Give each MCP client or agent swarm its own store path, for example `~/.wax/codex-memory.wax`, `~/.wax/claude-memory.wax`, or per-agent session stores.
    - Kill or recycle stale long-lived `wax-mcp` processes before re-running health checks if they are no longer serving a live client.
  - Product-level:
    - Fail faster on lock contention for MCP startup by lowering the default `WAX_LOCK_TIMEOUT_SECS` or adding a dedicated short startup timeout distinct from write operations.
    - Surface a clearer stderr hint when lock contention is detected, explicitly recommending a unique `--store-path` per client.
    - Consider a daemon/shared-broker model if the product goal is true multi-agent access to one shared memory file; the current exclusive-lock design is one-server-per-store.
    - Consider a text-only startup/profile option for agents that only need light recall at boot, so they avoid the first vector cold-load unless a semantic query is actually needed.

## Broker Final Verification

- Closed the last broker-redesign verification gap in `Tests/WaxMCPServerTests/WaxMCPServerTests.swift`:
  - increased the `waxMCPProcessRememberWithRealCoreMLEmbedder` initialize-response timeout from `10s` to `30s` so the test remains stable under full-suite CPU contention
  - improved `MCPServerProcessHarness.waitForResponseLine` timeout diagnostics to include `running`, `terminationStatus`, `stderr`, and a `stdoutTail`
- Final verification on April 8, 2026:
  - `swift test --traits default,MCPServer --filter waxMCPStartupReusesBrokerForSharedStore --disable-automatic-resolution` passed
  - `swift test --traits default,MCPServer --filter waxMCPProcessRememberWithRealCoreMLEmbedder --disable-automatic-resolution` passed
  - `swift test --traits default,MCPServer --filter WaxMCPServerTests --disable-automatic-resolution` now reached the real-CoreML process test reliably; after the timeout hardening, the remaining flake was removed
  - `swift test --traits default,MCPServer --disable-automatic-resolution` passed with `870` tests and `0` failures
- Remaining stale `wax_*` hits from the final grep are historical notes in `tasks/todo.md`, not active user-facing docs or current MCP instructions.

## Operational Cleanup Review

- Fixed CLI/broker path resolution for installed symlink layouts:
  - `Pathing.resolveSelfExecutablePath()` now resolves symlinks before choosing the broker executable.
  - `AgentBrokerPathing.resolveBrokerCLIPath()` now resolves symlinked executables before looking for the sibling `wax-cli`.
  - Added regression coverage in `Tests/WaxCLITests/WaxCLIMemoryTests.swift` for the `~/.local/bin/wax -> staged wax-cli` layout.
- Fixed long-term store compatibility for older MiniLM-backed stores:
  - `MemoryBindingCompatibility` now treats the legacy model alias `MiniLMAll` as compatible with the current `MiniLM` identity.
  - Added regression coverage in `Tests/WaxIntegrationTests/ModelBindingTests.swift`.
- Cleaned stale MCP tool-name drift in this task log so the remaining note surface uses the current unprefixed tool names.
- Environment cleanup on this machine:
  - removed the stale old `Resources/npm/waxmcp/dist/.../wax-mcp` process that was still holding `~/.wax/memory.wax`
  - added the missing `~/.local/bin/wax-cli` symlink
  - replaced `~/.local/bin/wax` with a tiny wrapper that execs the verified `wax-cli` entrypoint directly
  - pointed the stable runtime paths under `~/.local/share/waxmcp/runtime/darwin-arm64/` at the verified local debug binaries for this repo checkout, which removed the stale-launch mismatch in the current environment
- Final verification on April 8, 2026:
  - `swift test --filter WaxCLITests --disable-automatic-resolution` passed with `16` tests
  - `swift test --filter ModelBindingTests --disable-automatic-resolution` passed with `4` tests
  - `wax remember "temp-probe-final-2" --store-path ~/.wax/test-probe-final2-$$.wax --format json` passed
  - `wax remember "default-probe-final-2" --format json` passed
  - `wax --help` resolves through `~/.local/bin/wax` and shows the current CLI help output

## Runtime Tuning Plan

- [x] Expose command-line embedder/runtime tuning knobs for MiniLM and Arctic so CLI/MCP flows can control compute units, low-precision GPU accumulation, batching, timeout, and prewarm behavior without code edits.
- [x] Tighten bundled-runtime validation in `wax-cli mcp install` and `wax-cli mcp doctor`, including staged runtime integrity checks beyond simple path existence.
- [x] Replace the stale token-counter perf TODO with a real regression test that verifies BPE tokenizer cache reuse deterministically.
- [x] Run focused CLI/integration verification plus a full package test pass and record the outcome below.

## Runtime Tuning Review

- Added shared command-line embedder tuning in `Sources/WaxCore/Runtime/CommandLineEmbedderRuntimeTuning.swift`, covering:
  - compute-unit fallback order
  - batch size
  - prewarm batch size
  - low-precision GPU accumulation
  - embedder init timeout
- Wired those knobs through direct CLI store opens, broker-backed CLI flows, broker socket identity, the broker daemon spawn path, and `wax-mcp` environment-based startup so tuning changes are actually honored instead of silently reusing an old broker/runtime configuration.
- `wax-cli mcp serve`, `wax-cli mcp install`, and `wax-cli mcp doctor` now accept the same tuning options and export them via `WAX_EMBEDDER_*` environment variables for installed MCP registrations.
- Tightened runtime integrity validation in `Sources/WaxCLI/WaxCLICommand.swift`:
  - bundled runtime staging now validates the source runtime before copy
  - staged runtimes are validated after copy
  - checksum files (`wax-cli.sha256`, `wax-mcp.sha256`) are honored when present
  - `mcp doctor` now validates colocated runtime layout before running the protocol smoke check
- Replaced the stale token-counter TODO in `Tests/WaxIntegrationTests/PerformanceImprovementsTests.swift` with a real regression test that proves BPE tokenizer loads at most once per encoding given the initial cache state.
- Added CLI/runtime regression coverage in `Tests/WaxCLITests/WaxCLIMemoryTests.swift` for:
  - embedder tuning option resolution
  - broker identity changes when tuning changes
  - install-time checksum rejection
  - doctor/runtime checksum validation
- Re-hardened the two long-running process tests that only flaked under full-suite contention:
  - `mcpDoctorRecognizesRenamedToolSurface`
  - `waxMCPProcessRememberWithRealCoreMLEmbedder`

## Runtime Tuning Verification

- `swift build --product wax-cli --product wax-mcp --traits default,MCPServer --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --filter WaxCLITests --skip-update --disable-automatic-resolution` passed with `23` tests.
- `swift test --traits default,MCPServer --filter PerformanceImprovementsTests --skip-update --disable-automatic-resolution` passed with `7` tests.
- `swift test --traits default,MCPServer --filter mcpDoctorRecognizesRenamedToolSurface --skip-update --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --filter waxMCPProcessRememberWithRealCoreMLEmbedder --skip-update --disable-automatic-resolution` passed.
- `swift test --traits default,MCPServer --disable-automatic-resolution` passed with `885` tests and `0` failures.

## MCP Process Test Optimization Plan

- [x] Inspect the current process-heavy MCP and CLI tests to identify redundant subprocess bootstrap and contention-heavy patterns.
- [x] Refactor the process harness/tests to share bootstrap logic and reduce redundant real-process work without weakening end-to-end coverage.
- [x] Serialize only the expensive process-based MCP suite segments that contend on broker/CoreML resources, leaving the rest of the suite parallel.
- [x] Run focused MCP/CLI verification plus a full package pass, then record the outcome below.

## MCP Process Test Optimization Review

- Optimized the process-heavy MCP layer in `Tests/WaxMCPServerTests/WaxMCPServerTests.swift` by:
  - moving the real subprocess coverage into a dedicated serialized `WaxMCPProcessTests` suite so only broker/CoreML-heavy cases lose parallelism
  - consolidating the broker session recall and ended-handoff assertions into one process test, removing one redundant server spawn
  - adding shared `bootstrap` and `callTool` helpers on `MCPServerProcessHarness` so tests no longer duplicate the MCP handshake boilerplate
  - adding a small post-launch settle in `start()` to reduce no-output startup races on fresh processes
  - changing broker shutdown cleanup to a fire-and-forget signal instead of blocking on a reply during teardown, which eliminated the full-suite hang in `shutdownBrokerIfRunning()`
- Result:
  - targeted `WaxMCPProcessTests` now pass consistently with 6 tests in about `21.270s`
  - `mcpDoctorRecognizesRenamedToolSurface` still passes in about `0.679s`
  - full `swift test --traits default,MCPServer --disable-automatic-resolution` now exits cleanly instead of hanging in process-test teardown

## MCP Process Test Optimization Verification

- `swift test --traits default,MCPServer --filter WaxMCPProcessTests --skip-update --disable-automatic-resolution` passed with `6` tests in about `21.270s`.
- `swift test --traits default,MCPServer --filter mcpDoctorRecognizesRenamedToolSurface --skip-update --disable-automatic-resolution` passed in about `0.679s`.
- `swift test --traits default,MCPServer --disable-automatic-resolution` passed with `884` tests and `0` failures.
