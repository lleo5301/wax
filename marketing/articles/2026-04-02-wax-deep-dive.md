# Wax: Building a Single-File Memory Engine for On-Device AI Agents

*How we packed SQLite FTS5, Metal HNSW, and a crash-resilient WAL into one portable binary*

---

## The Problem

AI agents need memory. Not just context windows—persistent, searchable memory that survives sessions.

Today's approach: send everything to the cloud. Query Pinecone for vectors. Query Elasticsearch for text. Hope the network doesn't flap.

For chatbots, fine. For agents running hundreds of queries per minute? It's a bottleneck.

We wanted something different: a memory engine that runs entirely on-device, stays fast at scale, and fits in a single portable file.

## The Architecture

Wax is a Swift-native persistence engine. It stores documents, embeddings, and structured knowledge in a `.wax` file.

The file format has five regions:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Dual Header Pages (A/B) - 8 KiB                     │
│   Magic "WAX1", version, generation counter, WAL/TOC pointers          │
├─────────────────────────────────────────────────────────────────────────┤
│                          WAL (256 MiB ring buffer)                      │
│   Crash-resilient uncommitted mutations with padding records            │
├─────────────────────────────────────────────────────────────────────────┤
│                         Compressed Data Frames                          │
│   LZ4/LZFSE compressed documents with SHA-256 checksums                │
├─────────────────────────────────────────────────────────────────────────┤
│                          Hybrid Search Indices                          │
│   SQLite FTS5 (text) + Metal HNSW (vectors)                            │
├─────────────────────────────────────────────────────────────────────────┤
│                     TOC (Table of Contents) + Footer                    │
│   Frame manifest, index locations, Merkle root                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Dual Headers for Atomic Updates

The header region contains two 4KB pages (A and B). Each stores:

- Magic bytes (`0x57415831`)
- Format version (packed major/minor)
- Generation counter (monotonically increasing)
- Pointers to WAL and TOC
- SHA-256 checksums

On every commit, we increment the generation counter and write to the *other* header page. On crash recovery, we read both pages and select the one with the higher generation.

No complex rollback logic. No fsync storms. Just pick the newer header.

```swift
package static func selectValidPage(pageA: Data, pageB: Data) -> (page: WaxHeaderPage, pageIndex: Int)? {
    let a = try? WaxHeaderPage.decodeWithChecksumValidation(from: pageA)
    let b = try? WaxHeaderPage.decodeWithChecksumValidation(from: pageB)

    switch (a, b) {
    case (let aPage?, let bPage?):
        if aPage.headerPageGeneration >= bPage.headerPageGeneration {
            return (aPage, 0)
        }
        return (bPage, 1)
    // ... handle nil cases
    }
}
```

### The WAL Ring Buffer

The Write-Ahead Log is a 256 MiB ring buffer. Mutations go here first, then get committed to the main data region.

The tricky part: wraparound. When the write position reaches the end of the buffer, we need to handle padding records and sentinel bytes for corruption detection.

```swift
// Simplified ring buffer write
private func append(payload: Data) throws -> UInt64 {
    let entrySize = headerSize + payload.count

    // Handle wraparound with padding
    if walSize - writePos < entrySize {
        let padding = WALRecord.padding(sequence: lastSequence + 1,
                                        skipBytes: walSize - writePos - headerSize)
        try file.writeAll(padding.encode(), at: walOffset + writePos)
        writePos = 0
    }

    // Write actual record
    let record = WALRecord.data(sequence: lastSequence + 1, payload: payload)
    try file.writeAll(record.encode(), at: walOffset + writePos)
    writePos += entrySize

    return lastSequence
}
```

The ring buffer also supports state snapshots for fast recovery. Instead of replaying the entire WAL, we can start from a known-good checkpoint stored in the header.

### Compression Strategy

Frames use platform-appropriate compression:

- **macOS/iOS**: Apple's Compression framework (LZFSE, LZ4, or Deflate)
- **Linux**: C interop with system libraries

LZ4 is the default for hot data—it decompresses at ~GB/s. LZFSE gives better ratios but costs more CPU.

Every compressed frame stores both:
- `canonical_checksum`: SHA-256 of the uncompressed payload
- `stored_checksum`: SHA-256 of the compressed payload

This lets us verify integrity without decompressing first.

## Hybrid Search

The core innovation: one query fans out to two search engines, then fuses the results.

### Text Search (SQLite FTS5)

SQLite FTS5 handles full-text search with BM25 ranking. The FTS5 database lives as a blob inside the `.wax` file and gets deserialized into a temp directory on open.

```swift
package func search(query: String, topK: Int) async throws -> [TextSearchResult] {
    let sql = """
        SELECT m.frame_id AS frame_id,
               bm25(frames_fts) AS rank,
               snippet(frames_fts, 0, '[', ']', '...', 10) AS snippet
        FROM frames_fts
        JOIN frame_mapping m ON m.rowid_ref = frames_fts.rowid
        WHERE frames_fts MATCH ?
        ORDER BY rank ASC, m.frame_id ASC
        LIMIT ?
        """
    // ...
}
```

Batch indexing collapses up to 2048 operations into a single SQLite transaction. This is critical for ingest throughput.

### Vector Search (Metal HNSW)

Vectors use the MetalANNS framework for GPU-accelerated HNSW (Hierarchical Navigable Small World) graphs.

Key numbers:
- **384 dimensions** (all-MiniLM-L6-v2)
- **Cosine similarity** with L2-normalized vectors
- **5.4x speedup** over CPU for warm queries
- **1.58ms** to search 1K vectors

The vector index serializes as a flat float32 array plus a frame ID mapping, stored in a custom `MV2V` binary format.

### Reciprocal Rank Fusion

Results combine using Reciprocal Rank Fusion:

```
RRF(d) = Σ (weight_i / (k + rank_i(d) + 1))
```

Where:
- `k = 60` (standard constant)
- `weight = alpha` for text results
- `weight = 1 - alpha` for vector results

Default `alpha = 0.5` (equal weight). Tunable per query.

```swift
package static func rrfFusion(
    textResults: [(UInt64, Float)],
    vectorResults: [(UInt64, Float)],
    k: Int = 60,
    alpha: Float = 0.5
) -> [(UInt64, Float)] {
    // Score each result by weighted rank position
    // Tie-break on best rank, then frameId
}
```

## Embeddings

The MiniLM embedder uses CoreML with batch processing:

```swift
package actor MiniLMEmbedder: EmbeddingProvider, BatchEmbeddingProvider {
    let dimensions: Int = 384
    let normalize: Bool = true
    let batchSize: Int  // Default 256

    private let model: MiniLMEmbeddings  // CoreML model
}
```

**Compute unit strategy:**
1. CLI/MCP: CPU-only for determinism
2. App: `.all` or `.cpuAndNeuralEngine` for ANE acceleration
3. Automatic fallback chain: ANE → GPU → CPU

**Throughput benchmarks (CPU-only):**

| Batch Size | Total | Per Text | Throughput |
|------------|-------|----------|------------|
| 8 | 99.9 ms | 12.49 ms | 80.1 texts/s |
| 16 | 142.3 ms | 8.90 ms | 112.4 texts/s |
| 32 | 220.1 ms | 6.88 ms | 145.4 texts/s |
| 64 | 601.1 ms | 9.39 ms | 106.5 texts/s |

Orchestrator-level throughput: **85.9 documents/sec** with full hybrid indexing.

## Structured Memory

Beyond unstructured search, Wax supports Entity-Attribute-Value (EAV) storage for durable facts.

```swift
// Store an entity
await memory.upsertEntity(key: "user:123", kind: "person", aliases: ["Alice"])

// Assert a fact with temporal validity
await memory.assertFact(
    subject: "user:123",
    predicate: "prefers",
    object: .string("dark mode"),
    valid: .init(fromMs: now, toMs: nil),  // Still true
    system: .init(fromMs: now, toMs: nil)  // Known since now
)
```

Facts have two time dimensions:
- **Valid time**: When the fact is true in reality
- **System time**: When the system learned the fact

This enables temporal queries: "What did the agent know about the user at time T?"

## Performance Numbers

From our March 2026 benchmark suite (M3 Max):

| Metric | Result | Baseline | Improvement |
|--------|--------|----------|-------------|
| Cold open p95 | 9.2 ms | 2.65 s | 288x faster |
| Hybrid search p95 | 6.1 ms | 43.9 ms | 7.2x faster |
| Ingest (text-only) | 82 ms | 320 ms | 3.9x faster |
| MemoryOrchestrator | 339 ms | 2.001 s | 5.9x faster |
| WAL commit p95 | 34.25 ms | 197 ms | 5.75x faster |

**Metal vector engine:**

| Benchmark | Result |
|-----------|--------|
| Search (1K vectors, 128d) | 1.58 ms |
| Per-vector latency | 0.0016 ms |
| Cold search (10K, 384d) | 4.87 ms |
| Warm search | 0.91 ms |
| Speedup vs CPU | 5.4x |

## Why a Single File?

Most RAG setups need:
- A vector database (Pinecone, Weaviate, Qdrant)
- A text database (Elasticsearch, Typesense)
- A document store (S3, local files)
- Orchestration glue

Wax bundles everything into one binary. Benefits:

1. **Zero setup**: No Docker stack, no database to babysit
2. **Portable**: Move the file with AirDrop, iCloud, or Git
3. **Atomic**: Backup, copy, or delete one file
4. **Private**: 100% on-device, no network calls

For on-device AI agents, this matters. Your memory lives where your agent lives.

## What's Next

- **Arctic Embeddings**: Alternative to MiniLM (Snowflake Arctic Embed Small)
- **Multimodal RAG**: Video and photo memory via VideoRAGOrchestrator
- **Foundation Models**: iOS 26+ integration with @Generable
- **Maintenance**: Automatic memory consolidation and surrogate generation

---

*Wax is open source under Apache 2.0. [GitHub](https://github.com/christopherkarani/Wax)*

*Swift 6.1+, iOS 18+, macOS 15+*
