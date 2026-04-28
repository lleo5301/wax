# LinkedIn Post - 2026-04-02

I spent the last 3 months building a memory engine for AI agents.

not another cloud service.

a single portable file that runs entirely on-device.

the problem: most agent memory needs 3 separate services.

a vector database for semantic search. a text database for keyword search. a document store for raw data.

for cloud deployments, fine.

for on-device agents? it's overhead that kills latency and privacy.

so we built Wax.

a Swift-native persistence engine that packs everything into one .wax file:

SQLite FTS5 for BM25 text search
Metal HNSW for GPU-accelerated vectors
LZ4 compressed documents
crash-resilient WAL ring buffer
structured memory with temporal reasoning

one binary. no setup. no cloud dependency.

the numbers from our M3 Max benchmarks:

6.1ms hybrid search latency (p95)
9.2ms cold open (288x faster than baseline)
85.9 docs/sec ingest throughput
5.4x Metal GPU speedup over CPU

cloud RAG services hit 150ms+ on good days.

the hardest part wasn't the Metal kernels.

it was the file format.

dual A/B headers for atomic updates. ring buffer WAL with padding records for wraparound. sentinel bytes for corruption detection. SHA-256 checksums on everything.

file formats are infrastructure. get them wrong and nothing else scales.

what I learned:

single-file architectures force clarity. when your entire state is one binary, you think harder about what goes in.

Apple Silicon changes the math. Metal GPU plus ANE makes on-device ML competitive with cloud services.

hybrid search beats single-mode. fusing text and vector catches what neither handles alone.

Wax is open source (Apache 2.0).

Swift 6.1+, iOS 18+, macOS 15+.

if you're building on-device AI agents and need persistent memory without cloud dependency, take a look.

github.com/christopherkarani/Wax
