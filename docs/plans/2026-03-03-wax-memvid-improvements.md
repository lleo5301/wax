# Wax v2: Closing the Gaps — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Address Wax's 7 verified weaknesses by adopting proven patterns from memvid's Rust codebase, adapted for Swift 6.2 and Apple platforms.

**Architecture:** Changes are grouped to minimize cross-target impact, but they are **not all independent**. Multiple tasks modify `MemoryOrchestrator.swift`, so sequencing matters. The TOC has reserved extension tags (`memory_binding`, `replay_manifest`, `enrichment_queue`), but current v1 decode rejects non-zero tags; adding data to those slots requires explicit decode/compatibility work in `WaxTOC`. Backward compatibility target is: **new code can read existing v1 stores**.

**Tech Stack:** Swift 6.2, swift-testing, Foundation.Calendar, CryptoKit (SHA-256), Apple Data Protection

**Source of inspiration:** memvid Rust codebase at `memvid-main/` — specific file references included per task.

---

## Verified Weaknesses (from head-to-head analysis)

> Note: numeric score deltas are external-analysis heuristics and are **Unverified** by this repository alone.

| # | Gap | Wax Score | Memvid Score | Delta |
|---|---|:---:|:---:|:---:|
| 1 | Benchmark coverage gaps (especially deterministic gating/baselines for new features) | 30 | 40 | -10 |
| 2 | No frame content dedup in `remember()` | 62 | 82 | -20 |
| 3 | No store-level embedding model binding | 75 | 85 | -10 |
| 4 | Only Retracts, no Sets/Updates/Extends | 78 | 80 | -2 |
| 5 | No temporal NLP ("last Tuesday") | 40 | 82 | -42 |
| 6 | No async enrichment pipeline | 62 | 82 | -20 |
| 7 | No data protection on .wax files | 10 | 78 | -68 |

---

## Task 1: Benchmark Suite

**Why:** Wax already has substantial benchmark coverage in `Tests/WaxIntegrationTests` (`RAGBenchmarks`, MiniLM, WAL compaction, vector/Metal benches). The gap is not "no benchmarks"; the gap is inconsistent deterministic baselines and missing benchmarks for the new behaviors in this plan.

**Memvid reference:**
```rust
// memvid-main/benches/vec_search_benchmark.rs:17-57
fn bench_search_10k(c: &mut Criterion) {
    let vectors = generate_vectors(10_000, 128);
    let query = generate_vectors(1, 128).pop().unwrap();
    let mut builder = VecIndexBuilder::new();
    for (i, vec) in vectors.iter().enumerate() {
        builder.add_document(i as FrameId, vec.clone());
    }
    let artifact = builder.finish().expect("finish hnsw");
    let hnsw_index = VecIndex::decode(&artifact.bytes).expect("decode hnsw");
    let mut group = c.benchmark_group("search_10k");
    group.bench_function("hnsw", |b| {
        b.iter(|| { hnsw_index.search(black_box(&query), black_box(10)); })
    });
    group.finish();
}
```

**Files:**
- Modify: `Tests/WaxIntegrationTests/RAGBenchmarks.swift`
- Modify: `Tests/WaxIntegrationTests/RAGBenchmarkSupport.swift` (only if new deterministic fixtures/helpers are needed)
- Optional: Modify benchmark docs with baseline-capture instructions

**Step 1: Extend existing benchmark harness (do not add a new test target)**

Add benchmark cases to `RAGPerformanceBenchmarks` so the new work is measured in the same runner/setup already used by CI and perf-lab jobs.

**Step 2: Keep fixtures deterministic**

- Use deterministic text generation and deterministic embedders already present in benchmark support.
- Do **not** use `Float.random` in benchmark inputs.
- Warm up once, then measure repeated runs.

**Step 3: Gate on relative regressions, not absolute shared-runner latency**

- Keep hard thresholds only behind explicit perf-lab env flags.
- In CI, compare against checked-in/saved baselines and fail on meaningful deltas (for example, p95/p99 relative increase thresholds).
- Record sample count and environment in benchmark output so regressions are attributable.

**Step 4: Verify**

```bash
swift test --filter RAGPerformanceBenchmarks
```

**Step 5: Commit**

```bash
git add Tests/WaxIntegrationTests/RAGBenchmarks.swift Tests/WaxIntegrationTests/RAGBenchmarkSupport.swift
git commit -m "test: extend existing benchmark harness with deterministic regression coverage"
```

---

## Task 2: Frame Content Dedup

**Why:** Calling `remember("same text")` twice creates duplicate frames. Agent loops commonly re-ingest identical content. Memvid prevents this with BLAKE3 hashing at ingest time.

**Memvid reference:**
```rust
// memvid-main/src/memvid/mutation.rs:3299-3312
if options.dedup {
    if let Some(bytes) = payload {
        let content_hash = hash(bytes);  // BLAKE3
        if let Some(existing_frame) = self.find_frame_by_hash(content_hash.as_bytes()) {
            return Ok(existing_frame.id);  // Skip duplicate
        }
    }
}
```

**Wax already has:** `StructuredMemoryHasher` (SHA-256 for fact dedup, `Sources/WaxCore/StructuredMemory/StructuredMemoryHashing.swift`). We extend this pattern to frames.

**Files:**
- Create: `Sources/WaxCore/ContentHasher.swift`
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` — dedup check in `remember()`
- Modify: `Sources/Wax/WaxSession.swift` — add metadata lookup helper over existing frame metadata APIs
- Create: `Tests/WaxCoreTests/ContentHasherTests.swift`
- Create: `Tests/WaxIntegrationTests/DeduplicationTests.swift`

**Step 1: Write failing test for ContentHasher**

```swift
// Tests/WaxCoreTests/ContentHasherTests.swift
import Testing
@testable import WaxCore

@Test func identicalContentProducesSameHash() throws {
    let content = Data("Hello, world!".utf8)
    let hash1 = ContentHasher.hash(content)
    let hash2 = ContentHasher.hash(content)
    #expect(hash1 == hash2)
    #expect(hash1.count == 32) // SHA-256 = 32 bytes
}

@Test func differentContentProducesDifferentHash() throws {
    let hash1 = ContentHasher.hash(Data("Hello".utf8))
    let hash2 = ContentHasher.hash(Data("World".utf8))
    #expect(hash1 != hash2)
}
```

**Step 2: Run to verify failure**

```bash
swift test --filter ContentHasherTests
```
Expected: compilation error — `ContentHasher` does not exist.

**Step 3: Implement ContentHasher**

```swift
// Sources/WaxCore/ContentHasher.swift
import Foundation

/// Content-addressed hashing for frame deduplication.
/// Uses SHA-256 (already a dependency via swift-crypto) to match
/// Wax's existing checksum infrastructure.
///
/// Inspired by memvid's BLAKE3 dedup (memvid-main/src/memvid/mutation.rs:3299).
public enum ContentHasher {
    public static func hash(_ data: Data) -> Data {
        SHA256Checksum.digest(data)
    }
}
```

**Step 4: Run test to verify it passes**

```bash
swift test --filter ContentHasherTests
```

**Step 5: Write failing integration test for dedup in MemoryOrchestrator**

```swift
// Tests/WaxIntegrationTests/DeduplicationTests.swift
import Foundation
import Testing
import Wax

@Test func rememberIdenticalContentTwiceIsIdempotent() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false

        let orchestrator = try await MemoryOrchestrator(at: url, config: config)
        try await orchestrator.remember("Duplicate content test")
        try await orchestrator.flush()

        let afterFirst = await orchestrator.runtimeStats().frameCount
        try await orchestrator.remember("Duplicate content test")
        try await orchestrator.flush()
        let afterSecond = await orchestrator.runtimeStats().frameCount

        // Remembering identical content should not add new document/chunk frames.
        #expect(afterSecond == afterFirst)
        try await orchestrator.close()
    }
}

@Test func rememberDifferentContentIncreasesFrameCount() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false

        let orchestrator = try await MemoryOrchestrator(at: url, config: config)
        try await orchestrator.remember("First content")
        try await orchestrator.flush()
        let afterFirst = await orchestrator.runtimeStats().frameCount

        try await orchestrator.remember("Second content")
        try await orchestrator.flush()
        let afterSecond = await orchestrator.runtimeStats().frameCount

        #expect(afterSecond > afterFirst)
        try await orchestrator.close()
    }
}
```

**Step 6: Run to verify failure**

```bash
swift test --filter DeduplicationTests
```
Expected: `rememberIdenticalContentTwiceIsIdempotent` FAILS before implementation.

**Step 7: Define metadata key (no binary schema change)**

Do **not** add a binary field to `FrameMeta`. Keep format compatibility by storing the content hash inside existing metadata entries:

```swift
// In FrameMeta or as a metadata key constant
public enum WaxMetadataKeys {
    public static let contentHash = "wax.content.hash"
}
```

**Step 8: Implement dedup check in MemoryOrchestrator.remember()**

In `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`, add dedup logic at the top of `remember()` (line ~233):

```swift
public func remember(_ content: String, metadata: [String: String] = [:]) async throws {
    lastWriteActivityAt = .now

    // --- Content dedup (inspired by memvid mutation.rs:3299) ---
    let contentData = Data(content.utf8)
    let contentHash = ContentHasher.hash(contentData)
    let hashHex = contentHash.map { String(format: "%02x", $0) }.joined()

    if let existingFrameId = await session.findFrameByMetadata(
        key: WaxMetadataKeys.contentHash, value: hashHex
    ) {
        // Identical content already ingested — skip
        return
    }

    // Inject hash into metadata for future dedup lookups
    var docMeta = Metadata(metadata)
    docMeta.entries[WaxMetadataKeys.contentHash] = hashHex

    // ... rest of existing remember() logic, using docMeta ...
```

**Step 9: Implement `findFrameByMetadata` on WaxSession**

Add a lookup method to `WaxSession` that scans existing frame metadata via public Wax APIs:

```swift
public func findFrameByMetadata(key: String, value: String) async -> UInt64? {
    let frames = await wax.frameMetas()
    for frame in frames where frame.status == .active && frame.supersededBy == nil {
        if let meta = frame.metadata,
           meta.entries[key] == value,
           frame.role == .document {
            return frame.id
        }
    }
    return nil
}
```

**Step 10: Run integration tests**

```bash
swift test --filter DeduplicationTests
```
Expected: both PASS.

**Step 11: Commit**

```bash
git add Sources/WaxCore/ContentHasher.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift Sources/Wax/WaxSession.swift \
    Tests/WaxCoreTests/ContentHasherTests.swift Tests/WaxIntegrationTests/DeduplicationTests.swift
git commit -m "feat: add frame content dedup via SHA-256 hash in remember()"
```

---

## Task 3: Store-Level Embedding Model Binding

**Why:** A user can create a `.wax` store with one embedding model and reopen it with another. We need a store-level binding that is validated at open/ingest time.

**False-positive cleanup:** keep package layering clean. `WaxCore` must not depend on `WaxVectorSearch` types.

**Memvid reference:**
```rust
// memvid-main/src/types/embedding_identity.rs:17-23
pub struct EmbeddingIdentity {
    pub provider:   Option<Box<str>>,
    pub model:      Option<Box<str>>,
    pub dimension:  Option<u32>,
    pub normalized: Option<bool>,
}
```

**Files:**
- Create: `Sources/WaxCore/FileFormat/MemoryBinding.swift` (pure data type only)
- Modify: `Sources/WaxCore/FileFormat/WaxTOC.swift` (persist/restore binding in reserved slot)
- Modify: `Sources/WaxCore/Wax.swift` (add binding read/write actor APIs)
- Create: `Sources/Wax/Embeddings/MemoryBindingCompatibility.swift` (maps `EmbeddingIdentity` to `MemoryBinding` and checks compatibility)
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` (validate on init; set binding on first successful embedding ingest)
- Create: `Tests/WaxCoreTests/MemoryBindingTests.swift`
- Create: `Tests/WaxIntegrationTests/ModelBindingTests.swift`

**Step 1: TDD — codec test for `MemoryBinding`**

```swift
// Tests/WaxCoreTests/MemoryBindingTests.swift
import Testing
@testable import WaxCore

@Test func memoryBindingRoundTrips() throws {
    let binding = MemoryBinding(
        embeddingProvider: "local",
        embeddingModel: "all-MiniLM-L6-v2",
        embeddingDimensions: 384,
        embeddingNormalized: true
    )
    var encoder = BinaryEncoder()
    var mutable = binding
    try mutable.encode(to: &encoder)

    var decoder = try BinaryDecoder(data: encoder.data)
    let decoded = try MemoryBinding.decode(from: &decoder)

    #expect(decoded == binding)
}
```

**Step 2: Implement `MemoryBinding` (no `EmbeddingIdentity` reference in WaxCore)**

```swift
// Sources/WaxCore/FileFormat/MemoryBinding.swift
import Foundation

public struct MemoryBinding: Equatable, Sendable {
    public var embeddingProvider: String?
    public var embeddingModel: String?
    public var embeddingDimensions: UInt32?
    public var embeddingNormalized: Bool?
}

extension MemoryBinding: BinaryCodable {
    public mutating func encode(to encoder: inout BinaryEncoder) throws {
        try encoder.encode(embeddingProvider)
        try encoder.encode(embeddingModel)
        encoder.encode(embeddingDimensions)
        let normalizedRaw: UInt8? = embeddingNormalized.map { $0 ? 1 : 0 }
        encoder.encode(normalizedRaw)
    }

    public static func decode(from decoder: inout BinaryDecoder) throws -> MemoryBinding {
        let normalizedRaw = try decoder.decodeOptional(UInt8.self)
        MemoryBinding(
            embeddingProvider: try decoder.decodeOptional(String.self),
            embeddingModel: try decoder.decodeOptional(String.self),
            embeddingDimensions: try decoder.decodeOptional(UInt32.self),
            embeddingNormalized: normalizedRaw.map { $0 != 0 }
        )
    }
}
```

**Step 3: Wire binding into TOC + Wax actor APIs**

- In `WaxTOC`, replace the `memory_binding` placeholder with `MemoryBinding?` encode/decode.
- Update `WaxTOC.decode` to accept optional tag `0/1` for `memory_binding` (and keep strict behavior for unknown tags).
- Add compatibility tests:
  - New code reads old files where `memory_binding` tag is `0`.
  - New code round-trips files where `memory_binding` tag is `1`.
  - Explicitly document that older binaries (without this change) cannot read files written with non-empty binding.
- In `Wax`, add explicit APIs:
  - `public func memoryBinding() async -> MemoryBinding?`
  - `public func setMemoryBindingIfMissing(_ binding: MemoryBinding) async throws`
  - Optional test helper: `public func overwriteMemoryBindingForTesting(_ binding: MemoryBinding?) async throws`

This removes prior false-positive calls to non-existent APIs (`currentTOC()`, `setMemoryBinding()`).

**Step 4: Add compatibility logic in Wax layer**

```swift
// Sources/Wax/Embeddings/MemoryBindingCompatibility.swift
import WaxCore
import WaxVectorSearch

enum MemoryBindingCompatibility {
    static func binding(from identity: EmbeddingIdentity) -> MemoryBinding {
        MemoryBinding(
            embeddingProvider: identity.provider,
            embeddingModel: identity.model,
            embeddingDimensions: identity.dimensions.map(UInt32.init),
            embeddingNormalized: identity.normalized
        )
    }

    static func isCompatible(_ binding: MemoryBinding, with identity: EmbeddingIdentity) -> Bool {
        if let expected = binding.embeddingDimensions, let actual = identity.dimensions.map(UInt32.init), expected != actual { return false }
        if let expected = binding.embeddingModel, let actual = identity.model, expected != actual { return false }
        if let expected = binding.embeddingProvider, let actual = identity.provider, expected != actual { return false }
        if let expected = binding.embeddingNormalized, let actual = identity.normalized, expected != actual { return false }
        return true
    }
}
```

**Step 5: Validate in `MemoryOrchestrator`**

- On init/open: if store has binding and embedder has identity, fail-fast on incompatibility.
- During ingest: after first successful embedding write, call `setMemoryBindingIfMissing(...)`.

**Step 6: Integration tests**

```swift
@Test func reopeningWithDifferentEmbedderThrows() async throws {
    struct MismatchedEmbedder: EmbeddingProvider {
        let dimensions: Int = 4
        let normalize: Bool = true
        let identity: EmbeddingIdentity? = EmbeddingIdentity(
            provider: "Other",
            model: "Other",
            dimensions: 4,
            normalized: true
        )
        let executionMode: ProviderExecutionMode = .onDeviceOnly

        func embed(_ text: String) async throws -> [Float] {
            _ = text
            return [1, 0, 0, 0]
        }
    }

    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = true

        let first = try await MemoryOrchestrator(at: url, config: config, embedder: DeterministicTextEmbedder())
        try await first.remember("seed")
        try await first.close()

        // Use a test embedder with a different identity (provider/model/dims)
        let mismatched = MismatchedEmbedder()
        await #expect(throws: WaxError.self) {
            _ = try await MemoryOrchestrator(at: url, config: config, embedder: mismatched)
        }
    }
}
```

**Step 7: Run tests + commit**

```bash
swift test --filter MemoryBindingTests
swift test --filter ModelBindingTests
git add Sources/WaxCore/FileFormat/MemoryBinding.swift Sources/WaxCore/FileFormat/WaxTOC.swift \
    Sources/WaxCore/Wax.swift Sources/Wax/Embeddings/MemoryBindingCompatibility.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift \
    Tests/WaxCoreTests/MemoryBindingTests.swift Tests/WaxIntegrationTests/ModelBindingTests.swift
git commit -m "feat: bind embedding model identity at store level with compatibility checks"
```

---

## Task 4: Version Relations (Sets/Updates/Extends)

**Why:** Wax supports retract but not explicit semantic relations for evolving facts.

**False-positive cleanup:** integration tests must use public APIs only (no direct `session.textEngine` access).

**Files:**
- Create: `Sources/WaxCore/StructuredMemory/VersionRelation.swift`
- Modify: `Sources/WaxTextSearch/StructuredMemorySchema.swift` (`version_relation` column on `sm_fact`)
- Modify: `Sources/WaxTextSearch/FTS5Schema.swift` (schema version bump + migration path)
- Modify: `Sources/WaxTextSearch/FTS5SearchEngine.swift` (store relation and supersede on update/retract semantics)
- Modify: `Sources/Wax/StructuredMemorySession.swift` (thread relation through)
- Modify: `Sources/Wax/WaxSession.swift` (thread relation through)
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` (`assertFact(... relation: VersionRelation = .sets, ...)`)
- Modify: `Sources/WaxCLI/FactsCommand.swift` (`--relation`)
- Create: `Tests/WaxIntegrationTests/VersionRelationTests.swift`

**Step 1: TDD — unit enum tests**

```swift
@Test func versionRelationRawValues() {
    #expect(VersionRelation.sets.rawValue == 0)
    #expect(VersionRelation.updates.rawValue == 1)
    #expect(VersionRelation.extends.rawValue == 2)
    #expect(VersionRelation.retracts.rawValue == 3)
}
```

**Step 2: Implement enum in WaxCore**

```swift
public enum VersionRelation: UInt8, Sendable, Equatable, CaseIterable {
    case sets = 0
    case updates = 1
    case extends = 2
    case retracts = 3

    public var supersedes: Bool {
        switch self {
        case .updates, .retracts: return true
        case .sets, .extends: return false
        }
    }
}
```

**Step 3: Plumb relation through public APIs**

- Add `relation: VersionRelation = .sets` to:
  - `FTS5SearchEngine.assertFact`
  - `StructuredMemorySession.assertFact`
  - `WaxSession.assertFact`
  - `MemoryOrchestrator.assertFact`

When `relation.supersedes == true`, close open spans for same `(subject, predicate)` before inserting new span.

Add explicit migration:
- bump SQLite `user_version`
- `ALTER TABLE sm_fact ADD COLUMN version_relation INTEGER NOT NULL DEFAULT 0` for existing stores
- add migration test fixture(s) from pre-migration schema to validate open/read/write upgrade path.

**Step 4: Integration test via `MemoryOrchestrator` public API**

```swift
@Test func updateFactRetractsPrior() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false
        config.enableStructuredMemory = true
        let orchestrator = try await MemoryOrchestrator(at: url, config: config)

        _ = try await orchestrator.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("employer"),
            object: .string("Google"),
            relation: .sets
        )

        _ = try await orchestrator.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("employer"),
            object: .string("Anthropic"),
            relation: .updates
        )

        let result = try await orchestrator.facts(
            about: EntityKey("user:chris"),
            predicate: PredicateKey("employer"),
            asOfMs: Int64.max,
            limit: 10
        )
        #expect(result.hits.count == 1)
        #expect(result.hits.first?.fact.object == .string("Anthropic"))
        try await orchestrator.close()
    }
}
```

**Step 5: CLI support**

```swift
@Option(name: .long, help: "Version relation: sets, updates, extends, retracts")
var relation: String = "sets"
```

Parse into `VersionRelation` and pass to `memory.assertFact(...)`.

**Step 6: Run tests + commit**

```bash
swift test --filter VersionRelationTests
git add Sources/WaxCore/StructuredMemory/VersionRelation.swift \
    Sources/WaxTextSearch/StructuredMemorySchema.swift \
    Sources/WaxTextSearch/FTS5Schema.swift \
    Sources/WaxTextSearch/FTS5SearchEngine.swift \
    Sources/Wax/StructuredMemorySession.swift Sources/Wax/WaxSession.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift Sources/WaxCLI/FactsCommand.swift \
    Tests/WaxIntegrationTests/VersionRelationTests.swift
git commit -m "feat: add fact version relations and public API support"
```

---

## Task 5: Temporal NLP Parser

**Why:** The largest gap (Wax 40 vs memvid 82). Agents ask "what did I discuss last week?" and Wax has no way to resolve "last week" to a date range. Memvid has a full NLP temporal normalizer.

**Memvid reference:**
```rust
// memvid-main/src/analysis/temporal.rs:82-168
pub struct TemporalNormalizer { context: TemporalContext }

pub fn resolve(&self, phrase: &str) -> Result<TemporalResolution> {
    let lower = trimmed.to_ascii_lowercase();
    if let Some(r) = self.resolve_fixed(&lower)           { return Ok(r); }
    if let Some(r) = self.resolve_relative_days(&lower)   { return Ok(r); }
    if let Some(r) = self.resolve_relative_weeks(&lower)  { return Ok(r); }
    if let Some(r) = self.resolve_weekday_phrases(&lower) { return Ok(r); }
    // ...
}

fn resolve_fixed(&self, phrase: &str) -> Option<TemporalResolution> {
    match phrase {
        "today"      => Some(self.date_resolution(self.anchor_date())),
        "yesterday"  => Some(self.date_resolution(add_days(self.anchor_date(), -1))),
        "last week"  => Some(self.week_range(-1)),
        "q4 2025"    => Some(self.quarter_range(2025, 4)),
        // ...
    }
}
```

**Files:**
- Create: `Sources/Wax/Temporal/TemporalNormalizer.swift`
- Create: `Sources/Wax/Temporal/TemporalResolution.swift`
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` — integrate temporal in `recall()`
- Modify: `Sources/Wax/RAG/FastRAGContextBuilder.swift` — plumb `timeRange` into `SearchRequest`
- Create: `Tests/WaxTests/TemporalNormalizerTests.swift`
- Create: `Tests/WaxIntegrationTests/TemporalRecallIntegrationTests.swift`

**Step 1: Write comprehensive failing tests**

```swift
// Tests/WaxTests/TemporalNormalizerTests.swift
import Foundation
import Testing
@testable import Wax

@Test func resolvesToday() throws {
    let anchor = Date(timeIntervalSince1970: 1_740_000_000) // 2025-02-19
    let normalizer = TemporalNormalizer(anchor: anchor)
    let result = try normalizer.resolve("today")
    #expect(result.kind == .date)
    let cal = Calendar(identifier: .gregorian)
    #expect(cal.isDate(result.start, inSameDayAs: anchor))
}

@Test func resolvesYesterday() throws {
    let anchor = Date(timeIntervalSince1970: 1_740_000_000)
    let normalizer = TemporalNormalizer(anchor: anchor)
    let result = try normalizer.resolve("yesterday")
    let cal = Calendar(identifier: .gregorian)
    let expected = cal.date(byAdding: .day, value: -1, to: anchor)!
    #expect(cal.isDate(result.start, inSameDayAs: expected))
}

@Test func resolvesLastWeek() throws {
    let anchor = Date(timeIntervalSince1970: 1_740_000_000)
    let normalizer = TemporalNormalizer(anchor: anchor)
    let result = try normalizer.resolve("last week")
    #expect(result.kind == .range)
    #expect(result.end != nil)
    #expect(result.start < anchor)
    #expect(result.end! < anchor)
}

@Test func resolvesRelativeDays() throws {
    let anchor = Date(timeIntervalSince1970: 1_740_000_000)
    let normalizer = TemporalNormalizer(anchor: anchor)

    let inThree = try normalizer.resolve("in 3 days")
    let cal = Calendar(identifier: .gregorian)
    let expected = cal.date(byAdding: .day, value: 3, to: anchor)!
    #expect(cal.isDate(inThree.start, inSameDayAs: expected))

    let twoAgo = try normalizer.resolve("2 days ago")
    let expectedAgo = cal.date(byAdding: .day, value: -2, to: anchor)!
    #expect(cal.isDate(twoAgo.start, inSameDayAs: expectedAgo))
}

@Test func resolvesLastFriday() throws {
    // 2025-02-19 is a Wednesday
    let anchor = Date(timeIntervalSince1970: 1_740_000_000)
    let normalizer = TemporalNormalizer(anchor: anchor)
    let result = try normalizer.resolve("last friday")
    let cal = Calendar(identifier: .gregorian)
    let weekday = cal.component(.weekday, from: result.start)
    #expect(weekday == 6) // Friday = 6 in Calendar
    #expect(result.start < anchor)
}

@Test func resolvesQuarter() throws {
    let normalizer = TemporalNormalizer(anchor: Date())
    let result = try normalizer.resolve("q3 2025")
    #expect(result.kind == .range)
    let cal = Calendar(identifier: .gregorian)
    #expect(cal.component(.month, from: result.start) == 7) // Q3 starts July
    #expect(cal.component(.month, from: result.end!) == 10) // Q3 ends before Oct
}

@Test func unsupportedPhraseThrows() {
    let normalizer = TemporalNormalizer(anchor: Date())
    #expect(throws: WaxError.self) {
        _ = try normalizer.resolve("the heat death of the universe")
    }
}
```

**Step 2: Run to verify failure**

```bash
swift test --filter TemporalNormalizerTests
```

**Step 3: Implement TemporalResolution type**

```swift
// Sources/Wax/Temporal/TemporalResolution.swift
import Foundation

public struct TemporalResolution: Sendable, Equatable {
    public enum Kind: Sendable, Equatable {
        case date       // Single day
        case dateTime   // Specific time
        case range      // Start..end
    }

    public var kind: Kind
    public var start: Date
    public var end: Date?

    /// Convert to milliseconds-since-epoch range for `TimeRange`
    public var asTimeRange: (afterMs: Int64, beforeMs: Int64) {
        let startMs = Int64(start.timeIntervalSince1970 * 1000)
        let endMs: Int64
        if let end = end {
            endMs = Int64(end.timeIntervalSince1970 * 1000)
        } else {
            // Single date: expand to full day
            let cal = Calendar(identifier: .gregorian)
            let endOfDay = cal.date(byAdding: .day, value: 1, to: cal.startOfDay(for: start))!
            endMs = Int64(endOfDay.timeIntervalSince1970 * 1000)
        }
        return (startMs, endMs)
    }
}
```

**Step 4: Implement TemporalNormalizer**

```swift
// Sources/Wax/Temporal/TemporalNormalizer.swift
import Foundation

/// Pure Swift temporal phrase parser inspired by memvid's TemporalNormalizer
/// (memvid-main/src/analysis/temporal.rs:82-168).
///
/// Uses Foundation.Calendar for date math — no external dependencies.
public struct TemporalNormalizer: Sendable {
    public let anchor: Date
    public let calendar: Calendar

    public init(anchor: Date = Date(), calendar: Calendar = .init(identifier: .gregorian)) {
        self.anchor = anchor
        self.calendar = calendar
    }

    public func resolve(_ phrase: String) throws -> TemporalResolution {
        let lower = phrase.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
        guard !lower.isEmpty else {
            throw WaxError.io("empty temporal phrase")
        }

        if let r = resolveFixed(lower)         { return r }
        if let r = resolveRelativeDays(lower)  { return r }
        if let r = resolveRelativeWeeks(lower) { return r }
        if let r = resolveWeekday(lower)       { return r }
        if let r = resolveQuarter(lower)       { return r }

        throw WaxError.io("unsupported temporal phrase: \(phrase)")
    }

    // MARK: - Fixed phrases

    private func resolveFixed(_ phrase: String) -> TemporalResolution? {
        switch phrase {
        case "today":
            return dateResolution(anchor)
        case "yesterday":
            return dateResolution(addDays(-1))
        case "tomorrow":
            return dateResolution(addDays(1))
        case "last week":
            return weekRange(offset: -1)
        case "this week":
            return weekRange(offset: 0)
        case "next week":
            return weekRange(offset: 1)
        case "last month":
            return monthRange(offset: -1)
        case "this month":
            return monthRange(offset: 0)
        case "next month":
            return monthRange(offset: 1)
        default:
            return nil
        }
    }

    // MARK: - Relative days ("in 3 days", "5 days ago")

    private func resolveRelativeDays(_ phrase: String) -> TemporalResolution? {
        // "in N days"
        if phrase.hasPrefix("in ") && phrase.hasSuffix(" days"),
           let n = parseNumber(String(phrase.dropFirst(3).dropLast(5))) {
            return dateResolution(addDays(n))
        }
        // "N days ago"
        if phrase.hasSuffix(" days ago"),
           let n = parseNumber(String(phrase.dropLast(9))) {
            return dateResolution(addDays(-n))
        }
        return nil
    }

    // MARK: - Relative weeks ("in 2 weeks", "3 weeks ago")

    private func resolveRelativeWeeks(_ phrase: String) -> TemporalResolution? {
        if phrase.hasPrefix("in ") && phrase.hasSuffix(" weeks"),
           let n = parseNumber(String(phrase.dropFirst(3).dropLast(6))) {
            return dateResolution(addWeeks(n))
        }
        if phrase.hasSuffix(" weeks ago"),
           let n = parseNumber(String(phrase.dropLast(10))) {
            return dateResolution(addWeeks(-n))
        }
        return nil
    }

    // MARK: - Weekday phrases ("last friday", "next monday")

    private func resolveWeekday(_ phrase: String) -> TemporalResolution? {
        let parts = phrase.split(separator: " ")
        guard parts.count == 2 else { return nil }

        let direction = parts[0]
        guard let weekday = parseWeekday(String(parts[1])) else { return nil }

        let offset: Int
        switch direction {
        case "last": offset = -1
        case "next": offset = 1
        case "this": offset = 0
        default: return nil
        }

        return dateResolution(findWeekday(weekday, weeksOffset: offset))
    }

    // MARK: - Quarters ("q3 2025", "q1 2026")

    private func resolveQuarter(_ phrase: String) -> TemporalResolution? {
        guard phrase.count >= 6,
              phrase.first == "q",
              let quarter = Int(String(phrase[phrase.index(phrase.startIndex, offsetBy: 1)])),
              quarter >= 1 && quarter <= 4 else { return nil }

        let yearStr = phrase.dropFirst(2).trimmingCharacters(in: .whitespaces)
        guard let year = Int(yearStr), year >= 1970 && year <= 2100 else { return nil }

        let startMonth = (quarter - 1) * 3 + 1
        let endMonth = startMonth + 3

        var startComps = DateComponents()
        startComps.year = year
        startComps.month = startMonth
        startComps.day = 1

        var endComps = DateComponents()
        endComps.year = endMonth > 12 ? year + 1 : year
        endComps.month = endMonth > 12 ? endMonth - 12 : endMonth
        endComps.day = 1

        guard let start = calendar.date(from: startComps),
              let end = calendar.date(from: endComps) else { return nil }

        return TemporalResolution(kind: .range, start: start, end: end)
    }

    // MARK: - Helpers

    private func dateResolution(_ date: Date) -> TemporalResolution {
        TemporalResolution(kind: .date, start: calendar.startOfDay(for: date))
    }

    private func addDays(_ n: Int) -> Date {
        calendar.date(byAdding: .day, value: n, to: anchor)!
    }

    private func addWeeks(_ n: Int) -> Date {
        calendar.date(byAdding: .weekOfYear, value: n, to: anchor)!
    }

    private func weekRange(offset: Int) -> TemporalResolution {
        let targetWeek = calendar.date(byAdding: .weekOfYear, value: offset, to: anchor)!
        let startOfWeek = calendar.dateInterval(of: .weekOfYear, for: targetWeek)!
        return TemporalResolution(kind: .range, start: startOfWeek.start, end: startOfWeek.end)
    }

    private func monthRange(offset: Int) -> TemporalResolution {
        let targetMonth = calendar.date(byAdding: .month, value: offset, to: anchor)!
        let interval = calendar.dateInterval(of: .month, for: targetMonth)!
        return TemporalResolution(kind: .range, start: interval.start, end: interval.end)
    }

    private func findWeekday(_ target: Int, weeksOffset: Int) -> Date {
        let currentWeekday = calendar.component(.weekday, from: anchor)
        var dayDiff = target - currentWeekday
        if weeksOffset < 0 {
            if dayDiff >= 0 { dayDiff -= 7 }
            dayDiff += (weeksOffset + 1) * 7
        } else if weeksOffset > 0 {
            if dayDiff <= 0 { dayDiff += 7 }
            dayDiff += (weeksOffset - 1) * 7
        }
        return calendar.date(byAdding: .day, value: dayDiff, to: anchor)!
    }

    private func parseWeekday(_ name: String) -> Int? {
        switch name {
        case "sunday":    return 1
        case "monday":    return 2
        case "tuesday":   return 3
        case "wednesday": return 4
        case "thursday":  return 5
        case "friday":    return 6
        case "saturday":  return 7
        default:          return nil
        }
    }

    private func parseNumber(_ s: String) -> Int? {
        if let n = Int(s) { return n }
        switch s.trimmingCharacters(in: .whitespaces) {
        case "one", "a": return 1
        case "two":      return 2
        case "three":    return 3
        case "four":     return 4
        case "five":     return 5
        case "six":      return 6
        case "seven":    return 7
        default:         return nil
        }
    }
}
```

**Step 5: Run tests**

```bash
swift test --filter TemporalNormalizerTests
```

**Step 6: Integrate into recall()**

In `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`, parse temporal phrases in `recall()` and thread `timeRange` through the existing recall pipeline:
`recall(...) -> buildRecallContext(...) -> FastRAGContextBuilder.build(...) -> SearchRequest(timeRange:)`.

```swift
public func recall(query: String, ...) async throws -> RAGContext {
    let embedding = try await queryEmbedding(for: query, policy: .ifAvailable)
    let normalizer = TemporalNormalizer(anchor: Date())
    let parsedRange = extractTemporalRange(from: query, normalizer: normalizer)
    let timeRange = parsedRange.map { TimeRange(after: $0.afterMs, before: $0.beforeMs) }
    return try await buildRecallContext(query: query, embedding: embedding, timeRange: timeRange)
}

private func buildRecallContext(
    query: String,
    embedding: [Float]?,
    frameFilter: FrameFilter? = nil,
    timeRange: TimeRange? = nil
) async throws -> RAGContext {
    return try await ragBuilder.build(
        query: query,
        embedding: embedding,
        vectorEnginePreference: preference,
        wax: wax,
        session: session,
        frameFilter: frameFilter,
        timeRange: timeRange,
        accessStatsManager: config.enableAccessStatsScoring ? accessStatsManager : nil,
        config: ragConfigForRecall()
    )
}

private func extractTemporalRange(
    from query: String,
    normalizer: TemporalNormalizer
) -> (afterMs: Int64, beforeMs: Int64)? {
    // Temporal phrase detection: parse a temporal subphrase from full query text.
    // Sliding n-gram scan over the query (max phrase len 4 words).
    // This supports phrases embedded in longer queries:
    // "what did we discuss last week about vector search"
    let words = query.lowercased().split(whereSeparator: \.isWhitespace).map(String.init)
    guard !words.isEmpty else { return nil }
    for window in stride(from: min(4, words.count), through: 1, by: -1) {
        for i in 0...(words.count - window) {
            let candidate = words[i..<(i + window)].joined(separator: " ")
            if let resolution = try? normalizer.resolve(candidate) {
                return resolution.asTimeRange
            }
        }
    }
    return nil
}
```

Also update `FastRAGContextBuilder.build` signature to accept `timeRange: TimeRange?` and pass it into the `SearchRequest` initializer.

**Step 7: Add end-to-end integration tests**

Add an integration test that ingests frames with controlled timestamps and asserts that a query containing a temporal phrase (for example, `"last week"`) changes the recalled set via `timeRange`.

**Step 8: Commit**

```bash
git add Sources/Wax/Temporal/ Tests/WaxTests/TemporalNormalizerTests.swift \
    Tests/WaxIntegrationTests/TemporalRecallIntegrationTests.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift \
    Sources/Wax/RAG/FastRAGContextBuilder.swift
git commit -m "feat: add temporal NLP parser for date-aware recall queries"
```

---

## Task 6: Async Enrichment Pipeline

**Why:** Wax's `remember()` does synchronous FTS5 indexing in the same call. Memvid returns instantly from `put()` and runs entity extraction, tagging, and triplet generation in a background worker thread. This pattern is especially valuable for agent loops that ingest rapidly.

**Memvid reference:**
```rust
// memvid-main/src/enrichment_worker.rs:359-438
pub fn run_worker_loop<G, P, M, C>(
    handle: &EnrichmentWorkerHandle,
    config: &EnrichmentWorkerConfig,
    mut get_next_task: G,      // || -> Option<EnrichmentTask>
    mut process_task: P,       // |&task| -> TaskResult
    mut mark_complete: M,      // |frame_id|
    mut checkpoint: C,         // ||
) {
    while !handle.should_stop() {
        let task = get_next_task().unwrap_or_else(|| { sleep; continue });
        let result = process_task(&task);
        handle.inc_frames_processed();
        mark_complete(task.frame_id);
        tasks_since_checkpoint += 1;
        if tasks_since_checkpoint >= config.checkpoint_interval { checkpoint(); }
    }
}

// memvid-main/src/memvid/enrichment.rs:86-143
pub fn start_enrichment_worker(memvid: Arc<Mutex<Memvid>>, config) -> EnrichmentHandle {
    let thread = std::thread::spawn(move || {
        run_worker_loop(&handle, &config,
            || { mv.next_enrichment_task() },
            |task| { mv.process_enrichment_task(task) },
            |id| { mv.complete_enrichment_task(id); },
            || { mv.commit().ok(); },
        );
    });
    EnrichmentHandle { handle, thread }
}
```

**Files:**
- Create: `Sources/Wax/Enrichment/EnrichmentTask.swift`
- Create: `Sources/Wax/Enrichment/EnrichmentPipeline.swift`
- Create: `Sources/Wax/Enrichment/KeywordExtractor.swift`
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` — enqueue enrichment after frame persist
- Create: `Tests/WaxTests/KeywordExtractorTests.swift`
- Create: `Tests/WaxIntegrationTests/EnrichmentPipelineTests.swift`

**Step 1: Write failing test for KeywordExtractor**

```swift
// Tests/WaxTests/KeywordExtractorTests.swift
import Testing
@testable import Wax

@Test func extractsTopKeywords() {
    let text = "Swift concurrency enables structured concurrency patterns with actors and async await"
    let keywords = KeywordExtractor.extract(from: text, topK: 5)
    #expect(keywords.contains("concurrency"))
    #expect(keywords.contains("swift"))
    #expect(keywords.count <= 5)
    // Stopwords excluded
    #expect(!keywords.contains("and"))
    #expect(!keywords.contains("with"))
}

@Test func emptyTextReturnsEmpty() {
    let keywords = KeywordExtractor.extract(from: "", topK: 10)
    #expect(keywords.isEmpty)
}
```

**Step 2: Implement KeywordExtractor**

```swift
// Sources/Wax/Enrichment/KeywordExtractor.swift
import Foundation

/// Extracts top-K keywords from text by token frequency, filtering stopwords.
/// Inspired by memvid's auto_tag.rs (memvid-main/src/analysis/auto_tag.rs:21-39).
public enum KeywordExtractor {
    private static let stopwords: Set<String> = [
        "the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for",
        "of", "with", "by", "from", "is", "are", "was", "were", "be", "been",
        "being", "have", "has", "had", "do", "does", "did", "will", "would",
        "could", "should", "may", "might", "can", "shall", "it", "its",
        "this", "that", "these", "those", "i", "you", "he", "she", "we", "they",
        "not", "no", "nor", "so", "if", "then", "than", "too", "very", "just",
    ]

    public static func extract(from text: String, topK: Int = 12) -> [String] {
        let tokens = text.lowercased()
            .components(separatedBy: CharacterSet.alphanumerics.inverted)
            .filter { $0.count >= 3 && !stopwords.contains($0) }

        var frequency: [String: Int] = [:]
        for token in tokens {
            frequency[token, default: 0] += 1
        }

        return frequency
            .sorted { $0.value > $1.value }
            .prefix(topK)
            .map(\.key)
    }
}
```

**Step 3: Write failing test for EnrichmentPipeline**

```swift
// Tests/WaxIntegrationTests/EnrichmentPipelineTests.swift
import Foundation
import Testing
@testable import Wax

@Test func enrichmentPipelineProcessesEnqueuedTasks() async throws {
    let pipeline = EnrichmentPipeline()
    await pipeline.start { task in
        return EnrichmentResult(
            frameId: task.frameId,
            keywords: KeywordExtractor.extract(from: task.text),
            entities: []
        )
    }

    try await pipeline.enqueue(EnrichmentTask(frameId: 1, text: "Swift concurrency is great"))
    try await pipeline.enqueue(EnrichmentTask(frameId: 2, text: "Rust ownership model"))

    // Deterministic wait for drain
    try await pipeline.waitUntilProcessed(atLeast: 2, timeout: .seconds(2))
    try await pipeline.stop()

    #expect(await pipeline.stats >= 2)
}
```

**Step 4: Implement EnrichmentTask and EnrichmentPipeline**

```swift
// Sources/Wax/Enrichment/EnrichmentTask.swift
import Foundation

public struct EnrichmentTask: Sendable {
    public let frameId: UInt64
    public let text: String
}

public struct EnrichmentResult: Sendable {
    public let frameId: UInt64
    public let keywords: [String]
    public let entities: [(subject: String, predicate: String, object: String)]
}
```

```swift
// Sources/Wax/Enrichment/EnrichmentPipeline.swift
import Foundation

/// Background enrichment pipeline for post-ingest processing.
/// Decouples frame persistence from NLP enrichment.
///
/// Inspired by memvid's enrichment worker
/// (memvid-main/src/enrichment_worker.rs:359-438).
public actor EnrichmentPipeline {
    private enum State { case idle, running, stopping, stopped }

    private var state: State = .idle
    private var stream: AsyncStream<EnrichmentTask>?
    private var continuation: AsyncStream<EnrichmentTask>.Continuation?
    private var processingTask: Task<Void, Never>?
    private var processedCount: UInt64 = 0
    private var pendingCount: UInt64 = 0

    public init() {}

    public func start(
        handler: @escaping @Sendable (EnrichmentTask) async -> EnrichmentResult
    ) {
        guard state == .idle || state == .stopped else { return }
        let (stream, continuation) = AsyncStream<EnrichmentTask>.makeStream()
        self.stream = stream
        self.continuation = continuation
        state = .running

        processingTask = Task {
            for await task in stream {
                let _ = await handler(task)
                processedCount += 1
                if pendingCount > 0 { pendingCount -= 1 }
            }
        }
    }

    public func enqueue(_ task: EnrichmentTask) throws {
        guard state == .running, continuation != nil else {
            throw WaxError.io("enrichment pipeline not running")
        }
        pendingCount += 1
        continuation?.yield(task)
    }

    public func stop(timeout: Duration = .seconds(2)) async throws {
        guard state == .running || state == .stopping else { return }
        state = .stopping
        continuation?.finish()
        if let processingTask {
            let completed = await withTaskGroup(of: Bool.self) { group in
                group.addTask {
                    _ = await processingTask.result
                    return true
                }
                group.addTask {
                    try? await Task.sleep(for: timeout)
                    return false
                }
                let first = await group.next() ?? false
                group.cancelAll()
                return first
            }
            if !completed {
                processingTask.cancel()
            }
        }
        processingTask = nil
        state = .stopped
    }

    public var stats: UInt64 { processedCount }

    public func waitUntilProcessed(atLeast target: UInt64, timeout: Duration) async throws {
        let deadline = ContinuousClock.now + timeout
        while processedCount < target {
            if ContinuousClock.now >= deadline {
                throw WaxError.io("enrichment timeout waiting for \(target) tasks")
            }
            if state == .stopped && pendingCount > 0 {
                throw WaxError.io("enrichment stopped with pending tasks")
            }
            try await Task.sleep(for: .milliseconds(20))
        }
    }
}
```

**Step 5: Wire into MemoryOrchestrator**

In `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`, add `enrichmentPipeline` property. After each batch of frames is persisted in `remember()`, enqueue enrichment:

```swift
// After frame persist + embedding:
if let pipeline = enrichmentPipeline {
    for (frameId, chunkText) in zip(frameIds, batchChunks) {
        try await pipeline.enqueue(EnrichmentTask(
            frameId: frameId,
            text: chunkText
        ))
    }
}
```

Add pipeline lifecycle to `close()`:

```swift
public func close() async throws {
    try await enrichmentPipeline?.stop(timeout: .seconds(2))
    // ... existing close logic ...
}
```

**Step 6: Run tests**

```bash
swift test --filter EnrichmentPipeline
swift test --filter KeywordExtractor
```

**Step 7: Commit**

```bash
git add Sources/Wax/Enrichment/ \
    Tests/WaxTests/KeywordExtractorTests.swift \
    Tests/WaxIntegrationTests/EnrichmentPipelineTests.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift
git commit -m "feat: add async enrichment pipeline with keyword extraction"
```

---

## Task 7: iOS Data Protection

**Why:** Wax currently does not set file protection attributes for `.wax` files on iOS-family platforms. This is a platform-specific hardening task and requires device-level validation for lock-state guarantees.

**Files:**
- Modify: `Sources/WaxCore/Wax.swift` — set file protection after create/open
- Create: `Tests/WaxCoreTests/DataProtectionTests.swift`

**Step 1: Write failing test**

```swift
// Tests/WaxCoreTests/DataProtectionTests.swift
import Foundation
import Testing
@testable import WaxCore

#if os(iOS)
@Test func waxFileSetsCompleteProtection() async throws {
    let url = TempFiles.uniqueURL()
    defer { try? FileManager.default.removeItem(at: url) }

    let wax = try await Wax.create(at: url)
    try await wax.close()

    let attributes = try FileManager.default.attributesOfItem(atPath: url.path)
    let protection = attributes[.protectionKey] as? FileProtectionType
    #expect(protection == .complete)
}
#endif

@Test func waxFileIsReadableAfterCreate() async throws {
    let url = TempFiles.uniqueURL()
    defer { try? FileManager.default.removeItem(at: url) }

    let wax = try await Wax.create(at: url)
    try await wax.close()
    #expect(FileManager.default.isReadableFile(atPath: url.path))
}
```

**Step 2: Implement file protection**

In `Sources/WaxCore/Wax.swift`, after creating or opening the file, set the protection attribute:

```swift
// After file creation:
#if os(iOS) || os(watchOS) || os(tvOS) || os(visionOS)
try FileManager.default.setAttributes(
    [.protectionKey: FileProtectionType.complete],
    ofItemAtPath: url.path
)
#endif
```

Run this on iOS simulator/device (not macOS-only `swift test`) and treat simulator lock-state behavior as `Unverified` for confidentiality guarantees. Use a package/app test scheme that includes `WaxCoreTests`:

```bash
xcodebuild test \
  -scheme <YourTestScheme> \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```

**Step 3: Run tests**

```bash
swift test --filter DataProtection
```

`swift test` validates compile/runtime basics only; it does not prove locked-device confidentiality. Add a manual/device validation checklist before closing this task.

**Step 4: Commit**

```bash
git add Sources/WaxCore/Wax.swift Tests/WaxCoreTests/DataProtectionTests.swift
git commit -m "feat: set NSFileProtectionComplete on .wax files for iOS data protection"
```

---

## Execution Order & Dependencies

```
Wave 1 (parallel): Task 1 (Benchmarks), Task 7 (Data Protection)
Wave 2 (sequential): Task 2 (Content Dedup) -> Task 3 (Model Binding)
Wave 3 (sequential): Task 4 (Version Relations) -> Task 5 (Temporal NLP)
Wave 4 (sequential): Task 6 (Enrichment)
```

**Rationale:**
- Tasks 2/3/4/5/6 all touch `MemoryOrchestrator.swift`, so they should not be developed in parallel.
- Task 4 requires schema migration groundwork before downstream behavior validation.
- Task 6 depends on stable ingest semantics from Task 2 and should run last.

---

## Post-Implementation Validation Criteria

> Numeric score improvements are `Unverified` until measured in this repository.

- All new APIs compile against current target boundaries (`WaxCore` remains independent of `WaxVectorSearch`).
- Migration tests pass for pre-change structured-memory DB files and pre-change TOC files.
- Temporal phrase parsing affects recall candidate filtering via `timeRange` in integration tests.
- Enrichment pipeline has deterministic drain/stop behavior with timeout + cancellation coverage.
- Benchmark jobs produce deterministic fixture inputs and stable regression signals (relative deltas, not only absolute thresholds).
