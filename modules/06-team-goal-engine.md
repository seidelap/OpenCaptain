# Module 6: Team Goal Engine

## 1. Purpose

A **decision engine** for its assigned team. Reasons over a recursive tree of `Goal` entities — initiatives, epics, tickets, subtasks, "get access" requests, "clarify this disagreement" tasks — and continuously asks one question: *which action would most advance this team's goals right now?* The engine never communicates directly. Every action it concludes is worth taking is submitted to the Policy Engine via `submitAction()`, which alone decides whether and how to deliver it.

The engine treats detection of conflicts, coverage gaps, and stale dependencies as *cheap and exhaustive* — every relevant `relates_to` edge, every uncovered Goal, every blocked dependency is a candidate. What's selective is the *acting*: candidates are scored by expected goal advancement, and only the highest-scoring proposals are submitted. Most candidates score near zero and the engine stays silent. Silence is a deliberate decision, not a missed signal — and is itself recorded for audit.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | Inbound events via EventBus (`TicketUpdated`, `TicketCreated`, `MessageReceived` in team channels, `DecisionRecorded`) | Drives tree maintenance and triggers re-scoring. Ticket lifecycle, team-channel messages, and explicit decision events are the engine's external signal sources. |
| **Identity & Permission Engine (Module 2)** | `resolveIdentity()`, `resolveGroup()` | Resolves Goal owners/assignees and verifies team membership (via `resolveGroup(team_dl).members`) so the goal tree is scoped to this team's accountability. |
| **Policy & Governance Engine (Module 3)** | `submitAction()` | Single egress point. Every proposed action — surface a clarification, request access, suggest a sync, raise a coverage gap — is submitted for evaluation. The engine never dispatches directly. |
| **Knowledge Store (Module 4)** | `queryClaimsForEntity()`, `queryClaimsForTopic()`, `getRelatedClaims()`, `writeClaim()`, `writeArtifact()`, `generateContextPack()`, `subscribeInvalidations()` | Reads the goal tree and surrounding Claims; writes engine-derived Claims (progress, coverage, alignment, relevance assessments); writes `DETECTION_RECORD` artifacts to ground those Claims; subscribes to invalidation signals to recompute scores reactively. |
| **Audit & Observability Layer (Module 7)** | `logEvent()` | Pushes engine telemetry: candidates enumerated, scores written, actions proposed, decisions to stay silent. Silence is logged so audit can verify the engine considered a signal and decided not to act. |
| **Configuration & Admin Interface (Module 8)** | `getGoalEngineConfig()` | Team scope (which team DL this instance serves), relevance thresholds, heuristic weights, LLM tier-up threshold, max concurrent open OutreachRequests per recipient. |

### 2.2 Modules That Depend On This Module

*None.* The Goal Engine is a leaf consumer. Its outputs reach the rest of the system through (a) Claims and `DETECTION_RECORD` artifacts written to the Knowledge Store, and (b) `submitAction()` calls to the Policy Engine. No module calls into it directly.

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

*None.* The engine is a leaf consumer. All outputs flow through the Knowledge Store (Claims, DETECTION_RECORD artifacts) and the Policy Engine (`submitAction()`).

### 3.2 Required Interfaces (This Module Consumes)

#### From Platform Integration Layer (Module 1):

```
subscribe(event_types: [TicketUpdated, TicketCreated, MessageReceived,
                        DecisionRecorded], callback) -> SubscriptionHandle
```

#### From Identity & Permission Engine (Module 2):

```
resolveIdentity(platform: Platform, platform_user_id: string) -> UnifiedIdentity?
resolveGroup(group_ref: UnifiedGroupRef) -> ResolvedGroup
```

#### From Policy & Governance Engine (Module 3):

```
submitAction(proposed_action: ProposedAction, source_module: GOAL_ENGINE) -> ActionOutcome
```

#### From Knowledge Store (Module 4):

```
queryClaimsForEntity(entity_id: EntityId, scope: ClaimScope?) -> ClaimSet
queryClaimsForTopic(topic: string, scope: ClaimScope?, as_of: timestamp?) -> ClaimSet
getRelatedClaims(entity_id: EntityId?, topic: string?, scope: ClaimScope?) -> RelatedPair[]
writeClaim(claim: LayerBClaim) -> ClaimId
writeArtifact(artifact: LayerAArtifact) -> ArtifactId
  — Writes DETECTION_RECORD artifacts that ground engine-derived Claims
generateContextPack(query: ContextQuery) -> LayerCArtifact
subscribeInvalidations(filter: InvalidationFilter, callback) -> SubscriptionHandle
  — Reactive recomputation: when Claims/Entities the engine has scored against
    change, the engine receives an invalidation event and re-scores the
    affected candidates only.
```

#### From Audit & Observability Layer (Module 7):

```
logEvent(entry: AuditLogEntry) -> AuditLogId
```

#### From Configuration & Admin Interface (Module 8):

```
getGoalEngineConfig() -> GoalEngineConfig
  — Returns: team_dl, relevance_threshold, heuristic_weights,
    llm_tier_up_threshold, llm_model, max_open_outreach_per_recipient,
    cascade_resolution_enabled
```

---

## 4. Architecture: Three Stages

The engine is organized as three stages operating on the same goal tree. Each stage has a single responsibility; together they form one continuous action-scoring loop.

```
┌──────────────────────────────────────────────────────────────────┐
│                       Team Goal Engine                            │
│                                                                   │
│   Stage 1: Tree Maintenance                                       │
│   - Ingest events (tickets, messages, decisions)                  │
│   - Ensure goal tree reflects current state                       │
│   - Cascade resolution down; propagate progress up                │
│              │                                                    │
│              ▼                                                    │
│   Stage 2: Candidate Scoring                                      │
│   - Enumerate candidate actions from the tree                     │
│   - Score each with a heuristic pass (cheap, exhaustive)          │
│   - Tier up to LLM scoring on semantic close-calls                │
│   - Write goal_relevance Claim + DETECTION_RECORD per candidate   │
│              │                                                    │
│              ▼                                                    │
│   Stage 3: Action Proposal                                        │
│   - Filter candidates above threshold                             │
│   - Submit top-N to Policy Engine via submitAction()              │
│   - Log submitted and silent decisions to Audit Layer             │
└──────────────────────────────────────────────────────────────────┘
```

The loop is **reactive, not periodic**. Stage 1 runs on inbound platform events. Stage 2 runs on Knowledge Store invalidation events for the affected subtree only. Stage 3 runs whenever Stage 2 produces a relevance score crossing the threshold.

---

## 5. Stage 1: Tree Maintenance

### 5.1 `handleEvent(event: OpenCaptainEvent) -> void`

**Inputs:**
- `event` — `TicketUpdated`, `TicketCreated`, `MessageReceived`, or `DecisionRecorded`

**What it does:**
Routes inbound events that may affect the goal tree. The engine does **not** extract Goals or Claims from raw events itself — that's Module 4's extraction pipeline. This handler responds to the *consequences* of extraction:

- `TicketCreated` / `TicketUpdated`: queries the Knowledge Store for Goals with `tracked_in` referencing this ticket. If found, reads the latest Claims (status, assignee, parent epic) and ensures structural edges (`decomposes_into`, `depends_on`) match the platform's parent/child and link relationships. If no matching Goal exists, waits for the extraction pipeline to create one — does not pre-empt extraction.
- `DecisionRecorded` (a `DECISION` artifact written to Layer A): queries for Goals affected by the decision; if the decision marks a Goal explicitly resolved or obsoleted, invokes `cascadeResolution()`.
- `MessageReceived` in a team channel: usually triggers nothing directly. Extraction may produce new Claims, which arrive as invalidation events handled by Stage 2.

### 5.2 `propagateProgress(goal_id: GoalId) -> void`

**Inputs:**
- `goal_id` — a Goal whose state may have changed

**What it does:**
Walks `decomposes_into` edges *upward* from the changed Goal. For each ancestor, computes a `progress_assessment` value from the children's current `status` and `progress_assessment` Claims. Default behavior:

- All required children done → ancestor is `done`
- Any child blocked or `at_risk` → ancestor is `at_risk`
- Mixed in-progress → ancestor is `in_progress`
- If the ancestor has a `completion_logic` Claim (free-form authored expression like "A OR (B AND C)"), uses it as guidance; otherwise uses the conservative ALL_OF default

Writes the new `progress_assessment` as a Claim version on each affected ancestor. The Claim's `grounded_in` points to a `DETECTION_RECORD` artifact capturing the children's states at evaluation time.

### 5.3 `cascadeResolution(goal_id: GoalId, reason: ResolutionReason) -> void`

**Inputs:**
- `goal_id` — a Goal becoming resolved
- `reason` — completed | abandoned | obsoleted | merged

**What it does:**
Enforces the active-view invariant: when a Goal is resolved, its still-active descendants must also be resolved. Walks `decomposes_into` edges *downward*. For each active descendant:

- Writes a `status: resolved` Claim with `nature: obsoleted_by_parent`
- The Claim's `grounded_in` points to the parent's resolution `DETECTION_RECORD`, creating an auditable cascade chain

Skips descendants already resolved. Does not cascade across `depends_on` edges — only `decomposes_into`. A dependency relationship doesn't imply ownership; resolving a Goal that another Goal depends on may simply unblock the dependent, not obsolete it.

### 5.4 `mergeGoals(canonical: GoalId, duplicate: GoalId, evidence: ArtifactId) -> void`

**Inputs:**
- `canonical` — the Goal to keep
- `duplicate` — the Goal to merge in
- `evidence` — Layer A artifact justifying the merge (e.g., human confirmation, entity-resolver output)

**What it does:**
Used when two Goals — typically one extracted from a meeting and one authored via the admin interface — turn out to refer to the same outcome. Writes a `same_as` edge from `duplicate` to `canonical`. Re-roots the `duplicate`'s incoming/outgoing structural edges (`decomposes_into`, `depends_on`) to `canonical`. The duplicate's Claims are not deleted; they accumulate as additional `grounded_in` evidence on the canonical Goal's Claims (per the answer to design question §3 above).

---

## 6. Stage 2: Candidate Scoring

### 6.1 `enumerateCandidates(scope: TeamScope) -> CandidateAction[]`

**Inputs:**
- `scope` — the team scope this engine instance serves

**What it does:**
Walks the active subtree of Goals owned by or affecting this team and produces a candidate action for every potential intervention. Candidates come from five sources, each tied to a graph pattern:

| Candidate kind | Graph pattern that generates it | Example proposed action |
|---|---|---|
| `clarify_conflict` | `relates_to` edge between two Claims grounded in different authors, with mutual-acknowledgement gap | Surface both understandings in the team channel |
| `coverage_gap` | Goal with no `decomposes_into` children and no `assigned_to` Claim | Suggest the team decide who owns it or how to break it down |
| `stale_dependency` | Goal with `depends_on` an unresolved Goal whose `progress_assessment` has been `at_risk` past a threshold age | Surface the dependency to the dependent Goal's owner |
| `cross_team_unblock` | Goal with `depends_on` a Goal whose `owner` is outside this team's scope | Propose a request-access or coordination message; if not currently authorized, the proposal is itself the request-access action |
| `commitment_tension` | Same Identity has `assigned_to` Claims on two Goals whose deadlines or required time conflict | Surface privately to that Identity |

Each candidate carries: the affected Goal IDs (with their full ancestor chain, used in scoring), the supporting Claim/edge IDs (the *evidence*), and a kind-specific draft of the action that would be proposed. Enumeration is exhaustive and cheap — no LLM calls — and produces no Knowledge Store writes by itself.

### 6.2 `scoreHeuristic(candidate: CandidateAction) -> HeuristicScore`

**Inputs:**
- `candidate` — a candidate action from `enumerateCandidates()`

**What it does:**
Computes a deterministic score in `[0, 1]` from a weighted sum of features. Weights are sourced from `getGoalEngineConfig().heuristic_weights` and documented so the engine is replayable. Features:

| Feature | Signal | Direction |
|---|---|---|
| `execution_proximity` | Graph distance from the candidate's affected Goals to the nearest leaf Goal with `assigned_to` and active `status` | Closer → higher |
| `ancestor_priority` | Highest-priority Claim found on any ancestor Goal (priority is a Claim, optional) | Higher priority → higher |
| `evidence_age` | Max age of supporting Claims | Older stale items → higher (for `stale_dependency`); fresh conflicts → higher (for `clarify_conflict`) |
| `acknowledgement_gap` | Boolean: do the conflicting Claims' authors lack `grounded_in` edges to each other? | Gap → higher; mutual ack → ~0 |
| `recipient_load` | Count of open `OutreachRequest` nodes targeting the same Identity in a recent window | Higher load → lower |
| `recent_resolution_signal` | Has a related `DECISION` artifact appeared since the conflict? | Yes → ~0 (likely already resolved out-of-band) |

The output `HeuristicScore` carries the score, the per-feature contributions, and a confidence value. Confidence drops when features disagree (e.g., high execution_proximity but high recipient_load) — this is the signal that Stage 2 should tier up to LLM scoring.

### 6.3 `scoreLLM(candidate: CandidateAction, heuristic: HeuristicScore) -> LLMScore`

**Inputs:**
- `candidate` — the candidate action
- `heuristic` — the heuristic score (used as prior, not overridden blindly)

**What it does:**
Invoked only when `heuristic.confidence < llm_tier_up_threshold`. Calls the configured mini model (default: Claude Haiku) with:

- The candidate's affected Goal subtree (titles, statuses, owners — limited to ~10 nodes)
- The supporting Claims with their `grounded_in` snippets
- The draft action text
- The heuristic score and feature breakdown as prior

Asks the model to assess: (a) whether the action would semantically advance the affected Goals, (b) whether the framing of the action is appropriate to the social context, (c) any reason the heuristic over- or under-weighted a feature. Returns `LLMScore` with a numeric score in `[0, 1]`, a brief rationale (≤200 tokens), and a structured list of any feature adjustments suggested. **Uses prompt caching** on the framework instructions so repeated invocations cache the static portion (~90% of input tokens).

The LLM's score does not replace the heuristic; it is recorded alongside. The final `goal_relevance` is a documented combination (default: max of the two, with rationale linking to whichever was higher).

### 6.4 `recordScore(candidate: CandidateAction, heuristic: HeuristicScore, llm: LLMScore?) -> void`

**Inputs:**
- `candidate` — the scored candidate
- `heuristic` — heuristic result
- `llm` — LLM result, if tier-up was invoked

**What it does:**
Persists the scoring outcome durably:

1. Writes a `DETECTION_RECORD` artifact to Layer A capturing: candidate fingerprint, enumeration source, heuristic features and weights at evaluation time, LLM rationale (if any), final score, and the engine config version. This is the audit-grade record that lets a future replay reproduce or critique the decision.
2. Writes a `goal_relevance` Claim on each affected Goal, with `grounded_in` → the DETECTION_RECORD. The Claim's value is the score; its `version` increments when `recomputeOnInvalidation()` later re-scores.
3. Indexes the candidate's fingerprint so duplicate enumerations don't produce duplicate records (the same conflict scored an hour later under the same conditions yields a *new version* of the same Claim, not a parallel one).

### 6.5 `recomputeOnInvalidation(invalidation: InvalidationEvent) -> void`

**Inputs:**
- `invalidation` — emitted by Module 4 when a Claim or Entity changes

**What it does:**
The reactive entry point. Looks up which `goal_relevance` Claims depend on the changed entity (via the reverse index built in `recordScore()`), regenerates their candidate definitions from the current graph state, and re-runs `scoreHeuristic()` (and `scoreLLM()` if needed). New scores are written as new Claim versions. If a score crosses the relevance threshold (in either direction), Stage 3 is notified — either to propose a new action or to mark a previously-active proposal as no-longer-relevant (which Policy reads when re-evaluating an existing OutreachRequest).

---

## 7. Stage 3: Action Proposal

### 7.1 `proposeActions(scored_candidates: ScoredCandidate[]) -> void`

**Inputs:**
- `scored_candidates` — candidates whose latest `goal_relevance` Claim crosses the threshold

**What it does:**
The single egress point to the Policy Engine. For each candidate above threshold:

1. Composes the `ProposedAction` from the candidate's draft, attaching epistemic labels and the affected Goal context
2. Attaches the `goal_relevance` score and the DETECTION_RECORD ID to the action's metadata, so the Policy Engine's `assembleContext()` can read them when applying dedup and approval rules
3. Calls `submitAction(proposed_action, source_module: GOAL_ENGINE)`
4. Logs the submission to the Audit Layer with the score and rationale

The engine does not track the outcome of `submitAction()` for re-proposal — that's Policy's job (via the `OutreachRequest` graph). If Policy denies, defers, or queues for approval, the engine simply moves on. If the underlying state later changes such that the candidate's score increases, the next invalidation cycle will reproduce a fresh proposal; Policy will then evaluate it against the existing OutreachRequest.

### 7.2 `logSilence(candidate: CandidateAction, score: float) -> void`

**Inputs:**
- `candidate` — a candidate that scored below threshold
- `score` — the relevance score that produced silence

**What it does:**
Writes an audit log entry recording that this candidate was considered and the engine chose not to propose. This is non-trivial: silence is the engine's most common output, and without explicit logging the audit layer cannot distinguish "engine considered and declined" from "engine never noticed." The log entry is correlated to the DETECTION_RECORD by ID.

Silence logging is rate-limited per fingerprint — the engine doesn't log silence on every invalidation cycle for the same low-scoring candidate. Default: log on score creation, and again only when the score changes by more than a configured delta or when the candidate's fingerprint disappears from enumeration.

---

## 8. Key Design Constraints

1. **Decision-only.** The engine never communicates with humans or external systems. Every action is submitted to Policy via `submitAction()`. There is no fallback path.

2. **One Entity type for the goal hierarchy.** `Goal` is the only Entity type the engine creates or consumes for the goal tree. Variation across initiatives, epics, tickets, subtasks, blockers, access requests, and clarification needs is expressed through Claim predicates (`nature`, `origin`, `tracked_in`, `assigned_to`, etc.), not through type proliferation.

3. **Detection is exhaustive; action is selective.** Stage 2 enumerates every candidate the graph admits. Stage 3 acts on the highest-scoring few. Most candidates produce silence, and silence is logged.

4. **Active-view invariant.** Filtering the goal tree to `status: active` always yields a coherent forest. Resolving a Goal triggers `cascadeResolution()` to mark active descendants `obsoleted_by_parent`. The engine refuses to mark a Goal resolved while required active children remain, unless the resolution is explicitly cascading.

5. **Engine-derived state is versioned and grounded.** `progress_assessment`, `coverage_assessment`, `alignment_assessment`, `goal_relevance` are Claims, not flat fields. Each version's `grounded_in` points to a `DETECTION_RECORD` artifact in Layer A so the engine's reasoning is replayable.

6. **Reactive, not periodic.** Recomputation is driven by Knowledge Store invalidation events. There is no polling cycle. Wall-clock-based candidates (e.g., "deadline passed") are surfaced via wake events scheduled on Module 1, not by an internal timer.

7. **Two-stage scoring.** Heuristic first; LLM only on semantic close-calls. Both are recorded; the final `goal_relevance` is a documented combination of the two. Heuristic-only operation is supported (LLM tier-up is configurable off) for environments that cannot reach an LLM endpoint.

8. **Never adjudicates.** When Claims conflict, the engine surfaces both with epistemic labels via a proposed action. The team resolves; the engine observes the resolution via subsequent extraction and updates its scores.

9. **Team-scoped.** One engine instance per team DL. Cross-team coordination flows through Policy and the shared Knowledge Store, not engine-to-engine calls.

---

## 9. Data Model

The engine operates entirely on the Layer B `Goal` Entity and its attached Claims. There are no engine-local data structures that mirror or shadow graph state.

### Goal (Layer B Entity)

```
Goal {
  entity_id: EntityId               // stable URI
  // No inline fields. All semantics live in Claims and edges.
}
```

### Claim predicates the engine reads

Authored or extracted by other parts of the system, consumed by the engine:

| Predicate | Range | Source |
|---|---|---|
| `title` | string | Authoring or extraction |
| `description` | string | Authoring or extraction |
| `owner` | Identity \| Team | Authoring or extraction |
| `assigned_to` | Identity | Ticket sync or extraction |
| `tracked_in` | ArtifactId (TICKET) | Ticket sync |
| `status` | platform-native string \| `resolved` | Ticket sync, decision events, cascade |
| `nature` | build \| unblock \| clarify \| request_access \| decide \| research \| obsoleted_by_parent | Authoring or engine (for `obsoleted_by_parent`) |
| `origin` | authored \| extracted \| detected | Whichever subsystem created the Goal |
| `priority` | enum or number | Authoring |
| `deadline` | timestamp | Authoring or extraction |
| `completion_logic` | free-form string | Authoring (optional; default ALL_OF) |

### Claim predicates the engine writes

| Predicate | Range | When written |
|---|---|---|
| `progress_assessment` | on_track \| at_risk \| behind \| done \| unknown | `propagateProgress()` |
| `coverage_assessment` | covered \| partial \| uncovered | During candidate enumeration when a coverage gap is detected |
| `alignment_assessment` | aligned \| divergent \| unknown | During candidate enumeration on `clarify_conflict` cases |
| `goal_relevance` | float in [0, 1] | `recordScore()`, re-versioned on invalidation |

Every engine-written Claim has `grounded_in` → a `DETECTION_RECORD` artifact in Layer A.

### Edges the engine relies on

| Edge | From → To | Source |
|---|---|---|
| `decomposes_into` | Goal → Goal | Ticket parent/child relationships, extraction, authoring |
| `depends_on` | Goal → Goal | Ticket links, extraction, authoring |
| `same_as` | Goal → Goal | `mergeGoals()` |
| `relates_to` | Claim ↔ Claim | Module 4 extraction pipeline |
| `grounded_in` | Claim → Artifact (with position) | All Claim writes |

### CandidateAction (transient, in-process)

```
CandidateAction {
  fingerprint: string                        // stable hash of (kind, affected_goal_ids, evidence_ids)
  kind: clarify_conflict | coverage_gap | stale_dependency |
        cross_team_unblock | commitment_tension
  affected_goals: GoalId[]
  evidence: (ClaimId | EdgeId | ArtifactId)[]
  draft_action: ProposedActionDraft          // kind-specific template
}
```

Not persisted as a graph node. The `fingerprint` is what gets persisted — on the `goal_relevance` Claim and on the `DETECTION_RECORD` artifact — so identical candidates re-derived later collapse onto the same Claim history.

### DETECTION_RECORD (Layer A artifact)

A new `artifact_type` introduced for this module. Required for grounding engine-derived Claims. See Module 4 §5 for the schema addition.

---

## 10. Worked Example: "Get access to API X"

To make the recursion and decision flow concrete:

**Initial state.** Team A has authored Goal G1 "Ship feature Y" with `priority: high`. The extraction pipeline, processing a recent design meeting, creates Goal G2 "Use API X" with `decomposes_into` G1, and infers a dependency on Team Z's API.

**Engine reaction.** A Knowledge Store invalidation arrives (G2 was created). Stage 2 enumerates candidates affecting G2. One candidate emerges: G2 depends on a resource owned outside Team A (`cross_team_unblock`). The engine creates a draft Goal G3 "Get access to API X" with `nature: request_access`, `origin: detected`, `decomposes_into G2`. (Note: the *Goal* is created by the engine — this is one of the few places Stage 1 originates structure rather than reacting to it. It's done because the access request is itself a unit of work the team must track.)

**Scoring.** Stage 2 enumerates the `cross_team_unblock` candidate again, now keyed to G3.
- Heuristic features: high `execution_proximity` (G2 is active and assigned), high `ancestor_priority` (G1 is high-priority), zero `recipient_load` (no open OutreachRequest to Team Z's admin), no `recent_resolution_signal`.
- Heuristic score: 0.84, confidence high. No LLM tier-up.

**Proposal.** Score crosses threshold. Stage 3 composes a `SendMessage` action drafted as "Team A is working on G1 and needs access to API X. Could we discuss the right next step?" — addressed to Team Z's admin DL. Submits to Policy.

**Policy reaction.** The Policy Engine sees a cross-team comm (out of whitelist) and verdicts `REQUIRE_APPROVAL`. It creates an OutreachRequest node `regarding G3`, `targets` Team A's admin DL, and sends the approval request. The engine moves on.

**Approval.** Team A's admin approves. Policy updates the OutreachRequest, dispatches the original message to Team Z, writes an OUTREACH_SENT artifact. Team Z replies a few hours later with provisioned credentials.

**Resolution.** Extraction processes Team Z's reply, creates a Claim "access granted to API X" grounded in the message. An invalidation fires. The engine's `recomputeOnInvalidation()` re-scores the `cross_team_unblock` candidate; the new state shows the dependency satisfied. Score drops to ~0. The engine's resolution detector sees the access-granted Claim and writes `status: resolved` on G3 with `nature: completed`. `propagateProgress()` walks up: G2's `progress_assessment` becomes `in_progress` (was blocked); G1 is unaffected.

**Same machinery, same Goal type, same loop** — used for the OKR at the top of the tree and the "get access" task at the bottom.

---

## 11. Test Plan

### Unit Tests

| Function | Test | Expected |
|---|---|---|
| `handleEvent()` | `TicketUpdated` for tracked Goal with new parent epic | `decomposes_into` edge written; no Claim duplication |
| `handleEvent()` | `TicketUpdated` for ticket with no matching Goal | Event noted; no Goal created (waits for extraction) |
| `propagateProgress()` | Leaf Goal marked done; ancestor has all-children-done | Ancestor's `progress_assessment` becomes `done` |
| `propagateProgress()` | Sibling becomes `at_risk` | Ancestor's `progress_assessment` becomes `at_risk`; new Claim version |
| `cascadeResolution()` | Goal resolved with two active descendants | Both descendants get `status: resolved`, `nature: obsoleted_by_parent`, `grounded_in` chain to parent's record |
| `cascadeResolution()` | Cascade does not traverse `depends_on` | Dependent Goal not auto-resolved; only structural descendants |
| `mergeGoals()` | Authored Goal merged into extraction-created Goal | `same_as` edge written; structural edges re-rooted; Claims preserved as additional grounding |
| `enumerateCandidates()` | Two Claims about same Goal grounded in different authors, no mutual ack | One `clarify_conflict` candidate with both Claims as evidence |
| `enumerateCandidates()` | Goal with no children, no `assigned_to` | One `coverage_gap` candidate |
| `enumerateCandidates()` | Goal with mutual `grounded_in` ack between conflicting Claims | No `clarify_conflict` candidate (gap closed) |
| `scoreHeuristic()` | Conflict on a Goal far from active execution | Low score, primarily from `execution_proximity` |
| `scoreHeuristic()` | Conflict on an active, high-priority leaf | High score; high confidence; no LLM tier-up |
| `scoreHeuristic()` | High-stakes feature mix with conflicting signals | Lower confidence; flagged for LLM tier-up |
| `scoreLLM()` | Tier-up invoked | Returns score + rationale; cached prompt portion reflected in token accounting |
| `recordScore()` | First scoring of a candidate | New `goal_relevance` Claim version 1; new `DETECTION_RECORD` artifact; reverse-index entry created |
| `recordScore()` | Re-score after invalidation | Same Claim, version 2; new `DETECTION_RECORD`; reverse-index updated |
| `recomputeOnInvalidation()` | Underlying Claim retracted | Score drops to ~0; if previously above threshold, Stage 3 notified to mark stale |
| `proposeActions()` | Score above threshold | `submitAction()` called; metadata includes score and DETECTION_RECORD ID |
| `logSilence()` | Candidate scored below threshold | Audit entry written; subsequent below-threshold scores within delta do not duplicate the entry |

### Integration Tests

| Test | Approach |
|---|---|
| End-to-end: cross-team unblock | Seed Goal tree with cross-team dependency; advance through enumeration → scoring → proposal → Policy approval → OUTREACH_SENT → reply extraction → resolution. Verify cascade and active-view consistency at each step. |
| Active-view invariant under cascade | Resolve a top-level Goal with active descendants; query `status: active`; verify no descendants of the resolved Goal appear |
| Authored vs. extracted merge | Author a Goal; have extraction independently create a same-subject Goal; trigger merge; verify single canonical Goal with combined `grounded_in` chain |
| Reactive recomputation | Score a candidate; modify an underlying Claim; verify only affected scores recomputed (not the whole tree) |
| Heuristic-only mode | Disable LLM tier-up; verify all candidates scored; verify confidence-flagged candidates score on heuristic alone |
| Worked example (§10) reproducibility | Replay the §10 scenario from a saved A-snapshot; verify scoring and decisions match |

### Property Tests

| Property | Enforcement |
|---|---|
| Every engine-written Claim has `grounded_in` to a `DETECTION_RECORD` | Schema check on writes |
| Active-view tree is connected | After any sequence of mutations, filtering by `status: active` yields a forest with no orphaned active nodes |
| Score versions are monotonic by timestamp | `goal_relevance` Claim versions never re-order in time |
| Identical candidates produce identical fingerprints | Property-based test on enumeration |

### Load Tests

| Test | Acceptance Criterion |
|---|---|
| Enumeration on a large team tree | 500 Goals, 200 active; enumeration completes in <500ms p95 |
| Reactive recomputation under invalidation storm | 100 invalidations/sec; engine keeps up without backlog growth |
| LLM tier-up cost ceiling | At configured threshold, ≤20% of candidates tier up; daily cost stays under configured cap |

