# Module 3: Policy & Governance Engine

## 1. Purpose

The behavioral and communication governance brain of OpenCaptain. Encodes and enforces all etiquette rules, social constraints, and configurable thresholds that govern **when, where, how, and to whom** the agent may act or communicate. Every proposed action in the system — whether from the Meeting Engine, Goal Engine, or another agent — must pass through this module before it reaches the outside world. The module comprises three sub-components: the **Core Rules Engine** (evaluate), the **Outbound Gate** (enforce), and the **Approval Module** (manage human approvals).

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | `dispatchAction()` | The Outbound Gate dispatches ALLOW-verdicted actions through the Platform Integration Layer for delivery to the target platform |
| **Identity & Permission Engine (Module 2)** | `resolveIdentity()`, `resolveGroup()`, `canAccess()`, `getEffectiveScope()` | The Context Assembler calls the Identity Engine to resolve identities, determine group memberships (whitelist checks use `resolveGroup().members`), and verify platform-level permissions |
| **Knowledge Store (Module 4)** | `writeArtifact()`, `queryAuthorizations()` | AuthorizationRecords are durably written to Layer A as governance artifacts. The Context Assembler queries active authorizations from the Authorization Store (which rebuilds from Layer A on restart) |
| **Audit & Observability Layer (Module 7)** | `logEvent()` | Every policy evaluation (ALLOW, DENY, REQUIRE_APPROVAL) is pushed to the Audit Layer's immutable log via `logEvent()` |
| **Configuration & Admin Interface (Module 8)** | `getPolicyRules()`, `getThresholds()`, `getWhitelists()`, `getAdminDLs()` | All policy rules, thresholds, whitelist configurations, and admin DL definitions are sourced from the config module |

### 2.2 Modules That Depend On This Module

| Consumer | What It Uses | Why |
|---|---|---|
| **Meeting Participation Engine (Module 5)** | `submitAction()` | Every proposed meeting action (speak, raise hand) is submitted to this module for policy evaluation before execution |
| **Team Goal Engine (Module 6)** | `submitAction()` | Every proposed goal-related action (surface blocker, suggest sync, request access) is submitted for policy evaluation |
| **Audit & Observability Layer (Module 7)** | Policy decision records | Every evaluation (ALLOW, DENY, REQUIRE_APPROVAL) is logged with full context for the audit trail |
| **Configuration & Admin Interface (Module 8)** | Policy validation feedback | When admins author or modify policies, this module validates the rules and returns any errors or conflicts |

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

#### Action Submission (Outbound Gate)

```
submitAction(proposed_action: ProposedAction, source_module: ModuleId, dry_run: bool = false) -> ActionOutcome
  Inputs:
    proposed_action     — the action to evaluate and potentially dispatch
    source_module       — which module is proposing this action (MEETING_ENGINE, GOAL_ENGINE, etc.)
    dry_run             — if true, runs context assembly + policy evaluation but does NOT dispatch
                          or enqueue for approval. Returns verdict only. Used by the Admin
                          Interface to test rule changes against hypothetical actions.
  Returns:
    ActionOutcome {
      status: DISPATCHED | DENIED | PENDING_APPROVAL | DRY_RUN_VERDICT,
      action_id: string,
      verdict: PolicyVerdict?,              // populated when dry_run: true
      approval_request_id: string?,         // if PENDING_APPROVAL
      dispatch_result: ActionResult?        // if DISPATCHED
    }

  PolicyVerdict {
    decision: ALLOW | DENY | REQUIRE_APPROVAL,
    rules_evaluated: RuleEvaluation[],
    binding_rule: RuleId?,
    denial_reason: string?,
    approval_target: UnifiedGroupRef?,      // admin DL, if REQUIRE_APPROVAL
    conditions: string[]?                    // any ALLOW conditions (e.g., "admin must be CC'd")
  }
```

This is the **single entry point** for all modules that want to execute actions. It wraps the full pipeline: context assembly -> policy evaluation -> outbound gate -> dispatch or approval queue. Pass `dry_run: true` for policy testing without side effects.

#### Approval Status Query

```
getApprovalStatus(action_id: string) -> ApprovalStatus
  Inputs:
    action_id           — the pending action ID
  Returns:
    ApprovalStatus { status: PENDING | APPROVED | DENIED | EXPIRED, authorization: AuthorizationRecord? }

getActiveAuthorizations(scope: AuthorizationScope) -> AuthorizationRecord[]
  Inputs:
    scope               — filter by action scope, target scope, or approver
  Returns:
    List of currently active (non-expired) authorization records
```

#### Policy Validation (for Admin Interface)

```
validatePolicyRule(rule: PolicyRule) -> ValidationResult
  Inputs:
    rule                — a proposed policy rule in declarative format
  Returns:
    ValidationResult { valid: bool, errors: string[], warnings: string[], conflicts_with: RuleId[] }
```

### 3.2 Required Interfaces (This Module Consumes)

#### From Platform Integration Layer (Module 1):

```
dispatchAction(action: OpenCaptainAction) -> ActionResult
  — Outbound Gate dispatches approved actions for platform delivery
```

#### From Identity & Permission Engine (Module 2):

```
resolveIdentity(ref: PlatformRef | EmailRef) -> UnifiedIdentity?
resolveGroup(group_ref: UnifiedGroupRef) -> ResolvedGroup
  — Whitelist membership is checked as: resolveGroup(whitelist_dl).members.contains(identity)
canAccess(agent_or_user: UnifiedIdentity, resource: ResourceRef, action: ActionType) -> PermissionResult
getEffectiveScope(agent: UnifiedIdentity, platform: Platform) -> EffectiveScope
```

#### From Knowledge Store (Module 4):

```
writeArtifact(artifact: LayerAArtifact) -> ArtifactId
  — Write AuthorizationRecords and approval audit records to Layer A

queryAuthorizations(scope: AuthorizationScope) -> AuthorizationRecord[]
  — Query active authorizations from Layer A for the Authorization Store cache

queryArtifacts(query: ArtifactQuery) -> LayerAArtifact[]
  — Context Assembler queries MEETING_STATE artifacts to get participant count,
    hand-raise status, and called-upon state for evaluating meeting policy rules.
    Replaces the former getMeetingState() call to Module 5, breaking the circular dependency.
```

#### From Audit & Observability Layer (Module 7):

```
logEvent(entry: AuditLogEntry) -> AuditLogId
  — Log every policy evaluation result to the immutable audit log
```

#### From Configuration & Admin Interface (Module 8):

```
getPolicyRules() -> PolicyRule[]
getThresholds() -> ThresholdConfig
getWhitelists() -> WhitelistConfig
getAdminDLs() -> AdminDLConfig
onConfigChanged(callback: (change: ConfigChange) -> void) -> void
```

---

## 4. Core Functions

### 4.1 `assembleContext(proposed_action: ProposedAction) -> EvaluationContext`

**Inputs:**
- `proposed_action` — the action being evaluated, including target, content, and source module

**What it does:**
Builds the full evaluation context needed by policy rules. Resolves the recipient's identity and group memberships via the Identity Engine. Determines whether the recipient is within the whitelisted team DL. Retrieves active authorizations from the Authorization Store that may apply to this action scope. Retrieves meeting state if the action is meeting-related (participant count, whether agent was called upon, hand-raise status). Attaches temporal context: current time, day of week, whether within working hours. Returns a complete `EvaluationContext` object that policy rules can match against.

### 4.2 `evaluateRules(context: EvaluationContext, proposed_action: ProposedAction) -> PolicyVerdict`

**Inputs:**
- `context` — fully assembled evaluation context
- `proposed_action` — the action being checked

**What it does:**
Loads all active policy rules from the Policy Rule Store (sourced from Module 8). Evaluates each rule's `condition` clause against the context. For matching rules, checks the `unless` clauses (exceptions like "agent was called upon" or "active authorization exists"). For matching rules without satisfied exceptions, applies the rule's `effect` (ALLOW, DENY, REQUIRE_APPROVAL). When multiple rules match and conflict, applies precedence: **most-restrictive wins** (DENY > REQUIRE_APPROVAL > ALLOW). Checks `override` clauses that can relax a restriction (e.g., meeting organizer blanket permission). Returns the final verdict with a trace of all rules evaluated and which rule was binding.

### 4.3 `enforceVerdict(verdict: PolicyVerdict, proposed_action: ProposedAction) -> ActionOutcome`

**Inputs:**
- `verdict` — the policy evaluation result
- `proposed_action` — the original action

**What it does:**
This is the Outbound Gate's core logic. If `ALLOW`: dispatches the action via the Platform Integration Layer's `dispatchAction()`. If the action was approved via an out-of-scope authorization, attaches admin CC per policy. Logs the dispatch to the Audit Layer. If `DENY`: discards the action. Logs the denial with the binding rule and reason. Returns the denial to the calling module. If `REQUIRE_APPROVAL`: forwards the action to the Approval Module's Pending Action Queue. Generates an approval request and sends it to the appropriate admin DL. Returns `PENDING_APPROVAL` status with the approval request ID.

### 4.4 `generateApprovalRequest(proposed_action: ProposedAction, verdict: PolicyVerdict) -> ApprovalRequest`

**Inputs:**
- `proposed_action` — the action that needs approval
- `verdict` — the policy verdict (specifies which admin DL to target)

**What it does:**
Creates a human-readable approval request message that includes: a description of what the agent wants to do, who the target is, why the agent wants to do it (justification from the source module), what policy triggered the approval requirement, suggested scope for the approval (per-conversation, per-person, per-topic), and suggested duration. Queues the action in the Pending Action Queue with an expiry timer. Sends the approval request to the admin DL via `submitAction()` (which itself passes through policy — but "request approval from admin" is always an ALLOW action).

### 4.5 `parseApprovalResponse(message: OpenCaptainEvent) -> ParsedApproval`

**Inputs:**
- `message` — a `MessageReceived` or `ApprovalReceived` event from an admin in the admin DL channel/thread

**What it does:**
Uses LLM-based natural language parsing to extract the approval intent from the admin's response. Classifies the response as: APPROVED, DENIED, CONDITIONAL, or AMBIGUOUS. Extracts scope constraints: "only for this topic," "just this once," "for this week," "only for person X." Extracts duration: "until Friday," "for 24 hours," "permanently." If AMBIGUOUS: triggers the Confirmation Loop — echoes the parsed interpretation back to the admin and waits for confirmation. Rejects self-approvals (agent cannot approve its own requests). Rejects approvals from non-admin users (verifies approver is a member of the admin DL via Identity Engine).

### 4.6 `normalizeApproval(parsed: ParsedApproval, pending_action: PendingAction) -> AuthorizationRecord`

**Inputs:**
- `parsed` — the parsed and confirmed approval
- `pending_action` — the original pending action that was awaiting approval

**What it does:**
Converts the parsed approval into a structured `AuthorizationRecord` with all fields: approver identity, action scope (what actions are authorized), target scope (who can be contacted), duration/expiry, conditions (e.g., "admin must be CC'd"), observer list, and the source message ID that contained the approval. **Critical:** The AuthorizationRecord is created and written to Layer A *before* the authorized action executes. Writes the record to the Authorization Store (operational cache) and durably to Layer A via the Knowledge Store. Ensures the approval scope cannot exceed the original request scope (even if the admin says "approved for X and Y," only X is authorized if X was requested).

### 4.7 `evaluateAgentToAgentAction(message: AgentMessage) -> PolicyVerdict`

**Inputs:**
- `message` — structured agent-to-agent message with header fields

**What it does:**
Validates the `authority_chain` field is populated (rejects empty chains). Verifies the source agent identity via platform service principal (prevents impersonation). Checks that both agents' admin DLs have approved the inter-agent communication. Verifies the source agent has the permissions it claims (no agent can grant another agent permissions it doesn't have). Applies all standard policy rules — agent-to-agent actions pass through the same pipeline as agent-to-human actions. Returns the verdict. All inter-agent actions are logged with full provenance.

---

## 5. Key Data Structures

### Policy Rule Format (Declarative YAML)

```yaml
- id: string
  description: string
  condition:
    event_type: EventType
    [field]: { operator: value }          # e.g., participant_count: { gt: 3 }
  action: ActionType                       # which action this rule governs
  effect: ALLOW | DENY | REQUIRE_APPROVAL
  unless:                                  # exceptions that flip the effect
    - [condition]: value
  override:                                # higher-level overrides
    - approval_type: string
  priority: number                         # for ordering; lower = higher priority
```

### AuthorizationRecord

```
AuthorizationRecord {
  authorization_id: string (UUID)
  approver: UnifiedIdentity
  action_scope: ActionScope                # what actions are authorized
  target_scope: TargetScope                # who can be contacted
  duration: Duration
  expires_at: timestamp
  conditions: string[]                     # e.g., ["admin_cc_required"]
  observers: UnifiedIdentity[]
  created_at: timestamp
  approval_source_message_id: string       # provenance link to the chat message
  pending_action_id: string                # link to the original request
  status: ACTIVE | EXPIRED | REVOKED
}
```

### AgentMessage Header

```
AgentMessage {
  source_agent_id: string
  target_agent_id: string
  message_type: QUERY | INFORM | REQUEST_ACTION | COORDINATE
  topic_scope: string
  authority_chain: AuthorizationId[]       # must be non-empty
  content: any
  epistemic_status: string
  response_requested: bool
  urgency: LOW | NORMAL | HIGH
}
```

---

## 6. Policy Categories

| Category | Example Rule IDs | Effect |
|---|---|---|
| **Meeting behavior** | `meeting.speak.require_called_upon`, `meeting.speak.hand_raise_required`, `meeting.speak.max_duration_seconds`, `meeting.join.auto_join`, `meeting.speak.priority` | Controls when/how agent speaks in meetings based on participant count thresholds |
| **Communication boundaries** | `comms.initiate.allowed_groups`, `comms.initiate.outside_whitelist`, `comms.respond.outside_whitelist`, `comms.initiate.always_cc_admin` | Controls who the agent can contact and requires approval for out-of-scope communication |
| **Agent-to-agent** | `agent.comms.allowed`, `agent.comms.require_human_in_loop` | Both agents' admin DLs must approve; human always in the loop by default |
| **Content** | `content.share.cross_team`, `content.share.reframe_not_quote`, `content.epistemic_labels.required` | Controls information sharing across teams; requires epistemic labels on all outputs |

---

## 7. Test Plan

### Unit Tests

| Function | Test | Expected |
|---|---|---|
| `submitAction()` | Action from whitelisted source to allowed target | Verdict: ALLOW; action dispatched via Platform Integration Layer |
| `submitAction()` | Action targeting out-of-scope channel with no authorization | Verdict: REQUIRE_APPROVAL; approval request created |
| `submitAction()` | Action violating a hard DENY rule | Verdict: DENY; denial reason populated; action not dispatched |
| `submitAction()` | `dry_run: true` with approvable action | Returns verdict without dispatching; no approval request created; no side effects |
| `submitAction()` | `dry_run: true` with DENY rule | Returns DENY verdict with reason; no log entry in action audit log |
| `evaluateRules()` | Multiple matching rules — most specific wins | Binding rule is the narrowest-scope match; others are logged as evaluated-but-not-binding |
| `evaluateRules()` | Unless-condition blocks an otherwise-matching rule | Rule does not fire; unless-check result recorded in evaluation context |
| `evaluateRules()` | No rule matches | Default-deny behavior; `binding_rule` is null |
| `assembleContext()` | Agent queries MEETING_STATE artifact for active meeting | Correct MEETING_STATE artifact returned; state fields populated in evaluation context |
| `assembleContext()` | Authorization Store queried for active authorization | Matching AuthorizationRecord returned; `delegated: true` in context |
| `requestApproval()` | Approval request sent to correct admin DL | ApprovalRequest created; `logEvent()` called; action status set to PENDING |
| `processApproval()` | Admin approves with conditions | AuthorizationRecord written to Layer A; conditions stored; action proceeds with conditions applied |
| `processApproval()` | Non-admin attempts to approve | SELF_APPROVAL_ATTEMPT security alert fired; approval rejected |
| `processApproval()` | Approval expires before action executes | Authorization marked expired; action not allowed; new approval required |
| `validatePolicyRule()` | Valid rule structure | Returns valid: true |
| `validatePolicyRule()` | Rule references undefined entity (unknown group ref) | Returns valid: false with specific error message |

### Integration Tests

| Test | Approach |
|---|---|
| End-to-end: goal engine blocker → policy evaluation → platform dispatch | Goal Engine detects blocker; submits proposed `SendMessage`; Policy Engine evaluates; Outbound Gate dispatches to Platform Layer; verify message delivered |
| Approval workflow: out-of-scope message → request → approval → execute | Submit action targeting out-of-scope channel; verify approval request sent to admin DL; admin approves; verify original action executes |
| Authorization expiry: action approved at T=0 blocked at T=expiry+1 | Grant authorization with 60-second TTL; verify ALLOW at T=30; verify DENY at T=90 |
| MEETING_STATE context: large vs. small meeting rule | Meeting Engine writes MEETING_STATE with 15 participants; Policy Engine evaluates `speak` action; verify correct threshold-based rule fires |
| Multi-rule: most specific rule wins | Configure both a broad DENY and a narrow ALLOW for same action type; verify narrow ALLOW takes precedence |

### Security Tests

| Test | Approach |
|---|---|
| Self-approval blocked | Submitting module attempts to approve its own proposed action; verify blocked and SELF_APPROVAL_ATTEMPT alert fired |
| Rule bypass via dry-run | Verify `dry_run: true` never dispatches an action even if verdict is ALLOW |
| Admin DL poisoning: non-admin in admin DL | Attempt approval from user not in admin DL; verify denied with security event logged |
| Replay attack: reuse expired authorization | Attempt to reuse an expired `AuthorizationId`; verify denied |
