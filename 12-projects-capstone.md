# 12 — Capstone Projects

Three large projects, each 2–6 weeks. Pick based on what you want to be good at. Each is written the
way I would scope a real project: a problem statement, requirements, a design you must produce, and
an acceptance bar.

**The deliverable for every capstone is a design doc plus a working system plus an eval report.** If
you only build the system, you have done half the work — the doc and the numbers are what make it
reviewable, and reviewability is the actual skill.

---

## C1 — `memoryd`: a production memory service

**Scope:** 4–6 weeks · **Chapters:** all · **Outcome:** the portfolio piece.

### Problem statement

Build a standalone memory service that any agent can call over HTTP. It must serve 10M tenants with
the workload characteristics from chapter 07.1, meet a p99 read latency of 150ms, support a 5-minute
freshness SLO, and pass a deletion audit.

### Required capabilities

**API**

```
POST   /v1/memories                 write (async; returns job id)
GET    /v1/memories/search          retrieve, tenant-scoped, budgeted
GET    /v1/memories                 list for a tenant (paginated, for the UI)
PATCH  /v1/memories/{id}            user correction
DELETE /v1/memories/{id}            erasure, cascading
POST   /v1/sessions/{id}/ingest     bulk ingest a completed session
GET    /v1/memories/{id}/provenance why do you believe this
POST   /v1/tenants/{id}/export      portability
POST   /v1/tenants/{id}/erase       full erasure
GET    /v1/tenants/{id}/snapshots   rollback points
```

**Storage** — Postgres, hash-partitioned by `tenant_id`, with RLS. Facts + episodes + vectors +
lexical index in one database. Object storage for cold episodes.

**Write path** — async queue partitioned by tenant, idempotent on
`(session_id, turn_index, extractor_version)`, transactional outbox for index updates, DLQ with
alerting, backpressure that degrades to a cheaper extractor rather than dropping.

**Read path** — gate → parallel retrieval arms → RRF → optional rerank → budget → render. Deadline
propagation with graceful degradation at every stage; the skip order must be documented.

**Background** — consolidation, reflection, decay/tiering, contradiction sweep, re-embedding
migration support (dual model_id serving).

**Safety** — provenance with trust levels, untrusted-content-cannot-become-instruction enforcement,
secret/PII scanning, per-tenant snapshots, kill switch (global/tenant/category) without deploy.

### Design doc you must write

Sections, in this order — this is the shape I expect in a real review:

1. **Workload characterisation.** Your own numbers, with the reasoning. Get this wrong and everything
   downstream is wrong.
2. **Data model.** Full DDL, with an explanation of each non-obvious column.
3. **Read path.** Sequence diagram with a latency budget summing to your SLO. State what is skipped
   first under pressure.
4. **Write path.** Delivery guarantees, ordering, idempotency, failure handling.
5. **Multi-tenancy and isolation.** What is the boundary, and what enforces it below the app layer.
6. **Deletion.** The full cascade, including derived artefacts, caches, analytics, backups.
7. **Migrations.** How you re-embed and how you upgrade the extractor without invalidating user data.
8. **Capacity and cost.** Monthly cost at target scale, with the largest line item identified and a
   plan to reduce it.
9. **Observability.** The metrics from 07.9 and the alerts on each.
10. **Failure modes.** The 07.10 table, filled in for your design.
11. **Alternatives considered.** At least two, with the reason you rejected each. A design doc without
    this section is an advertisement, not a design.

### Acceptance criteria

- [ ] p99 read latency < 150ms at 1000 QPS with 10k tenants × 5k memories, measured under load
- [ ] p99 write freshness < 5 minutes under sustained load
- [ ] Cross-tenant isolation test passes, enforced by RLS not application code
- [ ] Deletion cascade test: deleted content appears in zero outputs including summaries
- [ ] Scenario suite from 04.10: 8/8
- [ ] Eval harness from 08.4 with ≥ 100 cases including abstention; report the triple per category
- [ ] Red team: zero untrusted payloads become instruction-stance memories
- [ ] Graceful degradation demonstrated: artificial latency in one arm does not violate SLO
- [ ] Kill switch works without a deploy, demonstrated live
- [ ] Restore a tenant from snapshot, demonstrated live

### Stretch

- Region-sharded deployment with data residency enforcement.
- A memory management UI: list, view provenance, edit, delete, disable per category.
- Dual-model embedding migration executed end to end on live data with zero downtime.

---

## C2 — `codemem`: memory for a coding agent

**Scope:** 3–4 weeks · **Chapters:** 02, 06, 07 · **Outcome:** deep expertise in the highest-value
current application of agent memory.

### Problem statement

Coding agents fail on long tasks for memory reasons: they forget conventions, re-attempt approaches
that already failed, lose track of what they changed, and degrade after compaction. Build the memory
layer that fixes this for a real repository.

### Required capabilities

**Procedural memory** (`PROJECT.md`, agent-maintained, git-tracked)
- Rules learned from failures, with provenance comments and confidence.
- Human-pinned entries immune to agent pruning — enforced in code.
- Size cap with forced consolidation; rules unused for 90 days demoted to an archive section.
- Every rule change appears as a diff in a PR, so humans review the agent's learning.

**Task memory** (per task, externalised to files)
- `PLAN.md`, `PROGRESS.md`, `FINDINGS.md`, and critically `DEAD_ENDS.md`.
- Structured compaction carry-over (02.2 Level 3) that survives ≥ 10 rounds.
- `compaction_guard` in CI over recorded transcripts; identifier loss is a tracked metric.

**Codebase memory**
- Structural index: symbols, call graph, ownership, test↔source mapping.
- Just-in-time retrieval: the agent gets identifiers (file paths, symbol names) and pulls content only
  when it needs it, rather than pre-loading.
- Invalidation on file change — a memory about `payments.go` must expire when that file changes, and
  this is the hard part.

**Trajectory memory**
- Store task trajectories with outcome, duration, token cost, and extracted lessons.
- Retrieve similar past tasks at task start; weight failures higher than successes.
- Hard rule: a retrieved trajectory never authorises an action that would otherwise need confirmation.

**Sub-agent isolation**
- Delegate exploration (search, test runs, log analysis) to sub-agents with isolated contexts that
  return ≤ 2k-token summaries.

### Evaluation

Build a benchmark of 30 real tasks in one repository, spanning: small bug fixes, multi-file
refactors, a migration requiring 20+ file changes, and two tasks requiring knowledge learned from an
earlier task in the set (this is the memory test).

**Measure**
| Config | Success rate | Tokens/task | Wall clock | Convention violations | Repeated dead ends |
|---|---|---|---|---|---|
| No memory | | | | | |
| + procedural file | | | | | |
| + task files & compaction | | | | | |
| + trajectory memory | | | | | |
| + sub-agents | | | | | |

**"Repeated dead ends"** is the metric that most directly demonstrates memory value and the one
nobody measures. Instrument it explicitly.

### Acceptance criteria

- [ ] The two knowledge-transfer tasks succeed only with memory enabled
- [ ] Convention violations drop by ≥ 50% with the procedural file
- [ ] Zero repeated dead ends within a task after `DEAD_ENDS.md` is introduced
- [ ] Structured carry-over survives 10 compactions with < 5% identifier loss
- [ ] File-change invalidation demonstrated: stale codebase memory does not surface after an edit
- [ ] Procedural rules land as reviewable diffs, and a human-pinned rule survives an agent pruning pass

---

## C3 — `orgmem`: shared memory for a multi-agent system

**Scope:** 4–5 weeks · **Chapters:** 05, 06, 07, 09 · **Outcome:** expertise in the hardest open
problem in the field.

### Problem statement

Build shared memory for a team of specialised agents (say: a research agent, an analysis agent, a
drafting agent, and a reviewer) working on behalf of an organisation. They must accumulate shared
knowledge without contaminating each other, and without leaking across organisational boundaries.

This is genuinely unsolved territory — cross-agent identity, shared-state consistency, and blast-radius
containment are open problems. Treat the capstone as research, and expect to document what does not
work as carefully as what does.

### Required capabilities

**Namespacing and access control**
- Memory keyed by `(org, namespace, subject, category)`.
- Per-agent read/write scopes. Default: agents write to their own namespace and read shared.
- Promotion to shared memory is an explicit, privileged operation with a review step.

**Trust propagation**
- Every memory carries trust level and source.
- **Derived memories inherit the minimum trust of their inputs.** Implement and test the laundering
  attack from 09.6.
- Agents cannot escalate trust by paraphrasing.

**Consistency**
- Concurrent writes from multiple agents to the same subject. Choose and justify a policy: last-write-
  wins, per-agent branches with reconciliation, or serialised via a coordinator.
- Bi-temporal facts (chapter 05) so disagreement between agents becomes a versioned belief, not a
  race.
- A conflict surface: when agents disagree about a fact, the disagreement must be visible, not
  silently resolved.

**Coordination**
- Shared memory blocks (Letta-style) for task state.
- Handoff protocol: what does agent A pass to agent B, and how much context survives the boundary?
- Measure fidelity loss per handoff — this is the multi-agent equivalent of context collapse and it is
  the core failure mode.

**Containment**
- Blast-radius limits: a memory written by a low-privilege agent must not alter a high-privilege
  agent's behaviour.
- Per-agent behavioural drift monitoring against a fixed probe set.
- Snapshot and rollback at org granularity.

### Evaluation

Build a workflow requiring all four agents (e.g. "produce a competitive analysis of X with sources,
reviewed for accuracy"), run it 20 times with variations.

**Measure**
- Task success rate with shared memory vs. isolated agents vs. one monolithic agent.
- Information fidelity across handoffs: inject 10 specific facts at the research stage, count how many
  survive to the draft correctly.
- Contamination: inject one poisoned memory into one agent's namespace; measure how many agents'
  behaviour changes, and how many hops it travels.
- Cost: total tokens vs. the monolithic baseline. Multi-agent is often more expensive; quantify what
  you are buying.

### Acceptance criteria

- [ ] Shared memory beats isolated agents on task success, with the cost delta quantified
- [ ] Handoff fidelity ≥ 80% on injected facts, with the loss mechanism characterised
- [ ] A poisoned memory in a low-privilege namespace does not change high-privilege agent behaviour
- [ ] Trust laundering attack blocked and tested
- [ ] Concurrent-write policy documented, implemented, and demonstrated under contention
- [ ] Agent disagreement is surfaced rather than silently resolved
- [ ] A written analysis of what did *not* work and why — required, not optional

---

## How to present a capstone

If you are using this for interviews or promotion, package it as:

1. **README** — the problem in three sentences, the architecture diagram, the headline numbers.
2. **Design doc** — the full doc from C1's structure. This is what senior engineers will actually
   read.
3. **Eval report** — the triple (accuracy, tokens, latency), broken down by category, with a baseline
   comparison and an honest section on what is still broken.
4. **A 10-minute demo** — including the failure cases. Showing a system's limits is a stronger signal
   of engineering maturity than showing its successes.

The honest limitations section is the part that distinguishes senior work. Every memory system has
sharp edges; knowing exactly where yours are is the deliverable.

Next: `13-reading-list.md`.
