# LinkedIn Post — 2026-04-02

I spent the last 3 months building a memory engine for AI agents.

Not another cloud service.

A single portable file that runs entirely on-device.

---

The problem: most agent memory needs 3 separate services.

A vector database for semantic search.
A text database for keyword search.
A document store for raw data.

For cloud deployments, fine.

For on-device agents? It's overhead that kills latency and privacy.

---

So we built Wax.

A Swift-native persistence engine that packs everything into one .wax file:

→ SQLite FTS5 for BM25 text search
→ Metal HNSW for GPU-accelerated vectors
→ LZ4 compressed documents
→ Crash-resilient WAL ring buffer
→ Structured memory with temporal reasoning

One binary. No setup. No cloud dependency.

---

The numbers from our M3 Max benchmarks:

• 6.1ms hybrid search latency (p95)
• 9.2ms cold open (288x faster than baseline)
• 85.9 docs/sec ingest throughput
• 5.4x Metal GPU speedup over CPU

Cloud RAG services hit 150ms+ on good days.

---

The hardest part wasn't the Metal kernels.

It was the file format.

Dual A/B headers for atomic updates. Ring buffer WAL with padding records for wraparound. Sentinel bytes for corruption detection. SHA-256 checksums on everything.

File formats are infrastructure. Get them wrong and nothing else scales.

---

What I learned:

1. Single-file architectures force clarity. When your entire state is one binary, you think harder about what goes in.

2. Apple Silicon changes the math. Metal GPU + ANE makes on-device ML competitive with cloud services.

3. Hybrid search beats single-mode. Fusing text and vector catches what neither handles alone.

---

Wax is open source (Apache 2.0).

Swift 6.1+, iOS 18+, macOS 15+.

If you're building on-device AI agents and need persistent memory without cloud dependency — take a look.

🔗 github.com/christopherkarani/Wax

#Swift #OnDeviceAI #AIAgents #OpenSource #AppleSilicon #RAG #VectorSearch
