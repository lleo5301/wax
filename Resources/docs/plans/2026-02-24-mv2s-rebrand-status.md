# MV2S → Wax Rebrand: Status Report & Continuation Plan

> **Date:** 2026-02-24
> **Branch:** `mcp-release`
> **Original plan:** `docs/plans/2026-02-23-mv2s-to-wax-rebrand.md`
> **Goal:** Completely rebrand all "MV2S" identifiers to "Wax"/"WAX1" across four layers — binary magic bytes, Swift type names, `.mv2s` file extension, and documentation.

---

## Executive Summary

Layers 1 and 2 (binary magic bytes and Swift type renames) are **complete and committed**. Layer 3 (file extension `.mv2s` → `.wax`) and Layer 4 (documentation) are **pending — zero work started**. The build is clean (excluding a pre-existing `AsyncRunner.swift` concurrency error unrelated to the rebrand). All 311 WaxCoreTests + 415 WaxIntegrationTests passed at baseline. **No tests have been re-run after the Layer 2 type renames** — this is a gap that must be addressed before continuing.

---

## What's Done

### Layer 1 — Binary Magic Bytes ✅ COMMITTED

**Commit:** `ec60dd2 refactor: rebrand binary magic bytes MV2S→WAX1, MV2SFOOT→WAX1FOOT`

| File | Change |
|------|--------|
| `Sources/WaxCore/Constants.swift` | Header magic `0x4D,0x56,0x32,0x53` → `0x57,0x41,0x58,0x31` (ASCII "MV2S" → "WAX1") |
| `Sources/WaxCore/Constants.swift` | Footer magic `MV2SFOOT` → `WAX1FOOT` (8 bytes) |
| `Sources/WaxCore/Constants.swift` | Doc comments updated: spec label, WAL record label |
| `Tests/WaxCoreTests/SmokeTests.swift` | Assertions updated: `"WAX1"` and `"WAX1FOOT"` |

**Verification:** TDD red-green confirmed. `constantsAreCorrect` test failed with old values, passed with new.

### Layer 2 — Swift Type Renames ✅ DONE (UNCOMMITTED)

All type renames are applied and build-verified, but **not committed** yet.

| Old Name | New Name | Status |
|----------|----------|--------|
| `MV2SFooter` | `WaxFooter` | ✅ All references replaced, file renamed |
| `MV2SHeaderPage` | `WaxHeaderPage` | ✅ All references replaced, file renamed |
| `MV2STOC` | `WaxTOC` | ✅ All references replaced, file renamed |
| `MV2SEnums.swift` | `WaxEnums.swift` | ✅ File renamed (no internal type name change needed) |
| `MV2STOCTests.swift` | `WaxTOCTests.swift` | ✅ File renamed, references updated |

**Files touched (type renames):**

Sources:
- `Sources/WaxCore/FileFormat/WaxFooter.swift` (was MV2SFooter.swift)
- `Sources/WaxCore/FileFormat/WaxHeaderPage.swift` (was MV2SHeaderPage.swift)
- `Sources/WaxCore/FileFormat/WaxTOC.swift` (was MV2STOC.swift)
- `Sources/WaxCore/FileFormat/WaxEnums.swift` (was MV2SEnums.swift)
- `Sources/WaxCore/Wax.swift`
- `Sources/WaxCore/FileFormat/FooterScanner.swift`
- `Sources/WaxCore/WaxCore.docc/Documentation.md`

Tests:
- `Tests/WaxCoreTests/HeaderFooterTests.swift`
- `Tests/WaxCoreTests/CrashRecoveryTests.swift`
- `Tests/WaxCoreTests/FooterScannerTests.swift`
- `Tests/WaxCoreTests/ProductionReadinessRecoveryTests.swift`
- `Tests/WaxCoreTests/OpenValidationTests.swift`
- `Tests/WaxCoreTests/DurabilityRegressionTests.swift`
- `Tests/WaxCoreTests/VerificationTests.swift`
- `Tests/WaxCoreTests/WaxTOCTests.swift` (was MV2STOCTests.swift)
- `Tests/WaxIntegrationTests/WALCompactionBenchmarks.swift`

Demos:
- `WaxDemo/Sources/WaxDemo/main.swift`
- `WaxDemo/Sources/WaxDemoCorruptTOC/main.swift`
- `WaxDemo/Sources/WaxDemoMultiFooter/main.swift`

**Build status:** Clean (no errors related to renames).

---

## Known Issues / Gaps to Investigate

### 1. MISSED MV2SFooter references in two test files

The grep audit found **stale `MV2SFooter` type names** in two files that were NOT updated during Layer 2:

| File | Lines | Issue |
|------|-------|-------|
| `Tests/WaxCoreTests/FooterScannerEdgeCaseTests.swift` | 25, 30, 33, 42, 53, 60, 68, 80, 87, 157, 213, 222, 293, 305, 321 | **15 occurrences** of `MV2SFooter` still present |
| `Tests/WaxCoreTests/OpenValidationTests.swift` | 32, 100 | **2 occurrences** of `MV2SFooter` still present |

**Impact:** These files will **fail to compile** once the old `MV2SFooter` type is fully gone from the module. The build currently passes because Swift resolves the renamed type from the new `WaxFooter.swift` file — but the *source text* still says `MV2SFooter` in these test files. **Wait — actually MV2SFooter no longer exists as a type.** So either:
- (a) These files aren't being compiled (possible if excluded from test targets), or
- (b) The build error is masked somehow.

**ACTION REQUIRED:** These must be fixed with `replace_all` before the Layer 2 commit.

### 2. Layer 2 not yet committed

All the renames are applied but there's no commit yet. The plan called for:
```
git commit -m "refactor: rename MV2SHeaderPage→WaxHeaderPage, MV2STOC→WaxTOC, MV2SFooter→WaxFooter"
```

### 3. Full test suite not re-run after Layer 2

The baseline (311 + 415 tests) was green before any changes. After Layer 2 type renames, only `swift build` was verified — the full test suite was NOT re-run. A fatalError was observed during the last attempted test run but we didn't get full output.

**ACTION REQUIRED:** Run `swift test --filter WaxCoreTests` and `swift test --filter WaxIntegrationTests` after fixing the missed references.

### 4. Pre-existing build error in AsyncRunner.swift

```
Sources/WaxCLI/AsyncRunner.swift:12:9: error: passing closure as a 'sending' parameter risks causing data races
```

This is **unrelated** to the rebrand and exists on the branch independently. It doesn't block the rebrand work.

---

## What's Pending

### Layer 3 — File Extension (.mv2s → .wax) — NOT STARTED

**18 source files + 6 test files + 1 demo file** need `.mv2s` → `.wax` string replacement.

#### Source files (18):
| File | Type of reference |
|------|-------------------|
| `Sources/Wax/Orchestrator/MemoryOrchestrator+Maintenance.swift` | `appendingPathExtension("mv2s")`, `hasSuffix(".mv2s")` — **runtime-critical** |
| `Sources/WaxCLI/StoreOptions.swift` | Default path strings |
| `Sources/WaxCLI/StoreSession.swift` | Default path strings |
| `Sources/WaxCLI/WaxCLICommand.swift` | Help text |
| `Sources/WaxMCPServer/main.swift` | Default paths + help strings |
| `Sources/WaxRepo/Commands/IndexCommand.swift` | Default paths |
| `Sources/WaxRepo/Commands/SearchCommand.swift` | Default paths |
| `Sources/WaxRepo/Commands/StatsCommand.swift` | Default paths |
| `Sources/WaxRepo/Store/RepoStore.swift` | Doc comments |
| `Sources/WaxCore/Wax.swift` | File extension references |
| `Sources/WaxCore/WaxCore.docc/Documentation.md` | Doc references |
| `Sources/WaxCore/WaxCore.docc/Articles/ConcurrencyModel.md` | Doc references |
| `Sources/WaxCore/WaxCore.docc/Articles/FileFormat.md` | Doc references |
| `Sources/WaxCore/WaxCore.docc/Articles/GettingStarted.md` | Doc references |
| `Sources/Wax/Wax.docc/Articles/GettingStarted.md` | Doc references |
| `Sources/Wax/Wax.docc/Articles/Architecture.md` | Doc references |
| `Sources/Wax/Wax.docc/Articles/MemoryOrchestrator.md` | Doc references |
| `Sources/WaxTextSearch/WaxTextSearch.docc/Documentation.md` | Doc references |

#### Test files (6):
| File |
|------|
| `Tests/WaxCoreTests/ProductionReadinessRecoveryTests.swift` |
| `Tests/WaxIntegrationTests/WALCompactionBenchmarks.swift` |
| `Tests/WaxIntegrationTests/VectorSearchEngineTests.swift` |
| `Tests/WaxIntegrationTests/TextSearchEngineTests.swift` |
| `Tests/WaxIntegrationTests/OptimizationComparisonBenchmark.swift` |
| `Tests/WaxIntegrationTests/StructuredMemoryWaxPersistenceTests.swift` |

#### Demo files (1):
| File |
|------|
| `WaxDemo/Sources/WaxDemo/main.swift` |

**Note:** The original plan listed additional test files (TempFiles.swift helpers, WaxMCPServerTests, etc.) and the WaxCrashHarness. These should be re-verified via grep during execution.

### Layer 4 — Documentation & Skills — NOT STARTED

#### Remaining `MV2S` prose references in source code:
| File | Line | Content |
|------|------|---------|
| `Sources/WaxCore/Checksum/SHA256Checksum.swift` | 4 | `/// Simple SHA-256 wrapper used by MV2S codecs.` |
| `Sources/WaxCore/BinaryCodec/BinaryDecoder.swift` | 3 | `/// Deterministic binary decoder for MV2S primitives.` |

#### DocC articles with MV2S references:
| File | Issues |
|------|--------|
| `Sources/WaxCore/WaxCore.docc/Articles/FileFormat.md` | Title "The MV2S File Format", magic byte tables, prose — heavy rewrite needed |

#### External documentation:
| File | Occurrences |
|------|-------------|
| `README.md` | 5 occurrences: code examples with `brain.mv2s`, prose "`.mv2s` file" |
| `skills/public/wax/SKILL.md` | Multiple: code examples, prose |
| `skills/public/wax/references/constraints.md` | Type names, extension references |

---

## Continuation Prompt

Use this prompt to resume the work in a new session:

```
Continue the MV2S → Wax rebrand on the `mcp-release` branch.

**Context:**
- Plan: `docs/plans/2026-02-23-mv2s-to-wax-rebrand.md`
- Status: `docs/plans/2026-02-24-mv2s-rebrand-status.md`
- Layer 1 (magic bytes): COMMITTED as ec60dd2
- Layer 2 (type renames): APPLIED but NOT COMMITTED
- Layer 3 (file extension .mv2s → .wax): NOT STARTED
- Layer 4 (documentation): NOT STARTED

**Immediate actions needed (in order):**

1. FIX MISSED REFERENCES: Replace `MV2SFooter` in:
   - `Tests/WaxCoreTests/FooterScannerEdgeCaseTests.swift` (15 occurrences)
   - `Tests/WaxCoreTests/OpenValidationTests.swift` (2 occurrences)

2. RUN FULL TEST SUITE to verify Layer 2:
   ```
   swift test --filter WaxCoreTests
   swift test --filter WaxIntegrationTests
   ```

3. COMMIT Layer 2:
   ```
   git add Sources/WaxCore/FileFormat/ Sources/WaxCore/Wax.swift \
           Sources/WaxCore/FileFormat/FooterScanner.swift \
           Sources/WaxCore/WaxCore.docc/ \
           Tests/WaxCoreTests/ Tests/WaxIntegrationTests/ WaxDemo/
   git commit -m "refactor: rename MV2SHeaderPage→WaxHeaderPage, MV2STOC→WaxTOC, MV2SFooter→WaxFooter"
   ```

4. EXECUTE Layer 3 (.mv2s → .wax) following the plan Tasks 7-11.
   - Start with runtime-critical `MemoryOrchestrator+Maintenance.swift`
   - Then CLI/MCP defaults
   - Then test TempFiles helpers
   - Then remaining test files
   - Full grep verification: `grep -rn '\.mv2s' Sources/ Tests/ WaxDemo/`

5. EXECUTE Layer 4 (documentation) following Tasks 12-14.
   - Update SHA256Checksum.swift and BinaryDecoder.swift doc comments
   - Update FileFormat.md DocC article (heavy — title, tables, prose)
   - Update README.md (5 occurrences)
   - Update SKILL.md and constraints.md
   - Full grep verification: `grep -rn 'MV2S' Sources/ Tests/ WaxDemo/ docs/ skills/ README.md`

6. FINAL VERIFICATION (Task 15):
   - `grep -rn '\.mv2s' Sources/ Tests/ WaxDemo/ docs/ README.md skills/` → expect NO output
   - `grep -rn 'MV2SHeaderPage\|MV2STOC\|MV2SFooter\|MV2SEnums' Sources/ Tests/ WaxDemo/ docs/` → expect NO output
   - `grep -rn '"MV2S"\|"MV2SFOOT"' Sources/ Tests/` → expect NO output
   - `swift build` → clean
   - `swift test --filter WaxCoreTests` → all pass
   - `swift test --filter WaxIntegrationTests` → all pass
   - `swift test --filter WaxMCPServerTests` → all pass

**Known pre-existing issue:** `AsyncRunner.swift:12` has a Swift concurrency error unrelated to rebrand. Ignore it.

**Approach:** Use `superpowers:executing-plans` skill. Work in batches of 3 tasks. TDD red-green cycle. Commit at each layer boundary.
```

---

## File-Level Audit Summary

### Files with NO remaining MV2S references (clean):
- `Sources/WaxCore/Constants.swift` ✅
- `Sources/WaxCore/FileFormat/WaxFooter.swift` ✅
- `Sources/WaxCore/FileFormat/WaxHeaderPage.swift` ✅
- `Sources/WaxCore/FileFormat/WaxTOC.swift` ✅
- `Sources/WaxCore/FileFormat/WaxEnums.swift` ✅
- `Sources/WaxCore/FileFormat/FooterScanner.swift` ✅
- `Tests/WaxCoreTests/SmokeTests.swift` ✅
- `Tests/WaxCoreTests/HeaderFooterTests.swift` ✅
- `Tests/WaxCoreTests/CrashRecoveryTests.swift` ✅
- `Tests/WaxCoreTests/WaxTOCTests.swift` ✅
- All demo files (type renames only) ✅

### Files with remaining MV2S type names:
- `Tests/WaxCoreTests/FooterScannerEdgeCaseTests.swift` — 15 `MV2SFooter`
- `Tests/WaxCoreTests/OpenValidationTests.swift` — 2 `MV2SFooter`

### Files with remaining `.mv2s` extension:
- 18 source files (see Layer 3 table above)
- 6 test files
- 1 demo file

### Files with remaining `MV2S` in prose/comments:
- 2 source files (SHA256Checksum, BinaryDecoder)
- 1 DocC article (FileFormat.md — heavy)
- 5 other DocC articles (lighter)
- README.md (5 occurrences)
- 2 skill files
