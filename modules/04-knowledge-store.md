# Module 4: Knowledge Store (A/B/C)

## 1. Purpose

The agent's memory system, implementing the three-layer A/B/C model. **Layer A** is the immutable evidence/provenance layer (raw artifacts, governance records). **Layer B** is the semantic claims graph (versioned claims, entities, relationships, and grounded_in positions). **Layer C** is the cached derived-views layer (meeting prep, context packs, query results). The Knowledge Store is the single source of truth for everything the agent knows, believes, and has been told — and it maintains full provenance and versioning so that any conclusion can be traced back to its sources.

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
    artifact            — raw source data with required metadata: source_platform, author_id,
                          timestamp, artifact_type
                          artifact_type includes: TRANSCRIPT | MESSAGE | TICKET | APPROVAL |
                          DECISION | ACL_SNAPSHOT | CALENDAR_EVENT | MEETING_STATE |
                          OUTREACH_SENT | AUTHORIZATION_GRANTED | AUTHORIZATION_DENIED
                          MEETING_STATE artifacts are written by the Meeting Engine on every
                          state transition and queried by the Policy Engine's Context Assembler.
  Returns:
    ArtifactId (stable, immutable reference)

  Behavior: Append-only. If an artifact with the same external reference exists, a new version is
  created — the original is never overwritten. W3C PROV provenance edges are auto-generated.

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
    claim               — semantic claim with: subject, predicate, object, grounded_in (GroundedInRef[]),
                          valid_from, scope
  Returns:
    ClaimId (stable URI)

  Behavior: If a claim with this claim_id already exists, a new version is created when the
  content (object or predicate) has materially changed — otherwise a new grounded_in ref is
  added to the current version. On insertion of a new version, automatically re-runs
  relationship detection and creates fresh `relates_to` edges for the new version.

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

getRelatedClaims(entity_id: EntityId?, topic: string?, scope: ClaimScope?) -> RelatedPair[]
  Inputs:
    entity_id           — optional entity to scope the query
    topic               — optional topic filter
    scope               — optional team/project scope filter
  Returns:
    Pairs of claims connected by `relates_to` edges, including the grounded_in
    positions of each author so callers can evaluate the acknowledgement gap

queryAuthorizations(scope: AuthorizationScope) -> AuthorizationRecord[]
  Inputs:
    scope               — filter by action scope, target scope, expiry status
  Returns:
    Active AUTHORIZATION_GRANTED artifacts from Layer A, shaped as AuthorizationRecord
    for the Authorization Store cache rebuild

queryOutreachRequests(regarding: EntityId, status: OutreachStatus?) -> OutreachRequest[]
  Inputs:
    regarding           — entity (Blocker, WorkItem, Claim, PendingAction) this outreach concerns
    status              — optional filter: OPEN | RESPONDED | CANCELLED
  Returns:
    OutreachRequest nodes linked to this entity, ordered by last_sent_at desc

writeOutreachRequest(request: OutreachRequest) -> OutreachRequestId
updateOutreachRequest(id: OutreachRequestId, patch: OutreachRequestPatch) -> void
  — Create and update OutreachRequest nodes. Policy Engine is the only caller.
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

```

#### Extraction Pipeline

```
ingestTranscript(transcript: TranscriptChunk, meeting_id: string) -> ArtifactId
  Inputs:
    transcript          — real-time transcript segment with speaker attribution
    meeting_id          — meeting reference for grouping
  Returns:
    Layer A artifact ID for the ingested transcript chunk

  Side effect: Triggers async claim extraction pipeline (A -> B) internally.
  External callers do not need to trigger extraction separately.
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

onConfigChanged(callback: (change: ConfigChange) -> void) -> SubscriptionHandle
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
5. **Relationship detection:** For each new claim, queries existing claims in the same scope/topic and creates `relates_to` edges for related pairs

For each extracted claim: creates a Layer B claim node with `grounded_in` links back to the Layer A artifact. If human-in-the-loop review is configured for this artifact type, flags ambiguous extractions for review.

### 4.3 `queryWithManifest(query: ContextQuery) -> LayerCArtifact`

**Inputs:**
- `query` — topic, audience, purpose, constraints

**What it does:**
Executes retrieval over Layers A and B for content matching the query. Constructs a **retrieval manifest** recording: every source retrieved (artifact IDs, claim IDs, versions), every source that was eligible but not retrieved (and why — token limit, relevance cutoff), any truncation applied, and the time/token budget consumed. Preserves all related claims; if `relates_to` pairs exist, both sides are included and labeled with their respective `grounded_in` positions. Generates the output with epistemic labels attached to every assertion. Assigns a confidence score based on source coverage and conflict density. Returns the full Layer C artifact with manifest — no C artifact can exist without a retrieval manifest.

### 4.4 `detectRelationships(new_claim: LayerBClaim) -> RelatedPair[]`

**Inputs:**
- `new_claim` — a newly extracted or inserted claim

**What it does:**
Queries the B graph for existing claims that share the same subject/topic/scope. Uses semantic similarity and logical consistency checks to identify related claims. For each detected relationship: creates a `relates_to` edge between the specific versions being compared. Called both on initial claim insertion and on new version creation — edges from prior versions are not transferred, they are re-evaluated fresh. Returns the list of related pairs for upstream consumers (Policy Engine, Goal Engine, Meeting Engine). The Policy Engine then evaluates each pair to determine whether the acknowledgement gap and context warrant outreach.

### 4.5 `invalidateCache(trigger: InvalidationTrigger) -> InvalidatedArtifactId[]`

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
  artifact_type: TRANSCRIPT | MESSAGE | TICKET | APPROVAL | DECISION | ACL_SNAPSHOT | CALENDAR_EVENT |
               MEETING_STATE | OUTREACH_SENT | AUTHORIZATION_GRANTED | AUTHORIZATION_DENIED
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
  grounded_in: GroundedInRef[]             // must be non-empty
  valid_from: timestamp
  scope: ClaimScope                        // team, project, or global
  confidence: float?
}
```

### GroundedInRef

```
GroundedInRef {
  artifact_id: ArtifactId
  position: ASSERTS | AGREES | UNCERTAIN | DISAGREES | RETRACTS
}
```

`ASSERTS` — the artifact's author is making this claim directly.
`AGREES` / `DISAGREES` / `UNCERTAIN` — the author is expressing a stance toward an existing claim.
`RETRACTS` — the author is withdrawing their prior ASSERTS or AGREES on this claim.
Multiple authors can each have a `grounded_in` ref on the same claim with different positions. The current state of a claim is fully derivable from these positions — no separate `epistemic_status` field is needed.

### Claim Versioning

`claim_id` is stable across all versions of a claim. `version` is monotonically increasing. Each version has its own `grounded_in` refs — the evidence that produced or supports that version's content specifically.

**What creates a new version vs. a new `grounded_in` ref:**

- Content changes (object or predicate materially shifts) → new version
- Stance changes (someone AGREES, DISAGREES, RETRACTS) → new `grounded_in` ref on the current version

**Who can produce a new version:** anyone whose evidence materially changes the claim content. This is a judgment call by the extraction pipeline — there is no restriction to the original claimer. If a teammate's new statement produces a refined understanding of the same claim, and the original claimer's grounded_in position on the new version is AGREES, that is a valid version transition.

**Current version:** max version number for a given `claim_id`. No explicit pointer needed.

**`relates_to` edges and versioning:** edges connect specific versions, not `claim_id`s abstractly. When a new version is created, relationship detection re-runs and produces fresh edges for that version. Old edges on prior versions are preserved as history. Policy evaluates edges on the latest version.

### OutreachRequest

```
OutreachRequest {
  outreach_id: OutreachRequestId (stable UUID)
  status: OPEN | RESPONDED | CANCELLED
  attempt_count: int
  last_sent_at: timestamp
  // edges (not inline fields):
  //   regarding  →  Entity | Claim
  //   targets    →  Identity
  //   grounded_in → OUTREACH_SENT artifact per attempt
}
```

Policy Engine is the only writer. Escalation is not a status — it is a new `OutreachRequest` with a different `targets` identity, linked by sharing the same `regarding` target.

### Layer B Relationship Types

**Claim-to-claim:**

| Type | Notes |
|---|---|
| `relates_to` | Flags the pair for Policy evaluation; most edges result in no action |

**Entity-to-entity:**

| Type | Notes |
|---|---|
| `depends_on` | Dependency relationship |
| `same_as` | Deduplication / entity merge link |

**Crosses layers (B → A), used on both `Claim` and `OutreachRequest` nodes:**

| Type | Carries | Constraint |
|---|---|---|
| `grounded_in` | `GroundedInRef` with position field | Must be non-empty on every Claim |

**OutreachRequest edges:**

| Type | To | Notes |
|---|---|---|
| `regarding` | Entity \| Claim | Determines downstream behavior when status → RESPONDED |
| `targets` | Entity (Identity) | Who the agent is waiting on |

**Dropped:** `supports`, `contradicts`, `clarifies`, `narrows`, `supersedes`, `approved_by`, `rejected_by` — collapsed into `relates_to` and `grounded_in` positions. `valid_until` on claims is also dropped from the logical model; temporal evolution is handled by timestamps and `grounded_in` artifacts. Explicit retraction uses `epistemic_status: RETRACTED`.

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

---

## 6. Test Plan

### Unit Tests

| Function | Test | Expected |
|---|---|---|
| `writeArtifact()` | Write same artifact twice (same external ref) | Second write creates version 2; version 1 unchanged; both retrievable |
| `writeArtifact()` | Write `MEETING_STATE` artifact | Artifact stored with correct `artifact_type`; queryable via `queryArtifacts(artifact_type: MEETING_STATE)` |
| `writeArtifact()` | Attempt overwrite of existing artifact | Rejected; new version created instead |
| `writeClaim()` | New claim related to existing claim in same scope | `relates_to` edge automatically created between the two claims |
| `writeClaim()` | New claim unrelated to existing claims | No `relates_to` edge; claim stored normally |
| `writeClaim()` | Claim without `grounded_in` populated | Rejected — constraint enforced |
| `detectRelationships()` | Two claims with opposing predicates for same subject | Both claims connected with `relates_to` edge |
| `generateContextPack()` | Query with topic match in Layer B | Returns `LayerCArtifact` with populated `retrieval_manifest` |
| `generateContextPack()` | Query where sources exceed token budget | Truncation flag set in manifest; eligible-but-not-retrieved sources listed |
| `invalidateCache()` | Layer A artifact updated | All Layer C artifacts whose `retrieval_manifest` references it are marked `invalidated_by` |
| `ingestTranscript()` | Transcript chunk written | Layer A artifact created with correct `artifact_type: TRANSCRIPT`; async extraction triggered |

### Property Tests

| Property | Enforcement |
|---|---|
| No `LayerCArtifact` exists without a `retrieval_manifest` | Schema validation test: attempt to create C artifact without manifest; verify rejection |
| No `LayerBClaim` exists without at least one `grounded_in` link to Layer A | Constraint enforcement: attempt to write claim with empty `grounded_in`; verify rejection |
| Every Layer A write is append-only | Attempt direct field mutation on existing artifact; verify rejected; verify version created |

### Integration Tests

| Test | Approach |
|---|---|
| Extraction pipeline: meeting transcript → Layer B claims | Feed gold-standard transcript; verify expected commitments, entities, and relations appear in Layer B |
| Relationship detection across extraction runs | Feed two transcripts with conflicting requirement statements; verify `relates_to` edge in Layer B |
| Query engine includes all related claims with positions | Insert related claims; query topic; verify both sides returned with their `grounded_in` positions |
| C promotion to A preserves full lineage | Promote a Layer C artifact; verify new Layer A artifact's provenance chain traces back to all original sources |
| MEETING_STATE artifact queryable by meeting ID | Meeting Engine writes state; Policy Engine queries `artifact_type: MEETING_STATE, meeting_id: X`; verify correct state returned |

### Load Tests

| Test | Acceptance Criterion |
|---|---|
| Layer B query at 6-month synthetic data volume | `queryClaimsForTopic()` returns within 500ms at p95 |
| Relationship detection on bulk claim import | 1,000 claims imported; all `relates_to` edges detected; no missed relationships |
| Concurrent Layer A writes from multiple modules | No data loss; all artifacts correctly versioned under concurrent write load |
