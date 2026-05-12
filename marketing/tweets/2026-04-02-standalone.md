# Standalone Tweets — 2026-04-02

## Tweet 1 — [type: metric bomb]

9.2ms cold open.
6.1ms hybrid search.
85.9 docs/sec ingest.

All on-device. No cloud. Single .wax file.

Wax is a memory engine for AI agents that runs on Metal.

---

## Tweet 2 — [type: hot take]

Stop shipping user data to cloud vector databases for RAG.

We built a 256MB ring buffer + Metal HNSW + SQLite FTS5 into one portable file.

6ms recall latency. 100% local. No API keys.

Your agents deserve better than round-trips to Pinecone.

---

## Tweet 3 — [type: code flex]

All you need for persistent agent memory:

```swift
let memory = try await Memory(at: url)
try await memory.save("User building habit tracker in SwiftUI")
let results = try await memory.search("What is the user building?")
```

That's it. Hybrid text+vector search. On-device. Swift native.

📎 Image: `../assets/code-images/01-basic-api.png`
🔗 Repo: https://github.com/christopherkarani/Wax

---

## Tweet 4 — [type: TIL]

TIL: SQLite FTS5 + GPU-accelerated HNSW can coexist in a single binary file.

Wax embeds both search engines inside `.wax`. One query fans out to BM25 (text) and cosine similarity (vectors), then fuses results with Reciprocal Rank Fusion.

Latency stays under 7ms.

---

## Tweet 5 — [type: metric bomb + insight]

288x faster cold open.

From 2.65s → 9.2ms.

The trick? Dual A/B header pages with SHA-256 checksums. Recovery just picks the header with the higher generation counter. No SQLite journal. No fsync storms.

---

## Tweet 6 — [type: API showcase]

Structured memory for agent reasoning:

```swift
await memory.upsertEntity(key: "user", kind: "person")
await memory.assertFact(
    subject: "user",
    predicate: "prefers",
    object: "dark mode"
)
```

Entity-Attribute-Value with temporal validity. Facts know *when* they were true.

📎 Image: `../assets/code-images/02-structured-memory.png`

---

## Tweet 7 — [type: diagram post]

Wax file format visualized:

📎 Image: `../assets/diagrams/01-wax-file-format.svg`

Dual headers → WAL ring buffer → Compressed frames → Hybrid indices → TOC with Merkle root.

One file. Atomic. Portable.

---

## Tweet 8 — [type: counterintuitive finding]

Counterintuitive: CPU-only MiniLM beats ANE for benchmark determinism.

The ANECompilerService process causes noise in latency measurements. We force CPU-only mode in XCTest for stable numbers.

Real-world? ANE still wins for throughput. Benchmarks just lie about tail latency.

---

## Tweet 9 — [type: insight]

Lesson from building Wax:

The hardest part wasn't the Metal kernels or HNSW graph.

It was the WAL ring buffer. Circular writes. Padding records. Sentinel bytes. State snapshots for rollback.

File formats are infrastructure. Get them wrong and nothing else matters.
