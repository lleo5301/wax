# Wax Framework Audit (2026-03-06)

## 1. Framework Summary

- **Purpose:** On-device persistent memory + retrieval framework for Swift AI agents and apps (text, vector, and structured memory orchestration).
- **Swift tools/runtime target:** `swift-tools-version: 6.1`, platform targets iOS 18 / macOS 15.
- **Dependency profile:** USearch, GRDB, swift-testing, swift-log, MCP Swift SDK, swift-argument-parser, swift-crypto, swift-docc-plugin, SwiftTUI, Noora.
- **Public API mapped:** ~1321 public/open declarations across 119 Swift source files (scripted inventory).
- **Overall health score:** **8.1 / 10**.

First impression: Wax already has strong architecture (actors, Sendable-forward APIs, extensive tests, and broad docs), but there were still a few avoidable force unwraps and optional handling issues in core/runtime-facing paths. This patch addresses the highest-signal safety issues without changing API behavior.

## 2. Bug Report

| Severity | File | Line | Bug | Fix Applied |
|---|---|---:|---|---|
| High | `Sources/WaxVectorSearch/MetalVectorEngine.swift` | 473 | Force unwrap of `raw.baseAddress!` in buffer copy path. | Replaced with guarded optional base address handling before copy. |
| High | `Sources/WaxVectorSearch/MetalVectorEngine.swift` | 498 | Force unwrap `computePipelineSIMD8!` in pipeline selection path. | Replaced with nil-coalescing pipeline selection. |
| Medium | `Sources/WaxCore/BinaryCodec/BinaryDecoder.swift` | 147-152 | Repeated `as!` casts in generic decode dispatch. | Introduced checked cast helper that throws `WaxError.decodingError` on mismatch. |
| Medium | `Sources/WaxTextSearch/FTS5SearchEngine.swift` | 223-226 | Optional comparisons used `toMs!` force unwrap. | Replaced with `if let` + guarded validation. |
| Medium | `Sources/WaxMCPServer/WaxMCPTools.swift` | 760 | Force unwrap `unicodeScalars.first!` in JSON string escape routine. | Replaced with safe optional binding + continue fallback. |

## 3. Type System Improvements

1. **Before:** `BinaryDecoder.decode<T>` used `as!` casting for primitive dispatch.  
   **After:** typed cast helper throws decoding error if cast fails.  
   **Why:** preserves crash-free decoding behavior while keeping generic API.  
   **Breaking:** No.

2. **Before:** implicit assumptions around scalar existence (`unicodeScalars.first!`).  
   **After:** safe binding fallback in string escaping path.  
   **Why:** removes force-unwrap panic vectors in utility code used by MCP response encoding.  
   **Breaking:** No.

## 4. Naming Changes

No public symbol renames were applied in this patch to avoid breaking consumers.

| Current Name | Proposed Name | Reason | Breaking |
|---|---|---|---|
| `enableTextSearch()` (deprecated session API) | `openSession(_:config:)` | Aligns with modern session model and removes capability-style method naming ambiguity. | Yes (already covered by existing deprecations) |
| `enableVectorSearch(...)` (deprecated) | `openSession(_:config:)` | Unifies setup path and reduces API branching. | Yes (already deprecated) |

## 5. Namespace Audit

- **Public stdlib/Foundation extension pollution:** no broad `public extension String/Array/...` pollution detected in modified code.
- **Crowding risks (for future pass):** umbrella names like `Wax`, `WaxSession`, and multiple session variants are understandable but still expose migration-era overlap in autocomplete.
- **Suggested next step:** complete removal timeline for deprecated session types and keep one obvious session API.

## 6. API Ergonomics Report

- **Human developer ergonomics:** **8.3 / 10**
- **AI coding agent ergonomics:** **8.0 / 10**

Top friction points:
1. Large public surface area (~1321 declarations) raises discoverability overhead.
2. Legacy + modern session APIs coexist, creating path ambiguity.
3. README had mismatch vs package platform baselines and lacked explicit “when not to use” guidance.
4. Some public paths still have sparse contract-focused docs.
5. Multi-module API entry points can be overwhelming without task-oriented guides.

## 7. Concurrency Report

- Strict concurrency flags are already enabled across targets.
- This patch did not require `@preconcurrency` additions.
- No actor isolation violations introduced.
- Unsafe unwraps in runtime code were removed to avoid cross-task crash vectors during async execution.

## 8. README Rewrite

README was rewritten to provide:
- one-line value proposition;
- quick-start within first screen;
- explicit requirements aligned to `Package.swift` platform minimums;
- “when to use / when not to use” guidance;
- direct links to docs and contribution expectations.

## 9. New & Improved Tests

Added a new decoding-safety unit test:
- `unsupportedGenericDecodeTypeThrows` validates that unsupported generic decode requests fail with a typed decoding error rather than trapping.
