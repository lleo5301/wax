# Wax Production-Readiness Audit Report

**Date:** 2026-03-07
**Auditor Role:** Principal Engineer
**Codebase:** Wax — On-device RAG framework (Swift 6.1, ~53K LOC)
**Scope:** Full repository audit: core engine, search modules, orchestrators, MCP server, CLI, tests

---

## 1. Executive Summary

### Overall Production Readiness Score: **7.0 / 10**

The Wax codebase demonstrates strong engineering discipline in several areas — the binary file format has crash-injection testing, SQL queries use parameterized arguments, the concurrency model leverages Swift actors correctly (the `AsyncReadWriteLock` and `MetalVectorEngine` actor implementations were verified sound), and the WAL implementation is carefully designed with fault injection support. No correctness blockers were confirmed after verification. However, several structural issues — fragile `@unchecked Sendable` conformances, fire-and-forget CLI cleanup, incomplete license enforcement, and test coverage gaps — must be addressed before this framework is production-grade for third-party consumers.

### Top 5 Critical Risks

| # | Risk | Severity | Location |
|---|------|----------|----------|
| 1 | ObjC-runtime ivar access to USearch native handle is inherently fragile — breaks silently on library update | **Major** | `WaxVectorSearch/USearchSendable.swift:92-108` |
| 2 | `FDFile` is `@unchecked Sendable` with mutable state and no internal locking; TOCTOU race on `isClosed` | **Major** | `WaxCore/IO/FDFile.swift:456` |
| 3 | `defer { Task { try? await memory.close() } }` in CLI commands — fire-and-forget close may never complete before exit | **Major** | All `Sources/WaxCLI/*.swift` files |
| 4 | License validation is client-side format-only; `pingActivation` is a no-op placeholder | **Major** | `WaxMCPServer/LicenseValidator.swift:148-151` |
| 5 | Crash injection via `WAX_CRASH_INJECT_CHECKPOINT` env var is not gated behind `#if DEBUG` — reachable in release builds | **Major** | `WaxCore/Wax.swift:2267-2276` |

### Release Blockers

1. **Gate crash injection behind `#if DEBUG`** — the `WAX_CRASH_INJECT_CHECKPOINT` path should never be reachable in a release build. An attacker with env var control could cause data loss by triggering `SIGKILL` mid-commit.

> **Note:** Two previously reported blockers were retracted after verification:
> - ~~`AsyncReadWriteLock` reader count bug~~ — **False positive.** `writeUnlock()` correctly increments `readers` before resuming continuations, and actor isolation serializes all state mutations.
> - ~~`WALRingWriter` `@unchecked Sendable` without locks~~ — **Downgraded to Major.** While it has no internal locking, all access is serialized through the `Wax` actor's `io.run {}` closures. The concern is fragility, not a current bug.

---

## 2. Correctness Issues

### 2.1 ~~AsyncReadWriteLock Reader Admission Bug~~ [Verified Safe]

**File:** `Sources/WaxCore/Concurrency/ReadWriteLock.swift:87-94`

```swift
public func readLock() async {
    if writers > 0 || !writerWaiters.isEmpty {
        await withCheckedContinuation { continuation in
            readerWaiters.append(continuation)
        }
    } else {
        readers += 1  // Only incremented on the non-suspended path
    }
}
```

~~When a reader suspends (enters `readerWaiters`), `readers` is NOT incremented.~~ **False positive:** `writeUnlock()` (lines 125-129) increments `readers` for each waiting reader **before** resuming the continuation. By the time `readLock()` returns from the `await`, the reader is already counted. The implementation is correct.

**Original concern (retracted):** The initial audit hypothesized a race between reader resumption and `readUnlock()` calls, but since `AsyncReadWriteLock` is an actor, all state mutations (including `writeUnlock()` incrementing `readers` and resuming continuations) are serialized. A concurrent `readUnlock()` cannot execute until `writeUnlock()` yields, at which point all resumed readers are already counted.

### 2.2 `BinaryDecoder` Force Casts [Minor]

**File:** `Sources/WaxCore/BinaryCodec/BinaryDecoder.swift:139-144`

```swift
if type == UInt8.self { return try decode(UInt8.self) as! T }
```

These `as!` casts are technically safe because the `type == UInt8.self` check guarantees the cast will succeed. However, this bypasses the compiler's type-safety guarantees and would crash the process if the type system changes. **Acceptable but fragile.**

### 2.3 `readEmbeddings` Redundant Guard [Minor]

**File:** `Sources/Wax/Orchestrator/MemoryOrchestrator.swift:980`

```swift
let dimension = try Int(readUInt32())
guard dimension >= 0 else { ... }  // UInt32 cast to Int is always >= 0
```

This guard is a tautology for 64-bit platforms but would be meaningful on hypothetical 32-bit platforms where `UInt32.max > Int.max`. Low risk.

### 2.4 FTS5 `docCount` Tracking with Wrapping Arithmetic [Minor]

**File:** `Sources/WaxTextSearch/FTS5SearchEngine.swift:600-606`

```swift
docCount &+= UInt64(addedCount)  // wrapping add
docCount = docCount > removedU ? (docCount &- removedU) : 0  // wrapping subtract with floor
```

Using wrapping arithmetic (`&+`) means `docCount` can silently overflow to 0 on a massive corpus. This is extremely unlikely but the inconsistency between the add (wrapping) and subtract (floor at 0) suggests defensive coding rather than intentional wrap semantics.

### 2.5 NativeBpeTokenizer `preconditionFailure` in Initializer [Major]

**File:** `Sources/Wax/RAG/NativeBpeTokenizer.swift:14`

```swift
preconditionFailure("Invalid cl100k_base regex: \(error)")
```

If the regex compilation fails (e.g., on a platform where ICU support differs), this crashes the process rather than propagating an error. Since `TokenCounter` is used throughout RAG context building, this could crash an app on an unusual platform configuration.

### 2.6 CoreML Generated Model Force Unwraps [Major]

**File:** `Sources/WaxVectorSearchMiniLM/CoreML/all-MiniLM-L6-v2.swift:54, 96`

```swift
// Line 54 — output property
public var var_554: MLMultiArray {
    provider.featureValue(for: "var_554")!.multiArrayValue!
}

// Line 96 — model resource URL
return bundle.url(forResource: "all-MiniLM-L6-v2", withExtension: "mlmodelc")!
```

Two force unwraps in the CoreML-generated model wrapper. The resource URL force unwrap on line 96 is particularly dangerous — if the `.mlmodelc` bundle is missing or corrupted, the entire app crashes during model initialization with no opportunity for error recovery. This is on a critical initialization path used by `MiniLMEmbeddings`.

### 2.7 BertTokenizer Manual Lock/Unlock Pattern [Minor]

**File:** `Sources/WaxVectorSearchMiniLM/CoreML/BertTokenizer.swift:384-413`

The vocab cache loading uses manual `lock()`/`unlock()` calls with multiple exit points rather than the safer `withLock {}` pattern. If an exception occurs between lock acquisition and release, the lock leaks and subsequent callers deadlock. The `MiniLMEmbeddings.ModelCache` has a similar pattern (lines 213-237) but partially mitigates with `defer`.

---

## 3. Architecture & Design Gaps

### 3.1 `@unchecked Sendable` Overuse [Major]

The codebase uses `@unchecked Sendable` on **19 types** across the source tree:

- `ReadWriteLock`, `UnfairLock` — justified (internally synchronized)
- `FDFile` — **unjustified** (mutable `isClosed` and `faultInjectionState` with no locking)
- `MappedWritableRegion` — **unjustified** (mutable `isClosed` with no locking)
- `WALRingWriter` — **fragile but currently safe** (10 mutable properties, no internal locking, but exclusively accessed through `Wax` actor's `io.run {}` — see Section 4.1)
- `BertTokenizer`, `BasicTokenizer`, `WordpieceTokenizer` — **justified** (immutable after init)
- `BatchInputBuffers` — questionable (MLMultiArray pointers)
- `BlockingIOExecutor` — justified (DispatchQueue is inherently thread-safe)
- `FileLock` — mutable `isReleased` and `mode` without locking
- `NativeBpeTokenizer` — has internal `LockedCache`, but the tokenizer's regex/tables are immutable after init, so justified
- `MiniLMEmbeddings.ModelCache` — has internal `NSLock`, justified
- `DatabaseQueue` — GRDB's own synchronization, justified

**Verdict:** At least 3 `@unchecked Sendable` conformances (`FDFile`, `MappedWritableRegion`, `FileLock`) are unjustified without internal locking. `WALRingWriter` is safe in practice due to actor serialization but the conformance is fragile and undocumented.

### 3.2 `Wax` Actor is a God Object (~2,300 lines) [Major]

**File:** `Sources/WaxCore/Wax.swift`

The `Wax` actor manages: file I/O, WAL, header pages, TOC, footer, frame storage, embedding indices, checkpointing, compaction, crash injection, writer leases, session management, and verification. This violates separation of concerns.

**Recommendation:** Extract `WALManager`, `HeaderManager`, `CommitCoordinator`, and `FrameStore` as separate internally-isolated types.

### 3.3 Mixed Concurrency Primitives [Minor]

The codebase simultaneously uses:
- Swift actors (`Wax`, `FTS5SearchEngine`, `MemoryOrchestrator`, etc.)
- `AsyncReadWriteLock` (actor-based)
- `ReadWriteLock` / `UnfairLock` (pthread-based)
- `NSLock`
- `DispatchQueue` with barriers (`BlockingIOExecutor`)

While each is appropriate for its use case, the mixture creates cognitive overhead and increases the surface area for deadlock bugs, especially since `BlockingIOExecutor.run()` bridges sync GCD work back to async continuations.

### 3.4 `nonisolated(unsafe)` Static Mutable State [Major]

**File:** `Sources/WaxMCPServer/LicenseValidator.swift:28-30`

```swift
nonisolated(unsafe) private static var _trialDefaults: UserDefaults = .standard
nonisolated(unsafe) private static var _firstLaunchKey = "wax_first_launch"
nonisolated(unsafe) private static var _keychainEnabled = true
```

These are protected by an `NSLock` through computed property accessors, which is correct, but the `nonisolated(unsafe)` annotation tells the compiler to skip data-race checking. If anyone accidentally accesses the underscored properties directly in future code changes, this is a latent data race. Would be safer as a locked struct or actor.

### 3.5 MCP Server Version Hardcoded [Minor]

**File:** `Sources/WaxMCPServer/main.swift:94`

```swift
let serverVersion = "0.1.12"
```

This is manually synced with `npm/waxmcp/package.json`. A drift here creates confusion for clients.

---

## 4. Concurrency & Safety

### 4.1 WALRingWriter Thread Safety [Major — Fragile but Currently Safe]

**File:** `Sources/WaxCore/WAL/WALRingWriter.swift`

`WALRingWriter` is a `final class` marked `@unchecked Sendable` with 10 mutable stored properties and zero internal synchronization. It relies entirely on the caller (`Wax` actor) for thread safety.

1. The `Wax` actor dispatches I/O through `BlockingIOExecutor.run {}`, which executes on a GCD queue.
2. `WALRingWriter` methods like `append()` mutate `writePos`, `pendingBytes`, `lastSequence`, etc. and perform I/O.
3. `WALRingWriter` is captured in closures passed to `io.run {}`. Swift's `Sendable` checking is satisfied (because of `@unchecked Sendable`), but the actual access happens on the GCD thread, not the actor's executor.

**Verification:** All usage sites were audited. `WALRingWriter` is created inside `io.run {}` closures, stored exclusively in the `Wax` actor, and accessed only through `io.run {}` closures that are serialized by actor isolation. **No current data race exists.** However, the `@unchecked Sendable` conformance is misleading — it tells the type system this is safe to share freely, but actual safety depends on implicit invariants that a future maintainer could violate.

**Recommendation:** Either remove `@unchecked Sendable` (if it doesn't need to cross isolation boundaries) or add a comment documenting the actor-serialization invariant. Adding internal `OSAllocatedUnfairLock` would make the safety explicit at a small performance cost.

### 4.2 FDFile Thread Safety [Major]

**File:** `Sources/WaxCore/IO/FDFile.swift`

`FDFile` is `@unchecked Sendable` with mutable `isClosed` (Bool) and `faultInjectionState` (optional reference). Same analysis as WALRingWriter — relies on callers for synchronization.

Additionally, `isClosed` is checked in `ensureOpen()` and set in `close()` without any memory barrier. If the same `FDFile` is accidentally accessed from two threads (feasible given its `Sendable` conformance), this is a TOCTOU race.

### 4.3 `defer { Task { try? await memory.close() } }` Anti-pattern [Major]

**Files:** All `Sources/WaxCLI/*.swift` commands

```swift
defer { Task { try? await memory.close() } }
```

This creates an unstructured `Task` in a `defer` block. When the enclosing function returns, the task is launched but not awaited. The CLI command's `run()` method then returns and the process calls `exit()`, potentially before `close()` completes. This can lead to:
- Uncommitted WAL data loss
- Corrupt file state if `close()` was mid-write
- File lock not released (though OS releases on process exit)

**Fix:** Use `try await memory.close()` in a proper cleanup path, or use `withTaskGroup` to ensure the close completes.

### 4.4 Unstructured Task in Wax Actor [Minor]

**File:** `Sources/WaxCore/Wax.swift:349`

```swift
Task { [waiterId] in
    await self.timeoutWriterWaiter(id: waiterId, duration: timeout)
}
```

This unstructured `Task` captures `self` strongly. If the `Wax` actor is closed/deinit'd before the timeout fires, the task keeps the actor alive. Not a safety issue per se, but creates a potential resource leak if many writer lease timeouts are pending.

### 4.5 VideoRAGOrchestrator `@MainActor` Dispatch [Minor]

**File:** `Sources/Wax/VideoRAG/VideoRAGOrchestrator.swift:926`

```swift
Task { @MainActor in ... }
```

Dispatching to `@MainActor` from within an actor (VideoRAGOrchestrator) in a library context is concerning — the library consumer may not be running on a main-thread-equipped environment (e.g., server-side Swift). This appears to be for Photos framework access which requires main thread on Apple platforms.

---

## 5. Performance Bottlenecks

### 5.1 Embedding Staging via Temp Files [Minor]

**File:** `Sources/Wax/Orchestrator/MemoryOrchestrator.swift:280-340`

During ingestion, embeddings are serialized to temporary files on disk, then read back. For small batches, the disk I/O overhead may exceed the cost of keeping them in memory. This is likely intentional for large ingestion jobs to avoid memory pressure, but there's no threshold to skip temp files for small batches.

### 5.2 FTS5 Serialization via VACUUM + Raw sqlite3 I/O [Minor]

**File:** `Sources/WaxTextSearch/FTS5SearchEngine.swift:486-501`

Each serialization of the FTS5 engine does a `VACUUM` if there are freelist pages, which rewrites the entire SQLite database. This is O(n) in database size and blocks all reads during the operation.

### 5.3 USearch Linux Fallback Uses Temp Files [Minor]

**File:** `Sources/WaxVectorSearch/USearchSendable.swift:116-138`

On Linux (where ObjC runtime is unavailable), USearch serialization falls back to writing to a temporary file and reading it back. This adds significant latency for every serialize/deserialize cycle compared to the in-memory buffer path on macOS.

### 5.4 `pendingKeys` Array for Ordered Flush [Minor]

**File:** `Sources/WaxTextSearch/FTS5SearchEngine.swift:19, 530-536`

The FTS5 engine maintains both a `pendingOps: [Int64: PendingOp]` dictionary and a `pendingKeys: [Int64]` array to preserve insertion order. This doubles memory for pending operations. An `OrderedDictionary` would be more efficient.

### 5.5 Per-Element Float Deserialization in readEmbeddings [Minor]

**File:** `Sources/Wax/Orchestrator/MemoryOrchestrator.swift:978-998`

Embeddings are deserialized one float at a time with individual `withUnsafeMutableBytes` calls. A single `memcpy` per vector (after endian check) would be significantly faster for large embedding batches.

---

## 6. Security Risks

### 6.1 License Validation is Format-Only [Major]

**File:** `Sources/WaxMCPServer/LicenseValidator.swift`

License validation only checks that the key matches the pattern `^[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}$`. The `pingActivation` function is a no-op. Any string matching this regex (e.g., `AAAA-AAAA-AAAA-AAAA`) will pass validation. The trial period is also trivially bypassable by clearing UserDefaults.

**Impact:** No meaningful license enforcement. This is explicitly documented in comments but is a commercial risk.

### 6.2 SQL Injection — Properly Mitigated [None]

All SQL queries in `FTS5SearchEngine` use parameterized arguments (`?` placeholders with GRDB's `arguments:` parameter). The dynamic SQL construction in `facts()` and `evidenceFrameIds()` builds WHERE clauses from hardcoded column names, not user input. **No injection risk found.**

### 6.3 ObjC Runtime Ivar Access [Major]

**File:** `Sources/WaxVectorSearch/USearchSendable.swift:92-108`

```swift
guard let ivar = class_getInstanceVariable(type(of: self), "nativeIndex") else { ... }
let ptr = Unmanaged.passUnretained(self).toOpaque()
let offset = ivar_getOffset(ivar)
let ivarPtr = ptr.advanced(by: offset)
let handle = ivarPtr.assumingMemoryBound(to: UnsafeMutableRawPointer?.self).pointee
```

This accesses a private ivar of `USearchIndex` via ObjC runtime introspection. This is:
- **Fragile:** A USearch library update that renames/removes the ivar silently breaks this code.
- **Unsound:** `assumingMemoryBound` has strict preconditions about actual memory layout.
- **Non-portable:** Only works on platforms with ObjC runtime (macOS/iOS), hence the Linux fallback.

If USearch changes the ivar name, this will throw `failedToGetHandle` instead of silently corrupting. But the approach is inherently brittle for production use.

### 6.4 MCP Tool Input Validation [Good]

**File:** `Sources/WaxMCPServer/WaxMCPTools.swift`

Input validation is present and thorough:
- `maxContentBytes = 128 * 1024` limit on content
- `maxTopK = 200`, `maxRecallLimit = 100`, `maxGraphLimit = 500`
- Allowlisted character sets for identifiers
- String length limits on identifiers

### 6.5 Temp File Creation Uses UUID Names [Good]

Temp files use `UUID().uuidString` in their names, making path prediction and symlink attacks infeasible.

### 6.6 FTS5Serializer Uninitialized Memory on Nil baseAddress [Minor]

**File:** `Sources/WaxTextSearch/FTS5Serializer.swift:37-41`

```swift
data.withUnsafeBytes { raw in
    if let base = raw.baseAddress {
        memcpy(buffer, base, size)
    }
}
```

If `baseAddress` is nil (theoretically impossible for non-empty Data, but the code uses `if let` instead of `guard let ... else { throw }`), the `sqlite3_deserialize` call on line 43 receives uninitialized `sqlite3_malloc64` memory. This should use `guard let` with a thrown error for defensive correctness.

### 6.7 CLI Credential Leakage via Process Arguments [Minor]

**File:** `Sources/WaxCLI/WaxCLICommand.swift`

The `install` command passes the license key as a Claude MCP environment argument:
```swift
addArguments.append(contentsOf: ["-e", "WAX_LICENSE_KEY=\(key)"])
```

This makes the license key visible in process listings (`ps aux`). Additionally, the `serve` command inherits the full parent environment (`ProcessInfo.processInfo.environment`), which may include AWS credentials, GitHub tokens, and other secrets. A scoped, minimal environment should be constructed instead.

### 6.8 No Path Traversal Validation on Store Path [Minor]

**File:** `Sources/WaxCLI/WaxCLICommand.swift`

The `--store-path` CLI option is normalized via `standardizedFileURL` but has no validation against `..` traversal. A user-supplied path like `~/.wax/../../../etc/shadow` would be normalized and potentially accessed. The practical risk is low since the CLI runs as the current user, but a library consumer embedding Wax with user-supplied paths should validate.

### 6.9 Crash Injection via Environment Variable [Minor]

**File:** `Sources/WaxCore/Wax.swift:2267-2275`

The `WAX_CRASH_INJECT_CHECKPOINT` environment variable, if set in a production environment, would cause the process to `SIGKILL` itself during a commit. This should be gated behind `#if DEBUG` or a compile-time flag.

**Impact:** An attacker with environment variable control could cause data loss by triggering crash injection during a commit cycle. Low probability but non-zero in containerized environments.

---

## 7. Testing Review

### 7.1 Test Coverage — Generally Strong

The test suite contains ~80 test files covering:
- WAL compaction benchmarks
- Concurrency stress tests
- Determinism property tests
- FTS5 serialization
- Vector search correctness
- Photo/Video RAG orchestration
- Structured memory CRUD
- Production readiness stability tests
- Coverage gap tests (meta-tests)

### 7.2 Missing Coverage [Major]

| Area | Gap |
|------|-----|
| `AsyncReadWriteLock` | No tests for concurrent reader/writer interleaving — while Section 2.1 was verified correct, stress tests would catch future regressions |
| `FDFile` concurrent access | No tests verify that `@unchecked Sendable` claim is safe |
| `WALRingWriter` partial-write recovery | Fault injection exists but no test exercises concurrent writers (N/A given actor isolation, but the `@unchecked Sendable` conformance implies it should be tested) |
| `LicenseValidator` trial clock manipulation | No test for system clock rollback/advance attacks on trial period |
| `USearchSendable` ivar name stability | No test verifies the `"nativeIndex"` ivar name exists on the current USearch version |
| CLI `defer { Task { } }` close behavior | No test verifies data is flushed before process exit |
| MCP server shutdown | No test for graceful shutdown completing pending operations |
| Task cancellation | No tests verify correct behavior when `Task.cancel()` is called mid-operation (e.g., during embedding, search, or WAL write) |
| Disk-full / ENOSPC handling | No tests simulate disk-full conditions; `FDFile.write` returns `ENOSPC` but no test verifies the error propagates correctly or that partial writes are handled |
| Operations on closed orchestrator | No tests verify that calling methods after `close()` returns appropriate errors rather than crashing or corrupting state |
| High-concurrency crash scenarios | Existing concurrency tests use modest parallelism; no test exercises 1000+ concurrent operations to surface rare race conditions |
| Maintenance/compaction under load | WAL compaction and index maintenance are not tested while concurrent reads/writes are in flight |

### 7.3 Test Anti-patterns [Minor]

- **Trivial "always pass" tests:** `DependencyTests.swift` contains tests that only verify a dependency can be imported — these always pass at compile time and provide zero runtime coverage. They should either test actual functionality or be removed.
- **Weak search quality assertions:** Several FTS5 and vector search tests assert only that results are non-empty (`#expect(!results.isEmpty)`) without validating ranking order, score ranges, or result relevance. A broken ranking algorithm would pass these tests.
- Several benchmark tests (`RAGBenchmarks.swift`, `BatchEmbeddingBenchmark.swift`) are in the integration test target and will run with regular `swift test`, potentially slowing CI.
- **Force-unwrap in CoreML model init:** `all-MiniLM-L6-v2.swift` uses force-unwraps (`!`) on model output properties and resource URLs. Test coverage does not exercise the failure path because tests run on machines with the model available.
- Mock providers in `Tests/WaxIntegrationTests/Mocks/` are comprehensive and realistic.
- The use of Swift Testing framework (`@Test`, `#expect`) is modern and appropriate.

### 7.4 Property Testing Opportunities [Minor]

- **Binary codec round-trip:** Fuzz `BinaryEncoder` output through `BinaryDecoder` to verify bijection.
- **WAL ring buffer wrap-around:** Property test that any sequence of append/checkpoint operations maintains invariant `pendingBytes == actual_pending_data_on_disk`.
- **FTS5 search ordering:** Property test that BM25 scores are monotonically related to term frequency.

---

## 8. Refactoring Opportunities

### 8.1 Extract WAL Manager from Wax Actor [Major]

The `Wax` actor at ~2,300 lines is too large. The WAL management (ring writer, checkpointing, proactive commit) could be extracted into a separate `WALManager` type that the `Wax` actor owns.

### 8.2 Protocol for Compression Backend [Minor]

`PayloadCompressor` uses `#if canImport(Compression)` / `#elseif os(Linux)` compile-time branching. A protocol (`CompressionBackend`) with platform-specific conformances would improve testability.

### 8.3 Reduce MemoryOrchestrator Init Complexity [Minor]

**File:** `Sources/Wax/Orchestrator/MemoryOrchestrator.swift:136-201`

The init does parallel async work (tokenizer prewarm), config mutation, file operations, and session opening. Consider a builder or factory pattern.

### 8.4 Unify Error Types [Minor]

`WaxError` is a large enum serving as the universal error type. Consider splitting into domain-specific errors: `WaxIOError`, `WaxFormatError`, `WaxCapacityError`.

### 8.5 Remove Dead Code Paths [Minor]

- `MappedWritableRegion.copyBytes(from:)` — only called from test harnesses, not production code (verify before removing)
- `FDFile.installFaultPlan()` / `clearFaultPlan()` — test-only infrastructure in production source

### 8.6 Naming Improvements [Minor]

| Current | Suggested | Reason |
|---------|-----------|--------|
| `ids_to_tokens` | `idsToTokens` | Swift naming convention |
| `stagedLexIndexStampCounter` | `lexIndexStagingGeneration` | Clearer semantics |
| `walAutoCommitCount` | `proactiveCommitCount` | Matches the concept name |

---

## 9. Metal / GPU-Specific Issues

### 9.1 ~~MetalVectorEngine `stageForCommit()` Data Race~~ [Verified Safe]

**File:** `Sources/WaxVectorSearch/MetalVectorEngine.swift`

~~The `dirty` flag is read without holding the write lock in `stageForCommit()`.~~

**False positive:** `MetalVectorEngine` is declared as an `actor`. All access to the `dirty` property is serialized by actor isolation. Two concurrent calls to `stageForCommit()` cannot race because they are enqueued on the actor's serial executor.

### 9.2 SIMD Alignment Assumptions in Metal Shaders [Major]

**File:** `Sources/WaxVectorSearch/Shaders/CosineDistance.metal`

The SIMD4/SIMD8 cosine distance kernels cast raw `float*` pointers to `float4*`/`float8*` without verifying 16/32-byte alignment. If the underlying `MTLBuffer` is not properly aligned (e.g., odd-dimension vectors at non-aligned offsets), this causes GPU crashes or silent data corruption.

### 9.3 NaN Handling in TopKReduction Shader [Minor]

**File:** `Sources/WaxVectorSearch/Shaders/TopKReduction.metal`

`INFINITY` sentinel comparisons don't handle NaN distances. If a zero-magnitude query vector produces NaN distances, corrupted results are returned because `NaN < INFINITY` evaluates to false.

### 9.4 No Threadgroup Memory Size Validation [Minor]

**File:** `Sources/WaxVectorSearch/MetalVectorEngine.swift`

`setThreadgroupMemoryLength(dimensions * 4, ...)` has no check against the GPU's threadgroup memory limit (~48KB on Apple GPUs). Vectors with > 12,288 dimensions would cause a silent kernel launch failure.

### 9.5 VectorSerializer Integer Overflow [Minor]

**File:** `Sources/WaxVectorSearch/VectorSerializer.swift:117`

```swift
let expectedVectorBytes = Int(header.vectorCount) * Int(header.dimension) * MemoryLayout<Float>.stride
```

Three unchecked multiplications on deserialized values can overflow on crafted input, leading to incorrect buffer size validation.

---

## Appendix: Dependency Risk Assessment

| Dependency | Version | Risk | Notes |
|------------|---------|------|-------|
| USearch | >= 2.24.0 | **Medium** | ObjC runtime ivar access is fragile |
| GRDB.swift | >= 6.24.0 | Low | Mature, well-maintained |
| swift-crypto | >= 3.7.0 | Low | Apple-maintained |
| swift-log | >= 1.5.0 | Low | Apple-maintained |
| swift-sdk (MCP) | >= 0.10.0 | **Medium** | Protocol still evolving |
| SwiftTUI | branch: main | **High** | Pinned to `main` branch, no semver — breaking changes possible at any time |
| Noora | >= 0.54.0 | Low | Tuist-maintained |
| swift-argument-parser | >= 1.3.0 | Low | Apple-maintained |

**Critical:** `SwiftTUI` is pinned to `branch: main`. This means `swift package resolve` will pull whatever HEAD is at resolve time. A breaking change upstream will silently break builds.

---

## Summary of Required Actions

### Blockers (Must fix before any production release)

1. **Gate crash injection behind `#if DEBUG`** — the `WAX_CRASH_INJECT_CHECKPOINT` path should never be reachable in a release build.

> ~~Fix `AsyncReadWriteLock` readLock counting~~ — **Retracted.** Verified correct; `writeUnlock()` increments `readers` before resuming continuations.
> ~~Fix `MetalVectorEngine.stageForCommit()` data race~~ — **Retracted.** `MetalVectorEngine` is an actor; `dirty` is protected by actor isolation.

### Major (Should fix before production)

2. Fix `defer { Task { } }` close pattern in CLI commands.
3. Replace ObjC runtime ivar access in USearchSendable with a proper USearch API (or pin to specific USearch version).
4. Resolve `@unchecked Sendable` on `FDFile`, `MappedWritableRegion`, and `FileLock` — either add internal locking or remove the conformance. Document `WALRingWriter`'s actor-serialization invariant.
5. Implement actual license validation backend or remove the gate.
6. Pin `SwiftTUI` dependency to a tagged release.
7. Guard `NativeBpeTokenizer` regex failure with a throwing init instead of `preconditionFailure`.
8. Validate SIMD alignment assumptions in Metal shaders or add runtime dimension checks.
9. Add stress tests for `AsyncReadWriteLock` (verified correct, but no regression tests exist).

### Minor (Improve for robustness)

10. Decompose `Wax` actor into smaller components.
11. Move fault injection infrastructure behind `#if DEBUG`.
12. Add USearch ivar name verification test.
13. Separate benchmarks from integration tests in CI.
14. Use consistent error domain types.
15. Fix FTS5Serializer `if let` to `guard let` for `baseAddress` nil safety.
16. Remove or replace trivial "always pass" dependency tests with meaningful functionality tests.
17. Strengthen search quality assertions to validate ranking correctness, not just non-empty results.
18. Add task cancellation and disk-full error path tests.
19. Add tests for operations on a closed orchestrator.
