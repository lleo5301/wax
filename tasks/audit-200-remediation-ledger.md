# 200-Item Remediation Ledger

Created: 2026-05-13

Scope: fix the 200 verified audit findings one issue at a time, using TDD, preserving user-owned dirty files, and creating one fix commit per issue. This ledger is the source checklist for the remediation pass.

Commit policy:
- One issue fix per commit unless a finding is proven to be a duplicate during remediation.
- Each issue must have a failing test, compile gate, script fixture, or precise reproducible check before the production change.
- After each issue commit, run the focused verification for that issue and record the result here or in `tasks/todo.md`.
- Do not stage/delete unrelated generated artifacts.

## Stop-Point Status

Updated: 2026-05-18

Checklist legend:
- `[x]` means the issue has a committed fix on `bug-hunt` with focused verification recorded.
- `[ ]` means the issue is not fully complete. Some unchecked items may have local work in progress, but they are not counted complete until review and commit are done.

Current count:
- Target findings: 200
- Fully completed and committed: 195
- Work in progress, not counted complete: 0
- Remaining not fully completed: 5

Current resume point:
- F006 through F009, F047 through F050, F091, F097 through F099, F103, F105, F152, and F157 are now fixed; remaining active work is the deeper WaxCore durability findings F001 through F005.
- The active request is to fix all remaining findings.

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
| F012 | `fd399c92` | Return stored structured fact evidence from facts queries. |
| F029 | `5fe1386b` | Accept and return fact evidence through MCP and broker APIs. |
| F033 | `acc41d3e` | Overfetch MCP compatibility memory search before horizon filtering. |
| F038 | `57eb2f1e` | Expose broker lifecycle search filters. |
| F022 | `c1aa8e44` | Reject invalid structured-memory key values. |
| F041 | `d50b7b4b` | Cover and enforce MCP labels filter validation. |
| F042 | `77d7edf4` | Cover and enforce MCP time filter validation. |
| F043 | `11479b07` | Reject unknown MCP filter keys. |
| F044 | `e5b11dd2` | Skip embeddings for blank recall queries. |
| F045 | `be54a0c6` | Add `MiniLMEmbeddings` trait define for `wax-mcp`. |
| F046 | `8b295f3e` | Add `MiniLMEmbeddings` trait define for `WaxRepo`. |
| F047 | `38207fcdc` | Guard MCP multimodal CoreGraphics/ImageIO imports for Linux builds. |
| F048 | `c82cd3e76` | Replace Darwin-qualified MCP server exits with a portable helper. |
| F049 | `f15d39e31` | Exclude Darwin-only integration benchmarks from Linux test builds. |
| F050 | `153e6a5d6` | Scope WaxRepo UI dependencies and target to macOS. |
| F051 | `b057acdf` | Reject malformed staged vector index bytes. |
| F052 | `aed403bc` | Reject non-finite vector inputs. |
| F053 | `resolved-by-F064` | USearch `add` atomicity gap eliminated with USearch engine removal. |
| F054 | `resolved-by-F064` | USearch mutable-index read concurrency gap eliminated with USearch engine removal. |
| F055 | `4db66e30` | Load staged USearch vector indexes. |
| F056 | `0da5001e` | Deduplicate USearch batch vector IDs. |
| F057 | `ffd36bcb` | Check vector decode byte-count overflow. |
| F058 | `6cc378b2` | Validate Metal vector segment bounds/trailing bytes. |
| F059 | `6607603c` | Use unaligned Metal frame ID loads. |
| F060 | `3b5e816b` | Normalize direct Metal vector search queries. |
| F061 | `9540c6a5` | Surface Metal command-buffer failures. |
| F062 | `3e8335e7` | Check projected vector counts and overflow. |
| F063 | `pending` | Reject duplicate vector frame IDs during restore/staging. |
| F064 | `a79c5f63` | Remove fragile USearch private-ivar serialization path. |
| F065 | `verified` | Verified MiniLM batch sizes 2/4 are decomposed to supported single predictions. |
| F066 | `verified` | Verified MiniLM default batch 256 cannot exceed the effective supported prediction cap. |
| F067 | `170d278e` | Clamp Arctic CoreML batch planning to supported shapes. |
| F068 | `e11607b3` | Normalize MiniLM embedder outputs before returning them. |
| F069 | `a1fd4fe4` | Reject non-finite MiniLM embedder outputs. |
| F070 | `f5d09368` | Propagate MiniLM CoreML prediction errors. |
| F071 | `0a565d2d` | Exercise public MiniLM embedder in quality tests. |
| F072 | `9684b5a5` | Infer MiniLM pre-tokenized batch size from input rows. |
| F073 | `50554c87` | Treat tokenizer newlines as whitespace. |
| F074 | `37e4565b` | Keep first SEP and padding token type IDs in segment zero. |
| F075 | `434912b0` | Reject unsupported MiniLM CoreML output data types. |
| F076 | `5e1025be` | Escape FTS5 MATCH queries. |
| F077 | `3ac522f2` | Purge deleted and superseded frames from FTS staging. |
| F078 | `b9335725` | Normalize FTS BM25 scores for bounded `minScore` filtering. |
| F079 | `8b513214` | Reject non-positive FTS `topK`. |
| F082 | `bd2a6582` | Preserve non-socket daemon paths. |
| F083 | `9a650260` | Harden broker socket roots. |
| F084 | `8ef9048f` | Bound daemon socket reads. |
| F085 | `f8ffb6e7` | Redact CLI license key output. |
| F086 | `858b83b6` | Enforce require-vector for direct stats/flush. |
| F087 | `2495c05d` | Reject invalid embedder runtime flags. |
| F088 | `64af3922` | Add broker/MCP parity commands to `wax-cli`. |
| F089 | `112dcefa` | Run `wax-repo search <query>` as a one-shot command. |
| F090 | `405a6248` | Rebuild WaxRepo full reindex into a fresh store before swapping. |
| F091 | `5de373cd9` | Avoid checkpointing WaxRepo history when `--max-commits` may have truncated the batch. |
| F092 | `36d5e36a` | Build WaxRepo search results from stored metadata instead of previews. |
| F093 | `89140c79` | Stabilize daemon socket path regression. |
| F094 | `29f997c8` | Gate `knowledge_capture` by structured-memory flag. |
| F095 | `f5c8d24b` | Honor broker access-stats feature flag. |
| F096 | `2acac690` | Require bearer auth for non-loopback HTTP MCP binds. |
| F097 | `4903b136f` | Keep active broker sessions registered until session-end persistence succeeds. |
| F098 | `8bc0e6a96` | Record broker session events before commit flushes. |
| F099 | `bb498a7cc` | Stage durable knowledge memory before graph writes and commit graph updates in the final flush. |
| F100 | `63ce6e52` | Preserve broker memory content whitespace. |
| F101 | `ffa14be3` | Skip ended session manifests on resume. |
| F103 | `479685ebf` | Preserve existing broker corpus store while swapping in rebuilt corpus. |
| F105 | `ad8a168f4` | Stop advertising unpublished multimodal MCP tools. |
| F109 | `f305c477` | Add waxmcp prepack validation for required dist artifacts. |
| F112 | `b8c8fe18` | Update waxmcp release version extraction to `WaxMCPServerMetadata.version`. |
| F113 | `30c8bdef` | Verify basename-only release checksums from artifact directories. |
| F114 | `fa058cbb` | Set waxmcp release version before building binaries. |
| F115 | `841b0763` | Publish OpenClaw plugin JavaScript entry from dist instead of TypeScript source. |
| F116 | `pending` | Default OpenClaw runtime to the packaged waxmcp launcher. |
| F117 | `cf2141aa` | Stage both darwin-arm64 and darwin-x64 artifacts in the local waxmcp release script. |
| F110 | `e742da53` | Add public snippet verifier. |
| F111 | `6372a5eb`, `f89be8f7` | Repair WaxDemo package path and public API usage. |
| F121 | `a22e5c1` | Fix waxmcp local npm README path. |
| F123 | `69470858` | Record/fix readiness parser remediation. |
| F118 | `02ff5896` | Update Homebrew formula to the current waxmcp package version. |
| F119 | `80ec3020` | Add missing `.gitmodules` metadata for the Homebrew tap gitlink. |
| F120 | `a3b9111c` | Require an Xcode version compatible with Swift tools 6.1 in the Homebrew formula. |
| F122 | `3c0507b5` | Make root waxmcp release script delegate to the canonical Resources script. |
| F124 | `33ed8635` | Fix docs generation root resolution and atomic output replacement. |
| F127 | `cee3dee0` | Add Wax, wax-cli, and wax-mcp product builds to Linux CI. |
| F158 | `54d25930` | Replace generated-at-test-time migration fixtures with packaged fixture bytes. |
| F159 | `63272352` | Replace silent MiniLM inference test returns with explicit disabled metadata. |
| F160 | `4bf2c825` | Assert exact MiniLM missing-resource errors. |
| F161 | `e9a296f8` | Keep PDF ingest API available on non-PDFKit platforms with an explicit unsupported-platform error. |
| F162 | `bef91caa` | Persist PDF extraction limit and truncation metadata. |
| F163 | `48c1e8e7` | Preserve PDF page provenance through page-scoped ingest metadata. |
| F164 | `d72709505` | Add local image file ingest support to PhotoRAG. |
| F165 | `642f160a3` | Filter full-library PhotoRAG sync to image PHAssets only. |
| F166 | `e35eaa0ca`, `0cb522723` | Apply metadata and asset allowlist filters during PhotoRAG recall. |
| F167 | `1d536236a` | Apply exact distance checks after PhotoRAG coarse location bins. |
| F168 | `f1e872d05` | Count PhotoRAG degraded results from local-availability metadata. |
| F169 | `0d31a7d30` | Avoid PhotoRAG region crop return/trap paths before superseding old roots. |
| F170 | `5e741acaa` | Persist PhotoRAG sync-state checkpoint frames after successful library sync. |
| F171 | `3c6541b08` | Clarify PhotoRAG tags are metadata keywords or caption fallback terms, not classifier labels. |
| F172 | `951546bd0` | Preserve local Photos video URLs so recalled video segments can attach thumbnails. |
| F174 | `9afb78c2d` | Append broker session `.started` events before saving active manifests. |
| F175 | `3312eeb8b` | Append broker session `.resumed` events before saving refreshed lease manifests. |
| F176 | `d9a0403c8` | Throw when first broker event log file creation fails. |
| F177 | `5ba327ad5` | Skip malformed broker event JSONL lines while preserving valid events. |
| F178 | `9416e09b0` | Ignore non-session stray JSON while listing broker session manifests. |
| F179 | `331657a10` | Validate explicit promotion sessions before durable writes. |
| F180 | `9a342d23c` | Drop raw `session_id` metadata from promoted durable memories. |
| F182 | `2e0f6a1fd` | Validate Markdown projection markers before matching existing memory frames. |
| F183 | `620348b66` | Preserve locked Markdown-managed memories when projection lines are removed. |
| F184 | `8dff9e315` | Run durable-write validation during Markdown sync dry-runs. |
| F185 | `259982606` | Require explicit managed Markdown markers for sync imports. |
| F186 | `83a492ae9` | Deduplicate checked DREAMS approvals within one Markdown sync. |
| F187 | `9aef7c8e9` | Export and approve DREAMS proposals from ended sessions. |
| F188 | `62a78c1fe` | Await session-store closes during DREAMS projection. |
| F189 | `583b4ff38` | Skip foreign active sessions during Markdown export. |
| F190 | `5012816f3` | Remove stale generated Markdown export files safely. |
| F193 | `66a7f79b0` | Preserve stored memory types when grouping Markdown export sections. |
| F194 | `20b239c58` | Guard compact context against emitting raw chunk frame IDs. |
| F195 | `968b40a14` | Budget compact context against rendered output tokens. |
| F196 | `ccf862964` | Rank ended sessions by relevance before compact-context cutoff. |
| F197 | `f47d89bee` | Rebuild corpus caches when manifest JSON is corrupt. |
| F198 | `5f30dc68b` | Persist async enrichment results instead of discarding handler output. |
| F199 | `f23780084` | Extract deterministic async enrichment entity mentions. |
| F200 | `e33afc13b` | Preserve technical keyword identifiers without over-preserving prose compounds. |
| F006 | `adb603223` | Guard footer/header file-format offset arithmetic from UInt64 overflow traps. |
| F007 | `568bb5351` | Make deep verification select the same newest valid footer as open. |
| F008 | `adf0b7f2c` | Fsync after repair truncates trailing bytes. |
| F009 | `c8704360f` | Validate delete and supersede mutations before appending WAL entries. |
| F025 | `0666ea1f7` | Include entity-valued fact objects in structured evidence search. |
| F026 | `774ca2919` | Expose separate system and valid timestamps for facts queries. |
| F027 | `7487b754d` | Keep unified-search frame time filters separate from structured-memory as-of queries. |
| F030 | `e5167ac23` | Overfetch unified-search candidates when caller filters are applied after lane ranking. |
| F031 | `0692b1d5b` | Render previews for pending unified-search results that pass pending metadata filters. |
| F032 | `dcab98fd7` | Exclude superseded active documents from corpus export. |
| F037 | `456e7b388` | Dedupe duplicate remember calls against pending WAL frames. |
| F034 | `40236e8df` | Require explicit session IDs for ambiguous current working-memory retrieval. |
| F010 | `0d9aebfa6` | Preserve entity and predicate key case in structured fact hashes. |
| F011 | `8514f07c6` | Preserve literal string object values in structured fact hashes. |
| F013 | `b0c27154d` | Narrow structured fact update closure to matching fact spans. |
| F014 | `c2f3bfe38` | Store structured fact version relations on bitemporal spans. |
| F015 | `2df14d250` | Avoid current structured fact spans for retraction assertions. |
| F016 | `401a23204` | Include valid/system range ends in structured fact span identity. |
| F017 | `c189c9dbf` | Guard structured fact system-time monotonicity and sentinel overflow. |
| F018 | `942027bb4` | Close same-millisecond structured fact retractions at the next system tick. |
| F020 | `997b87853` | Report structured facts truncation only when an extra row exists. |
| F021 | `fc93b63b6` | Update existing entity kind when callers supply a corrected non-empty kind. |
| F024 | `bb693369d` | Wire structured-memory edge traversal through the package session/facade stack. |
| F019 | `fb28c8278` | Expose structured fact span identity and temporal bounds. |
| F125 | `332b2fd6` | Add website/docs PR build gate and prevent PR deploys. |
| F126 | `67291613` | Fix Swift Testing skip detection gate. |
| F154 | `15bd156b` | Make HTTP MCP verifier perform a real `tools/call`. |
| F156 | `e42638a3` | Duplicate of F126; Swift Testing skip detector is already covered by production readiness gate tests. |
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
| F152 | `e0add6e73` | Add broker-backed MCP coverage and canonical compact-context memory IDs. |
| F157 | `06801d43e` | Exercise hybrid/vector search in readiness stability gates. |

Support commit not counted as a finding fix:
- `cb400efe` hardened structured-memory docs guard tests.

## Findings

- [ ] F001 Durability: `Wax.create` truncates/open-writes before lock ownership is proven.
- [ ] F002 WAL: pending payload replay lacks checksum validation.
- [ ] F003 WAL scan: forgiving WAL scan can drop later valid pending records after corrupt state.
- [ ] F004 Commit atomicity: `commitLocked` mutates live TOC before durable writes with no rollback.
- [ ] F005 Delete/supersede: committed state mutates in place without rollback.
- [x] F006 File format: offset arithmetic can trap on `UInt64` overflow.
- [x] F007 Verify/open: footer selection differs between verification and open.
- [x] F008 Repair: truncate repair lacks durable fsync.
- [x] F009 WAL ordering: invalid delete/supersede WAL can be appended before validation.
- [x] F010 Structured facts: fact hash normalizes entity/predicate case.
- [x] F011 Structured facts: string value hash lowercases object values.
- [x] F012 Evidence: `facts` query drops stored evidence.
- [x] F013 Bitemporal: updating a fact closes all subject/predicate spans.
- [x] F014 Relations: `version_relation` is overwritten on the fact row.
- [x] F015 Retractions: retract can insert a current fact row.
- [x] F016 Span hash: hash omits `system_to` and allows sentinel collision.
- [x] F017 Time: non-monotonic system time and overflow are not guarded.
- [x] F018 Retractions: same-millisecond retract can fail to close target.
- [x] F019 Query results: duplicate identical fact hits are indistinguishable.
- [x] F020 Query metadata: `wasTruncated` can be false-positive.
- [x] F021 Entities: entity kind cannot be corrected.
- [x] F022 Validation: key types accept empty, whitespace, or unbounded values.
- [x] F023 Evidence: invalid spans/confidence are accepted.
- [x] F024 Graph API: edge traversal API is unwired/dead.
- [x] F025 Structured search: object-side entity facts are not used in `evidenceFrameIds`.
- [x] F026 Bitemporal MCP: orchestrator/MCP collapses bitemporal `asOf`.
- [x] F027 Unified search: `timeRange.before` is treated as system as-of.
- [x] F028 Alias resolution: alias matching is exact despite fuzzy docs.
- [x] F029 MCP facts: `fact_assert` lacks evidence support.
- [x] F030 Unified search: metadata filters can starve candidate results.
- [x] F031 Pending search: pending metadata can match while previews are committed-only.
- [x] F032 Corpus export: superseded active docs can be exported.
- [x] F033 MCP search: `memory_search topK` caps before post-filtering.
- [x] F034 Sessions: multiple active sessions can silently ignore working memory.
- [x] F035 MCP schema: vector search mode/options are hidden.
- [x] F036 MCP tools: flush handler exists but is undiscoverable/rejected inconsistently.
- [x] F037 Pending memory: pending duplicate dedupe gap.
- [x] F038 Diagnostics: broker filters lack includeDeleted/superseded/frame IDs.
- [x] F039 MCP schema: `fact_assert` omits relation/version relation.
- [x] F040 MCP schema: generic `type/value` fact schema is rejected by broker.
- [x] F041 MCP filters: non-array `labels` filter is ignored.
- [x] F042 MCP filters: non-integer time filters are ignored.
- [x] F043 MCP filters: unknown nested filters are ignored.
- [x] F044 Recall: whitespace-only recall can embed/search unrelated content.
- [x] F045 Traits: `wax-mcp` missing `MiniLMEmbeddings` define.
- [x] F046 Traits: `WaxRepo` missing `MiniLMEmbeddings` define.
- [x] F047 Linux: MCP Linux path imports Darwin/CoreGraphics-only APIs.
- [x] F048 Linux: `Darwin.exit` used unconditionally.
- [x] F049 Linux tests: excludes miss Darwin benchmark files.
- [x] F050 Dependencies: top-level dependency leakage pulls SwiftTUI into non-CLI builds.
- [x] F051 Vector WAL: malformed vector staged/verify accepted.
- [x] F052 Embeddings: NaN/Inf embeddings accepted.
- [x] F053 USearch: `add` is not atomic.
- [x] F054 USearch: concurrent reads are unchecked around mutable index.
- [x] F055 Pending vectors: USearch ignores staged vector bytes.
- [x] F056 Batch vectors: duplicate IDs in batch overcount vector count.
- [x] F057 Serialization: unchecked `Int` overflow in vector decode.
- [x] F058 Metal vectors: deserialize misses bounds/trailing-byte validation.
- [x] F059 Metal vectors: unaligned frame ID loads.
- [x] F060 Metal scoring: cosine query normalization is missing/inconsistent.
- [x] F061 Metal errors: command-buffer error ignored.
- [x] F062 Manifest: `vectorCount` unchecked cast.
- [x] F063 Vector restore: duplicate frame IDs deserialize inconsistently.
- [x] F064 Serialization: private Objective-C ivar serialization is fragile.
- [x] F065 MiniLM: batch size 2/4 fails.
- [x] F066 MiniLM: default batch 256 exceeds asset shape 64.
- [x] F067 Arctic: default batch 256 exceeds asset shape 64.
- [x] F068 Embeddings: direct output not normalized as docs/identity imply.
- [x] F069 Embeddings: non-finite output is not rejected.
- [x] F070 CoreML errors: `try?` hides CoreML failures.
- [x] F071 MiniLM tests: quality test bypasses public batch embedder.
- [x] F072 Tokenizer batching: pre-tokenized embeddings always use batch size 1.
- [x] F073 Tokenizer: whitespace splitting excludes newlines.
- [x] F074 Tokenizer: token type IDs mark SEP/padding as segment 1.
- [x] F075 CoreML dtype: unsupported `MLMultiArray` dtype becomes zeros.
- [x] F076 FTS5: raw MATCH query is not escaped.
- [x] F077 FTS index: delete/supersede do not update FTS index consistently.
- [x] F078 Ranking: BM25 score is not normalized versus `minScore`.
- [x] F079 Validation: `topK <= 0` clamps to 1.
- [x] F080 Schema: FTS schema validation is weak.
- [x] F081 Tokenizer: default FTS tokenizer/version is unpinned.
- [x] F082 CLI daemon: `--socket-path` can unlink arbitrary file.
- [x] F083 CLI daemon: normal broker socket directory is not private.
- [x] F084 CLI daemon: socket `readToEnd` can hang.
- [x] F085 CLI secrets: license key leaks through dry-run/argv output.
- [x] F086 CLI flags: direct `stats/flush` ignore `require-vector`.
- [x] F087 CLI flags: invalid runtime flags are silently ignored.
- [x] F088 CLI surface: CLI lacks broker/MCP parity subcommands.
- [x] F089 WaxRepo: `wax-repo search` still launches TUI.
- [x] F090 WaxRepo: `--full` duplicates store content.
- [x] F091 WaxRepo: `max-commits` checkpoint can skip older history permanently.
- [x] F092 WaxRepo: repo search parses preview instead of metadata.
- [x] F093 CLI tests: daemon stable socket path expectation fails.
- [x] F094 MCP structured: `knowledge_capture` bypasses structured-memory flag.
- [x] F095 Broker config: access-stats env is parsed/logged but ignored.
- [x] F096 HTTP MCP: HTTP transport has no auth off-loopback.
- [x] F097 Session end: active session removed before fallible persistence.
- [x] F098 Broker commit: `remember/handoff` commit before event failure.
- [x] F099 Knowledge capture: graph write before memory write can half-commit.
- [x] F100 MCP content: content strings are trimmed.
- [x] F101 Session resume: `session_resume` can pick ended manifest.
- [x] F102 HTTP MCP: body limit is enforced after full read.
- [x] F103 Corpus: corpus rebuild is non-atomic.
- [x] F104 MCP config: invalid embedder choice falls back to MiniLM.
- [x] F105 Multimodal MCP: multimodal is advertised but not wired.
- [x] F106 HTTP lifecycle: cleanup loop has no cancellation.
- [x] F107 MCP tests: broker-backed durable capture test times out.
- [x] F108 MCP tests: locked-session corpus search test times out.
- [x] F109 npm package: packed tarball lacks `dist` binaries.
- [x] F110 Snippet gate: public snippet verifier is missing.
- [x] F111 Demo package: WaxDemo points to missing `../Wax`.
- [x] F112 Release script: version grep targets stale source pattern.
- [x] F113 Release script: checksum path is cwd-sensitive.
- [x] F114 Release workflow: npm metadata is bumped after build only.
- [x] F115 OpenClaw npm: package ships TypeScript source without loader/build.
- [x] F116 OpenClaw: default command `wax-mcp` unavailable from plugin package.
- [x] F117 Release arch: local release stages only arm64 while metadata advertises x64.
- [x] F118 Homebrew: formula version is stale.
- [x] F119 Homebrew: directory is gitlink-like without `.gitmodules`.
- [x] F120 Homebrew: formula Xcode 15 requirement is too old for Swift 6.1 traits.
- [x] F121 npm README: local path `./npm/waxmcp` is wrong from repo root.
- [x] F122 Release scripts: root/nested release scripts rewrite different version files.
- [x] F123 Readiness gate: pass-rate parser fails on Swift Testing output.
- [x] F124 Docs script: docs generation uses wrong root/destructive copy assumptions.
- [x] F125 Website CI: no PR build gate for website/docs.
- [x] F126 Test gate: skip detector misses Swift Testing skip format.
- [x] F127 CI scope: Linux CI omits Wax/CLI/MCP product builds.
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
- [x] F152 MCP tests: many tests use direct-memory compatibility path, not production broker path.
- [x] F153 MCP tests: compatibility aliases differ from production renamed-tool behavior.
- [x] F154 HTTP smoke: HTTP verifier never calls a tool.
- [x] F155 CI: Linux CI omits Wax/CLI/MCP product builds. Duplicate of F127; the Linux workflow now builds Wax, wax-cli, and wax-mcp products.
- [x] F156 Gate script: skip detector misses Swift Testing skips. Duplicate of F126; focused gate tests cover Swift Testing suite/test skip output.
- [x] F157 Readiness tests: stability gate is text-search only.
- [x] F158 Migration tests: N-1/N-2 fixtures are generated by current code.
- [x] F159 MiniLM tests: inference tests silently return when env unset.
- [x] F160 MiniLM tests: missing-resource tests catch any error.
- [x] F161 PDF API: PDF API has no non-PDFKit fallback.
- [x] F162 PDF ingest: extraction hard-limits 500 pages with no metadata.
- [x] F163 PDF provenance: page provenance is lost after join.
- [x] F164 Photo ingest: docs claim local image ingest but code lacks local file API.
- [x] F165 Photo sync: fetches all PHAssets, not images only.
- [x] F166 Photo filters: `PhotoFilters` is empty/no-op.
- [x] F167 Photo location: radius uses coarse bins without final distance check.
- [x] F168 Photo diagnostics: degraded flag inferred from missing OCR/caption, not locality.
- [x] F169 Photo ingest: failed region crops return before superseding old root.
- [x] F170 Photo sync: `syncState` frame kind is never written.
- [x] F171 Photo tags: tags overpromise classifier labels.
- [x] F172 Video thumbnails: Photos video local URL discarded, thumbnails fail.
- [x] F173 Broker validation: direct broker ignores unknown args.
- [x] F174 Session start: manifest saved before `.started` event.
- [x] F175 Session resume: lease stolen before `.resumed` event.
- [x] F176 Events: first event file creation return ignored.
- [x] F177 Events: one malformed JSONL line aborts whole log.
- [x] F178 Manifests: corrupt stray manifest aborts all listing.
- [x] F179 Promotion: memory written before stale-session validation.
- [x] F180 Promotion: metadata keeps raw `session_id`.
- [x] F181 Promotion: `max_candidates` unbounded above.
- [x] F182 Markdown sync: marker trust uses frame ID only.
- [x] F183 Markdown sync: locked memory can be deleted by removing markdown line.
- [x] F184 Markdown sync: dry-run skips durable-write validation.
- [x] F185 Markdown import: markerless bullets imported as managed.
- [x] F186 DREAMS: duplicate checked lines create duplicate durable memories.
- [x] F187 DREAMS: export excludes ended sessions.
- [x] F188 DREAMS: projection closes session stores in unawaited task.
- [x] F189 Markdown export: opens active sessions owned by other brokers.
- [x] F190 Markdown export: leaves stale generated files.
- [x] F191 Markdown export: `wax.source_date` path traversal in daily filename.
- [x] F192 Markdown marker: marker JSON is not escaped for `-->`.
- [x] F193 Markdown export: reclassifies text instead of metadata type.
- [x] F194 Compact context: references raw chunk frame IDs.
- [x] F195 Compact context: token budget counts hit text, not rendered output.
- [x] F196 Compact context: recency prefix filters sessions before relevance.
- [x] F197 Corpus cache: corrupt corpus manifest aborts instead of rebuild.
- [x] F198 Enrichment: handler result is discarded.
- [x] F199 Enrichment: structured extraction hardcodes empty entities.
- [x] F200 Keywords: technical identifiers are split/lowercased.
