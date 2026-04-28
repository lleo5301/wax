# Standalone Tweets - 2026-04-02

## Tweet 1

9.2ms cold open.
6.1ms hybrid search.
85.9 docs/sec ingest.

all on-device. no cloud. single .wax file.

wax is a memory engine for AI agents. runs on Metal.

📎 Image: `../assets/code-images/01-basic-api.png`

---

## Tweet 2

stop shipping user data to cloud vector databases for RAG.

we packed a 256MB ring buffer, Metal HNSW, and SQLite FTS5 into one file.

6ms recall. 100% local. no API keys.

your agents deserve better than round-trips to Pinecone.

---

## Tweet 3

all you need for persistent agent memory:

```swift
let memory = try await Memory(at: url)
try await memory.save("User building habit tracker in SwiftUI")
let results = try await memory.search("What is the user building?")
```

that's it. hybrid text + vector search. on-device.

📎 Image: `../assets/code-images/01-basic-api.png`
🔗 Repo: https://github.com/christopherkarani/Wax

---

## Tweet 4

TIL SQLite FTS5 and GPU-accelerated HNSW can coexist in one binary file.

wax embeds both search engines inside .wax. one query fans out to BM25 and cosine similarity, then fuses with Reciprocal Rank Fusion.

latency stays under 7ms.

---

## Tweet 5

288x faster cold open.

from 2.65s to 9.2ms.

the trick: dual A/B header pages with SHA-256 checksums. recovery picks the header with the higher generation counter. no SQLite journal. no fsync storms.

📎 Image: `../assets/diagrams/01-wax-file-format.svg`

---

## Tweet 6

structured memory for agent reasoning:

```swift
await memory.upsertEntity(key: "user", kind: "person")
await memory.assertFact(
    subject: "user",
    predicate: "prefers",
    object: "dark mode"
)
```

EAV with temporal validity. facts know when they were true.

📎 Image: `../assets/code-images/02-structured-memory.png`

---

## Tweet 7

wax file format visualized:

📎 Image: `../assets/diagrams/01-wax-file-format.svg`

dual headers, WAL ring buffer, compressed frames, hybrid indices, TOC with Merkle root.

one file. atomic. portable.

---

## Tweet 8

counterintuitive: CPU-only MiniLM beats ANE for benchmark determinism.

ANECompilerService causes noise in latency measurements. we force CPU-only in XCTest for stable numbers.

real world? ANE still wins for throughput. benchmarks just lie about tail latency.

---

## Tweet 9

lesson from building wax:

hardest part wasn't the Metal kernels or HNSW graph.

it was the WAL ring buffer. circular writes. padding records. sentinel bytes. state snapshots for rollback.

file formats are infrastructure. get them wrong and nothing else matters.
