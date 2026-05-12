# Wax: Deep Technical Analysis

## Executive Summary

Wax is a Swift-native, single-file memory engine for on-device AI agents. It combines SQLite FTS5 full-text search with Metal-accelerated HNSW vector search in a portable `.wax` binary format. The project targets Apple Silicon (M-series) with performance claims of 6.1ms hybrid search latency (p95) and 85.9 docs/s ingest throughput.

---

## 1. Binary File Format (.wax)

### 1.1 High-Level Layout

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Dual Header Pages (A/B) - 8 KiB total                   │
│   Page A (4KB): Magic, Version, Generation, WAL/TOC pointers, Checksums    │
│   Page B (4KB): Same structure (used for atomic updates)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                          WAL (Write-Ahead Log)                              │
│   Default: 256 MiB ring buffer                                             │
│   Ring buffer for crash-resilient uncommitted mutations                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                         Compressed Data Frames                              │
│   Frame 0 (LZ4)     Frame 1 (LZ4)     Frame 2 (LZ4)    ...                │
│   [Raw Document]    [Metadata/JSON]    [System Info]                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                          Hybrid Search Indices                              │
│   ┌─────────────────────────────┐  ┌─────────────────────────────┐         │
│   │ SQLite FTS5 Blob            │  │ Metal HNSW Index            │         │
│   │ (Text Search + EAV Facts)   │  │ (Vector Search)             │         │
│   └─────────────────────────────┘  └─────────────────────────────┘         │
├─────────────────────────────────────────────────────────────────────────────┤
│                          TOC (Table of Contents)                            │
│   Frame metadata, index manifests, segment catalog, merkle root            │
├─────────────────────────────────────────────────────────────────────────────┤
│                          Footer (64 bytes)                                  │
│   Magic: "WAX1FOOT", committed sequence, generation                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Header Page Structure (4096 bytes)

From `WaxHeaderPage.swift`:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 | magic | `0x57415831` ("WAX1") |
| 4 | 2 | format_version | Packed `((major << 8) \| minor)` |
| 6 | 1 | spec_major | Major version (currently 1) |
| 7 | 1 | spec_minor | Minor version (currently 0) |
| 8 | 8 | header_page_generation | Monotonically increasing |
| 16 | 8 | file_generation | Incremented on each commit |
| 24 | 8 | footer_offset | Offset to footer region |
| 32 | 8 | wal_offset | Offset to WAL (default: 8192) |
| 40 | 8 | wal_size | WAL ring buffer size |
| 48 | 8 | wal_write_pos | Current write position in WAL |
| 56 | 8 | wal_checkpoint_pos | Last checkpoint position |
| 64 | 8 | wal_committed_seq | Last committed sequence number |
| 72 | 32 | toc_checksum | SHA-256 of TOC |
| 104 | 32 | header_checksum | SHA-256 of header (excluding itself) |
| 136 | 8 | wal_snapshot_magic | "WALSNAP1" if snapshot present |
| 144-200 | varies | WALReplaySnapshot | Recovery state for WAL replay |

**Atomic Update Strategy:**
- Two header pages (A and B) at offsets 0 and 4096
- Each write increments `header_page_generation`
- On recovery: validate both, select the one with higher generation
- If both valid and same generation: prefer page A

### 1.3 Table of Contents (TOC)

From `WaxTOC.swift`, the TOC contains:

```swift
package struct WaxTOC {
    var tocVersion: UInt64              // Currently v1
    var frames: [FrameMeta]             // All frame metadata
    var indexes: IndexManifests         // Lex + Vec index locations
    var timeIndex: TimeIndexManifest?   // Optional temporal index
    var segmentCatalog: SegmentCatalog  // Track/role segments
    var ticketRef: TicketRef            // Concurrency ticket
    var memoryBinding: MemoryBinding?   // Provider binding info
    var merkleRoot: Data                // 32-byte Merkle root
    var tocChecksum: Data               // 32-byte SHA-256
}
```

The TOC is encoded with a custom binary format using `BinaryEncoder`/`BinaryDecoder`, ending with a 32-byte SHA-256 checksum.

### 1.4 Frame Structure

From `FrameMeta.swift`:

```swift
package struct FrameMeta {
    var id: UInt64                      // Dense, sequential ID
    var timestamp: Int64                // Creation timestamp (ms)
    var anchorTs: Int64?                // Optional anchor timestamp
    var kind: String?                   // MIME type hint
    var track: String?                  // Logical track name
    var payloadOffset: UInt64           // Offset to compressed payload
    var payloadLength: UInt64           // Compressed size
    var checksum: Data                  // SHA-256 of canonical (uncompressed) payload
    var uri: String?                    // Optional URI
    var title: String?                  // Optional title
    var canonicalEncoding: CanonicalEncoding // plain|lz4|lzfse|deflate
    var canonicalLength: UInt64?        // Uncompressed size (if compressed)
    var storedChecksum: Data?           // SHA-256 of stored (compressed) payload
    var metadata: Metadata?             // Rich metadata
    var searchText: String?             // Pre-extracted search text
    var tags: [TagPair]                 // Key-value tags
    var labels: [String]                // Category labels
    var role: FrameRole                 // document|surrogate|etc.
    var status: FrameStatus             // active|deleted|superseded
    var supersedes: UInt64?             // Version chain
    var supersededBy: UInt64?           // Version chain
}
```

---

## 2. Compression Strategy

From `PayloadCompressor.swift`, Wax supports three compression algorithms:

| Algorithm | macOS/iOS | Linux | Notes |
|-----------|-----------|-------|-------|
| LZFSE | ✅ (Apple Compression) | ❌ | Apple-optimized, good ratio |
| LZ4 | ✅ (Apple Compression) | ✅ (C interop) | Fast decompression |
| Deflate | ✅ (Apple Compression) | ✅ (C interop) | Universal, slower |

**Compression Flow:**
```
Document → canonical encoding → compressed payload → stored in frame
                                    ↓
                            checksum computed (both canonical + stored)
```

For Linux builds, Wax uses C interop (`WaxCoreCompressionC`) with linked libraries for LZ4 and deflate.

---

## 3. Write-Ahead Log (WAL) Implementation

### 3.1 Ring Buffer Architecture

From `WALRingWriter.swift`:

```swift
package final class WALRingWriter {
    let file: FDFile              // Low-level file descriptor
    let walOffset: UInt64         // Start of WAL region
    let walSize: UInt64           // Ring buffer capacity (default: 256 MiB)
    var writePos: UInt64          // Current write position (modulo walSize)
    var checkpointPos: UInt64     // Last checkpoint position
    var pendingBytes: UInt64      // Bytes since last checkpoint
    var lastSequence: UInt64      // Monotonically increasing sequence
    var wrapCount: UInt64         // Number of buffer wraps
}
```

### 3.2 WAL Record Format

WAL entries are 48-byte header + variable-length payload:

| Field | Type | Description |
|-------|------|-------------|
| sequence | UInt64 | Monotonically increasing |
| type | UInt8 | 0=data, 1=padding, 2=sentinel |
| flags | UInt8 | WAL flags (batch markers, etc.) |
| payload_length | UInt32 | Payload size in bytes |
| ... | ... | Additional header fields |

### 3.3 Crash Recovery

The WAL supports three fsync policies:

```swift
package enum WALFsyncPolicy {
    case always         // Fsync every write (safest)
    case onCommit       // Fsync only at commit (default)
    case everyBytes(n)  // Fsync after N bytes accumulated
}
```

**Recovery State Machine:**
1. Read header page A and B
2. Select page with higher `header_page_generation`
3. Check for WAL snapshot in header
4. If snapshot valid: replay from snapshot state
5. Otherwise: scan WAL from checkpoint position, replay uncommitted records
6. Validate sequence numbers, skip padding/sentinel records

**Fault Recovery on Write Failure:**
```swift
private func faultAndRestore(_ snapshot: WriterStateSnapshot) {
    restoreState(snapshot)      // Restore in-memory state
    isFaulted = true            // Mark writer as faulted
    // Write sentinel to prevent misinterpretation of partial writes
    try? writeAllCounted(Self.sentinelData, at: walOffset + snapshot.writePos)
}
```

---

## 4. Metal GPU-Accelerated Vector Search

### 4.1 HNSW Index Implementation

From `MetalANNSVectorEngine.swift`, Wax uses the MetalANNS framework:

```swift
package actor MetalANNSVectorEngine: VectorSearchEngine {
    private let metric: VectorMetric         // cosine | dot | l2
    let dimensions: Int                       // Typically 384 (MiniLM)
    private var index: VectorIndex<UInt64, VectorIndexState.Ready>?
    private var frameIds: [UInt64] = []       // ID mapping
    private var vectors: [Float] = []         // Flat vector storage
    private var positions: [UInt64: Int] = [:] // O(1) ID lookup
}
```

**HNSW Parameters:**
- Uses `IndexConfiguration.default` from MetalANNS
- Automatic index rebuild threshold: 10,000 vectors
- Metric-aware normalization for cosine similarity

### 4.2 MetalANNS Integration

The `MetalANNS` package (v0.1.3 from christopherkarani/MetalANNS) provides:

```swift
// From MetalANNSVectorEngine.swift
private func rebuildIndex() async throws {
    guard !frameIds.isEmpty else { index = nil; return }
    let builder = VectorIndex<UInt64, VectorIndexState.Unbuilt>(configuration: configuration)
    index = try await builder.build(vectors: matrixVectors(), ids: frameIds)
}
```

The framework handles:
- GPU memory allocation via Metal buffers
- Kernel dispatch for distance calculations
- Index construction with parallel graph building
- Query execution with GPU acceleration

### 4.3 Vector Serialization

From `VectorSerializer.swift`, vectors are stored in a custom binary format:

```
┌──────────────────────────────────────────────┐
│ VecSegmentHeaderV1 (36 bytes)                │
│   magic: "MV2V" (0x4D563256)                │
│   version: 1                                 │
│   encoding: 1=uSearch, 2=metal, 3=flat       │
│   similarity: 0=cosine, 1=dot, 2=l2         │
│   dimension: UInt32                          │
│   vectorCount: UInt64                        │
│   payloadLength: UInt64                      │
│   reserved: 8 bytes (zeros)                  │
├──────────────────────────────────────────────┤
│ Vector Data (float32 array, row-major)       │
├──────────────────────────────────────────────┤
│ Frame IDs (uint64 array)                     │
└──────────────────────────────────────────────┘
```

### 4.4 Performance Characteristics

From the benchmark report (2026-03-06):

| Metric | Result |
|--------|--------|
| Metal search avg (1K vectors, 128d) | 1.58 ms |
| Latency per vector | 0.0016 ms |
| Cold search with GPU sync (10K vectors, 384d) | 4.87 ms |
| Warm search avg without sync | 0.91 ms |
| Warm search speedup vs CPU | 5.4x |
| Memory bandwidth saved per warm query | 14.6 MB |

---

## 5. Embedding Model Integration (MiniLM)

### 5.1 Model Architecture

From `MiniLMEmbedder.swift` and Package.swift:

- **Model**: `all-MiniLM-L6-v2.mlmodelc` (CoreML compiled)
- **Dimensions**: 384
- **Normalization**: L2 normalized for cosine similarity
- **Tokenizer**: BERT WordPiece tokenizer (`WaxBertTokenizer`)
- **Vocabulary**: Bundled `bert_tokenizer_vocab.txt`

### 5.2 CoreML Integration

```swift
package actor MiniLMEmbedder: EmbeddingProvider, BatchEmbeddingProvider {
    private let model: MiniLMEmbeddings
    let dimensions: Int = 384
    let normalize: Bool = true
    let batchSize: Int  // Default 256, max 256

    // Compute unit support
    func isUsingANE() -> Bool {
        model.computeUnits == .all || model.computeUnits == .cpuAndNeuralEngine
    }
}
```

**Compute Unit Strategy:**
1. CLI/MCP path defaults to `cpuOnly` for determinism
2. App path can use `.all` or `.cpuAndNeuralEngine` for ANE acceleration
3. Fallback chain: ANE → GPU → CPU

### 5.3 Batch Processing

```swift
package func embed(batch texts: [String]) async throws -> [[Float]] {
    let plannedBatches = Self.planBatchSizes(for: texts.count, maxBatchSize: batchSize)
    // Splits into optimal batch sizes for CoreML
    // Single items use direct embed, batches use embedBatchCoreML
}
```

**Batch Benchmarks (32 texts, CPU-only):**

| Batch Size | Total | Per Text | Throughput |
|------------|-------|----------|------------|
| 8 | 99.9 ms | 12.49 ms | 80.1 texts/sec |
| 16 | 142.3 ms | 8.90 ms | 112.4 texts/sec |
| 32 | 220.1 ms | 6.88 ms | 145.4 texts/sec |
| 64 | 601.1 ms | 9.39 ms | 106.5 texts/sec |

**Orchestrator throughput**: 85.9 docs/sec (full hybrid indexing)

### 5.4 Prewarming

```swift
package func prewarm(batchSize: Int = 16) async throws {
    _ = try await embed(" ")                    // 32-token bucket
    _ = try await embed("token " * 30)          // 64-token bucket
    _ = try await embed("token " * 60)          // 128-token bucket
    if batchSize > 1 {
        _ = try await embed(batch: ["token " * 12] * batchSize)  // Batch bucket
    }
}
```

---

## 6. FTS5 Text Search

### 6.1 Schema

From `FTS5Schema.swift`, the text search uses SQLite FTS5:

```sql
CREATE VIRTUAL TABLE frames_fts USING fts5(
    content='frame_mapping',    -- External content table
    content_rowid='rowid_ref'   -- Row ID mapping
);

CREATE TABLE frame_mapping (
    frame_id INTEGER PRIMARY KEY,
    rowid_ref INTEGER NOT NULL
);
```

### 6.2 BM25 Ranking

From `FTS5SearchEngine.swift`:

```swift
private static func scoreFromBM25Rank(_ rank: Double) -> Double {
    // SQLite FTS5 bm25() rank is "lower is better" (often negative)
    // Convert to "higher is better"
    guard rank.isFinite else { return 0 }
    return -rank
}
```

### 6.3 Batch Operations

```swift
// Flush threshold: 2048 ops before forcing SQLite write
private static let flushThreshold = 2_048

package func indexBatch(frameIds: [UInt64], texts: [String]) async throws {
    // Enqueue all operations
    // Flush when threshold exceeded
    // Single transaction for all ops
}
```

---

## 7. Hybrid Search (Text + Vector Fusion)

### 7.1 Reciprocal Rank Fusion (RRF)

From `HybridSearch.swift`:

```swift
package static func rrfFusion(
    textResults: [(UInt64, Float)],
    vectorResults: [(UInt64, Float)],
    k: Int = 60,
    alpha: Float = 0.5    // Weight balance: 0.5 = equal weight
) -> [(UInt64, Float)] {
    // For each result at rank r in list:
    //   score = weight / (k + r + 1)
    // Final score = sum of weighted RRF scores
    // Tie-break: best rank, then frameId
}
```

**Formula:**
```
RRF(d) = Σ (weight_i / (k + rank_i(d) + 1))
```

Where:
- `k = 60` (standard RRF constant)
- `alpha` controls text vs vector weight (default 0.5)
- `weight = alpha` for text, `1 - alpha` for vector

### 7.2 Adaptive Fusion

The `AdaptiveFusionConfig` allows dynamic alpha adjustment based on query characteristics:

```swift
// From AdaptiveFusionConfig.swift
// Rule-based query classification → alpha tuning
// E.g., "what is X" → higher vector weight
//       "find the text about Y" → higher text weight
```

---

## 8. Structured Memory (EAV Model)

### 8.1 Schema

The FTS5 engine also manages structured memory tables:

```sql
-- Entity-Attribute-Value tables
CREATE TABLE sm_entity (
    entity_id INTEGER PRIMARY KEY,
    key TEXT UNIQUE NOT NULL,
    kind TEXT,
    created_at_ms INTEGER
);

CREATE TABLE sm_entity_alias (
    entity_id INTEGER,
    alias TEXT,
    alias_norm TEXT,
    created_at_ms INTEGER
);

CREATE TABLE sm_predicate (
    predicate_id INTEGER PRIMARY KEY,
    key TEXT UNIQUE NOT NULL,
    created_at_ms INTEGER
);

CREATE TABLE sm_fact (
    fact_id INTEGER PRIMARY KEY,
    subject_entity_id INTEGER,
    predicate_id INTEGER,
    object_kind INTEGER,    -- 1=string, 2=int, 3=double, 4=bool, 5=blob, 6=time, 7=entity
    object_text TEXT,
    object_int INTEGER,
    object_real REAL,
    object_bool INTEGER,
    object_blob BLOB,
    object_time_ms INTEGER,
    object_entity_id INTEGER,
    version_relation INTEGER,
    fact_hash TEXT UNIQUE,
    created_at_ms INTEGER
);

-- Temporal validity
CREATE TABLE sm_fact_span (
    span_id INTEGER PRIMARY KEY,
    fact_id INTEGER,
    valid_from_ms INTEGER,
    valid_to_ms INTEGER,      -- NULL = open-ended
    system_from_ms INTEGER,
    system_to_ms INTEGER,     -- NULL = open-ended
    span_key_hash TEXT UNIQUE
);

-- Evidence linking
CREATE TABLE sm_evidence (
    evidence_id INTEGER PRIMARY KEY,
    span_id INTEGER,
    fact_id INTEGER,
    source_frame_id INTEGER,
    confidence REAL,
    asserted_at_ms INTEGER
);
```

### 8.2 Temporal Reasoning

Facts support two time dimensions:
- **Valid time**: When the fact is true in the real world
- **System time**: When the fact was known/asserted in the system

```swift
// Query facts "as of" a specific time
package func facts(
    about subject: EntityKey?,
    predicate: PredicateKey?,
    asOf: StructuredMemoryAsOf,  // systemTimeMs + validTimeMs
    limit: Int
) async throws -> StructuredFactsResult
```

---

## 9. Memory API Architecture

### 9.1 Main Entry Point

From `Memory.swift`:

```swift
public actor Memory {
    private let orchestrator: MemoryOrchestrator

    public init(at url: URL, config: Config = .default) async throws {
        self.orchestrator = try await MemoryOrchestrator(at: url, config: ...)
    }

    // Core operations
    public func save(_ text: String, metadata: [String: String] = [:]) async throws
    public func search(_ query: String, options: SearchOptions = .default) async throws -> Results
    public func flush() async throws
    public func close() async throws
}
```

### 9.2 Configuration

```swift
public struct Config: Sendable, Equatable {
    var enableTextSearch: Bool = true
    var enableVectorSearch: Bool = true
    var enableStructuredMemory: Bool = false
    var enableAccessStatsScoring: Bool = false
    var ingestConcurrency: Int = 1
    var ingestBatchSize: Int = 32
    var requireOnDeviceProviders: Bool = true
}
```

### 9.3 Concurrency Model

Wax uses Swift actors extensively:
- `Memory` - public API actor
- `MemoryOrchestrator` - internal orchestration
- `FTS5SearchEngine` - text search actor
- `MetalANNSVectorEngine` - vector search actor
- `MiniLMEmbedder` - embedding actor

All actor boundaries are designed for `Sendable` compliance with strict concurrency checking enabled.

---

## 10. Performance Benchmarks (2026-03-06)

### 10.1 Headline Numbers

| Metric | Before Optimization | After | Improvement |
|--------|---------------------|-------|-------------|
| Cold open p95 | 2.65 s | 9.2 ms | 288x faster |
| Warm hybrid p95 | 43.9 ms | 6.1 ms | 7.2x faster |
| MemoryOrchestrator ingest | 2.001 s | 0.339 s | 5.9x faster |
| Text-only ingest | 0.320 s | 0.082 s | 3.9x faster |
| WAL commit p95 (10K hybrid) | 197 ms | 34.25 ms | 5.75x faster |

### 10.2 Search Latency Breakdown

| Mode | mean | p50 | p95 | p99 |
|------|------|-----|-----|-----|
| Hybrid warm (with previews) | 5.6 ms | 5.5 ms | 6.1 ms | 6.5 ms |
| Hybrid warm (without previews) | 5.7 ms | 5.5 ms | 7.2 ms | 7.4 ms |
| Hybrid warm (CPU-only) | 5.3 ms | 5.2 ms | 5.7 ms | 5.7 ms |
| Cold open | 8.8 ms | 8.8 ms | 9.2 ms | 9.2 ms |

### 10.3 Metal Vector Engine Performance

| Benchmark | Result |
|-----------|--------|
| Metal search (1K vectors, 128d) | 1.58 ms |
| Latency per vector | 0.0016 ms |
| Cold search with GPU sync (10K, 384d) | 4.87 ms |
| Warm search without sync | 0.91 ms |
| Speedup vs CPU | 5.4x |

### 10.4 WAL Compaction Matrix

| Workload | Writes | Mode | commit p95 | reopen p95 |
|----------|--------|------|------------|------------|
| small_text | 500 | text | 11.94 ms | 2.41 ms |
| small_hybrid | 500 | hybrid | 10.63 ms | 4.39 ms |
| medium_text | 5,000 | text | 14.81 ms | 22.77 ms |
| medium_hybrid | 5,000 | hybrid | 18.29 ms | 42.05 ms |
| large_text_10k | 10,000 | text | 23.04 ms | 45.07 ms |
| large_hybrid_10k | 10,000 | hybrid | 34.25 ms | 83.17 ms |

### 10.5 Hardware Configuration

- **Platform**: macOS, Apple Silicon
- **Test machine**: M3 Max (implied by 85.9 docs/s throughput)
- **Branch**: `feat/wax-v2-improvements`
- **Benchmark commits**: `3ff3246e` (main), `bd65ceae` (MiniLM fix)

---

## 11. Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| MetalANNS | 0.1.3 | GPU-accelerated HNSW |
| USearch | 2.24.0 | CPU vector index (fallback) |
| GRDB.swift | 7.0.0 | SQLite FTS5 wrapper |
| swift-crypto | 3.7.0 | SHA-256 checksums |
| swift-sdk (MCP) | 0.10.0 | MCP server protocol |
| swift-argument-parser | 1.3.0 | CLI tool |

---

## 12. Design Principles

### 12.1 Atomicity
- Dual-header A/B pages for crash-safe header updates
- WAL ring buffer for uncommitted mutations
- SHA-256 checksums on headers, TOC, frames, and vectors
- Merkle root for integrity verification

### 12.2 Performance
- Apple Silicon native (Metal, ANE, Accelerate)
- LZ4 compression for fast decompression
- Batch operations throughout (embedding, indexing, WAL writes)
- Actor isolation for thread-safe concurrent access

### 12.3 Portability
- Single `.wax` file = complete memory store
- Works with any sync layer (iCloud, AirDrop, Git)
- No external database or server required
- Cross-platform (macOS, iOS, Linux)

### 12.4 Privacy
- 100% on-device processing
- No network calls during inference
- CoreML models run locally (CPU/GPU/ANE)

---

## 13. Comparison with Alternatives

| Feature | Wax | SQLite FTS5 | Cloud Vector DB |
|---------|-----|-------------|-----------------|
| Search type | Hybrid (text + vector) | Text only | Vector only |
| Latency (p95) | 6.1 ms | ~12 ms | 150-500+ ms |
| Privacy | 100% local | 100% local | Cloud-hosted |
| Setup | Zero config | Low | Complex (API keys) |
| Architecture | Apple Silicon native | Generic | Varies |
| Storage | Single file | Single file | Distributed |

---

## 14. Future Considerations

Based on codebase signals:
1. **Arctic Embeddings**: Alternative to MiniLM (Snowflake Arctic Embed Small)
2. **Foundation Models integration**: iOS 26+ `@Generable` for structured output
3. **VideoRAG/PhotoRAG**: Multimodal memory support
4. **Maintenance/Surrogates**: Automatic memory consolidation
5. **Enrichment pipeline**: Post-ingest processing (keyword extraction, etc.)

---

## Appendix: Key Code Locations

| Component | File Path |
|-----------|-----------|
| Header page | `Sources/WaxCore/FileFormat/WaxHeaderPage.swift` |
| TOC | `Sources/WaxCore/FileFormat/WaxTOC.swift` |
| Frame metadata | `Sources/WaxCore/FileFormat/FrameMeta.swift` |
| WAL writer | `Sources/WaxCore/WAL/WALRingWriter.swift` |
| Compression | `Sources/WaxCore/Compression/PayloadCompressor.swift` |
| Metal vector engine | `Sources/WaxVectorSearch/MetalANNSVectorEngine.swift` |
| Vector serialization | `Sources/WaxVectorSearch/VectorSerializer.swift` |
| MiniLM embedder | `Sources/WaxVectorSearchMiniLM/MiniLMEmbedder.swift` |
| FTS5 search | `Sources/WaxTextSearch/FTS5SearchEngine.swift` |
| Hybrid search | `Sources/Wax/UnifiedSearch/HybridSearch.swift` |
| Memory API | `Sources/Wax/Memory.swift` |
| Constants | `Sources/WaxCore/Constants.swift` |
| Benchmark results | `Resources/docs/benchmarks/2026-03-06-performance-results.md` |
