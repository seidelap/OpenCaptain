# Module 1: Platform Integration Layer

## 1. Purpose

Provide normalized, bidirectional interfaces to each supported platform (Slack, Teams, Zoom, Google Workspace, Jira, Asana) so that every other module in OpenCaptain operates on a unified event/action model rather than platform-specific APIs. This module is the only component that speaks native platform protocols; everything above it sees a single, platform-agnostic vocabulary of events and actions.

---

## 2. Module Dependencies

### 2.1 Modules This Module Depends On

| Dependency | What It Needs | Why |
|---|---|---|
| **Identity & Permission Engine (Module 2)** | `resolveIdentity()`, `getGroupMembers()` | When translating platform events, the adapter needs to resolve platform-specific user IDs (e.g., Slack `U01ABC`) into unified OpenCaptain identity references so downstream consumers get consistent identities |
| **Configuration & Admin Interface (Module 8)** | `getPlatformConfig()`, `getAdapterSettings()` | Each adapter needs its OAuth credentials, enabled/disabled status, rate limit overrides, and per-platform settings from the central config store |

### 2.2 Modules That Depend On This Module

| Consumer | What It Uses | Why |
|---|---|---|
| **Identity & Permission Engine (Module 2)** | `queryPlatformACL()`, `queryGroupMembership()` | The Identity Engine delegates platform-native ACL and group membership queries back to the adapters, since only the adapter knows how to call the platform's permission APIs |
| **Policy & Governance Engine (Module 3)** | `dispatchAction()` | The Policy Engine's Outbound Gate dispatches ALLOW-verdicted actions through this layer for delivery to the target platform |
| **Knowledge Store (Module 4)** | Inbound `OpenCaptainEvent` stream | All platform events (messages, meetings, ticket updates) flow into the Knowledge Store's Layer A as raw source artifacts |
| **Meeting Participation Engine (Module 5)** | Inbound meeting events, `dispatchAction()` for hand-raise/speak | Meeting engine receives `MeetingStarted`, `HandRaiseDetected`, `CalledUpon` events and sends `RaiseHand`, `Speak` actions back through this layer |
| **Team Goal Engine (Module 6)** | Inbound ticket/task events | Goal engine receives `TicketUpdated` events to track work item progress |
| **Audit & Observability Layer (Module 7)** | Event/action metadata | Audit layer logs every inbound event and outbound action with platform-level metadata (timestamps, platform IDs, delivery status) |

---

## 3. Interfaces

### 3.1 Provided Interfaces (This Module Exposes)

#### `EventBus` — Normalized Inbound Event Stream

```
subscribe(event_types: EventType[], callback: (event: OpenCaptainEvent) -> void) -> SubscriptionHandle
unsubscribe(handle: SubscriptionHandle) -> void
```

All downstream modules subscribe to this bus to receive platform events in normalized form.

#### `ActionDispatcher` — Normalized Outbound Action Execution

```
dispatchAction(action: OpenCaptainAction) -> ActionResult
  Inputs:
    action: OpenCaptainAction  — normalized action (SendMessage, RaiseHand, Speak, CreateTicket, etc.)
  Returns:
    ActionResult { status: DELIVERED | FAILED | RATE_LIMITED, platform_ref: string?, error: string? }
```

Called by the Policy Engine's Outbound Gate after an action is approved. Routes to the correct platform adapter based on the `platform` field in the action.

#### `PlatformACLQuery` — Platform-Native Permission Queries

```
queryPlatformACL(platform: Platform, query: ACLQuery) -> ACLResult
  Inputs:
    platform: Platform          — which platform to query
    query: ACLQuery             — { subject_id, resource_id, action_type }
  Returns:
    ACLResult { allowed: bool, scope: string[], reason: string? }

queryGroupMembership(platform: Platform, group_ref: string) -> GroupMembershipResult
  Inputs:
    platform: Platform          — which platform to query
    group_ref: string           — platform-native group/DL/channel identifier
  Returns:
    GroupMembershipResult { members: PlatformUserId[], nested_groups: string[] }
```

Used by the Identity & Permission Engine to delegate ACL and group lookups to the source platform.

#### `PlatformHealthStatus` — Adapter Health for Observability

```
getAdapterHealth(platform: Platform) -> AdapterHealth
  Returns:
    AdapterHealth { status: HEALTHY | DEGRADED | DOWN, latency_ms: number, last_event_at: timestamp, error_count_1h: number }
```

### 3.2 Required Interfaces (This Module Consumes)

#### From Identity & Permission Engine (Module 2):

```
resolveIdentity(platform: Platform, platform_user_id: string) -> UnifiedIdentity?
  — Called during event normalization to attach unified identity to sender/participant fields

getGroupMembers(group_ref: UnifiedGroupRef) -> UnifiedIdentity[]
  — Called when adapter needs unified identity list for a platform group
```

#### From Configuration & Admin Interface (Module 8):

```
getPlatformConfig(platform: Platform) -> PlatformConfig
  — Returns OAuth credentials, scopes, webhook URLs, rate limit settings

getAdapterSettings(platform: Platform) -> AdapterSettings
  — Returns enabled/disabled, feature flags, custom field mappings

onConfigChanged(callback: (platform: Platform, change: ConfigChange) -> void) -> SubscriptionHandle
  — Subscribe to live config changes (e.g., admin disables a platform)
```

---

## 4. Core Functions

### 4.1 `normalizeEvent(platformEvent: RawPlatformEvent) -> OpenCaptainEvent`

**Inputs:**
- `platformEvent` — raw event payload from a platform webhook or subscription (e.g., Slack `message` event, Zoom `meeting.participant_joined` webhook)

**What it does:**
Translates a platform-specific event into the normalized `OpenCaptainEvent` schema. Resolves platform user IDs to unified identities via the Identity Engine. Maps platform-specific fields (e.g., Slack `ts` to `timestamp`, Teams `from.user.id` to `sender_id`). Attaches platform provenance metadata (platform name, raw event ID, received timestamp). Publishes the normalized event to the EventBus.

### 4.2 `translateAction(action: OpenCaptainAction) -> PlatformAPICall`

**Inputs:**
- `action` — normalized OpenCaptain action (e.g., `SendMessage`, `CreateTicket`, `RaiseHand`)

**What it does:**
Converts a normalized action into the platform-specific API call. Maps unified identity references back to platform-native user IDs. Transforms content format (e.g., Markdown to Slack Block Kit, or to Teams Adaptive Card). Constructs the full API request including auth headers, endpoint URL, and payload.

### 4.3 `executeWithRetry(apiCall: PlatformAPICall) -> ActionResult`

**Inputs:**
- `apiCall` — fully constructed platform API call from `translateAction`

**What it does:**
Executes the API call against the target platform. Implements per-platform rate limiting (respects `Retry-After` headers, token bucket per adapter). Retries transient failures (5xx, network errors) with exponential backoff up to a configurable max. Returns `ActionResult` with delivery status and platform-specific reference (e.g., Slack message `ts`, Jira issue key). Emits action execution metrics to the Audit Layer.

### 4.4 `refreshOAuthToken(platform: Platform, credential: OAuthCredential) -> OAuthCredential`

**Inputs:**
- `platform` — which platform's token to refresh
- `credential` — current OAuth credential including refresh token

**What it does:**
Checks if the current access token is expired or nearing expiry. Calls the platform's OAuth token endpoint with the refresh token. Stores the new access token and updated expiry. Handles refresh failures gracefully — marks adapter as DEGRADED if refresh fails, alerts via Audit Layer.

### 4.5 `subscribeToWebhooks(platform: Platform, config: PlatformConfig) -> void`

**Inputs:**
- `platform` — which platform to set up
- `config` — platform config including webhook endpoints and event subscriptions

**What it does:**
Registers or updates webhook subscriptions with the platform API (e.g., Slack Events API subscriptions, Zoom webhook endpoints, Jira webhooks). Validates webhook signatures on incoming payloads to prevent spoofing. Routes validated payloads to the appropriate adapter for normalization.

### 4.6 `queryPlatformACL(platform: Platform, query: ACLQuery) -> ACLResult`

**Inputs:**
- `platform` — which platform to query
- `query` — subject, resource, and action type to check

**What it does:**
Calls the platform's native permission API to check whether a specific user/app can perform a specific action on a specific resource. Caches results with a configurable TTL (invalidated by group membership change events). Used by the Identity Engine to enforce the "intersection of technical scope and platform ACL" principle.

### 4.7 `queryGroupMembership(platform: Platform, group_ref: string) -> GroupMembershipResult`

**Inputs:**
- `platform` — which platform to query
- `group_ref` — platform-native group identifier (Slack channel ID, Teams group ID, Google group email, etc.)

**What it does:**
Resolves group membership by calling the platform's group/directory API. Handles nested groups by recursively resolving until all leaf members are found. Returns the flat list of platform user IDs plus any nested group references. Result is cached and invalidated based on platform-specific change events or a configurable polling interval.

---

## 5. Normalized Event Schema

| Event Type | Key Fields | Source Platforms |
|---|---|---|
| `MessageReceived` | platform, channel_id, sender_id, content, timestamp, thread_id? | Slack, Teams, Google Chat |
| `MeetingStarted` | platform, meeting_id, participants[], organizer_id, scheduled_agenda? | Zoom, Teams, Google Meet, Slack Huddles |
| `MeetingEnded` | platform, meeting_id, duration, final_participants[] | Zoom, Teams, Google Meet, Slack Huddles |
| `ParticipantJoined` | platform, meeting_id, participant_id, timestamp | Zoom, Teams, Google Meet |
| `ParticipantLeft` | platform, meeting_id, participant_id, timestamp | Zoom, Teams, Google Meet |
| `HandRaiseDetected` | platform, meeting_id, participant_id, timestamp | Zoom, Teams |
| `CalledUpon` | platform, meeting_id, participant_id, caller_id, timestamp | Zoom, Teams |
| `TicketUpdated` | platform, ticket_id, field_changes[], updater_id, timestamp | Jira, Asana |
| `TicketCreated` | platform, ticket_id, project_id, fields, creator_id, timestamp | Jira, Asana |
| `ApprovalReceived` | platform, channel_id, approver_id, content, timestamp, thread_id | Slack, Teams |
| `ReactionAdded` | platform, channel_id, message_id, reactor_id, emoji, timestamp | Slack, Teams |
| `CalendarEventCreated` | platform, event_id, organizer_id, participants[], time, agenda? | Google Calendar, Outlook |

## 6. Normalized Action Schema

| Action Type | Key Fields | Target Platforms |
|---|---|---|
| `SendMessage` | platform, channel_id, content, thread_id?, reply_to? | Slack, Teams, Google Chat |
| `RaiseHand` | platform, meeting_id | Zoom, Teams |
| `Speak` | platform, meeting_id, content | Zoom, Teams, Slack Huddles |
| `CreateTicket` | platform, project_id, fields | Jira, Asana |
| `UpdateTicket` | platform, ticket_id, field_changes | Jira, Asana |
| `ScheduleMeeting` | platform, participants[], agenda, time | Google Calendar, Outlook |
| `RequestApproval` | platform, channel_id, admin_id, action_description | Slack, Teams |

---

## 7. Adapter Contract

Every platform adapter must implement the following interface:

```
interface PlatformAdapter {
  initialize(config: PlatformConfig) -> void
  normalizeEvent(raw: RawPlatformEvent) -> OpenCaptainEvent
  translateAction(action: OpenCaptainAction) -> PlatformAPICall
  executeAction(call: PlatformAPICall) -> ActionResult
  queryACL(query: ACLQuery) -> ACLResult
  queryGroupMembership(group_ref: string) -> GroupMembershipResult
  refreshAuth() -> void
  getHealth() -> AdapterHealth
  shutdown() -> void
}
```

This contract ensures uniform behavior across all platforms and allows new adapters to be added without changing downstream modules.
