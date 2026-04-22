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
- **Needed (design intent):** durable **outreach / coordination** state, e.g. per **stable fingerprint** of an issue: last nudge time, count, link to last outbound or Layer A `GOAL_INTERVENTION` (name TBD), **cooldowns** and caps from `getGoalEngineConfig()`.
- **Policy** can **DENY** ill-formed duplicates in theory, but **dedup policy** should be computed in the **decision + store** path, not duplicated as static policy YAML for every case.

---

## 6. Shared machinery: approvals, nudges, clarifications

**Hypothesis (to validate in a later pass):** the same **“open loop”** and **anti-duplicate** substrate can back:

- **Approval / authorization** pending and satisfied states  
- **Goal engine** interventions (blockers, sync suggestions, stale commitments)  
- **Clarification** and other “waiting on a human (or agent)” states  

**Not** that every case uses identical schema — but the **discipline** is shared: *durable record, id/correlation, status machine, don’t re-send until rules say so*.

---

## 7. Open design question: “questions raised but unanswered”

**Idea to explore next session:** a first-class (or near-first-class) object for **questions / obligations** that are **unresolved** because:

- the **owner of the answer** has not responded yet, or  
- someone explicitly said **“more time needed”** (and optionally a new expectation of when to follow up)  

**Parties** may include **users A/B** and the **agent** as asker *or* as the party expected to answer (depending on flow).

**Possible states (sketch, not final):** `OPEN` | `ANSWERED` | `MORE_TIME` | `TIMEOUT` | `CANCELLED` — exact names and transitions TBD.

**Use cases:** reduce duplicate “did you see this?” / duplicate approvals / duplicate blocker pings; support honest “I’ll get back by Friday”; unify mental model with pending approvals where appropriate.

This is **intentionally unfinished**; it should be refined against:

- platform threading and identity (Module 1)  
- Knowledge Store artifacts (Module 4)  
- Policy and governance (Module 3)  
- Team Goal and Meeting engines (5 & 6)  

---

## 8. Suggested follow-ups (when you resume)

1. Name and place the **outreach** / **open-loop** record (Layer A types vs. small operational table vs. both).  
2. Define **fingerprinting** for “same issue” (blocker, approval, clarification).  
3. Specify **one** state machine for **authorizations** and **separate** (or branched) machine for **Q&A / clarification** if semantics differ.  
4. Add a subsection to `modules/06-team-goal-engine.md` (and/or design.md) once the above is stable.  
5. Reconcile with **scheduled reminder** events (wake without external webhook).

---

## 9. Related files in this repo

- `design.md` — system design, Knowledge Store, Policy/Approval, Meeting `MEETING_STATE`  
- `modules/06-team-goal-engine.md` — Goal Engine interfaces and `Blocker` model  
- `modules/03-policy-governance-engine.md` — `submitAction`, approval flow  
- `modules/04-knowledge-store.md` — Layer A/B/C, append-only, provenance  
- `development-plan.md` — phased implementation order  
