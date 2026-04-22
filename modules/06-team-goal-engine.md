# Module 6: Team Goal Engine

## 1. Purpose

The "scrum lead" decision brain for its assigned team. Tracks the team's owned work, progress toward goals, and shared understanding of what each task requires. When blockers, risks, or requirement disagreements are detected, the engine decides which actions to take to help the team make progress — then **proposes** those actions to the Policy Engine. The Goal Engine itself has **no communication channel** to the outside world; it reads from the Knowledge Store and proposes actions. Even requesting access to a currently disallowed action is itself a normal proposed action routed through the Policy Engine.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | Inbound events via EventBus (`TicketUpdated`, `TicketCreated`, `MessageReceived` in team channels) | The engine receives ticket/task lifecycle events and team channel messages to track work item progress and extract requirement understanding |
| **Identity & Permission Engine (Module 2)** | `resolveIdentity()`, `resolveGroup()` | Resolves task assignees and verifies team membership (via `resolveGroup(team_dl).members`) to scope the goal tracker to this team's work only |
| **Policy & Governance Engine (Module 3)** | `submitAction()` | Every proposed action (surface blocker, suggest sync, request access) is submitted to the Policy Engine. The Goal Engine never communicates directly. |
| **Knowledge Store (Module 4)** | `queryClaimsForEntity()`, `queryClaimsForTopic()`, `getContradictions()`, `writeClaim()`, `generateContextPack()` | Reads team-scoped claims (tasks, dependencies, commitments, requirements) from Layer B. Detects requirement disagreements via contradiction queries. Writes state updates and new claims. Requests goal briefs (Layer C) for team status summaries. |
| **Audit & Observability Layer (Module 7)** | `logEvent()` | Pushes telemetry (detected blockers, requirement disagreements, proposed actions, policy verdicts) to the immutable audit log |
| **Configuration & Admin Interface (Module 8)** | `getGoalEngineConfig()` | Team scope configuration (which team DL this engine instance serves), blocker detection sensitivity, sync suggestion thresholds, work item refresh intervals |

### 2.2 Modules That Depend On This Module

*None. Like the Meeting Engine, the Goal Engine is a true leaf consumer. It reads from other modules and pushes events outward. No module calls into it directly.*

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

*None. This module is a leaf consumer. Goal state changes (detected blockers, requirement disagreements, work item updates) are pushed to the Audit Layer via `logEvent()`. No module polls this module directly.*

### 3.2 Required Interfaces (This Module Consumes)

#### From Platform Integration Layer (Module 1):

```
subscribe(event_types: [TicketUpdated, TicketCreated, MessageReceived], callback) -> SubscriptionHandle
```

#### From Identity & Permission Engine (Module 2):

```
resolveIdentity(platform: Platform, platform_user_id: string) -> UnifiedIdentity?
resolveGroup(group_ref: UnifiedGroupRef) -> ResolvedGroup
  — Used as resolveGroup(team_dl).members to scope work tracking to this team
```

#### From Policy & Governance Engine (Module 3):

```
submitAction(proposed_action: ProposedAction, source_module: GOAL_ENGINE) -> ActionOutcome
```

#### From Knowledge Store (Module 4):

```
queryClaimsForEntity(entity_id: EntityId, scope: ClaimScope?) -> ClaimSet
queryClaimsForTopic(topic: string, scope: ClaimScope?, as_of: timestamp?) -> ClaimSet
getContradictions(scope: ClaimScope?, topic: string?) -> ConflictPair[]
writeClaim(claim: LayerBClaim) -> ClaimId
generateContextPack(query: ContextQuery) -> LayerCArtifact
```

#### From Audit & Observability Layer (Module 7):

```
logEvent(entry: AuditLogEntry) -> AuditLogId
  — Push telemetry: detected blockers, requirement disagreements, proposed actions,
    policy verdicts, work item state changes
```

#### From Configuration & Admin Interface (Module 8):

```
getGoalEngineConfig() -> GoalEngineConfig
  — Returns: team_dl, blocker_detection_sensitivity, sync_suggestion_threshold,
    work_item_refresh_interval, external_dependency_alert_enabled
```

---

## 4. Core Functions

### 4.1 `syncWorkItems(team_dl: UnifiedGroupRef) -> void`

**Inputs:**
- `team_dl` — the team this engine instance serves

**What it does:**
Retrieves the list of team members via the Identity Engine's `resolveGroup(team_dl).members`. Queries the Knowledge Store for all work item claims (tasks, epics, tickets) from Jira/Asana that are assigned to team members or tagged with the team's project. Builds an internal dependency graph: which tasks block which other tasks, which tasks share deliverables, which tasks have due dates. Scopes the goal graph strictly to team-owned items — items owned by other teams are tracked only as external dependencies, not managed. Compares current work item state against previous state to detect progress, stalls, and new items.

### 4.2 `trackRequirementAlignment(work_item_id: EntityId) -> RequirementAlignmentStatus`

**Inputs:**
- `work_item_id` — the task/ticket entity in the B graph

**What it does:**
This is the Requirement Alignment Tracker. Queries the Knowledge Store for all claims about what this work item requires — sourced from meeting transcripts, messages, ticket descriptions, comments, and any other artifact mentioning the work item. Groups claims by author to build a per-person understanding: "Person A believes this task requires X, Person B believes it requires Y." Detects divergent understandings by comparing claim content semantically. If divergent understandings are found: flags them as B-layer claims with `contradicts` edges. Does **not** resolve the disagreement — surfaces it for the team to resolve. Returns the alignment status: ALIGNED (all team members share the same understanding), DIVERGENT (conflicting requirements detected), or UNKNOWN (insufficient data).

### 4.3 `detectBlockers(team_dl: UnifiedGroupRef) -> Blocker[]`

**Inputs:**
- `team_dl` — team scope

**What it does:**
This is the Blocker & Disagreement Detector. Monitors the B graph for:
- **Conflicting requirements:** Claims from different team members about what "done" means for a task that have `contradicts` edges
- **Dependencies at risk:** Tasks whose upstream dependencies have slipped, stalled, or have conflicting status claims
- **Stale action items:** Commitments or action items extracted from meetings that have passed their stated deadline with no resolution claim
- **Commitment tensions:** Two commitments from the same person or team that conflict in scope or timing
- **External dependency risks:** Cross-team dependencies where the other team's progress claims suggest delay

For each detected blocker: assesses severity (blocking progress now vs. risk of future block) and identifies which team members are affected. Returns the list of blockers sorted by severity.

### 4.4 `decideAction(blocker: Blocker) -> ProposedAction?`

**Inputs:**
- `blocker` — a detected blocker, disagreement, or risk

**What it does:**
This is the Action Decision Engine. Given a detected issue, determines what action (if any) the agent should propose to help the team make progress. Decision logic:

| Situation | Proposed Action |
|---|---|
| Requirement disagreement between team members | Propose: surface both understandings to the team channel with epistemic labels, suggest a sync |
| Intra-team dependency at risk | Propose: alert the dependent task owner and the blocking task owner in the team channel |
| Stale action item past deadline | Propose: remind the team in the channel with the original commitment context |
| Cross-team dependency at risk | Propose: request access to contact the other team (if not currently allowed — this is itself a proposed action) |
| Commitment conflict (same person, two conflicting deadlines) | Propose: surface both commitments to the individual privately with epistemic labels |

For every proposed action: attaches epistemic labels to the content ("Two different requirements have been stated for this task...", "This dependency may be at risk..."). Never adjudicates disagreements — always surfaces both sides. Submits the proposed action to the Policy Engine via `submitAction()`.

### 4.5 `generateGoalBrief(team_dl: UnifiedGroupRef) -> LayerCArtifact`

**Inputs:**
- `team_dl` — team scope

**What it does:**
Requests a Layer C context pack from the Knowledge Store scoped to the team's goals, active work items, blockers, and risks. The context pack includes: team goal summary, work item status overview, detected blockers/disagreements, at-risk dependencies, and upcoming deadlines. The brief correctly references all relevant Layer A/B sources via the retrieval manifest. This brief can be used by the agent for meeting prep (handed to the Meeting Engine) or proposed as a team status message via the Policy Engine.

### 4.6 `handleTicketEvent(event: OpenCaptainEvent) -> void`

**Inputs:**
- `event` — a `TicketUpdated` or `TicketCreated` event

**What it does:**
Checks if the ticket belongs to this team's scope (assignee is a team member, or project matches team config). If in scope: updates the internal work item tracker and dependency graph. Checks if the update changes a dependency relationship (e.g., a blocking task was marked done). Checks if the update contradicts a previous requirement claim in the B graph. If state change detected: re-runs `detectBlockers()` to check for newly resolved or newly created blockers.

---

## 5. Key Design Constraints

1. **Decision-only module:** The Goal Engine never communicates directly with humans or other systems. All proposed actions go through the Policy Engine.
2. **Never adjudicates:** When requirement disagreements are found, the engine surfaces both sides with epistemic labels. The team resolves the disagreement. The engine records the resolution when it observes it.
3. **Requesting access is a normal action:** If the engine needs to contact another team or access a resource it currently can't, it proposes a "request access" action — which the Policy Engine evaluates. There is no special escalation path.
4. **Team-scoped:** Each Goal Engine instance serves exactly one team DL. Cross-team coordination happens via the Policy Engine's boundary rules and the Knowledge Store's shared B graph — not via direct Goal Engine-to-Goal Engine communication.
5. **Un-siloing is emergent:** The engine keeps team members aligned toward shared goals by surfacing information they might not have seen. Un-siloing within the team is a natural effect of this alignment, not a stated design objective.

---

## 6. Data Model

### WorkItem

```
WorkItem {
  entity_id: EntityId                      // stable reference in B graph
  platform: Platform                       // Jira, Asana, etc.
  platform_ref: string                     // external ticket ID
  title: string
  assignee: UnifiedIdentity?
  status: string                           // platform-native status
  dependencies: EntityId[]                 // tasks this blocks or is blocked by
  due_date: timestamp?
  requirement_claims: ClaimId[]            // B graph claims about what this requires
  alignment_status: ALIGNED | DIVERGENT | UNKNOWN
}
```

### Blocker

```
Blocker {
  blocker_id: string
  type: REQUIREMENT_DISAGREEMENT | DEPENDENCY_AT_RISK | STALE_ACTION_ITEM |
        COMMITMENT_CONFLICT | EXTERNAL_DEPENDENCY_RISK
  severity: HIGH | MEDIUM | LOW
  affected_work_items: EntityId[]
  affected_team_members: UnifiedIdentity[]
  evidence_claims: ClaimId[]               // B graph claims supporting the detection
  detected_at: timestamp
  resolved: bool
}
```

---

## 7. Test Plan

### Unit Tests

| Function | Test | Expected |
|---|---|---|
| `syncWorkItems()` | Team DL with 3 members, 5 assigned tasks | All 5 tasks loaded; dependency graph built; tasks not assigned to team members excluded |
| `syncWorkItems()` | Cross-team dependency (task blocked by another team's ticket) | External dependency recorded; tracked as dependency risk, not managed work |
| `trackRequirementAlignment()` | Two team members have identical requirement claims for a task | ALIGNED status returned; no `contradicts` edge created |
| `trackRequirementAlignment()` | Two team members have divergent requirement claims | DIVERGENT status returned; `contradicts` edge created between conflicting B claims; no auto-resolution |
| `trackRequirementAlignment()` | Only one team member has stated a requirement | UNKNOWN status returned; insufficient data to determine alignment |
| `detectBlockers()` | Task with stale action item past deadline and no resolution | STALE_ACTION_ITEM blocker detected; affected team members populated |
| `detectBlockers()` | Dependency where upstream task has conflicting status claims | DEPENDENCY_AT_RISK blocker detected; severity based on due date proximity |
| `detectBlockers()` | No issues detected | Returns empty blocker list; no actions proposed |
| `decideAction()` | REQUIREMENT_DISAGREEMENT blocker detected | `submitAction()` called with proposed message surfacing both understandings; epistemic labels attached |
| `decideAction()` | EXTERNAL_DEPENDENCY_RISK detected; no authorization to contact other team | `submitAction()` called with proposed request-access action; Policy Engine evaluates |
| `decideAction()` | COMMITMENT_CONFLICT for same person | `submitAction()` proposed for private message to individual; both commitments included |
| `generateGoalBrief()` | Team with active work items, one blocker | Layer C context pack returned; retrieval manifest references Layer A/B sources; blocker included |
| `handleTicketEvent()` | TicketUpdated for task assigned to team member | Work item state updated; dependency graph re-evaluated; `detectBlockers()` re-run |
| `handleTicketEvent()` | TicketUpdated for task not assigned to this team | Event ignored; no state change |

### Integration Tests

| Test | Approach |
|---|---|
| Blocker detection end-to-end | Insert conflicting requirement claims in B graph for a tracked task; verify Goal Engine detects REQUIREMENT_DISAGREEMENT and proposes action via Policy Engine |
| Cross-team dependency risk → access request | Configure team with external dependency; simulate upstream team delay claims; verify Goal Engine proposes request-access action for out-of-scope contact |
| Policy gate on all proposed actions | Verify every `decideAction()` output routes through `submitAction()` and is not dispatched directly |
| Goal brief references correct sources | Generate goal brief; verify all claims in brief have traceable `grounded_in` links to Layer A artifacts |

### Load Tests

| Test | Acceptance Criterion |
|---|---|
| `detectBlockers()` at large team scale | 50 work items, 20 team members; blocker detection completes within 2 seconds at p95 |
| `syncWorkItems()` on large backlog | 200 work items with 50 dependency edges; dependency graph builds without performance degradation |
