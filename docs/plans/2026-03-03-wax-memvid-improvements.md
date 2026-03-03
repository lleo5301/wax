# Wax v2: Closing the Gaps — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Address Wax's 7 verified weaknesses by adopting proven patterns from memvid's Rust codebase, adapted for Swift 6.2 and Apple platforms.

**Architecture:** Each improvement is an independent module change, touching 1-2 Wax targets. The TOC already has reserved extension slots for `memory_binding` and `enrichment_queue` (WaxTOC.swift:124-126). We exploit these rather than bumping the format version. All changes maintain backward compatibility — v1 files remain readable.

**Tech Stack:** Swift 6.2, swift-testing, Foundation.Calendar, CryptoKit (SHA-256), Apple Data Protection

**Source of inspiration:** memvid Rust codebase at `memvid-main/` — specific file references included per task.

---

## Verified Weaknesses (from head-to-head analysis)

| # | Gap | Wax Score | Memvid Score | Delta |
|---|---|:---:|:---:|:---:|
| 1 | No benchmark suite | 0 | 40 | -40 |
| 2 | No frame content dedup in `remember()` | 62 | 82 | -20 |
| 3 | No store-level embedding model binding | 75 | 85 | -10 |
| 4 | Only Retracts, no Sets/Updates/Extends | 78 | 80 | -2 |
| 5 | No temporal NLP ("last Tuesday") | 40 | 82 | -42 |
| 6 | No async enrichment pipeline | 62 | 82 | -20 |
| 7 | No data protection on .wax files | 10 | 78 | -68 |

---

## Task 1: Benchmark Suite

**Why:** Neither project benchmarks E2E `remember→recall`. Memvid has Criterion benchmarks for vector search scaling (10K/50K/100K) at `memvid-main/benches/vec_search_benchmark.rs`. Wax has zero benchmarks. Can't optimize what you don't measure.

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
- Create: `Tests/WaxBenchmarks/RememberRecallBenchmark.swift`
- Create: `Tests/WaxBenchmarks/VectorSearchBenchmark.swift`
- Create: `Tests/WaxBenchmarks/HybridSearchBenchmark.swift`
- Modify: `Package.swift` — add `WaxBenchmarks` test target

**Step 1: Add benchmark test target to Package.swift**

Add a new test target in `Package.swift` after the existing test targets:

```swift
.testTarget(
    name: "WaxBenchmarks",
    dependencies: [
        "Wax",
        .product(name: "Testing", package: "swift-testing"),
    ],
    path: "Tests/WaxBenchmarks"
),
```

**Step 2: Write the remember→recall benchmark**

```swift
// Tests/WaxBenchmarks/RememberRecallBenchmark.swift
import Foundation
import Testing
import Wax

@Test("Benchmark: remember() 100 frames, text-only")
func benchmarkRemember100Frames() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false

        let orchestrator = try await MemoryOrchestrator(at: url, config: config)
        let content = String(repeating: "The quick brown fox jumps over the lazy dog. ", count: 20)

        let start = ContinuousClock.now
        for i in 0..<100 {
            try await orchestrator.remember("\(content) Frame \(i)")
        }
        try await orchestrator.flush()
        let ingestElapsed = ContinuousClock.now - start

        // recall benchmark
        let recallStart = ContinuousClock.now
        for _ in 0..<50 {
            _ = try await orchestrator.recall(query: "quick brown fox")
        }
        let recallElapsed = ContinuousClock.now - recallStart

        try await orchestrator.close()

        let ingestMs = ingestElapsed.components.seconds * 1000 + ingestElapsed.components.attoseconds / 1_000_000_000_000_000
        let recallMs = recallElapsed.components.seconds * 1000 + recallElapsed.components.attoseconds / 1_000_000_000_000_000

        print("BENCHMARK remember(100 frames): \(ingestMs) ms")
        print("BENCHMARK recall(50 queries):    \(recallMs) ms")

        // Regression gate: remember 100 frames should complete < 5s on any Apple Silicon
        #expect(ingestMs < 5000, "remember() regression: \(ingestMs) ms > 5000 ms budget")
        // Recall 50 queries should complete < 2s
        #expect(recallMs < 2000, "recall() regression: \(recallMs) ms > 2000 ms budget")
    }
}
```

**Step 3: Write the vector search scaling benchmark**

```swift
// Tests/WaxBenchmarks/VectorSearchBenchmark.swift
import Foundation
import Testing
import Wax
import WaxVectorSearch

@Test("Benchmark: USearch HNSW search at 1K/10K vectors", arguments: [1_000, 10_000])
func benchmarkVectorSearch(count: Int) async throws {
    try await TempFiles.withTempFile { url in
        let wax = try await Wax.create(at: url)
        let session = try await WaxVectorSearchSession(
            wax: wax, metric: .cosine, dimensions: 384
        )

        // Insert random vectors
        for i in 0..<UInt64(count) {
            var vec = [Float](repeating: 0, count: 384)
            for j in 0..<384 { vec[j] = Float.random(in: -1...1) }
            let norm = sqrt(vec.reduce(0) { $0 + $1 * $1 })
            vec = vec.map { $0 / norm }
            try await session.add(frameId: i, vector: vec)
        }

        // Query
        var query = [Float](repeating: 0, count: 384)
        for j in 0..<384 { query[j] = Float.random(in: -1...1) }
        let qnorm = sqrt(query.reduce(0) { $0 + $1 * $1 })
        query = query.map { $0 / qnorm }

        let start = ContinuousClock.now
        let iterations = 100
        for _ in 0..<iterations {
            _ = try await session.search(vector: query, topK: 10)
        }
        let elapsed = ContinuousClock.now - start
        let perQueryUs = (elapsed / iterations).components.attoseconds / 1_000_000_000_000

        print("BENCHMARK vector search \(count) vectors: \(perQueryUs) µs/query")
        // At 10K vectors, HNSW top-10 should be < 5ms per query
        #expect(perQueryUs < 5_000, "vector search regression at \(count): \(perQueryUs) µs > 5000 µs")

        try await wax.close()
    }
}
```

**Step 4: Write hybrid search benchmark**

```swift
// Tests/WaxBenchmarks/HybridSearchBenchmark.swift
import Foundation
import Testing
import Wax

@Test("Benchmark: hybrid recall with RRF fusion, 500 frames")
func benchmarkHybridRecall() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = true

        let embedder = DeterministicTextEmbedder(
            dimensions: 384, normalize: true,
            identity: EmbeddingIdentity(provider: "test", model: "bench", dimensions: 384, normalized: true),
            executionMode: .onDeviceOnly
        )
        let orchestrator = try await MemoryOrchestrator(at: url, config: config, embedder: embedder)

        for i in 0..<500 {
            try await orchestrator.remember("Document \(i) about Swift concurrency and actor isolation patterns")
        }
        try await orchestrator.flush()

        let start = ContinuousClock.now
        for _ in 0..<20 {
            _ = try await orchestrator.recall(query: "actor isolation")
        }
        let elapsed = ContinuousClock.now - start
        let perRecallMs = (elapsed / 20).components.seconds * 1000 + (elapsed / 20).components.attoseconds / 1_000_000_000_000_000

        print("BENCHMARK hybrid recall (500 frames): \(perRecallMs) ms/query")
        #expect(perRecallMs < 200, "hybrid recall regression: \(perRecallMs) ms > 200 ms budget")

        try await orchestrator.close()
    }
}
```

**Step 5: Run benchmarks to verify they pass**

```bash
swift test --filter WaxBenchmarks
```

**Step 6: Commit**

```bash
git add Tests/WaxBenchmarks/ Package.swift
git commit -m "feat: add benchmark suite for remember/recall, vector search, and hybrid search"
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
- Modify: `Sources/WaxCore/FileFormat/FrameMeta.swift` — add `contentHash` field
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` — dedup check in `remember()`
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

@Test func rememberIdenticalContentTwiceCreatesOneFrame() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false

        let orchestrator = try await MemoryOrchestrator(at: url, config: config)
        try await orchestrator.remember("Duplicate content test")
        try await orchestrator.remember("Duplicate content test") // same content
        try await orchestrator.flush()

        let ctx = try await orchestrator.recall(query: "Duplicate content")
        // Should have exactly 1 result, not 2
        #expect(ctx.items.count == 1, "Expected 1 frame, got \(ctx.items.count) — dedup failed")
        try await orchestrator.close()
    }
}

@Test func rememberDifferentContentCreatesSeparateFrames() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false

        let orchestrator = try await MemoryOrchestrator(at: url, config: config)
        try await orchestrator.remember("First content")
        try await orchestrator.remember("Second content")
        try await orchestrator.flush()

        let ctx = try await orchestrator.recall(query: "content")
        #expect(ctx.items.count >= 2, "Expected 2+ frames for different content")
        try await orchestrator.close()
    }
}
```

**Step 6: Run to verify failure**

```bash
swift test --filter DeduplicationTests
```
Expected: `rememberIdenticalContentTwiceCreatesOneFrame` FAILS (2 frames created).

**Step 7: Add `contentHash` to FrameMeta**

In `Sources/WaxCore/FileFormat/FrameMeta.swift`, add an optional `contentHash` field. This is stored in frame metadata entries, not as a new binary field, to maintain format compatibility:

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

    if let existingFrameId = try await session.findFrameByMetadata(
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

Add a lookup method to `WaxSession` that scans the TOC's frame metadata for a matching key-value pair. This uses the in-memory frame list (already loaded on open):

```swift
public func findFrameByMetadata(key: String, value: String) async throws -> UInt64? {
    let toc = await wax.currentTOC()
    for frame in toc.frames {
        if let meta = frame.metadata, meta.entries[key] == value {
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
git add Sources/WaxCore/ContentHasher.swift Sources/WaxCore/FileFormat/FrameMeta.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift Sources/Wax/WaxSession.swift \
    Tests/WaxCoreTests/ContentHasherTests.swift Tests/WaxIntegrationTests/DeduplicationTests.swift
git commit -m "feat: add frame content dedup via SHA-256 hash in remember()"
```

---

## Task 3: Store-Level Embedding Model Binding

**Why:** A user could open a `.wax` file with MiniLM (384-dim), close it, then reopen with a 768-dim model. Wax validates dimensions per-operation but not per-store. Memvid prevents this at the file level.

**Memvid reference:**
```rust
// memvid-main/src/types/embedding_identity.rs:17-23
pub struct EmbeddingIdentity {
    pub provider:   Option<Box<str>>,
    pub model:      Option<Box<str>>,
    pub dimension:  Option<u32>,
    pub normalized: Option<bool>,
}

// memvid-main/src/memvid/mutation.rs:3326-3382
// Fail-fast if incoming embeddings have mismatched dimensions
if incoming_dimension != existing_dimension {
    return Err(MemvidError::VecDimensionMismatch {
        expected: existing_dimension,
        actual: incoming_dimension,
    });
}
```

**Wax already has:** `EmbeddingIdentity` struct at `Sources/WaxVectorSearch/Embeddings/EmbeddingProvider.swift:27-44` and per-frame metadata storage at `Sources/Wax/VectorSearchSession.swift:99-108`. The TOC has a `memory_binding` placeholder slot at `Sources/WaxCore/FileFormat/WaxTOC.swift:124`.

**Files:**
- Create: `Sources/WaxCore/FileFormat/MemoryBinding.swift`
- Modify: `Sources/WaxCore/FileFormat/WaxTOC.swift` — encode/decode `MemoryBinding`
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` — validate on open
- Create: `Tests/WaxCoreTests/MemoryBindingTests.swift`
- Create: `Tests/WaxIntegrationTests/ModelBindingTests.swift`

**Step 1: Write failing test for MemoryBinding codec**

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

    #expect(decoded.embeddingProvider == "local")
    #expect(decoded.embeddingModel == "all-MiniLM-L6-v2")
    #expect(decoded.embeddingDimensions == 384)
    #expect(decoded.embeddingNormalized == true)
}

@Test func memoryBindingNilFieldsRoundTrip() throws {
    let binding = MemoryBinding(
        embeddingProvider: nil,
        embeddingModel: nil,
        embeddingDimensions: nil,
        embeddingNormalized: nil
    )
    var encoder = BinaryEncoder()
    var mutable = binding
    try mutable.encode(to: &encoder)

    var decoder = try BinaryDecoder(data: encoder.data)
    let decoded = try MemoryBinding.decode(from: &decoder)

    #expect(decoded.embeddingProvider == nil)
    #expect(decoded.embeddingDimensions == nil)
}
```

**Step 2: Run to verify failure**

```bash
swift test --filter MemoryBindingTests
```

**Step 3: Implement MemoryBinding**

```swift
// Sources/WaxCore/FileFormat/MemoryBinding.swift
import Foundation

/// Persisted embedding model identity for a .wax store.
/// Prevents mixing embeddings from different models.
///
/// Inspired by memvid's EmbeddingIdentity (memvid-main/src/types/embedding_identity.rs:17-23)
/// and dimension validation (memvid-main/src/memvid/mutation.rs:3326-3382).
public struct MemoryBinding: Equatable, Sendable {
    public var embeddingProvider: String?
    public var embeddingModel: String?
    public var embeddingDimensions: UInt32?
    public var embeddingNormalized: Bool?

    public init(
        embeddingProvider: String? = nil,
        embeddingModel: String? = nil,
        embeddingDimensions: UInt32? = nil,
        embeddingNormalized: Bool? = nil
    ) {
        self.embeddingProvider = embeddingProvider
        self.embeddingModel = embeddingModel
        self.embeddingDimensions = embeddingDimensions
        self.embeddingNormalized = embeddingNormalized
    }

    public func isCompatible(with identity: EmbeddingIdentity) -> Bool {
        if let expected = embeddingDimensions, let actual = identity.dimensions.map(UInt32.init) {
            if expected != actual { return false }
        }
        if let expected = embeddingModel, let actual = identity.model {
            if expected != actual { return false }
        }
        return true
    }
}

extension MemoryBinding: BinaryCodable {
    public mutating func encode(to encoder: inout BinaryEncoder) throws {
        try encoder.encodeOptionalString(embeddingProvider)
        try encoder.encodeOptionalString(embeddingModel)
        try encoder.encodeOptionalUInt32(embeddingDimensions)
        try encoder.encodeOptionalBool(embeddingNormalized)
    }

    public static func decode(from decoder: inout BinaryDecoder) throws -> MemoryBinding {
        let provider = try decoder.decodeOptionalString()
        let model = try decoder.decodeOptionalString()
        let dimensions = try decoder.decodeOptionalUInt32()
        let normalized = try decoder.decodeOptionalBool()
        return MemoryBinding(
            embeddingProvider: provider,
            embeddingModel: model,
            embeddingDimensions: dimensions,
            embeddingNormalized: normalized
        )
    }
}
```

**Step 4: Run unit test**

```bash
swift test --filter MemoryBindingTests
```

**Step 5: Wire into WaxTOC**

Modify `Sources/WaxCore/FileFormat/WaxTOC.swift`:

Add `memoryBinding: MemoryBinding?` property to `WaxTOC` struct (after `ticketRef`).

In encode (line 124), replace the `UInt8(0)` placeholder:
```swift
// Before: encoder.encode(UInt8(0)) // memory_binding absent in v1
// After:
try encoder.encode(memoryBinding) { encoder, value in
    var mutable = value
    try mutable.encode(to: &encoder)
}
```

In decode (line 195), replace the guard:
```swift
// Before: let memoryBindingTag = try decoder.decode(UInt8.self)
//         guard memoryBindingTag == 0 ...
// After:
let memoryBinding = try decodeOptional(MemoryBinding.self, from: &decoder)
```

**Step 6: Write integration test for model mismatch rejection**

```swift
// Tests/WaxIntegrationTests/ModelBindingTests.swift
import Foundation
import Testing
import Wax

@Test func reopeningWithDifferentEmbedderThrows() async throws {
    try await TempFiles.withTempFile { url in
        // Create store with 384-dim embedder
        let embedder384 = DeterministicTextEmbedder(
            dimensions: 384, normalize: true,
            identity: EmbeddingIdentity(provider: "test", model: "model-A", dimensions: 384, normalized: true),
            executionMode: .onDeviceOnly
        )
        var config = OrchestratorConfig.default
        config.enableVectorSearch = true

        let first = try await MemoryOrchestrator(at: url, config: config, embedder: embedder384)
        try await first.remember("test content")
        try await first.close()

        // Reopen with 768-dim embedder — should throw
        let embedder768 = DeterministicTextEmbedder(
            dimensions: 768, normalize: true,
            identity: EmbeddingIdentity(provider: "test", model: "model-B", dimensions: 768, normalized: true),
            executionMode: .onDeviceOnly
        )

        await #expect(throws: WaxError.self) {
            _ = try await MemoryOrchestrator(at: url, config: config, embedder: embedder768)
        }
    }
}

@Test func reopeningWithSameEmbedderSucceeds() async throws {
    try await TempFiles.withTempFile { url in
        let embedder = DeterministicTextEmbedder(
            dimensions: 384, normalize: true,
            identity: EmbeddingIdentity(provider: "test", model: "model-A", dimensions: 384, normalized: true),
            executionMode: .onDeviceOnly
        )
        var config = OrchestratorConfig.default
        config.enableVectorSearch = true

        let first = try await MemoryOrchestrator(at: url, config: config, embedder: embedder)
        try await first.remember("test content")
        try await first.close()

        // Reopen with same embedder — should succeed
        let second = try await MemoryOrchestrator(at: url, config: config, embedder: embedder)
        let ctx = try await second.recall(query: "test")
        #expect(!ctx.items.isEmpty)
        try await second.close()
    }
}
```

**Step 7: Implement validation in MemoryOrchestrator.init**

In `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`, in the `init` method, after opening the store:

```swift
// After: let wax = try await Wax.open(at: url) or Wax.create(at: url)
// Add:
if let embedder = embedder, let existingBinding = await wax.currentTOC().memoryBinding {
    if let identity = embedder.identity {
        guard existingBinding.isCompatible(with: identity) else {
            throw WaxError.io(
                "Embedding model mismatch: store bound to \(existingBinding.embeddingModel ?? "unknown") " +
                "(\(existingBinding.embeddingDimensions.map(String.init) ?? "?")d), " +
                "but embedder provides \(identity.model ?? "unknown") (\(identity.dimensions.map(String.init) ?? "?")d)"
            )
        }
    }
}

// On first embed, write binding to TOC:
// In remember(), after first successful embedding batch, if memoryBinding is nil:
if await wax.currentTOC().memoryBinding == nil, let identity = embedder?.identity {
    let binding = MemoryBinding(
        embeddingProvider: identity.provider,
        embeddingModel: identity.model,
        embeddingDimensions: identity.dimensions.map(UInt32.init),
        embeddingNormalized: identity.normalized
    )
    try await wax.setMemoryBinding(binding)
}
```

**Step 8: Run integration tests**

```bash
swift test --filter ModelBindingTests
```

**Step 9: Commit**

```bash
git add Sources/WaxCore/FileFormat/MemoryBinding.swift Sources/WaxCore/FileFormat/WaxTOC.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift \
    Tests/WaxCoreTests/MemoryBindingTests.swift Tests/WaxIntegrationTests/ModelBindingTests.swift
git commit -m "feat: store-level embedding model binding with mismatch prevention"
```

---

## Task 4: Version Relations (Sets/Updates/Extends)

**Why:** Wax has fact retraction (`system_to_ms` soft-delete) but no way to express "this fact updates a previous value" or "this fact extends a list." Knowledge evolution requires version semantics.

**Memvid reference:**
```rust
// memvid-main/src/types/memory_card.rs:76-86
pub enum VersionRelation {
    Sets = 0,     // First time this slot is being set
    Updates = 1,  // Replaces a previous value entirely
    Extends = 2,  // Adds to existing value
    Retracts = 3, // Negates/removes a previous value
}

// memvid-main/src/types/memory_card.rs:248-272
pub fn supersedes(&self, other: &MemoryCard) -> bool {
    match self.version_relation {
        VersionRelation::Updates | VersionRelation::Retracts => {
            let self_time = self.event_date.or(self.document_date).unwrap_or(0);
            let other_time = other.event_date.or(other.document_date).unwrap_or(0);
            self_time > other_time
        }
        VersionRelation::Sets | VersionRelation::Extends => false,
    }
}
```

**Wax already has:** `StructuredOp.retractFact` in `Sources/WaxTextSearch/FTS5SearchEngine.swift:527`, bi-temporal `sm_fact_span` table with `system_to_ms` soft-delete, and `StructuredMemoryHasher` for fact dedup.

**Files:**
- Create: `Sources/WaxCore/StructuredMemory/VersionRelation.swift`
- Modify: `Sources/WaxTextSearch/StructuredMemorySchema.swift` — add column
- Modify: `Sources/WaxTextSearch/FTS5SearchEngine.swift` — add `VersionRelation` to `assertFact`
- Modify: `Sources/WaxCLI/FactsCommand.swift` — expose `--relation` flag
- Create: `Tests/WaxIntegrationTests/VersionRelationTests.swift`

**Step 1: Write failing tests**

```swift
// Tests/WaxIntegrationTests/VersionRelationTests.swift
import Testing
@testable import WaxCore

@Test func versionRelationRawValues() {
    #expect(VersionRelation.sets.rawValue == 0)
    #expect(VersionRelation.updates.rawValue == 1)
    #expect(VersionRelation.extends.rawValue == 2)
    #expect(VersionRelation.retracts.rawValue == 3)
}

@Test func updatesSupersedes() {
    #expect(VersionRelation.updates.supersedes == true)
    #expect(VersionRelation.retracts.supersedes == true)
    #expect(VersionRelation.sets.supersedes == false)
    #expect(VersionRelation.extends.supersedes == false)
}
```

**Step 2: Run to verify failure**

```bash
swift test --filter VersionRelationTests
```

**Step 3: Implement VersionRelation**

```swift
// Sources/WaxCore/StructuredMemory/VersionRelation.swift
import Foundation

/// Semantic versioning for facts, inspired by memvid's MemoryCard version relations
/// (memvid-main/src/types/memory_card.rs:76-86).
///
/// - `sets`: First assertion of this (subject, predicate) pair. Immutable baseline.
/// - `updates`: Replaces the previous value entirely. Auto-retracts prior open spans.
/// - `extends`: Additive — appends to the existing value set (e.g., list of hobbies).
/// - `retracts`: Negates the previous value (existing behavior via `system_to_ms`).
public enum VersionRelation: UInt8, Sendable, Equatable, CaseIterable {
    case sets = 0
    case updates = 1
    case extends = 2
    case retracts = 3

    /// Whether this relation supersedes prior facts with the same (subject, predicate).
    public var supersedes: Bool {
        switch self {
        case .updates, .retracts: return true
        case .sets, .extends: return false
        }
    }
}
```

**Step 4: Run unit test**

```bash
swift test --filter VersionRelationTests
```

**Step 5: Add `version_relation` column to `sm_fact` schema**

In `Sources/WaxTextSearch/StructuredMemorySchema.swift`, add to the CREATE TABLE for `sm_fact`:

```sql
version_relation INTEGER NOT NULL DEFAULT 0
```

**Step 6: Update `assertFact` to accept VersionRelation**

In `Sources/WaxTextSearch/FTS5SearchEngine.swift`, modify the `assertFact` signature:

```swift
public func assertFact(
    subject: EntityKey,
    predicate: PredicateKey,
    object: FactValue,
    valid: StructuredTimeRange,
    system: StructuredTimeRange,
    evidence: [StructuredEvidence],
    relation: VersionRelation = .sets  // NEW parameter with backward-compatible default
) async throws -> FactRowID {
```

When `relation == .updates`, before inserting the new fact span, auto-retract all open spans for the same (subject, predicate):

```swift
if relation.supersedes {
    // Auto-retract open spans for same (subject, predicate)
    try retractOpenSpansForSubjectPredicate(
        subject: subject, predicate: predicate, atMs: system.fromMs
    )
}
```

**Step 7: Write integration test for updates superseding**

```swift
@Test func updateFactRetractsPrior() async throws {
    try await TempFiles.withTempFile { url in
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false
        let orchestrator = try await MemoryOrchestrator(at: url, config: config)

        // Assert initial fact
        try await orchestrator.session.textEngine.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("employer"),
            object: .string("Google"),
            valid: StructuredTimeRange(fromMs: 0),
            system: StructuredTimeRange(fromMs: 1000),
            evidence: [],
            relation: .sets
        )

        // Update fact
        try await orchestrator.session.textEngine.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("employer"),
            object: .string("Anthropic"),
            valid: StructuredTimeRange(fromMs: 0),
            system: StructuredTimeRange(fromMs: 2000),
            evidence: [],
            relation: .updates
        )

        try await orchestrator.session.textEngine.flushPendingStructuredOps()

        // Query current facts — should only see "Anthropic"
        let facts = try await orchestrator.session.textEngine.queryFacts(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("employer"),
            asOf: .latest
        )
        #expect(facts.count == 1)
        #expect(facts.first?.object == .string("Anthropic"))

        try await orchestrator.close()
    }
}
```

**Step 8: Run all tests**

```bash
swift test --filter VersionRelation
```

**Step 9: Update CLI `fact-assert` command**

In `Sources/WaxCLI/FactsCommand.swift`, add a `--relation` option:

```swift
@Option(name: .long, help: "Version relation: sets, updates, extends, retracts (default: sets)")
var relation: String = "sets"
```

Parse to `VersionRelation` and pass to `assertFact`.

**Step 10: Commit**

```bash
git add Sources/WaxCore/StructuredMemory/VersionRelation.swift \
    Sources/WaxTextSearch/StructuredMemorySchema.swift \
    Sources/WaxTextSearch/FTS5SearchEngine.swift \
    Sources/WaxCLI/FactsCommand.swift \
    Tests/WaxIntegrationTests/VersionRelationTests.swift
git commit -m "feat: add version relations (sets/updates/extends/retracts) to fact graph"
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
- Modify: `Sources/Wax/UnifiedSearch/SearchRequest.swift` — add temporal phrase support
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` — integrate temporal in `recall()`
- Create: `Tests/WaxTests/TemporalNormalizerTests.swift`

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

    /// Convert to milliseconds-since-epoch range for SearchRequest.TimeRange
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

In `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`, modify `recall()` to detect temporal phrases in the query and add a `TimeRange` to the `SearchRequest`:

```swift
public func recall(query: String, ...) async throws -> RAGContext {
    var request = SearchRequest(query: query, ...)

    // Temporal phrase detection: if query starts with a temporal marker, extract range
    let normalizer = TemporalNormalizer(anchor: Date())
    if let temporalRange = extractTemporalRange(from: query, normalizer: normalizer) {
        request.timeRange = SearchRequest.TimeRange(
            after: temporalRange.afterMs,
            before: temporalRange.beforeMs
        )
    }
    // ... existing recall logic ...
}

private func extractTemporalRange(
    from query: String,
    normalizer: TemporalNormalizer
) -> (afterMs: Int64, beforeMs: Int64)? {
    let temporalPrefixes = [
        "last week", "this week", "yesterday", "today", "last month",
        "this month", "next week", "next month"
    ]
    let lower = query.lowercased()
    for prefix in temporalPrefixes {
        if lower.contains(prefix) {
            if let resolution = try? normalizer.resolve(prefix) {
                return resolution.asTimeRange
            }
        }
    }
    return nil
}
```

**Step 7: Commit**

```bash
git add Sources/Wax/Temporal/ Tests/WaxTests/TemporalNormalizerTests.swift \
    Sources/Wax/Orchestrator/MemoryOrchestrator.swift
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
    var processedIds: [UInt64] = []

    await pipeline.start { task in
        processedIds.append(task.frameId)
        return EnrichmentResult(
            frameId: task.frameId,
            keywords: KeywordExtractor.extract(from: task.text),
            entities: []
        )
    }

    await pipeline.enqueue(EnrichmentTask(frameId: 1, text: "Swift concurrency is great"))
    await pipeline.enqueue(EnrichmentTask(frameId: 2, text: "Rust ownership model"))

    // Wait for processing
    try await Task.sleep(for: .milliseconds(500))
    await pipeline.stop()

    #expect(processedIds.contains(1))
    #expect(processedIds.contains(2))
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
    private var stream: AsyncStream<EnrichmentTask>?
    private var continuation: AsyncStream<EnrichmentTask>.Continuation?
    private var processingTask: Task<Void, Never>?
    private var processedCount: UInt64 = 0

    public init() {}

    public func start(
        handler: @escaping @Sendable (EnrichmentTask) async -> EnrichmentResult
    ) {
        let (stream, continuation) = AsyncStream<EnrichmentTask>.makeStream()
        self.stream = stream
        self.continuation = continuation

        processingTask = Task {
            for await task in stream {
                let _ = await handler(task)
                processedCount += 1
            }
        }
    }

    public func enqueue(_ task: EnrichmentTask) {
        continuation?.yield(task)
    }

    public func stop() {
        continuation?.finish()
        processingTask?.cancel()
        processingTask = nil
    }

    public var stats: UInt64 { processedCount }
}
```

**Step 5: Wire into MemoryOrchestrator**

In `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`, add `enrichmentPipeline` property. After each batch of frames is persisted in `remember()`, enqueue enrichment:

```swift
// After frame persist + embedding:
if let pipeline = enrichmentPipeline {
    for chunk in batchChunks {
        await pipeline.enqueue(EnrichmentTask(
            frameId: chunkFrameId,
            text: chunk.text
        ))
    }
}
```

Add pipeline lifecycle to `close()`:

```swift
public func close() async throws {
    await enrichmentPipeline?.stop()
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

**Why:** Wax scores 10/100 on security vs memvid's 78. The fix is trivial — 1 line of code — but currently Wax doesn't set `NSFileProtectionComplete` on `.wax` files, leaving them readable when the device is locked.

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
    try await TempFiles.withTempFile { url in
        let wax = try await Wax.create(at: url)
        try await wax.close()

        let attributes = try FileManager.default.attributesOfItem(atPath: url.path)
        let protection = attributes[.protectionKey] as? FileProtectionType
        #expect(protection == .complete)
    }
}
#endif

@Test func waxFileIsReadableAfterCreate() async throws {
    try await TempFiles.withTempFile { url in
        let wax = try await Wax.create(at: url)
        try await wax.close()
        #expect(FileManager.default.isReadableFile(atPath: url.path))
    }
}
```

**Step 2: Implement file protection**

In `Sources/WaxCore/Wax.swift`, after creating or opening the file, set the protection attribute:

```swift
// After file creation:
#if os(iOS) || os(watchOS) || os(tvOS) || os(visionOS)
try? FileManager.default.setAttributes(
    [.protectionKey: FileProtectionType.complete],
    ofItemAtPath: url.path
)
#endif
```

**Step 3: Run tests**

```bash
swift test --filter DataProtection
```

**Step 4: Commit**

```bash
git add Sources/WaxCore/Wax.swift Tests/WaxCoreTests/DataProtectionTests.swift
git commit -m "feat: set NSFileProtectionComplete on .wax files for iOS data protection"
```

---

## Execution Order & Dependencies

```
Task 7 (Data Protection)  ──────────────────────── Independent, trivial
Task 1 (Benchmarks)       ──────────────────────── Independent, establishes baselines
Task 2 (Content Dedup)    ──────────────────────── Independent
Task 3 (Model Binding)    ──────────────────────── Independent (uses TOC slot)
Task 4 (Version Relations) ─────────────────────── Independent (schema change)
Task 5 (Temporal NLP)     ──────────────────────── Independent
Task 6 (Enrichment)       ── depends on Task 2 ── Uses dedup in pipeline
```

**Recommended parallel grouping:**
- **Wave 1** (parallel): Tasks 7, 1, 2 — trivial/foundational
- **Wave 2** (parallel): Tasks 3, 4, 5 — independent module changes
- **Wave 3** (sequential): Task 6 — builds on dedup from Task 2

---

## Post-Implementation Scorecard (Expected)

| Dimension | Before | After | Delta |
|---|:---:|:---:|:---:|
| Ingestion Pipeline | 62 | 80 | +18 |
| Temporal Intelligence | 40 | 75 | +35 |
| Embedding System | 82 | 88 | +6 |
| Structured Memory | 78 | 85 | +7 |
| Encryption & Security | 10 | 70 | +60 |
| Benchmark Coverage | 0 | 60 | +60 |
| **Unweighted Average** | **72** | **83** | **+11** |
