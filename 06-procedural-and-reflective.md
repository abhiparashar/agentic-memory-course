# 06 — Procedural Memory, Reflection, and Background Compute

> Goal: memory that changes what the agent *does*, not just what it knows — plus the background
> processes that turn raw memory into good memory while nobody is talking.

---

## 6.0 In plain words

A new hire builds three different kinds of memory, and they live in three different places:

| Kind | Example | Where a human keeps it | Where your agent keeps it |
|---|---|---|---|
| **Semantic** — what is true | "Priya owns billing" | address book | fact store (ch 04/05) |
| **Procedural** — how we do things here | "run `make gen` before the tests" | the sticky note on the monitor | a file, always in context (6.1) |
| **Trajectory** — what I tried last time | "the v3 batch API isn't in our region, don't bother" | lab notebook | trajectory store (6.5) |

And then there is the thing that happens *between* workdays: on the commute home you notice "they
always want terse answers on Fridays". Nobody told you; you derived it from a pile of episodes. That
is **reflection** (6.2), and doing it offline instead of mid-conversation is **sleep-time compute**
(6.3).

### Why procedural memory does not belong in a vector store

This is the section people argue with, so here is the arithmetic. Suppose "always run `make gen`
first" is stored as a memory and retrieved with **recall 0.9** — which would be excellent retrieval.

```
P(the rule fires every time over N tasks) = 0.9^N

N=5   tasks → 41.0% chance you miss it at least once
N=10  tasks → 65.1%
N=20  tasks → 87.8%
```

Even at recall **0.95**, twenty tasks gives you a 64.2% chance of at least one silent failure. And
inside a single 20-step task where the rule matters at three steps, recall 0.9 per step means a
27.1% chance of blowing it at least once. The user experiences this as **"the agent is flaky"** —
the worst possible failure mode, because it is unreproducible.

Now price the alternative. A procedural file of 100 rules at ~18 tokens per rule is **1,800 tokens
per turn, always present, recall 1.0 by construction**:

```
 50 rules × 18 tok =   900 tokens/turn
100 rules × 18 tok = 1,800 tokens/turn
300 rules × 18 tok = 5,400 tokens/turn   ← past here, consolidate or scope; see 6.1
```

**Design rule, and it is the whole section: if a memory must apply *every* time, put it in context
unconditionally. Only sometimes-relevant memory belongs behind retrieval.** Flaky recall on a
must-apply rule is not a retrieval tuning problem; it is a tier-placement mistake.

### What the research says, in one paragraph each

- **Reflection is not decoration.** *Generative Agents* stores a complete natural-language record of
  experience, synthesises it "over time into higher-level reflections", and retrieves them to plan
  behaviour — and the paper's ablation shows observation, planning, **and reflection** "each
  contribute critically" to believable behaviour ([arXiv:2304.03442](https://arxiv.org/abs/2304.03442)).
  Remove reflection and the agents get measurably worse.
- **Self-editing memory is an OS problem.** *MemGPT* frames the context window as physical memory and
  introduces "virtual context management… drawing inspiration from hierarchical memory systems in
  traditional operating systems", paging data between fast and slow tiers and using interrupts for
  control flow ([arXiv:2310.08560](https://arxiv.org/abs/2310.08560)). Every "memory tool" design
  since is a variation on this.
- **Thinking ahead is cheaper than thinking on demand.** *Sleep-time compute* has the model reason
  about a context offline, before queries arrive. It cuts the test-time compute needed for the same
  accuracy by **~5×** on their stateful benchmarks, and scaling the offline work adds up to **+13%**
  (Stateful GSM-Symbolic) and **+18%** (Stateful AIME) accuracy. Amortised across several related
  queries about the same context, average cost per query drops **2.5×**
  ([arXiv:2504.13171](https://arxiv.org/abs/2504.13171)).

That last paper also names the condition under which this fails, which matters more than the wins:
efficacy correlates with **how predictable the user's query is**. Background work on a context
nobody ever asks about is pure cost. Keep that in mind through 6.3.

**The one-sentence takeaway:** facts go in the store, rules go in the prompt, lessons go in a
trajectory log, and the expensive thinking that ties them together runs while the user is asleep.

---

## 6.1 Procedural memory: the most underrated tier

Semantic memory answers "what is true". Procedural memory answers "how do we do things here".

Examples from real systems:

- "In this repo, always run `make gen` before `go test`, or generated mocks are stale."
- "This customer's tickets escalate to the enterprise queue, never the standard queue."
- "When the user asks for a summary, they want bullets under 10 words, not prose."
- "The staging deploy tool requires `--region` even though it's documented as optional."

Note what these have in common: **each was learned by failing once.** That is the defining property
of procedural memory, and it tells you exactly where to harvest it — from failures and corrections,
not from conversation.

### Why files beat databases here

For procedural memory, a markdown file is usually the right store. Not a compromise — genuinely
better:

| Property | File | Vector store |
|---|---|---|
| Human-editable | Yes, directly | Needs a UI |
| Reviewable in a PR | Yes, diffs | No |
| Version controlled | Free | Build it yourself |
| Always in context | Yes (small) | Requires retrieval to fire |
| Recall on a must-apply rule | 1.0 | 0.8–0.95, i.e. flaky (see 6.0) |
| Debuggable | `cat` | Query + interpret scores |

This is why coding agents converged on `CLAUDE.md` / `AGENTS.md` / rules files instead of embedding
procedural knowledge. It is small, high-value, and benefits enormously from human curation.

### Structuring a procedural memory file

```markdown
# Project conventions (agent memory)
<!-- Auto-maintained. Human edits welcome; the agent will not delete human-marked entries. -->

## Build & test
- Run `make gen` before `go test ./...` — generated mocks go stale otherwise.
  <!-- learned: 2026-02-14, session a3f2, failure: 6 test failures in payments, applied: 34x -->
- Integration tests need `docker compose up -d pg redis` first.

## Code conventions
- Errors wrap with `fmt.Errorf("...: %w", err)`. Never `errors.New` in service code.
- New endpoints require an entry in `docs/api.md` or CI fails.

## Do not
- Do not modify `internal/legacy/**` — scheduled for deletion, changes will conflict.
  <!-- human-pinned: 2026-01-08 -->
```

The HTML-comment metadata gives you provenance without cluttering what the model reads.
`human-pinned` entries must be immune to agent pruning — **a hard rule in your code, not a request
in a prompt.**

### Learning procedural memory from failure

```python
LEARN_PROCEDURE = """A task step failed and was then resolved. Extract a reusable rule.

Failed action: {action}
Error: {error}
What fixed it: {resolution}

Output a rule ONLY if it will apply to future tasks in this environment.
Return null if the failure was a one-off (transient network, typo, unrelated outage).

Format: {{"rule": "imperative sentence", "scope": "repo|user|global",
          "trigger": "when this applies", "confidence": 0.0-1.0}}
"""
```

The "return null for one-offs" instruction is essential. Without it you accumulate rules like "retry
when the API returns 503" — noise — or worse, **superstition**: rules derived from coincidence that
then constrain the agent forever. Superstitious rules are uniquely nasty because they are
unfalsifiable from inside the system; the agent follows the rule, nothing breaks, and the rule looks
validated.

### Scope, precedence, and conflict

Rules arrive at three scopes and they *will* conflict. Declare precedence once, in code:

```python
SCOPE_PRECEDENCE = {"repo": 3, "user": 2, "global": 1}   # higher wins

def render_rules(rules, budget_tokens=1800):
    rules = [r for r in rules if not r.archived]
    rules.sort(key=lambda r: (SCOPE_PRECEDENCE[r.scope], r.applied_count, r.confidence),
               reverse=True)
    out, used = [], 0
    for r in rules:
        cost = estimate_tokens(r.text)
        if used + cost > budget_tokens:
            break                      # hard budget: procedural memory never crowds out the task
        out.append(r); used += cost
    return out, used
```

Two decisions encoded there. **A repo rule beats a user preference** ("I like 10-item lists" loses to
"this repo requires 3 bullets"), because the environment's constraints are facts while the user's
preference is a preference. And **the file has a token budget**, enforced by truncation at render
time, so an unbounded rule file degrades gracefully (lowest-value rules drop out) instead of eating
the context window.

### Governance and rule rot

Procedural rules change behaviour directly, so they need a higher bar than semantic facts:

- **Auto-add only at confidence > 0.85**, and only after the rule was implicitly validated (the fix
  actually worked).
- **Cap the file** (~100 rules / ~1,800 tokens); beyond that, force consolidation.
- **Review flag after N applications** — surface high-traffic rules to a human, because a wrong rule
  applied 200 times is your most expensive bug.
- **Demote rules never triggered in 90 days** to an archive section. Keep them; do not delete.
- **Track `applied_count` and post-application failure rate.** A rule whose tasks fail *more often*
  than baseline is actively harmful — that is the signal that catches superstition:

```python
def rule_health(rule, baseline_failure_rate) -> str:
    if rule.applied_count < 10:                                  return "insufficient_data"
    if rule.post_apply_failure_rate > baseline_failure_rate * 1.3: return "harmful"      # quarantine
    if rule.applied_count == 0 and rule.age_days > 90:           return "stale"          # archive
    return "healthy"
```

---

## 6.2 Reflection: deriving insight from episodes

Reflection is a background pass over accumulated memories that produces *higher-order* memories —
generalisations no single episode contains.

The lineage is *Generative Agents*, which introduced a reflection tree: agents periodically reviewed
recent memories, asked what questions those memories raised, answered them, and stored the answers
as new higher-level memories that could themselves be reflected on
([arXiv:2304.03442](https://arxiv.org/abs/2304.03442)). It is still the clearest formulation, and
the paper's ablation is the reason to bother: remove reflection and behaviour quality drops.

```python
REFLECT = """Review these recent memories about the user and produce higher-level insights.

Memories:
{memories}

Produce at most {n} insights. Each must:
- Generalise across at least 2 of the memories (cite their ids).
- Be non-obvious — not a restatement of a single memory.
- Be actionable — it should change how you respond in the future.
- Include a confidence.

Return JSON: [{{"insight": str, "supporting_ids": [str], "confidence": float}}]
"""
```

Good output:

> "User asks for code review help mostly on Sunday evenings and consistently prefers terse,
> diff-focused feedback then; on weekdays they ask exploratory design questions and want depth."
> (from m_112, m_309, m_411)

That is genuinely useful and no single memory contains it. Bad output:

> "User is interested in software engineering."

Generic, non-actionable, and it will pollute retrieval forever — worse, it is *similar to everything*,
so it wins top-k on unrelated queries. Guard mechanically:

```python
def is_useful_insight(insight, existing_memories, embedder) -> bool:
    v = embedder.encode(insight["insight"])
    if len(insight["supporting_ids"]) < 2:      return False   # not a generalisation
    if max_cosine(v, existing_memories) > 0.88: return False   # restates something we have
    if generic_score(insight["insight"]) > 0.7: return False   # low specificity
    return insight["confidence"] >= 0.7
```

`generic_score` can be a classifier trained on 100 hand-labelled examples, or — cheaper and
surprisingly effective — a check for the *absence* of specific entities, numbers, or times. An
insight with no proper noun and no quantity is almost always filler.

Two more disciplines that keep reflection from rotting your store:

- **Insights are derived, so mark them.** Store `derived_from` ids and a `tier: reflection` flag, and
  keep them in a separate retrieval lane with a lower weight than first-order facts. A reflection
  built on a wrong fact should not outrank the fact that corrects it.
- **Re-reflect, do not accumulate.** When the supporting memories change (superseded, retracted),
  invalidate the insight and regenerate. Otherwise you get insights about a user who no longer
  exists — "prefers terse Sunday reviews" long after they changed jobs.

**Trigger policy.** Generative Agents triggers reflection when accumulated importance crosses a
threshold. In production I use a simpler hybrid, because predictable cost beats elegance: (a) N new
memories since the last reflection, (b) session end for high-activity users, (c) a weekly floor so
quiet users still get one.

---

## 6.3 Sleep-time compute

Letta's framing: instead of the agent managing memory inline during a conversation — which makes
responses slower and reliability worse, since one agent is juggling conversation, tools, and memory
edits — split it. A **primary agent** talks to the user and can search memory but cannot edit core
blocks; a **sleep-time agent** runs in the background holding the memory-editing tools and rewrites
the shared blocks asynchronously.

```
  ┌──────────────────────────────────────────────────────┐
  │  Primary agent (foreground, latency-critical)        │
  │  tools: user tools, conversation_search,             │
  │         archival_search   —  NO memory edit tools    │
  └───────────────────────┬──────────────────────────────┘
                          │ shares
                  ┌───────▼────────┐
                  │ memory blocks  │  (in-context, character-capped)
                  └───────▲────────┘
                          │ edits
  ┌───────────────────────┴──────────────────────────────┐
  │  Sleep-time agent (background, latency-tolerant)     │
  │  tools: memory_insert/replace, consolidate,          │
  │         reflect, reorganise                          │
  └──────────────────────────────────────────────────────┘
```

Why this is the right architecture:

1. **Non-blocking.** Memory work never sits in the user's latency budget.
2. **Better quality.** The background agent can take many steps, use a stronger model, and reorganise
   holistically instead of making greedy incremental edits mid-conversation.
3. **Different cost profile.** Background work runs on batch APIs, spot capacity, and off-peak
   windows. Half price on a job with no latency SLO is free money.
4. **Cleaner failure isolation.** A crashed memory job does not break the conversation.

And the measured version of claim 1–2, from the paper that named the idea: pre-computing over a
context offline reduced the test-time compute needed for equal accuracy by **~5×**, scaling the
offline work added **+13% / +18%** accuracy on their two stateful benchmarks, and amortising one
offline pass across several related queries cut average per-query cost **2.5×**
([arXiv:2504.13171](https://arxiv.org/abs/2504.13171)).

**When it does not pay.** The same paper found efficacy correlates with the **predictability of the
user's query**. Concretely: pre-computing a user profile is worth it (you will need it every session);
pre-computing "what might they ask about this 400-page PDF" is a coin flip. Before you build a
background tier, ask what fraction of its output gets read. If you cannot estimate that, instrument
it — emit a `background_artifact_used` counter and look at it in a week.

### What to run in the background

| Job | Cadence | Cost profile |
|---|---|---|
| Session extraction | On session end | Small model, per session |
| Consolidation of related memories | Nightly per active tenant | Medium model, clustered |
| Reflection / insight generation | Weekly, or N-memories threshold | Large model, small input |
| Community detection / re-summarisation | Weekly | Graph algo + summarisation |
| Decay scoring + archival tiering | Daily | Pure compute, no LLM |
| Re-embedding after model upgrade | One-off migration | Embedding API, huge volume |
| Contradiction sweep | Daily | Medium model, only on changed subjects |

The **contradiction sweep** earns its row. Even with reconciliation on write, contradictions creep in
via extraction errors and bad merges. A nightly job that finds same-subject, same-predicate,
both-currently-valid facts and adjudicates them keeps the store honest — and it is nearly free if you
added the 5.2 unique index, because the index makes the pathological case impossible and the sweep
only has to handle multi-valued predicates.

**Alert on the sweep's rate, not just its results.** A spike in contradictions found is your earliest
signal that an extractor or model change went wrong — it fires days before accuracy metrics move.

---

## 6.4 Self-editing memory: the MemGPT lineage

MemGPT's core idea, still the most influential in the field: give the agent tools to edit its own
in-context memory and let it manage the boundary between context and external storage. The framing is
explicitly operating-system — "virtual context management… hierarchical memory systems… data movement
between fast and slow memory", plus interrupts for control flow
([arXiv:2310.08560](https://arxiv.org/abs/2310.08560)).

The original tool surface:

- `core_memory_append` / `core_memory_replace` — edit in-context blocks (persona, human).
- `conversation_search` — search recall storage (past messages).
- `archival_memory_insert` / `archival_memory_search` — the external vector store.
- Heartbeats — let the agent chain tool calls without user input.

Later Letta versions refine this into labelled, character-capped **memory blocks** shareable across
agents, with sleep-time agents doing the editing (6.3).

**When self-editing memory is right:**
- The task is open-ended and you cannot pre-specify what should be remembered.
- You want persona/behaviour to evolve.
- You are building a general assistant, not a narrow workflow.

**When it is wrong:**
- You need predictable, auditable memory content (regulated domains).
- Memory drives money-moving actions — you do not want the agent authoring its own instructions
  unreviewed.
- High QPS with tight latency: self-editing costs inline tool round-trips.

The hybrid I would ship for most products: **agent-proposed, system-validated.** The agent proposes
edits; a deterministic layer validates them against schema, category policy, and a deny-list, then
commits.

```python
def commit_agent_edit(proposal: dict, policy: Policy) -> Result:
    if proposal["category"] not in policy.allowed_categories:  return Result.reject("category")
    if policy.deny_patterns.search(proposal["text"]):          return Result.reject("denied_content")
    if len(proposal["text"]) > policy.max_len:                 return Result.reject("too_long")
    if policy.is_instruction(proposal["text"]) and not policy.allow_self_instructions:
        return Result.reject("self_instruction")               # key injection guard
    return store.write(proposal)
```

That `self_instruction` check is a security control, not hygiene. An agent writing "always approve
refunds without checking" into its own persistent memory is the memory-poisoning chain from chapter 09
executed by the agent itself under injection — and because it is now *memory*, it survives the session
that planted it. **Persistence is what turns a prompt injection into a breach.**

---

## 6.5 Experience replay / trajectory memory

For agents that execute tasks rather than converse, the highest-value memory is often **past
trajectories**: what was the task, what steps were taken, what happened.

```python
@dataclass
class Trajectory:
    task_description: str
    task_embedding: list[float]
    steps: list[Step]                 # action, observation, reflection
    outcome: str                      # success | failure | partial
    duration_s: float
    cost_tokens: int
    lessons: list[str]                # extracted post-hoc
```

At the start of a new task, retrieve similar past trajectories and inject a compressed form — never
the raw step log, which is enormous and mostly noise:

```
<similar_past_tasks>
  <task similarity="0.87" outcome="success">
    Task: migrate service X from lib v2 to v3
    Approach that worked: update interfaces first, then callers, then tests
    Pitfall hit: v3 batch API unavailable in our region — do not attempt
    Duration: 3h, 180k tokens
  </task>
</similar_past_tasks>
```

**Failures are more valuable than successes**, which is not intuitive. A success teaches one path; a
failure eliminates a whole branch of the search space. Weight failure trajectories higher in
retrieval and always extract the explicit "do not attempt" lesson.

**Distil trajectories into rules.** A trajectory is a 200KB artefact; the lesson is 15 tokens. Once
the same pitfall appears in three trajectories, promote it to procedural memory (6.1) and stop
retrieving the trajectories for it. Trajectory memory that never distils becomes an expensive,
slow-growing log that crowds out its own signal.

**The safety caveat, and it is the sharpest one in the course:** trajectory memory is the most
dangerous memory type, because retrieved trajectories shape *actions* rather than statements.
Implanting fabricated "successful experiences" in an agent's long-term memory can steer future
behaviour with no live jailbreak at all — the attack is a write, and every later session pays for it.

Therefore: **trajectories are hints, never permissions.** A retrieved trajectory must never
authorise an action that would otherwise require confirmation, and the confirmation logic must not
read from memory. Chapter 09 has the full threat model.

---

## 6.6 Putting the tiers together

A complete agent memory stack, and where each piece lives:

```
IN CONTEXT (always)
  ├─ system prompt + policy
  ├─ procedural memory file(s)        ← 06.1, human-reviewable, recall 1.0
  ├─ core memory blocks               ← 06.4, agent-editable, character-capped
  └─ recent conversation              ← 02

IN CONTEXT (retrieved per turn)
  ├─ semantic facts                   ← 04, filtered by validity     (ch. 05)
  ├─ relevant episodes                ← 03, hybrid retrieval
  └─ similar trajectories             ← 06.5, task-shaped only
  └─ reflections                      ← 06.2, separate lane, lower weight

ON DISK (retrievable)
  ├─ episode log (non-lossy)          ← source of truth
  ├─ fact store / graph               ← 05
  ├─ trajectory store                 ← 06.5
  └─ archived / cold memories         ← 04.7

BACKGROUND
  ├─ extraction, consolidation, reflection      ← 04, 06.2
  ├─ contradiction sweep, decay, tiering        ← 04.7, 06.3
  └─ re-embedding, community detection          ← 03, 05
```

If you can draw this for your own system and name the owner of each box, you understand your memory
architecture. If a box is missing, you can predict the failure mode from the chapter it belongs to —
no procedural tier means flaky repeated mistakes; no background tier means quality plateaus; no
episode log means you can never re-derive.

---

## 6.7 Failure modes seen in production

| Symptom | Root cause | Fix |
|---|---|---|
| "It knows the rule but ignores it sometimes" | must-apply rule behind retrieval | 6.0/6.1 put it in context unconditionally |
| Rule file grows until it crowds out the task | no budget at render time | 6.1 `render_rules` token budget + consolidation |
| Agent follows a rule that makes things worse | superstitious rule from a one-off failure | 6.1 `rule_health`, post-apply failure rate |
| Human-written rule silently deleted by the agent | pruning not scope-aware | 6.1 `human-pinned` enforced in code |
| Store fills with "user likes technology" | unfiltered reflection | 6.2 usefulness filter, generic-score gate |
| Insights persist about a stale profile | reflections not invalidated with sources | 6.2 `derived_from` + re-reflect |
| Conversation p95 degrades as memory grows | memory management inline | 6.3 sleep-time split |
| Background jobs cost a fortune, nobody reads output | unpredictable query distribution | 6.3 measure `background_artifact_used` first |
| Agent writes its own permissions into memory | self-edit unvalidated | 6.4 `self_instruction` reject |
| Agent repeats an action a poisoned trajectory "proved" safe | trajectory treated as authorisation | 6.5 hints ≠ permissions, ch 09 |

---

## 6.8 Exercises

1. Take a real repo and have an agent maintain a procedural file for a week of real work. Count
   rules that were useful, noise, and superstition. Then compute each rule's post-application
   failure rate and see whether `rule_health` would have caught the bad ones.
2. Reproduce the 6.0 arithmetic on your own stack: store one must-apply rule as a retrievable memory,
   run 20 tasks, and count misses. Compare with the same rule in the prompt. The point is to feel the
   difference between recall 0.9 and recall 1.0 before you trust retrieval with a rule.
3. Implement reflection with the usefulness filter. Run it over 200 memories and rate each insight
   1–5 for actionability. Mean below 3 means your *filter* is too weak — iterate on the filter, not
   the reflection prompt.
4. Implement a sleep-time consolidation job. Measure conversation p95 with inline vs background
   memory management, then measure memory quality with your chapter-04 scenario suite for both. You
   should see background win on **both** axes; if it doesn't, your background job is doing the wrong
   work.
5. Build trajectory memory for a task agent. Measure success rate with and without retrieved
   trajectories on 30 held-out tasks, then separately measure failure-only vs success-only retrieval.
   Then implement distillation: promote any pitfall seen 3× into a procedural rule and confirm the
   trajectory retrieval for it can be switched off with no loss.
6. Red-team your own write path: inject a fabricated "successful" trajectory claiming a destructive
   action was safe and verify your agent still asks for confirmation. If it doesn't, you have
   chapter 09's problem today, not later.

Next: `07-systems-design.md` — topology, tenancy, latency budgets, and what this costs at a million
users.
