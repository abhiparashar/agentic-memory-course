# 13 — Reading List

> Goal: the sources this course is built on, each with a verified citation, a reading order that
> respects dependencies, and an explicit note on what each source does *not* say. Plus a filter for
> the firehose, sized against real arXiv volume.
>
> Every citation below was checked mechanically on 2026-09-12 (§13.1), which is how the three
> defects in the first draft of this chapter were found.

---

## 13.0 In plain words

### The map cabinet

A good map library does not hand you every map. It hands you the one map at the right scale, and
tells you which roads on it are out of date. Two maps of the same terrain at different scales are
not redundant — they answer different questions — but forty maps of the same valley is not a
library, it is a hoarding problem.

This chapter is a small cabinet: about thirty sources, ordered, with the out-of-date roads marked.

### The naive version

```
"I'll just follow arXiv cs.AI and the memory-framework changelogs."
```

Perfectly reasonable. It fails for a reason you can count.

### The arithmetic that kills it

arXiv submissions whose abstract contains both "memory" and "agent", by submission year (arXiv API,
queried 2026-09-12):

```
2022      152 papers
2023      219 papers
2024      359 papers
2025    1,080 papers
2026    2,603 papers (Jan-Aug)  ->  325/month  ->  10.7/day
```

**12.4× growth from 2023 to 2026**, and the 2026 rate is one new paper every two and a quarter
hours. Now price reading them:

```
325 papers/month x 40 min to read one properly  =  217 h/month
realistic reading budget for a working engineer =    4 h/month
                                                   ---------
                                                      54x over
```

You are not behind. **You are 54× over budget and always will be**, so the only useful skill is
triage: a rule for which 4 hours to spend. Reading more is not available; reading *in dependency
order* is, and that is nearly free — six papers, read in the right sequence, make the other 2,597
skimmable because each one becomes a delta against a model you already hold.

And there is a second, quieter failure. A list of *names* rots silently. Three defects in the first
draft of this very chapter (§13.1) were all of that kind: a page count that was wrong, a paper cited
by name with no identifier, and a name that appeared unfindable when it was not. None of them would
have been caught by re-reading the text; all three fell out of a 30-line script.

### What fixes what

| Problem | Section |
|---|---|
| 54× more papers than reading hours | 13.8 triage rules, 13.2 dependency order |
| Citations that cannot be checked | 13.1 mechanical verification, IDs on everything |
| Reading a result without its caveat | every entry's "what it does not say" |
| Vendor claim taken as a benchmark | 13.4 how to read vendor docs |
| Knowing the papers, unable to build | 13.6 repositories + the five questions |
| No mental model to hang new work on | 13.8 the re-derive rule |

---

## 13.1 How this list was verified, and the three defects it found

Every arXiv identifier, DOI and URL in this course was checked against the source record. The script
is short enough to keep in CI, and a reading list that is not machine-checkable will be wrong within
a year:

```python
import urllib.request, urllib.parse, xml.etree.ElementTree as ET, json, re, glob

def arxiv_meta(ids):
    """Returns {id: (title, published, first_author)} for a batch of arXiv ids."""
    q = "http://export.arxiv.org/api/query?id_list=%s&max_results=100" % ",".join(ids)
    ns = {"a": "http://www.w3.org/2005/Atom", "arx": "http://arxiv.org/schemas/atom"}
    root = ET.fromstring(urllib.request.urlopen(q, timeout=60).read())
    out = {}
    for e in root.findall("a:entry", ns):
        eid = e.find("a:id", ns).text.rsplit("/", 1)[-1].split("v")[0]
        com = e.find("arx:comment", ns)
        out[eid] = (" ".join(e.find("a:title", ns).text.split()),
                    e.find("a:published", ns).text[:10],
                    e.find("a:author", ns).find("a:name", ns).text,
                    com.text if com is not None else None)      # venue often lives here
    return out

def doi_meta(doi):
    d = json.load(urllib.request.urlopen("https://api.crossref.org/works/" + doi))["message"]
    return d["title"][0], d.get("issued", {})["date-parts"][0][0], d.get("page")

ids = set()
for f in glob.glob("*.md"):
    ids |= set(re.findall(r"arxiv\.org/abs/([0-9]{4}\.[0-9]{4,5})", open(f).read()))
meta = arxiv_meta(sorted(ids))
missing = [i for i in ids if i not in meta]
assert not missing, f"arXiv has no record for: {missing}"
```

**Result: 13/13 arXiv identifiers used across chapters 00–12 resolve to the paper they are cited
as.** That is the good news, and it is the reason to record identifiers rather than titles.

Three defects surfaced anyway, and each teaches something different about citation hygiene.

**Defect 1 — a page count asserted from memory.** The draft described the RRF paper as "four pages".
Crossref:

```
10.1145/1571941.1572114 -> {'title': 'Reciprocal rank fusion outperforms condorcet and
  individual rank learning methods', 'authors': ['Cormack','Clarke','Buettcher'],
  'year': 2009, 'venue': 'Proceedings of the 32nd international ACM SIGIR conference...',
  'pages': '758-759', 'type': 'proceedings-article'}
```

Pages 758–759. It is a **two-page** paper. The claim was directionally fine ("short, read it today")
and factually wrong, which is the most common kind of citation error: harmless-feeling, and exactly
the sort of thing a reader checks when deciding whether to trust the rest of the document.

**Defect 2 — a paper cited by name with no identifier.** The draft listed *Agentic Context
Engineering (ACE) — ICLR 2026* with no arXiv ID, which makes the entry unverifiable by construction.
Resolving it:

```
2510.04618v3  2025-10-06  Agentic Context Engineering: Evolving Contexts for
                          Self-Improving Language Models   | Qizheng Zhang
    comment: ICLR 2026; 32 pages
```

The venue claim was **right** — the arXiv comment field says ICLR 2026. But nobody could have known
that from the draft, and an unverifiable true claim is indistinguishable from an unverifiable false
one. Identifiers are not pedantry; they are the only part of a citation a machine can check.

**Defect 3 — a failed lookup that looked like a missing paper.** Searching arXiv titles for
`"WorldLines"` returns three papers about noncommutative spacetime and none about agents, which is a
convincing-looking negative result. An abstract search finds it immediately:

```
2606.18847v2  2026-06-17  WorldLines: Benchmarking and Modeling Long-Horizon Stateful
                          Embodied Agents
```

The title-phrase search missed it because of the colon-subtitle format. **A failed lookup is not
evidence of absence** — it is evidence about your lookup. That is the same error class as §11.2's
compaction guard (which reported zero identifier loss while losing five of six) and §11.4's runner
(which reported 8/12 while checking one assertion key in three): a detector was trusted without its
own recall being measured.

**Link checking has a false-positive mode too.** Of 38 non-arXiv URLs cited across the course, 33
return HTTP 200 to a script. The other five:

```
302  https://ai.google.dev/gemini-api/docs/caching          (redirect, page fine)
302  https://ai.google.dev/gemini-api/docs/long-context      (redirect, page fine)
403  https://dl.acm.org/doi/10.1145/1571941.1572114          (bot block, page fine)
403  https://help.openai.com/en/articles/8590148-memory-faq  (bot block, page fine)
403  https://openai.com/index/memory-and-new-controls-for-chatgpt/ (bot block, page fine)
```

Zero dead links; five automated-client refusals. A CI link checker that treats 403 as broken will
cry wolf on exactly the sources most worth citing — vendor docs and paywalled proceedings. Follow
redirects, allowlist 403 from known bot-blocking hosts, and verify the DOI through Crossref instead
of the publisher's HTML.

---

## 13.2 The core sequence — six papers, in this order

Read these six before anything else. They are not the six best papers; they are the six that make
the rest legible, and the order matters because each is best understood as a reaction to the
previous one.

```
   MemGPT (2023-10)                  Generative Agents (2023-04)
   memory as an OS problem           reflection: memory that generalises
        │                                     │
        └──────────────┬──────────────────────┘
                       ▼
            Zep / Graphiti (2025-01)           Mem0 (2025-04)
            time as a first-class axis         write-path as four operations
                       │                                │
                       └────────────┬───────────────────┘
                                    ▼
                      LongMemEval (2024-10)  ->  LoCoMo (2024-02)
                      how to measure it          how to build the benchmark
```

**1. MemGPT: Towards LLMs as Operating Systems** — Packer et al., 2023-10-12,
[arXiv:2310.08560](https://arxiv.org/abs/2310.08560)
The founding framing: main context as physical memory, external stores as virtual memory, the model
paging between tiers through tool calls. Even if you never run Letta, this vocabulary shapes every
system that followed (ch 01, 06.4, 10.2).
*Read for:* the memory hierarchy, self-editing memory, and the function-call loop with heartbeats.
*What it does not say:* how to decide **what** is worth paging in — salience is assumed. Chapter 04
exists because of that gap.

**2. Generative Agents: Interactive Simulacra of Human Behavior** — Park et al., 2023-04-07,
[arXiv:2304.03442](https://arxiv.org/abs/2304.03442)
The reflection paper: a memory stream, retrieval scored by recency + importance + relevance, and
periodic reflection producing higher-level insights that can themselves be reflected on. Its
ablation — observation, planning and reflection each contributing critically — is the evidence that
reflection is load-bearing rather than decorative (ch 06.2).
*Read for:* the reflection tree and the three-component retrieval score.
*What it does not say:* anything about cost control or multi-tenancy. It is a simulation of 25
agents, not a service for 10M users.

**3. Zep: A Temporal Knowledge Graph Architecture for Agent Memory** — Rasmussen et al., 2025-01-20,
[arXiv:2501.13956](https://arxiv.org/abs/2501.13956)
The best-documented production architecture in the open literature: bi-temporal edges, a three-tier
subgraph hierarchy, edge invalidation instead of deletion, composed retrieval with reranking (ch
05.2, 05.5, 10.4).
*Read for:* bi-temporality, the entity/fact/community tiers, and the systems framing of "millions of
small graphs" rather than one large one.
*What it does not say:* what the graph costs you when your queries are single-hop — chapter 05.4's
decision rule is the missing half.

**4. Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory** — Chhikara et al.,
2025-04-28, [arXiv:2504.19413](https://arxiv.org/abs/2504.19413); ECAI 2025
([DOI 10.3233/FAIA251160](https://doi.org/10.3233/FAIA251160))
The cleanest statement of the extraction → reconciliation write path with ADD/UPDATE/DELETE/NOOP,
plus a graph variant, plus the accuracy-versus-token-cost framing this course adopts (ch 04.5,
04.10, 10.3).
*Read for:* the two-stage write path and the >90% token-cost reduction argument.
*What it does not say:* what the shipping code does now. Chapter 10.3 documents the divergence —
the current pipeline is ADD-only with linking, and `delete_linked=False` leaves erasure residue.
**Read the paper for the architecture and the repository for the behaviour.**

**5. LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory** — Wu et al.,
2024-10-14, [arXiv:2410.10813](https://arxiv.org/abs/2410.10813); ICLR 2025
The benchmark that fixed the ability taxonomy everyone now uses: information extraction,
multi-session reasoning, temporal reasoning, knowledge updates, and **abstention** (ch 08.2, 08.6).
*Read for:* the category design, and especially abstention — the single design choice most worth
copying into your own eval set.
*What it does not say:* how your traffic is distributed. Its category mix is not yours, and §8.3 is
about why that matters more than the leaderboard.

**6. Evaluating Very Long-Term Conversational Memory of LLM Agents (LoCoMo)** — Maharana et al.,
2024-02-27, [arXiv:2402.17753](https://arxiv.org/abs/2402.17753)
The multi-session conversation benchmark, constructed through a machine–human pipeline over persona
and event graphs.
*Read for:* how to **construct** a memory benchmark, which you will need for your own domain (ch
08.2, project S6).
*What it does not say:* anything that survives being quoted without its actor model and judge. §8.3
has the reasons published LoCoMo numbers are not comparable across papers.

---

## 13.3 Then, by topic

### Context and compaction (ch 02)

- **Context Rot: How Increasing Input Tokens Impacts LLM Performance** — Chroma, 2025
  ([research.trychroma.com/context-rot](https://research.trychroma.com/context-rot)). The empirical
  basis for "smaller curated context beats larger complete context". Vendor-published but with
  released methodology; treat the direction as solid and the magnitudes as model-specific.
- **Context Length Alone Hurts LLM Performance Despite Perfect Retrieval** — Du et al., 2025-10-06,
  [arXiv:2510.05381](https://arxiv.org/abs/2510.05381). Isolates length from retrieval quality —
  the cleanest experiment on the question, and the one to cite when someone says "long context
  solves memory".
- **Effective context engineering for AI agents** — Anthropic, 2025
  ([anthropic.com/engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).
  Compaction, structured note-taking, sub-agent isolation. Short, practical, and honest about
  trade-offs.
- **Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models** — Zhang et
  al., 2025-10-06, [arXiv:2510.04618](https://arxiv.org/abs/2510.04618); ICLR 2026. Frames context
  as an evolving collection of delta entries rather than a monolith rewritten each round — the
  principled fix for the collapse mechanism in 02.3.
- **Parallel Context Compaction for Long-Horizon LLM Agent Serving** — Cim et al., 2026-05-22,
  [arXiv:2605.23296](https://arxiv.org/abs/2605.23296). The serving-systems view of compaction.
  Read it if you operate the infrastructure; skip it if you call an API.
- **Context Engineering for AI Agents: Lessons from Building Manus** — Manus, 2025
  ([manus.im/blog](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)).
  A practitioner log. Unusually specific about KV-cache economics and file-system-as-context.

### Retrieval (ch 03)

- **Reciprocal Rank Fusion outperforms Condorcet and individual rank learning methods** — Cormack,
  Clarke & Buettcher, SIGIR 2009, pp. 758–759
  ([DOI 10.1145/1571941.1572114](https://dl.acm.org/doi/10.1145/1571941.1572114)). **Two pages** —
  the source of `k=60`. Read it this afternoon and you will use it for the rest of your career.
  *What it does not say:* that `k=60` is optimal for you. §11.3 shows the rank-1:rank-2 weight ratio
  at `k=60` is 1.016, i.e. deliberately almost rank-insensitive; that is a design choice you should
  make knowingly.
- **Dense Passage Retrieval for Open-Domain Question Answering** — Karpukhin et al., 2020-04-10,
  [arXiv:2004.04906](https://arxiv.org/abs/2004.04906). The bi-encoder baseline everything is
  measured against.
- **ColBERT** — Khattab & Zaharia, 2020-04-27, [arXiv:2004.12832](https://arxiv.org/abs/2004.12832)
  and **ColBERTv2** — Santhanam et al., 2021-12-02,
  [arXiv:2112.01488](https://arxiv.org/abs/2112.01488). Late interaction: the middle ground between
  bi-encoders and cross-encoders. Worth knowing when your rerank budget is tight.
- **Efficient and robust approximate nearest neighbor search using HNSW graphs** — Malkov &
  Yashunin, 2016-03-30, [arXiv:1603.09320](https://arxiv.org/abs/1603.09320). Read sections 3–4 so
  `M` and `ef` stop being magic numbers (ch 03.3).
- **From Local to Global: A Graph RAG Approach to Query-Focused Summarization** — Edge et al.,
  2024-04-24, [arXiv:2404.16130](https://arxiv.org/abs/2404.16130). Community detection plus
  hierarchical summarisation: the direct ancestor of the community tier in graph memory (ch 05.5).
- **Filtered HNSW, in practice** — [Qdrant on filterable
  HNSW](https://qdrant.tech/articles/filterable-hnsw/) and [Supabase on pgvector HNSW
  tuning](https://supabase.com/blog/increase-performance-pgvector-hnsw). The filtered-ANN
  percolation problem in 03.4 is explained better in vendor engineering posts than in papers,
  because it is an operational failure rather than a research result.

### Procedural memory and background compute (ch 06)

- **Sleep-time Compute: Beyond Inference Scaling at Test-time** — Lin et al., 2025-04-17,
  [arXiv:2504.13171](https://arxiv.org/abs/2504.13171). Reason about a context offline, before
  queries arrive: ~5× less test-time compute for equal accuracy, +13%/+18% on their stateful
  benchmarks, 2.5× lower average cost per query when amortised across related queries.
  *What it does not say* — and this is the important part — that it always pays. Efficacy correlates
  with how predictable the query is. Background work on a context nobody asks about is pure cost
  (ch 06.3).
- **Claude Code memory / `CLAUDE.md`** ([code.claude.com/docs](https://code.claude.com/docs/en/memory))
  and the **memory tool** ([platform.claude.com/docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)),
  plus **compaction** and **context editing** docs. The procedural-memory-as-a-file design in 06.1
  and the measured cost of always-in-context files in 10.7 both come from here.
- **A-MEM: Agentic Memory for LLM Agents** — Xu et al., 2025-02-17,
  [arXiv:2502.12110](https://arxiv.org/abs/2502.12110). Agent-constructed memory links rather than
  a fixed schema. Read as the counter-position to typed categories (04.2) — and note what it gives
  up: you cannot enforce an invariant on a structure the model invents at write time.
- **Titans: Learning to Memorize at Test Time** — Behrouz et al., 2024-12-31,
  [arXiv:2501.00663](https://arxiv.org/abs/2501.00663). Memory in the weights instead of in a
  store. Adjacent to this course rather than in it, but it is the honest answer to "will
  architectures make all of this unnecessary?" — read it so your answer is informed rather than
  defensive.

### Time and graphs (ch 05)

- **Zep** (above) is the primary source. Pair it with the **Graphiti repository**
  ([github.com/getzep/graphiti](https://github.com/getzep/graphiti)) — 10.4's audit found the four
  timestamps really are in the edge schema, which is rarer than you would hope.
- **Temporal Data & the Relational Model** — Date, Darwen & Lorentzos (book, below). The formal
  grounding: bi-temporality is settled 1990s database technology, not a 2025 invention.
- **State of the Art: Agent Memory** — Zep, 2025 ([blog.getzep.com](https://blog.getzep.com/state-of-the-art-agent-memory/)).
  Vendor survey; useful for orientation, and a good exercise in spotting which comparisons are
  apples-to-apples (§8.11).

### Security and privacy (ch 09)

- **A Survey on Long-Term Memory Security in LLM Agents: Attacks, Defenses…** — Lin et al.,
  2026-04-17, [arXiv:2604.16548](https://arxiv.org/abs/2604.16548). Organised by lifecycle phase:
  write, retrieve, forget. The FORGET section on incomplete deletion and residual state is required
  reading **before** you design an erasure path — §9.6 measured seven residue sites after a naive
  DELETE, and the survey predicts most of them.
- **MemoryGraft: Persistent Compromise of LLM Agents via Poisoned Experience Retrieval** —
  Srivastava et al., 2025-12-18, [arXiv:2512.16962](https://arxiv.org/abs/2512.16962). Why
  trajectory memory is the most dangerous memory type: the attack is a *write*, and every later
  session pays for it (ch 06.5, 09.3 T3).
- **Trojan Hippo: Weaponizing Agent Memory for Data Exfiltration** — Das et al., 2026-05-03,
  [arXiv:2605.01970](https://arxiv.org/abs/2605.01970). Memory as an exfiltration channel (09.3 T4).
- **OWASP Top 10 for Agentic Applications**, ASI06 Memory & Context Poisoning
  ([genai.owasp.org](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/)). The
  control set your security reviewer will cite, so read it before they do.
- **GDPR, the six articles that actually bind you**: [Art. 9](https://gdpr-info.eu/art-9-gdpr/)
  (special categories), [12](https://gdpr-info.eu/art-12-gdpr/) (the one-month clock),
  [15](https://gdpr-info.eu/art-15-gdpr/) (access), [16](https://gdpr-info.eu/art-16-gdpr/)
  (rectification), [17](https://gdpr-info.eu/art-17-gdpr/) (erasure),
  [20](https://gdpr-info.eu/art-20-gdpr/) (portability). Read the article text, not a summary blog
  post; it is short and the summaries consistently drop the deadlines (§9.6).

### Evaluation and the benchmark frontier (ch 08)

- **BEAM / Beyond a Million Tokens** — Tavakoli et al., 2025-10-31,
  [arXiv:2510.27246](https://arxiv.org/abs/2510.27246),
  [code](https://github.com/mohammadtavakoli78/BEAM). Million-to-ten-million-token memory
  evaluation.
- **HaluMem: Evaluating Hallucinations in Memory Systems of Agents** — Chen et al., 2025-11-05,
  [arXiv:2511.03506](https://arxiv.org/abs/2511.03506). Memory consistency and fabrication — the
  failure mode that accuracy metrics hide.
- **RealMem: Benchmarking LLMs in Real-World Memory-Driven Interactions** — Bian et al.,
  2026-01-11, [arXiv:2601.06966](https://arxiv.org/abs/2601.06966). Project-oriented long-term
  interaction.
- **WorldLines: Benchmarking and Modeling Long-Horizon Stateful Embodied Agents** — 2026-06-17,
  [arXiv:2606.18847](https://arxiv.org/abs/2606.18847). Where the frontier is going: agentic,
  tool-using, multi-week, stateful.
- **The Tail at Scale** — Dean & Barroso, Google, 2013
  ([research.google](https://research.google/pubs/the-tail-at-scale/)). Not a memory paper. It is
  the reason §7.4's fan-out arithmetic works the way it does, and the single most useful thing to
  read before writing a latency budget.

---

## 13.4 Vendor documentation, and how to read it

Vendor docs are the only public source for how memory behaves at consumer scale, and they are also
marketing. Both things are true at once, so read them with a fixed set of questions.

| Source | Read it for | Read it sceptically for |
|---|---|---|
| [OpenAI memory FAQ](https://help.openai.com/en/articles/8590148-memory-faq) + [announcement](https://openai.com/index/memory-and-new-controls-for-chatgpt/) | The five places deleted content can persist, stated by the vendor itself (10.8) | Retention specifics, which change without notice |
| [Anthropic context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), [compaction](https://platform.claude.com/docs/en/build-with-claude/compaction), [context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing) | Concrete compaction and note-taking mechanics | Generalisation to other models |
| [AWS Bedrock AgentCore memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-organization.html) + [strategies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-strategies.html) | A managed service's memory taxonomy — a useful cross-check on your own categories | Whether the taxonomy fits your domain |
| [Gemini Enterprise memory bank](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank), [context caching](https://ai.google.dev/gemini-api/docs/caching), [long context](https://ai.google.dev/gemini-api/docs/long-context) | Cache pricing mechanics — the arithmetic behind 10.7's 7.8× cache win | "Long context replaces memory" framing |
| [Cursor semantic search](https://cursor.com/blog/semsearch) | A rare published A/B of retrieval on real developer traffic (03.8) | Transfer to non-code domains |
| [OpenAI pricing](https://developers.openai.com/api/docs/pricing) + [prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) | The rates every cost model in this course uses | Staleness — re-derive, always |
| [Postgres 16 RLS](https://www.postgresql.org/docs/16/ddl-rowsecurity.html) | The isolation primitive, including `BYPASSRLS` and table-owner exemption (7.3) | Nothing; this one is just correct |

Three questions to bring to any vendor memory claim, from §8.3: what was the **actor model**, what
was the **judge**, and what was the **token cost**? A claim missing any of the three is not
comparable to anything, including its own previous version.

---

## 13.5 Books

There is still no good book on agentic memory — the field is too young. These are the books that
give you the underlying competencies, and each is worth more than another twenty blog posts.

**1. *Designing Data-Intensive Applications* — Kleppmann.** The most important book on this list and
it never mentions LLMs. Replication, partitioning, transactions, consistency, batch versus stream.
Chapters 5–7 and 11 map directly onto chapter 07. If you read one, read this.

**2. *Introduction to Information Retrieval* — Manning, Raghavan & Schütze**
([free online](https://nlp.stanford.edu/IR-book/)). The IR foundations behind chapter 03: indexing
(1–2), scoring (6), evaluation (8), probabilistic models and BM25 (11). The evaluation chapter alone
will improve how you measure retrieval — and chapter 11's negative-IDF bug is a direct consequence
of reading the BM25 formula without reading what Lucene actually ships.

**3. *Database Internals* — Petrov.** Storage engines, B-trees, LSM trees, distributed transactions.
Read it to understand why your vector index behaves as it does under deletes, and why compaction is
a universal pattern rather than an LLM one.

**4. *Temporal Data & the Relational Model* — Date, Darwen & Lorentzos.** Dense and old; the
rigorous treatment of bi-temporal modelling chapter 05 rests on. Snodgrass's *Developing
Time-Oriented Database Applications in SQL* is the practical alternative if the formalism is heavy.

**5. *Site Reliability Engineering* — Beyer et al.** ([free online](https://sre.google/books/)).
SLOs, error budgets, and the operational discipline behind chapter 07's budgets and runbooks.

**6. *AI Engineering* — Chip Huyen.** The best general treatment of the surrounding engineering:
evaluation, data flywheels, deployment. Read the evaluation chapters alongside chapter 08.

**7. *Memory: A Very Short Introduction* — Foster.** Two hours. Gives you the
episodic/semantic/procedural vocabulary properly rather than second-hand, so you can tell when a
paper's cognitive analogy is load-bearing and when it is decoration.

---

## 13.6 Repositories to read

Ordered by reading value per hour, not by popularity — 10.0 shows star counts spread 39× across
these with no relationship to the property you care about.

**1. [`getzep/graphiti`](https://github.com/getzep/graphiti)** — compact, legible, and the
bi-temporal invalidation logic is the single most transferable code in the ecosystem. Start at edge
invalidation and temporal extraction (10.4).

**2. [`mem0ai/mem0`](https://github.com/mem0ai/mem0)** — read the extraction and update prompts
specifically; they are short and encode a lot of hard-won behaviour. Compare against your own from
project S3, then compare the code against the paper (10.3).

**3. [`letta-ai/letta`](https://github.com/letta-ai/letta)** — the agent loop, memory blocks, and
the sleep-time agent. Large; navigate via the docs rather than reading top-down. Note the repository
split and the git-backed MemFS direction (10.2).

**4. [`langchain-ai/langgraph`](https://github.com/langchain-ai/langgraph)** — the `BaseStore`
interface and the checkpointer. The namespace design, the thread-state/store separation, and the one
real foreign-key cascade found in the 10.5 audit.

**5. [`pgvector/pgvector`](https://github.com/pgvector/pgvector)** — the index implementation.
Reading how HNSW is built and queried *inside a transactional database* demystifies a great deal
about deletes, vacuum and recall drift.

**6. [`topoteretes/cognee`](https://github.com/topoteretes/cognee)** — pipeline memory as explicit,
re-runnable ETL, plus the declared-cardinality contradiction task that 10.6 calls the most honest
file in the audit.

**Read every one with the five questions from 10.11:** where is the write path, what is the conflict
policy, how is tenancy enforced, what runs in the background, and what does deletion actually do.
Then add a sixth, earned by chapter 11: **can their test suite fail?** Grep for the negative
assertions and ask what positive control proves the mechanism could have violated them.

---

## 13.7 Courses and talks

- **DeepLearning.AI — *LLMs as Operating Systems: Agent Memory*** (with Letta). Short, hands-on,
  and the fastest way to get the MemGPT model into your fingers.
- **Anthropic's engineering blog** — the context-engineering and multi-agent research posts.
  Practitioner-written, specific, unusually honest about trade-offs.
- **Vector-database vendor talks on filtered ANN search** — the 03.4 filtered-HNSW problem is best
  explained in conference talks, because it is an operational war story rather than a result.

---

## 13.8 How to keep current without drowning

You are 54× over budget (§13.0), so the filter is the skill. Mine, in order of leverage:

**1. Track changelogs, not announcements.** Four or five repositories' commit histories tell you
what is actually being built. 10.3's finding — the shipped pipeline diverging from the published
paper — came from the code, and no blog post would have said it.

**2. Triage in twenty minutes, with a fixed rubric.** For any new paper, answer four questions
before reading the body:

```
1. Which of the five write-path decisions or six read-path stages does it change?   (ch 01)
2. Does it report accuracy AND tokens AND latency?                                  (rule 5)
3. Is the comparison paired, and is the judge's agreement stated?                   (ch 08.7-8.8)
4. What does it cost to run at my scale?                                            (ch 07.8)
```

Two "no"s and you skim the abstract and move on. This is not dismissiveness; it is the only way to
spend four hours on the 1% of 325 papers that change a decision you are actually making.

**3. Read benchmark papers, skip benchmark press releases.** If a claim does not report the actor
model, the judge and the token cost, it is not comparable to anything (§8.3).

**4. Follow the security literature closely.** It moves fast, it is empirical, and it describes
failure modes years before the product literature admits to them. Chapter 09's threat catalogue is
downstream of three papers, all from the last year.

**5. Re-derive, do not adopt.** When a technique appears, place it in the chapter-01 pipeline.
Almost everything is a refinement of one box. If you cannot place it, either it is genuinely novel —
rare, and worth deep attention — or it is a rebrand of a box you already have.

**6. Keep a decisions log, not a papers log.** One line per paper you read: *what decision would
this change?* Most entries will say "none", which is the correct and useful answer, and the few that
say something are the reason you read at all.

---

## 13.9 Failure modes of reading this field

| Symptom | Root cause | Fix |
|---|---|---|
| Constant feeling of being behind | measuring against 325 papers/month | 13.0 accept 54× over; triage instead |
| Read 40 papers, built nothing | no dependency order, no decisions log | 13.2 the six, in order; 13.8 rule 6 |
| Implemented a paper, behaviour differs | paper describes v1, code ships v3 | 13.2 Mem0 entry; read code for behaviour |
| Quoted a benchmark number that was wrong | no actor model / judge / cost | 13.8 rule 3, §8.3 |
| Citation in your doc turns out wrong | claim asserted from memory | 13.1 mechanical verification in CI |
| "That paper doesn't exist" | title search failed on a subtitle | 13.1 defect 3; search abstracts too |
| Link checker floods CI with failures | 403 from bot-blocking publishers | 13.1 allowlist, verify DOIs via Crossref |
| Adopted a graph/framework because a paper used one | no placement in your own pipeline | 13.8 rule 5, §05.4, §10.10 |
| Vendor doc read as a benchmark | marketing and documentation in one file | 13.4 the three questions |

---

## 13.10 Exercises

1. **Verify this chapter.** Run the §13.1 script over the repository. Acceptance: every arXiv ID
   resolves, and you can state the count — 13/13 for chapters 00–12, and **24/24 once this
   chapter's identifiers are included**, on 2026-09-12. Any ID that fails is either a typo or a
   withdrawn paper; both are worth knowing.
2. **Add link checking to CI.** Extend the script to non-arXiv URLs with redirect-following and a
   403 allowlist. Acceptance: the checker reports zero false positives on the five bot-blocked URLs
   in §13.1, and fails if you deliberately corrupt one URL.
3. **Count your own firehose.** Re-run the arXiv volume query for the sub-topic you care about.
   Acceptance: papers/month, your reading hours/month, and the ratio — written down where you will
   see it when you feel behind.
4. **Triage ten papers in one hour.** Apply §13.8's four questions to ten recent arXiv memory
   papers. Acceptance: ten one-line verdicts, at most two marked "read fully", and for each of those
   two, the decision it would change.
5. **Read the two-page paper.** Read RRF (pp. 758–759) and implement fusion from the paper alone, no
   blog posts. Acceptance: your implementation matches §11.3's output on the worked example, and you
   can explain why `k=60` makes the rank-1:rank-2 ratio 1.016.
6. **Paper versus code.** Pick one system from chapter 10, read its paper's write-path section, then
   read the corresponding code. Acceptance: one written difference between the two, with file and
   line. There will be one; 10.3 found several.
7. **Start the decisions log.** For the next five papers you read, write the single line: what
   decision would this change? Acceptance: five lines, and at least three of them say "none" — if
   every paper changes your design, you are not evaluating, you are collecting.

Next: `14-design-review-playbook.md` — the questions that turn all of this into a review, and the
rubric that scores it.
