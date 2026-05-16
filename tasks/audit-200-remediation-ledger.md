# 200-Item Remediation Ledger

Created: 2026-05-13

Scope: fix the 200 verified audit findings one issue at a time, using TDD, preserving user-owned dirty files, and creating one fix commit per issue. This ledger is the source checklist for the remediation pass.

Commit policy:
- One issue fix per commit unless a finding is proven to be a duplicate during remediation.
- Each issue must have a failing test, compile gate, script fixture, or precise reproducible check before the production change.
- After each issue commit, run the focused verification for that issue and record the result here or in `tasks/todo.md`.
- Do not stage/delete unrelated generated artifacts.

## Stop-Point Status

Updated: 2026-05-17

Checklist legend:
- `[x]` means the issue has a committed fix on `bug-hunt` with focused verification recorded.
- `[ ]` means the issue is not fully complete. Some unchecked items may have local work in progress, but they are not counted complete until review and commit are done.

Current count:
- Target findings: 200
- Fully completed and committed: 61
- Work in progress, not counted complete: 0
- Remaining not fully completed: 139

Current resume point:
- `F063` duplicate vector frame-id restore/staging validation is reviewed and ready to commit.
- The active request is to fix all remaining F-through-A tier findings.

Current untracked/generated artifacts to preserve and not stage/delete:
- `.build-codex/`
- `.playwright-mcp/`
- `.qwen/`
- `issue61_full.png`
- `issue61_snapshot.md`

Known existing verification blockers from earlier runs:
- Full `WaxMCPServerTests` has reproducible existing failures in `rememberSearchAndRecallExposeTypedExplainableMemory` and `waxMCPProcessRememberWithRealCoreMLEmbedder`.
- Those failures are tracked separately and must not be attributed to unrelated issue fixes without a focused repro.

## Completed Issue Commits

| ID | Commit | Summary |
|---|---:|---|
| F022 | `c1aa8e44` | Reject invalid structured-memory key values. |
| F041 | `d50b7b4b` | Cover and enforce MCP labels filter validation. |
| F042 | `77d7edf4` | Cover and enforce MCP time filter validation. |
| F043 | `11479b07` | Reject unknown MCP filter keys. |
| F044 | `e5b11dd2` | Skip embeddings for blank recall queries. |
| F045 | `be54a0c6` | Add `MiniLMEmbeddings` trait define for `wax-mcp`. |
| F046 | `8b295f3e` | Add `MiniLMEmbeddings` trait define for `WaxRepo`. |
| F051 | `b057acdf` | Reject malformed staged vector index bytes. |
| F052 | `aed403bc` | Reject non-finite vector inputs. |
| F055 | `4db66e30` | Load staged USearch vector indexes. |
| F056 | `0da5001e` | Deduplicate USearch batch vector IDs. |
| F057 | `ffd36bcb` | Check vector decode byte-count overflow. |
| F058 | `6cc378b2` | Validate Metal vector segment bounds/trailing bytes. |
| F059 | `6607603c` | Use unaligned Metal frame ID loads. |
| F060 | `3b5e816b` | Normalize direct Metal vector search queries. |
| F061 | `9540c6a5` | Surface Metal command-buffer failures. |
| F062 | `3e8335e7` | Check projected vector counts and overflow. |
| F063 | `pending` | Reject duplicate vector frame IDs during restore/staging. |
| F073 | `50554c87` | Treat tokenizer newlines as whitespace. |
| F076 | `5e1025be` | Escape FTS5 MATCH queries. |
| F079 | `8b513214` | Reject non-positive FTS `topK`. |
| F082 | `bd2a6582` | Preserve non-socket daemon paths. |
| F083 | `9a650260` | Harden broker socket roots. |
| F084 | `8ef9048f` | Bound daemon socket reads. |
| F085 | `f8ffb6e7` | Redact CLI license key output. |
| F086 | `858b83b6` | Enforce require-vector for direct stats/flush. |
| F087 | `2495c05d` | Reject invalid embedder runtime flags. |
| F093 | `89140c79` | Stabilize daemon socket path regression. |
| F094 | `29f997c8` | Gate `knowledge_capture` by structured-memory flag. |
| F095 | `f5c8d24b` | Honor broker access-stats feature flag. |
| F100 | `63ce6e52` | Preserve broker memory content whitespace. |
| F101 | `ffa14be3` | Skip ended session manifests on resume. |
| F110 | `e742da53` | Add public snippet verifier. |
| F111 | `6372a5eb`, `f89be8f7` | Repair WaxDemo package path and public API usage. |
| F121 | `a22e5c1` | Fix waxmcp local npm README path. |
| F123 | `69470858` | Record/fix readiness parser remediation. |
| F126 | `67291613` | Fix Swift Testing skip detection gate. |
| F128 | `c57937ca` | Add Foundation import to README quick start. |
| F129 | `e941be9e` | Clarify README memory recall mode. |
| F130 | `80726257` | Fix WaxCore docs public surface. |
| F131 | `8b2cd1a2` | Remove package-only WaxCore DocC topics. |
| F132 | `2605cbb0` | Fix WaxCore getting-started option labels. |
| F133 | `abf9841e` | Remove nonexistent frame method docs. |
| F134 | `fec3d9f7` | Remove structured-memory public docs. |
| F135 | `7568c54f` | Fix vector docs public API boundary. |
| F136 | `b8bf7a4d` | Remove USearch construction docs. |
| F137 | `39e691f7` | Remove Metal construction docs. |
| F138 | `7bf87d20` | Update vector engine preference docs. |
| F139 | `373bfb99` | Remove stale vector streaming docs. |
| F140 | `06e1627d` | Fix text search docs public surface. |
| F141 | `9f015f39` | Remove `TextSearchResult` public docs. |
| F142 | `28dd7e1d` | Remove structured text package-only examples. |
| F143 | `7b9fa267` | Remove `WaxSession` public docs. |
| F144 | `6c7b25cd` | Remove `SearchRequest` public docs. |
| F145 | `4b635876` | Remove vector preference public docs. |
| F146 | `b3e2100d` | Cover session text put docs. |
| F147 | `419d5cbf` | Fix PhotoRAG docs public surface. |
| F148 | `c9b68117` | Cover PhotoRAG embedding provider docs. |
| F149 | `58f0e63d` | Fix PhotoRAG sync scope docs. |
| F150 | `281c5b5f` | Clarify VideoRAG docs access level. |
| F151 | `0cf62175` | Enforce MCP trait test inventory. |

Support commit not counted as a finding fix:
- `cb400efe` hardened structured-memory docs guard tests.

## Findings

- [ ] F001 Durability: `Wax.create` truncates/open-writes before lock ownership is proven.
- [ ] F002 WAL: pending payload replay lacks checksum validation.
- [ ] F003 WAL scan: forgiving WAL scan can drop later valid pending records after corrupt state.
- [ ] F004 Commit atomicity: `commitLocked` mutates live TOC before durable writes with no rollback.
- [ ] F005 Delete/supersede: committed state mutates in place without rollback.
- [ ] F006 File format: offset arithmetic can trap on `UInt64` overflow.
- [ ] F007 Verify/open: footer selection differs between verification and open.
- [ ] F008 Repair: truncate repair lacks durable fsync.
- [ ] F009 WAL ordering: invalid delete/supersede WAL can be appended before validation.
- [ ] F010 Structured facts: fact hash normalizes entity/predicate case.
- [ ] F011 Structured facts: string value hash lowercases object values.
- [ ] F012 Evidence: `facts` query drops stored evidence.
- [ ] F013 Bitemporal: updating a fact closes all subject/predicate spans.
- [ ] F014 Relations: `version_relation` is overwritten on the fact row.
- [ ] F015 Retractions: retract can insert a current fact row.
- [ ] F016 Span hash: hash omits `system_to` and allows sentinel collision.
- [ ] F017 Time: non-monotonic system time and overflow are not guarded.
- [ ] F018 Retractions: same-millisecond retract can fail to close target.
- [ ] F019 Query results: duplicate identical fact hits are indistinguishable.
- [ ] F020 Query metadata: `wasTruncated` can be false-positive.
- [ ] F021 Entities: entity kind cannot be corrected.
- [x] F022 Validation: key types accept empty, whitespace, or unbounded values.
- [ ] F023 Evidence: invalid spans/confidence are accepted.
- [ ] F024 Graph API: edge traversal API is unwired/dead.
- [ ] F025 Structured search: object-side entity facts are not used in `evidenceFrameIds`.
- [ ] F026 Bitemporal MCP: orchestrator/MCP collapses bitemporal `asOf`.
- [ ] F027 Unified search: `timeRange.before` is treated as system as-of.
- [ ] F028 Alias resolution: alias matching is exact despite fuzzy docs.
- [ ] F029 MCP facts: `fact_assert` lacks evidence support.
- [ ] F030 Unified search: metadata filters can starve candidate results.
- [ ] F031 Pending search: pending metadata can match while previews are committed-only.
- [ ] F032 Corpus export: superseded active docs can be exported.
- [ ] F033 MCP search: `memory_search topK` caps before post-filtering.
- [ ] F034 Sessions: multiple active sessions can silently ignore working memory.
- [ ] F035 MCP schema: vector search mode/options are hidden.
- [ ] F036 MCP tools: flush handler exists but is undiscoverable/rejected inconsistently.
- [ ] F037 Pending memory: pending duplicate dedupe gap.
- [ ] F038 Diagnostics: broker filters lack includeDeleted/superseded/frame IDs.
- [ ] F039 MCP schema: `fact_assert` omits relation/version relation.
- [ ] F040 MCP schema: generic `type/value` fact schema is rejected by broker.
- [x] F041 MCP filters: non-array `labels` filter is ignored.
- [x] F042 MCP filters: non-integer time filters are ignored.
- [x] F043 MCP filters: unknown nested filters are ignored.
- [x] F044 Recall: whitespace-only recall can embed/search unrelated content.
- [x] F045 Traits: `wax-mcp` missing `MiniLMEmbeddings` define.
- [x] F046 Traits: `WaxRepo` missing `MiniLMEmbeddings` define.
- [ ] F047 Linux: MCP Linux path imports Darwin/CoreGraphics-only APIs.
- [ ] F048 Linux: `Darwin.exit` used unconditionally.
- [ ] F049 Linux tests: excludes miss Darwin benchmark files.
- [ ] F050 Dependencies: top-level dependency leakage pulls SwiftTUI into non-CLI builds.
- [x] F051 Vector WAL: malformed vector staged/verify accepted.
- [x] F052 Embeddings: NaN/Inf embeddings accepted.
- [ ] F053 USearch: `add` is not atomic.
- [ ] F054 USearch: concurrent reads are unchecked around mutable index.
- [x] F055 Pending vectors: USearch ignores staged vector bytes.
- [x] F056 Batch vectors: duplicate IDs in batch overcount vector count.
- [x] F057 Serialization: unchecked `Int` overflow in vector decode.
- [x] F058 Metal vectors: deserialize misses bounds/trailing-byte validation.
- [x] F059 Metal vectors: unaligned frame ID loads.
- [x] F060 Metal scoring: cosine query normalization is missing/inconsistent.
- [x] F061 Metal errors: command-buffer error ignored.
- [x] F062 Manifest: `vectorCount` unchecked cast.
- [x] F063 Vector restore: duplicate frame IDs deserialize inconsistently.
- [ ] F064 Serialization: private Objective-C ivar serialization is fragile.
- [ ] F065 MiniLM: batch size 2/4 fails.
- [ ] F066 MiniLM: default batch 256 exceeds asset shape 64.
- [ ] F067 Arctic: default batch 256 exceeds asset shape 64.
- [ ] F068 Embeddings: direct output not normalized as docs/identity imply.
- [ ] F069 Embeddings: non-finite output is not rejected.
- [ ] F070 CoreML errors: `try?` hides CoreML failures.
- [ ] F071 MiniLM tests: quality test bypasses public batch embedder.
- [ ] F072 Tokenizer batching: pre-tokenized embeddings always use batch size 1.
- [x] F073 Tokenizer: whitespace splitting excludes newlines.
- [ ] F074 Tokenizer: token type IDs mark SEP/padding as segment 1.
- [ ] F075 CoreML dtype: unsupported `MLMultiArray` dtype becomes zeros.
- [x] F076 FTS5: raw MATCH query is not escaped.
- [ ] F077 FTS index: delete/supersede do not update FTS index consistently.
- [ ] F078 Ranking: BM25 score is not normalized versus `minScore`.
- [x] F079 Validation: `topK <= 0` clamps to 1.
- [ ] F080 Schema: FTS schema validation is weak.
- [ ] F081 Tokenizer: default FTS tokenizer/version is unpinned.
- [x] F082 CLI daemon: `--socket-path` can unlink arbitrary file.
- [x] F083 CLI daemon: normal broker socket directory is not private.
- [x] F084 CLI daemon: socket `readToEnd` can hang.
- [x] F085 CLI secrets: license key leaks through dry-run/argv output.
- [x] F086 CLI flags: direct `stats/flush` ignore `require-vector`.
- [x] F087 CLI flags: invalid runtime flags are silently ignored.
- [ ] F088 CLI surface: CLI lacks broker/MCP parity subcommands.
- [ ] F089 WaxRepo: `wax-repo search` still launches TUI.
- [ ] F090 WaxRepo: `--full` duplicates store content.
- [ ] F091 WaxRepo: `max-commits` checkpoint can skip older history permanently.
- [ ] F092 WaxRepo: repo search parses preview instead of metadata.
- [x] F093 CLI tests: daemon stable socket path expectation fails.
- [x] F094 MCP structured: `knowledge_capture` bypasses structured-memory flag.
- [x] F095 Broker config: access-stats env is parsed/logged but ignored.
- [ ] F096 HTTP MCP: HTTP transport has no auth off-loopback.
- [ ] F097 Session end: active session removed before fallible persistence.
- [ ] F098 Broker commit: `remember/handoff` commit before event failure.
- [ ] F099 Knowledge capture: graph write before memory write can half-commit.
- [x] F100 MCP content: content strings are trimmed.
- [x] F101 Session resume: `session_resume` can pick ended manifest.
- [ ] F102 HTTP MCP: body limit is enforced after full read.
- [ ] F103 Corpus: corpus rebuild is non-atomic.
- [ ] F104 MCP config: invalid embedder choice falls back to MiniLM.
- [ ] F105 Multimodal MCP: multimodal is advertised but not wired.
- [ ] F106 HTTP lifecycle: cleanup loop has no cancellation.
- [ ] F107 MCP tests: broker-backed durable capture test times out.
- [ ] F108 MCP tests: locked-session corpus search test times out.
- [ ] F109 npm package: packed tarball lacks `dist` binaries.
- [x] F110 Snippet gate: public snippet verifier is missing.
- [x] F111 Demo package: WaxDemo points to missing `../Wax`.
- [ ] F112 Release script: version grep targets stale source pattern.
- [ ] F113 Release script: checksum path is cwd-sensitive.
- [ ] F114 Release workflow: npm metadata is bumped after build only.
- [ ] F115 OpenClaw npm: package ships TypeScript source without loader/build.
- [ ] F116 OpenClaw: default command `wax-mcp` unavailable from plugin package.
- [ ] F117 Release arch: local release stages only arm64 while metadata advertises x64.
- [ ] F118 Homebrew: formula version is stale.
- [ ] F119 Homebrew: directory is gitlink-like without `.gitmodules`.
- [ ] F120 Homebrew: formula Xcode 15 requirement is too old for Swift 6.1 traits.
- [x] F121 npm README: local path `./npm/waxmcp` is wrong from repo root.
- [ ] F122 Release scripts: root/nested release scripts rewrite different version files.
- [x] F123 Readiness gate: pass-rate parser fails on Swift Testing output.
- [ ] F124 Docs script: docs generation uses wrong root/destructive copy assumptions.
- [ ] F125 Website CI: no PR build gate for website/docs.
- [x] F126 Test gate: skip detector misses Swift Testing skip format.
- [ ] F127 CI scope: Linux CI omits Wax/CLI/MCP product builds.
- [x] F128 README: quick-start omits required `Foundation` import.
- [x] F129 README: advertises hybrid recall where public `Memory` path is text-only.
- [x] F130 Public API: WaxCore documents package-only `Wax` actor.
- [x] F131 DocC: WaxCore topics list package-only symbols.
- [x] F132 Docs: `WaxOptions` labels are wrong.
- [x] F133 Docs: `putFrame/frame/readPayload` methods do not exist.
- [x] F134 Docs/API: structured-memory types documented public but package-only.
- [x] F135 Vector API: `VectorSearchEngine` documented public but package-only.
- [x] F136 Vector docs: docs instantiate package-only `USearchVectorEngine`.
- [x] F137 Vector docs: docs instantiate package-only `MetalVectorEngine`.
- [x] F138 Vector docs: docs teach deprecated `.metalPreferred`.
- [x] F139 Vector docs: docs claim protocol has `addBatchStreaming`.
- [x] F140 Text API: `FTS5SearchEngine` documented public but package-only.
- [x] F141 Text API: `TextSearchResult` documented public but package-only.
- [x] F142 Text docs: structured text examples use package-only engine/types.
- [x] F143 Session API: `WaxSession` documented user-facing but package-only.
- [x] F144 Unified API: docs construct package-only `SearchRequest`.
- [x] F145 Config docs: docs expose package-only vector config enums.
- [x] F146 Session docs: `session.put(text:)` signatures do not exist.
- [x] F147 Photo API: `PhotoRAGOrchestrator` documented public but package-only.
- [x] F148 Photo API: docs say `EmbeddingProvider`, code needs `MultimodalEmbeddingProvider`.
- [x] F149 Photo docs: sample uses nonexistent `.all` scope.
- [x] F150 Video API: `VideoRAGOrchestrator` documented public but package-only.
- [x] F151 Test coverage: default tests miss MCP trait suite.
- [ ] F152 MCP tests: many tests use direct-memory compatibility path, not production broker path.
- [ ] F153 MCP tests: compatibility aliases differ from production renamed-tool behavior.
- [ ] F154 HTTP smoke: HTTP verifier never calls a tool.
- [ ] F155 CI: Linux CI omits Wax/CLI/MCP product builds.
- [ ] F156 Gate script: skip detector misses Swift Testing skips.
- [ ] F157 Readiness tests: stability gate is text-search only.
- [ ] F158 Migration tests: N-1/N-2 fixtures are generated by current code.
- [ ] F159 MiniLM tests: inference tests silently return when env unset.
- [ ] F160 MiniLM tests: missing-resource tests catch any error.
- [ ] F161 PDF API: PDF API has no non-PDFKit fallback.
- [ ] F162 PDF ingest: extraction hard-limits 500 pages with no metadata.
- [ ] F163 PDF provenance: page provenance is lost after join.
- [ ] F164 Photo ingest: docs claim local image ingest but code lacks local file API.
- [ ] F165 Photo sync: fetches all PHAssets, not images only.
- [ ] F166 Photo filters: `PhotoFilters` is empty/no-op.
- [ ] F167 Photo location: radius uses coarse bins without final distance check.
- [ ] F168 Photo diagnostics: degraded flag inferred from missing OCR/caption, not locality.
- [ ] F169 Photo ingest: failed region crops return before superseding old root.
- [ ] F170 Photo sync: `syncState` frame kind is never written.
- [ ] F171 Photo tags: tags overpromise classifier labels.
- [ ] F172 Video thumbnails: Photos video local URL discarded, thumbnails fail.
- [ ] F173 Broker validation: direct broker ignores unknown args.
- [ ] F174 Session start: manifest saved before `.started` event.
- [ ] F175 Session resume: lease stolen before `.resumed` event.
- [ ] F176 Events: first event file creation return ignored.
- [ ] F177 Events: one malformed JSONL line aborts whole log.
- [ ] F178 Manifests: corrupt stray manifest aborts all listing.
- [ ] F179 Promotion: memory written before stale-session validation.
- [ ] F180 Promotion: metadata keeps raw `session_id`.
- [ ] F181 Promotion: `max_candidates` unbounded above.
- [ ] F182 Markdown sync: marker trust uses frame ID only.
- [ ] F183 Markdown sync: locked memory can be deleted by removing markdown line.
- [ ] F184 Markdown sync: dry-run skips durable-write validation.
- [ ] F185 Markdown import: markerless bullets imported as managed.
- [ ] F186 DREAMS: duplicate checked lines create duplicate durable memories.
- [ ] F187 DREAMS: export excludes ended sessions.
- [ ] F188 DREAMS: projection closes session stores in unawaited task.
- [ ] F189 Markdown export: opens active sessions owned by other brokers.
- [ ] F190 Markdown export: leaves stale generated files.
- [ ] F191 Markdown export: `wax.source_date` path traversal in daily filename.
- [ ] F192 Markdown marker: marker JSON is not escaped for `-->`.
- [ ] F193 Markdown export: reclassifies text instead of metadata type.
- [ ] F194 Compact context: references raw chunk frame IDs.
- [ ] F195 Compact context: token budget counts hit text, not rendered output.
- [ ] F196 Compact context: recency prefix filters sessions before relevance.
- [ ] F197 Corpus cache: corrupt corpus manifest aborts instead of rebuild.
- [ ] F198 Enrichment: handler result is discarded.
- [ ] F199 Enrichment: structured extraction hardcodes empty entities.
- [ ] F200 Keywords: technical identifiers are split/lowercased.
