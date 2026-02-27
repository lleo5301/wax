# MV2S → Wax Rebrand Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Completely rebrand all "MV2S" identifiers to "Wax"/"WAX1" across three layers — binary magic bytes, Swift type names, and the `.mv2s` file extension — with no backward compatibility required.

**Architecture:** Work layer-by-layer in dependency order: constants first (everything reads from them), then type renames (compiler will catch any misses), then file extension strings (grep-verified), then docs. Each layer is independently verifiable and committable.

**Tech Stack:** Swift Package Manager, `swift build`, `swift test`, `Edit` tool with `replace_all`, `Bash mv` for file renames.

---

## Pre-Flight: Confirm Tests Are Green

Before touching anything, establish a clean baseline.

### Task 0: Baseline test run

**Files:**
- No changes

**Step 1: Run WaxCoreTests**

```bash
swift test --filter WaxCoreTests 2>&1 | tail -20
```
Expected: All tests pass (or note which ones are already failing so we don't blame ourselves).

**Step 2: Run WaxIntegrationTests**

```bash
swift test --filter WaxIntegrationTests 2>&1 | tail -20
```
Expected: All tests pass.

**Step 3: Record the baseline**

Note any pre-existing failures. Do not proceed if there are unexpected failures — resolve them first.

---

## Layer 1 — Binary Magic Bytes

### Task 1: Update Constants.swift magic bytes and doc comments

**Files:**
- Modify: `Sources/WaxCore/Constants.swift`

**Context:** This is the single source of truth for magic bytes. All validators read from here. Change it once and the whole binary format updates.

**Step 1: Read the current file**

Open `Sources/WaxCore/Constants.swift` and confirm:
- Header magic is `Data([0x4D, 0x56, 0x32, 0x53])` (ASCII "MV2S")
- Footer magic is `Data([0x4D, 0x56, 0x32, 0x53, 0x46, 0x4F, 0x4F, 0x54])` (ASCII "MV2SFOOT")

**Step 2: Replace header magic bytes**

Using Edit with `replace_all: false` (exact single occurrence):

Old:
```swift
Data([0x4D, 0x56, 0x32, 0x53])
```
New:
```swift
Data([0x57, 0x41, 0x58, 0x31])
```
(ASCII "WAX1")

**Step 3: Replace footer magic bytes**

Old:
```swift
Data([0x4D, 0x56, 0x32, 0x53, 0x46, 0x4F, 0x4F, 0x54])
```
New:
```swift
Data([0x57, 0x41, 0x58, 0x31, 0x46, 0x4F, 0x4F, 0x54])
```
(ASCII "WAX1FOOT")

**Step 4: Update doc comment — header magic label**

Old:
```swift
/// Header magic: "MV2S" (4 bytes)
```
New:
```swift
/// Header magic: "WAX1" (4 bytes)
```

**Step 5: Update doc comment — footer magic label**

Old:
```swift
/// Footer magic: "MV2SFOOT" (8 bytes)
```
New:
```swift
/// Footer magic: "WAX1FOOT" (8 bytes)
```

**Step 6: Update WAL record header size comment**

Old:
```swift
/// WAL record header size: 48 bytes (fixed for MV2S v1).
```
New:
```swift
/// WAL record header size: 48 bytes (fixed for Wax v1).
```

**Step 7: Update module-level doc comment**

Old:
```swift
/// Constants matching `MV2S_SPEC.md` (MV2S v1.0).
```
New:
```swift
/// Constants matching `WAX_SPEC.md` (Wax v1.0).
```

**Step 8: Build to confirm no compile errors**

```bash
swift build 2>&1 | grep -E "error:|warning:" | head -20
```
Expected: No new errors.

---

### Task 2: Update SmokeTests.swift magic byte assertions

**Files:**
- Modify: `Tests/WaxCoreTests/SmokeTests.swift`

**Context:** These tests assert the exact magic byte values. They will fail after Task 1 changes — that's expected and correct. Now we update the expected values to match the new constants.

**Step 1: Read the file first**

Open `Tests/WaxCoreTests/SmokeTests.swift` and find the two magic byte assertions.

**Step 2: Replace header magic assertion**

Old:
```swift
#expect(Constants.magic == "MV2S".data(using: .utf8)!)
```
New:
```swift
#expect(Constants.magic == "WAX1".data(using: .utf8)!)
```

**Step 3: Replace footer magic assertion**

Old:
```swift
#expect(Constants.footerMagic == "MV2SFOOT".data(using: .utf8)!)
```
New:
```swift
#expect(Constants.footerMagic == "WAX1FOOT".data(using: .utf8)!)
```

**Step 4: Run SmokeTests to confirm they pass**

```bash
swift test --filter WaxCoreTests/SmokeTests 2>&1 | tail -10
```
Expected: PASS.

**Step 5: Commit Layer 1**

```bash
git add Sources/WaxCore/Constants.swift Tests/WaxCoreTests/SmokeTests.swift
git commit -m "refactor: rebrand binary magic bytes MV2S→WAX1, MV2SFOOT→WAX1FOOT"
```

---

## Layer 2 — Type Renames

### Task 3: Rename MV2SFooter → WaxFooter across all files

**Files:**
- Modify (rename): `Sources/WaxCore/FileFormat/MV2SFooter.swift` → `WaxFooter.swift`
- Modify: `Sources/WaxCore/Wax.swift`
- Modify: `Sources/WaxCore/FileFormat/FooterScanner.swift`
- Modify: `Tests/WaxCoreTests/HeaderFooterTests.swift`
- Modify: `Tests/WaxCoreTests/CrashRecoveryTests.swift`

**Context:** Do type-name replacement in all files *before* the `mv` rename so every file is consistent. The compiler will catch any missed occurrences.

**Step 1: Replace in definition file**

In `Sources/WaxCore/FileFormat/MV2SFooter.swift`, use Edit `replace_all: true`:
- `MV2SFooter` → `WaxFooter`

Also update the doc comment on line 65:
Old: `/// MV2S v1 semantics:`
New: `/// Wax v1 semantics:`

**Step 2: Replace in Wax.swift**

In `Sources/WaxCore/Wax.swift`, use Edit `replace_all: true`:
- `MV2SFooter` → `WaxFooter`

**Step 3: Replace in FooterScanner.swift**

In `Sources/WaxCore/FileFormat/FooterScanner.swift`, use Edit `replace_all: true`:
- `MV2SFooter` → `WaxFooter`

Also update the doc comment on line 10:
Old: `/// Scans for the most recent valid MV2S footer.`
New: `/// Scans for the most recent valid Wax footer.`

**Step 4: Replace in HeaderFooterTests.swift**

In `Tests/WaxCoreTests/HeaderFooterTests.swift`, use Edit `replace_all: true`:
- `MV2SFooter` → `WaxFooter`

**Step 5: Replace in CrashRecoveryTests.swift**

In `Tests/WaxCoreTests/CrashRecoveryTests.swift`, use Edit `replace_all: true`:
- `MV2SFooter` → `WaxFooter`

**Step 6: Rename the file**

```bash
mv Sources/WaxCore/FileFormat/MV2SFooter.swift Sources/WaxCore/FileFormat/WaxFooter.swift
```

**Step 7: Build to confirm**

```bash
swift build 2>&1 | grep "error:" | head -20
```
Expected: No errors mentioning `MV2SFooter`.

---

### Task 4: Rename MV2SHeaderPage → WaxHeaderPage across all files

**Files:**
- Modify (rename): `Sources/WaxCore/FileFormat/MV2SHeaderPage.swift` → `WaxHeaderPage.swift`
- Modify: `Sources/WaxCore/Wax.swift`
- Modify: `Tests/WaxCoreTests/HeaderFooterTests.swift`
- Modify: `Tests/WaxCoreTests/CrashRecoveryTests.swift`
- Modify: `WaxDemo/Sources/WaxDemo/main.swift`
- Modify: `WaxDemo/Sources/WaxDemoCorruptTOC/main.swift`
- Modify: `WaxDemo/Sources/WaxDemoMultiFooter/main.swift`

**Step 1: Replace in definition file**

In `Sources/WaxCore/FileFormat/MV2SHeaderPage.swift`, use Edit `replace_all: true`:
- `MV2SHeaderPage` → `WaxHeaderPage`

**Step 2: Replace in Wax.swift**

In `Sources/WaxCore/Wax.swift`, use Edit `replace_all: true`:
- `MV2SHeaderPage` → `WaxHeaderPage`

**Step 3: Replace in HeaderFooterTests.swift**

In `Tests/WaxCoreTests/HeaderFooterTests.swift`, use Edit `replace_all: true`:
- `MV2SHeaderPage` → `WaxHeaderPage`

**Step 4: Replace in CrashRecoveryTests.swift**

In `Tests/WaxCoreTests/CrashRecoveryTests.swift`, use Edit `replace_all: true`:
- `MV2SHeaderPage` → `WaxHeaderPage`

**Step 5: Replace in all three demo mains**

In each of:
- `WaxDemo/Sources/WaxDemo/main.swift`
- `WaxDemo/Sources/WaxDemoCorruptTOC/main.swift`
- `WaxDemo/Sources/WaxDemoMultiFooter/main.swift`

Use Edit `replace_all: true`:
- `MV2SHeaderPage` → `WaxHeaderPage`

**Step 6: Rename the file**

```bash
mv Sources/WaxCore/FileFormat/MV2SHeaderPage.swift Sources/WaxCore/FileFormat/WaxHeaderPage.swift
```

**Step 7: Build to confirm**

```bash
swift build 2>&1 | grep "error:" | head -20
```
Expected: No errors mentioning `MV2SHeaderPage`.

---

### Task 5: Rename MV2STOC → WaxTOC across all files

**Files:**
- Modify (rename): `Sources/WaxCore/FileFormat/MV2STOC.swift` → `WaxTOC.swift`
- Modify (rename): `Tests/WaxCoreTests/MV2STOCTests.swift` → `WaxTOCTests.swift`
- Modify: `Sources/WaxCore/Wax.swift`
- Modify: `WaxDemo/Sources/WaxDemo/main.swift`
- Modify: `WaxDemo/Sources/WaxDemoCorruptTOC/main.swift`
- Modify: `WaxDemo/Sources/WaxDemoMultiFooter/main.swift`

**Step 1: Replace in definition file**

In `Sources/WaxCore/FileFormat/MV2STOC.swift`, use Edit `replace_all: true`:
- `MV2STOC` → `WaxTOC`

**Step 2: Replace in Wax.swift**

In `Sources/WaxCore/Wax.swift`, use Edit `replace_all: true`:
- `MV2STOC` → `WaxTOC`

**Step 3: Replace in MV2STOCTests.swift**

In `Tests/WaxCoreTests/MV2STOCTests.swift`, use Edit `replace_all: true`:
- `MV2STOC` → `WaxTOC`

**Step 4: Replace in demo mains** (if any usages exist — verify first)

Check each demo main for `MV2STOC` and replace if found.

**Step 5: Rename the two files**

```bash
mv Sources/WaxCore/FileFormat/MV2STOC.swift Sources/WaxCore/FileFormat/WaxTOC.swift
mv Tests/WaxCoreTests/MV2STOCTests.swift Tests/WaxCoreTests/WaxTOCTests.swift
```

**Step 6: Rename MV2SEnums.swift** (enums inside have no MV2S prefix; file-only rename)

```bash
mv Sources/WaxCore/FileFormat/MV2SEnums.swift Sources/WaxCore/FileFormat/WaxEnums.swift
```

**Step 7: Build to confirm**

```bash
swift build 2>&1 | grep "error:" | head -20
```
Expected: Zero errors.

**Step 8: Run WaxCoreTests**

```bash
swift test --filter WaxCoreTests 2>&1 | tail -20
```
Expected: All pass.

**Step 9: Commit Layer 2**

```bash
git add Sources/WaxCore/FileFormat/ Sources/WaxCore/Wax.swift \
        Sources/WaxCore/FileFormat/FooterScanner.swift \
        Tests/WaxCoreTests/ WaxDemo/
git commit -m "refactor: rename MV2SHeaderPage→WaxHeaderPage, MV2STOC→WaxTOC, MV2SFooter→WaxFooter"
```

---

### Task 6: Update remaining MV2S doc comments

**Files:**
- Modify: `Sources/WaxCore/Checksum/SHA256Checksum.swift` (line 4)
- Modify: `Sources/WaxCore/BinaryCodec/BinaryDecoder.swift` (line 3)

**Context:** These are doc comments only — no runtime behavior. Verify no type names are lurking.

**Step 1: Update SHA256Checksum.swift**

Old:
```swift
/// Simple SHA-256 wrapper used by MV2S codecs.
```
New:
```swift
/// Simple SHA-256 wrapper used by Wax codecs.
```

**Step 2: Update BinaryDecoder.swift**

Old:
```swift
/// Deterministic binary decoder for MV2S primitives.
```
New:
```swift
/// Deterministic binary decoder for Wax primitives.
```

**Step 3: Grep to confirm no MV2S type names remain in Sources/**

```bash
grep -rn 'MV2SHeaderPage\|MV2STOC\|MV2SFooter\|MV2SEnums' Sources/ Tests/ WaxDemo/
```
Expected: No output.

**Step 4: Grep to confirm no old magic byte strings remain**

```bash
grep -rn '"MV2S"\|"MV2SFOOT"' Sources/ Tests/
```
Expected: No output.

**Step 5: Commit**

```bash
git add Sources/WaxCore/Checksum/SHA256Checksum.swift \
        Sources/WaxCore/BinaryCodec/BinaryDecoder.swift
git commit -m "docs: update remaining MV2S→Wax doc comments in codec files"
```

---

## Layer 3 — File Extension (.mv2s → .wax)

### Task 7: Update functional extension references in MemoryOrchestrator

**Files:**
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator+Maintenance.swift` (lines 437, 607)

**Context:** These are runtime-critical. Line 437 creates new files; line 607 filters files by extension. Both must be correct or the orchestrator will stop finding/creating `.wax` files.

**Step 1: Read lines 430–615 of the file to see exact context**

Verify there are exactly two occurrences: one `appendingPathExtension("mv2s")` and one `hasSuffix(".mv2s")`.

**Step 2: Replace appendingPathExtension**

Old:
```swift
.appendingPathExtension("mv2s")
```
New:
```swift
.appendingPathExtension("wax")
```

**Step 3: Replace hasSuffix**

Old:
```swift
name.hasSuffix(".mv2s")
```
New:
```swift
name.hasSuffix(".wax")
```

**Step 4: Also update doc comment on line 172 if it references `.mv2s`**

Old (approximate):
```swift
/// ... .mv2s ...
```
New:
```swift
/// ... .wax ...
```

**Step 5: Build**

```bash
swift build 2>&1 | grep "error:" | head -10
```

---

### Task 8: Update CLI and MCP server default paths

**Files:**
- Modify: `Sources/WaxCLI/main.swift`
- Modify: `Sources/WaxMCPServer/main.swift`
- Modify: `Sources/WaxRepo/Commands/IndexCommand.swift`
- Modify: `Sources/WaxRepo/Commands/SearchCommand.swift`
- Modify: `Sources/WaxRepo/Commands/StatsCommand.swift`
- Modify: `Sources/WaxRepo/Store/RepoStore.swift`

**Context:** These are default path strings and help text shown to users. Incorrect extension here means the CLI defaults to a wrong filename — user-visible bug.

**Step 1: Update WaxCLI/main.swift**

In `Sources/WaxCLI/main.swift`, use Edit `replace_all: true`:
- `".mv2s"` → `".wax"`
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`
- `hasSuffix(".mv2s")` → `hasSuffix(".wax")`
- Any path string literal ending in `.mv2s`

**Step 2: Update WaxMCPServer/main.swift**

In `Sources/WaxMCPServer/main.swift`, use Edit `replace_all: true`:
- `".mv2s"` → `".wax"` (3 default paths + 3 help strings)

**Step 3: Update WaxRepo commands**

In each of `IndexCommand.swift`, `SearchCommand.swift`, `StatsCommand.swift`:
Use Edit `replace_all: true`:
- `".mv2s"` → `".wax"`
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`

**Step 4: Update RepoStore.swift doc comment**

In `Sources/WaxRepo/Store/RepoStore.swift`:
Replace any doc comment prose referencing `.mv2s` → `.wax`.

**Step 5: Build**

```bash
swift build 2>&1 | grep "error:" | head -10
```
Expected: Clean.

---

### Task 9: Update demo and harness files

**Files:**
- Modify: `WaxDemo/Sources/WaxDemo/main.swift`
- Modify: `WaxDemo/Sources/WaxDemoCorruptTOC/main.swift`
- Modify: `WaxDemo/Sources/WaxDemoMultiFooter/main.swift`
- Modify: `Sources/WaxCrashHarness/main.swift` (line 146)

**Step 1: Update each demo main**

In each of the three demo mains, use Edit `replace_all: true`:
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`
- Any `".mv2s"` string → `".wax"`

**Step 2: Update WaxCrashHarness/main.swift line 146**

Read the file around line 146. Apply same replacement.

**Step 3: Build**

```bash
swift build 2>&1 | grep "error:" | head -10
```

---

### Task 10: Update test files — TempFiles helpers first

**Files:**
- Modify: `Tests/WaxCoreTests/TempFiles.swift`
- Modify: `Tests/WaxIntegrationTests/TempFiles.swift`

**Context:** TempFiles.swift is a shared helper used by many tests. Updating it first means all downstream tests automatically get `.wax` temp files. These are high-impact; get them right first.

**Step 1: Update WaxCoreTests/TempFiles.swift**

Use Edit `replace_all: true`:
- `".mv2s"` → `".wax"`
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`

**Step 2: Update WaxIntegrationTests/TempFiles.swift**

Same replacements.

**Step 3: Run a quick test to verify helpers work**

```bash
swift test --filter WaxCoreTests/SmokeTests 2>&1 | tail -10
```

---

### Task 11: Update remaining test files (.mv2s occurrences)

**Files:**
- Modify: `Tests/WaxCoreTests/ProductionReadinessRecoveryTests.swift`
- Modify: `Tests/WaxIntegrationTests/TextSearchEngineTests.swift`
- Modify: `Tests/WaxIntegrationTests/VectorSearchEngineTests.swift`
- Modify: `Tests/WaxIntegrationTests/MetalVectorEnginePoolTests.swift`
- Modify: `Tests/WaxIntegrationTests/UnifiedSearchTests.swift`
- Modify: `Tests/WaxIntegrationTests/UnifiedSearchEngineCacheCoverageTests.swift`
- Modify: `Tests/WaxIntegrationTests/WALCompactionBenchmarks.swift`
- Modify: `Tests/WaxIntegrationTests/WALCompactionWorkloadSupport.swift`
- Modify: `Tests/WaxIntegrationTests/OptimizationComparisonBenchmark.swift`
- Modify: `Tests/WaxIntegrationTests/ProductionReadinessStabilityTests.swift`
- Modify: `Tests/WaxIntegrationTests/RAGBenchmarks.swift`
- Modify: `Tests/WaxIntegrationTests/LongMemoryBenchmarkHarness.swift`
- Modify: `Tests/WaxIntegrationTests/MemoryOrchestratorSessionGraphAndStatsTests.swift`
- Modify: `Tests/WaxIntegrationTests/StructuredMemoryWaxPersistenceTests.swift`
- Modify: `Tests/WaxIntegrationTests/LiveSetRewriteCompactionTests.swift`
- Modify: `Tests/WaxMCPServerTests/WaxMCPServerTests.swift`

**Step 1: Update each file**

For each file listed, use Edit `replace_all: true`:
- `".mv2s"` → `".wax"`
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`
- `hasSuffix(".mv2s")` → `hasSuffix(".wax")`

Process them in batches of 4-5 to keep changes reviewable.

**Step 2: Run all tests**

```bash
swift test --filter WaxCoreTests 2>&1 | tail -20
swift test --filter WaxIntegrationTests 2>&1 | tail -20
swift test --filter WaxMCPServerTests 2>&1 | tail -20
```
Expected: All pass.

**Step 3: Commit Layer 3**

```bash
git add Sources/ Tests/ WaxDemo/
git commit -m "refactor: rename file extension .mv2s→.wax across all source, test, and demo files"
```

---

## Layer 4 — Documentation and Skills

### Task 12: Update docs/wax-features-deep-dive.md

**Files:**
- Modify: `docs/wax-features-deep-dive.md`

**Step 1: Read lines 1375–1410 to see the type name and extension references**

Confirm the 14+ references including type names on lines 1383–1387 and 1399.

**Step 2: Replace type names (replace_all)**

- `MV2SHeaderPage` → `WaxHeaderPage`
- `MV2STOC` → `WaxTOC`
- `MV2SFooter` → `WaxFooter`

**Step 3: Replace extension strings (replace_all)**

- `".mv2s"` → `".wax"`
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`

**Step 4: Replace any remaining "MV2S" prose**

- `MV2S` → `Wax` (in prose context)
- `"MV2S"` → `"WAX1"` (in magic byte context if any)

---

### Task 13: Update skills/public/wax/SKILL.md

**Files:**
- Modify: `skills/public/wax/SKILL.md`

**Step 1: Read lines 40–135 to see prose and code examples**

The spec flags lines 46, 83, and 127 as having `appendingPathExtension("mv2s")`.

**Step 2: Replace code examples**

Use Edit `replace_all: true`:
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`
- `".mv2s"` → `".wax"`

**Step 3: Replace prose references**

Use Edit `replace_all: true`:
- `MV2S` → `Wax` (in prose)
- `"MV2S"` → `"WAX1"` (in magic byte context)

---

### Task 14: Update constraints.md and any other doc files

**Files:**
- Modify: `skills/public/wax/references/constraints.md`
- Modify: `README.md`
- Modify: `CLAUDE.md`

**Step 1: Read each file and apply same replacements**

For each file, use Edit `replace_all: true`:
- `MV2SHeaderPage` → `WaxHeaderPage`
- `MV2STOC` → `WaxTOC`
- `MV2SFooter` → `WaxFooter`
- `".mv2s"` → `".wax"`
- `appendingPathExtension("mv2s")` → `appendingPathExtension("wax")`
- `MV2S` → `Wax` (in prose — verify each replacement is contextually correct)

---

## Verification Pass

### Task 15: Full grep verification

**Step 1: No `.mv2s` literals remain anywhere**

```bash
grep -rn '\.mv2s' Sources/ Tests/ WaxDemo/ docs/ README.md CLAUDE.md skills/ 2>/dev/null
```
Expected: **No output.** If any remain, fix them before proceeding.

**Step 2: No MV2S type names remain**

```bash
grep -rn 'MV2SHeaderPage\|MV2STOC\|MV2SFooter\|MV2SEnums' Sources/ Tests/ WaxDemo/ docs/ 2>/dev/null
```
Expected: **No output.**

**Step 3: No old magic byte strings remain**

```bash
grep -rn '"MV2S"\|"MV2SFOOT"' Sources/ Tests/ 2>/dev/null
```
Expected: **No output.**

**Step 4: Full build**

```bash
swift build 2>&1
```
Expected: Build succeeded, zero errors.

**Step 5: Full test suite**

```bash
swift test --filter WaxCoreTests 2>&1 | tail -20
swift test --filter WaxIntegrationTests 2>&1 | tail -20
swift test --filter WaxMCPServerTests 2>&1 | tail -20
```
Expected: All pass.

**Step 6: Final commit**

```bash
git add docs/ skills/ README.md CLAUDE.md
git commit -m "docs: update all MV2S→Wax references in documentation and skills"
```

---

## Summary of All Commits

| # | Commit message |
|---|----------------|
| 1 | `refactor: rebrand binary magic bytes MV2S→WAX1, MV2SFOOT→WAX1FOOT` |
| 2 | `refactor: rename MV2SHeaderPage→WaxHeaderPage, MV2STOC→WaxTOC, MV2SFooter→WaxFooter` |
| 3 | `docs: update remaining MV2S→Wax doc comments in codec files` |
| 4 | `refactor: rename file extension .mv2s→.wax across all source, test, and demo files` |
| 5 | `docs: update all MV2S→Wax references in documentation and skills` |
