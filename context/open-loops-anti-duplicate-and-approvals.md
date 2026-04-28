# Open Loops, Anti-Duplicate Messaging, and Approvals — Design Notes

*Reference for continuing design work. Summarizes threads from prior conversations (persistence, policy vs authorization, Team Goal Engine, and unified “open loop” state).*

---

## 1. Persistent state (process is not the database)

- Long-lived truth belongs in **stores**, not the Python (or any) process: **Knowledge Store** (A/B/C), **versioned config** (Module 8), **append-only audit** (Module 7).
- **Authorizations:** Layer A is authoritative; fast caches (e.g. Authorization Store) **rebuild on restart** from A.
- **Runtime:** event-driven handling (webhooks, queues) + **async** pipelines (e.g. A→B extraction) + optional **scheduled** work (e.g. reminder / wake events). Deployment (containers, cloud DBs) is separate from the *logical* durability model.

---

## 2. Policy verdict vs. human-gate resolution

- The **Policy Engine** can keep a **small** set of **principled** outcomes, e.g. `ALLOW` | `DENY` | `REQUIRES_HUMAN_GATE` (aka “need governance before this action is allowed” — what we often call REQUIRE_APPROVAL at the *rules* layer).
- **Do not fold** “pending / already asked / now authorized” into that same enum in one pass. A **second step** (or orchestration) combines the rule result with:
  - current **authorization** records, and
  - **pending** request state  
  to get **execute now**, **wait (already requested, don’t spam)**, or **open a new request**.

This keeps “what do the rules say?” separate from “what is the status of *this* real-world request?”

---

## 3. Where approvals live (and what “config” means)

- **Config** is a good home for **rules, defaults, admin DL pointers, thresholds** — not for a high-churn stream of every **instance** grant (unless you namespace operational state very clearly).
- **Durable** approval evidence stays **governance-oriented** (e.g. Layer A artifacts) + a **queryable** pending/granted model for UI, idempotency, and “don’t double-notify.”
- **Admin UI** can still group “policy + authorizations” in one product surface, with **two logical backends** (or namespaces).

---

## 4. Chat as channel vs. source of truth

- Using **Slack/Teams messages** for human *interaction* is right; using **only** channel history as the **sole** **database** of pending work is **fragile** (binding, replays, edits, which “yes” matches which request).
- **Pattern:** message-first UX, **durable** minimal record (pending id, thread id, target action, status). Parse admin replies in context (threads, admin DL, optional opaque id in body).

---

## 5. Anti-duplicate messaging and Module 6 (Team Goal Engine)

> **Status:** The function names below (`decideAction`, `detectBlockers`, `syncWorkItems`, `Blocker.resolved`) are obsolete after the Module 6 rewrite — Module 6 is now organized as three stages (Tree Maintenance / Candidate Scoring / Action Proposal) over a unified `Goal` Entity. The dedup resolution described in this section (Policy evaluates graph state via `OutreachRequest` query) is unchanged and remains the canonical answer.

- Module 6’s **`decideAction` → `submitAction`** flow does **not** yet fully specify how to **avoid repeating** the same nudge (blocker surface, reminder, “two requirements” post, access request) on every `detectBlockers()` run.
- **Existing hooks:** `Blocker.resolved`, **B graph** changes (`supersedes`, ticket updates), `syncWorkItems` “previous state” — help when the *world* changes; they do not alone solve “same open issue, many evaluation cycles.”
- **Resolution (decided):** dedup is handled by the **Policy Engine evaluating graph state**, not by a timestamp field or static policy YAML. Before re-sending, `assembleContext()` queries the Knowledge Store for any `OutreachRequest` node linked to the same fingerprint and includes its full context (attempt count, status, recipient activity signals, blocker severity). Policy rules evaluate that context and DENY if re-sending isn’t appropriate. This allows arbitrarily complex re-outreach logic — not just “has the cooldown expired?” but e.g. “severity dropped,” “higher-priority item already open for this person,” “recipient showed partial signal.”
- **No separate dedup store needed.** The graph is the source of truth.

---

## 6. Shared machinery: approvals, nudges, clarifications

**Confirmed (no longer a hypothesis):** the same **`OutreachRequest`** pattern backs all three cases:

- **Approval / authorization** pending and satisfied states  
- **Goal engine** interventions (blockers, sync suggestions, stale commitments)  
- **Clarification** and other “waiting on a human (or agent)” states  

**The pattern:** a graph-resident `OutreachRequest` node in the Knowledge Store, with the immutable record of what was sent in **Layer A** and the status + relationships as **Layer B** claims pointing back to it. `AuthorizationRecord` in Module 3 is already this pattern for the approval case; `OutreachRequest` extends it to nudges and clarifications.

**Layer split:**
- **Layer A** — immutable record of what was sent (auditable, append-only)
- **Layer B** — status (`OPEN` | `RESPONDED` | `ESCALATED` | `CANCELLED`) and graph edges to the blocker, work item, and target identity

**Not** that every case uses identical schema — but the **discipline** is shared: *durable record, id/correlation, status machine, Policy evaluates graph state before re-sending*.

---

## 7. Graph schema (settled)

### Node types

**Layer A — one node type:** `Artifact`, discriminated by `artifact_type`:

| artifact_type | What it records |
|---|---|
| TRANSCRIPT, MESSAGE, TICKET | Platform events, raw source |
| DECISION, MEETING_STATE | Governance and meeting records |
| OUTREACH_SENT | Immutable record of a message the agent sent |
| AUTHORIZATION_GRANTED, AUTHORIZATION_DENIED | Approval outcomes (replaces standalone `AuthorizationRecord`) |
| ACL_SNAPSHOT, CALENDAR_EVENT | Identity and scheduling records |
| DETECTION_RECORD | Goal Engine scoring snapshot — grounds engine-derived Claims (progress, coverage, alignment, relevance) so decisions are replayable |

W3C PROV edges only (`wasAttributedTo`, `wasGeneratedBy`, `wasDerivedFrom`). Never point into Layer B.

**Layer B — Entity types:**

| Entity | What it holds |
|---|---|
| `Identity` | A person or service principal — subject/object of claims |
| `Team` | A coordination scope |
| `Goal` | The unified node type for the goal hierarchy: initiatives, epics, tickets, subtasks, "get access" tasks, "clarify this disagreement" tasks. Variation expressed via Claim predicates (`nature`, `origin`, `tracked_in`, `assigned_to`, `status`), not via subtypes. There is no separate `WorkItem` or `Blocker` Entity — those are Goals with specific Claim profiles. |

**Layer B — non-Entity nodes:**

| Node | What it holds |
|---|---|
| `Claim` | Subject-predicate-object world-state assertion |
| `OutreachRequest` | Agent communication state: `status` (OPEN \| RESPONDED \| CANCELLED), `attempt_count`, `last_sent_at` |

Escalation is not a status on `OutreachRequest` — it is a new `OutreachRequest` targeting a different identity, linked by sharing the same `regarding` target.

### Edge types

**Claim-to-claim:**

| Edge | Notes |
|---|---|
| `relates_to` | Flags the pair for Policy evaluation; most edges result in silence |

**Entity-to-entity:**

| Edge | Notes |
|---|---|
| `decomposes_into` | Goal → Goal. Parent-child structural decomposition; the Goal Engine walks these for progress propagation and resolution cascade. |
| `depends_on` | Goal → Goal. Sequencing/precedence; orthogonal to decomposition. Resolution does not cascade across these. |
| `same_as` | Deduplication link; used by Goal Engine `mergeGoals()` when authored and extracted Goals refer to the same outcome. |

**OutreachRequest edges:**

| Edge | To | Notes |
|---|---|---|
| `regarding` | Entity \| Claim | Target type determines downstream behavior on RESPONDED |
| `targets` | Identity | Who the agent is waiting on |

**Crosses layers (B → A only):**

| Edge | Carries | Notes |
|---|---|---|
| `grounded_in` | position: ASSERTS \| AGREES \| UNCERTAIN \| DISAGREES \| RETRACTS | Used on both `Claim` and `OutreachRequest` nodes. Current claim state is fully derivable from these positions — no separate `epistemic_status` field needed. |

**Dropped:** `supports`, `contradicts`, `clarifies`, `narrows`, `supersedes`, `revises`, `approved_by`, `rejected_by` — collapsed into `relates_to` + `grounded_in` positions.

### Claim versioning

`claim_id` is stable across versions; `version` is monotonically increasing. `grounded_in` refs live on a specific version — they are evidence for that version's content.

- **New version:** content (object or predicate) materially changes. Anyone's evidence can produce a new version — no restriction to the original claimer. Extraction pipeline makes the judgment call.
- **New `grounded_in` ref:** stance changes (AGREES, DISAGREES, RETRACTS) on existing content. No new version.
- **Current version:** max version for a given `claim_id`.
- **`relates_to` edges:** connect specific versions. Re-evaluated fresh on new version creation; prior-version edges preserved as history. Policy evaluates latest-version edges only.

### Outreach trigger condition

`relates_to` + silence is the default — most edges never generate outreach. An `OutreachRequest` is created only when all three conditions are met:

1. `relates_to` exists between claims from different authors
2. At least one author lacks a `grounded_in` edge to the opposing claim (mutual acknowledgement gap)
3. The claims require convergence in the current context (actionable, in scope, timely)

Once both authors have `grounded_in` edges to each other’s claims, the conflict is mutually known — the appropriate response shifts from surfacing to resolution, which is a different kind of outreach or none at all.

---

## 8. Suggested follow-ups (when you resume)

1. ~~Name and place the outreach / open-loop record.~~ **Done:** `OutreachRequest` in Layer B; `OutreachSentRecord` as OUTREACH_SENT artifact in Layer A.
2. ~~Reconcile with scheduled reminder events.~~ **Done:** Policy evaluates graph state. Optional `scheduleWakeEvent()` on Module 1 for precise timing only.
3. ~~Flesh out graph schema.~~ **Done:** see §7.
4. ~~Specify how `AuthorizationRecord` and `OutreachRequest` relate.~~ **Done:** `AuthorizationRecord` is now an AUTHORIZATION_GRANTED artifact in Layer A; `OutreachRequest` is the pending state in Layer B.
5. ~~Propagate graph schema to module specs.~~ **Done:** Module 4 (Entity types, edge types, DETECTION_RECORD, subscribeInvalidations), Module 3 (AuthorizationRecord as artifact), Module 6 (rewritten as a three-stage decision engine over the unified Goal entity).
6. ~~Goal Engine redesign.~~ **Done:** Module 6 collapsed to one Entity type (`Goal`); recursive `decomposes_into`; engine-derived Claims (`progress_assessment`, `coverage_assessment`, `alignment_assessment`, `goal_relevance`) grounded in DETECTION_RECORD artifacts; reactive recomputation via `subscribeInvalidations()`; two-stage scoring (heuristic + LLM tier-up); active-view invariant via cascade resolution.
7. **Next:** define **candidate fingerprinting** algorithm — the stable hash of (kind, affected_goal_ids, evidence_ids) that identifies "same issue" across enumeration cycles. Currently sketched in Module 6 §9 but not formally specified.
8. **Next:** work out what the `relates_to` detection algorithm looks like — what signals cause the extraction pipeline to create a `relates_to` edge, and what is the expected false-positive rate? Goal Engine scoring is robust to false positives (low-relevance candidates produce silence), but extraction quality still affects audit cost.
9. **Next:** define the heuristic feature weights and the LLM tier-up confidence threshold defaults. Module 6 leaves these to config; we need first-pass values to ship.

---

## 9. Related files in this repo

- `design.md` — system design, Knowledge Store, Policy/Approval, Meeting `MEETING_STATE`  
- `modules/06-team-goal-engine.md` — Goal Engine interfaces and `Blocker` model  
- `modules/03-policy-governance-engine.md` — `submitAction`, approval flow  
- `modules/04-knowledge-store.md` — Layer A/B/C, append-only, provenance  
- `development-plan.md` — phased implementation order  
