# Module 4: Knowledge Store (A/B/C)

## 1. Purpose

The agent's memory system, implementing the three-layer A/B/C model. **Layer A** is the immutable evidence/provenance layer (raw artifacts, governance records). **Layer B** is the semantic claims graph (entities, relationships, contradictions, supersessions). **Layer C** is the cached derived-views layer (meeting prep, context packs, query results). The Knowledge Store is the single source of truth for everything the agent knows, believes, and has been told — and it maintains full provenance and versioning so that any conclusion can be traced back to its sources.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | Inbound `OpenCaptainEvent` stream (via EventBus subscription) | All platform events (messages, meeting transcripts, ticket updates) are ingested into Layer A as raw source artifacts |
| **Configuration & Admin Interface (Module 8)** | `getKnowledgeConfig()` | Extraction pipeline parameters, Layer C TTLs, invalidation sensitivity, graph partitioning settings, and retention policies |

### 2.2 Modules That Depend On This Module

| Consumer | What It Uses | Why |
|---|---|---|
| **Identity & Permission Engine (Module 2)** | `writeArtifact()` (Layer A) | Writes ACL snapshots and identity resolution records as governance artifacts |
| **Policy & Governance Engine (Module 3)** | `writeArtifact()` (Layer A), `queryAuthorizations()` | Writes AuthorizationRecords to Layer A; queries active authorizations from Layer A for the Authorization Store rebuild |
| **Meeting Participation Engine (Module 5)** | `ingestTranscript()`, `queryClaimsForTopic()`, `getContradictions()`, `generateContextPack()` | Meeting engine ingests real-time transcripts into Layer A, queries Layer B for relevant claims/contradictions, and requests Layer C context packs for contribution composition |
| **Team Goal Engine (Module 6)** | `queryClaimsForEntity()`, `queryClaimsForTopic()`, `getContradictions()`, `writeClaim()`, `generateContextPack()` | Goal engine reads team-scoped claims (tasks, dependencies, commitments) from Layer B, detects requirement disagreements via contradiction queries, writes state updates, and generates goal briefs |
| **Audit & Observability Layer (Module 7)** | `queryArtifacts()`, `getProvenance()`, `getContradictions()` | Audit layer queries Layer A for action records, governance artifacts, and provenance chains for compliance reporting; queries contradictions for the CONTRADICTION_STATUS compliance report |

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

#### Layer A — Source/Provenance/Governance Operations

```
writeArtifact(artifact: LayerAArtifact) -> ArtifactId
  Inputs:
    artifact            — raw source data (transcript, message, ticket, approval, ACL snapshot, etc.)
                          with required metadata: source_platform, author_id, timestamp, artifact_type
  Returns:
    ArtifactId (stable, immutable reference)

  Behavior: Append-only. If an artifact with the same external reference exists, a new version is
  created — the original is never overwritten. W3C PROV provenance edges are auto-generated.

getArtifact(artifact_id: ArtifactId, version: VersionRef?) -> LayerAArtifact
  Inputs:
    artifact_id         — stable artifact reference
    version             — optional; defaults to latest. Can specify exact version or "as_of" timestamp.
  Returns:
    The artifact at the requested version

queryArtifacts(query: ArtifactQuery) -> LayerAArtifact[]
  Inputs:
    query               — filter by: artifact_type, source_platform, author_id, time_range,
                          tags, provenance_chain
  Returns:
    Matching artifacts, ordered by timestamp

getProvenance(artifact_id: ArtifactId) -> ProvenanceGraph
  Inputs:
    artifact_id         — starting artifact
  Returns:
    ProvenanceGraph with W3C PROV edges: wasGeneratedBy, wasDerivedFrom,
    wasAttributedTo, actedOnBehalfOf
```

#### Layer B — Semantic Claim Graph Operations

```
writeClaim(claim: LayerBClaim) -> ClaimId
  Inputs:
    claim               — semantic claim with: subject, predicate, object, grounded_in (ArtifactId[]),
                          epistemic_status, valid_from, scope
  Returns:
    ClaimId (stable URI)

  Behavior: On insertion, automatically runs contradiction detection against existing claims in
  the same scope/topic. If conflicts found, creates "contradicts" edges — never auto-promotes
  to "supersedes" without governance evidence.

writeRelationship(source: ClaimId, target: ClaimId, rel_type: RelationshipType, evidence: ArtifactId?) -> void
  Inputs:
    source, target      — the two claims being related
    rel_type            — supports | contradicts | supersedes | clarifies | narrows |
                          depends_on | same_as | approved_by | rejected_by
    evidence            — optional Layer A artifact that justifies this relationship
  
  Behavior: "supersedes" requires a non-null evidence parameter pointing to a governance artifact
  (decision record, ADR, explicit approval) in Layer A. Without evidence, the write is rejected.

queryClaimsForTopic(topic: string, scope: ClaimScope?, as_of: timestamp?) -> ClaimSet
  Inputs:
    topic               — topic keyword, entity reference, or semantic query
    scope               — optional team/project scope filter
    as_of               — optional point-in-time for temporal queries
  Returns:
    ClaimSet { claims: LayerBClaim[], relationships: Relationship[], conflicts: ConflictPair[] }

queryClaimsForEntity(entity_id: EntityId, scope: ClaimScope?) -> ClaimSet
  Inputs:
    entity_id           — a specific entity (person, task, goal, etc.) in the B graph
    scope               — optional scope filter
  Returns:
    All claims involving this entity, with relationships and conflicts

getContradictions(scope: ClaimScope?, topic: string?) -> ConflictPair[]
  Inputs:
    scope               — optional filter to team/project scope
    topic               — optional topic filter
  Returns:
    Pairs of claims connected by "contradicts" edges that have not been resolved
    (no "supersedes" edge exists for either)

queryAuthorizations(scope: AuthorizationScope) -> AuthorizationRecord[]
  Inputs:
    scope               — filter by action scope, target scope, expiry status
  Returns:
    Active AuthorizationRecords stored as governance artifacts in Layer A,
    indexed for fast query by the Authorization Store
```

#### Layer C — Thoughts Cache Operations

```
generateContextPack(query: ContextQuery) -> LayerCArtifact
  Inputs:
    query               — { topic, audience, purpose, max_tokens, include_conflicts: bool }
  Returns:
    LayerCArtifact {
      c_id, created_at, query, a_snapshot_id, b_snapshot_id,
      retrieval_manifest: RetrievalManifest,
      reasoning_policy: string,
      conflicts_detected: ConflictPair[],
      missing_sources: string[],
      output: string,
      confidence: float,
      invalidated_by: ArtifactId?
    }

  Behavior: Queries Layer A and B for relevant content. Records exactly which sources were
  retrieved, which were eligible but not retrieved, and any truncation in the retrieval manifest.
  Applies epistemic labeling to the output.

getCachedArtifact(c_id: string) -> LayerCArtifact?
  Inputs:
    c_id                — Layer C artifact ID
  Returns:
    The cached artifact if still valid (not invalidated, not expired)

promoteToA(c_id: string, approver: UnifiedIdentity) -> ArtifactId
  Inputs:
    c_id                — Layer C artifact to promote
    approver            — human who approved the promotion
  Returns:
    New Layer A ArtifactId with full derivation lineage from the C artifact
```

#### Extraction Pipeline

```
ingestTranscript(transcript: TranscriptChunk, meeting_id: string) -> ArtifactId
  Inputs:
    transcript          — real-time transcript segment with speaker attribution
    meeting_id          — meeting reference for grouping
  Returns:
    Layer A artifact ID for the ingested transcript chunk

  Side effect: Triggers async claim extraction pipeline (A -> B)

triggerExtraction(artifact_id: ArtifactId) -> ExtractionJobId
  Inputs:
    artifact_id         — Layer A artifact to extract claims from
  Returns:
    Job ID for tracking the async extraction
  
  Behavior: LLM-based pipeline performs: claim extraction, NER, entity resolution,
  relation extraction, contradiction detection against existing B graph
```

### 3.2 Required Interfaces (This Module Consumes)

#### From Platform Integration Layer (Module 1):

```
subscribe(event_types: EventType[], callback: (event: OpenCaptainEvent) -> void) -> SubscriptionHandle
  — Subscribe to all platform events for ingestion into Layer A
```

#### From Configuration & Admin Interface (Module 8):

```
getKnowledgeConfig() -> KnowledgeConfig
  — Returns extraction pipeline parameters, C artifact TTLs, invalidation sensitivity,
    graph partitioning settings, retention policies

onConfigChanged(callback: (change: ConfigChange) -> void) -> void
  — Subscribe to config changes affecting knowledge store behavior
```

---

## 4. Core Functions

### 4.1 `ingestEvent(event: OpenCaptainEvent) -> ArtifactId`

**Inputs:**
- `event` — any normalized platform event from the EventBus

**What it does:**
Creates a Layer A artifact from the event with full metadata: source platform, author, timestamp, artifact type, raw content. Assigns a stable artifact ID and version number (v1 for new artifacts). Generates W3C PROV provenance edges: `wasAttributedTo` (author), `wasGeneratedBy` (platform event). Indexes the artifact by type, time, author, and platform for query. Triggers the async extraction pipeline to create Layer B claims from the artifact content. Returns the immutable artifact ID.

### 4.2 `extractClaims(artifact_id: ArtifactId) -> ClaimExtractionResult`

**Inputs:**
- `artifact_id` — the Layer A artifact to extract claims from

**What it does:**
Retrieves the artifact content from Layer A. Runs LLM-based extraction pipeline:
1. **Claim extraction:** Identifies factual claims, commitments, decisions, deadlines, ownership statements, requirements, and opinions
2. **Named entity recognition:** Identifies people, teams, projects, tasks, dates, and deliverables
3. **Entity resolution:** Maps extracted entities to existing entities in the B graph (or creates new ones)
4. **Relation extraction:** Identifies relationships between entities (depends_on, owns, committed_to, blocked_by)
5. **Contradiction detection:** For each new claim, queries existing claims in the same scope/topic and checks for conflicts

For each extracted claim: creates a Layer B claim node with `grounded_in` links back to the Layer A artifact. For detected contradictions: creates `contradicts` edges between the conflicting claims — never auto-promotes to `supersedes`. If human-in-the-loop review is configured for this artifact type, flags ambiguous extractions for review.

### 4.3 `queryWithManifest(query: ContextQuery) -> LayerCArtifact`

**Inputs:**
- `query` — topic, audience, purpose, constraints

**What it does:**
Executes retrieval over Layers A and B for content matching the query. Constructs a **retrieval manifest** recording: every source retrieved (artifact IDs, claim IDs, versions), every source that was eligible but not retrieved (and why — token limit, relevance cutoff), any truncation applied, and the time/token budget consumed. Applies governance rules to resolve conflicts: if contradicting claims exist without a supersession, both are preserved and labeled. Generates the output with epistemic labels attached to every assertion. Assigns a confidence score based on source coverage and conflict density. Returns the full Layer C artifact with manifest — no C artifact can exist without a retrieval manifest.

### 4.4 `detectContradiction(new_claim: LayerBClaim) -> ConflictPair[]`

**Inputs:**
- `new_claim` — a newly extracted or inserted claim

**What it does:**
Queries the B graph for existing claims that share the same subject/topic/scope. Uses semantic similarity and logical consistency checks to identify potential conflicts. For each conflict: creates a `contradicts` edge between the new claim and the existing claim. **Critical invariant:** `contradicts` never auto-promotes to `supersedes`. Supersession requires explicit governance evidence from Layer A (a decision record, an ADR, an approved resolution). Returns the list of detected conflict pairs for upstream consumers (Goal Engine, Meeting Engine).

### 4.5 `resolveSupersession(superseding: ClaimId, superseded: ClaimId, evidence: ArtifactId) -> void`

**Inputs:**
- `superseding` — the claim that replaces the old one
- `superseded` — the claim being replaced
- `evidence` — Layer A governance artifact that justifies the supersession (decision record, approval, etc.)

**What it does:**
Validates that the evidence artifact exists in Layer A and is of a governance type (decision, ADR, approval). Creates a `supersedes` edge from the superseding claim to the superseded claim. Marks the superseded claim's `valid_until` timestamp. Triggers invalidation of any Layer C artifacts that referenced the superseded claim. Logs the supersession with full provenance for audit.

### 4.6 `invalidateCache(trigger: InvalidationTrigger) -> InvalidatedArtifactId[]`

**Inputs:**
- `trigger` — the event that triggered invalidation (Layer A artifact change, Layer B node re-version, new source in topic neighborhood)

**What it does:**
Identifies all Layer C artifacts that depend on the changed source (via retrieval manifests). Marks each affected C artifact as invalidated with a reference to the trigger. Does not delete the invalidated C artifacts (they remain for audit purposes) but marks them so consumers know to regenerate. Returns the list of invalidated C artifact IDs so consuming modules can request fresh context packs.

---

## 5. Data Model

### Layer A Artifact

```
LayerAArtifact {
  artifact_id: ArtifactId (stable UUID)
  version: number (monotonically increasing)
  artifact_type: TRANSCRIPT | MESSAGE | TICKET | APPROVAL | DECISION | ACL_SNAPSHOT | CALENDAR_EVENT
  source_platform: Platform
  author_id: UnifiedIdentity
  content: any
  timestamp: timestamp
  valid_from: timestamp
  valid_until: timestamp?                  // null if still current
  tags: string[]
  provenance: ProvenanceEdge[]             // W3C PROV vocabulary
  created_at: timestamp                    // system timestamp of ingestion
}
```

### Layer B Claim

```
LayerBClaim {
  claim_id: ClaimId (stable URI)
  version: number
  subject: EntityId
  predicate: string
  object: any
  grounded_in: ArtifactId[]               // must be non-empty
  epistemic_status: ASSERTED | COMMITTED | DECIDED | SPECULATED | RETRACTED
  valid_from: timestamp
  valid_until: timestamp?
  scope: ClaimScope                        // team, project, or global
  confidence: float?
}
```

### Layer B Relationship Types

| Type | Meaning | Requires Evidence |
|---|---|---|
| `supports` | Corroborating claim | No |
| `contradicts` | Conflicting claim — not yet resolved | No |
| `supersedes` | Replaces a previous claim | **Yes** — governance artifact in Layer A required |
| `clarifies` | Adds detail to an existing claim | No |
| `narrows` | Restricts scope of an existing claim | No |
| `depends_on` | Dependency relationship | No |
| `same_as` | Entity deduplication link | No |
| `approved_by` | Governance approval link | Yes — AuthorizationRecord |
| `rejected_by` | Governance rejection link | Yes — AuthorizationRecord |

### Layer C Artifact

```
LayerCArtifact {
  c_id: string (UUID)
  created_at: timestamp
  query: ContextQuery
  a_snapshot_id: string                    // Layer A state reference
  b_snapshot_id: string                    // Layer B state reference
  retrieval_manifest: RetrievalManifest    // REQUIRED — no C without manifest
  reasoning_policy: string
  conflicts_detected: ConflictPair[]
  missing_sources: string[]
  output: string
  confidence: float
  invalidated_by: ArtifactId?
  ttl: Duration
}
```

### Retrieval Manifest

```
RetrievalManifest {
  sources_retrieved: SourceRef[]           // artifact/claim IDs + versions used
  sources_eligible_not_retrieved: SourceRef[] // and why (token limit, relevance cutoff, etc.)
  truncation_flags: TruncationFlag[]
  time_budget_ms: number
  token_budget: number
  time_consumed_ms: number
  tokens_consumed: number
}
```
