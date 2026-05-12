# Thinking About Memory for On-Device AI Agents

*A look at why single-file storage makes sense for local inference, and the engineering decisions behind one approach to the problem*

---

## Where We Started

When we began working on Wax, the observation was straightforward. Most AI agent memory systems rely on cloud infrastructure. You send queries to Pinecone for vectors, Elasticsearch for text, and somewhere else for document storage. Each service has its own authentication, latency profile, and failure modes.

This architecture works fine for many use cases. But for agents running on Apple devices, doing on-device inference, it introduces an awkward dependency. Your compute is local. Your models are local. But your memory is remote.

We wanted to explore what happens when memory stays co-located with the agent. Not as a hard rule against cloud systems, but as a first-class option for scenarios where latency, privacy, or offline capability matters.

## The Single-File Question

The decision to use a single file as the storage container was not obvious to us at first. There are reasonable arguments against it. File-based storage can create concurrency challenges. Large files can become unwieldy. Recovery from corruption is harder than restarting a database server.

But there are also practical advantages worth considering.

A single file is atomic in a way that distributed systems are not. You can back it up with a simple copy. You can transfer it with AirDrop or sync it with iCloud without worrying about consistency between separate services. You can delete it and know that everything is gone. These operations sound simple, but they matter when you are building applications that need to work reliably on user devices without server infrastructure.

For agents, there is another consideration. Memory is context. If your agent's memory lives in a database that requires network access, you have implicitly created a dependency on connectivity. On-device agents should be able to function when the network is unavailable. A local file enables this naturally.

### What Goes Into the File

The .wax format is a binary container with several regions:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Dual Header Pages (A/B) - 8 KiB                      │
│   Magic "WAX1", version, generation counter, WAL/TOC pointers           │
├──────────────────────────────────────────────────────────────────────────┤
│                          WAL (256 MiB ring buffer)                       │
│   Crash-resilient uncommitted mutations with padding records             │
├──────────────────────────────────────────────────────────────────────────┤
│                         Compressed Data Frames                           │
│   LZ4/LZFSE compressed documents with SHA-256 checksums                 │
├──────────────────────────────────────────────────────────────────────────┤
│                          Hybrid Search Indices                           │
│   SQLite FTS5 (text) + Metal HNSW (vectors)                             │
├──────────────────────────────────────────────────────────────────────────┤
│                     TOC (Table of Contents) + Footer                     │
│   Frame manifest, index locations, Merkle root                          │
└──────────────────────────────────────────────────────────────────────────┘
```

The format has evolved through several iterations. The current design reflects lessons from earlier versions where recovery was unreliable and indexing was slow. What follows is a walk through the major components and the reasoning behind them.

## Atomic Updates: The Dual Header Approach

Header corruption is a common failure mode in file-based storage. If the header is wrong, the entire file becomes unreadable.

Our approach uses two header pages, labeled A and B, each 4KB. Every time the header is updated, we write to the alternate page and increment a generation counter. On startup, we read both pages and use whichever has the higher generation.

This is not a novel idea. It appears in various forms across database systems and file formats. But it is effective. The implementation is a few hundred lines of Swift, and it eliminates most corruption scenarios without requiring complex rollback logic or fsync storms.

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

## Write-Ahead Logging and the Ring Buffer

The WAL (Write-Ahead Log) handles uncommitted mutations. Writes go to the WAL first, then get incorporated into the main data region during compaction.

We use a ring buffer with a default size of 256 MiB. The ring buffer approach means the WAL has a fixed maximum size, which prevents unbounded growth. The tradeoff is handling wraparound correctly.

When the write position reaches the end of the buffer, we write a padding record that signals the continuation point. Recovery scans through the buffer, skipping padding and sentinel records, and replays valid data entries.

```swift
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

The WAL also supports state snapshots. Instead of replaying the entire log on recovery, we can start from a checkpoint stored in the header. This reduces recovery time significantly for files that have been running for a while.

## Compression and Integrity

Frames in the data region use LZ4 compression by default on Apple platforms. LZFSE is available as an option with better compression ratios but higher CPU cost. On Linux, we use C interop with system libraries for both LZ4 and Deflate.

Each compressed frame stores two checksums:

- The SHA-256 of the original uncompressed data
- The SHA-256 of the compressed bytes

This lets us verify integrity without decompressing first, which is useful for validation passes and debugging.

## Search Architecture: Hybrid Text and Vectors

The search system combines two engines:

1. **SQLite FTS5** for full-text search with BM25 ranking
2. **Metal-accelerated HNSW** for vector similarity search

Results from both engines are fused using Reciprocal Rank Fusion (RRF):

```
RRF(d) = Σ (weight_i / (k + rank_i(d) + 1))
```

With k = 60 (the standard constant) and alpha = 0.5 by default (equal weighting). Alpha is adjustable per query, so if you know a particular query benefits more from semantic matching, you can weight the vector results higher.

### Text Search

The FTS5 implementation lives inside the .wax file as a SQLite blob. On open, it gets deserialized to a temp directory. We batch indexing operations and flush every 2048 operations to keep SQLite transaction overhead reasonable.

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

### Vector Search

Vectors use the MetalANNS framework for GPU-accelerated HNSW graphs on Apple Silicon. The current configuration uses 384-dimensional embeddings from the all-MiniLM-L6-v2 model with cosine similarity.

The vector index serializes as a flat float32 array plus a frame ID mapping. We store this in a custom binary format (MV2V) alongside the FTS5 blob in the search indices region.

From our benchmark on an M3 Max:

| Metric | Result |
|--------|--------|
| Search (1K vectors, 128d) | 1.58 ms |
| Cold search with GPU sync (10K, 384d) | 4.87 ms |
| Warm search without sync | 0.91 ms |
| Speedup vs CPU | 5.4x |

The GPU acceleration matters most for warm queries where the index is already resident. Cold searches include a GPU synchronization overhead that narrows the gap with CPU execution.

## Embeddings

The MiniLM embedder uses CoreML with the all-MiniLM-L6-v2 model:

```swift
package actor MiniLMEmbedder: EmbeddingProvider, BatchEmbeddingProvider {
    let dimensions: Int = 384
    let normalize: Bool = true
    let batchSize: Int  // Default 256

    private let model: MiniLMEmbeddings  // CoreML model
}
```

The compute unit strategy varies by context:

- CLI and MCP server paths default to CPU-only for determinism
- App paths can use the Neural Engine or GPU for faster inference
- There is a fallback chain from ANE to GPU to CPU

Batch processing is important for throughput. The orchestrator achieves around 86 documents per second with full hybrid indexing on an M3 Max:

| Batch Size | Total Time | Per Text | Throughput |
|------------|------------|----------|------------|
| 8 | 99.9 ms | 12.49 ms | 80.1 texts/s |
| 16 | 142.3 ms | 8.90 ms | 112.4 texts/s |
| 32 | 220.1 ms | 6.88 ms | 145.4 texts/s |
| 64 | 601.1 ms | 9.39 ms | 106.5 texts/s |

The throughput peaks around batch size 32, likely due to CoreML scheduling characteristics on the tested hardware.

## Structured Memory

Beyond unstructured search, Wax includes an Entity-Attribute-Value (EAV) model for storing durable facts. This lives in the same SQLite instance as the FTS5 index but uses separate tables.

The notable design choice is dual time dimensions on facts:

- **Valid time**: When the fact is true in reality
- **System time**: When the system learned the fact

This distinction matters for agents that need to reason about what they knew at a particular point in time. If a user's preferences change, the old facts remain queryable by timestamp even though they are no longer current.

```swift
// Store an entity
await memory.upsertEntity(key: "user:123", kind: "person", aliases: ["Alice"])

// Assert a fact with temporal validity
await memory.assertFact(
    subject: "user:123",
    predicate: "prefers",
    object: .string("dark mode"),
    valid: .init(fromMs: now, toMs: nil),
    system: .init(fromMs: now, toMs: nil)
)
```

## Performance Characteristics

From our March 2026 benchmark suite on M3 Max:

| Metric | Result | Before Optimization |
|--------|--------|---------------------|
| Cold open p95 | 9.2 ms | 2.65 s |
| Hybrid search p95 | 6.1 ms | 43.9 ms |
| Ingest (text-only) | 82 ms | 320 ms |
| MemoryOrchestrator | 339 ms | 2.001 s |
| WAL commit p95 | 34.25 ms | 197 ms |

The cold open improvement was the most significant optimization. The original implementation had unnecessary synchronous work during initialization. Restructuring the startup sequence to defer non-critical operations brought this from seconds to milliseconds.

## The MCP Server and CLI Tool

For integration with AI coding assistants like Claude Code, Wax provides two tools: an MCP server and a CLI.

### MCP Server

The MCP (Model Context Protocol) server exposes Wax operations as tools that Claude Code can invoke directly. When connected, the agent can save memories, search for context, manage entities and facts, and perform session handoffs without leaving the conversation.

The server communicates over stdio and supports a set of tool calls:

| Tool | Purpose |
|------|---------|
| `wax_remember` | Store a memory with optional metadata |
| `wax_recall` | Retrieve context assembled for a query |
| `wax_search` | Raw ranked search with hybrid mode support |
| `wax_session_start` | Begin a tracked session |
| `wax_handoff` | Save context for the next session |
| `wax_entity_upsert` | Create or update an entity |
| `wax_fact_assert` | Assert a structured fact |

The server supports cross-session retrieval through `wax_corpus_search`, which can query across multiple session files in the `~/.wax/sessions` directory. This is useful when an agent needs to reference work from a previous conversation.

Installation is straightforward for Claude Code users:

```bash
npx -y waxmcp@latest mcp install --scope user
```

This stages the Wax runtime locally and registers the server with Claude Code. After installation, the server starts automatically when needed.

### CLI Tool

The CLI provides the same operations from the command line or from scripts. It is built with Swift Argument Parser and supports subcommands for all memory operations:

```bash
# Store a memory
wax-cli remember "The project uses SwiftUI for the UI layer"

# Search with hybrid mode
wax-cli search "What UI framework does the project use?" --mode hybrid

# Check store health
wax-cli stats --store-path ~/.wax/memory.wax
```

The CLI can also run as a persistent daemon, which avoids the overhead of loading the embedder model on each invocation:

```bash
wax-cli daemon --store-path ~/.wax/memory.wax
```

Once the daemon is running, commands can be sent as JSON lines over stdin. This mode is useful for CI pipelines or scripting scenarios where the store is accessed repeatedly.

### Compute Considerations

Both the MCP server and CLI default to CPU-only inference for the embedding model. This is intentional. GPU and Neural Engine access can introduce variability in timing and resource consumption. For agent workflows where predictability matters, CPU-only mode provides consistent behavior.

The tradeoff is throughput. CPU-only embedding is slower than GPU-accelerated embedding, but for the typical interaction patterns of an AI assistant, it is usually sufficient.

## On-Device vs Cloud: A Practical View

We are not making the case that on-device memory is universally better than cloud-based systems. Each approach has genuine advantages.

Cloud vector databases handle scale differently. They replicate data, distribute load, and provide operational features that a local file cannot match. If your agent needs to persist memory across devices or share context between multiple agents, a cloud system is the appropriate choice.

On-device storage has different strengths. Lower latency for local operations. No dependency on network availability. Data stays on the user's device. No per-query API costs. Simpler deployment for applications that do not need distributed state.

The position we have taken with Wax is that on-device memory should be a viable option, not a compromise. If you are building an agent that runs on Apple hardware and does not require distributed state, a single local file should work well. The benchmarks suggest the performance is there, and the portability of a single file is genuinely useful for applications that ship to end users.

## What We Are Working On

A few areas of active development:

- **Alternative embedding models**: We have been experimenting with Snowflake Arctic Embed Small as an alternative to MiniLM. Different models have different tradeoffs in quality, speed, and memory usage.
- **Multimodal support**: Video and photo memory through dedicated orchestrators. This is still in early stages.
- **iOS 26 integration**: The Foundation Models framework and `@Generable` macro may simplify structured output generation.
- **Memory maintenance**: Automatic consolidation of old memories and generation of summary records for long-running sessions.

---

Wax is open source under Apache 2.0. The source is available on [GitHub](https://github.com/christopherkarani/Wax).

Requires Swift 6.1 or later. Targets iOS 18+ and macOS 15+.
