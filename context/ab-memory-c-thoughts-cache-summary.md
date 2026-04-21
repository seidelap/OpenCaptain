# A/B Memory + C Thoughts Cache

## Overview

This model separates durable knowledge from derived interpretations.

- **A** is the source, provenance, and governance layer.
- **B** is the semantic claim layer.
- **C** is an optional thoughts cache: derived, scoped, traceable views computed from A and B.

The goal is to support agent memory that is:
- auditable
- conflict-aware
- versioned
- reproducible
- safe against silently collapsing contradictions into false certainty

---

## Layer A: Source / Provenance / Governance Graph

A represents the durable record of what exists and what happened.

### Typical contents
- documents
- meeting transcripts
- chat threads
- tickets
- code diffs / PRs
- decision records
- approval events
- authorship / ownership records
- workflow events
- timestamps
- references and lineage between artifacts

### What belongs in A
A should contain things like:
- “Document D was approved by Alice”
- “Transcript T mentions requirement R”
- “PR 238 implemented change X”
- “Decision ADR-7 rejected option Y”
- “Spec v3 supersedes spec v2”

These are **facts about the record** or **institutional/governance facts**.

### Properties of A
- Mostly append-only or immutable-ish
- Strong provenance
- Version-aware
- Does not rewrite history
- Can include explicit decisions and approvals as first-class records

### Purpose of A
A answers:
- What evidence exists?
- Who said what?
- What was approved, rejected, merged, or superseded?
- What is the lineage of this artifact?

---

## Layer B: Semantic Claim Graph

B represents semantic understanding extracted from or grounded in A.

### Typical contents
- entities
- concepts
- claims
- semantic relationships between claims
- scoped or time-bound statements

### Examples of B nodes
- “Retry count is 3”
- “Launch date is June 15”
- “Team X owns service Y”
- “Project Z depends on vendor Q”

### Examples of B relationships
- `supports`
- `contradicts`
- `clarifies`
- `narrows`
- `depends_on`
- `implies`
- `same_as` / entity-resolution style links
- scoped validity relationships

### Important principle
B is about **claims about the world/project/system**, not just artifacts.

### Properties of B
- Versioned
- Stable identities when possible
- Claim meanings should remain distinct from their current endorsement status
- Grounded back to A via provenance links

### Purpose of B
B answers:
- What is being claimed?
- Which claims align or conflict?
- What semantic structure exists across the corpus?

---

## Why Separate A and B

The main distinction is:

- **A = what happened in the record**
- **B = what is being claimed about the world**

Examples:

- “Approved by Alice” belongs in **A**
- “Retry count is 3” belongs in **B**

This matters because:
- approval is a fact about governance
- the requirement itself is a domain claim

The system should not collapse:
- “X is approved”
into
- “X is true”

Approval is evidence and authority, but not identical to truth.

---

## Contradiction vs Supersession

A core rule of the model is:

**Contradiction does not imply replacement.**

Two claims may conflict without one automatically superseding the other.

### Contradiction
Use when:
- two claims cannot both be true in the same scope/time/context

### Supersession / replacement
Use only when there is additional evidence such as:
- explicit acknowledgment of the earlier claim
- formal lineage (“v3 replaces v2”)
- a ratified decision
- stronger authority applied through governance records
- explicit rejection or retraction

### Recommended rule
Never automatically turn `contradicts` into `supersedes`.

This preserves unresolved disagreements instead of hiding them.

---

## Decisions as First-Class Objects

Some conflicts should be resolved by explicit decision records rather than inferred from raw claim comparison.

Example:
- Claim A: “Use PostgreSQL”
- Claim B: “Use DynamoDB”
- Decision D: “Architecture review selected DynamoDB”

Then:
- D is stored in **A**
- A and B link so the system can see:
  - D considered A
  - D considered B
  - D selected B
  - D rejected A

This is cleaner than pretending the later claim automatically overwrote the earlier one.

---

## Why Not a Fully Separate Durable Layer C

We considered a permanent third layer for beliefs or current worldview.

The conclusion was:

- durable belief storage is **not strictly necessary**
- the “current opinion” can often be **computed from A and B at query time**
- but some derived views are still useful to cache or snapshot

This leads to an optional **C thoughts cache** rather than a full durable worldview graph.

---

## Layer C: Thoughts Cache

C contains **derived views**, not primary truth.

C is a cache of:
- query-specific syntheses
- context packs
- temporary working interpretations
- conversation-scoped summaries
- compiled “best current view” outputs

### What C is
C is like:
- a build artifact
- a cached query result
- a reproducible synthesis
- a thought product derived from A and B

### What C is not
C is **not** the authoritative source of truth.

If a C artifact becomes formally approved, it can be promoted into A as a new authoritative artifact.

---

## Why Have a C Thoughts Cache at All

C is useful for:

### 1. Performance
Some queries may require expensive reasoning over large A/B graphs.

### 2. Consistency
Repeated reasoning from scratch may produce inconsistent answers if retrieval differs.

### 3. Auditability
It can be valuable to know:
- what the system concluded
- why
- with what inputs
- under what reasoning policy
- with what omissions

### 4. Reproducibility
A derived answer should be replayable from the A/B state it used.

---

## Recommended Structure of a C Artifact

Each C artifact should be traceable to the exact context that produced it.

### Suggested fields
- `c_id`
- `created_at`
- `created_for_query` or task description
- `conversation_id` or workflow id
- `a_snapshot_id` or source manifest
- `b_snapshot_id`
- `retrieval_manifest`
- `reasoning_policy_id`
- `claim_set_used`
- `conflicts_detected`
- `missing_expected_sources`
- `output`
- `confidence`
- `invalidated_by`

---

## Retrieval Manifest

A key concept in C is the **retrieval manifest**.

This goes beyond normal citations.

It should record:
- which artifacts from A were retrieved
- which versions were used
- which sources were eligible but not retrieved
- whether retrieval was truncated
- whether linked artifacts were missing
- whether source classes were excluded
- whether token or time limits constrained reasoning

This allows the system to distinguish:

- “No contradiction found”
from
- “No contradiction found in the subset of evidence actually retrieved”

That distinction is critical for trust.

---

## Missingness and Uncertainty

C should explicitly track missingness.

A good system should be able to say:
- this answer is based only on specs, not meeting notes
- this result is incomplete because linked ADRs were unavailable
- this synthesis used a truncated retrieval set
- this conclusion may change if code history is included

This is better than false confidence.

---

## Invalidation and Refresh

C artifacts should be invalidate-able.

Possible invalidation triggers:
- a cited artifact in A changes
- a relevant B node is re-versioned
- a new authoritative source appears in the same topic neighborhood
- a prior conflict is resolved
- entity resolution changes

This keeps C disposable and regenerable.

---

## How Reasoning Works with A, B, and C

### Durable truth substrate
- A contains the evidence and governance record
- B contains extracted semantic claims and relations

### Query-time reasoning
When asked a question, the system:
1. retrieves relevant artifacts and claim subgraphs
2. checks support, contradiction, scope, and time
3. applies governance / authority rules from A
4. computes a best current answer
5. exposes unresolved conflicts if needed
6. optionally stores the result as a C artifact

### Authority examples
Examples of rules that may be applied:
- approved spec outranks brainstorm transcript
- merged implementation may outrank stale documentation for implementation truth
- explicit rejection demotes a claim
- equal-authority contradictions remain unresolved
- missing evidence lowers confidence

---

## Promotion from C Back Into A

Most C artifacts should remain temporary.

However, if a synthesized view is:
- reviewed
- approved
- adopted
- or otherwise formalized

then it can be promoted into A as a new authoritative artifact.

Example:
- an agent synthesizes several conflicting documents
- a human approves the synthesis
- the synthesis becomes a durable decision memo or state-of-record doc
- future reasoning can now cite it as part of A

This closes the loop without making cached thoughts the source of truth by default.

---

## Design Principles

### 1. Preserve history
Do not rewrite raw sources or erase outdated claims.

### 2. Ground every belief
Every claim or synthesis should point back to evidence.

### 3. Keep contradiction visible
Unresolved disagreement should remain visible until explicitly adjudicated.

### 4. Separate approval from truth
Governance matters, but should not be confused with reality.

### 5. Treat answers as derived artifacts
A synthesis should be traceable, scoped, and reproducible.

### 6. Make caches disposable
C should be safe to regenerate from A and B.

### 7. Store adjudications durably
If a human or trusted process resolves a conflict, that resolution belongs in A.

---

## Mental Model

A concise mental model:

- **A = evidence, provenance, governance**
- **B = semantic propositions**
- **C = compiled thoughts / cached interpretations**

Or:

- **A = what happened in the record**
- **B = what is being claimed**
- **C = what the system concluded for a given context**

Another helpful analogy:

- **A = source repository**
- **B = normalized intermediate representation**
- **C = compiled output / build artifact**

---

## Why This Fits Agent Memory Well

This structure is better than naive long-term memory because it supports:
- cross-document reasoning
- conflict awareness
- versioned understanding
- explicit authority handling
- reproducible query-time synthesis
- safe caching without silently overwriting history

It is especially suited for systems where:
- sources arrive over time
- contradictions are common
- authority matters
- decisions may be made later than the evidence appears
- conversation outputs should remain explainable

---

## Summary

Recommended durable architecture:

- **A**: source / provenance / governance graph
- **B**: semantic claim graph

Recommended optional derived layer:

- **C thoughts cache**: traceable, versioned, invalidatable derived views tied to exact A/B inputs and retrieval context

Core rule:

**Use A and B as the truth substrate. Use C as a reproducible thought product, not as the canonical memory of record.**
