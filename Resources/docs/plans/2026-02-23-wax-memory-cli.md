# Wax Memory CLI Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `wax memory` subcommand group to `WaxCLI` that exposes all 14 core MCP memory tools as direct CLI commands — no server process required.

**Architecture:** Extend the existing `WaxCLI` target (`Sources/WaxCLI/main.swift`) with a new `memory` subcommand group. Each command opens a `MemoryOrchestrator` directly (the same library the MCP server uses), performs the operation, flushes, closes, and prints results. Async commands use the `Task + dispatchMain()` pattern already established in `WaxMCPServer/main.swift`. Text output by default; `--json` flag for machine-readable output.

**Tech Stack:** Swift 6.2, ArgumentParser 1.7, Wax library (`MemoryOrchestrator`, `EntityKey`, `PredicateKey`, `FactValue`), WaxVectorSearchMiniLM (optional, behind `MiniLMEmbeddings` trait)

---

## Context: Existing Codebase

- `Sources/WaxCLI/main.swift` — single 516-line file; `wax mcp serve/install/doctor/uninstall`. Imports only `ArgumentParser` + `Foundation`. **Does not yet depend on `Wax`.**
- `Sources/WaxMCPServer/WaxMCPTools.swift` — 14 MCP tool handlers; reference for orchestrator call patterns.
- `Sources/WaxMCPServer/main.swift` — `Task + dispatchMain()` async pattern; `buildEmbedder()` for MiniLM.
- `Package.swift` — `WaxCLI` target has NO `Wax` dependency yet. Must add `Wax` + conditional `WaxVectorSearchMiniLM`.

**Note:** `WaxCLI/main.swift` currently has dead video/photo store options on `Serve`, `Install`, and `Doctor` subcommands (`--video-store-path`, `--photo-store-path`). Remove these in Task 1 as part of the cleanup.

---

## Task 1: Add `Wax` to WaxCLI and clean up dead options

**Files:**
- Modify: `Package.swift` (lines ~147–159, the `WaxCLI` executableTarget block)
- Modify: `Sources/WaxCLI/main.swift` (remove video/photo options from Serve, Install, Doctor)

### Step 1: Add Wax dependency to WaxCLI target in Package.swift

Replace the current `WaxCLI` executableTarget block:

```swift
.executableTarget(
    name: "WaxCLI",
    dependencies: [
        "Wax",
        .product(name: "ArgumentParser", package: "swift-argument-parser"),
        .target(
            name: "WaxVectorSearchMiniLM",
            condition: .when(traits: ["MiniLMEmbeddings"])
        ),
    ],
    path: "Sources/WaxCLI",
    swiftSettings: [
        .enableExperimentalFeature("StrictConcurrency"),
        .define("MiniLMEmbeddings", .when(traits: ["MiniLMEmbeddings"])),
    ]
),
```

### Step 2: Remove dead video/photo options from existing MCP subcommands

In `Sources/WaxCLI/main.swift`, delete the following lines from `Serve`, `Install`, and `Doctor`:

```swift
// Remove from Serve.run() arguments array:
"--video-store-path", Pathing.expandPath(videoStorePath),
"--photo-store-path", Pathing.expandPath(photoStorePath),

// Remove @Option declarations:
@Option(name: .customLong("video-store-path"), help: "Path to video store")
var videoStorePath = "~/.wax/video.mv2s"

@Option(name: .customLong("photo-store-path"), help: "Path to photo store")
var photoStorePath = "~/.wax/photo.mv2s"
```

Apply to all three commands (`Serve`, `Install`, `Doctor`). Also remove the `--video-store-path` and `--photo-store-path` args from `Install`'s `addArguments` build and from `Doctor`'s `arguments` array.

### Step 3: Verify build

```bash
swift build --product WaxCLI
```

Expected: `Build complete!` (no errors, no warnings about unused vars)

### Step 4: Commit

```bash
git add Package.swift Sources/WaxCLI/main.swift
git commit -m "feat(cli): add Wax dependency to WaxCLI, remove dead video/photo options"
```

---

## Task 2: Shared store factory (`StoreConfig.swift`)

**Files:**
- Create: `Sources/WaxCLI/Memory/StoreConfig.swift`

This file provides a reusable way to open a `MemoryOrchestrator` from a CLI context — same defaults and patterns as the MCP server. All memory commands share this.

### Step 1: Create the file

```swift
import Foundation
import Wax

#if MiniLMEmbeddings && canImport(WaxVectorSearchMiniLM)
import WaxVectorSearchMiniLM
#endif

/// Shared configuration and factory for memory commands.
enum StoreConfig {
    static let defaultStorePath = "~/.wax/memory.mv2s"

    /// Resolves, tilde-expands, and creates parent directory for a store path.
    static func resolveURL(_ rawPath: String) throws -> URL {
        let expanded = (rawPath as NSString).expandingTildeInPath
        let trimmed = expanded.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !trimmed.isEmpty else {
            throw CLIMemoryError("Store path cannot be empty")
        }
        let url = URL(fileURLWithPath: trimmed).standardizedFileURL
        let dir = url.deletingLastPathComponent()
        try FileManager.default.createDirectory(at: dir, withIntermediateDirectories: true)
        return url
    }

    /// Opens a MemoryOrchestrator with text-only search (no embedder required).
    /// Use this for commands that don't need vector search.
    static func openTextOnly(at url: URL) async throws -> MemoryOrchestrator {
        var config = OrchestratorConfig.default
        config.enableVectorSearch = false
        config.rag.searchMode = .textOnly
        config.enableStructuredMemory = true
        return try await MemoryOrchestrator(at: url, config: config)
    }

    /// Opens a MemoryOrchestrator with MiniLM embedder when available,
    /// falling back to text-only if the embedder cannot be loaded.
    static func open(at url: URL) async throws -> MemoryOrchestrator {
        let embedder: (any EmbeddingProvider)? = await {
            #if MiniLMEmbeddings && canImport(WaxVectorSearchMiniLM)
            do {
                let e = try MiniLMEmbedder()
                try? await e.prewarm(batchSize: 4)
                return e
            } catch {
                return nil
            }
            #else
            return nil
            #endif
        }()

        var config = OrchestratorConfig.default
        config.enableStructuredMemory = true
        if embedder == nil {
            config.enableVectorSearch = false
            config.rag.searchMode = .textOnly
        }
        return try await MemoryOrchestrator(at: url, config: config, embedder: embedder)
    }
}

struct CLIMemoryError: LocalizedError {
    let message: String
    init(_ message: String) { self.message = message }
    var errorDescription: String? { message }
}
```

### Step 2: Build to verify it compiles

```bash
swift build --product WaxCLI
```

Expected: `Build complete!`

### Step 3: Commit

```bash
git add Sources/WaxCLI/Memory/StoreConfig.swift
git commit -m "feat(cli/memory): add shared StoreConfig factory"
```

---

## Task 3: Memory command group skeleton + wire into main

**Files:**
- Create: `Sources/WaxCLI/Memory/MemoryCommand.swift`
- Modify: `Sources/WaxCLI/main.swift` (add `Memory.self` to subcommands)

### Step 1: Create MemoryCommand.swift with the group shell

```swift
import ArgumentParser
import Foundation

extension WaxCLI {
    struct Memory: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Read and write Wax memory",
            subcommands: [
                Remember.self,
                Recall.self,
                Search.self,
                Stats.self,
                Handoff.self,
                Entity.self,
                Facts.self,
            ]
        )
    }
}
```

### Step 2: Add `Memory.self` to WaxCLI root subcommands in main.swift

```swift
struct WaxCLI: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "wax",
        abstract: "Wax developer CLI",
        subcommands: [Memory.self, MCP.self]  // Memory first
    )
}
```

### Step 3: Build

```bash
swift build --product WaxCLI
```

Expected: `Build complete!` — the group is registered, subcommands will be added in subsequent tasks.

### Step 4: Commit

```bash
git add Sources/WaxCLI/Memory/MemoryCommand.swift Sources/WaxCLI/main.swift
git commit -m "feat(cli/memory): wire Memory subcommand group into WaxCLI"
```

---

## Task 4: `wax memory remember` and `wax memory stats`

**Files:**
- Create: `Sources/WaxCLI/Memory/CoreCommands.swift`

Maps to MCP tools: `wax_remember`, `wax_stats`.

### Step 1: Create CoreCommands.swift

```swift
import ArgumentParser
import Darwin
import Dispatch
import Foundation
import Wax

extension WaxCLI.Memory {
    struct Remember: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Store text in Wax memory."
        )

        @Argument(help: "Text content to store.")
        var content: String

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Option(name: .customLong("session-id"), help: "Optional session UUID to scope this write.")
        var sessionId: String?

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.open(at: url)
            defer { Task { try? await memory.close() } }

            var metadata: [String: String] = [:]
            if let sid = sessionId {
                guard UUID(uuidString: sid) != nil else {
                    throw CLIMemoryError("session-id must be a valid UUID")
                }
                metadata["session_id"] = sid
            }

            let before = await memory.runtimeStats()
            try await memory.remember(content, metadata: metadata)
            try await memory.flush()
            let after = await memory.runtimeStats()

            let totalBefore = before.frameCount + before.pendingFrames
            let totalAfter = after.frameCount + after.pendingFrames
            let added = totalAfter >= totalBefore ? (totalAfter - totalBefore) : 0

            if json {
                printJSON([
                    "status": "ok",
                    "framesAdded": added,
                    "frameCount": after.frameCount,
                ])
            } else {
                print("Remembered. +\(added) frame(s). Total: \(after.frameCount)")
            }
        }
    }

    struct Stats: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Show Wax memory runtime and storage stats."
        )

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            let stats = await memory.runtimeStats()
            let diskBytes: UInt64 = {
                guard let attrs = try? FileManager.default.attributesOfItem(atPath: stats.storeURL.path),
                      let size = attrs[.size] as? NSNumber else { return 0 }
                return size.uint64Value
            }()

            if json {
                printJSON([
                    "frameCount": stats.frameCount,
                    "pendingFrames": stats.pendingFrames,
                    "diskBytes": diskBytes,
                    "vectorSearchEnabled": stats.vectorSearchEnabled,
                    "storePath": stats.storeURL.path,
                ])
            } else {
                print("Frames:         \(stats.frameCount) committed, \(stats.pendingFrames) pending")
                print("Disk:           \(diskBytes / 1024) KB")
                print("Vector search:  \(stats.vectorSearchEnabled ? "enabled" : "disabled")")
                print("Store:          \(stats.storeURL.path)")
            }
        }
    }
}

// MARK: - JSON helper (shared by all memory commands)

func printJSON(_ value: some Encodable) {
    let encoder = JSONEncoder()
    encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
    if let data = try? encoder.encode(value), let str = String(data: data, encoding: .utf8) {
        print(str)
    }
}

// Note: For heterogeneous JSON dicts, use a [String: JSONValue] approach or AnyCodable.
// For simplicity, the JSON helper above works for Codable types.
// Commands that need ad-hoc JSON dicts use manual JSONSerialization below.

func printJSONDict(_ dict: [String: Any]) {
    guard let data = try? JSONSerialization.data(withJSONObject: dict, options: [.prettyPrinted, .sortedKeys]),
          let str = String(data: data, encoding: .utf8) else { return }
    print(str)
}
```

**Note:** The `printJSON` helper can't handle `[String: Any]` directly because it's not `Encodable`. Use `printJSONDict` for ad-hoc dictionaries throughout memory commands.

### Step 2: Update MemoryCommand.swift to add Stats to subcommands

It's already in the list from Task 3. No change needed if you wrote it correctly.

### Step 3: Build

```bash
swift build --product WaxCLI
```

Expected: `Build complete!`

### Step 4: Smoke test

```bash
.build/debug/WaxCLI memory remember "Swift actors isolate mutable state"
.build/debug/WaxCLI memory stats
.build/debug/WaxCLI memory stats --json
```

Expected:
```
Remembered. +1 frame(s). Total: 1
Frames:   1 committed, 0 pending
...
```

### Step 5: Commit

```bash
git add Sources/WaxCLI/Memory/CoreCommands.swift Sources/WaxCLI/Memory/MemoryCommand.swift
git commit -m "feat(cli/memory): add wax memory remember and stats commands"
```

---

## Task 5: `wax memory recall` and `wax memory search`

**Files:**
- Modify: `Sources/WaxCLI/Memory/CoreCommands.swift` (add Recall and Search structs)

Maps to MCP tools: `wax_recall`, `wax_search`.

### Step 1: Add Recall to CoreCommands.swift

```swift
extension WaxCLI.Memory {
    struct Recall: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Recall context for a query using Wax RAG assembly."
        )

        @Argument(help: "Recall query text.")
        var query: String

        @Option(name: .customLong("limit"), help: "Max context items (1–100). Default: 5.")
        var limit = 5

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Option(name: .customLong("session-id"), help: "Optional session UUID for scoped recall.")
        var sessionId: String?

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            guard limit >= 1, limit <= 100 else {
                throw CLIMemoryError("limit must be between 1 and 100")
            }
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.open(at: url)
            defer { Task { try? await memory.close() } }

            let frameFilter: FrameFilter? = try {
                guard let sid = sessionId else { return nil }
                guard UUID(uuidString: sid) != nil else {
                    throw CLIMemoryError("session-id must be a valid UUID")
                }
                return FrameFilter(
                    metadataFilter: MetadataFilter(requiredEntries: ["session_id": sid])
                )
            }()

            let context = try await memory.recall(query: query, frameFilter: frameFilter)
            let selected = context.items.prefix(limit)

            if json {
                let items = selected.map { item -> [String: Any] in
                    ["kind": item.kind, "frameId": item.frameId, "score": item.score, "text": item.text]
                }
                printJSONDict(["query": context.query, "totalTokens": context.totalTokens, "items": items])
            } else {
                print("Query: \(context.query)  |  Tokens: \(context.totalTokens)")
                for (i, item) in selected.enumerated() {
                    print("\(i + 1). [\(item.kind)] score=\(String(format: "%.4f", item.score))  \(item.text)")
                }
            }
        }
    }

    struct Search: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Run direct Wax search and return ranked raw hits."
        )

        @Argument(help: "Search query text.")
        var query: String

        @Option(name: .customLong("mode"), help: "Search mode: text or hybrid. Default: hybrid.")
        var mode = "hybrid"

        @Option(name: .customLong("top-k"), help: "Max hits (1–200). Default: 10.")
        var topK = 10

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Option(name: .customLong("session-id"), help: "Optional session UUID for scoped search.")
        var sessionId: String?

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            guard topK >= 1, topK <= 200 else {
                throw CLIMemoryError("top-k must be between 1 and 200")
            }
            let searchMode: MemoryOrchestrator.DirectSearchMode
            switch mode.lowercased() {
            case "text": searchMode = .text
            case "hybrid": searchMode = .hybrid(alpha: 0.5)
            default: throw CLIMemoryError("mode must be 'text' or 'hybrid'")
            }

            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.open(at: url)
            defer { Task { try? await memory.close() } }

            let frameFilter: FrameFilter? = try {
                guard let sid = sessionId else { return nil }
                guard UUID(uuidString: sid) != nil else {
                    throw CLIMemoryError("session-id must be a valid UUID")
                }
                return FrameFilter(
                    metadataFilter: MetadataFilter(requiredEntries: ["session_id": sid])
                )
            }()

            let hits = try await memory.search(query: query, mode: searchMode, topK: topK, frameFilter: frameFilter)

            if json {
                let rows = hits.enumerated().map { i, h -> [String: Any] in
                    ["rank": i + 1, "frameId": h.frameId, "score": h.score,
                     "sources": h.sources.map(\.rawValue), "preview": h.previewText ?? ""]
                }
                printJSONDict(["count": hits.count, "hits": rows])
            } else {
                if hits.isEmpty {
                    print("No results.")
                    return
                }
                for (i, hit) in hits.enumerated() {
                    let src = hit.sources.map(\.rawValue).joined(separator: "+")
                    print("\(i + 1). frame=\(hit.frameId) score=\(String(format: "%.4f", hit.score)) [\(src)]  \(hit.previewText ?? "")")
                }
            }
        }
    }
}
```

### Step 2: Build

```bash
swift build --product WaxCLI
```

### Step 3: Smoke test

```bash
.build/debug/WaxCLI memory recall "actors"
.build/debug/WaxCLI memory search "actors" --mode text --top-k 3
.build/debug/WaxCLI memory search "actors" --json
```

### Step 4: Commit

```bash
git add Sources/WaxCLI/Memory/CoreCommands.swift
git commit -m "feat(cli/memory): add wax memory recall and search commands"
```

---

## Task 6: `wax memory handoff`

**Files:**
- Create: `Sources/WaxCLI/Memory/HandoffCommands.swift`

Maps to MCP tools: `wax_handoff`, `wax_handoff_latest`.

### Step 1: Create HandoffCommands.swift

```swift
import ArgumentParser
import Dispatch
import Foundation
import Wax

extension WaxCLI.Memory {
    struct Handoff: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Store and retrieve cross-session handoff notes.",
            subcommands: [Write.self, Read.self]
        )
    }
}

extension WaxCLI.Memory.Handoff {
    struct Write: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Store a handoff note for the next session."
        )

        @Argument(help: "Handoff note content.")
        var content: String

        @Option(name: .customLong("project"), help: "Optional project scope.")
        var project: String?

        @Option(name: .customLong("task"), help: "Pending task to carry over (repeatable).")
        var pendingTasks: [String] = []

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            let frameId = try await memory.rememberHandoff(
                content: content,
                project: project,
                pendingTasks: pendingTasks,
                sessionId: nil
            )
            try await memory.flush()

            if json {
                printJSONDict(["status": "ok", "frame_id": frameId])
            } else {
                let scope = project.map { " (project: \($0))" } ?? ""
                print("Handoff stored\(scope). frame_id=\(frameId)")
                if !pendingTasks.isEmpty {
                    print("Pending tasks:")
                    pendingTasks.forEach { print("  - \($0)") }
                }
            }
        }
    }

    struct Read: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Fetch the latest handoff note."
        )

        @Option(name: .customLong("project"), help: "Optional project scope.")
        var project: String?

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            guard let latest = try await memory.latestHandoff(project: project) else {
                if json {
                    printJSONDict(["found": false])
                } else {
                    print("No handoff note found\(project.map { " for project '\($0)'" } ?? "").")
                }
                return
            }

            if json {
                printJSONDict([
                    "found": true,
                    "frame_id": latest.frameId,
                    "timestamp_ms": latest.timestampMs,
                    "project": latest.project as Any,
                    "pending_tasks": latest.pendingTasks,
                    "content": latest.content,
                ])
            } else {
                print(latest.content)
                if !latest.pendingTasks.isEmpty {
                    print("\nPending tasks:")
                    latest.pendingTasks.forEach { print("  - \($0)") }
                }
            }
        }
    }
}
```

### Step 2: Update MemoryCommand.swift — add `Handoff` to subcommands

`Handoff` is already in the subcommands list from Task 3.

### Step 3: Build

```bash
swift build --product WaxCLI
```

### Step 4: Smoke test

```bash
.build/debug/WaxCLI memory handoff write "Work in progress on actor refactor" --project wax --task "add graph tests"
.build/debug/WaxCLI memory handoff read --project wax
.build/debug/WaxCLI memory handoff read --json
```

### Step 5: Commit

```bash
git add Sources/WaxCLI/Memory/HandoffCommands.swift Sources/WaxCLI/Memory/MemoryCommand.swift
git commit -m "feat(cli/memory): add wax memory handoff write/read commands"
```

---

## Task 7: `wax memory entity` subcommands

**Files:**
- Create: `Sources/WaxCLI/Memory/GraphCommands.swift`

Maps to MCP tools: `wax_entity_upsert`, `wax_entity_resolve`.

### Step 1: Create GraphCommands.swift (entity portion)

```swift
import ArgumentParser
import Dispatch
import Foundation
import Wax

extension WaxCLI.Memory {
    struct Entity: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Manage structured-memory entities.",
            subcommands: [Upsert.self, Resolve.self]
        )
    }
}

extension WaxCLI.Memory.Entity {
    struct Upsert: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Upsert a structured-memory entity by key."
        )

        @Argument(help: "Entity key in 'namespace:id' format, e.g. agent:codex.")
        var key: String

        @Option(name: .customLong("kind"), help: "Entity kind, e.g. agent, project.")
        var kind: String

        @Option(name: .customLong("alias"), help: "Alias for entity resolution (repeatable).")
        var aliases: [String] = []

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            let entityID = try await memory.upsertEntity(
                key: EntityKey(key),
                kind: kind,
                aliases: aliases,
                commit: true
            )

            if json {
                printJSONDict(["status": "ok", "entity_id": entityID.rawValue, "key": key])
            } else {
                print("Upserted entity '\(key)' (id=\(entityID.rawValue))")
            }
        }
    }

    struct Resolve: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Resolve entities by alias."
        )

        @Argument(help: "Alias to search for.")
        var alias: String

        @Option(name: .customLong("limit"), help: "Max matches (1–100). Default: 10.")
        var limit = 10

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            let matches = try await memory.resolveEntities(matchingAlias: alias, limit: limit)

            if json {
                let rows = matches.map { ["id": $0.id, "key": $0.key.rawValue, "kind": $0.kind] }
                printJSONDict(["count": matches.count, "entities": rows])
            } else {
                if matches.isEmpty { print("No entities found for alias '\(alias)'."); return }
                for m in matches {
                    print("\(m.key.rawValue) [\(m.kind)] id=\(m.id)")
                }
            }
        }
    }
}
```

### Step 2: Build and verify

```bash
swift build --product WaxCLI
```

### Step 3: Smoke test

```bash
.build/debug/WaxCLI memory entity upsert "agent:codex" --kind agent --alias codex --alias assistant
.build/debug/WaxCLI memory entity resolve "codex"
```

### Step 4: Commit

```bash
git add Sources/WaxCLI/Memory/GraphCommands.swift
git commit -m "feat(cli/memory): add wax memory entity upsert/resolve commands"
```

---

## Task 8: `wax memory facts` subcommands

**Files:**
- Modify: `Sources/WaxCLI/Memory/GraphCommands.swift` (add Facts group and subcommands)

Maps to MCP tools: `wax_fact_assert`, `wax_fact_retract`, `wax_facts_query`.

### Step 1: Add Facts commands to GraphCommands.swift

```swift
extension WaxCLI.Memory {
    struct Facts: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Assert, retract, and query structured-memory facts.",
            subcommands: [Assert.self, Retract.self, Query.self]
        )
    }
}

extension WaxCLI.Memory.Facts {
    struct Assert: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Assert a fact: <subject> <predicate> <object>."
        )

        @Argument(help: "Subject entity key, e.g. agent:codex.")
        var subject: String

        @Argument(help: "Predicate key, e.g. learned_behavior.")
        var predicate: String

        @Argument(help: "Object value (string, integer, or boolean).")
        var object: String

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            // Parse object: try Int64, then Bool, fall back to String
            let factValue: FactValue = {
                if let i = Int64(object) { return .int(i) }
                if object.lowercased() == "true" { return .bool(true) }
                if object.lowercased() == "false" { return .bool(false) }
                return .string(object)
            }()

            let factID = try await memory.assertFact(
                subject: EntityKey(subject),
                predicate: PredicateKey(predicate),
                object: factValue,
                validFromMs: nil,
                validToMs: nil,
                commit: true
            )

            if json {
                printJSONDict(["status": "ok", "fact_id": factID.rawValue])
            } else {
                print("Asserted fact id=\(factID.rawValue): \(subject) \(predicate) \(object)")
            }
        }
    }

    struct Retract: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Retract a fact by its ID."
        )

        @Argument(help: "Fact row ID to retract.")
        var factId: Int64

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            try await memory.retractFact(factId: FactRowID(rawValue: factId), atMs: nil, commit: true)

            if json {
                printJSONDict(["status": "ok", "fact_id": factId])
            } else {
                print("Retracted fact id=\(factId)")
            }
        }
    }

    struct Query: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Query structured-memory facts."
        )

        @Option(name: .customLong("subject"), help: "Filter by subject entity key.")
        var subject: String?

        @Option(name: .customLong("predicate"), help: "Filter by predicate key.")
        var predicate: String?

        @Option(name: .customLong("limit"), help: "Max results (1–500). Default: 20.")
        var limit = 20

        @Option(name: .customLong("store-path"), help: "Path to memory store.")
        var storePath = StoreConfig.defaultStorePath

        @Flag(name: .customLong("json"), help: "Output JSON.")
        var json = false

        mutating func run() throws {
            let cmd = self
            var runError: Error?
            let sema = DispatchSemaphore(value: 0)
            Task {
                do { try await cmd.runAsync() }
                catch { runError = error }
                sema.signal()
            }
            sema.wait()
            if let err = runError { throw err }
        }

        private func runAsync() async throws {
            let url = try StoreConfig.resolveURL(storePath)
            let memory = try await StoreConfig.openTextOnly(at: url)
            defer { Task { try? await memory.close() } }

            let result = try await memory.facts(
                about: subject.map { EntityKey($0) },
                predicate: predicate.map { PredicateKey($0) },
                asOfMs: Int64.max,
                limit: limit
            )

            if json {
                let rows = result.hits.map { h -> [String: Any] in
                    [
                        "fact_id": h.factId.rawValue,
                        "subject": h.fact.subject.rawValue,
                        "predicate": h.fact.predicate.rawValue,
                        "object": "\(h.fact.object)",
                        "is_open_ended": h.isOpenEnded,
                    ]
                }
                printJSONDict(["count": result.hits.count, "truncated": result.wasTruncated, "hits": rows])
            } else {
                if result.hits.isEmpty { print("No facts found."); return }
                for h in result.hits {
                    print("[\(h.factId.rawValue)] \(h.fact.subject.rawValue) \(h.fact.predicate.rawValue) = \(h.fact.object)")
                }
                if result.wasTruncated { print("(truncated — use --limit to see more)") }
            }
        }
    }
}
```

### Step 2: Build

```bash
swift build --product WaxCLI
```

### Step 3: Smoke test

```bash
.build/debug/WaxCLI memory facts assert "agent:codex" learned_behavior "Prefer focused patches"
.build/debug/WaxCLI memory facts query --subject "agent:codex"
.build/debug/WaxCLI memory facts retract 1
.build/debug/WaxCLI memory facts query --subject "agent:codex" --json
```

### Step 4: Commit

```bash
git add Sources/WaxCLI/Memory/GraphCommands.swift
git commit -m "feat(cli/memory): add wax memory facts assert/retract/query commands"
```

---

## Task 9: Tests

**Files:**
- Create: `Tests/WaxCLITests/WaxCLIMemoryTests.swift`
- Modify: `Package.swift` (add `WaxCLITests` test target)

Since the CLI commands are thin wrappers over `MemoryOrchestrator`, test the orchestrator operations directly — same pattern as `WaxMCPServerTests`.

### Step 1: Add test target to Package.swift

```swift
.testTarget(
    name: "WaxCLITests",
    dependencies: [
        "Wax",
        .product(name: "Testing", package: "swift-testing"),
    ],
    swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
),
```

### Step 2: Create Tests/WaxCLITests/WaxCLIMemoryTests.swift

```swift
import Foundation
import Testing
import Wax

@Test
func rememberAndRecallRoundTrip() async throws {
    try await withMemory { memory in
        try await memory.remember("CLI test frame alpha", metadata: [:])
        try await memory.flush()
        let context = try await memory.recall(query: "alpha", frameFilter: nil)
        #expect(context.items.contains(where: { $0.text.contains("alpha") }))
    }
}

@Test
func searchReturnsHits() async throws {
    try await withMemory { memory in
        try await memory.remember("CLI search frame beta", metadata: [:])
        try await memory.flush()
        let hits = try await memory.search(query: "beta", mode: .text, topK: 5, frameFilter: nil)
        #expect(!hits.isEmpty)
    }
}

@Test
func handoffRoundTrip() async throws {
    try await withMemory { memory in
        let frameId = try await memory.rememberHandoff(
            content: "CLI handoff test",
            project: "cli-tests",
            pendingTasks: ["verify tests"],
            sessionId: nil
        )
        #expect(frameId > 0)
        guard let latest = try await memory.latestHandoff(project: "cli-tests") else {
            Issue.record("Expected handoff but got nil")
            return
        }
        #expect(latest.content.contains("CLI handoff test"))
        #expect(latest.pendingTasks.contains("verify tests"))
    }
}

@Test
func entityAndFactRoundTrip() async throws {
    try await withMemory { memory in
        let entityID = try await memory.upsertEntity(
            key: EntityKey("cli:test"),
            kind: "test",
            aliases: ["cli-test"],
            commit: true
        )
        #expect(entityID.rawValue > 0)

        let factID = try await memory.assertFact(
            subject: EntityKey("cli:test"),
            predicate: PredicateKey("status"),
            object: .string("active"),
            validFromMs: nil,
            validToMs: nil,
            commit: true
        )
        #expect(factID.rawValue > 0)

        let result = try await memory.facts(
            about: EntityKey("cli:test"),
            predicate: PredicateKey("status"),
            asOfMs: Int64.max,
            limit: 10
        )
        #expect(result.hits.count == 1)

        let matches = try await memory.resolveEntities(matchingAlias: "cli-test", limit: 5)
        #expect(matches.contains(where: { $0.key == EntityKey("cli:test") }))
    }
}

// MARK: - Helpers

private func withMemory(_ body: @Sendable (MemoryOrchestrator) async throws -> Void) async throws {
    let url = FileManager.default.temporaryDirectory
        .appendingPathComponent("wax-cli-tests-\(UUID().uuidString)")
        .appendingPathExtension("mv2s")
    defer { try? FileManager.default.removeItem(at: url) }

    var config = OrchestratorConfig.default
    config.enableVectorSearch = false
    config.enableStructuredMemory = true
    config.rag = FastRAGConfig(
        maxContextTokens: 120, expansionMaxTokens: 60,
        snippetMaxTokens: 30, maxSnippets: 8,
        searchTopK: 20, searchMode: .textOnly
    )

    let memory = try await MemoryOrchestrator(at: url, config: config)
    var deferredError: Error?
    do { try await body(memory) } catch { deferredError = error }
    do { try await memory.close() } catch { if deferredError == nil { deferredError = error } }
    if let e = deferredError { throw e }
}
```

### Step 3: Run tests

```bash
swift test --filter WaxCLITests
```

Expected: all 4 tests pass.

### Step 4: Commit

```bash
git add Tests/WaxCLITests/ Package.swift
git commit -m "test(cli/memory): add WaxCLITests for memory round-trip operations"
```

---

## Task 10: Final cleanup and verification

### Step 1: Full help output check

```bash
.build/debug/WaxCLI --help
.build/debug/WaxCLI memory --help
.build/debug/WaxCLI memory recall --help
.build/debug/WaxCLI memory handoff --help
.build/debug/WaxCLI memory entity --help
.build/debug/WaxCLI memory facts --help
```

Confirm all subcommands appear with correct descriptions and no leftover video/photo options.

### Step 2: Full test suite

```bash
swift test --traits MCPServer --filter WaxMCPServerTests
swift test --filter WaxCLITests
```

Expected: 12 MCP tests + 4 CLI tests = 16 tests passing.

### Step 3: Update CHANGELOG.md

Add to the `[0.1.2]` section under `### Added`:
```
- `wax memory` CLI command group: `remember`, `recall`, `search`, `stats`, `handoff write/read`, `entity upsert/resolve`, `facts assert/retract/query`
```

### Step 4: Final commit

```bash
git add CHANGELOG.md
git commit -m "docs: update CHANGELOG with wax memory CLI commands"
```

---

## Command Reference (final CLI surface)

```
wax memory remember "<text>" [--session-id UUID] [--json]
wax memory recall "<query>" [--limit N] [--session-id UUID] [--json]
wax memory search "<query>" [--mode text|hybrid] [--top-k N] [--session-id UUID] [--json]
wax memory stats [--json]
wax memory handoff write "<text>" [--project <name>] [--task <task>]... [--json]
wax memory handoff read [--project <name>] [--json]
wax memory entity upsert <key> --kind <kind> [--alias <alias>]... [--json]
wax memory entity resolve <alias> [--limit N] [--json]
wax memory facts assert <subject> <predicate> <object> [--json]
wax memory facts retract <fact-id> [--json]
wax memory facts query [--subject <key>] [--predicate <key>] [--limit N] [--json]
```

All commands accept `--store-path` to override the default `~/.wax/memory.mv2s`.

---

## Gotchas

1. **Async pattern:** `ParsableCommand.run()` is synchronous. Use `DispatchSemaphore` + `Task` (not `dispatchMain()`) since the CLI exits after the command; no need for a run loop.

2. **`FactValue` printing:** `h.fact.object` uses `"\(h.fact.object)"` in text output. Verify this produces readable output by checking the `FactValue` description. If not, add a helper function to format it.

3. **`EntityKey` / `PredicateKey` validation:** The CLI doesn't replicate the full validation from `WaxMCPTools.swift` (charset checks, namespace requirement). The orchestrator will throw if inputs are invalid — that's sufficient for CLI. Add validation only if the error messages from the orchestrator are confusing.

4. **Concurrency:** Swift 6 strict concurrency is enabled. `ParsableCommand` structs are value types. The `let cmd = self` capture pattern is used to avoid mutating-self capture issues in Tasks. Do not change to `var cmd = self`.

5. **`MemoryOrchestrator.close()` in defer:** Using `Task { try? await memory.close() }` in `defer` is a fire-and-forget pattern. For CLI tools it's acceptable — the process exits after the command completes. If clean shutdown matters, call `try await memory.close()` directly (no defer).
