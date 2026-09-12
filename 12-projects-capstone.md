# 12 — Capstone Projects

> Goal: three projects at 3–6 week scale where the deliverable is a design doc, a working system,
> and an eval report — plus the scoping and measurement arithmetic that decides whether you finish.
> Every number below was executed before it was written, including a real churn study of three
> memory-system repositories.

---

## 12.0 In plain words

### A dish versus a restaurant

You can cook one excellent dish with a recipe, a pan, and ninety minutes. A restaurant is a
different category of problem: forty covers arriving in a ninety-minute window, a supplier who is
late, an allergy on table six, a health inspection, and staff who quit. None of the individual
skills are harder than the dish. The difficulty is that they all have to be true **at the same
time, repeatedly, while someone is watching.**

Chapter 11 was six dishes. This chapter is the restaurant. The technical content is not harder than
S3; what is harder is that a capstone has to hold latency, cost, isolation, deletion, evaluation and
operability true simultaneously, and has to prove it to someone who did not build it.

### The naive capstone plan

Here is the plan almost everyone writes, and it fits in twenty lines:

```markdown
# memoryd — my capstone
- [ ] Postgres schema with pgvector
- [ ] POST /memories, GET /memories/search
- [ ] extraction + reconciliation (reuse S3)
- [ ] background jobs
- [ ] multi-tenancy
- [ ] deletion
- [ ] evaluation
- [ ] load test: p99 < 150ms at 1000 QPS
- [ ] write the design doc
Timeline: 4 weeks. Basically S3 but bigger.
```

Three problems, none of them about code. The list has no sizing, so week three is a surprise. The
acceptance criteria are unmeasurable as written. And "basically S3 but bigger" is false in the way
that matters: S3 had one tenant, one writer, no SLO, and no adversary.

### The arithmetic that kills it

**1. Your load test cannot see your SLO.** Take a read path whose *true* p99 is 164ms — a genuine
violation of a 150ms SLO. Simulated from the 07.4 stage shapes, 2M-sample population, then
resampled 1,000 times at each test length:

```
population: p50 78ms  p95 129ms  p99 164ms   SLO=150ms -> FAIL

n=   100 (   0s @1000 QPS)  p99 CI [127.3, 207.4]ms  width  80.0ms  reports PASS 44.1% of runs
n=  1000 (   1s @1000 QPS)  p99 CI [149.7, 177.8]ms  width  28.1ms  reports PASS  2.7% of runs
n= 10000 (  10s @1000 QPS)  p99 CI [159.0, 168.4]ms  width   9.4ms  reports PASS  0.0% of runs
n= 60000 (  60s @1000 QPS)  p99 CI [161.9, 165.6]ms  width   3.8ms  reports PASS  0.0% of runs
```

**A 100-request load test passes a failing system 44% of the time.** A p99 is, by construction, one
observation in a hundred; estimating it from 100 samples means estimating it from ~1 sample. The
acceptance criterion "p99 < 150ms at 1000 QPS" is therefore incomplete — it needs a duration. Sixty
seconds at 1000 QPS is 60,000 samples and a ±2ms interval; that is a criterion. This is the same
lesson as §11.0's accuracy intervals, one distribution further into the tail.

**2. Your A/B comparison cannot see your improvement.** C3 as originally scoped says "run the
workflow 20 times". Bootstrap intervals on 20 runs:

```
16/20 = 80%   95% CI [0.60, 0.95]
14/20 = 70%   95% CI [0.50, 0.90]
```

Two configurations 10pp apart with intervals overlapping across 30pp of the range. The claim "shared
memory beat isolated agents" is not supported by that experiment, no matter which way the numbers
fell. Fix: **pair the comparison** — same tasks, same seeds, both configurations — and report the
per-task differences (§8.8, §11.7). Pairing is free and it is the difference between an eval report
and an anecdote.

**3. Your subsystem count exceeds your hours.** C1 as specified has twelve subsystems: schema,
write API, read API, extraction, reconciliation, background jobs, RLS, deletion cascade, migrations,
observability, kill switch, snapshots. Derived inline:

```
12 subsystems x 8 h (build + test + the bug you did not expect)  =  96 h
eval harness (100 cases, judge calibration, per-category report)  =  20 h
load test rig + capacity measurement                              =  12 h
design doc, 11 sections, with real numbers in each                =  15 h
                                                                   ------
                                                                    143 h

available: 4-6 weeks x 15 h/week of evenings  =  60-90 h      -> 1.6-2.4x over
available: 4-6 weeks full time                =  160-240 h    -> fits, with slack
```

At evening pace the plan is **1.6–2.4× over budget**, which means it will be cut. The only question
is whether you cut deliberately in week one or panic in week four. §12.1 is that decision.

### What fixes what

| The thing that sinks capstones | Section |
|---|---|
| Acceptance criteria that cannot be measured | 12.1, and every criterion in 12.2–12.4 |
| A load test too short to see a p99 | 12.2, the 60,000-sample rule |
| Unpaired 20-run comparisons | 12.4, paired protocol |
| Scope 2× over budget, discovered late | 12.1 hours budget + cut order |
| A design doc that is an advertisement | 12.5 required numbers per section |
| Codebase memory that silently goes stale | 12.3 measured churn: 20–63% stale at 90 days |
| One poisoned namespace reaching the privileged agent | 12.4 min-trust: 85% → 0% |
| Fidelity loss across agent handoffs | 12.4 `p ≥ 0.928` for 80% over 3 hops |
| "It works" with no instrument panel | 12.1 reuse `memlab` from §11.1 |

**The deliverable for every capstone is a design doc plus a working system plus an eval report.**
Build only the system and you have done a third of the work — the doc and the numbers are what make
it reviewable, and reviewability is the actual skill being trained.

---

## 12.1 Scoping: the decision you make in week one

Four rules, in order of how much pain they save.

**1. Write the hours budget before the task list.** Count subsystems, multiply by 8 hours, add the
eval harness, the load rig and the doc. If the total exceeds your available hours, cut *now* — and
cut whole subsystems, not the depth of every subsystem. A capstone with 8 complete subsystems is a
portfolio piece; one with 12 half-subsystems is a tutorial that stopped.

**2. Cut in this order.** The order is not arbitrary: it removes the things a reviewer can grant you
on paper and keeps the things that can only be demonstrated.

```
cut first   multi-region, UI, graph tier, reranker, snapshot rollback UI, dual-model migration
cut next    breadth of API surface (5 endpoints beat 10), category count, background job count
cut last    deletion cascade, tenant isolation, the eval harness, the kill switch
never cut   provenance columns, the episode log, the triple in every report
```

The "never cut" row is retrofit-impossible. You can add a reranker in an afternoon in month three;
you cannot add `derived_from` to memories that were written without it, and you cannot re-derive
facts whose episodes you discarded (04.6).

**3. One novel thing.** Choose exactly one place where you do something harder than the textbook —
bi-temporal invalidation, or hash-based codebase invalidation, or trust propagation. Everywhere else,
do the boring known-good thing. Capstones fail by being novel in six places at once, each 70% done.

**4. Instrument on day one, with §11.1's harness.** `Run.report`, `boot_ci`, one definition of
`recall_at_k`, `mutation_score`. Every number in your eval report should come out of the same code
path, and the load rig should exist before the system is finished — it is how you notice, in week
two, that your read path is 400ms.

**Weekly checkpoint, one line each:** what is measurable today that was not measurable last week?
If the answer is "nothing, I was building", that is fine once. Twice means you are building without
a feedback loop and the eval report at the end will be assembled from memory.

---

## 12.2 C1 — `memoryd`: a production memory service

**Scope:** 4–6 weeks · **Chapters:** all · **Outcome:** the portfolio piece.

### Problem statement

Build a standalone memory service any agent can call over HTTP. Target the workload in §7.1, a
**p99 read latency of 150ms measured over ≥ 60,000 requests**, a 5-minute freshness SLO, and a
deletion audit that passes at 100%.

### Required capabilities

**API** — five endpoints are enough if the five are complete; the rest is surface area.

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

**Storage** — Postgres, hash-partitioned by `tenant_id`, with RLS (§7.3's lab is the reference, and
it includes the two bugs: owner bypass and the unset GUC). Facts, episodes, vectors and the lexical
index in one database. Object storage for cold episodes.

**Write path** — async queue partitioned by tenant, idempotent on
`(session_id, turn_index, extractor_version)`, transactional outbox for index updates, DLQ with
alerting, backpressure that degrades to a cheaper extractor rather than dropping writes.

**Read path** — gate → parallel arms → RRF → optional rerank → budget → render. Deadline propagation
with graceful degradation, and the skip order documented. §7.4's `TaskGroup` deadline bug — measured
at 901ms against a 120ms budget — is the one to avoid re-deriving the hard way.

**Background** — consolidation, reflection, decay/tiering, contradiction sweep, and re-embedding
support (dual `model_id` serving).

**Safety** — provenance with trust levels, the hard rule that untrusted content cannot become an
instruction-stance memory, secret/PII scanning, per-tenant snapshots, and a kill switch at global,
tenant and category granularity that works without a deploy.

### Capacity arithmetic you must produce before writing code

This is the part of the design doc that determines whether the rest is fiction. All three below were
computed for this chapter; reproduce them with *your* numbers.

**Concurrency (Little's law: in-flight = throughput × latency):**

```
1000 QPS x  75ms =  75 requests in flight  -> x3 retrieval arms = 225 concurrent DB ops
1000 QPS x 150ms = 150 requests in flight  -> x3 retrieval arms = 450 concurrent DB ops
3000 QPS x  75ms = 225 requests in flight  -> x3 retrieval arms = 675 concurrent DB ops
```

Note the feedback loop hiding in the second row: **doubling latency doubles the pool you need**, and
exhausting the pool raises latency further. A connection pool of 100 with pgbouncer in front is a
different design from 450 direct connections; that choice belongs in the doc, sized by this table.

**Storage** (text 400B + vector + ~200B row and index overhead):

```
    10,000 tenants x 5,000 mem =     50,000,000 rows   float32   337 GB   int8   107 GB
 1,000,000 tenants x   500 mem =    500,000,000 rows   float32 3,372 GB   int8 1,068 GB
10,000,000 tenants x   200 mem =  2,000,000,000 rows   float32 13,488 GB   int8 4,272 GB
```

`int8` quantisation is a **3.1× reduction** and the accuracy cost is small enough that you must
measure it rather than assume it (03.6). At the 10M-tenant row the float32 column alone is 13.5 TB,
which is the number that decides whether vectors live in Postgres at all (§7.2's graduation
criteria).

**Cost** — build §7.8's spreadsheet with your own rates and identify the largest line item. At every
scale in that chapter it was injected tokens, by 11× over all extraction. If your spreadsheet says
otherwise, recheck it; if it still says otherwise, you have found something interesting and it goes
in the doc.

### The design doc

Eleven sections, in this order. The parenthetical is the *number* the section must carry — a section
without its number is prose, and prose is where designs hide.

1. **Workload characterisation** — QPS, read:write ratio, memories per tenant p50/p99, session
   length distribution. (Your own numbers, with the derivation.)
2. **Data model** — full DDL, every non-obvious column explained. (Row size in bytes; index sizes.)
3. **Read path** — sequence diagram with a latency budget summing to the SLO, and the skip order
   under pressure. (Per-stage p50/p95, composed p99 — *composed*, not summed: §7.4 and §11.6 show
   the naive sum overstating by 26%.)
4. **Write path** — delivery guarantees, ordering, idempotency key, failure handling. (Freshness
   p50/p99 in seconds; queue depth at steady state, from §7.7's M/M/1 arithmetic.)
5. **Multi-tenancy and isolation** — the boundary, and what enforces it *below* the app layer.
   (Result of the cross-tenant test, plus the RLS-under-partition-pruning check from §7.3.)
6. **Deletion** — the full cascade including derived artefacts, caches, analytics, backups.
   (Residue-site count found by audit — §9.6 found seven after a naive DELETE.)
7. **Migrations** — re-embedding and extractor upgrades without invalidating user data. (Rows/hour
   re-embed throughput; total wall-clock for your corpus.)
8. **Capacity and cost** — monthly cost at target scale, largest line item, plan to reduce it.
   ($/MAU/month and the sensitivity to your context block size.)
9. **Observability** — §7.10's metric set, with an alert per metric. (Citation/usage rate: what
   fraction of injected memories the model actually used.)
10. **Failure modes** — §7.11's table filled in for *your* design. (Time-to-detect per row.)
11. **Alternatives considered** — at least two, with why you rejected each. (The number that made
    each decision.)

Section 11 is the one that separates a design doc from an advertisement. "We chose Postgres over a
dedicated vector DB because at 50M rows the int8 column is 107 GB and one instance serves it, and a
second datastore adds a consistency boundary we would have to reconcile on deletion" is a design.
"We chose Postgres because it is flexible" is a slogan.

### Acceptance criteria

Each one names its measurement protocol. A criterion without a protocol is a wish.

- [ ] **p99 read latency < 150ms at 1000 QPS**, measured over ≥ 60,000 requests in steady state,
      with the p99's bootstrap CI reported. (At 60k the interval is ±2ms; at 100 requests you would
      pass a failing system 44% of the time.)
- [ ] **p99 write freshness < 5 minutes** under sustained load, measured `episode_ts → indexed_ts`
      from the rows themselves, not from the queue's own metrics.
- [ ] **Cross-tenant isolation enforced below the application layer**, proven by a test that
      connects as the app role with the wrong `tenant_id` GUC and gets zero rows — including the
      unset-GUC case, which must fail closed (§7.3).
- [ ] **Deletion cascade at 100%**: a deleted fact appears in no memory row, no vector for any
      `model_id`, no lexical index, no derived summary, no cache, no analytics extract. Enumerate
      sites, then prove each empty.
- [ ] **Write-path scenario suite 12/12 with mutation score ≥ 5/5** (§11.4 — a green suite that
      cannot fail is not a criterion).
- [ ] **Eval harness ≥ 100 cases including ~15% abstention**, reporting accuracy + CI, tokens/query
      and p95 **per category**.
- [ ] **Red team: zero untrusted payloads become instruction-stance memories**, across all three
      delivery channels (user text, document content, tool result).
- [ ] **Graceful degradation demonstrated**: inject 2s of artificial latency into one retrieval arm
      and show the SLO holds and the skip order matches the doc.
- [ ] **Kill switch without a deploy**, demonstrated live at tenant and category granularity.
- [ ] **Restore a tenant from snapshot**, demonstrated live, with the RPO stated.

### Stretch

- Region-sharded deployment with data-residency enforcement.
- A memory-management UI: list, view provenance, edit, delete, disable per category. (This is also
  the fastest way to discover that your provenance data is incomplete.)
- Dual-model embedding migration executed end-to-end on live data with zero downtime.

---

## 12.3 C2 — `codemem`: memory for a coding agent

**Scope:** 3–4 weeks · **Chapters:** 02, 06, 07 · **Outcome:** deep expertise in the highest-value
current application of agent memory.

### Problem statement

Coding agents fail on long tasks for memory reasons: they forget conventions, re-attempt approaches
that already failed, lose track of what they changed, and degrade after compaction. Build the memory
layer that fixes this for a real repository.

### The measurement that should shape your design

Before designing invalidation, measure how fast codebase memory rots. I ran this over three real
memory-system repositories — deepened clones, `.py` files present at HEAD, merges excluded — asking:
**what fraction of source files were touched within the last N days?** A memory scoped to a file is
stale if that file changed after the memory was written.

```
repo        files  hist(d)  stale 7d       30d       90d      180d  edits/file/yr
mem0          403      330     16.1%     25.8%     62.8%     86.6%           3.61
graphiti      268      674      7.1%      9.3%     20.1%     40.3%           3.04
langgraph     452      357      1.1%      7.5%     29.4%     61.1%           4.00
```

(Snapshot 2026-09-12: [mem0](https://github.com/mem0ai/mem0),
[graphiti](https://github.com/getzep/graphiti), [langgraph](https://github.com/langchain-ai/langgraph).
Re-run it on *your* repo; the shape transfers, the constants do not.)

**A file-scoped memory in an active repo has a 20–63% chance of being wrong after 90 days.** That is
the whole argument for invalidation, and it is measured rather than asserted.

Now the interesting part. If you model edits as a uniform Poisson process at the mean rate — the
natural way to pick a TTL — you get this:

```
mem0       λ=3.61 edits/file/yr  half-life 70d  Poisson P(stale@90d)=58.9%  measured=62.8%
graphiti   λ=3.04 edits/file/yr  half-life 83d  Poisson P(stale@90d)=52.7%  measured=20.1%
langgraph  λ=4.00 edits/file/yr  half-life 63d  Poisson P(stale@90d)=62.7%  measured=29.4%
top-10% most-edited files account for 37% (mem0), 56% (graphiti), 41% (langgraph) of all edits
```

For graphiti the mean-rate model predicts **2.6× more staleness than actually occurred**
(52.7% vs 20.1%), and for langgraph 2.1×. The reason is in the last line: edits concentrate. Most
files are quiet for months while a tenth of them absorb half the churn.

Two design consequences, both non-obvious before the measurement:

1. **A uniform TTL is wrong in both directions at once** — too aggressive for the stable majority
   (throwing away valid memory, which costs re-derivation) and too lax for the hot tenth (serving
   stale memory, which costs correctness). Do not pick a TTL from a mean rate.
2. **Invalidate on content hash, not on time.** Store `file_path` plus the blob hash the memory was
   derived from; on read, compare against the current hash. Exact, cheap, and it makes churn
   concentration irrelevant. Keep per-file edit rate as a *ranking* signal — a memory about a hot
   file deserves lower confidence than one about a file nobody has touched in a year — not as an
   expiry rule.

```python
# the invalidation check, in full
def memory_is_current(mem, repo) -> bool:
    """mem: {'path': 'internal/payments.go', 'blob_sha': 'a3f2...', 'symbol': 'Charge'}"""
    now_sha = repo.blob_sha(mem["path"])
    if now_sha is None:      return False          # file deleted -> memory invalid
    if now_sha == mem["blob_sha"]: return True     # byte-identical -> memory valid
    return mem.get("symbol") is not None and \
           repo.symbol_body_sha(mem["path"], mem["symbol"]) == mem.get("symbol_sha")
```

The third branch is the refinement worth building: a file changes constantly while the function your
memory is about does not. Symbol-level hashing turns most file edits into non-events, and it is the
"one novel thing" (§12.1) this capstone should spend its novelty budget on.

### Required capabilities

**Procedural memory** (`PROJECT.md`, agent-maintained, git-tracked) — rules learned from failures with
provenance comments and confidence; human-pinned entries immune to agent pruning, **enforced in
code**; a size cap with forced consolidation at ~100 rules / ~1,800 tokens (06.1); rules unused for
90 days demoted to an archive section; every rule change landing as a reviewable diff. `rule_health`
from 06.1 to catch superstition — a rule whose tasks fail *more* often than baseline.

**Task memory** (per task, externalised to files) — `PLAN.md`, `PROGRESS.md`, `FINDINGS.md`, and
critically `DEAD_ENDS.md`. Structured compaction carry-over (02.2, level 3) surviving ≥ 10 rounds,
with the identifier guard from §11.2 — and remember that the guard needs its own recall measured, or
it passes while losing five identifiers in six.

**Codebase memory** — structural index (symbols, call graph, ownership, test↔source mapping);
just-in-time retrieval, where the agent gets identifiers and pulls content only when needed;
hash-based invalidation per the study above.

**Trajectory memory** — task trajectories with outcome, duration, token cost, extracted lessons;
retrieval of similar past tasks at task start, weighting failures higher than successes; and the hard
rule from 06.5: **a retrieved trajectory never authorises an action that would otherwise need
confirmation.**

**Sub-agent isolation** — delegate exploration (search, test runs, log analysis) to sub-agents with
isolated contexts returning ≤ 2k-token summaries. The always-in-context cost measured in 10.7 is the
reason this pays.

### The metric nobody measures, defined in code

"Repeated dead ends" is the most direct evidence of memory value in a coding agent, and it stays
unmeasured because it needs a definition. Here is one:

```python
def repeated_dead_ends(trace) -> int:
    """A dead end = an (action_kind, target, failure_class) that already failed in this repo.
    Counted once per repeat, so 3 attempts of the same failing approach = 2 repeats."""
    seen, repeats = set(), 0
    for step in trace:
        if step["outcome"] != "failure": continue
        key = (step["action_kind"], step["target"], classify_failure(step["stderr"]))
        if key in seen: repeats += 1
        else:           seen.add(key)
    return repeats
```

`classify_failure` is a dozen regexes over compiler/test output — import error, type error, assertion
failure, timeout, permission. Crude is fine; consistent is what matters, because you are comparing
configurations against each other, not against an absolute.

### Evaluation

30 real tasks in one repository: small bug fixes, multi-file refactors, a migration touching 20+
files, and **two tasks that require knowledge learned from an earlier task in the set** — those two
are the actual memory test, and they are the reason task order is fixed.

| Config | Success rate (+CI) | Tokens/task | Wall clock | Convention violations | Repeated dead ends |
|---|---|---|---|---|---|
| No memory | | | | | |
| + procedural file | | | | | |
| + task files & compaction | | | | | |
| + hash-invalidated codebase memory | | | | | |
| + trajectory memory | | | | | |
| + sub-agents | | | | | |

Run it **paired**: same tasks, same base commit, same seed, every configuration. At a realistic 70%
success rate, 30 tasks gives a 95% interval of `[0.53, 0.87]` — **±17pp** — so an unpaired
comparison cannot resolve the 10pp differences you are looking for; paired per-task differences can.

### Acceptance criteria

- [ ] The two knowledge-transfer tasks succeed **only** with memory enabled, and fail with it
      disabled — both directions demonstrated.
- [ ] Convention violations drop ≥ 50% with the procedural file, counted by a linter you wrote for
      *your* conventions (if it is not machine-checkable, it is not a convention).
- [ ] Zero repeated dead ends within a task after `DEAD_ENDS.md` is introduced, using the definition
      above, with the pre-introduction count reported for comparison.
- [ ] Structured carry-over survives 10 compactions with < 5% identifier loss, measured by a guard
      whose own recall is ≥ 9/10 on a labelled set of identifier shapes.
- [ ] **Hash invalidation demonstrated**: edit a file, show the memory about it is not served; edit a
      different function in the same file, show the symbol-scoped memory *is* still served.
- [ ] Your own repo's churn table (the §12.3 study, re-run locally) is in the design doc, with your
      chosen invalidation strategy justified by it.
- [ ] A human-pinned rule survives an agent pruning pass on an over-cap rule file.
- [ ] A fabricated "successful" trajectory for a destructive action does not remove the confirmation
      prompt.

---

## 12.4 C3 — `orgmem`: shared memory for a multi-agent system

**Scope:** 4–5 weeks · **Chapters:** 05, 06, 07, 09 · **Outcome:** expertise in the hardest open
problem in the field.

### Problem statement

Build shared memory for a team of specialised agents — say research, analysis, drafting, review —
working for an organisation. They must accumulate shared knowledge without contaminating each other
and without leaking across organisational boundaries.

This is genuinely unsolved territory: cross-agent identity, shared-state consistency and
blast-radius containment are open problems. Treat it as research, and document what does not work as
carefully as what does. The honest negative result is the deliverable here.

### The two arithmetics that define the project

**Handoff fidelity compounds.** Four agents means three boundaries. If each boundary preserves a
fact with probability `p`:

```
p=0.80 -> 3 hops 51.2%   5 hops 32.8%
p=0.90 -> 3 hops 72.9%   5 hops 59.0%
p=0.95 -> 3 hops 85.7%   5 hops 77.4%
p=0.98 -> 3 hops 94.1%   5 hops 90.4%

to hit >=80% end-to-end over 3 hops you need p >= 0.928
to hit >=80% end-to-end over 5 hops you need p >= 0.956
```

So the criterion "handoff fidelity ≥ 80%" is really a demand for **≥ 92.8% per boundary** — which no
free-text summary achieves. That arithmetic tells you the design before you write it: pass
*structured references* (ids into shared memory) rather than prose, so that fidelity per hop is 1.0
by construction for anything that is referenced rather than retold. Reserve prose for the parts a
reference cannot carry, and measure fidelity separately for referenced versus retold facts. This is
the multi-agent form of 06.0's rule: if it must survive, do not put it behind a lossy channel.

**Contamination spreads unless trust cannot climb.** Simulate one poisoned memory in the
lowest-trust namespace, agents promoting memories to each other with probability 0.5, 20,000 trials,
five agents at trust levels research(1), scraper(1), analysis(2), drafting(2), review(3):

```
min-trust inheritance OFF:  mean hops 1.96  agents touched 4.42/5
   research(t1) 100%  scraper(t1) 86%  analysis(t2) 85%  drafting(t2) 86%  review(t3) 85%

min-trust inheritance ON :  mean hops 0.50  agents touched 1.50/5
   research(t1) 100%  scraper(t1) 50%  analysis(t2) 0%  drafting(t2) 0%  review(t3) 0%

blast radius above trust 1: OFF = 85% of high-trust agents, ON = 0%
```

Without the rule, a single poisoned memory reaches the **highest-privilege agent 85% of the time in
about two hops**. With derived memories inheriting the *minimum* trust of their inputs (09.6), it
reaches 0% above its own level, and the containment is structural rather than probabilistic — it does
not depend on the promotion probability at all, which is why it is a rule and not a mitigation.

### Required capabilities

**Namespacing and access control** — memory keyed by `(org, namespace, subject, category)`; per-agent
read/write scopes, defaulting to write-own / read-shared; promotion to shared memory as an explicit
privileged operation with a review step.

**Trust propagation** — every memory carries trust level and source; **derived memories inherit the
minimum trust of their inputs**; agents cannot escalate trust by paraphrasing. Implement and test the
laundering attack from 09.6 — an agent summarising untrusted content into its own words, with the
summary claiming the agent as source.

**Consistency** — concurrent writes from multiple agents to the same subject; choose and justify a
policy (last-write-wins, per-agent branches with reconciliation, or serialised via a coordinator);
bi-temporal facts (ch 05) so disagreement becomes a versioned belief rather than a race; and a
**conflict surface** where disagreement is visible instead of silently resolved.

**Coordination** — shared memory blocks for task state (10.2); a handoff protocol stating exactly
what crosses the boundary; fidelity loss measured per handoff.

**Containment** — blast-radius limits so a low-privilege agent's memory cannot alter a
high-privilege agent's behaviour; per-agent behavioural drift monitoring against a fixed probe set
(09.8 — content scanning cannot find sleepers, drift can); snapshot and rollback at org granularity.

### Evaluation

Build a workflow requiring all four agents ("produce a competitive analysis of X with sources,
reviewed for accuracy"), then:

- **Task success**, shared memory vs isolated agents vs one monolithic agent. **Paired**: identical
  task instances and seeds across all three, reported as per-task differences. Twenty unpaired runs
  give [0.60, 0.95] against [0.50, 0.90] and settle nothing.
- **Handoff fidelity**: inject 10 specific facts at the research stage, count how many reach the
  draft correctly, and split the count into *referenced* and *retold*. Report the implied per-hop
  `p` and compare against the 0.928 requirement.
- **Contamination**: inject one poisoned memory into the lowest-trust namespace; count agents whose
  behaviour changes and the number of hops travelled, with and without min-trust inheritance. The
  simulation above is the prediction; your measurement is the result.
- **Cost**: total tokens versus the monolithic baseline. Multi-agent is usually more expensive —
  quantify what the money buys.

### Acceptance criteria

- [ ] Shared memory beats isolated agents on task success in a **paired** comparison, with the cost
      delta in tokens quantified.
- [ ] Handoff fidelity ≥ 80% end-to-end on injected facts, reported alongside the implied per-hop
      rate and split referenced/retold.
- [ ] A poisoned memory in the lowest-trust namespace changes **zero** higher-trust agents'
      behaviour, measured on the drift probe set, not by inspecting memory text.
- [ ] The trust-laundering attack is blocked, with a test that fails if min-trust inheritance is
      removed (mutation-tested, per §11.4).
- [ ] Concurrent-write policy documented, implemented and demonstrated under contention, with the
      losing write still visible in the audit trail.
- [ ] Agent disagreement is surfaced, not silently resolved — show the conflict surface with a real
      disagreement in it.
- [ ] A written analysis of what did **not** work and why. Required, not optional.

---

## 12.5 Reviewing your own capstone

Before you show anyone, score yourself against chapter 14's rubric. The mapping from capstone
evidence to a rubric score is the part people get wrong — a 3 is not "I built it", it is "I can show
the measurement".

| Rubric dimension (14.3) | What a **2** looks like | What a **3** looks like |
|---|---|---|
| Write path decomposition | Extraction and reconciliation separated, 4 ops | + typed policy, provenance, confidence, and the 12/12 suite with mutation score 5/5 |
| Temporal correctness ★ | Validity windows present | + Q2 ≠ Q3 demonstrated on your own data (§11.5) |
| Retrieval | Hybrid + fusion | + gate with measured skip rate, and the citation rate for injected memories |
| Isolation ★ | RLS or partitioning in place | + the unset-GUC fail-closed test and the partition-pruning check in CI |
| Deletion ★ | Derived memories regenerated | + residue sites enumerated and each proven empty, in a test |
| Injection safety ★ | Trust levels enforced in code | + drift monitoring and a mutation test on the trust rule |
| Evaluation | Offline eval set | + abstention category, CIs on every number, judge agreement reported |
| Cost awareness | Modelled | + largest line item identified with a reduction plan and the sensitivity |
| Operability | Full metric set | + runbook, kill switch and snapshot restore demonstrated live |
| Background tier | Consolidation | + reflection with the usefulness filter, contradiction sweep, decay, and a p95 win |

Two starred rows below 2 is not a nit; it is a blocker. And the profile 14.3 warns about — 3 on
retrieval, 0 on deletion — is the default outcome of a capstone that was fun to build, because
retrieval is the enjoyable part and deletion is the part nobody volunteers for. Check for it
explicitly.

---

## 12.6 How to present a capstone

For interviews, promotion packets, or a public writeup, package four artefacts:

1. **README** — the problem in three sentences, one architecture diagram, and the headline numbers
   as a triple (accuracy + CI, tokens/query, p95/p99). Nobody reads past this unless the numbers are
   there.
2. **Design doc** — §12.2's eleven sections, including *Alternatives considered*.
3. **Eval report** — per-category triple, baseline comparison, paired where comparative, judge
   agreement stated, and a section on what is still broken.
4. **A 10-minute demo including the failure cases.** Show the SLO holding while one arm is degraded;
   show the poisoned payload being rejected; show the deletion cascade emptying every site. A system
   whose limits you can demonstrate reads as engineering; a system with only a happy path reads as a
   tutorial.

The honest-limitations section is what distinguishes senior work. Every memory system has sharp
edges — knowing exactly where yours are, and saying so before you are asked, is the deliverable.

---

## 12.7 Failure modes of capstone projects

| Symptom | Root cause | Fix |
|---|---|---|
| Week 4 arrives with 12 half-built subsystems | no hours budget | 12.1 count subsystems × 8h before the task list |
| Load test says PASS, production p99 misses | test too short to estimate a tail | 12.2 ≥ 60,000 samples, report the p99's CI |
| "Shared memory is better" is unfalsifiable | 20 unpaired runs, overlapping CIs | 12.4 paired protocol, per-task differences |
| Codebase memories quietly go stale | TTL picked from a mean edit rate | 12.3 hash/symbol invalidation; churn is bursty |
| Invalidation throws away good memory | uniform TTL over concentrated churn | 12.3 top-10% of files carry 37–56% of edits |
| Poisoned namespace reaches the reviewer agent | derived memories keep the deriver's trust | 12.4 min-trust inheritance (85% → 0%) |
| Facts evaporate across agent handoffs | prose handoffs at p<0.93 per hop | 12.4 pass references, not retellings |
| Design doc reads as an advertisement | no *Alternatives considered*, no numbers | 12.2 required number per section |
| Deletion scored 0 while retrieval scored 3 | built the fun half | 12.5 check the starred rows first |
| Eval report assembled from memory at the end | instrumented late | 12.1 `memlab` on day one, weekly checkpoint |
| Reviewer asks "what is it bad at" and you stall | never characterised the query distribution | 12.6 honest-limitations section, written first |

---

## 12.8 Exercises

These are the things to do *before* week one, and each takes under two hours.

1. **Budget it.** Write the hours table for your chosen capstone: subsystems × 8h + eval + load rig +
   doc, against your real available hours. Acceptance: the ratio is a number, and if it exceeds 1.0
   you have a written cut list in §12.1's order.
2. **Size your load test.** Take any latency distribution (simulate it if the system does not exist
   yet) and compute the bootstrap CI of its p99 at n = 100 / 1,000 / 10,000 / 60,000. Acceptance: you
   can state the sample count your SLO criterion requires, and it is written into the criterion.
3. **Measure your own churn.** Run the §12.3 study on the repository you will target: staleness at
   7/30/90/180 days, edits per file per year, and the share of edits in the top 10% of files.
   Acceptance: a table in your design doc, and an invalidation strategy justified by it rather than
   by a default TTL.
4. **Derive your handoff requirement.** For your agent topology, compute the per-hop fidelity needed
   for an 80% end-to-end target. Acceptance: the number, plus a statement of which facts will travel
   as references rather than prose.
5. **Prove your containment rule matters.** Implement min-trust inheritance, then mutation-test it:
   remove the rule and confirm a test fails. Acceptance: the test name, and the measured blast radius
   with the rule on and off.
6. **Write section 11 first.** Draft *Alternatives considered* before you build, with the number that
   decided each choice. Acceptance: two rejected alternatives, each with a quantitative reason.
   Doing this first is how you discover in week one that one of them was the better design.
7. **Pre-commit your acceptance criteria.** Copy the criteria list for your capstone into the repo as
   a checklist file on day one. Acceptance: each item names its measurement protocol and sample size.
   Criteria written at the end are always shaped to the system you happened to build.

Next: `13-reading-list.md` — every source cited across this course, ordered by reading value, with
what to take from each.
