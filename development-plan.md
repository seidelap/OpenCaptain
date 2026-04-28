# OpenCaptain Development Plan

## Implementation Order

Dependencies first, highest independent value early. Each phase ends with end-to-end tests verifying the system behaves correctly through all completed modules.

---

## Phase 1: Configuration & Admin Interface Core (Module 8 — partial)

**Scope:** Configuration store, rule/whitelist CRUD, `getPolicyRules()`, `getThresholds()`, `getWhitelists()`, `getAdminDLs()`, `onConfigChanged()` subscriptions. Exclude the Admin UI and dry-run features (those depend on Module 3).

**Why first:** Every other module reads its configuration from Module 8. Building this first means all subsequent modules can be wired to real config from day one rather than hardcoded values.

**Deliverables:**
- Config store with versioning and rollback
- YAML/JSON config-as-code import (atomic, all-or-nothing)
- `getPlatformConfig()`, `getAdapterSettings()`, `getIdentityConfig()`, `getKnowledgeConfig()`, `getPolicyRules()`, `getThresholds()`, `getWhitelists()`, `getAdminDLs()`, `getMeetingConfig()`, `getGoalEngineConfig()`, `getAuditConfig()`
- `onConfigChanged()` pub/sub for all consumers
- Admin-only write enforcement (reject non-admin writes, log security events)
- Config versioning: every change versioned, previous versions retained

**End-to-End Tests — Phase 1:**
- Write a policy rule via config store; read it back via `getPolicyRules()`; verify round-trip fidelity
- Submit config file with one invalid rule; verify all-or-nothing rejection
- Modify a config value; verify `onConfigChanged()` fires for all subscribers with correct change payload
- Non-admin write attempt; verify rejected with security event logged
- Roll back to previous config version; verify previous values restored

---

## Phase 2: Platform Integration Layer + Identity & Permission Engine (Modules 1 & 2)

**Scope:** Both modules together because they have an unavoidable circular dependency (M1 calls M2 for identity resolution; M2 calls M1 for platform ACL queries). Building them in the same phase avoids a stub/mock cycle.

**Deliverables — Module 1:**
- Platform adapters for Slack and one additional platform (Teams or Jira)
- `normalizeEvent()` for all event types in the normalized schema
- `translateAction()` and `executeWithRetry()` for all action types
- `EventBus` subscribe/unsubscribe
- `ActionDispatcher.dispatchAction()`
- `queryPlatformACL()` and `queryGroupMembership()`
- OAuth token management and webhook signature validation
- Adapter health reporting (`getAdapterHealth()`)

**Deliverables — Module 2:**
- Unified identity registry with three-tier resolution (email → manual → heuristic)
- `resolveIdentity(ref: PlatformRef | EmailRef)` and `resolveGroup()`
- `canAccess()` with delegated + app-level modes
- `getEffectiveScope()` — intersection of OAuth scopes and platform ACL
- Group membership cache with configurable SLA and invalidation
- ACL snapshot writing to Layer A (stubbed until Module 4 is live; swap in Phase 3)

**End-to-End Tests — Phase 2:**
- Send a Slack message in test workspace; verify `MessageReceived` event fires on EventBus with correct normalized fields and resolved `UnifiedIdentity` on sender
- Dispatch `SendMessage` action; verify it appears in Slack test channel
- Seed Slack user and Jira user with same email; call `resolveIdentity()` for each; verify same `unified_id` returned
- `canAccess()` with delegated mode for a resource the delegating user cannot see; verify DENY even if agent OAuth scope technically allows it
- Group membership change in Slack; verify `resolveGroup()` reflects change within configured cache SLA
- Webhook with invalid HMAC signature; verify rejected before normalization

---

## Phase 3: Knowledge Store (Module 4)

**Scope:** All three layers (A, B, C), extraction pipeline, contradiction detection, cache invalidation.

**Deliverables:**
- Layer A: `writeArtifact()` (append-only, versioned), `queryArtifacts()`, `getProvenance()` with W3C PROV edges
- Layer B: `writeClaim()` with auto-contradiction detection, `queryClaimsForTopic()`, `queryClaimsForEntity()`, `getContradictions()`, `queryAuthorizations()`
- Layer C: `generateContextPack()` with mandatory retrieval manifest
- `ingestTranscript()` with async extraction pipeline (A → B)
- `resolveSupersession()` with governance evidence enforcement
- `invalidateCache()` propagating through retrieval manifests
- Integration with Module 2 for ACL snapshot writes (completing the stub from Phase 2)
- Integration with Module 8 for knowledge config

**End-to-End Tests — Phase 3:**
- Platform event (Slack message) flows from EventBus → `ingestEvent()` → Layer A artifact stored → async claim extraction → Layer B claim with `grounded_in` link to artifact
- Write two artifacts with conflicting requirement statements; verify `contradicts` edge created in Layer B; verify `getContradictions()` returns the pair
- Call `resolveSupersession()` with valid governance artifact; verify `supersedes` edge created and superseded claim gets `valid_until`
- Call `resolveSupersession()` with no evidence; verify rejected
- Generate context pack; verify `retrieval_manifest` is populated and non-null
- Update a Layer A artifact; verify all Layer C artifacts referencing it are marked `invalidated_by`
- Write MEETING_STATE artifact; query it back via `queryArtifacts(artifact_type: MEETING_STATE)`; verify correct state returned

---

## Phase 4: Audit & Observability Layer (Module 7)

**Scope:** Immutable audit log, policy decision log, compliance reports, dashboards, anomaly detection, alerts.

**Deliverables:**
- `logEvent()` — append-only, validated, indexed
- `logPolicyDecision()` — separate policy decision store
- `queryAuditLog()` with `log_type` filter (GENERAL | POLICY_DECISION | ALL)
- `getComplianceReport()` for all five report types (ACTIVITY_SUMMARY, APPROVAL_SUMMARY, BOUNDARY_CROSSING, CONTRADICTION_STATUS, FULL_AUDIT)
- `getDashboardData()` for all dashboard types
- `subscribeToAlerts()` with all alert types wired
- `detectAnomalies()` with configurable sensitivity
- `traceAction()` — full decision chain reconstruction
- Integration with Module 4 (`queryArtifacts()`, `getProvenance()`, `getContradictions()`)

**End-to-End Tests — Phase 4:**
- Every module that has been built to this point (M1, M2, M4, M8) calls `logEvent()` for its significant operations; verify all entries appear in `queryAuditLog()` with correct `module_source`
- Platform event → Module 1 normalizes → Module 4 ingests; verify audit entries present for both steps; `traceAction()` reconstructs the chain
- Trigger N+1 boundary-crossing events in a window; verify BOUNDARY_CROSSING_SPIKE alert fires
- Attempt to modify or delete an existing audit entry; verify rejected; verify original entry unchanged
- Generate CONTRADICTION_STATUS report; verify it reads open contradictions from Module 4's Layer B
- Config change in Module 8; verify audit entry created with old/new values

---

## Phase 5: Policy & Governance Engine (Module 3)

**Scope:** Rules engine, context assembler, outbound gate, approval module. Also completes the Module 8 admin features that depend on Module 3 (dry-run, `validatePolicyRule()`).

**Deliverables:**
- `submitAction()` — full evaluation pipeline: context assembly → rule evaluation → verdict → gate/approval/deny
- `submitAction(dry_run: true)` — verdict-only mode with no side effects
- Context assembler reading MEETING_STATE from Module 4, active authorizations from Authorization Store
- Rules engine with most-specific-wins, unless-conditions, default-deny
- Outbound Gate dispatching ALLOW verdicts through Module 1
- Approval module: `requestApproval()`, `processApproval()`, `AuthorizationRecord` written to Layer A
- `validatePolicyRule()` — exposed to Module 8 for rule authoring
- Module 8 Phase 1 completion: `testPolicyRule()` dry-run workflow, Admin UI policy authoring
- Policy decision records pushed to Module 7 via `logPolicyDecision()`

**End-to-End Tests — Phase 5:**
- Full action lifecycle: event arrives → Module 4 extracts claims → Module 3 evaluates proposed action → ALLOW → Module 1 dispatches → Module 7 logs; verify every step produces correct artifacts and log entries
- Out-of-scope message: Goal Engine proposes message to non-whitelisted channel → Policy Engine returns REQUIRE_APPROVAL → approval request sent to admin DL → admin approves → original action executes
- DENY rule: action matching a hard DENY rule; verify never dispatched; denial reason in audit log
- Authorization expiry: approve action with 60-second TTL; verify ALLOW at T=30; verify DENY at T=90 with expiry reason
- Self-approval attempt: non-admin approves own proposed action; verify SELF_APPROVAL_ATTEMPT security event
- Dry-run: Admin tests new rule via `testPolicyRule()`; verify verdict returned; verify no action dispatched, no approval created, no action audit entry

---

## Phase 6: Team Goal Engine (Module 6)

**Scope:** Three-stage decision engine over the unified `Goal` Entity tree — tree maintenance, candidate scoring (heuristic + LLM tier-up), action proposal.

**Prerequisites in Module 4:** `Goal` Entity type, `decomposes_into` edge, `DETECTION_RECORD` artifact type, `subscribeInvalidations()` interface.

**Deliverables:**

*Stage 1 — Tree Maintenance:*
- `handleEvent()` — react to `TicketUpdated`/`TicketCreated`/`MessageReceived`/`DecisionRecorded`; reconcile structural edges with platform state
- `propagateProgress()` — walk `decomposes_into` upward; write versioned `progress_assessment` Claims grounded in DETECTION_RECORD
- `cascadeResolution()` — walk `decomposes_into` downward when a Goal is resolved; auto-resolve active descendants with `nature: obsoleted_by_parent` to preserve the active-view invariant
- `mergeGoals()` — merge authored and extracted Goals via `same_as` when they refer to the same outcome

*Stage 2 — Candidate Scoring:*
- `enumerateCandidates()` — exhaustive enumeration of `clarify_conflict`, `coverage_gap`, `stale_dependency`, `cross_team_unblock`, `commitment_tension` candidates from the active goal tree
- `scoreHeuristic()` — deterministic weighted-feature score with confidence
- `scoreLLM()` — mini-model (Claude Haiku) tier-up on heuristic close-calls, using prompt caching for the static framework portion
- `recordScore()` — write `goal_relevance` Claim + `DETECTION_RECORD` artifact; maintain reverse index for invalidation
- `recomputeOnInvalidation()` — reactive re-scoring via `subscribeInvalidations()`

*Stage 3 — Action Proposal:*
- `proposeActions()` — `submitAction()` to Policy with score and DETECTION_RECORD ID in metadata
- `logSilence()` — audit-log below-threshold candidates so silence is a recorded decision

All telemetry pushed to Module 7 via `logEvent()`.

**End-to-End Tests — Phase 6:**
- Cross-team unblock worked example (Module 6 §10): seed `decomposes_into` chain ending at a `cross_team_unblock` candidate; verify enumeration → scoring → proposal → Policy approval cycle → resolution detection on inbound reply → cascade to ancestors
- Active-view invariant: resolve a top-level Goal with active descendants; query `status: active`; verify no orphaned subtrees
- Reactive recomputation: score a candidate; modify an underlying Claim; verify only affected scores recomputed (not the whole tree)
- Conflict on inactive subtree stays silent: seed long-dormant `relates_to` edge; activate dependent Goal; verify score crosses threshold and proposal fires
- Authored vs. extracted merge: author a Goal; have extraction independently create a same-subject Goal; trigger `mergeGoals()`; verify single canonical Goal with combined `grounded_in` chain
- Silence audit: candidate scored below threshold; verify single audit entry; subsequent below-threshold scores within delta do not duplicate
- Heuristic-only mode (LLM tier-up disabled): all candidates scored on heuristic alone; confidence-flagged candidates still produce scores

---

## Phase 7: Meeting Participation Engine (Module 5)

**Scope:** Real-time meeting state management, contribution evaluation, transcript ingestion, multi-instance coordination.

**Deliverables:**
- `handleMeetingEvent()` — full meeting lifecycle state machine
- `decideMeetingContribution()` — contribution relevance scoring against all criteria
- `composeContribution()` — context-aware contribution with epistemic labels
- `handleCalledUpon()` — immediate response pathway
- `ingestTranscriptChunk()` — real-time ingestion into Module 4
- `writeMeetingStateArtifact()` — MEETING_STATE written on every state transition
- Cross-meeting context (claims from concurrent meeting sessions available as contribution context)
- All telemetry pushed to Module 7 via `logEvent()`

**End-to-End Tests — Phase 7:**
- MeetingStarted event → state machine initialized → MEETING_STATE artifact written → readable by Policy Engine via Module 4
- Real-time transcript fed to engine → ingested into Layer A → claims extracted to Layer B → within same meeting, agent detects relevant contradiction and proposes RaiseHand
- Agent called upon (CalledUpon event) → agent immediately proposes Speak action → Policy Engine evaluates → action dispatched → transcript of agent's contribution ingested
- Large meeting (> small-meeting threshold): verify `speak.require_called_upon` rule fires correctly by checking MEETING_STATE participant count in policy evaluation
- Two concurrent meeting instances: verify claim from meeting A is available as cross-meeting context in meeting B; verify no state bleed between instances
- All agent contributions contain epistemic labels; verify no raw assertion without provenance marker

---

## Phase 8: Module 8 — Admin UI & Full Admin Capabilities

**Scope:** Web UI, alert subscriptions surface, compliance report scheduling, anomaly dashboard, full admin workflow.

**Deliverables:**
- Web UI for policy authoring, whitelist management, threshold tuning
- Chat-based admin commands (direct message to agent)
- Compliance report scheduling (automated delivery)
- Alert subscription management UI
- Anomaly detection dashboard
- Full dry-run workflow in UI: write rule → preview verdict on hypothetical action → publish

**End-to-End Tests — Phase 8 (Full System):**
- **Admin onboarding flow:** Admin adds team DL to whitelist via Web UI; verifies agent can now communicate with team; verifies audit trail of the config change
- **Policy author + dry-run:** Admin writes DENY rule for cross-team messages; dry-runs against hypothetical action; sees DENY verdict; publishes rule; Goal Engine proposes the blocked action; verify DENY with correct denial reason
- **Full approval lifecycle:** Agent detects blocker requiring out-of-scope contact; generates approval request; admin receives request (Slack DM); approves with condition ("CC admin on all messages"); agent executes with condition applied; compliance report shows boundary crossing with full authorization chain
- **Contradiction lifecycle:** Two team members state conflicting requirements in separate meetings; Goal Engine surfaces disagreement in team channel; team resolves disagreement in meeting; agent observes resolution; resolution claim created with `supersedes` edge; contradiction cleared from CONTRADICTION_STATUS report
- **Audit trace:** Admin pulls FULL_AUDIT report for a 24-hour window; selects one action; calls trace; verifies full chain from originating platform event through policy evaluation to dispatch
- **Anomaly detection:** Simulate burst of policy denials; verify HIGH_DENIAL_RATE alert fires and appears in admin dashboard; verify alert dismissed after rate returns to baseline
- **Multi-team, multi-meeting load:** Two teams running concurrent goal tracking; two active meeting sessions; verify no cross-team data leakage; all actions correctly scoped; no performance degradation
