# 10 — Case Studies

> Goal: read real architectures, recognise the patterns from chapters 01–09 in them, and be able to
> make a build/buy decision you can defend.
>
> **Currency warning.** Specific versions, licences, star counts, benchmark numbers, and pricing in
> this chapter are a snapshot and churn fast — this ecosystem has seen licence changes, community
> editions retired, and features moved from open-source SDKs into hosted platforms with little
> notice. Verify before you commit. The *architectural patterns* are the durable part.

---

## 10.1 MemGPT / Letta — memory as an operating system

**Repo:** `letta-ai/letta` (formerly MemGPT) · **Paper:** *MemGPT: Towards LLMs as Operating Systems*
(Packer et al., 2023) · **Licence:** Apache 2.0, self-hostable.

### The core idea

Treat the context window as physical memory and external storage as virtual memory, with the LLM
itself acting as the operating system that pages between them. The agent is given tools to manage its
own memory hierarchy.

```
┌─ Main context (in the window) ──────────────────┐
│  system instructions                            │
│  memory blocks  ← agent-editable, char-capped   │
│    • persona: who the agent is                  │
│    • human:   what it knows about the user      │
│  file blocks                                    │
│  recent messages                                │
└────────────────┬────────────────────────────────┘
                 │ tool calls
   ┌─────────────▼────────────┐   ┌───────────────────────┐
   │ Recall storage           │   │ Archival storage      │
   │ full message history     │   │ embedding-based store │
   │ conversation_search      │   │ archival_*_search     │
   └──────────────────────────┘   └───────────────────────┘
```

Original tool surface: `core_memory_append` / `core_memory_replace` for in-context blocks,
`conversation_search` for recall storage, `archival_memory_insert` / `archival_memory_search` for the
vector store, plus `send_message` (every action, including replying, is a tool call) and *heartbeats*
that let the agent chain multiple tool calls without user input. When the window fills, the
conversation is compacted into a recursive summary stored as a memory block, and all data is
persisted indefinitely so old messages remain searchable.

### The evolution that matters: sleep-time compute

The original design bundled conversation, tool use, and memory management into one agent, which made
responses slower and less reliable. Letta split it: the **primary agent** talks and can search memory
but has no memory-edit tools; a **sleep-time agent** holds the edit tools and rewrites shared memory
blocks asynchronously in the background. Learned context is written to a memory block that can be
shared across agents.

### What to steal

- **Memory blocks as a first-class abstraction** — labelled, character-capped regions of the context
  window, addressable via API. It is the cleanest formulation of "what is always in context" that
  anyone has shipped.
- **Foreground/background split.** Discussed in chapter 06; this is the reference implementation.
- **Shared blocks across agents** as a multi-agent coordination primitive.

### What to be careful about

- Self-editing memory is flexible but hard to audit. In regulated domains, use the
  agent-proposes/system-validates hybrid from 06.4.
- Operationally heavier than a memory library; it is a full stateful agent server, not a component
  you bolt on.
- Steeper learning curve; docs assume more background than the alternatives.

**Use it when:** you want stateful agents with evolving personas, you are researching memory
architectures, or you need low-level control and self-hosting.

---

## 10.2 Mem0 — extraction and reconciliation as a service

**Repo:** `mem0ai/mem0` · **Paper:** *Mem0: Building Production-Ready AI Agents with Scalable
Long-Term Memory* (Chhikara et al., ECAI 2025).

### The core idea

A two-stage pipeline, and it is exactly the write path from chapter 04:

1. **Extraction** — an LLM, given the new exchange plus context, pulls out salient facts.
2. **Update** — a decision step issues **ADD / UPDATE / DELETE / NOOP** against existing memories via
   tool calls, keeping the store internally consistent.

A graph variant adds entity nodes and relationship edges to capture structure that flat facts miss.
Later work moved toward token-efficient retrieval that fuses semantic similarity, BM25, and entity
matching into a single score — with the reported gains concentrated exactly where you would predict
from chapters 03 and 05: temporal queries and multi-hop reasoning.

### Why it became the default bolt-on

It optimises for the thing most teams need: adding memory to an existing agent quickly, with a small
API surface (`add`, `search`, `get_all`, `delete`), managed hosting, and integrations with the common
agent frameworks.

### What to steal

- **The ADD/UPDATE/DELETE/NOOP vocabulary.** Even if you build your own, use these four operations;
  they are the right decomposition.
- **Extraction and reconciliation as separate stages** with separate prompts, so you can debug and
  evaluate them independently.
- **Reporting accuracy alongside token cost.** Their benchmark posts consistently pair the two, which
  is the discipline chapter 08 argues for.

### What to be careful about

- Extraction-based memory is lossy by construction. If your users ask verbatim-recall questions
  ("what exactly did I say about X?"), you need the episode log alongside it.
- LLM-in-the-write-path means write cost scales with conversation volume. Batch at session end.
- Feature availability has moved between the open-source SDK and the hosted platform across major
  versions. If graph features are load-bearing for you, pin versions and read the changelog before
  upgrading.

**Use it when:** you want production memory fast, your workload is conversational, and you are
comfortable with a managed dependency.

---

## 10.3 Zep / Graphiti — temporal knowledge graph memory

**Repo:** `getzep/graphiti` (open source, the engine) · **Paper:** *Zep: A Temporal Knowledge Graph
Architecture for Agent Memory*, arXiv 2501.13956.

Covered in depth in chapter 05; the summary here is for comparison.

### Architecture recap

Three subgraph tiers — **episodes** (raw, non-lossy, timestamped) → **semantic entities and facts**
(bi-temporal edges with validity windows) → **communities** (clusters with generated summaries,
maintained by label propagation). Retrieval composes cosine similarity, BM25 full-text, and
breadth-first traversal, then fuses and reranks (RRF, MMR, optional cross-encoder), and a context
constructor renders facts *with their temporal validity ranges*.

The defining feature is bi-temporality: four timestamps per fact — when it became and stopped being
true in the world, and when the system created and invalidated it. Conflicting knowledge invalidates
rather than deletes.

### What to steal

- **Bi-temporal edges.** The single most transferable idea in the whole field.
- **Bidirectional episode↔fact indices** for citation and incremental update.
- **Rendering validity ranges into the prompt.** Nearly free, disproportionate quality gain.
- **The insight that agent memory is millions of small, mostly-cold graphs** — which is a different
  systems problem than one large graph.

### What to be careful about

- Write-path cost. Entity extraction + resolution + contradiction detection is several LLM calls per
  episode. Chapter 05.8 has the mitigations.
- Operational weight if you self-host a graph database.
- Product/licence churn: the open-source engine and the hosted service have diverged over time.

**Use it when:** temporal correctness matters (CRM, healthcare, finance, support), or your queries
are genuinely multi-hop across entities.

---

## 10.4 LangGraph Store + LangMem — memory in a graph runtime

**Repos:** `langchain-ai/langgraph`, `langchain-ai/langmem`.

### The core idea

Two clearly separated scopes, and this separation is the lesson:

- **Thread state (short-term)** — the checkpointed state of one conversation/graph run. Durable,
  resumable, but scoped to the thread.
- **Store (long-term)** — a namespaced key-value store with optional vector indexing, shared across
  threads. Namespaces are tuples, e.g. `(user_id, "preferences")`.

```python
store.put(("user_123", "preferences"), key="diet", value={"text": "vegetarian"})
results = store.search(("user_123", "preferences"), query="what do they eat", limit=5)
```

LangMem layers memory management on top: extraction utilities, prompt optimisation from feedback, and
background memory managers.

### What to steal

- **Explicit namespace tuples.** `(tenant, subject, category)` as a first-class key structure makes
  isolation and deletion tractable, and it maps directly onto the multi-tenancy design in chapter 07.
- **The thread-state vs. store distinction.** Many home-grown systems blur session and long-term
  memory and then cannot reason about either. Draw the line explicitly.
- **Checkpointing as a memory primitive** — durable execution state means an agent can resume a
  long-running task, which is a form of memory people forget to design for.

**Use it when:** you are already on LangGraph, or you want a memory primitive rather than an opinion.

---

## 10.5 Cognee — pipeline-oriented graph + vector memory

**Repo:** `topoteretes/cognee` · Apache 2.0.

Positions memory as an **ingestion pipeline** ("cognify") producing a hybrid graph + vector store,
with many distinct retrieval modes and flexible deployment from local self-host to managed cloud.

### What to steal

- **Memory as an ETL pipeline with named stages.** Ingest → chunk → extract entities → build graph →
  embed → index. Making the stages explicit and individually re-runnable is exactly the "separate the
  five decisions" advice from chapter 04, applied at system level.
- **Multiple retrieval modes as a first-class concept.** Different query classes want different
  retrievers; exposing that choice rather than hiding it behind one `search()` is honest design.

**Use it when:** you are ingesting heterogeneous sources (docs + conversations + structured data) and
want a batteries-included pipeline.

---

## 10.6 Coding agents — file-based memory, and why it wins

Claude Code, Cursor, and the Devin-likes converged on something the memory-framework world often
overlooks: **for procedural and project knowledge, files beat databases.**

The stack, roughly:

```
persistent, always in context:
  CLAUDE.md / AGENTS.md / .cursorrules   ← project conventions, human-editable, in git

per-task, written by the agent:
  PLAN.md, PROGRESS.md, NOTES.md         ← externalised working memory

per-session, automatic:
  compaction of the transcript           ← summarise and reinitialise near the limit
  tool-result clearing                   ← lightest-touch compaction, drop old result bodies

per-subtask:
  sub-agents with isolated windows       ← return condensed summaries only
```

The techniques Anthropic describes for extended-horizon work are exactly these three: compaction,
structured note-taking (agentic memory — the agent writes notes persisted outside the context
window), and multi-agent architectures with isolated sub-agent contexts. Related platform primitives
push in the same direction: context editing, a memory tool for file-style CRUD across conversations,
context awareness so the model knows its remaining capacity, and programmatic tool calling so
intermediate tool output never enters the window at all.

### Why files win here

Re-read the table in chapter 06.1. Human-editable, diffable, reviewable in a PR, version controlled,
100% recall when always loaded, and debuggable with `cat`. For knowledge that must apply *every
time*, retrieval is a liability, not a feature.

### What to steal

- Before you build a vector pipeline, ask whether a markdown file solves 80% of the problem.
- **Tool-result clearing before full compaction.** The safest big win.
- **Sub-agent isolation** for anything search-shaped or explore-shaped.
- **A `dead_ends` section** in whatever carries state across compactions.

---

## 10.7 Consumer assistant memory — the product patterns

Consumer assistants (ChatGPT, Claude, Gemini and peers) have converged on a recognisable product
shape, regardless of backend differences. The patterns are worth cataloguing because they encode
lessons about *users*, not just systems:

1. **Two memory channels.** Explicit "saved" memories the user can see and edit, plus implicit
   retrieval over past conversation history. The explicit channel exists mostly for trust, not
   capability.
2. **A memory management UI.** List, view source, edit, delete, disable. As chapter 09 argued, this is
   simultaneously the best privacy control and the best quality feedback loop you can ship.
3. **Per-conversation opt-out** ("temporary chat"), because users need a way to interact without
   being recorded.
4. **Visible write moments.** Telling the user "I've saved that" when a memory is written. Silent
   memory formation is the fastest route to a creepiness complaint.
5. **Conservative extraction defaults.** Sensitive categories are excluded or opt-in.

If you are building a consumer product with memory, these five are effectively table stakes. Skipping
(2) and (4) is what turns a memory feature into a trust problem.

Note also that measured performance of assistant-style memory on public benchmarks tends to be
unremarkable compared to dedicated memory layers — the product constraints (latency, cost, privacy
conservatism, breadth of domain) are very different from a benchmark-maximising configuration. Do not
infer engineering quality from a leaderboard row.

---

## 10.8 Build vs. buy

The decision framework I would actually use.

**Buy (managed memory layer) when:**
- Memory is not your differentiator.
- Your workload is conversational and mainstream.
- You need to ship in weeks.
- Your compliance posture tolerates a third party holding personal data.

**Self-host open source when:**
- Data residency or regulatory constraints rule out SaaS.
- You need to modify write-path behaviour (custom categories, custom reconciliation).
- Cost at your volume exceeds the engineering cost of running it.

**Build when:**
- Memory *is* the product, or is core to the differentiation.
- Your domain has structure that generic extraction destroys (medical, legal, financial).
- You need guarantees — deletion, audit, isolation — that no vendor will contractually give you.
- Your scale makes per-call pricing absurd.

**The pragmatic path most teams should take**, and the one I would recommend:

1. Start with the simplest thing: session summarisation + a facts table in Postgres. Two weeks.
2. Add hybrid retrieval and the ADD/UPDATE/DELETE/NOOP write path. Four weeks.
3. Measure with your own eval harness (chapter 08). Two weeks.
4. *Only then* evaluate whether a framework beats what you have, using the fair-comparison harness in
   08.9.

Teams that adopt a memory framework at step 0 usually cannot answer "is it working?" at step 4,
because they never built the measurement. And because a memory layer is a deep dependency with an
expensive migration, weight governance and licence stability heavily in the choice — this ecosystem
has repeatedly changed the terms under teams that assumed stability.

---

## 10.9 Reading the code

If you read one codebase, read Graphiti — it is compact and the bi-temporal logic is legible. If you
read two, add Mem0's extraction/update prompts, which are short and instructive.

What to look for while reading any of them:

1. **Where is the write path?** Find the extraction prompt and the reconciliation decision. Compare
   to chapter 04.
2. **What is the conflict policy?** Search for `invalidate`, `supersede`, `expired`, `valid_to`. If
   there is none, the system cannot handle knowledge updates and will fail on LongMemEval's update
   category.
3. **How is tenancy enforced?** Is it a filter, or a boundary?
4. **What runs in the background?** If nothing does, quality will plateau (chapter 06.3).
5. **What is the deletion path?** Does it cascade to derived artefacts? This is the fastest way to
   judge production-readiness.

Those five questions, applied to any memory system, will tell you more in an hour than a week of
documentation.

Next: `11-projects-small.md`.
