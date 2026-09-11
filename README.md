# Agentic Memory Course

A from-scratch course on how memory for LLM agents actually works, and how it's built in
production. Written like an onboarding doc for a memory-infrastructure team: concepts first, then
data structures, then systems engineering, then the failure modes that only show up at scale.

No frameworks until chapter 10. You build your own chunker, retriever, and extractor first, so the
open-source tools (Mem0, Zep, Letta, ...) read as familiar rather than magic by the time you get there.

**Start here → [`00-README.md`](./00-README.md)** — full roadmap, 12-week study plan, and
environment setup.

**Writing status → [`PROGRESS.md`](./PROGRESS.md)** — which chapters have had the deep pass, what
each remaining one still needs, and where to resume.

---

## Contents

**Core chapters**
| | |
|---|---|
| [01 — Foundations](./01-foundations.md) | Why statelessness hurts, the memory taxonomy |
| [02 — Context Engineering](./02-context-engineering.md) | Compaction, note-taking, sub-agent isolation |
| [03 — Retrieval Fundamentals](./03-retrieval-fundamentals.md) | Embeddings, hybrid search, reranking |
| [04 — The Memory Write Path](./04-memory-write-path.md) | Extraction, ADD/UPDATE/DELETE/NOOP, forgetting |
| [05 — Temporal & Graph Memory](./05-temporal-and-graph-memory.md) | Bi-temporal facts, knowledge graphs |
| [06 — Procedural & Reflective Memory](./06-procedural-and-reflective.md) | Procedural memory, reflection, sleep-time compute |
| [07 — Systems Design](./07-systems-design.md) | Multi-tenancy, latency budgets, cost models |
| [08 — Evaluation](./08-evaluation.md) | LoCoMo, LongMemEval, building your own harness |
| [09 — Security, Privacy & Governance](./09-security-privacy-governance.md) | Memory poisoning, GDPR erasure |
| [10 — Case Studies](./10-case-studies.md) | MemGPT/Letta, Mem0, Zep/Graphiti, LangMem, Cognee |

**Practice**
| | |
|---|---|
| [11 — Small Projects](./11-projects-small.md) | Six projects, 2–8 hours each |
| [12 — Capstone Projects](./12-projects-capstone.md) | Three projects, 2–6 weeks each |
| [13 — Reading List](./13-reading-list.md) | Annotated papers, books, repos |
| [14 — Design Review Playbook](./14-design-review-playbook.md) | 25 questions + a scoring rubric |

---

## The one-paragraph version

An LLM is a stateless function. "Memory" is never inside the model — it's a system built around it
that decides, every turn, which tokens go into the context window. This course is about how you
choose those tokens: where you store candidates, how you rank them, how you keep them true over
time, how you delete them on request, and how you prove the whole thing works.
