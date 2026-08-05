# 06 — Procedural Memory, Reflection, and Background Compute

> Goal: memory that changes what the agent *does*, not just what it knows. Plus the background
> processes that turn raw memory into good memory.

---

## 6.1 Procedural memory: the most underrated tier

Semantic memory answers "what is true". Procedural memory answers "how do we do things here".

Examples from real systems:

- "In this repo, always run `make gen` before `go test`, or generated mocks are stale."
- "This customer's tickets should be escalated to the enterprise queue, never the standard queue."
- "When the user asks for a summary, they want bullets under 10 words, not prose."
- "The staging deploy tool requires `--region` even though it's documented as optional."

Note what these have in common: **each one was learned by failing once.** That is the defining
property of procedural memory and it tells you exactly where to harvest it — from failures and
corrections.

### Why files beat databases here

For procedural memory, a markdown file is usually the right store. Not a compromise — genuinely
better:

| Property | File | Vector store |
|---|---|---|
| Human-editable | Yes, directly | Needs UI |
| Reviewable in PR | Yes, diffs | No |
| Version controlled | Free | Build it yourself |
| Always in context | Yes (small) | Requires retrieval to fire |
| Debuggable | `cat` | Query + interpret scores |

This is why coding agents converged on `CLAUDE.md` / `AGENTS.md` / rules files rather than embedding
their procedural knowledge. Procedural memory is small (hundreds of lines), high-value, and benefits
enormously from human curation. Retrieval failure on a procedural rule is catastrophic — "always run
`make gen`" that fails to retrieve 20% of the time gives you a flaky agent — whereas an always-loaded
file has 100% recall by construction.

**Design rule: if a memory must apply every time, put it in context unconditionally. Only
sometimes-relevant memory belongs behind retrieval.**

### Structuring a procedural memory file

```markdown
# Project conventions (agent memory)
<!-- Auto-maintained. Human edits welcome; the agent will not delete human-marked entries. -->

## Build & test
- Run `make gen` before `go test ./...` — generated mocks go stale otherwise.
  <!-- learned: 2026-02-14, session a3f2, failure: 6 test failures in payments -->
- Integration tests need `docker compose up -d pg redis` first.

## Code conventions
- Errors wrap with `fmt.Errorf("...: %w", err)`. Never `errors.New` in service code.
- New endpoints require an entry in `docs/api.md` or CI fails.

## Do not
- Do not modify `internal/legacy/**` — scheduled for deletion, changes will conflict.
  <!-- human-pinned: 2026-01-08 -->
```

The HTML-comment metadata gives you provenance without cluttering what the model reads. `human-pinned`
entries should be immune to agent pruning — a hard rule in your code, not a request in a prompt.

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

The "return null for one-offs" instruction is essential. Without it you accumulate rules like
"retry when the API returns 503", which is noise, or worse, superstition — rules derived from
coincidence that then constrain the agent forever.

**Governance:** procedural rules directly change behaviour, so they need a higher bar than semantic
facts. My default policy:
- Auto-add only at confidence > 0.85, and only after the rule has been implicitly validated (the fix
  worked).
- Cap the file size (~100 rules); force consolidation beyond that.
- Every rule gets a review flag after N applications; surface it to the human.
- Rules that were never triggered in 90 days get demoted to an archive section.

---

## 6.2 Reflection: deriving insight from episodes

Reflection is a background pass over accumulated memories that produces *higher-order* memories —
generalisations that no single episode contains.

The lineage here is the *Generative Agents* paper (Park et al., 2023), which introduced a reflection
tree: agents periodically reviewed recent memories, asked themselves what questions those memories
raised, answered them, and stored the answers as new higher-level memories that could themselves be
reflected on. It remains the clearest formulation.

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

Good output looks like:
> "User asks for code review help mostly on Sunday evenings and consistently prefers terse,
> diff-focused feedback then; on weekdays they ask exploratory design questions and want depth."
> (from ids m_112, m_309, m_411)

That is genuinely useful and no single memory contains it.

Bad output looks like:
> "User is interested in software engineering."

Generic, non-actionable, and it will pollute retrieval forever. Guard against it:

```python
def is_useful_insight(insight, existing_memories, embedder) -> bool:
    v = embedder.encode(insight["insight"])
    if len(insight["supporting_ids"]) < 2:                     return False
    if max_cosine(v, existing_memories) > 0.88:                return False  # restates something
    if generic_score(insight["insight"]) > 0.7:                return False  # low-specificity
    return insight["confidence"] >= 0.7
```

`generic_score` can be as simple as a classifier trained on 100 hand-labelled examples, or a check
for the absence of specific entities/numbers/times. Do not skip it — unfiltered reflection is the
fastest way to fill your store with plausible-sounding nothing.

**Trigger policy.** The Generative Agents approach triggers reflection when accumulated importance
scores cross a threshold. In production I use a simpler hybrid: trigger on (a) N new memories since
last reflection, (b) session end for high-activity users, and (c) a weekly floor. Cost is
predictable and that matters more than elegance.

---

## 6.3 Sleep-time compute

Letta's framing: instead of the agent doing memory management inline during a conversation — which
makes responses slower and reliability worse, since one agent is juggling conversation, tool use, and
memory edits — you split it. A **primary agent** talks to the user and can search memory but cannot
edit its core memory blocks; a **sleep-time agent** runs in the background, holds the memory-editing
tools, and rewrites the shared memory blocks asynchronously.

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
2. **Better quality.** The background agent can take many steps, use a stronger model, and
   reorganise holistically rather than making greedy incremental edits mid-conversation.
3. **Different cost profile.** Background work can run on spot capacity, batch APIs, and off-peak
   windows. A batch API at half price on a job with no latency SLO is an easy win.
4. **Cleaner failure isolation.** A crashed memory job does not break the conversation.

The same idea appears everywhere once you look: background indexing in a search engine, compaction in
an LSM tree, GC in a runtime. **Foreground reads, background reorganisation.** If your memory system
has no background tier, it will plateau.

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

The **contradiction sweep** is worth calling out. Even with reconciliation on write, contradictions
creep in via extraction errors and merge mistakes. A nightly job that finds same-subject,
same-predicate, both-currently-valid facts and adjudicates them keeps the store clean. Alert on the
rate: a spike is your early warning that an extractor change went wrong.

---

## 6.4 Self-editing memory: the MemGPT lineage

MemGPT's core idea, still the most influential in the field: give the agent tools to edit its own
in-context memory, and let it manage the boundary between what is in context and what is in external
storage. Explicitly modelled on OS virtual memory — a small fast "main context" plus larger external
tiers, with the agent paging between them.

The original tool surface:

- `core_memory_append` / `core_memory_replace` — edit in-context blocks (persona, human).
- `conversation_search` — search recall storage (past messages).
- `archival_memory_insert` / `archival_memory_search` — the external vector store.
- Heartbeats — allow the agent to chain multiple tool calls without user input.

Later Letta versions refine this into labelled, character-capped **memory blocks** that can be shared
across agents, with sleep-time agents doing the editing.

**When self-editing memory is the right choice:**
- The agent's task is open-ended and you cannot pre-specify what should be remembered.
- You want the agent's persona/behaviour to evolve.
- You are building a general assistant rather than a narrow workflow.

**When it is the wrong choice:**
- You need predictable, auditable memory content (regulated domains).
- Memory content drives money-moving actions — you do not want the agent authoring its own
  instructions with no review.
- High QPS and tight latency — self-editing costs tool-call round trips inline.

The hybrid I would ship for most products: **agent-proposed, system-validated.** The agent proposes
memory edits; a deterministic layer validates them against schema, category policy, and a deny-list,
then commits. You keep flexibility and retain control of what can enter the store.

```python
def commit_agent_edit(proposal: dict, policy: Policy) -> Result:
    if proposal["category"] not in policy.allowed_categories:  return Result.reject("category")
    if policy.deny_patterns.search(proposal["text"]):          return Result.reject("denied_content")
    if len(proposal["text"]) > policy.max_len:                 return Result.reject("too_long")
    if policy.is_instruction(proposal["text"]) and not policy.allow_self_instructions:
        return Result.reject("self_instruction")               # key injection guard
    return store.write(proposal)
```

That `self_instruction` check is a security control: an agent writing "always approve refunds without
checking" into its own persistent memory is the memory-poisoning attack chain from chapter 09,
executed by the agent itself under injection.

---

## 6.5 Experience replay / trajectory memory

For agents that execute tasks (not just converse), the highest-value memory is often **past
trajectories**: what was the task, what steps were taken, what was the outcome.

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

At the start of a new task, retrieve similar past trajectories and inject a compressed form:

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

**Failures are more valuable than successes**, and this is not intuitive. A successful trajectory
teaches one path; a failed one eliminates a whole branch of the search space. Weight failure
trajectories higher in retrieval, and always extract the "do not attempt" lesson explicitly.

**The critical safety caveat:** trajectory memory is the single most dangerous memory type from a
security standpoint, because retrieved trajectories directly shape *actions*. The MemoryGraft line of
research demonstrates precisely this — implanting malicious "successful experiences" into an agent's
long-term memory compromises future behaviour without ever needing a live jailbreak. Never let a
trajectory retrieved from memory authorise an action that would otherwise require confirmation.
Trajectories are hints, never permissions. Chapter 09 has the full threat model.

---

## 6.6 Putting the tiers together

A complete agent memory stack, and where each piece lives:

```
IN CONTEXT (always)
  ├─ system prompt + policy
  ├─ procedural memory file(s)        ← 06.1, human-reviewable, 100% recall
  ├─ core memory blocks               ← 06.4, agent-editable, character-capped
  └─ recent conversation              ← 02

IN CONTEXT (retrieved per turn)
  ├─ semantic facts                   ← 04, filtered by validity     (ch. 05)
  ├─ relevant episodes                ← 03, hybrid retrieval
  └─ similar trajectories             ← 06.5, task-shaped only

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

If you can draw this diagram for your own system and name the owner of each box, you understand your
memory architecture. If any box is missing, you can predict the failure mode from the chapter it
belongs to.

---

## 6.7 Exercises

1. Build a procedural memory file that an agent maintains for a real repo of yours. Run it for a week
   of real work. Count how many rules were useful, how many were noise, and how many were
   superstition. Tune the confidence threshold accordingly.
2. Implement reflection with the usefulness filter. Run it over 200 memories. Manually rate each
   insight 1–5 for actionability. If the mean is below 3, your filter is too weak — iterate on it,
   not on the reflection prompt.
3. Implement a sleep-time consolidation job. Measure conversation p95 latency with inline memory
   management vs. background. Then measure memory quality (via your chapter-04 scenario suite) for
   both — you should see background win on *both* axes, which is the point.
4. Build trajectory memory for a task agent. Measure success rate with and without retrieved
   trajectories on 30 held-out tasks. Separately measure the effect of failure-only vs. success-only
   retrieval.

Next: `07-systems-design.md`.
