# 01 — Foundations

> Goal: after this chapter you can precisely define what "memory" means for an agent, place any
> product or paper into a taxonomy, and explain to a skeptical staff engineer why this is a systems
> problem rather than a model problem.

---

## 1.0 In plain words (read this first)

Imagine hiring a brilliant consultant with total amnesia. Every morning they arrive knowing
everything about the world and *nothing* about you. Before each meeting, someone has to hand
them a one-page brief: who you are, what you decided last time, what is still open.

The consultant is the model. **The person writing the brief is the memory layer.** This course
is about writing that brief: what goes on the page, where the facts come from, how you keep them
true, and how you prove the brief was good.

Three sentences that carry the whole chapter:

1. The model is a pure function. Same input, same output, no memory of yesterday.
2. So "remembering" is something *your code* does, before the call, by choosing tokens.
3. Choosing well is a storage + retrieval + consistency problem, which is why this is systems
   engineering and not prompt writing.

If you have not read `00-why-memory.md`, read it before this. It is the argument for why the
layer exists at all; this chapter is the vocabulary for building it.

### The words everyone uses and nobody defines

Keep this table open for the first three chapters. Most confusion in this field is two people
using one of these words to mean two different things.

| Word | Plain meaning | Not to be confused with |
|---|---|---|
| **Context window** | The list of tokens you send in one API call | Memory. The window is a *request*; memory is a *store*. |
| **Working memory** | Whatever is in the window right now | Long-term memory |
| **Session memory** | Server-side state for one conversation | Long-term memory across conversations |
| **Long-term memory** | A database row that outlives the conversation | The model's weights |
| **Episodic memory** | "What happened", timestamped, raw | Semantic memory |
| **Semantic memory** | A distilled fact: *user is vegetarian* | **Semantic search**, which is a retrieval *method*. LangChain's own docs flag this exact collision. |
| **Procedural memory** | "How to do things here" — usually a file of rules | Facts about the user |
| **Write path** | Code that decides what to store and how to fix contradictions | Ingestion in classic RAG |
| **Read path** | Code that decides what to inject this turn | The model's attention |
| **Compaction** | Replacing a long transcript with a summary and continuing | Deletion |
| **Consolidation** | Merging several related memories into one, offline | Compaction (that is about the transcript, this is about the store) |
| **Provenance** | The record of *where a memory came from* | Confidence |
| **Bi-temporal** | Storing both "true in the world from→to" and "we believed it from→to" | A single `updated_at` column |
| **Salience** | The decision "is this worth remembering at all" | Relevance, which is a *read*-time decision |

### Statelessness, in six lines you can run

```python
def call_model(messages):
    """Pretend this is an API call. The only thing the model ever sees is `messages`."""
    return f"[model saw {sum(len(m['content']) for m in messages)} chars of input]"

call_model([{"role": "user", "content": "My name is Priya."}])
call_model([{"role": "user", "content": "What is my name?"}])   # -> no idea. None.
```

There is no channel between those two calls. None. Not a cookie, not a session id, not a hidden
cache. If the second call is to know the name, *some code you wrote* has to put the string
"Priya" into the second `messages` list. Everything else in this course is the elaboration of
that one sentence.

---

## 1.1 The stateless function

Strip away the SDKs and an LLM call is:

```
f(system_prompt + messages + tools) -> next_tokens
```

No hidden state survives the call. The KV cache is an inference-time optimisation, not memory — it
is scoped to a single forward pass sequence and evicted. Fine-tuned weights are a kind of memory,
but they are slow, global, and non-deletable, which disqualifies them for per-user facts.

So when a user says "remember I'm vegetarian" and three weeks later asks "suggest a restaurant near
my office", *something outside the model* has to:

1. have decided, three weeks ago, that "vegetarian" was worth persisting;
2. have stored it durably, attributed to this user;
3. have decided, at query time, that this restaurant question is one where dietary preference is
   relevant;
4. have retrieved it and placed it in the context window in a form the model will actually use;
5. and have not simultaneously flooded the window with 200 other memories that drown the signal.

Each of those five steps is a separate subsystem with separate failure modes. **Agentic memory is
the name for that whole pipeline.** Most engineers, when they first hear "agent memory", think only
about step 4. Steps 1 and 5 are where the real engineering is.

### The uncomfortable framing

Here is the framing I use in design reviews:

> Memory is a *lossy compression* of an unbounded interaction history into a bounded token budget,
> performed under latency and cost constraints, where the loss function is "did the agent behave
> correctly on the next turn".

Every design decision is a point on that compression curve. Store everything verbatim and retrieve
nothing → cheap writes, useless reads. Summarise aggressively → cheap reads, irreversible loss.
Extract structured facts → good reads, but you have introduced an extraction model that hallucinates
and a consistency problem when facts change.

There is no free lunch and there is no universally correct point. What there *is*: a set of
well-understood architectures, each optimal for a different query distribution.

---

## 1.2 Why "just use a bigger context window" is not the answer

The most common objection: context windows are now enormous, so why not paste the whole history?

Four reasons, in increasing order of importance.

**Cost.** Attention cost is superlinear in sequence length and, more practically, you pay per input
token on every turn. A 500k-token history re-sent on each of 50 turns is 25M input tokens for one
session. Multiply by your DAU. This alone kills it for consumer products.

**Latency.** Time-to-first-token grows with prefill length. A 1M-token prefill is seconds, not
milliseconds, even with prefix caching. Interactive agents have a p95 budget of a few seconds
end-to-end and memory retrieval must fit in a fraction of it.

**Quality degradation — "context rot".** This is the one that surprises people. Model accuracy does
not stay flat as you fill the window; it degrades, and past some fraction of the maximum it can fall
off sharply rather than gradually. Irrelevant tokens are not neutral — they act as distractors.
Chroma's *Context Rot* report and the follow-on literature ("context length alone hurts LLM
performance despite perfect retrieval") document this across model families. **A smaller, curated
context beats a larger, complete one.** Internalise this; it justifies the entire field.

**Unbounded histories exist.** A coding agent working a multi-day migration, or an assistant used
daily for two years, generates more history than any window will ever hold. The problem is not "how
do I fit 500k tokens", it is "how do I fit an unbounded stream into a fixed window". That is a
compression problem forever, regardless of window size.

Anthropic's own engineering guidance on context makes the same point: windows of every size are
subject to context pollution and relevance concerns, so the answer is compaction, structured
note-taking, and multi-agent isolation rather than waiting for bigger windows.

> **Mental model:** the context window is a CPU register file, not a hard disk. It is small, fast,
> and every byte in it should be there for a reason. Memory systems are the memory hierarchy beneath
> it. The MemGPT paper made this analogy explicit and it remains the most useful one in the field.

---

## 1.3 The taxonomy

Two axes matter. Get comfortable with both because papers and vendors mix them constantly.

### Axis A: lifetime / scope

| Tier | Lives in | Lifetime | Typical size | Analogy |
|---|---|---|---|---|
| **Working memory** | The context window itself | One turn | 1k–200k tokens | CPU registers / L1 |
| **Session memory** | Server-side conversation state | One session | Unbounded, but scoped | RAM |
| **Long-term memory** | Database | Forever (until deleted) | Unbounded, cross-session | Disk |
| **Shared / org memory** | Database, multi-principal | Forever | Unbounded, cross-user | Network filesystem |

The tier boundary that matters most is **session ↔ long-term**, because that is where the
*extraction* decision lives: what from this session deserves to outlive it? Most systems get this
wrong in one of two directions — storing every utterance (write amplification, retrieval noise) or
storing only explicit "remember that" commands (misses 90% of useful signal).

### Axis B: content type (the cognitive taxonomy)

Borrowed from human memory research (Tulving's episodic/semantic distinction, plus procedural
memory). It is an *analogy*, not a mechanism, but it is a genuinely useful design vocabulary because
each type wants different storage and different retrieval.

| Type | Contains | Example | Natural storage | Natural retrieval |
|---|---|---|---|---|
| **Episodic** | Raw events, timestamped, "what happened" | "On 2026-03-04 the user asked me to book a flight to Delhi and I failed with a 402" | Append-only log, embedded | Similarity + time filter |
| **Semantic** | Distilled facts about entities | "User is vegetarian"; "Acme's renewal date is Sept 30" | Row per fact, or KG edge | Entity lookup, then similarity |
| **Procedural** | How to do things; learned behaviour | "When deploying this repo, run `make gen` before tests" | Prompt fragments / skill files | Task-type matching |
| **Working** | Current task state | "Step 3 of 7; two subtasks failed" | Scratchpad in context or file | Always in context |

A trap to avoid: many teams build only semantic memory (a table of extracted "user facts") and then
discover their agent cannot answer "what did we decide in that meeting last Tuesday" — a purely
episodic question. Conversely, pure-episodic systems (embed every message, retrieve top-k) cannot
answer "what is the user's current job title" when the user changed jobs twice, because they surface
three contradictory messages with no notion of which is current.

**Production systems need at least episodic + semantic, with an explicit link between them.**
The link matters: every semantic fact should point back to the episodes that produced it, so you can
cite it, audit it, and recompute it. Zep's Graphiti design makes this bidirectional — you can go
from a fact to its source episodes and from an episode to its derived facts. Copy that property.

### The third axis nobody names: *who* the memory is about

- **User memory** — facts about the human. Highest privacy sensitivity, GDPR-relevant.
- **Agent memory** — the agent's own persona, learned strategies, self-notes. MemGPT called this the
  "persona" block.
- **World/domain memory** — facts about the domain that are not user-specific (product catalogue,
  codebase structure, org chart). Often better served by classic RAG.
- **Task memory** — state for a specific long-running job.

Mixing these into one store is the single most common architectural mistake I see. They have
different retention policies, different access controls, different deletion semantics, and different
staleness tolerances. Give them separate namespaces from day one even if they share a table.

What "separate namespaces" means concretely — a key structure, not a comment in a design doc:

```python
from typing import NamedTuple

class MemKey(NamedTuple):
    tenant: str     # the isolation boundary. Every query filters on it. Never optional.
    subject: str    # WHO the memory is about: "user:42" | "agent" | "org:acme" | "task:job_9"
    kind: str       # WHAT it is: "semantic" | "episodic" | "procedural"
    category: str   # the typed slot: "diet" | "employer" | "deploy_rule"

MemKey("acme", "user:42", "semantic", "diet")      # deletable on user request
MemKey("acme", "agent",   "procedural", "tone")    # survives user deletion
MemKey("acme", "org:acme","semantic", "policy")    # shared; needs different ACLs
```

Now "delete everything about user 42" is a prefix scan, not an archaeology project. Two
independent products landed on exactly this shape: LangGraph's `BaseStore` keys memories by a
**namespace tuple** such as `(user_id, "preferences")`, and AWS AgentCore Memory writes every
extracted long-term memory under a configured **namespace** path scoped by `actorId`
([AgentCore memory organization](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-organization.html)).
When two unrelated teams pick the same primitive, it is because deletion and isolation force it.

Getting this wrong has a specific, expensive symptom, and OpenAI documents it from the user's
side: because memories are stored *separately* from chats, "deleting a chat doesn't erase its
memories; you must delete the memory itself"
([Memory and new controls for ChatGPT](https://openai.com/index/memory-and-new-controls-for-chatgpt/)).
That is not a bug — it is the unavoidable consequence of derived state. Chapter 09 is about
making it tractable rather than surprising.

---

## 1.4 The canonical pipeline

Almost every production memory system, regardless of vendor, is this diagram:

```
                       ┌──────────────────────────────────────────┐
   conversation turn   │              WRITE PATH                  │
   ──────────────────► │                                          │
                       │  1. capture      raw turn -> episode log │
                       │  2. extract      LLM -> candidate facts  │
                       │  3. resolve      dedupe / entity-link    │
                       │  4. reconcile    ADD|UPDATE|DELETE|NOOP  │
                       │  5. index        embed + write indexes   │
                       └──────────────────────────────────────────┘
                                          │
                                          ▼
                       ┌──────────────────────────────────────────┐
                       │              STORAGE                     │
                       │  episodes | facts | graph | vectors      │
                       │  + provenance, validity, tenancy         │
                       └──────────────────────────────────────────┘
                                          │
                       ┌──────────────────────────────────────────┐
   next user turn      │              READ PATH                   │
   ──────────────────► │                                          │
                       │  1. gate         is memory needed?       │
                       │  2. query build  rewrite / expand        │
                       │  3. candidates   vector + BM25 + graph   │
                       │  4. fuse+rerank  RRF, cross-encoder      │
                       │  5. budget       fit to token allowance  │
                       │  6. format       render into prompt      │
                       └──────────────────────────────────────────┘
                                          │
                                          ▼
                              context window -> LLM
                                          │
                       ┌──────────────────────────────────────────┐
                       │        BACKGROUND / MAINTENANCE          │
                       │  consolidate | summarise | decay         │
                       │  re-embed | community detect | GC        │
                       └──────────────────────────────────────────┘
```

Three observations that take people a year to learn:

1. **The write path is harder than the read path.** Retrieval is a solved-ish IR problem with 30
   years of literature. Deciding *what is worth remembering* and *what to do when new information
   contradicts old information* has no textbook answer.

2. **The background path is where quality comes from.** Systems that only write synchronously and
   read synchronously plateau fast. Consolidation, summarisation, and re-organisation done offline
   (Letta calls this sleep-time compute) is what turns a pile of facts into usable memory.

3. **Step 1 of the read path — the gate — is skipped by almost everyone and is worth a lot.** Most
   turns do not need memory. Running retrieval on "thanks!" costs latency and injects noise. A cheap
   classifier or heuristic that skips retrieval on 40% of turns is often the highest ROI change in
   the whole system.

---

## 1.5 Where the cognitive analogy breaks

Useful to know so you do not over-index on neuroscience papers.

- **Human forgetting is not deletion; it is retrieval failure.** Your system's forgetting probably
  *is* deletion (or must be, for compliance). Decay-based scoring is a fine heuristic, but do not
  design your GDPR erasure path around "the memory will decay eventually".
- **Consolidation in humans is a slow, biologically constrained process.** In your system it is a
  cron job whose cost is dollars and whose latency you control. Do not mimic biological schedules;
  optimise for cost and staleness targets.
- **Humans do not have a `user_id` partition.** Multi-tenancy, access control, and the fact that a
  memory can be *legally required to disappear* have no biological analogue and are half your
  engineering effort.
- **Human memory is reconstructive and unreliable.** Yours must be auditable and citable. Any design
  that cannot answer "why do you believe this about me, and where did you learn it" will fail a
  privacy review.

Use the taxonomy as vocabulary. Do not use neuroscience as a spec.

---

## 1.6 Reference architectures, one line each

Preview of chapter 10, so you have hooks while reading the middle chapters.

- **MemGPT / Letta** — OS analogy. In-context "core memory" blocks the agent edits with tools, plus
  external recall (conversation) and archival (vector) storage. The agent manages its own memory.
- **Mem0** — extraction + reconciliation. An LLM pulls facts from turns, then a second decision step
  emits ADD / UPDATE / DELETE / NOOP against existing memories. Vector-centric, with hybrid scoring
  in later versions.
- **Zep / Graphiti** — temporal knowledge graph. Episodes → entities/facts → communities, with
  bi-temporal edges so superseded facts are invalidated rather than deleted.
- **LangGraph Store + LangMem** — memory as a namespaced key-value/vector store attached to a graph
  runtime; explicit separation of thread-scoped state and cross-thread store.
- **Cognee** — pipeline-oriented ("cognify") graph + vector hybrid with many retrieval modes.
- **Coding agents (Claude Code, Cursor, Devin-likes)** — file-based memory. `AGENTS.md`/`CLAUDE.md`
  style instruction files, plus compaction of the running transcript, plus sub-agents with isolated
  contexts returning short summaries. Underrated: for many domains, **files are the best memory
  store**, because they are human-editable, diffable, and reviewable.

That last point deserves emphasis. Before you build a vector pipeline, ask whether a markdown file
per project, edited by the agent and reviewed by the human, solves 80% of your problem. Often it
does. Complexity should be earned.

---

## 1.7 Exercises

1. Take a product you use daily that has memory (an assistant, an IDE agent). Write down five things
   it remembers about you, and for each, classify it on both axes (lifetime, content type) and guess
   the write trigger. Which do you think were explicitly written vs. extracted?

2. Estimate the cost of the "just use a big window" approach for a hypothetical assistant with 1M
   DAU, 20 turns/session, 100k-token histories, at current input token prices. Compare to a retrieval
   approach injecting 2k tokens/turn. The ratio is the business case for this entire field.

3. Write, in one page, the failure mode of a system that has only semantic memory. Then the failure
   mode of one with only episodic memory. Keep this page; you will use it in the capstone design doc.

Next: `02-context-engineering.md`.
