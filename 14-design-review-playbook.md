# 14 — Design Review Playbook

> The questions I actually ask when someone brings a memory design to review, the answers that
> indicate depth, and a rubric.
>
> Use this two ways: to pressure-test your own design before review, and to prepare for system-design
> interviews on this topic (it is now a common one).

---

## 14.1 The opening question

> **"Walk me through what happens between the user saying 'I'm vegetarian' and the agent, three weeks
> later, recommending a restaurant."**

This one question exercises the whole system. What I listen for:

**Weak answer:** "We embed the message and store it, then do similarity search later." Collapses five
decisions into one and reveals no model of the write path.

**Strong answer** traces: salience decision → extraction into an atomic typed claim with provenance →
dedup and reconciliation against existing memories → async indexing with a freshness SLO → three
weeks later, a retrieval gate fires → query construction (noting that "recommend a restaurant"
contains no lexical overlap with "vegetarian", so slot-based retrieval or expansion is required) →
tenant-scoped hybrid retrieval → fusion → budget → rendering with validity and a data-not-instructions
fence.

The "no lexical overlap" observation is the tell. Candidates who spot it have actually built this;
candidates who don't have read about it.

---

## 14.2 The core question set

### On the write path

1. **What decides that something is worth remembering?** (Looking for: a typed policy, not "the LLM
   decides".)
2. **What happens when the user contradicts a stored memory?** (Looking for: supersession distinct
   from retraction; "outdated ≠ wrong".)
3. **Show me the ADD vs UPDATE vs DELETE decision.** (Looking for: DELETE reserved for facts that were
   never true.)
4. **Is extraction synchronous?** (Should be no. Follow-up: how do you handle read-your-writes within
   a session? Correct answer: session memory covers it; long-term only needs to be right by the next
   session.)
5. **What is your freshness SLO and how do you measure it?** (Looking for: `episode_ts →
   indexed_ts`, p99, monitored.)
6. **How do you prevent duplicate accumulation?** (Looking for: write-time dedup, and a measured
   duplicate rate.)

### On the read path

7. **Do you retrieve on every turn?** (Looking for: a gate, with a measured skip rate and a measured
   cost of skipping wrongly.)
8. **How many memories do you inject, and how do you know that is the right number?** (Looking for:
   the usage/citation metric from 07.9. If they have never measured whether injected memories are
   used, that is the single biggest gap in most designs.)
9. **What happens when retrieval returns contradictory facts?** (Looking for: validity filtering
   before ranking, and rendering with timestamps.)
10. **Dense only, or hybrid?** (Looking for: BM25 for identifiers, and awareness that short memory
    texts are exactly where dense retrieval is weakest.)
11. **What is your latency budget and what gets dropped first under pressure?** (Looking for: a
    documented skip order and deadline propagation.)

### On systems

12. **How is tenant isolation enforced?** (Looking for: something below the application layer — RLS or
    physical partitioning — plus a CI test.)
13. **What happens to the tenant with 500k memories?** (Looking for: working-set caps, separate
    background queue lanes, explicit product-level quota.)
14. **How do you re-embed when you change embedding models?** (Looking for: `model_id` per vector,
    dual-serving, per-tenant cutover, rollback.)
15. **What is the single largest line item in your cost model?** (Correct answer at most scales:
    injected tokens. If they say "the vector database", they have not built the spreadsheet.)
16. **Do you need a graph?** (Looking for: an honest answer. "Our queries are single-hop facts about
    one user, so no" is a *strong* answer.)

### On safety

17. **Can content from a web page the agent read become a durable memory?** (Looking for: trust levels
    and a hard rule that untrusted content cannot become an instruction-stance memory.)
18. **Can a memory reduce a confirmation requirement?** (Must be an immediate, unqualified no.)
19. **Walk me through deleting one fact.** (Looking for: the full cascade — vectors across all model
    ids, lexical index, graph edges, derived memories *regenerated* not just unlinked, session
    summaries, caches, analytics, backups.)
20. **How would you detect that a tenant's memory has been poisoned?** (Looking for: behavioural drift
    monitoring, not content scanning — sleeper entries look benign in isolation.)
21. **Can you turn memory off right now, without a deploy?** (Must be yes, at global, tenant, and
    category granularity.)

### On evaluation

22. **How do you know it works?** (Looking for: the triple — accuracy, tokens/query, p95 latency — and
    an eval set built from their own traffic.)
23. **What fraction of your eval set is abstention?** (Looking for: ~15%. Zero means they will ship a
    confabulator.)
24. **What is in CI?** (Looking for: write-path scenarios, isolation, deletion cascade at 100%, plus
    retrieval regression thresholds.)
25. **What broke last time you changed the extractor prompt?** (Looking for: any answer at all. "We
    don't know" means there is no regression suite.)

---

## 14.3 Rubric

Score each dimension 0–3. Below 2 on any starred row is a blocker, not a nit.

| Dimension | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| Write path decomposition | One LLM call does everything | Extraction separated | + reconciliation with 4 ops | + typed policy, provenance, confidence |
| Temporal correctness ★ | No time modelling | `created_at` only | Validity windows | Bi-temporal, point-in-time queries |
| Retrieval | Dense only, fixed k | + filters | Hybrid + fusion | + rerank, gate, measured usage |
| Isolation ★ | App-level filter | + code review discipline | RLS or partitioning | + CI test + prod audit sampling |
| Deletion ★ | Row delete | + vectors | + derived regenerated | + caches, analytics, backups, tested |
| Injection safety ★ | None | Prompt fence | + trust levels enforced in code | + drift monitoring + rollback |
| Evaluation | Vibes | Manual spot checks | Offline eval set | + abstention, CI gates, online metrics |
| Cost awareness | Unknown | Rough estimate | Modelled | + largest line item identified with a plan |
| Operability | No metrics | Basic latency | Full metric set | + runbook, kill switch, snapshots |
| Background tier | None | Summarisation | + consolidation | + reflection, contradiction sweep, decay |

A design scoring 2 across the board is shippable. A design scoring 3 on retrieval and 0 on deletion
is a lawsuit, and I have seen that exact profile more than once — retrieval is the fun part and
deletion is the part nobody volunteers for.

---

## 14.4 Red flags

Phrases that should trigger deeper questions:

- **"The LLM decides what to remember."** Fine as one component; alarming as the whole policy.
- **"We just use cosine similarity."** No hybrid, no filtering, no temporal handling.
- **"We store everything and retrieve top-20."** No salience policy, and top-20 is over-retrieval that
  degrades quality via context rot.
- **"Deletion just marks a row."** The cascade is missing.
- **"We'll add evaluation later."** It will not be added later, and by then no one will know whether
  changes help.
- **"Memory is in the system prompt so it's trusted."** Directly inverted threat model.
- **"We picked [framework] because it scored highest on [benchmark]."** Follow up on actor model,
  judge, and token cost. Usually the comparison is not apples to apples.
- **"It's basically RAG."** Reveals no model of the write path, which is the hard half.

Green flags, conversely:

- They can state the query distribution their design is optimised for, and what it is bad at.
- They have a number for how often injected memories are actually used.
- They chose *not* to use a graph, with a reason.
- They can describe a failure they had and the test they added afterwards.

---

## 14.5 Pressure-test your own design

Before review, answer these in writing. If any answer is longer than a paragraph or shorter than a
sentence, that section of your design is not ready.

1. Draw the chapter-06.7 tier diagram for your system, naming the owning module for each box.
2. For each of the 25 questions above, write your answer.
3. Fill in the 07.10 failure table for your architecture.
4. Compute your cost model and identify the largest line item.
5. Write the deletion cascade as an enumerated list of every place a fact leaves a trace. Then go find
   the ones you missed by grepping for where memory text is written.
6. List the three query types your system will handle worst, and what you would build if they became
   important.

That last one is what senior reviewers care about most. Every design has a shape it is bad at.
Knowing yours — and saying so unprompted — is the difference between a design review and a defence.

---

## 14.6 A closing note

The systems in this course are not hard because the ideas are hard. Extraction, retrieval, and
validity windows are all conceptually simple. They are hard because memory is **stateful, long-lived,
personal, and adversarially reachable** — four properties that individually make systems difficult and
jointly make them unforgiving.

The engineering discipline that matters, in priority order:

1. **Provenance on everything.** Every memory knows where it came from. This one property makes
   deletion, audit, debugging, citation, and re-derivation possible. Skipping it is the decision you
   will most regret.
2. **Never destroy the source.** Facts are derived; episodes are truth. You will want to re-extract
   with a better model, and you can only do that if you kept the raw material.
3. **Measure before optimising.** The triple, always. Most systems are injecting far more memory than
   the model uses, and nobody knows because nobody measured.
4. **Assume the memory is wrong.** Design read paths so that a bad memory degrades an answer rather
   than corrupting an action.

Get those four right and the rest is refinement.
