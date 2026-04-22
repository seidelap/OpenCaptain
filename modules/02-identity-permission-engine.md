# Module 2: Identity & Permission Engine

## 1. Purpose

Resolve user and group identities across all connected platforms into a unified identity model. Enforce the principle that OpenCaptain's access is always bounded by the **intersection** of (a) its technical OAuth/service-principal scopes and (b) the source platform's native ACLs. No action should succeed that the underlying platform would deny for the relevant user.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | `queryPlatformACL()`, `queryGroupMembership()` | The Identity Engine does not call platform APIs directly. It delegates all platform-native ACL queries and group membership lookups to the Platform Integration Layer's adapters. |
| **Knowledge Store (Module 4)** | `writeArtifact()` (Layer A) | ACL snapshots and identity resolution changes are written to Layer A as governance artifacts for temporal auditability ("who could see what, when?") |
| **Configuration & Admin Interface (Module 8)** | `getIdentityConfig()` | Manual identity mappings, cache SLA settings, heuristic matching toggles, and cross-platform join-key configuration come from the admin config |

### 2.2 Modules That Depend On This Module

| Consumer | What It Uses | Why |
|---|---|---|
| **Platform Integration Layer (Module 1)** | `resolveIdentity()`, `resolveGroup()` | Adapters call this module during event normalization to translate platform-specific user IDs into unified identities and resolve group membership |
| **Policy & Governance Engine (Module 3)** | `resolveIdentity()`, `resolveGroup()`, `canAccess()`, `getEffectiveScope()` | The Policy Engine's Context Assembler calls this module to resolve identities, determine group memberships (including whitelist checks via `resolveGroup()`) and verify effective permissions |
| **Meeting Participation Engine (Module 5)** | `resolveIdentity()`, `resolveGroup()` | Meeting engine resolves participant identities and checks meeting organizer/participant group memberships |
| **Team Goal Engine (Module 6)** | `resolveIdentity()`, `resolveGroup()` | Goal engine resolves task assignees and checks team membership via `resolveGroup(team_dl).members` |
| **Audit & Observability Layer (Module 7)** | `resolveIdentity()` | Audit layer resolves actor identities for display in compliance reports and dashboards |

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

#### Identity Resolution

```
resolveIdentity(ref: PlatformRef | EmailRef) -> UnifiedIdentity?
  Inputs:
    ref                 — either { platform: Platform, user_id: string }
                          or { email: string }
                          (email is the primary cross-platform join key)
  Returns:
    UnifiedIdentity { unified_id, display_name, email, platform_ids: Map<Platform, string>, org_unit?, role? }
    or null if no mapping exists
```

#### Group Resolution

```
resolveGroup(group_ref: UnifiedGroupRef) -> ResolvedGroup
  Inputs:
    group_ref           — unified group reference (platform-specific group ID, unified group ID,
                          team DL, or whitelist DL — all groups use the same interface)
  Returns:
    ResolvedGroup {
      group_id, display_name,
      members: UnifiedIdentity[],        // flat list, nested groups fully expanded
      nested_groups: UnifiedGroupRef[],
      platform_refs: Map<Platform, string>
    }

  Usage note: Callers check membership themselves: resolveGroup(whitelist_dl).members.contains(identity)
  Team and whitelist DLs are ordinary groups — no special-purpose methods needed.
```

#### Permission Checking

```
canAccess(agent_or_user: UnifiedIdentity, resource: ResourceRef, action: ActionType) -> PermissionResult
  Inputs:
    agent_or_user       — the identity to check (agent service principal or delegated user)
    resource            — platform-qualified resource (channel, meeting, ticket, etc.)
    action              — what action is being attempted (READ, WRITE, JOIN, SEND, etc.)
  Returns:
    PermissionResult { allowed: bool, scope: Scope, delegated: bool, reason: string? }

getEffectiveScope(agent: UnifiedIdentity, platform: Platform) -> EffectiveScope
  Inputs:
    agent               — the agent's unified identity
    platform            — which platform to check
  Returns:
    EffectiveScope { technical_scopes: string[], acl_constraints: string[], effective_permissions: string[] }
    — the intersection of OAuth scopes and platform ACLs
```

### 3.2 Required Interfaces (This Module Consumes)

#### From Platform Integration Layer (Module 1):

```
queryPlatformACL(platform: Platform, query: ACLQuery) -> ACLResult
queryGroupMembership(platform: Platform, group_ref: string) -> GroupMembershipResult
```

#### From Knowledge Store (Module 4):

```
writeArtifact(artifact: LayerAArtifact) -> ArtifactId
  — Write ACL snapshots and identity resolution audit records to Layer A
```

#### From Configuration & Admin Interface (Module 8):

```
getIdentityConfig() -> IdentityConfig
  — Returns manual identity mappings, cache SLA, heuristic matching settings

onConfigChanged(callback: (change: ConfigChange) -> void) -> SubscriptionHandle
  — Subscribe to config changes (e.g., new manual mapping added)
```

---

## 4. Core Functions

### 4.1 `resolveIdentity(platform: Platform, platform_user_id: string) -> UnifiedIdentity?`

**Inputs:**
- `platform` — source platform enum
- `platform_user_id` — native platform user ID

**What it does:**
Looks up the unified identity registry for an existing mapping from this platform+user_id pair. If found, returns the cached unified identity. If not found, initiates the three-tier resolution strategy:
1. **Primary:** Query the platform adapter for the user's email address; look up the email in the registry as a cross-platform join key
2. **Secondary:** Check admin-configured manual mappings (from Module 8 config)
3. **Tertiary:** Heuristic matching on display name + org unit; if a candidate match is found, it is flagged for human confirmation before being committed

New resolutions are written to the registry and an identity-resolution audit record is written to Layer A via the Knowledge Store.

### 4.2 `resolveGroup(group_ref: UnifiedGroupRef) -> ResolvedGroup`

**Inputs:**
- `group_ref` — group reference (may be platform-native or unified)

**What it does:**
Checks the group membership cache for a valid (non-expired) entry. If the cache is stale or missing, delegates to `queryGroupMembership()` on the Platform Integration Layer for the relevant platform(s). Recursively resolves nested groups until all leaf members are found. Resolves each platform member ID to a unified identity via `resolveIdentity()`. Returns the fully resolved group with both direct members and expanded nested members. Updates the cache with the new membership and records the cache refresh timestamp.

### 4.3 `canAccess(agent_or_user: UnifiedIdentity, resource: ResourceRef, action: ActionType) -> PermissionResult`

**Inputs:**
- `agent_or_user` — identity to check
- `resource` — platform-qualified resource reference
- `action` — the action type being attempted

**What it does:**
Determines the permission mode: **delegated** (act-as-user) or **app-level** (broad service principal). If delegated mode is configured (preferred), checks whether the delegating user has access to the resource on the source platform by calling `queryPlatformACL()` via the Platform Integration Layer. The agent's effective permission is the **intersection** of its technical OAuth scopes and the delegating user's platform ACL. If app-level mode is used (fallback only), checks the agent's service principal permissions. Returns the verdict with an explanation of which constraint was binding.

### 4.4 `getEffectiveScope(agent: UnifiedIdentity, platform: Platform) -> EffectiveScope`

**Inputs:**
- `agent` — the agent's unified identity
- `platform` — target platform

**What it does:**
Retrieves the agent's registered OAuth scopes for the given platform from config. Queries the platform's permission API for actual ACL constraints on the agent's service principal. Computes the intersection: the set of actions the agent can technically perform AND that the platform ACL allows. Returns the effective scope, which the Policy Engine uses as an outer bound on what actions can even be proposed.

### 4.5 `snapshotACLState(scope: SnapshotScope) -> ACLSnapshot`

**Inputs:**
- `scope` — which teams/channels/resources to snapshot (can be "all whitelisted" or a specific set)

**What it does:**
Captures a point-in-time snapshot of who-can-see-what across relevant platforms. Queries group memberships, channel memberships, and resource permissions for all entities in scope. Writes the snapshot as an immutable Layer A artifact in the Knowledge Store with a `valid_from` timestamp. These snapshots enable temporal permission queries: "Could user X see resource Y at time T?" Used by the Audit Layer for compliance reporting.

### 4.6 `invalidateGroupCache(platform: Platform, group_ref: string) -> void`

**Inputs:**
- `platform` — platform where the change occurred
- `group_ref` — the group whose membership changed

**What it does:**
Marks the cached membership for this group as stale. Triggers an async re-resolution of the group membership. Propagates the invalidation to any parent groups that contain this group as a nested member. Notifies the Policy Engine that any cached policy evaluations involving this group should be re-evaluated. Ensures cache is updated within the configurable SLA (e.g., 5 minutes).

---

## 5. Data Model

### Unified Identity Registry

```
UnifiedIdentity {
  unified_id: string (UUID)
  display_name: string
  email: string (primary cross-platform join key)
  org_unit: string?
  role: string?
  platform_ids: Map<Platform, string>
  resolution_method: PRIMARY_EMAIL | MANUAL_MAPPING | HEURISTIC_CONFIRMED
  created_at: timestamp
  last_verified_at: timestamp
}
```

### Group Membership Cache

```
CachedGroupMembership {
  group_ref: UnifiedGroupRef
  platform: Platform
  members: UnifiedIdentity[]
  nested_groups: UnifiedGroupRef[]
  fetched_at: timestamp
  expires_at: timestamp (fetched_at + configurable SLA)
  source_platform_ref: string
}
```

### Identity Resolution Strategy

| Priority | Method | Confidence | Human Confirmation Required |
|---|---|---|---|
| 1 | Email address match | High | No |
| 2 | Admin-configured manual mapping | High | No (pre-confirmed by admin) |
| 3 | Display name + org unit heuristic | Low | Yes — flagged for confirmation |
