# Module 7: Audit & Observability Layer

## 1. Purpose

Ensure that every action the agent takes, every policy decision, every approval, and every knowledge operation is logged immutably for compliance, debugging, and organizational trust. This layer **spans all other modules** — it is not a downstream consumer that only sees final outputs, but a cross-cutting concern that captures the full decision chain from event intake through policy evaluation to action dispatch. It also provides dashboards, alerts, and compliance reports that give administrators and stakeholders visibility into agent behavior.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | Event/action metadata, `getAdapterHealth()` | Logs every inbound event and outbound action with platform-level metadata (delivery status, latency, errors). Monitors adapter health for dashboard display. |
| **Identity & Permission Engine (Module 2)** | `resolveIdentity()` | Resolves actor identities (user IDs, agent IDs) into display names for compliance reports and dashboards |
| **Policy & Governance Engine (Module 3)** | Policy decision records (received as structured logs after every evaluation) | Every policy evaluation (ALLOW, DENY, REQUIRE_APPROVAL) with full context is logged: which rules were evaluated, which was binding, what the context was |
| **Knowledge Store (Module 4)** | `queryArtifacts()`, `getProvenance()`, `getContradictions()` | Queries Layer A artifacts for compliance reports (e.g., "what information crossed team boundaries?"), traces provenance chains for audit investigations, queries open contradictions for the CONTRADICTION_STATUS compliance report |
| **Meeting Participation Engine (Module 5)** | Meeting participation telemetry (via `getMeetingState()`, `getActiveMeetings()`) | Logs meeting join/leave, contributions made/suppressed, state transitions, and contribution latency |
| **Team Goal Engine (Module 6)** | Goal engine telemetry (via `getTeamGoalState()`) | Logs detected blockers, proposed actions, action verdicts, and team work tracking state |
| **Configuration & Admin Interface (Module 8)** | `getAuditConfig()`, config change events | Log retention policies, alert thresholds, dashboard configuration. Also logs all configuration changes as audit events. |

### 2.2 Modules That Depend On This Module

| Consumer | What It Uses | Why |
|---|---|---|
| **Configuration & Admin Interface (Module 8)** | `queryAuditLog()`, `getComplianceReport()`, dashboard data | The Admin Interface surfaces audit data, dashboards, and compliance reports to administrators |

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

#### Audit Logging (consumed by all modules)

```
logEvent(entry: AuditLogEntry) -> AuditLogId
  Inputs:
    entry               — structured audit record (see schema below)
  Returns:
    Immutable audit log entry ID

  Behavior: Append-only. Entries cannot be modified or deleted after creation.
  The entry is durably persisted before the function returns.
```

This is the primary write interface. Every module calls `logEvent()` to record significant actions, decisions, and state changes.

#### Audit Query

```
queryAuditLog(query: AuditQuery) -> AuditLogEntry[]
  Inputs:
    query               — filter by: time_range, actor_id, action_type, target_id,
                          policy_decision, approval_ref, module_source, severity
  Returns:
    Matching audit entries, ordered by timestamp

queryPolicyDecisionLog(query: PolicyDecisionQuery) -> PolicyDecisionEntry[]
  Inputs:
    query               — filter by: time_range, proposed_action_type, verdict,
                          binding_rule_id, target_id, source_module
  Returns:
    Policy decision records with full evaluation context
```

#### Compliance Reporting

```
getComplianceReport(report_type: ReportType, time_range: TimeRange, scope: ReportScope?) -> ComplianceReport
  Inputs:
    report_type         — ACTIVITY_SUMMARY | APPROVAL_SUMMARY | BOUNDARY_CROSSING |
                          CONTRADICTION_STATUS | FULL_AUDIT
    time_range          — start/end timestamps
    scope               — optional: filter by team, channel, or user
  Returns:
    ComplianceReport { report_type, generated_at, time_range, scope, sections: ReportSection[] }
```

Report types:
- **ACTIVITY_SUMMARY**: "What did the agent do last week?" — all actions taken, grouped by type and target
- **APPROVAL_SUMMARY**: "What approvals were granted/denied?" — full approval lifecycle with outcomes
- **BOUNDARY_CROSSING**: "What information crossed team boundaries?" — all out-of-scope communications with approval chains
- **CONTRADICTION_STATUS**: "What contradictions remain unresolved?" — open contradictions in the B graph with age and scope
- **FULL_AUDIT**: Complete action-by-action log with policy decisions and provenance chains

#### Dashboards & Alerts

```
getDashboardData(dashboard: DashboardType) -> DashboardData
  Inputs:
    dashboard           — AGENT_ACTIVITY | APPROVAL_RATES | BOUNDARY_CROSSINGS |
                          KNOWLEDGE_HEALTH | ANOMALY_DETECTION | ADAPTER_HEALTH
  Returns:
    DashboardData { dashboard, updated_at, metrics: Metric[], charts: ChartData[] }

subscribeToAlerts(alert_types: AlertType[], callback: (alert: Alert) -> void) -> SubscriptionHandle
  Inputs:
    alert_types         — which alerts to subscribe to (ANOMALY, HIGH_DENIAL_RATE,
                          ADAPTER_DOWN, EXTRACTION_LAG, APPROVAL_BACKLOG, etc.)
    callback            — handler for alert notifications
  Returns:
    SubscriptionHandle
```

### 3.2 Required Interfaces (This Module Consumes)

#### From Platform Integration Layer (Module 1):

```
getAdapterHealth(platform: Platform) -> AdapterHealth
```

#### From Identity & Permission Engine (Module 2):

```
resolveIdentity(platform: Platform, platform_user_id: string) -> UnifiedIdentity?
```

#### From Policy & Governance Engine (Module 3):

*Policy decisions are received as structured log entries emitted by the Policy Engine after every evaluation — no explicit pull interface. The Policy Engine calls `logEvent()` on this module.*

#### From Knowledge Store (Module 4):

```
queryArtifacts(query: ArtifactQuery) -> LayerAArtifact[]
getProvenance(artifact_id: ArtifactId) -> ProvenanceGraph
getContradictions(scope: ClaimScope?, topic: string?) -> ConflictPair[]
```

#### From Meeting Participation Engine (Module 5):

```
getMeetingState(meeting_id: string) -> MeetingState
getActiveMeetings() -> MeetingState[]
```

#### From Team Goal Engine (Module 6):

```
getTeamGoalState(team_dl: UnifiedGroupRef) -> TeamGoalState
```

#### From Configuration & Admin Interface (Module 8):

```
getAuditConfig() -> AuditConfig
  — Returns: log_retention_days, alert_thresholds, dashboard_refresh_interval,
    compliance_report_schedule, anomaly_detection_sensitivity

onConfigChanged(callback: (change: ConfigChange) -> void) -> void
  — Subscribe to config changes; logs each change as an audit event
```

---

## 4. Core Functions

### 4.1 `logEvent(entry: AuditLogEntry) -> AuditLogId`

**Inputs:**
- `entry` — structured audit record with required fields (see schema below)

**What it does:**
Validates that all required fields are present (timestamp, actor, action, target, module_source). Assigns an immutable audit log ID. Persists the entry to the append-only audit store. The store rejects any attempt to modify or delete existing entries. Indexes the entry by actor, action type, target, timestamp, and policy decision for efficient querying. If the entry represents a significant event (boundary crossing, denial, approval), checks alert rules and fires alerts if thresholds are met. Returns the audit log ID for cross-referencing.

### 4.2 `logPolicyDecision(decision: PolicyDecisionEntry) -> void`

**Inputs:**
- `decision` — full policy evaluation record from the Policy Engine

**What it does:**
Persists the complete policy evaluation record: proposed action, assembled context snapshot, every rule evaluated (with match/no-match result), the binding rule, the final verdict, and the approval target (if REQUIRE_APPROVAL). This is separate from the general audit log because it captures the **reasoning** behind each decision, not just the outcome. Enables compliance queries like "why was this action denied?" or "what rules were evaluated when the agent sent this message?"

### 4.3 `generateComplianceReport(report_type: ReportType, time_range: TimeRange, scope: ReportScope?) -> ComplianceReport`

**Inputs:**
- `report_type` — which type of report to generate
- `time_range` — reporting period
- `scope` — optional filter

**What it does:**
Queries the audit log, policy decision log, and Knowledge Store (Layer A) for relevant records within the time range and scope. Aggregates data into the appropriate report format:

- **ACTIVITY_SUMMARY**: Groups actions by type (messages sent, tickets created, meetings attended) and by target (teams, channels, individuals). Includes counts, timestamps, and policy verdicts.
- **APPROVAL_SUMMARY**: Lists every approval request, response, outcome, and scope. Tracks approval-to-action latency and approval expiration rates.
- **BOUNDARY_CROSSING**: Identifies every communication that crossed a whitelist boundary. Shows the authorization chain: who approved, when, with what scope, and whether admin was CC'd.
- **CONTRADICTION_STATUS**: Queries the Knowledge Store for open contradictions. Reports age, scope, involved parties, and whether any resolution has been attempted.
- **FULL_AUDIT**: Complete chronological log of every action, decision, and knowledge operation with cross-references.

### 4.4 `detectAnomalies(window: TimeRange) -> Anomaly[]`

**Inputs:**
- `window` — time window to analyze

**What it does:**
Analyzes audit log patterns within the window to detect unusual agent behavior:
- Sudden spike in boundary crossing requests
- Unusual hours of activity
- High rate of policy denials (may indicate misconfiguration or adversarial input)
- Approval backlog growing (admins not responding)
- Extraction pipeline lag (claims not being extracted from Layer A)
- Repeated failed actions to the same target
Compares metrics against configurable baselines and sensitivity thresholds. Fires alerts for detected anomalies.

### 4.5 `traceAction(audit_log_id: AuditLogId) -> ActionTrace`

**Inputs:**
- `audit_log_id` — starting audit entry to trace

**What it does:**
Builds a complete trace of a single action from inception to outcome. Follows cross-references through: the originating event (Layer A artifact), the policy decision record (rules evaluated, verdict), the approval chain (if applicable), the Knowledge Store references that informed the action, and the final dispatch result. Returns a unified `ActionTrace` that shows the full decision chain — useful for "why did the agent do X?" investigations.

---

## 5. Data Model

### AuditLogEntry

```
AuditLogEntry {
  audit_log_id: string (UUID, immutable)
  timestamp: timestamp
  actor: ActorRef                          // agent, user, or system
  action: string                           // action type (SEND_MESSAGE, EVALUATE_POLICY, CREATE_TICKET, etc.)
  target: TargetRef                        // channel, user, ticket, meeting, etc.
  module_source: ModuleId                  // which module generated this entry
  policy_decision: ALLOW | DENY | REQUIRE_APPROVAL | N/A
  approval_ref: AuthorizationId?           // if action was authorized by an approval
  knowledge_refs: ArtifactId[]             // Layer A artifacts related to this action
  outcome: SUCCESS | FAILED | PENDING | SUPPRESSED
  metadata: Map<string, any>               // module-specific additional context
  severity: INFO | WARN | ERROR            // for alert filtering
}
```

### PolicyDecisionEntry

```
PolicyDecisionEntry {
  decision_id: string (UUID)
  timestamp: timestamp
  proposed_action: ProposedAction
  context_snapshot: EvaluationContext       // full context at time of evaluation
  rules_evaluated: RuleEvaluation[]        // each rule, match/no-match, unless-check results
  binding_rule: RuleId?
  verdict: ALLOW | DENY | REQUIRE_APPROVAL
  denial_reason: string?
  approval_target: UnifiedGroupRef?
  source_module: ModuleId
}
```

### Alert Types

| Alert Type | Trigger | Default Severity |
|---|---|---|
| `ANOMALY` | Unusual activity pattern detected | WARN |
| `HIGH_DENIAL_RATE` | > N% of proposed actions denied in window | WARN |
| `ADAPTER_DOWN` | Platform adapter health = DOWN | ERROR |
| `EXTRACTION_LAG` | Claim extraction pipeline falling behind | WARN |
| `APPROVAL_BACKLOG` | > N pending approvals older than threshold | WARN |
| `BOUNDARY_CROSSING_SPIKE` | Unusual number of out-of-scope communications | WARN |
| `SELF_APPROVAL_ATTEMPT` | Agent or non-admin attempted to approve an action | ERROR |
| `SECURITY_EVENT` | Privilege escalation attempt, impersonation, etc. | ERROR |
