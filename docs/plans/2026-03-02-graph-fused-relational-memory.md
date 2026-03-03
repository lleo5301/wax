# Graph-Fused Relational Memory Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Transform Wax from a text/vector memory store into a graph-fused relational memory engine where agents use 7 intuitive tools (remember, recall, forget, context, reflect, handoff, handoff_latest) and the knowledge graph works invisibly behind the scenes.

**Architecture:** Inject a knowledge graph layer into the existing RAG pipeline. `remember` auto-extracts entities and facts via Apple Foundation Models (on-device). `recall` fuses text + vector + 2-hop graph walk results into a single ranked context. New tools (`forget`, `context`, `reflect`) provide correction, entity introspection, and proactive insights. All operations support optional project scoping.

**Tech Stack:** Swift 6, Apple Foundation Models (macOS 26+), existing GRDB/FTS5/USearch stack, MCP SDK 0.10.0, Swift Argument Parser

---

## Phase 1: Core Graph Infrastructure

### Task 1: EntityExtractor Protocol

Define the extraction contract that all extractors implement.

**Files:**
- Create: `Sources/WaxCore/Extraction/EntityExtractor.swift`
- Create: `Sources/WaxCore/Extraction/ExtractionResult.swift`
- Test: `Tests/WaxCoreTests/Extraction/EntityExtractorTests.swift`

**Step 1: Write the failing test**

```swift
// Tests/WaxCoreTests/Extraction/EntityExtractorTests.swift
import Testing
@testable import WaxCore

struct EntityExtractorTests {
    @Test func extractionResultRoundTrips() {
        let entity = ExtractedEntity(
            key: EntityKey("user:chris"),
            kind: "person",
            aliases: ["Chris", "Christopher"]
        )
        let fact = ExtractedFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("guard clauses"),
            confidence: 0.9
        )
        let result = ExtractionResult(entities: [entity], facts: [fact])
        #expect(result.entities.count == 1)
        #expect(result.facts.count == 1)
        #expect(result.facts[0].confidence == 0.9)
    }

    @Test func emptyExtractionResultIsValid() {
        let result = ExtractionResult(entities: [], facts: [])
        #expect(result.isEmpty)
    }
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --filter EntityExtractorTests`
Expected: FAIL — module/types not found

**Step 3: Write minimal implementation**

```swift
// Sources/WaxCore/Extraction/ExtractionResult.swift
import Foundation

/// An entity extracted from text content.
public struct ExtractedEntity: Sendable, Equatable {
    public var key: EntityKey
    public var kind: String
    public var aliases: [String]

    public init(key: EntityKey, kind: String, aliases: [String] = []) {
        self.key = key
        self.kind = kind
        self.aliases = aliases
    }
}

/// A fact extracted from text content.
public struct ExtractedFact: Sendable, Equatable {
    public var subject: EntityKey
    public var predicate: PredicateKey
    public var object: FactValue
    public var confidence: Double?

    public init(
        subject: EntityKey,
        predicate: PredicateKey,
        object: FactValue,
        confidence: Double? = nil
    ) {
        self.subject = subject
        self.predicate = predicate
        self.object = object
        self.confidence = confidence
    }
}

/// Result of extracting entities and facts from text.
public struct ExtractionResult: Sendable, Equatable {
    public var entities: [ExtractedEntity]
    public var facts: [ExtractedFact]

    public var isEmpty: Bool { entities.isEmpty && facts.isEmpty }

    public init(entities: [ExtractedEntity], facts: [ExtractedFact]) {
        self.entities = entities
        self.facts = facts
    }
}
```

```swift
// Sources/WaxCore/Extraction/EntityExtractor.swift
import Foundation

/// Protocol for extracting entities and facts from text content.
/// Implementations may use on-device ML, rule-based heuristics, or remote APIs.
public protocol EntityExtractor: Sendable {
    /// Extract entities and facts from the given text.
    func extract(from text: String) async throws -> ExtractionResult

    /// Whether this extractor is available on the current platform.
    var isAvailable: Bool { get }
}
```

**Step 4: Update Package.swift**

Add the new files to the `WaxCore` target. No new target needed — these types belong with the existing structured memory types in WaxCore.

**Step 5: Run test to verify it passes**

Run: `swift test --filter EntityExtractorTests`
Expected: PASS

**Step 6: Commit**

```bash
git add Sources/WaxCore/Extraction/ Tests/WaxCoreTests/Extraction/
git commit -m "feat(core): add EntityExtractor protocol and ExtractionResult types"
```

---

### Task 2: GraphWalker — 2-Hop Entity Graph Traversal

Implement graph walking over the existing `sm_fact` edge indexes. The `sm_fact_edge_out_idx` and `sm_fact_edge_in_idx` indexes are already built but no public traversal method exists.

**Files:**
- Create: `Sources/WaxCore/Extraction/GraphWalkResult.swift`
- Modify: `Sources/WaxTextSearch/FTS5SearchEngine.swift` (add `edges()` and `walkGraph()` methods)
- Modify: `Sources/Wax/WaxSession.swift` (expose graph walk)
- Test: `Tests/WaxCoreTests/GraphWalkResultTests.swift`
- Test: `Tests/WaxIntegrationTests/GraphWalkerIntegrationTests.swift`

**Step 1: Write the failing test for GraphWalkResult types**

```swift
// Tests/WaxCoreTests/GraphWalkResultTests.swift
import Testing
@testable import WaxCore

struct GraphWalkResultTests {
    @Test func graphWalkHitScoresDecayByHop() {
        let hop0 = GraphWalkHit(
            factId: FactRowID(1),
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("Swift"),
            hopDistance: 0,
            confidence: 0.9,
            evidenceFrameIds: [42]
        )
        let hop1 = GraphWalkHit(
            factId: FactRowID(2),
            subject: EntityKey("project:wax"),
            predicate: PredicateKey("uses"),
            object: .string("SQLite"),
            hopDistance: 1,
            confidence: 0.8,
            evidenceFrameIds: [43]
        )
        // Hop decay: 1.0 / (1 + hopDistance)
        #expect(hop0.hopDecay == 1.0)
        #expect(hop1.hopDecay > 0.49)
        #expect(hop1.hopDecay < 0.51)
    }

    @Test func graphWalkResultCollectsEvidenceFrameIds() {
        let result = GraphWalkResult(
            resolvedEntities: [EntityKey("user:chris")],
            hits: [
                GraphWalkHit(factId: FactRowID(1), subject: EntityKey("user:chris"),
                    predicate: PredicateKey("prefers"), object: .string("Swift"),
                    hopDistance: 0, confidence: 0.9, evidenceFrameIds: [42, 43]),
                GraphWalkHit(factId: FactRowID(2), subject: EntityKey("user:chris"),
                    predicate: PredicateKey("uses"), object: .entity(EntityKey("project:wax")),
                    hopDistance: 0, confidence: 0.8, evidenceFrameIds: [43, 44]),
            ]
        )
        #expect(result.allEvidenceFrameIds == [42, 43, 44])
    }
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --filter GraphWalkResultTests`
Expected: FAIL — types not found

**Step 3: Write GraphWalkResult types**

```swift
// Sources/WaxCore/Extraction/GraphWalkResult.swift
import Foundation

/// A single fact found during graph traversal.
public struct GraphWalkHit: Sendable, Equatable {
    public var factId: FactRowID
    public var subject: EntityKey
    public var predicate: PredicateKey
    public var object: FactValue
    public var hopDistance: Int
    public var confidence: Double?
    public var evidenceFrameIds: [UInt64]

    /// Decay factor: 1.0 / (1 + hopDistance)
    public var hopDecay: Float {
        1.0 / Float(1 + hopDistance)
    }

    public init(
        factId: FactRowID,
        subject: EntityKey,
        predicate: PredicateKey,
        object: FactValue,
        hopDistance: Int,
        confidence: Double?,
        evidenceFrameIds: [UInt64]
    ) {
        self.factId = factId
        self.subject = subject
        self.predicate = predicate
        self.object = object
        self.hopDistance = hopDistance
        self.confidence = confidence
        self.evidenceFrameIds = evidenceFrameIds
    }
}

/// Result of a graph walk from resolved entities.
public struct GraphWalkResult: Sendable, Equatable {
    public var resolvedEntities: [EntityKey]
    public var hits: [GraphWalkHit]

    /// All unique evidence frame IDs across all hits.
    public var allEvidenceFrameIds: Set<UInt64> {
        Set(hits.flatMap(\.evidenceFrameIds))
    }

    public init(resolvedEntities: [EntityKey], hits: [GraphWalkHit]) {
        self.resolvedEntities = resolvedEntities
        self.hits = hits
    }
}
```

**Step 4: Run test to verify it passes**

Run: `swift test --filter GraphWalkResultTests`
Expected: PASS

**Step 5: Commit**

```bash
git add Sources/WaxCore/Extraction/GraphWalkResult.swift Tests/WaxCoreTests/
git commit -m "feat(core): add GraphWalkResult types for 2-hop graph traversal"
```

**Step 6: Write the failing integration test for edges() on FTS5SearchEngine**

```swift
// Tests/WaxIntegrationTests/GraphWalkerIntegrationTests.swift
import Testing
@testable import Wax
@testable import WaxCore
@testable import WaxTextSearch

struct GraphWalkerIntegrationTests {
    @Test func edgesReturnsOutboundEntityRelationships() async throws {
        // Setup: create entities and entity-valued facts
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        // Create entities
        _ = try await memory.upsertEntity(key: EntityKey("user:chris"), kind: "person", aliases: ["Chris"])
        _ = try await memory.upsertEntity(key: EntityKey("project:wax"), kind: "project", aliases: ["Wax"])

        // Assert entity-valued fact (object_kind == 7)
        _ = try await memory.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("works_on"),
            object: .entity(EntityKey("project:wax"))
        )

        // Query outbound edges from chris
        let edges = try await memory.edges(
            from: EntityKey("user:chris"),
            direction: .outbound,
            limit: 10
        )

        #expect(edges.hits.count == 1)
        #expect(edges.hits[0].predicate == PredicateKey("works_on"))
        #expect(edges.hits[0].neighbor == EntityKey("project:wax"))
    }

    @Test func walkGraphReturns2HopFacts() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        // Build graph: chris -> works_on -> wax, wax -> uses -> sqlite
        _ = try await memory.upsertEntity(key: EntityKey("user:chris"), kind: "person", aliases: ["Chris"])
        _ = try await memory.upsertEntity(key: EntityKey("project:wax"), kind: "project", aliases: ["Wax"])

        _ = try await memory.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("guard clauses")
        )
        _ = try await memory.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("works_on"),
            object: .entity(EntityKey("project:wax"))
        )
        _ = try await memory.assertFact(
            subject: EntityKey("project:wax"),
            predicate: PredicateKey("uses"),
            object: .string("SQLite")
        )

        // Walk 2 hops from chris
        let context = StructuredMemoryQueryContext(
            asOf: .latest, maxResults: 50, maxTraversalEdges: 100, maxDepth: 2
        )
        let result = try await memory.walkGraph(
            from: EntityKey("user:chris"),
            context: context
        )

        #expect(result.resolvedEntities == [EntityKey("user:chris")])
        // Hop 0: chris->prefers->guard clauses, chris->works_on->wax
        // Hop 1: wax->uses->SQLite
        #expect(result.hits.count == 3)
        let hop0 = result.hits.filter { $0.hopDistance == 0 }
        let hop1 = result.hits.filter { $0.hopDistance == 1 }
        #expect(hop0.count == 2)
        #expect(hop1.count == 1)
        #expect(hop1[0].predicate == PredicateKey("uses"))
    }
}
```

**Step 7: Run test to verify it fails**

Run: `swift test --filter GraphWalkerIntegrationTests`
Expected: FAIL — `edges()` and `walkGraph()` don't exist on MemoryOrchestrator

**Step 8: Implement edges() on FTS5SearchEngine**

Add to `Sources/WaxTextSearch/FTS5SearchEngine.swift` after the existing `facts()` method (~line 396):

```swift
/// Query entity-to-entity edges (facts where object_kind == 7).
public func edges(
    from entity: EntityKey,
    direction: StructuredEdgeDirection,
    asOf: StructuredMemoryAsOf = .latest,
    limit: Int = 50
) async throws -> StructuredEdgesResult {
    try await flush()
    return try db.read { db in
        let sql: String
        switch direction {
        case .outbound:
            sql = """
                SELECT f.fact_id, p.key AS predicate, oe.key AS neighbor
                FROM sm_fact f
                JOIN sm_entity se ON f.subject_entity_id = se.entity_id
                JOIN sm_predicate p ON f.predicate_id = p.predicate_id
                JOIN sm_entity oe ON f.object_entity_id = oe.entity_id
                JOIN sm_fact_span s ON s.fact_id = f.fact_id
                WHERE se.key = ?
                  AND f.object_kind = 7
                  AND s.system_from_ms <= ?
                  AND (s.system_to_ms IS NULL OR s.system_to_ms > ?)
                  AND s.valid_from_ms <= ?
                  AND (s.valid_to_ms IS NULL OR s.valid_to_ms > ?)
                ORDER BY f.fact_id
                LIMIT ?
                """
        case .inbound:
            sql = """
                SELECT f.fact_id, p.key AS predicate, se.key AS neighbor
                FROM sm_fact f
                JOIN sm_entity se ON f.subject_entity_id = se.entity_id
                JOIN sm_predicate p ON f.predicate_id = p.predicate_id
                JOIN sm_entity oe ON f.object_entity_id = oe.entity_id
                JOIN sm_fact_span s ON s.fact_id = f.fact_id
                WHERE oe.key = ?
                  AND f.object_kind = 7
                  AND s.system_from_ms <= ?
                  AND (s.system_to_ms IS NULL OR s.system_to_ms > ?)
                  AND s.valid_from_ms <= ?
                  AND (s.valid_to_ms IS NULL OR s.valid_to_ms > ?)
                ORDER BY f.fact_id
                LIMIT ?
                """
        }
        let args: [DatabaseValueConvertible] = [
            entity.rawValue,
            asOf.systemTimeMs, asOf.systemTimeMs,
            asOf.validTimeMs, asOf.validTimeMs,
            limit
        ]
        var hits: [EdgeHit] = []
        let rows = try Row.fetchAll(db, sql: sql, arguments: StatementArguments(args))
        for row in rows {
            hits.append(EdgeHit(
                factId: FactRowID(row["fact_id"] as Int64),
                predicate: PredicateKey(row["predicate"] as String),
                direction: direction,
                neighbor: EntityKey(row["neighbor"] as String)
            ))
        }
        return StructuredEdgesResult(hits: hits, wasTruncated: hits.count >= limit)
    }
}
```

**Step 9: Implement walkGraph() on FTS5SearchEngine**

Add after edges():

```swift
/// Walk the entity graph up to `maxDepth` hops, collecting facts and edges.
public func walkGraph(
    from entity: EntityKey,
    context: StructuredMemoryQueryContext
) async throws -> GraphWalkResult {
    var allHits: [GraphWalkHit] = []
    var visited = Set<EntityKey>()
    var frontier: [EntityKey] = [entity]

    for depth in 0..<context.maxDepth {
        guard !frontier.isEmpty else { break }
        var nextFrontier: [EntityKey] = []

        for current in frontier where !visited.contains(current) {
            visited.insert(current)

            // Get all facts about this entity
            let factsResult = try await facts(
                about: current,
                predicate: nil,
                asOf: context.asOf,
                limit: context.maxTraversalEdges
            )

            // Get evidence frame IDs for this entity
            let evidenceIds = try await evidenceFrameIds(
                forSubjects: [current],
                asOf: context.asOf,
                limit: context.maxTraversalEdges
            )

            for hit in factsResult.hits {
                allHits.append(GraphWalkHit(
                    factId: hit.factId,
                    subject: hit.fact.subject,
                    predicate: hit.fact.predicate,
                    object: hit.fact.object,
                    hopDistance: depth,
                    confidence: hit.evidence.first?.confidence,
                    evidenceFrameIds: evidenceIds
                ))

                // If this fact points to another entity, add to next frontier
                if case .entity(let neighbor) = hit.fact.object,
                   !visited.contains(neighbor) {
                    nextFrontier.append(neighbor)
                }
            }

            guard allHits.count < context.maxResults else { break }
        }

        frontier = nextFrontier
    }

    return GraphWalkResult(
        resolvedEntities: [entity],
        hits: Array(allHits.prefix(context.maxResults))
    )
}
```

**Step 10: Expose on WaxSession**

Add to `Sources/Wax/WaxSession.swift` after `resolveEntities()` (~line 236):

```swift
public func edges(
    from entity: EntityKey,
    direction: StructuredEdgeDirection,
    asOf: StructuredMemoryAsOf = .latest,
    limit: Int = 50
) async throws -> StructuredEdgesResult {
    guard config.enableStructuredMemory, let textEngine else {
        throw WaxError.io("structured memory is disabled")
    }
    return try await textEngine.edges(from: entity, direction: direction, asOf: asOf, limit: limit)
}

public func walkGraph(
    from entity: EntityKey,
    context: StructuredMemoryQueryContext
) async throws -> GraphWalkResult {
    guard config.enableStructuredMemory, let textEngine else {
        throw WaxError.io("structured memory is disabled")
    }
    return try await textEngine.walkGraph(from: entity, context: context)
}
```

**Step 11: Expose on MemoryOrchestrator**

Add to `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` after `resolveEntities()` (~line 851):

```swift
public func edges(
    from entity: EntityKey,
    direction: StructuredEdgeDirection,
    limit: Int = 50
) async throws -> StructuredEdgesResult {
    try ensureStructuredMemoryEnabled()
    return try await session.edges(from: entity, direction: direction, limit: limit)
}

public func walkGraph(
    from entity: EntityKey,
    context: StructuredMemoryQueryContext
) async throws -> GraphWalkResult {
    try ensureStructuredMemoryEnabled()
    return try await session.walkGraph(from: entity, context: context)
}
```

**Step 12: Run integration test to verify it passes**

Run: `swift test --filter GraphWalkerIntegrationTests`
Expected: PASS

**Step 13: Commit**

```bash
git add Sources/WaxTextSearch/FTS5SearchEngine.swift Sources/Wax/WaxSession.swift \
  Sources/Wax/Orchestrator/MemoryOrchestrator.swift Tests/WaxIntegrationTests/
git commit -m "feat(graph): implement 2-hop graph traversal with edges() and walkGraph()"
```

---

### Task 3: ContradictionDetector

Detect and auto-retract conflicting facts (same subject + predicate, different object).

**Files:**
- Create: `Sources/Wax/Graph/ContradictionDetector.swift`
- Test: `Tests/WaxIntegrationTests/ContradictionDetectorTests.swift`

**Step 1: Write the failing test**

```swift
// Tests/WaxIntegrationTests/ContradictionDetectorTests.swift
import Testing
@testable import Wax
@testable import WaxCore

struct ContradictionDetectorTests {
    @Test func detectsConflictingFacts() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)
        _ = try await memory.upsertEntity(key: EntityKey("user:chris"), kind: "person")

        // Assert first fact
        let fact1 = try await memory.assertFact(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("prefers_editor"),
            object: .string("Vim")
        )

        // Assert contradicting fact — should auto-retract the first
        let fact2 = try await memory.assertFactWithContradictionCheck(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("prefers_editor"),
            object: .string("Neovim")
        )

        // Old fact should be retracted
        let facts = try await memory.facts(about: EntityKey("user:chris"), predicate: PredicateKey("prefers_editor"))
        #expect(facts.hits.count == 1)
        #expect(facts.hits[0].factId == fact2.factId)
    }

    @Test func allowsMultipleFactsWithDifferentPredicates() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)
        _ = try await memory.upsertEntity(key: EntityKey("user:chris"), kind: "person")

        _ = try await memory.assertFactWithContradictionCheck(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("prefers_editor"),
            object: .string("Neovim")
        )
        _ = try await memory.assertFactWithContradictionCheck(
            subject: EntityKey("user:chris"),
            predicate: PredicateKey("prefers_language"),
            object: .string("Swift")
        )

        let facts = try await memory.facts(about: EntityKey("user:chris"))
        #expect(facts.hits.count == 2)
    }
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --filter ContradictionDetectorTests`
Expected: FAIL — `assertFactWithContradictionCheck` doesn't exist

**Step 3: Write ContradictionDetector**

```swift
// Sources/Wax/Graph/ContradictionDetector.swift
import Foundation
import WaxCore

/// Result of a contradiction check during fact assertion.
public struct ContradictionCheckResult: Sendable, Equatable {
    public var factId: FactRowID
    public var retractedFactIds: [FactRowID]
    public var hadContradiction: Bool

    public init(factId: FactRowID, retractedFactIds: [FactRowID]) {
        self.factId = factId
        self.retractedFactIds = retractedFactIds
        self.hadContradiction = !retractedFactIds.isEmpty
    }
}
```

**Step 4: Add `assertFactWithContradictionCheck` to MemoryOrchestrator**

Add to `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` after `assertFact()`:

```swift
/// Assert a fact, automatically retracting any existing facts with the same
/// subject + predicate but different object value (contradiction resolution).
public func assertFactWithContradictionCheck(
    subject: EntityKey,
    predicate: PredicateKey,
    object: FactValue,
    validFromMs: Int64? = nil,
    validToMs: Int64? = nil,
    evidence: [StructuredEvidence] = [],
    commit: Bool = true
) async throws -> ContradictionCheckResult {
    try ensureStructuredMemoryEnabled()
    let nowMs = Int64(Date().timeIntervalSince1970 * 1000)

    // Check for existing facts with same subject + predicate
    let existing = try await session.facts(
        about: subject,
        predicate: predicate,
        asOf: .latest,
        limit: 100
    )

    // Retract contradicting facts (different object value)
    var retractedIds: [FactRowID] = []
    for hit in existing.hits where hit.fact.object != object {
        try await session.retractFact(factId: hit.factId, atMs: nowMs)
        retractedIds.append(hit.factId)
    }

    // Assert the new fact
    let valid = StructuredTimeRange(fromMs: validFromMs ?? nowMs, toMs: validToMs)
    let system = StructuredTimeRange(fromMs: nowMs, toMs: nil)
    let factId = try await session.assertFact(
        subject: subject,
        predicate: predicate,
        object: object,
        valid: valid,
        system: system,
        evidence: evidence
    )

    if commit {
        try await session.commit()
    }

    return ContradictionCheckResult(factId: factId, retractedFactIds: retractedIds)
}
```

**Step 5: Run test to verify it passes**

Run: `swift test --filter ContradictionDetectorTests`
Expected: PASS

**Step 6: Commit**

```bash
git add Sources/Wax/Graph/ Tests/WaxIntegrationTests/ContradictionDetectorTests.swift
git commit -m "feat(graph): add contradiction detection for fact assertion"
```

---

### Task 4: Apple Foundation Models Entity Extractor

On-device entity/fact extraction using Apple Foundation Models guided generation. Graceful degradation on unsupported platforms.

**Files:**
- Create: `Sources/Wax/Extraction/FoundationModelExtractor.swift`
- Create: `Sources/Wax/Extraction/NoOpExtractor.swift`
- Test: `Tests/WaxCoreTests/Extraction/NoOpExtractorTests.swift`
- Test: `Tests/WaxIntegrationTests/FoundationModelExtractorTests.swift`

**Step 1: Write the failing test for NoOpExtractor**

```swift
// Tests/WaxCoreTests/Extraction/NoOpExtractorTests.swift
import Testing
@testable import WaxCore

struct NoOpExtractorTests {
    @Test func noOpExtractorReturnsEmptyResult() async throws {
        let extractor = NoOpExtractor()
        let result = try await extractor.extract(from: "Chris prefers Swift")
        #expect(result.isEmpty)
        #expect(extractor.isAvailable == true)
    }
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --filter NoOpExtractorTests`
Expected: FAIL — NoOpExtractor not found

**Step 3: Implement NoOpExtractor (fallback for unsupported platforms)**

```swift
// Sources/WaxCore/Extraction/NoOpExtractor.swift
import Foundation

/// Fallback extractor that does nothing. Used when Foundation Models
/// is not available (older OS, Linux, CI).
public struct NoOpExtractor: EntityExtractor, Sendable {
    public init() {}

    public func extract(from text: String) async throws -> ExtractionResult {
        ExtractionResult(entities: [], facts: [])
    }

    public var isAvailable: Bool { true }
}
```

**Step 4: Run test to verify it passes**

Run: `swift test --filter NoOpExtractorTests`
Expected: PASS

**Step 5: Commit**

```bash
git add Sources/WaxCore/Extraction/NoOpExtractor.swift Tests/WaxCoreTests/Extraction/
git commit -m "feat(extraction): add NoOpExtractor fallback for unsupported platforms"
```

**Step 6: Implement FoundationModelExtractor**

```swift
// Sources/Wax/Extraction/FoundationModelExtractor.swift
import Foundation
import WaxCore

#if canImport(FoundationModels)
import FoundationModels

/// Schema for guided generation output.
@Generable
struct ExtractionOutput {
    @Guide(description: "Entities found in the text")
    var entities: [EntityOutput]
    @Guide(description: "Facts/relationships found in the text")
    var facts: [FactOutput]
}

@Generable
struct EntityOutput {
    @Guide(description: "Namespaced key like 'person:name' or 'project:name'")
    var key: String
    @Guide(description: "Entity kind: person, project, tool, language, concept, preference")
    var kind: String
    @Guide(description: "Alternative names for this entity")
    var aliases: [String]
}

@Generable
struct FactOutput {
    @Guide(description: "Subject entity key")
    var subject: String
    @Guide(description: "Relationship predicate like 'prefers', 'uses', 'works_on'")
    var predicate: String
    @Guide(description: "Object value (string)")
    var object: String
    @Guide(description: "Whether the object refers to another entity key")
    var objectIsEntity: Bool
}

/// Entity extractor using Apple Foundation Models (macOS 26+ / iOS 26+).
/// Uses guided generation for structured output.
public final class FoundationModelExtractor: EntityExtractor, @unchecked Sendable {
    private let systemPrompt: String

    public init() {
        self.systemPrompt = """
        Extract entities and factual relationships from the given text.
        Entities should use namespaced keys like "person:chris", "project:wax", "tool:sqlite".
        Facts should capture relationships as subject-predicate-object triples.
        Common predicates: prefers, uses, works_on, knows, dislikes, configured_with, built_with.
        Only extract clearly stated facts. Do not infer or speculate.
        """
    }

    public var isAvailable: Bool {
        LanguageModelSession.isAvailable
    }

    public func extract(from text: String) async throws -> ExtractionResult {
        guard isAvailable else {
            return ExtractionResult(entities: [], facts: [])
        }

        let session = LanguageModelSession(instructions: systemPrompt)
        let output = try await session.respond(
            to: "Extract entities and facts from this text:\n\n\(text)",
            generating: ExtractionOutput.self
        )

        let entities = output.entities.map { e in
            ExtractedEntity(
                key: EntityKey(e.key),
                kind: e.kind,
                aliases: e.aliases
            )
        }

        let facts = output.facts.map { f in
            ExtractedFact(
                subject: EntityKey(f.subject),
                predicate: PredicateKey(f.predicate),
                object: f.objectIsEntity ? .entity(EntityKey(f.object)) : .string(f.object),
                confidence: 0.85 // Foundation Models default confidence
            )
        }

        return ExtractionResult(entities: entities, facts: facts)
    }
}

#else

/// Stub for platforms without Foundation Models.
public final class FoundationModelExtractor: EntityExtractor, @unchecked Sendable {
    public init() {}
    public var isAvailable: Bool { false }

    public func extract(from text: String) async throws -> ExtractionResult {
        ExtractionResult(entities: [], facts: [])
    }
}

#endif
```

**Step 7: Write integration test (only runs on macOS 26+)**

```swift
// Tests/WaxIntegrationTests/FoundationModelExtractorTests.swift
import Testing
@testable import Wax
@testable import WaxCore

struct FoundationModelExtractorTests {
    @Test func extractorReportsAvailability() {
        let extractor = FoundationModelExtractor()
        // On CI (Linux/older macOS): false. On macOS 26+: true.
        // Just verify it doesn't crash.
        _ = extractor.isAvailable
    }

    @Test(.enabled(if: FoundationModelExtractor().isAvailable))
    func extractsEntitiesAndFactsFromText() async throws {
        let extractor = FoundationModelExtractor()
        let result = try await extractor.extract(
            from: "Chris prefers using guard clauses for error handling in Swift projects."
        )
        // At minimum, should extract a person entity
        #expect(!result.entities.isEmpty)
    }
}
```

**Step 8: Run tests**

Run: `swift test --filter FoundationModelExtractorTests`
Expected: PASS (availability test always passes, extraction test skipped on CI)

**Step 9: Commit**

```bash
git add Sources/Wax/Extraction/ Tests/WaxIntegrationTests/FoundationModelExtractorTests.swift
git commit -m "feat(extraction): add Apple Foundation Models entity extractor"
```

---

## Phase 2: Graph-Fused Recall Pipeline

### Task 5: Graph-Fused RAG Builder

Extend the recall pipeline to query the knowledge graph alongside text/vector search.

**Files:**
- Create: `Sources/Wax/RAG/GraphFusedRAGBuilder.swift`
- Modify: `Sources/Wax/RAG/RAGContext.swift` (add `.fact` item kind)
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` (update recall flow)
- Test: `Tests/WaxIntegrationTests/GraphFusedRecallTests.swift`

**Step 1: Write the failing test**

```swift
// Tests/WaxIntegrationTests/GraphFusedRecallTests.swift
import Testing
@testable import Wax
@testable import WaxCore

struct GraphFusedRecallTests {
    @Test func recallWithGraphReturnsFactItems() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        // Store content via remember (creates frame)
        try await memory.remember("Chris prefers guard-clause error handling in Swift")
        try await memory.flush()

        // Manually add graph data (simulating extraction)
        _ = try await memory.upsertEntity(key: EntityKey("person:chris"), kind: "person", aliases: ["Chris"])
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("guard-clause error handling")
        )

        // Recall with graph enabled
        let context = try await memory.recall(query: "Chris error handling", graphEnabled: true)

        // Should have both frame-based and fact-based items
        let factItems = context.items.filter { $0.kind == .fact }
        let frameItems = context.items.filter { $0.kind != .fact }
        #expect(!factItems.isEmpty, "Graph facts should appear in recall results")
        #expect(!frameItems.isEmpty, "Frame results should still appear")
    }

    @Test func recallWithGraphDisabledSkipsGraph() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        try await memory.remember("Chris prefers guard-clause error handling")
        try await memory.flush()
        _ = try await memory.upsertEntity(key: EntityKey("person:chris"), kind: "person", aliases: ["Chris"])
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("guard-clause error handling")
        )

        // Recall with graph disabled — regression test
        let context = try await memory.recall(query: "Chris error handling", graphEnabled: false)
        let factItems = context.items.filter { $0.kind == .fact }
        #expect(factItems.isEmpty, "No fact items when graph disabled")
    }

    @Test func recallWithProjectScopeFiltersResults() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        // Store content tagged with projects
        try await memory.remember("Auth uses JWT tokens", metadata: ["project": "projectA"])
        try await memory.remember("Auth uses session cookies", metadata: ["project": "projectB"])
        try await memory.flush()

        // Recall scoped to projectA
        let context = try await memory.recall(
            query: "auth",
            project: "projectA"
        )

        // Should only return projectA results
        #expect(context.items.count >= 1)
        // projectB content should not appear
    }
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --filter GraphFusedRecallTests`
Expected: FAIL — new parameters don't exist on recall()

**Step 3: Add `.fact` to RAGContext.ItemKind**

Modify `Sources/Wax/RAG/RAGContext.swift` line 3:

```swift
public enum ItemKind: Sendable, Equatable {
    case snippet
    case expanded
    case surrogate
    case fact  // NEW: graph-sourced fact
}
```

**Step 4: Implement GraphFusedRAGBuilder**

```swift
// Sources/Wax/RAG/GraphFusedRAGBuilder.swift
import Foundation
import WaxCore

/// Scores and converts graph walk results into RAGContext items.
struct GraphFusedRAGBuilder: Sendable {

    struct Config: Sendable {
        var graphWeight: Float = 0.4
        var textWeight: Float = 0.3
        var vectorWeight: Float = 0.3
        var maxGraphItems: Int = 10
    }

    /// Build graph-sourced RAGContext items from a graph walk result.
    static func buildGraphItems(
        from walkResult: GraphWalkResult,
        config: Config = .init()
    ) -> [RAGContext.Item] {
        walkResult.hits
            .sorted { a, b in
                // Sort by hop (lower first), then confidence (higher first)
                if a.hopDistance != b.hopDistance { return a.hopDistance < b.hopDistance }
                return (a.confidence ?? 0) > (b.confidence ?? 0)
            }
            .prefix(config.maxGraphItems)
            .map { hit in
                let text = formatFactAsText(hit)
                let score = computeGraphScore(hit, weight: config.graphWeight)
                return RAGContext.Item(
                    kind: .fact,
                    frameId: hit.evidenceFrameIds.first ?? 0,
                    score: score,
                    sources: [.structuredMemory],
                    text: text
                )
            }
    }

    /// Format a graph fact as human-readable text for LLM consumption.
    static func formatFactAsText(_ hit: GraphWalkHit) -> String {
        let objectStr: String
        switch hit.object {
        case .string(let s): objectStr = s
        case .int(let i): objectStr = String(i)
        case .double(let d): objectStr = String(d)
        case .bool(let b): objectStr = b ? "true" : "false"
        case .entity(let e): objectStr = e.rawValue
        case .timeMs(let t): objectStr = "time:\(t)"
        case .data: objectStr = "<data>"
        }
        return "\(hit.subject.rawValue) \(hit.predicate.rawValue) \(objectStr)"
    }

    /// Compute score for a graph hit with hop decay and recency weighting.
    static func computeGraphScore(_ hit: GraphWalkHit, weight: Float) -> Float {
        let baseConfidence = Float(hit.confidence ?? 0.5)
        let hopDecay = hit.hopDecay
        return weight * baseConfidence * hopDecay
    }

    /// Merge graph items into existing RAG context items, deduplicating by frameId.
    static func merge(
        graphItems: [RAGContext.Item],
        frameItems: [RAGContext.Item]
    ) -> [RAGContext.Item] {
        let existingFrameIds = Set(frameItems.map(\.frameId))
        // Graph items with evidence frames already in results get boosted, not duplicated
        var boostedFrameItems = frameItems
        var newGraphItems: [RAGContext.Item] = []

        for graphItem in graphItems {
            if graphItem.frameId != 0, existingFrameIds.contains(graphItem.frameId) {
                // Boost existing frame item score
                if let idx = boostedFrameItems.firstIndex(where: { $0.frameId == graphItem.frameId }) {
                    boostedFrameItems[idx].score += graphItem.score * 0.5
                    if !boostedFrameItems[idx].sources.contains(.structuredMemory) {
                        boostedFrameItems[idx].sources.append(.structuredMemory)
                    }
                }
            } else {
                newGraphItems.append(graphItem)
            }
        }

        // Combine and re-sort by score
        return (boostedFrameItems + newGraphItems).sorted { $0.score > $1.score }
    }
}
```

**Step 5: Update MemoryOrchestrator.recall() with graph fusion**

Add new recall overloads to `Sources/Wax/Orchestrator/MemoryOrchestrator.swift`:

```swift
/// Graph-aware recall with optional project scoping.
public func recall(
    query: String,
    project: String? = nil,
    graphEnabled: Bool = true,
    maxDepth: Int = 2,
    embeddingPolicy: QueryEmbeddingPolicy = .ifAvailable
) async throws -> RAGContext {
    // Build frame filter for project scoping
    var frameFilter: FrameFilter? = nil
    if let project {
        frameFilter = FrameFilter(metadataFilter: MetadataFilter(
            requiredEntries: ["project": project],
            requiredTags: [],
            requiredLabels: []
        ))
    }

    // Get embedding
    let embedding = try await resolveEmbedding(query: query, policy: embeddingPolicy)

    // Build base RAG context (text + vector)
    var context = try await buildRecallContext(
        query: query,
        embedding: embedding,
        frameFilter: frameFilter
    )

    // Graph fusion (if enabled and structured memory is available)
    if graphEnabled, config.enableStructuredMemory {
        let graphItems = try await buildGraphContext(
            query: query,
            maxDepth: maxDepth
        )
        context = RAGContext(
            query: context.query,
            items: GraphFusedRAGBuilder.merge(
                graphItems: graphItems,
                frameItems: context.items
            ),
            totalTokens: context.totalTokens
        )
    }

    await recordAccessesIfEnabled(frameIds: context.items.map(\.frameId))
    return context
}

/// Build graph context items by resolving entities in the query and walking the graph.
private func buildGraphContext(
    query: String,
    maxDepth: Int
) async throws -> [RAGContext.Item] {
    // Tokenize query into words and try resolving each as an entity alias
    let words = query.split(separator: " ").map(String.init)
    var resolvedEntities: [EntityKey] = []

    // Try single words and bigrams
    for word in words {
        let matches = try await session.resolveEntities(matchingAlias: word, limit: 3)
        resolvedEntities.append(contentsOf: matches.map(\.key))
    }
    for i in 0..<max(0, words.count - 1) {
        let bigram = "\(words[i]) \(words[i + 1])"
        let matches = try await session.resolveEntities(matchingAlias: bigram, limit: 3)
        resolvedEntities.append(contentsOf: matches.map(\.key))
    }

    // Deduplicate
    let uniqueEntities = Array(Set(resolvedEntities))
    guard !uniqueEntities.isEmpty else { return [] }

    // Walk graph from each resolved entity
    let queryContext = StructuredMemoryQueryContext(
        asOf: .latest,
        maxResults: 20,
        maxTraversalEdges: 50,
        maxDepth: maxDepth
    )

    var allGraphItems: [RAGContext.Item] = []
    for entity in uniqueEntities.prefix(5) {  // Cap at 5 entities
        let walkResult = try await session.walkGraph(from: entity, context: queryContext)
        let items = GraphFusedRAGBuilder.buildGraphItems(from: walkResult)
        allGraphItems.append(contentsOf: items)
    }

    return allGraphItems
}
```

**Step 6: Run test to verify it passes**

Run: `swift test --filter GraphFusedRecallTests`
Expected: PASS

**Step 7: Run full test suite for regression**

Run: `swift test`
Expected: All existing tests PASS

**Step 8: Commit**

```bash
git add Sources/Wax/RAG/ Sources/Wax/Orchestrator/MemoryOrchestrator.swift \
  Tests/WaxIntegrationTests/GraphFusedRecallTests.swift
git commit -m "feat(recall): graph-fused RAG pipeline with 3-source fusion"
```

---

### Task 6: Auto-Extraction on Remember

Wire the EntityExtractor into the remember() flow so entities and facts are extracted automatically.

**Files:**
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` (add extractor field, update remember)
- Modify: `Sources/Wax/Orchestrator/OrchestratorConfig.swift` (add extraction config)
- Test: `Tests/WaxIntegrationTests/AutoExtractionTests.swift`

**Step 1: Write the failing test**

```swift
// Tests/WaxIntegrationTests/AutoExtractionTests.swift
import Testing
@testable import Wax
@testable import WaxCore

/// Mock extractor for testing (deterministic, no ML).
struct MockExtractor: EntityExtractor {
    var isAvailable: Bool { true }

    func extract(from text: String) async throws -> ExtractionResult {
        // Simple rule: if text contains "prefers", extract a preference fact
        guard text.lowercased().contains("prefers") else {
            return ExtractionResult(entities: [], facts: [])
        }
        return ExtractionResult(
            entities: [ExtractedEntity(key: EntityKey("person:test"), kind: "person", aliases: ["Test"])],
            facts: [ExtractedFact(
                subject: EntityKey("person:test"),
                predicate: PredicateKey("prefers"),
                object: .string("testing"),
                confidence: 0.9
            )]
        )
    }
}

struct AutoExtractionTests {
    @Test func rememberWithExtractionCreatesGraphEntities() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(
            structured: true,
            extractor: MockExtractor()
        )

        try await memory.remember("Test prefers writing tests first", extractEntities: true)
        try await memory.flush()

        // Verify entity was created
        let entities = try await memory.resolveEntities(matchingAlias: "Test")
        #expect(!entities.isEmpty)

        // Verify fact was created
        let facts = try await memory.facts(about: EntityKey("person:test"))
        #expect(facts.hits.count >= 1)
    }

    @Test func rememberWithExtractionDisabledSkipsExtraction() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(
            structured: true,
            extractor: MockExtractor()
        )

        try await memory.remember("Test prefers writing tests first", extractEntities: false)
        try await memory.flush()

        let entities = try await memory.resolveEntities(matchingAlias: "Test")
        #expect(entities.isEmpty)
    }
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --filter AutoExtractionTests`
Expected: FAIL — new parameters don't exist

**Step 3: Update OrchestratorConfig**

Add to `Sources/Wax/Orchestrator/OrchestratorConfig.swift`:

```swift
public var enableAutoExtraction: Bool = false
```

**Step 4: Add extractor to MemoryOrchestrator**

Add an `extractor` property to `MemoryOrchestrator` and update `remember()`:

```swift
// Property
private let extractor: (any EntityExtractor)?

// In init or factory, accept optional extractor
// Update remember() to call extraction after frame storage:

public func remember(
    _ content: String,
    metadata: [String: String] = [:],
    extractEntities: Bool? = nil
) async throws -> RememberResult {
    let result = try await existingRemember(content, metadata: metadata)

    let shouldExtract = extractEntities ?? config.enableAutoExtraction
    if shouldExtract, config.enableStructuredMemory, let extractor, extractor.isAvailable {
        let extraction = try await extractor.extract(from: content)

        for entity in extraction.entities {
            _ = try await upsertEntity(
                key: entity.key,
                kind: entity.kind,
                aliases: entity.aliases,
                commit: false
            )
        }

        for fact in extraction.facts {
            let evidence = [StructuredEvidence(
                sourceFrameId: result.frameId,
                chunkIndex: nil,
                spanUTF8: nil,
                extractorId: "foundation_models",
                extractorVersion: "1.0",
                confidence: fact.confidence,
                assertedAtMs: Int64(Date().timeIntervalSince1970 * 1000)
            )]
            _ = try await assertFactWithContradictionCheck(
                subject: fact.subject,
                predicate: fact.predicate,
                object: fact.object,
                evidence: evidence,
                commit: false
            )
        }

        try await session.commit()
    }

    return result
}
```

**Step 5: Run test to verify it passes**

Run: `swift test --filter AutoExtractionTests`
Expected: PASS

**Step 6: Commit**

```bash
git add Sources/Wax/Orchestrator/ Tests/WaxIntegrationTests/AutoExtractionTests.swift
git commit -m "feat(extraction): auto-extract entities and facts on remember()"
```

---

## Phase 3: New Agent Tools

### Task 7: wax_forget — Natural Language Fact Retraction

**Files:**
- Create: `Sources/Wax/Graph/ForgetResolver.swift`
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` (add forget method)
- Create: `Sources/WaxCLI/ForgetCommand.swift`
- Modify: `Sources/WaxMCPServer/WaxMCPTools.swift` (add wax_forget handler)
- Modify: `Sources/WaxMCPServer/ToolSchemas.swift` (add wax_forget schema)
- Test: `Tests/WaxIntegrationTests/ForgetTests.swift`

**Step 1: Write the failing test**

```swift
// Tests/WaxIntegrationTests/ForgetTests.swift
import Testing
@testable import Wax
@testable import WaxCore

struct ForgetTests {
    @Test func forgetByFactIdRetractsFact() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)
        _ = try await memory.upsertEntity(key: EntityKey("person:chris"), kind: "person")
        let factId = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("Vim")
        )

        let result = try await memory.forget(factId: factId)
        #expect(result.retractedCount == 1)

        let facts = try await memory.facts(about: EntityKey("person:chris"))
        #expect(facts.hits.isEmpty)
    }

    @Test func forgetByNaturalLanguageResolvesAndRetracts() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)
        _ = try await memory.upsertEntity(key: EntityKey("person:chris"), kind: "person", aliases: ["Chris"])
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("Vim")
        )
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("uses"),
            object: .string("Swift")
        )

        // Forget by natural language — should match "Chris" entity and "prefers" predicate
        let result = try await memory.forget(content: "Chris prefers Vim")
        #expect(result.retractedCount == 1)

        // "uses Swift" fact should remain
        let facts = try await memory.facts(about: EntityKey("person:chris"))
        #expect(facts.hits.count == 1)
        #expect(facts.hits[0].fact.predicate == PredicateKey("uses"))
    }
}
```

**Step 2-6: Implement ForgetResolver, add to orchestrator, CLI command, MCP tool**

The implementation follows the same TDD pattern:
1. `ForgetResolver` tokenizes the input, resolves entities via alias matching, finds matching facts by predicate/object similarity, retracts them.
2. `MemoryOrchestrator.forget(content:)` and `forget(factId:)` methods delegate to ForgetResolver.
3. CLI `ForgetCommand` with `--content` or `--fact-id` arguments.
4. MCP `wax_forget` tool with `content` or `fact_id` parameters.

**Step 7: Commit**

```bash
git commit -m "feat(forget): add wax_forget tool for natural language fact retraction"
```

---

### Task 8: wax_context — Entity Knowledge Card

**Files:**
- Create: `Sources/Wax/Graph/EntityContext.swift`
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` (add context method)
- Create: `Sources/WaxCLI/ContextCommand.swift`
- Modify: `Sources/WaxMCPServer/WaxMCPTools.swift` (add wax_context handler)
- Modify: `Sources/WaxMCPServer/ToolSchemas.swift` (add wax_context schema)
- Test: `Tests/WaxIntegrationTests/EntityContextTests.swift`

**Step 1: Write the failing test**

```swift
// Tests/WaxIntegrationTests/EntityContextTests.swift
import Testing
@testable import Wax
@testable import WaxCore

struct EntityContextTests {
    @Test func contextReturnsEntityKnowledgeCard() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        _ = try await memory.upsertEntity(key: EntityKey("person:chris"), kind: "person", aliases: ["Chris"])
        _ = try await memory.upsertEntity(key: EntityKey("project:wax"), kind: "project", aliases: ["Wax"])
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("Swift")
        )
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("works_on"),
            object: .entity(EntityKey("project:wax"))
        )

        let card = try await memory.entityContext(for: "Chris")

        #expect(card.entity.key == EntityKey("person:chris"))
        #expect(card.entity.kind == "person")
        #expect(card.facts.count == 2)
        #expect(card.neighbors.contains(EntityKey("project:wax")))
    }

    @Test func contextReturnsNilForUnknownEntity() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)
        let card = try await memory.entityContext(for: "nonexistent")
        #expect(card == nil)
    }
}
```

**Step 2-6: Implement EntityContext type, orchestrator method, CLI command, MCP tool**

The `EntityContext` struct contains: entity info, all facts, neighbor entities, and source frame IDs.

**Step 7: Commit**

```bash
git commit -m "feat(context): add wax_context tool for entity knowledge cards"
```

---

### Task 9: wax_reflect — Introspection + Proactive Insights

**Files:**
- Create: `Sources/Wax/Graph/PatternDetector.swift`
- Create: `Sources/Wax/Graph/ReflectionResult.swift`
- Modify: `Sources/Wax/Orchestrator/MemoryOrchestrator.swift` (add reflect method)
- Create: `Sources/WaxCLI/ReflectCommand.swift`
- Modify: `Sources/WaxMCPServer/WaxMCPTools.swift` (add wax_reflect handler)
- Modify: `Sources/WaxMCPServer/ToolSchemas.swift` (add wax_reflect schema)
- Test: `Tests/WaxIntegrationTests/ReflectTests.swift`

**Step 1: Write the failing test**

```swift
// Tests/WaxIntegrationTests/ReflectTests.swift
import Testing
@testable import Wax
@testable import WaxCore

struct ReflectTests {
    @Test func reflectReturnsSummaryStats() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        _ = try await memory.upsertEntity(key: EntityKey("person:chris"), kind: "person")
        _ = try await memory.upsertEntity(key: EntityKey("project:wax"), kind: "project")
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("prefers"),
            object: .string("Swift")
        )
        _ = try await memory.assertFact(
            subject: EntityKey("person:chris"),
            predicate: PredicateKey("works_on"),
            object: .entity(EntityKey("project:wax"))
        )

        let reflection = try await memory.reflect()
        #expect(reflection.totalEntities == 2)
        #expect(reflection.totalFacts == 2)
        #expect(!reflection.topEntities.isEmpty)
        #expect(reflection.topEntities[0].key == EntityKey("person:chris"))
        #expect(reflection.topEntities[0].factCount == 2)
    }

    @Test func reflectWithProjectFilterScopes() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        try await memory.remember("Project A uses React", metadata: ["project": "projectA"])
        try await memory.remember("Project B uses Vue", metadata: ["project": "projectB"])
        try await memory.flush()

        let reflection = try await memory.reflect(project: "projectA")
        // Should only count projectA frames
        #expect(reflection.frameCount >= 1)
    }
}
```

**Step 2-6: Implement PatternDetector, ReflectionResult, orchestrator method, CLI command, MCP tool**

The `PatternDetector` queries the graph using pure SQL analytics:
- Top entities by fact count
- Predicate frequency distribution
- Recent facts (last 7 days)
- Topic clusters (entities grouped by shared predicates)

**Step 7: Commit**

```bash
git commit -m "feat(reflect): add wax_reflect tool with proactive insights"
```

---

## Phase 4: Update Existing Tools

### Task 10: Update wax_remember MCP Tool + CLI

**Files:**
- Modify: `Sources/WaxMCPServer/ToolSchemas.swift` (add `project`, `extract` params to wax_remember)
- Modify: `Sources/WaxMCPServer/WaxMCPTools.swift` (pass new params)
- Modify: `Sources/WaxCLI/RememberCommand.swift` (add `--project`, `--extract` flags)
- Test: `Tests/WaxMCPServerTests/WaxMCPServerTests.swift` (add new tests)
- Test: `Tests/WaxCLITests/WaxCLIMemoryTests.swift` (add new tests)

**Step 1: Write failing MCP test**

Test that wax_remember accepts `project` and `extract` parameters and passes them through.

**Step 2-4: Update schema, handler, verify**

Add to wax_remember schema:
```json
{
    "project": { "type": "string", "description": "Project tag for scoping" },
    "extract": { "type": "boolean", "description": "Auto-extract entities and facts", "default": true }
}
```

**Step 5: Update CLI command**

Add `--project` and `--extract/--no-extract` options to RememberCommand.

**Step 6: Commit**

```bash
git commit -m "feat(tools): add project and extract params to wax_remember"
```

---

### Task 11: Update wax_recall MCP Tool + CLI

**Files:**
- Modify: `Sources/WaxMCPServer/ToolSchemas.swift` (add `project`, `graph`, `max_depth` to wax_recall)
- Modify: `Sources/WaxMCPServer/WaxMCPTools.swift` (pass new params)
- Modify: `Sources/WaxCLI/RecallCommand.swift` (add new flags)
- Test: `Tests/WaxMCPServerTests/` and `Tests/WaxCLITests/`

**Step 1: Write failing test**

Test that wax_recall with `graph: true` returns fact items and `project` scopes results.

**Step 2-4: Update schema, handler, verify**

Add to wax_recall schema:
```json
{
    "project": { "type": "string", "description": "Scope recall to this project" },
    "graph": { "type": "boolean", "description": "Enable knowledge graph fusion", "default": true },
    "max_depth": { "type": "integer", "description": "Graph walk depth (1-3)", "default": 2, "minimum": 1, "maximum": 3 }
}
```

**Step 5: Update CLI command**

Add `--project`, `--graph/--no-graph`, `--max-depth` options to RecallCommand.

**Step 6: Commit**

```bash
git commit -m "feat(tools): add graph fusion params to wax_recall"
```

---

### Task 12: Update MCP Tool Descriptions

Update all tool description strings in `ToolSchemas.swift` to teach LLMs when to use each tool. Good descriptions = better tool selection by agents.

**Files:**
- Modify: `Sources/WaxMCPServer/ToolSchemas.swift`

**Step 1: Write new descriptions**

```
wax_remember: "Store knowledge in memory. Content is automatically analyzed to extract entities and relationships into a knowledge graph. Tag with a project name to organize memories by codebase. Use this whenever you learn something worth remembering across sessions."

wax_recall: "Retrieve relevant memories using semantic search fused with knowledge graph traversal. Finds both text matches and related facts from the entity graph. Scope to a project to prioritize project-specific knowledge. Use this to find what you know about a topic."

wax_forget: "Correct or remove wrong knowledge. Describe what to forget in natural language (e.g. 'Chris doesn't use Vim anymore') or provide a specific fact_id. The system finds and retracts matching facts from the knowledge graph."

wax_context: "Get a complete knowledge card for an entity. Returns all known facts, relationships, and source references for a person, project, tool, or concept. Use this to bootstrap understanding of a specific entity."

wax_reflect: "Introspect your memory. Returns a summary of what you know: total entities, facts, top entities by fact count, recent learnings, and topic clusters. Optionally scope to a project. Use this to understand the breadth of your knowledge."

wax_handoff: "Store a session transition note with pending tasks for the next session to pick up."

wax_handoff_latest: "Load the most recent handoff note to resume where the last session left off."
```

**Step 2: Update schemas, verify build**

**Step 3: Commit**

```bash
git commit -m "feat(tools): improve MCP tool descriptions for better LLM tool selection"
```

---

## Phase 5: Registration and Wiring

### Task 13: Wire New Tools into MCP Server + CLI

Register all new commands and tools.

**Files:**
- Modify: `Sources/WaxCLI/WaxCLICommand.swift` (register ForgetCommand, ContextCommand, ReflectCommand)
- Modify: `Sources/WaxMCPServer/ToolSchemas.swift` (add schemas for new tools)
- Modify: `Sources/WaxMCPServer/WaxMCPTools.swift` (add handlers + dispatch cases)
- Modify: `Sources/WaxMCPServer/main.swift` (pass extractor to MemoryOrchestrator)

**Step 1: Register CLI subcommands**

Add to `WaxCLICommand.swift` subcommands array:
```swift
ForgetCommand.self,
ContextCommand.self,
ReflectCommand.self,
```

**Step 2: Register MCP tool dispatch**

Add cases in `WaxMCPTools.handleCall()`:
```swift
case "wax_forget": return try await forget(args, memory)
case "wax_context": return try await entityContext(args, memory)
case "wax_reflect": return try await reflect(args, memory)
```

**Step 3: Wire extractor in main.swift**

```swift
let extractor: any EntityExtractor = FoundationModelExtractor()
// Pass to MemoryOrchestrator init
```

**Step 4: Run full test suite**

Run: `swift test`
Expected: ALL tests pass

**Step 5: Commit**

```bash
git commit -m "feat(tools): register all new tools in MCP server and CLI"
```

---

## Phase 6: End-to-End Integration Tests

### Task 14: Full Pipeline Integration Tests

Test the complete agent workflow: remember → extract → recall → forget → context → reflect.

**Files:**
- Create: `Tests/WaxIntegrationTests/RelationalMemoryE2ETests.swift`

**Step 1: Write the end-to-end test**

```swift
// Tests/WaxIntegrationTests/RelationalMemoryE2ETests.swift
import Testing
@testable import Wax
@testable import WaxCore

struct RelationalMemoryE2ETests {
    @Test func fullAgentWorkflow() async throws {
        let (memory, _) = try await TestHelpers.makeMemory(
            structured: true,
            extractor: MockExtractor()
        )

        // 1. Agent remembers content (auto-extraction)
        try await memory.remember(
            "Chris prefers guard-clause error handling in Swift",
            metadata: ["project": "wax"],
            extractEntities: true
        )
        try await memory.flush()

        // 2. Agent recalls with graph fusion
        let context = try await memory.recall(
            query: "Chris error handling",
            project: "wax",
            graphEnabled: true
        )
        #expect(!context.items.isEmpty)

        // 3. Agent gets entity context
        let card = try await memory.entityContext(for: "Chris")
        #expect(card != nil)

        // 4. Agent reflects on knowledge
        let reflection = try await memory.reflect(project: "wax")
        #expect(reflection.totalEntities >= 1)

        // 5. Agent corrects wrong knowledge
        _ = try await memory.forget(content: "Chris prefers tabs")

        // 6. Agent hands off
        try await memory.handoff(
            content: "Completed error handling analysis",
            project: "wax",
            pendingTasks: ["Review guard clause patterns"]
        )
    }

    @Test func recallWithoutGraphMatchesExistingBehavior() async throws {
        // Regression: verify recall(graphEnabled: false) returns same
        // results as the old recall() method
        let (memory, _) = try await TestHelpers.makeMemory(structured: true)

        try await memory.remember("Swift concurrency uses actors for isolation")
        try await memory.flush()

        let oldResult = try await memory.recall(query: "Swift concurrency")
        let newResult = try await memory.recall(
            query: "Swift concurrency",
            graphEnabled: false
        )

        #expect(oldResult.items.count == newResult.items.count)
        #expect(oldResult.items.map(\.frameId) == newResult.items.map(\.frameId))
    }
}
```

**Step 2: Run tests**

Run: `swift test --filter RelationalMemoryE2ETests`
Expected: PASS

**Step 3: Run full test suite**

Run: `swift test`
Expected: ALL 729+ tests pass

**Step 4: Commit**

```bash
git commit -m "test: add end-to-end integration tests for relational memory pipeline"
```

---

## Phase 7: Documentation

### Task 15: Update CLAUDE.md Memory Engine Section

Update the Memory Engine section in CLAUDE.md with the new tools and workflow.

**Step 1: Update CLI commands table**

Add `forget`, `context`, `reflect` commands with their arguments.

**Step 2: Update MCP tool list**

Add new tools to the MCP server documentation.

**Step 3: Update session lifecycle**

Describe how auto-extraction and graph-fused recall work in the agent workflow.

**Step 4: Commit**

```bash
git commit -m "docs: update CLAUDE.md with relational memory tools and workflow"
```

---

## Summary

| Phase | Tasks | What it delivers |
|-------|-------|-----------------|
| 1: Core Infrastructure | 1-4 | EntityExtractor protocol, GraphWalker (2-hop), ContradictionDetector, Foundation Models extractor |
| 2: Graph-Fused Recall | 5-6 | 3-source fusion in recall pipeline, auto-extraction on remember |
| 3: New Agent Tools | 7-9 | wax_forget, wax_context, wax_reflect (CLI + MCP) |
| 4: Update Existing Tools | 10-12 | Project scoping + graph params on remember/recall, better tool descriptions |
| 5: Registration | 13 | Wire everything together in MCP server + CLI |
| 6: Integration Tests | 14 | End-to-end agent workflow tests, regression tests |
| 7: Documentation | 15 | Updated CLAUDE.md |

**Total: 15 tasks, TDD throughout, frequent commits.**
