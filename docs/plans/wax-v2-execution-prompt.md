# Wax v2: End-to-End Execution Prompt

> Paste this prompt into a fresh Claude Code session to execute the full Wax v2 improvement plan.

---

## Context

You are implementing the Wax v2 improvement plan located at `docs/plans/2026-03-03-wax-memvid-improvements.md`. Read that file first — it is your authoritative spec. This prompt provides execution instructions.

**Project:** Wax — an on-device memory/RAG framework for Apple platforms, written in Swift 6.2.
**Repository:** `/Users/chriskarani/CodingProjects/AIStack/Wax`
**Goal:** Close 7 verified gaps between Wax and memvid by implementing features adapted for Swift 6.2 and Apple platforms.
**Approach:** TDD (red-green-commit), one task at a time, strict wave ordering.
**Branch:** Create a feature branch `feat/wax-v2-improvements` from `main` before starting.

---

## Pre-Flight Checklist

Before writing any code:

1. **Read the plan:** `docs/plans/2026-03-03-wax-memvid-improvements.md` — this is your spec.
2. **Read CLAUDE.md** and follow all workflow/memory/TDD instructions.
3. **Invoke the `wax` skill** — it contains Wax-specific guidance for MemoryOrchestrator, WaxSession, embeddings, and file format conventions.
4. **Create feature branch:**
   ```bash
   git checkout -b feat/wax-v2-improvements main
   ```
5. **Explore the codebase** to validate assumptions in the plan before coding:
   - Read `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` — understand `remember()` and `recall()` signatures
   - Read `Sources/Wax/WaxSession.swift` — understand frame metadata APIs
   - Read `Sources/WaxCore/FileFormat/WaxTOC.swift` — understand TOC extension tag mechanism
   - Read `Sources/WaxCore/Wax.swift` — understand file create/open lifecycle
   - Read `Sources/WaxTextSearch/StructuredMemorySchema.swift` — understand `sm_fact` schema
   - Read `Sources/WaxTextSearch/FTS5Schema.swift` — understand schema versioning/migration
   - Read `Tests/WaxIntegrationTests/RAGBenchmarks.swift` — understand existing benchmark harness
   - Read `Sources/WaxCore/StructuredMemory/StructuredMemoryHashing.swift` — understand existing hash infra
   - Read `Sources/Wax/RAG/FastRAGContextBuilder.swift` — understand `SearchRequest` and `timeRange`
6. **Validate plan assumptions** against what you read. If any API, type, or file path in the plan doesn't match reality, adapt the implementation to fit the actual codebase — do NOT blindly follow pseudocode that references nonexistent APIs.
7. **Set up task tracking** in `tasks/todo.md` with all 7 tasks and their wave groupings.

---

## Execution Order (STRICT — do not reorder)

```
Wave 1 (can be parallel): Task 1 (Benchmarks) + Task 7 (Data Protection)
Wave 2 (sequential):      Task 2 (Content Dedup) → Task 3 (Model Binding)
Wave 3 (sequential):      Task 4 (Version Relations) → Task 5 (Temporal NLP)
Wave 4 (sequential):      Task 6 (Enrichment Pipeline)
```

Tasks 2–6 all modify `MemoryOrchestrator.swift`. Execute them sequentially to avoid merge conflicts. Commit after each task.

---

## Per-Task Protocol

For EVERY task, follow this exact cycle:

### 1. RED — Write failing tests first
- Create test files as specified in the plan
- Run `swift test --filter <TestName>` to confirm compilation failure or test failure
- Do NOT proceed to implementation until you have a failing test

### 2. GREEN — Implement the minimum code to pass
- Create/modify source files as specified in the plan
- Adapt pseudocode to match actual Wax APIs (the plan contains approximations)
- Run `swift test --filter <TestName>` to confirm tests pass

### 3. VERIFY — Run broader test suite
- Run `swift test` (full suite) to confirm no regressions
- If any existing test breaks, fix it before committing

### 4. COMMIT — Atomic commit per task
- Stage only the files for this task
- Use the commit message from the plan
- Format: `git commit -m "feat: <description>"`

### 5. UPDATE — Mark task complete in `tasks/todo.md`

---

## Task-Specific Instructions

### Task 1: Benchmark Suite
- Extend `Tests/WaxIntegrationTests/RAGBenchmarks.swift` — do NOT create a new test target
- Use deterministic text generation and deterministic embedders already in benchmark support
- Do NOT use `Float.random` in benchmark inputs
- Add benchmarks for: dedup throughput, temporal parsing, enrichment pipeline drain time
- Gate on relative regressions, not absolute thresholds
- Verify: `swift test --filter RAGPerformanceBenchmarks`

### Task 2: Frame Content Dedup
- Create `Sources/WaxCore/ContentHasher.swift` using SHA-256 (via existing `SHA256Checksum`)
- Add dedup check at top of `MemoryOrchestrator.remember()`
- Store content hash in frame metadata under key `wax.content.hash`
- Add `findFrameByMetadata(key:value:)` to `WaxSession` — scans `frameMetas()` for matching document frames
- **Known issue:** `findFrameByMetadata` is O(n) — acceptable for v2, document as future optimization
- Verify: `swift test --filter ContentHasherTests && swift test --filter DeduplicationTests`

### Task 3: Store-Level Embedding Model Binding
- `MemoryBinding` struct lives in `Sources/WaxCore/FileFormat/` — pure data, no `WaxVectorSearch` imports
- Wire into `WaxTOC` via the reserved `memory_binding` extension tag slot
- **IMPORTANT:** The plan's BinaryCodable example has encode/decode field order mismatch. Use consistent field ordering: provider, model, dimensions, normalized — same order in both encode and decode.
- Compatibility bridge `MemoryBindingCompatibility` lives in `Sources/Wax/Embeddings/` (can import both WaxCore and WaxVectorSearch)
- Validate on MemoryOrchestrator init: if store has binding and embedder has identity, fail-fast on mismatch
- Set binding on first successful embedding ingest via `setMemoryBindingIfMissing()`
- Add backward compatibility tests: new code reads old files where binding tag is absent
- Verify: `swift test --filter MemoryBindingTests && swift test --filter ModelBindingTests`

### Task 4: Version Relations
- Create `VersionRelation` enum in `Sources/WaxCore/StructuredMemory/`
- Add `version_relation INTEGER NOT NULL DEFAULT 0` column to `sm_fact` table
- Bump SQLite `user_version` and add migration path in `FTS5Schema.swift`
- Plumb `relation: VersionRelation = .sets` through: `FTS5SearchEngine` → `StructuredMemorySession` → `WaxSession` → `MemoryOrchestrator`
- When `relation.supersedes == true`, close open spans for same (subject, predicate) before inserting
- Add `--relation` option to CLI `FactsCommand`
- **Add migration test fixtures** — create a test that opens a pre-migration DB and verifies upgrade works
- Verify: `swift test --filter VersionRelationTests`

### Task 5: Temporal NLP Parser
- Create `Sources/Wax/Temporal/TemporalNormalizer.swift` and `TemporalResolution.swift`
- Pure Swift using Foundation.Calendar — no external dependencies
- Support: today, yesterday, tomorrow, last/this/next week, last/this/next month, N days ago, in N days, N weeks ago, in N weeks, last/next <weekday>, quarter (q3 2025)
- Integrate into `MemoryOrchestrator.recall()` via n-gram sliding window temporal phrase extraction
- Thread `timeRange` through: `recall()` → `buildRecallContext()` → `FastRAGContextBuilder.build()` → `SearchRequest`
- **Caution:** The n-gram scanner can false-positive. Test with queries that contain temporal-looking words but aren't temporal references.
- Verify: `swift test --filter TemporalNormalizerTests && swift test --filter TemporalRecallIntegration`

### Task 6: Async Enrichment Pipeline
- Create `Sources/Wax/Enrichment/` directory with: `EnrichmentTask.swift`, `EnrichmentPipeline.swift`, `KeywordExtractor.swift`
- `EnrichmentPipeline` is an `actor` using `AsyncStream` for task queue
- **Swift 6.2 concurrency:** The plan shows `processedCount += 1` inside a `Task` closure. Since `EnrichmentPipeline` is an actor, mutations inside `Task { }` launched from actor context are safe — but verify this compiles under strict concurrency checking.
- Wire into `MemoryOrchestrator`: enqueue after frame persist, drain/stop in `close()`
- `KeywordExtractor` uses TF-based keyword extraction with stopword filtering
- Verify: `swift test --filter KeywordExtractor && swift test --filter EnrichmentPipeline`

### Task 7: iOS Data Protection
- Modify `Sources/WaxCore/Wax.swift` to set `FileProtectionType.complete` after file create/open
- Guard with `#if os(iOS) || os(watchOS) || os(tvOS) || os(visionOS)`
- The macOS-only `swift test` validates compile + runtime basics but does NOT prove locked-device protection
- Add a cross-platform readability test that works on macOS too
- Verify: `swift test --filter DataProtection`

---

## Post-Implementation Validation

After ALL 7 tasks are complete:

1. **Full test suite:** `swift test` — everything must pass, zero failures
2. **Build check:** `swift build` — clean build with no warnings in new code
3. **Package layering:** Verify `WaxCore` has no imports of `WaxVectorSearch` or `Wax`
4. **Backward compatibility:** Confirm new code can read existing v1 `.wax` stores (the migration and binding tests cover this)
5. **Diff review:** `git diff main...HEAD --stat` — review all changed files, ensure no unintended modifications
6. **Commit history:** `git log --oneline main..HEAD` — should show 7 clean atomic commits (one per task), possibly plus the initial benchmark additions

---

## Known Plan Errata (fix during implementation)

1. **Task 3 BinaryCodable:** Encode/decode field order is inconsistent in the plan. Use: provider → model → dimensions → normalized in BOTH encode and decode.
2. **Task 4 migration:** The plan mentions migration but doesn't provide a pre-migration test fixture. Create one: write a test that constructs a DB at the old schema version, then opens it with the new code and verifies the migration ran.
3. **Task 5 extractTemporalRange:** The n-gram window `for i in 0...(words.count - window)` will crash if `words.count < window`. Add a guard.
4. **Task 6 EnrichmentPipeline actor:** Verify that the `Task { for await task in stream { ... processedCount += 1 } }` pattern compiles under Swift 6.2 strict concurrency. The `Task` inherits actor isolation, so this should be fine, but confirm.
5. **API discovery:** The plan references APIs like `runtimeStats().frameCount`, `session.findFrameByMetadata()`, `orchestrator.assertFact()`. Verify these exist or adapt to actual API shapes during codebase exploration.

---

## Summary

This is a 7-task TDD implementation covering: benchmarks, content dedup, model binding, version relations, temporal NLP, async enrichment, and iOS data protection. Execute in wave order, commit atomically per task, run full test suite after each wave. The plan at `docs/plans/2026-03-03-wax-memvid-improvements.md` is your spec — this prompt provides execution guardrails and errata corrections.
