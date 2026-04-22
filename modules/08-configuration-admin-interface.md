# Module 8: Configuration & Admin Interface

## 1. Purpose

Allow administrators to author policies, manage whitelists, tune thresholds, and monitor agent behavior without requiring code changes. This module is the **single source of truth for all runtime configuration** — every other module reads its configuration from here. It provides multiple interface surfaces (Web UI, chat-based commands, configuration-as-code) and ensures that all configuration changes are validated, versioned, and audited.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Policy & Governance Engine (Module 3)** | `validatePolicyRule()`, `submitAction(dry_run: true)` | Validates new policy rules and tests them against hypothetical actions before persisting |
| **Audit & Observability Layer (Module 7)** | `queryAuditLog()`, `getComplianceReport()`, `getDashboardData()`, `subscribeToAlerts()` | Surfaces audit data, compliance reports, dashboards, and real-time alerts to administrators |

### 2.2 Modules That Depend On This Module

| Consumer | What It Uses | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | `getPlatformConfig()`, `getAdapterSettings()`, `onConfigChanged()` | Adapters read OAuth credentials, enabled/disabled status, rate limits, and event subscriptions from config |
| **Identity & Permission Engine (Module 2)** | `getIdentityConfig()`, `onConfigChanged()` | Identity engine reads manual identity mappings, cache SLA settings, and heuristic matching configuration |
| **Policy & Governance Engine (Module 3)** | `getPolicyRules()`, `getThresholds()`, `getWhitelists()`, `getAdminDLs()`, `onConfigChanged()` | Policy engine reads all policy rules, thresholds, whitelists, and admin DL definitions |
| **Knowledge Store (Module 4)** | `getKnowledgeConfig()`, `onConfigChanged()` | Knowledge Store reads extraction pipeline parameters, Layer C TTLs, invalidation sensitivity, retention policies |
| **Meeting Participation Engine (Module 5)** | `getMeetingConfig()`, `onConfigChanged()` | Meeting engine reads small meeting threshold, relevance thresholds, devil's advocate toggle, rate limits |
| **Team Goal Engine (Module 6)** | `getGoalEngineConfig()`, `onConfigChanged()` | Goal engine reads team scope, blocker sensitivity, sync suggestion thresholds |
| **Audit & Observability Layer (Module 7)** | `getAuditConfig()`, `onConfigChanged()` | Audit layer reads retention policies, alert thresholds, dashboard configuration |

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

#### Platform Configuration

```
getPlatformConfig(platform: Platform) -> PlatformConfig
  Inputs:
    platform            — which platform's config to retrieve
  Returns:
    PlatformConfig {
      platform, enabled: bool,
      oauth_client_id: string, oauth_client_secret: string (encrypted),
      scopes: string[], webhook_url: string,
      rate_limit_overrides: RateLimitConfig?,
      custom_field_mappings: FieldMapping[]
    }

getAdapterSettings(platform: Platform) -> AdapterSettings
  Inputs:
    platform            — which platform
  Returns:
    AdapterSettings { enabled: bool, feature_flags: Map<string, bool>, event_subscriptions: EventType[] }
```

#### Identity Configuration

```
getIdentityConfig() -> IdentityConfig
  Returns:
    IdentityConfig {
      manual_mappings: ManualIdentityMapping[],
      cache_sla_seconds: number,
      heuristic_matching_enabled: bool,
      cross_platform_join_key: "email" | "custom",
      acl_snapshot_interval_seconds: number
    }
```

#### Policy Configuration

```
getPolicyRules() -> PolicyRule[]
  Returns:
    All active policy rules in declarative format (YAML/JSON parsed into objects)

getThresholds() -> ThresholdConfig
  Returns:
    ThresholdConfig {
      small_meeting_threshold: number (default 2),
      approval_expiry_default_hours: number,
      contribution_relevance_threshold: float,
      hand_raise_rate_limit_per_meeting: number,
      max_speak_duration_seconds: number
    }

getWhitelists() -> WhitelistConfig
  Returns:
    WhitelistConfig {
      team_dls: UnifiedGroupRef[],          // whitelisted team distribution lists
      allowed_channels: ChannelRef[],       // specific channels the agent can operate in
      platform_scopes: Map<Platform, string[]>
    }

getAdminDLs() -> AdminDLConfig
  Returns:
    AdminDLConfig {
      primary_admin_dl: UnifiedGroupRef,    // main admin group for approvals
      per_team_admin_dls: Map<UnifiedGroupRef, UnifiedGroupRef>,  // team DL -> admin DL
      escalation_chain: UnifiedGroupRef[]
    }
```

#### Knowledge Store Configuration

```
getKnowledgeConfig() -> KnowledgeConfig
  Returns:
    KnowledgeConfig {
      extraction_pipeline: ExtractionConfig,  // model, confidence threshold, human_review_types
      layer_c_ttl_seconds: number,
      invalidation_sensitivity: LOW | MEDIUM | HIGH,
      graph_partition_strategy: "per_team" | "per_project" | "per_time_window",
      retention_policy: RetentionPolicy       // days to retain, archival rules, GDPR handling
    }
```

#### Meeting Configuration

```
getMeetingConfig() -> MeetingConfig
  Returns:
    MeetingConfig {
      small_meeting_threshold: number,
      contribution_relevance_threshold: float,
      devils_advocate_enabled: bool,
      hand_raise_rate_limit: number,
      max_speak_duration_seconds: number,
      cross_meeting_context_enabled: bool,
      anonymize_cross_meeting_sources: bool
    }
```

#### Goal Engine Configuration

```
getGoalEngineConfig() -> GoalEngineConfig
  Returns:
    GoalEngineConfig {
      team_dl: UnifiedGroupRef,
      blocker_detection_sensitivity: LOW | MEDIUM | HIGH,
      sync_suggestion_threshold: float,
      work_item_refresh_interval_seconds: number,
      external_dependency_alert_enabled: bool
    }
```

#### Audit Configuration

```
getAuditConfig() -> AuditConfig
  Returns:
    AuditConfig {
      log_retention_days: number,
      alert_thresholds: AlertThresholdConfig,
      dashboard_refresh_interval_seconds: number,
      compliance_report_schedule: CronExpression?,
      anomaly_detection_sensitivity: LOW | MEDIUM | HIGH
    }
```

#### Configuration Change Subscription

```
onConfigChanged(callback: (change: ConfigChange) -> void) -> SubscriptionHandle
  Inputs:
    callback            — handler for configuration changes
  Returns:
    SubscriptionHandle

  ConfigChange {
    config_section: string,                // e.g., "policy_rules", "whitelists", "meeting_config"
    change_type: CREATED | UPDATED | DELETED,
    changed_by: UnifiedIdentity,
    old_value: any?,
    new_value: any,
    timestamp: timestamp
  }
```

All modules subscribe to `onConfigChanged()` so they can react to live configuration updates without restart.

### 3.2 Required Interfaces (This Module Consumes)

#### From Policy & Governance Engine (Module 3):

```
validatePolicyRule(rule: PolicyRule) -> ValidationResult
  — Validates rule syntax, semantics, and checks for conflicts with existing rules

submitAction(proposed_action: ProposedAction, source_module: ADMIN, dry_run: true) -> ActionOutcome
  — Used to test policy rule changes: run a hypothetical action through the policy engine
    without dispatching it, to verify the new rules produce the expected verdicts
```

#### From Audit & Observability Layer (Module 7):

```
queryAuditLog(query: AuditQuery) -> AuditLogEntry[]
getComplianceReport(report_type: ReportType, time_range: TimeRange, scope: ReportScope?) -> ComplianceReport
getDashboardData(dashboard: DashboardType) -> DashboardData
subscribeToAlerts(alert_types: AlertType[], callback: (alert: Alert) -> void) -> SubscriptionHandle
  — Admin UI subscribes to real-time alerts for display in the dashboard
logEvent(entry: AuditLogEntry) -> AuditLogId
  — All config changes are logged as audit events
```

---

## 4. Core Functions

### 4.1 `updatePolicyRule(rule: PolicyRule, changed_by: UnifiedIdentity) -> UpdateResult`

**Inputs:**
- `rule` — the policy rule to create or update (declarative YAML/JSON format)
- `changed_by` — the admin making the change

**What it does:**
Validates the admin's identity and authorization to modify policies (must be a member of an admin DL). Calls the Policy Engine's `validatePolicyRule()` to check syntax, semantics, and conflicts with existing rules. If validation fails: returns clear error messages (malformed syntax, conflicting rules, invalid references). If validation passes: persists the rule to the versioned policy rule store. Creates a new version of the rule (old version is retained for audit). Emits a `ConfigChange` event to all subscribers so the Policy Engine picks up the new rule. Logs the change to the Audit Layer with who, what, and when.

### 4.2 `updateWhitelist(change: WhitelistChange, changed_by: UnifiedIdentity) -> UpdateResult`

**Inputs:**
- `change` — add/remove team DL, channel, or platform scope
- `changed_by` — the admin making the change

**What it does:**
Validates admin authorization. Applies the whitelist change to the config store. For additions: the new group/channel becomes immediately available to the Policy Engine's boundary checks. For removals: existing active authorizations that depend on the removed whitelist entry are flagged for review (not automatically revoked — admin is warned). Emits `ConfigChange` event. Logs to Audit Layer.

### 4.3 `updateThreshold(threshold: string, value: any, changed_by: UnifiedIdentity) -> UpdateResult`

**Inputs:**
- `threshold` — which threshold to update (e.g., "small_meeting_threshold", "approval_expiry_default_hours")
- `value` — the new value
- `changed_by` — the admin

**What it does:**
Validates admin authorization. Validates the value against the threshold's constraints (type, range, allowed values). Persists the change. Emits `ConfigChange` event. The consuming module receives the change and immediately starts using the new value. Logs to Audit Layer.

### 4.4 `updatePlatformCredentials(platform: Platform, credentials: EncryptedCredentials, changed_by: UnifiedIdentity) -> UpdateResult`

**Inputs:**
- `platform` — which platform
- `credentials` — encrypted OAuth credentials
- `changed_by` — the admin

**What it does:**
Validates admin authorization. Encrypts credentials at rest (if not already encrypted). Stores in the secure credential store (separate from general config for security isolation). Triggers the Platform Integration Layer adapter to re-initialize with the new credentials. Logs the credential rotation event to Audit Layer (logs the fact of rotation, never the credential values).

### 4.5 `importConfigFromCode(config_path: string, changed_by: UnifiedIdentity) -> ImportResult`

**Inputs:**
- `config_path` — path to a git-managed configuration file (YAML/JSON)
- `changed_by` — the admin or CI/CD pipeline identity

**What it does:**
Supports the configuration-as-code interface. Reads the config file, parses it, and validates each section:
- Policy rules: validated via the Policy Engine
- Whitelists: validated against known group references
- Thresholds: validated against constraints
- Platform settings: validated for required fields

If all validations pass: applies the full configuration atomically (all-or-nothing). If any validation fails: rejects the entire import with specific error locations. Emits `ConfigChange` events for each changed section. Logs the import to Audit Layer.

### 4.6 `handleChatConfigCommand(command: OpenCaptainEvent) -> void`

**Inputs:**
- `command` — a `MessageReceived` event in the admin DL channel that is detected as a configuration command

**What it does:**
Supports the chat-based configuration interface. Parses the admin's chat message for config commands (e.g., "set small meeting threshold to 3", "add #finance-team to whitelist"). Validates admin authorization (must be in admin DL). Executes the appropriate update function (`updateThreshold`, `updateWhitelist`, etc.). Responds in the channel with confirmation or error message. The config command itself is subject to policy evaluation (chat-based config changes go through the Policy Engine like any other action — but config commands from admin DL are ALLOW by default).

---

## 5. Configuration Surface Summary

| Category | Settings | Default Interface |
|---|---|---|
| **Whitelist management** | Team DLs, admin DLs, per-agent scope, allowed channels | Web UI, Chat, Config-as-code |
| **Policy authoring** | Declarative rules (YAML/JSON with validation) | Config-as-code, Web UI |
| **Threshold tuning** | Small meeting threshold, approval expiry, relevance thresholds | Web UI, Chat |
| **Feature toggles** | Devil's advocate mode, cross-meeting context, auto-extraction, anonymization | Web UI, Chat |
| **Per-platform settings** | Active platforms, auth credentials, scope configuration | Web UI, Config-as-code |
| **Knowledge store settings** | Extraction parameters, Layer C TTLs, invalidation sensitivity, retention | Config-as-code, Web UI |
| **Audit settings** | Log retention, alert thresholds, report schedules, anomaly sensitivity | Config-as-code, Web UI |

---

## 6. Security Model

- **Admin-only writes:** Only members of admin DLs can modify configuration. Non-admin attempts are rejected and logged as security events.
- **Credential isolation:** OAuth credentials are stored in a separate encrypted credential store, never in the general config store.
- **Change auditing:** Every configuration change is logged immutably to the Audit Layer with who, what, when, and the old/new values.
- **Versioned config:** All configuration is versioned. Previous versions are retained for rollback and audit.
- **Atomic imports:** Config-as-code imports are all-or-nothing — partial application of a config file is never allowed.
