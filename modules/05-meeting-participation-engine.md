# Module 5: Meeting Participation Engine

## 1. Purpose

Manages the agent's real-time behavior during meetings: listening, deciding when to contribute, raising hands, speaking when called upon, and capturing information for the Knowledge Store. The agent may participate in **multiple concurrent meetings simultaneously**, each handled by an independent engine instance with cross-meeting context available through the Knowledge Store. This module is a **decision-making component only** — it decides what to contribute and when, but every outbound action (speak, raise hand) is proposed to the Policy Engine for evaluation and dispatch.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | Inbound meeting events via EventBus (`MeetingStarted`, `MeetingEnded`, `ParticipantJoined`, `ParticipantLeft`, `HandRaiseDetected`, `CalledUpon`, `MessageReceived` in meeting context) | The engine receives all meeting lifecycle and interaction events to maintain meeting state and detect contribution opportunities |
| **Identity & Permission Engine (Module 2)** | `resolveIdentity()`, `resolveGroup()` | Resolves meeting participant identities and checks organizer/participant group memberships for policy context |
| **Policy & Governance Engine (Module 3)** | `submitAction()` | Every proposed action (RaiseHand, Speak) is submitted to the Policy Engine for evaluation. The engine never dispatches actions directly. |
| **Knowledge Store (Module 4)** | `ingestTranscript()`, `writeArtifact()`, `queryClaimsForTopic()`, `getContradictions()`, `generateContextPack()` | Ingests transcripts and writes meeting state artifacts to Layer A; queries Layer B for relevant claims, contradictions, and commitments; requests Layer C context packs |
| **Audit & Observability Layer (Module 7)** | `logEvent()` | Pushes telemetry (meeting state transitions, contribution attempts, delivery outcomes, latency) to the immutable audit log |
| **Configuration & Admin Interface (Module 8)** | `getMeetingConfig()` | Meeting behavior thresholds (small meeting threshold, contribution relevance threshold, devils advocate toggle, hand-raise rate limits) |

### 2.2 Modules That Depend On This Module

*None. The Meeting Engine is a true leaf consumer — it reads from other modules and pushes events outward. No module calls into it.*

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

*None. This module is a leaf consumer. Meeting state is written to the Knowledge Store (Layer A) as `MEETING_STATE` artifacts on every state change — the Policy Engine and Audit Layer read from there.*

### 3.2 Required Interfaces (This Module Consumes)

#### From Platform Integration Layer (Module 1):

```
subscribe(event_types: [MeetingStarted, MeetingEnded, ParticipantJoined, ParticipantLeft,
                        HandRaiseDetected, CalledUpon, MessageReceived], callback) -> SubscriptionHandle
```

#### From Identity & Permission Engine (Module 2):

```
resolveIdentity(platform: Platform, platform_user_id: string) -> UnifiedIdentity?
resolveGroup(group_ref: UnifiedGroupRef) -> ResolvedGroup
```

#### From Policy & Governance Engine (Module 3):

```
submitAction(proposed_action: ProposedAction, source_module: MEETING_ENGINE) -> ActionOutcome
```

#### From Knowledge Store (Module 4):

```
ingestTranscript(transcript: TranscriptChunk, meeting_id: string) -> ArtifactId
writeArtifact(artifact: LayerAArtifact) -> ArtifactId
  — Writes MEETING_STATE artifacts on every state transition (participant count, hand-raise
    status, called-upon status, agent behavioral state). This replaces the former
    getMeetingState() pull interface — consumers read state from Module 4 instead.
queryClaimsForTopic(topic: string, scope: ClaimScope?, as_of: timestamp?) -> ClaimSet
getContradictions(scope: ClaimScope?, topic: string?) -> ConflictPair[]
generateContextPack(query: ContextQuery) -> LayerCArtifact
```

#### From Audit & Observability Layer (Module 7):

```
logEvent(entry: AuditLogEntry) -> AuditLogId
  — Push meeting telemetry: state transitions, contribution decisions, delivery outcomes, latency
```

#### From Configuration & Admin Interface (Module 8):

```
getMeetingConfig() -> MeetingConfig
  — Returns: small_meeting_threshold, contribution_relevance_threshold,
    devils_advocate_enabled, hand_raise_rate_limit, max_speak_duration_seconds,
    cross_meeting_context_enabled, anonymize_cross_meeting_sources
```

---

## 4. Core Functions

### 4.1 `handleMeetingEvent(event: OpenCaptainEvent) -> void`

**Inputs:**
- `event` — a meeting-related platform event (MeetingStarted, ParticipantJoined, ParticipantLeft, MeetingEnded, CalledUpon, HandRaiseDetected)

**What it does:**
Routes the event to the correct meeting instance (creates a new instance for `MeetingStarted`). Updates the Meeting State Tracker: participant list, current speaker, hand-raise queue, agenda position. On `ParticipantJoined`/`ParticipantLeft`: updates participant count and triggers policy re-evaluation — if the count crosses the small meeting threshold, the agent's behavioral constraints change (e.g., must now raise hand, or can now speak freely). On `CalledUpon`: transitions the agent's state from `HAND_RAISED` to `SPEAKING` (if hand was raised) or directly to `SPEAKING` (if asked a direct question). On `MeetingEnded`: transitions to `IDLE`, writes a meeting summary artifact to Layer A.

### 4.2 `ingestTranscriptChunk(transcript: TranscriptChunk, meeting_id: string) -> void`

**Inputs:**
- `transcript` — real-time transcript segment with speaker attribution and timestamp
- `meeting_id` — which meeting this belongs to

**What it does:**
Writes the transcript chunk to Layer A via `ingestTranscript()` on the Knowledge Store (triggers async claim extraction to Layer B). Updates the current topic tracker based on transcript content. Feeds the transcript to the Contribution Evaluator to check if the agent should contribute.

### 4.3 `evaluateContribution(meeting_id: string, transcript_context: string) -> ContributionDecision`

**Inputs:**
- `meeting_id` — which meeting to evaluate for
- `transcript_context` — recent transcript for context

**What it does:**
This is the Contribution Evaluator — the core decision function that determines "should I contribute right now?" Queries the Knowledge Store for claims relevant to the current discussion topic. Checks for unresolved contradictions in the B graph that are relevant to the current agenda item. Checks for at-risk commitments or deadlines tracked in B. If `cross_meeting_context_enabled`, checks for observations from concurrent meeting sessions that are relevant. Computes a relevance score based on: direct topical relevance, contradiction urgency, commitment risk, and whether the agent was explicitly asked a question. Compares the relevance score against the configured threshold. Returns one of:
- `SILENT` — nothing relevant enough to contribute
- `QUEUE_CONTRIBUTION` — relevant content found; prepare a contribution
- `RAISE_HAND` — contribution queued and policy requires hand-raise (large meeting)
- `SPEAK` — contribution queued and policy allows direct speech (small meeting or called upon)

### 4.4 `composeContribution(meeting_id: string, trigger: ContributionTrigger) -> ProposedContribution`

**Inputs:**
- `meeting_id` — which meeting the contribution is for
- `trigger` — what triggered the contribution (relevant_claim, contradiction, at_risk_commitment, direct_question, devils_advocate)

**What it does:**
This is the Contribution Composer. Requests a Layer C context pack from the Knowledge Store scoped to the current discussion topic and audience. If the trigger is a **contradiction**: composes a statement that presents both sides with epistemic labels ("Two different understandings have been stated for this: [A] and [B]"). If the trigger is a **relevant claim**: composes a brief, factual contribution with source attribution and confidence labeling. If the trigger is an **at-risk commitment**: composes a risk flag with the specific dependency or deadline at risk. If the trigger is a **direct question**: composes a direct answer drawing from the Knowledge Store. If the trigger is **devils_advocate** (enabled by config, and discussion has converged with low objection count): surfaces counter-arguments from B graph or prior discussions, always prefixed with "For consideration, a previous discussion raised the concern that..." If `anonymize_cross_meeting_sources` is enabled: strips any session-identifying information from cross-meeting context. Applies **epistemic labels** to all output: current approved position, pending contradiction, unverified meeting assertion, awaiting human adjudication. Returns the proposed contribution as a `Speak` action to be submitted to the Policy Engine.

### 4.5 `submitContribution(meeting_id: string, contribution: ProposedContribution) -> void`

**Inputs:**
- `meeting_id` — which meeting
- `contribution` — the composed contribution

**What it does:**
Based on the current meeting state and policy requirements:
- If hand-raise is required (large meeting): first submits a `RaiseHand` action to the Policy Engine via `submitAction()`. Transitions state to `HAND_RAISED`. The contribution is queued and will be delivered when `CalledUpon` event is received.
- If direct speech is allowed (small meeting or already called upon): submits a `Speak` action to the Policy Engine via `submitAction()`.
- If the Policy Engine returns `DENIED`: logs the suppressed contribution (content + reason) to the Audit Layer. Does not speak. Returns to `LISTENING`.
- If the Policy Engine returns `DISPATCHED`: logs the delivered contribution. Transitions to `LISTENING`.

### 4.6 `handleStateTransition(meeting_id: string, new_event: MeetingEvent) -> void`

**Inputs:**
- `meeting_id` — which meeting
- `new_event` — the triggering event

**What it does:**
Implements the behavioral state machine:

```
IDLE -> LISTENING                    (MeetingStarted, agent is participant)
LISTENING -> CONTRIBUTION_QUEUED     (evaluator detects relevant input)
CONTRIBUTION_QUEUED -> HAND_RAISED   (policy requires hand-raise for this meeting size)
HAND_RAISED -> SPEAKING              (CalledUpon received)
CONTRIBUTION_QUEUED -> SPEAKING      (small meeting, policy allows direct speech)
SPEAKING -> LISTENING                (contribution delivered)
LISTENING -> IDLE                    (MeetingEnded)
```

At any state: if participant count changes (join/leave), re-evaluates policy constraints. If the meeting crosses the small-meeting threshold while the agent has a queued contribution, transitions from `CONTRIBUTION_QUEUED` to `HAND_RAISED`. If a participant leaves and drops below threshold, `HAND_RAISED` can transition directly to `SPEAKING` (cancels hand-raise, speaks directly). If the agent is never called upon and the meeting ends, the queued contribution is logged but never delivered (restraint behavior).

---

## 5. Concurrent Meeting Participation

Each active meeting runs as an **independent engine instance** with its own state machine. The instances share no direct state — coordination happens through the Knowledge Store:

- Transcript ingestion from any meeting enters Layer A and triggers claim extraction to Layer B
- The Contribution Evaluator in meeting X can query the Knowledge Store and find claims that were just extracted from meeting Y
- Cross-meeting context is subject to the same relevance threshold, policy evaluation, and epistemic labeling as any other contribution
- If `anonymize_cross_meeting_sources` is enabled, the contribution composer strips meeting identifiers from cross-session references

---

## 6. Contribution Relevance Criteria

| Criterion | Signal | Weight |
|---|---|---|
| Context from Knowledge Store is directly relevant to current topic | Semantic similarity score > threshold | High |
| Unresolved contradiction in B graph relevant to current agenda item | `contradicts` edge in same topic scope | High |
| Commitment or deadline tracked in B is at risk | Dependency conflict or stale action item | Medium |
| Relevant context observed in a concurrent meeting session | Cross-meeting claim match | Medium |
| Agent was explicitly asked a question | Direct address detected in transcript | Always respond (highest) |
| Discussion converging too quickly (devil's advocate) | Low objection count, high agreement signal | Low (only if enabled) |

---

## 7. Test Plan

### Unit Tests

| Function | Test | Expected |
|---|---|---|
| `handleMeetingEvent()` | `MeetingStarted` event received | New meeting state machine initialized; MEETING_STATE artifact written to Layer A |
| `handleMeetingEvent()` | `MeetingEnded` event received | State machine finalized; final MEETING_STATE artifact written; telemetry pushed to Audit Layer |
| `handleMeetingEvent()` | `ParticipantJoined` event — participant below small-meeting threshold | Meeting state updated; participant identity resolved; MEETING_STATE artifact written |
| `decideMeetingContribution()` | Agent is not called upon; no direct question | No RaiseHand proposed unless relevance threshold met |
| `decideMeetingContribution()` | Direct question addressed to agent in transcript | `submitAction(Speak)` proposed immediately; highest-priority trigger |
| `decideMeetingContribution()` | Unresolved contradiction in B graph relevant to current topic | RaiseHand proposed with relevance reason; contribution labeled epistemically |
| `decideMeetingContribution()` | Relevance score below threshold | No action proposed; decision logged |
| `composeContribution()` | Contribution contains claim from another meeting session | Cross-meeting claim included; epistemic label attached; source anonymized if configured |
| `composeContribution()` | Contribution context exceeds token budget | Retrieval manifest records truncation; contribution composed from top-ranked sources only |
| `handleCalledUpon()` | Agent called upon during meeting | `submitAction(Speak)` submitted immediately; context pack retrieved for content |
| `ingestTranscriptChunk()` | Real-time transcript chunk received | Written to Layer A via `ingestTranscript()`; async extraction triggered; speaker identity resolved |
| `writeMeetingStateArtifact()` | State transition occurs (participant joins) | `MEETING_STATE` artifact written to Layer A with meeting_id, participant_count, agenda phase |

### Integration Tests

| Test | Approach |
|---|---|
| Meeting transcript → Layer B claims | Feed gold-standard meeting transcript; verify commitments, decisions, and entities appear in Layer B with correct `grounded_in` links |
| Contribution relevance: contradiction in B triggers hand raise | Insert contradicting claims in B graph for a topic; simulate meeting discussion on that topic; verify agent proposes RaiseHand |
| Policy gate: speak action evaluated before dispatch | Meeting engine proposes Speak action; verify Policy Engine is called; verify action only dispatched if ALLOW verdict returned |
| MEETING_STATE readable by Policy Engine | Meeting Engine writes MEETING_STATE artifact; Policy Engine queries it via Knowledge Store; verify correct participant count and phase returned |
| Concurrent meetings: cross-meeting context | Run two simulated meeting instances; verify claim extracted from meeting A is available as context in meeting B via Knowledge Store |

### Load Tests

| Test | Acceptance Criterion |
|---|---|
| Transcript ingestion at real-time speed | Ingestion and extraction pipeline keeps pace with transcript stream; no backlog at p95 |
| Concurrent meeting instances | 10 concurrent meeting instances; no state bleed between instances; MEETING_STATE artifacts correctly partitioned by meeting_id |
