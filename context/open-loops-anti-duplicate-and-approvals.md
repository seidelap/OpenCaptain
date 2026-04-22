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

## 7. The `OutreachRequest` node (in progress)

A **first-class graph node** for any situation where the agent has sent a message and is waiting for a human (or agent) response. Replaces the former “open question” sketch with a concrete model.

**Sketch of graph edges (schema TBD in next pass):**

```
[OutreachRequest]
  --regarding-->   [Blocker | WorkItem | PendingAction]
  --targets-->     [UnifiedIdentity]
  --status-->      OPEN | RESPONDED | ESCALATED | CANCELLED
  --attempt-->     int
  --sent_at-->     timestamp
  --grounded_in--> [Layer A artifact: the actual message sent]
```

**Parties:** agent as asker *or* as the party expected to answer, depending on flow.

**”More time needed”:** when a recipient says “check back Friday,” this is an `OutreachRequest` status transition (not a timer). The Policy Engine evaluates context — including this signal — before deciding whether re-outreach is appropriate. No separate scheduler required for most cases; if precise wake timing matters, a minimal `scheduleWakeEvent()` on Module 1’s EventBus (with the `OutreachRequest` id as cancellation key) is sufficient.

**Open questions for next pass:**
- Exact status transitions and terminal states
- Whether `ESCALATED` is a status or a new `OutreachRequest` targeting a different identity
- Whether clarifications raised *by humans* (not the agent) also get an `OutreachRequest`, or a sibling type

**Use cases:** reduce duplicate blocker pings / approval requests; support “I’ll get back by Friday”; unify mental model across approvals, nudges, clarifications.

---

## 8. Suggested follow-ups (when you resume)

1. ~~Name and place the outreach / open-loop record.~~ **Done:** `OutreachRequest` node in Knowledge Store; Layer A for immutable send record, Layer B for status + edges.  
2. ~~Reconcile with scheduled reminder events.~~ **Done:** Policy evaluates graph state, not timers. Optional `scheduleWakeEvent()` on Module 1 for precise timing only.  
3. **Next:** flesh out the `OutreachRequest` node schema — exact fields, status machine, edge types (see §7 open questions).  
4. Define **fingerprinting** for “same issue” (blocker, approval, clarification) — what makes two outreach situations the same fingerprint?  
5. Specify how `AuthorizationRecord` (Module 3) and `OutreachRequest` relate — are approvals a specialization, or parallel types sharing a pattern?  
6. Add a subsection to `modules/06-team-goal-engine.md` and `modules/03-policy-governance-engine.md` once schema is stable.

---

## 9. Related files in this repo

- `design.md` — system design, Knowledge Store, Policy/Approval, Meeting `MEETING_STATE`  
- `modules/06-team-goal-engine.md` — Goal Engine interfaces and `Blocker` model  
- `modules/03-policy-governance-engine.md` — `submitAction`, approval flow  
- `modules/04-knowledge-store.md` — Layer A/B/C, append-only, provenance  
- `development-plan.md` — phased implementation order  
