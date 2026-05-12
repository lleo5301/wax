# Thread - Building a Memory Engine for AI Agents That Doesn't Need the Cloud - 2026-04-02

## 1/ (Hook)

I spent 3 months building a memory engine for AI agents.

the constraint: everything runs on-device. no cloud. no API keys. single portable file.

results: 6ms hybrid search, 85.9 docs/sec ingest, 288x faster cold opens.

here's how we did it.

---

## 2/

most agent memory today looks like this:

user asks question, send to cloud, query Pinecone/Weaviate, wait 150-500ms, get result, respond.

for a chatbot, fine.

for an agent running 100s of queries per minute? it's a bottleneck.

📎 Image: `../assets/diagrams/02-cloud-vs-local.svg`

---

## 3/

we wanted something different:

- 100% on-device (Apple Silicon)
- single portable file (no Docker, no database)
- hybrid search (text + vector)
- crash-resilient writes

the answer: pack SQLite FTS5 and Metal HNSW into one binary format.

---

## 4/

the .wax file format:

```
Dual Header (A/B) = atomic updates
WAL Ring Buffer = 256MB, crash-safe
Compressed Frames = LZ4/LZFSE
Hybrid Indices = FTS5 + HNSW
TOC + Merkle Root = integrity
```

one file contains everything: documents, embeddings, text index, vector index.

---

## 5/

the dual header trick:

two 4KB header pages. each has a generation counter.

every commit increments the counter and writes to the other page.

on crash recovery? just read both, pick the one with the higher generation.

no complex rollback logic. no fsync storms.

---

## 6/

the WAL was the hardest part.

ring buffer with padding records for wraparound. sentinel bytes for corruption detection. state snapshots for rollback.

we benchmarked commit latency at 34ms p95 for 10K hybrid docs.

📎 Image: `../assets/code-images/03-wal-ring-buffer.png`

---

## 7/

Metal HNSW vector search:

Apple's MetalANNS framework gives us GPU-accelerated HNSW graphs.

5.4x speedup over CPU for warm queries. 1.58ms to search 1K vectors at 128 dimensions.

the index lives in the .wax file as a flat float32 array plus frame ID mapping.

---

## 8/

MiniLM embeddings via CoreML:

all-MiniLM-L6-v2, 384 dimensions, L2 normalized for cosine similarity.

batch processing up to 256 texts. ANE/GPU acceleration with CPU fallback.

throughput: 85.9 documents/sec with full hybrid indexing.

---

## 9/

hybrid search fusion:

one query fans out to BM25 (text relevance) and cosine similarity (semantic match).

results fused with Reciprocal Rank Fusion:

```
RRF(d) = Σ weight_i / (k + rank_i + 1)
```

default alpha 0.5 (equal weight). tunable per query.

---

## 10/

structured memory for agent reasoning:

EAV with temporal validity. facts know when they were true in reality (valid time) and when the agent learned them (system time).

```swift
await memory.assertFact(
    subject: "user",
    predicate: "prefers",
    object: "dark mode",
    valid: .init(fromMs: now, toMs: nil)  // still true
)
```

---

## 11/

the numbers:

| Metric | Result |
|--------|--------|
| Cold open p95 | 9.2ms |
| Hybrid search p95 | 6.1ms |
| Ingest throughput | 85.9 docs/s |
| Metal vector speedup | 5.4x |

compared to cloud RAG (150ms+ latency), it's not even close.

---

## 12/ (Closer)

key insight from building wax:

the memory format IS the architecture. get the file format right, atomic updates, integrity checks, portable serialization, and everything else slots in.

bad formats don't get better with scale.

🔗 GitHub: https://github.com/christopherkarani/Wax
📖 Full technical breakdown in README
