# Reddit Post — r/swift — 2026-04-02

## Title

I built a single-file memory engine for AI agents — SQLite FTS5 + Metal HNSW in one portable binary. Benchmarks inside.

## Post

After months of work, I'm releasing [Wax](https://github.com/christopherkarani/Wax) — a Swift-native persistence engine for on-device AI agents.

**The problem I was solving:** Every agent memory setup I saw needed a cloud vector DB, a cloud text DB, and a document store. Three services for "remember this." For on-device agents, that's absurd.

**The solution:** Pack everything into one `.wax` file. Documents, embeddings, text index, vector index, crash-resilient WAL. Single binary.

### Architecture

```
Dual Header (A/B) → WAL (256MB ring) → Compressed Frames → Hybrid Indices → TOC
```

- **Dual headers** for atomic updates (pick the one with higher generation counter)
- **WAL ring buffer** with padding records for crash recovery
- **LZ4/LZFSE compressed** frames with SHA-256 checksums
- **SQLite FTS5** for BM25 text search
- **Metal HNSW** (via MetalANNS) for GPU-accelerated vector search
- **Reciprocal Rank Fusion** to combine text + vector results

### Benchmarks (M3 Max)

| Metric | Wax | Cloud RAG |
|--------|-----|-----------|
| Search latency (p95) | 6.1 ms | 150+ ms |
| Cold open (p95) | 9.2 ms | N/A |
| Ingest throughput | 85.9 docs/s | varies |

The Metal vector engine gives 5.4x speedup over CPU for warm queries. 1.58ms to search 1K vectors.

### What I learned

1. **File formats are infrastructure.** The WAL ring buffer was the hardest part—not the Metal kernels. Padding records, sentinel bytes, state snapshots for rollback. Get the format wrong and nothing else matters.

2. **CPU benchmarks can be misleading.** ANE (Apple Neural Engine) is faster for throughput, but ANECompilerService causes noise in latency measurements. We force CPU-only in XCTest for deterministic numbers.

3. **Hybrid search beats single-mode.** Fusing BM25 (exact text match) with cosine similarity (semantic match) catches cases neither handles alone. RRF is simple and works.

### Use cases

- On-device AI assistants with persistent memory
- CLI tools that remember context between invocations
- SwiftUI apps with semantic search
- Any agent that needs "remember X, recall Y" without cloud dependency

### Swift API

```swift
let memory = try await Memory(at: url)

// Store
try await memory.save("User prefers dark mode")

// Search (hybrid text + vector)
let results = try await memory.search("What does the user prefer?")

// Structured facts with temporal validity
await memory.assertFact(
    subject: "user",
    predicate: "prefers",
    object: "dark mode"
)
```

Swift 6.1+, iOS 18+, macOS 15+. Apache 2.0.

Happy to answer questions about the architecture, benchmarks, or Metal integration.

---

**Repo:** https://github.com/christopherkarani/Wax
