# 13 — Reading List

Ordered and annotated. Read in the order given within each section; the sequencing matters more than
the volume.

---

## Papers — the core sequence

Read these six in order. Together they are the intellectual history of the field.

**1. MemGPT: Towards LLMs as Operating Systems** — Packer et al., 2023 (arXiv 2310.08560)
The founding paper. Introduces the OS analogy: main context as physical memory, external stores as
virtual memory, with the LLM paging between tiers via tool calls. Even if you never use Letta, the
framing shapes how you think about every subsequent system.
*Read for:* the memory hierarchy, self-editing memory, the function-calling loop with heartbeats.

**2. Generative Agents: Interactive Simulacra of Human Behavior** — Park et al., 2023 (arXiv
2304.03442)
The reflection paper. Memory stream + retrieval scored by recency, importance, and relevance +
periodic reflection producing higher-level insights that can themselves be reflected on.
*Read for:* the reflection tree, the three-component retrieval score, and the honest discussion of
what breaks at scale.

**3. Zep: A Temporal Knowledge Graph Architecture for Agent Memory** — Rasmussen et al., 2025 (arXiv
2501.13956)
The best-documented production architecture in the open literature. Bi-temporal edges, three-tier
subgraph hierarchy, edge invalidation instead of deletion, composed retrieval with reranking.
*Read for:* bi-temporality, entity/fact/community tiers, and the systems framing of millions of small
graphs.

**4. Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory** — Chhikara et al.,
ECAI 2025 (arXiv 2504.19413)
The extraction/reconciliation architecture, plus the graph variant. The cleanest statement of the
ADD/UPDATE/DELETE/NOOP write path.
*Read for:* the two-stage write path and the accuracy-vs-token-cost framing.

**5. LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory** — Wu et al., ICLR
2025 (arXiv 2410.10813)
The benchmark that defines the ability taxonomy everyone now uses: information extraction,
multi-session reasoning, temporal reasoning, knowledge updates, abstention.
*Read for:* the category design, and especially the abstention task — the design choice most worth
copying into your own eval set.

**6. LoCoMo / Evaluating Very Long-Term Conversational Memory** — Maharana et al., 2024 (arXiv
2402.17753)
The multi-session conversation benchmark, built via a machine–human pipeline over persona and event
graphs.
*Read for:* how to *construct* a memory benchmark, which you will need to do for your own domain.

### Then, by topic

**Context and compaction**
- *Context Rot: How Increasing Input Tokens Impacts LLM Performance* — Chroma, 2025. The empirical
  basis for "smaller curated context beats larger complete context."
- *Context length alone hurts LLM performance despite perfect retrieval* — Du et al., 2025 (arXiv
  2510.05381). Isolates length from retrieval quality; the cleanest experiment on the question.
- *Effective context engineering for AI agents* — Anthropic engineering blog, 2025. Compaction,
  structured note-taking, sub-agent isolation. Short and practical.
- *Agentic Context Engineering (ACE)* — ICLR 2026. Frames context as an evolving collection of delta
  entries rather than a rewritten blob; the principled fix for context collapse.
- *Parallel Context Compaction for Long-Horizon LLM Agent Serving* — 2026 (arXiv 2605.23296). The
  serving-systems view of compaction; useful if you operate the infrastructure.

**Retrieval**
- *Reciprocal Rank Fusion outperforms Condorcet and individual rank learning methods* — Cormack et
  al., SIGIR 2009. Four pages. The source of `k=60`. Read it; it will take fifteen minutes and you
  will use it forever.
- *Dense Passage Retrieval* — Karpukhin et al., 2020. The bi-encoder baseline everything is measured
  against.
- *ColBERT / ColBERTv2* — Khattab & Zaharia. Late interaction: the middle ground between bi-encoders
  and cross-encoders. Worth knowing when your rerank budget is tight.
- *HNSW: Efficient and robust approximate nearest neighbor search* — Malkov & Yashunin, 2016. Read at
  least sections 3–4 so `M` and `ef` stop being magic numbers.
- *From Local to Global: A Graph RAG Approach* — Microsoft, 2024. Community detection plus hierarchical
  summarisation; the direct ancestor of the community tier in graph memory systems.

**Memory security**
- *A Survey on Long-Term Memory Security in LLM Agents* — 2026 (arXiv 2604.16548). Organised by
  memory lifecycle phase: write, retrieve, forget. The FORGET section on incomplete deletion and
  residual state is required reading before you design an erasure path.
- *MemoryGraft: Persistent Compromise of LLM Agents via Poisoned Experience Retrieval* — 2025 (arXiv
  2512.16962). Why trajectory memory is the most dangerous memory type.
- *Trojan Hippo: Weaponizing Agent Memory for Data Exfiltration* — 2026 (arXiv 2605.01970).
- OWASP Top 10 for Agentic Applications, ASI06 (Memory & Context Poisoning). The control set your
  security reviewer will cite.

**Newer directions worth tracking**
- BEAM (million-to-ten-million-token memory evaluation), HaluMem (memory consistency and
  hallucination), RealMem (project-oriented long-term interaction), WorldLines (long-horizon stateful
  embodied agents). These are where the benchmark frontier is moving: agentic, tool-using,
  multi-week.

---

## Books

There is no good book on agentic memory specifically — the field is too young. These are the books
that give you the underlying competencies, and they are worth more than another twenty blog posts.

**1. *Designing Data-Intensive Applications* — Martin Kleppmann**
The most important book on this list, and it never mentions LLMs. Replication, partitioning,
transactions, consistency, batch vs. stream processing. Chapters 5–7 and 11 map directly onto
chapter 07 of this course. If you read one book, read this one.

**2. *Introduction to Information Retrieval* — Manning, Raghavan, Schütze** (free online)
The IR foundations behind chapter 03. Chapters 1–2 (indexing), 6 (scoring/tf-idf), 8 (evaluation),
11 (probabilistic models, BM25). The evaluation chapter alone will improve how you measure retrieval.

**3. *Database Internals* — Alex Petrov**
Storage engines, B-trees, LSM trees, distributed transactions. Read it to understand why your vector
index behaves as it does under deletes and why compaction is a universal pattern.

**4. *Temporal Data & the Relational Model* — Date, Darwen, Lorentzos**
Dense and old, but it is the rigorous treatment of bi-temporal modelling that chapter 05 rests on.
Skim for the concepts if the formalism is heavy. Snodgrass's *Developing Time-Oriented Database
Applications in SQL* is the more practical alternative.

**5. *Site Reliability Engineering* — Beyer et al.** (free online)
SLOs, error budgets, and the operational discipline behind chapter 07's latency budgets and runbooks.

**6. *Building LLM-Powered Applications* / *AI Engineering* — Chip Huyen**
The best general-purpose treatment of the surrounding engineering: evaluation, data flywheels,
deployment. Read the evaluation chapters alongside chapter 08.

**7. *Memory: A Very Short Introduction* — Jonathan Foster**
Two hours. Gives you the episodic/semantic/procedural vocabulary properly rather than second-hand,
so you can tell when a paper's cognitive analogy is load-bearing and when it is decoration.

---

## Repositories to read

Ordered by reading value per hour, not by popularity.

**1. `getzep/graphiti`** — compact, legible, and the bi-temporal invalidation logic is the single most
transferable code in the ecosystem. Start at the edge invalidation and temporal extraction paths.

**2. `mem0ai/mem0`** — read the extraction and update prompts specifically. They are short and they
encode a lot of hard-won behaviour. Compare against your own from project S3.

**3. `letta-ai/letta`** — the agent loop, memory block management, and the sleep-time agent
implementation. Larger codebase; use the docs to navigate to the memory subsystem rather than reading
top-down.

**4. `langchain-ai/langgraph`** — read the `BaseStore` interface and the checkpointer. The
namespace design and the thread-state/store separation are the parts worth your time.

**5. `pgvector/pgvector`** — the index implementation. Reading how HNSW is built and queried inside a
transactional database demystifies a lot.

**6. `topoteretes/cognee`** — the pipeline structure ("cognify" stages) as an example of memory as
explicit, re-runnable ETL.

**Read them with the five questions from chapter 10.9:** where is the write path, what is the conflict
policy, how is tenancy enforced, what runs in the background, and what does deletion do.

---

## Courses and talks

- **DeepLearning.AI — *LLMs as Operating Systems: Agent Memory*** (with Letta). Short, hands-on, and
  the fastest way to get the MemGPT model into your fingers.
- **Anthropic's engineering blog** — the context engineering and multi-agent research system posts.
  Practitioner-written, specific, and unusually honest about trade-offs.
- **Talks from the vector database vendors on filtered ANN search** — the filtered-HNSW problem from
  chapter 03.4 is best explained in conference talks, not papers.

---

## How to keep current without drowning

The field produces more content than anyone can read, and most of it is vendor marketing. My filter:

1. **Track four or five repos' changelogs**, not blog posts. Code changes are signal; announcements
   are not.
2. **Read benchmark papers, skip benchmark press releases.** If a claim does not report the actor
   model, the judge, and the token cost, ignore it (chapter 08.3).
3. **Follow the security literature closely.** It moves fast, it is empirical, and it tells you about
   failure modes years before the product literature admits them.
4. **Re-derive, don't adopt.** When a new technique appears, ask which of the five write-path
   decisions or six read-path stages it changes. Almost everything is a refinement of one box in the
   chapter-01 pipeline. If you cannot place it in that diagram, either it is genuinely novel — rare
   and worth deep attention — or it is a rebrand.

Next: `14-design-review-playbook.md`.
