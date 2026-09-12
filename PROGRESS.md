# PROGRESS

Single source of truth for what is written, what is still first-draft, and where to resume.

**Legend**
- ✅ **deep** — rewritten to the target standard: plain-words opener, runnable snippets, cited
  evidence with links, worked numbers, failure-mode table, exercises.
- 🟡 **draft** — solid first-pass scaffold from the initial build. Correct and useful, but no
  plain-words on-ramp, thinner evidence, fewer worked examples.
- ⬜ **not started**

---

## Chapter status

| # | File | Status | Lines | Notes on what a "deep" pass added / still needs |
|---|---|---|---|---|
| 00 | `00-README.md` | ✅ deep | 161 | Roadmap, 12-week plan, environment setup |
| 00 | `00-why-memory.md` | ✅ deep | 390 | The "why" in three registers + cost arithmetic |
| 01 | `01-foundations.md` | ✅ deep | 355 | Plain-English on-ramp added |
| 02 | `02-context-engineering.md` | ✅ deep | 523 | Plain-English on-ramp added |
| 03 | `03-retrieval-fundamentals.md` | ✅ deep | 563 | Two-librarians opener, pgvector `ef_search` curve, filtered-ANN percolation, Cursor evidence |
| 04 | `04-memory-write-path.md` | ✅ deep | 1107 | Notebook opener, gate cascade (with the two regex bugs it fixes), worked extraction example, threshold calibration script, supersession trace, destructive budget, confidence calibration, cost model, 12 scenarios + gate recall set, failure table |
| 05 | `05-temporal-and-graph-memory.md` | ✅ deep | 680 | Two-clocks opener, SQLite-verified bitemporal queries (truth-vs-belief divergence), DB invariants, interval repair (overlap/gap/extend), honest graph decision rule, Graphiti properties cited from source |
| 06 | `06-procedural-and-reflective.md` | ✅ deep | 527 | Recall arithmetic for why rules belong in context (0.9^N), rule precedence + token budget, `rule_health` for superstition, reflection quality gate, sleep-time-compute numbers and when it does NOT pay, trajectory distillation, hints≠permissions |
| 07 | `07-systems-design.md` | ✅ deep | 995 | Library-counter opener, real PG16 RLS lab (owner + superuser bypass, fail-closed GUC, partition pruning under RLS), verified percentile composition, the `TaskGroup` deadline bug measured at 901ms vs 120ms budget, Little's-law capacity plan, shard-balance simulation, M/M/1 freshness arithmetic, priced cost model, re-embedding runbook, citation-rate instrument |
| 08 | `08-evaluation.md` | ✅ deep | 771 | Driving-test opener, bootstrap CI table (±8pp at n=100), six metrics all called "recall@5" from one run, judge attenuation (2q−1) verified against theory, paired McNemar vs unpaired power table (75% vs 6% at n=500), corrected LoCoMo/LongMemEval/BEAM specifics with links, deterministic-vs-stochastic CI gates, priced eval run |
| 09 | `09-security-privacy-governance.md` | ✅ deep | 683 | Filing-cabinet opener, executed two-session poisoning chain (extract → store → retrieve → destructive action) and its blocked replay, persistence arithmetic 1−(1−p)^N, real SQLite/FTS5 erasure cascade showing 7 residue sites after a naive DELETE and the contentless-FTS5 bug that forces capture-before-delete, GDPR article map with Art. 12(3) clock, min-trust inheritance, incident time boxes |
| 10 | `10-case-studies.md` | ✅ deep | 1072 | Filing-cabinet opener, star-count arithmetic (39× spread uncorrelated with the property that matters), executed P1–P5 mechanism-probe audit over 5,699 source files in six live clones + the two bugs it hit (`valid_to` inside `invalid_tool_message`; grep finding docstrings not mechanisms), Letta's repo split and git-backed MemFS, Mem0's V3 ADD-only pipeline + `delete_linked=False` erasure residue + NOOP→NONE correction, Graphiti's four timestamps from `edges.py`, LangGraph's real `store_vectors` FK cascade, Cognee's declared-cardinality contradiction task, tiktoken-measured cost of always-in-context files (7.8× cache win), ChatGPT's own five-site erasure warning. Over the 900-line target: seven systems, each with a diagram + source evidence |
| 11 | `11-projects-small.md` | ✅ deep | 828 | Logbook opener, `memlab` shared harness (triple + bootstrap CI + one metric definition + mutation score), measured 500-turn strategy table (23.7× SendAll vs window, 16.7× vs window+summary, fact leaves window at turn 15/25, 1.92× messages-vs-exchanges bug), the compaction-guard regex that PASSES while losing 5 of 6 identifiers (recall 1/6), BM25 negative-Robertson-IDF ranking the gold memory below a restaurant doc, zero-score arm voting in RRF, six recall@5 definitions spanning 0.286–1.000 on one run, offline scenario harness 8/12 → 6/12-verified → 12/12 with mutation score 2/5 → 5/5 via positive controls, bitemporal 6/6 vs flat 1/6 NOT-EXPRESSIBLE, inline-vs-background p95 2.32× and the 26% p95-addition error, synthetic-compression caveat, eval sizing (n=10 inverts 23% of runs) |
| 12 | `12-projects-capstone.md` | ✅ deep | 671 | Restaurant-vs-dish opener, scoping arithmetic (143 h needed vs 60–90 h evenings = 1.6–2.4× over, with a cut order), load-test sizing (a 100-request test passes a truly-164ms-p99 system 44% of the time; 60k samples gives ±2ms), Little's-law pool sizing + int8 storage table (337→107 GB at 50M rows, 13.5 TB at 2B), design doc with a required number per section, measured churn study of mem0/graphiti/langgraph (20–63% of files stale at 90 days; mean-rate Poisson TTL over-invalidates 2.1–2.6× because the top 10% of files carry 37–56% of edits) → hash/symbol invalidation, `repeated_dead_ends` defined in code, handoff fidelity requirement p ≥ 0.928 for 80% over 3 hops, contamination sim (85% → 0% blast radius above trust 1), rubric-mapping table, capstone failure modes, pre-week-one exercises |
| 13 | `13-reading-list.md` | 🟡 draft | 186 | Add the papers cited in the deep passes (Mem0, Zep, Cursor semsearch) |
| 14 | `14-design-review-playbook.md` | 🟡 draft | 190 | Add write-path questions from 04.12 failure table |

---

## Resume here

**Next chapter to deepen: `13-reading-list.md`.** Then 14. Deep passes done so far: 00–12.

What each remaining file needs (keep the same shape as 04/05/06):

- **07 — systems design.** DONE (2026-09-12).
- **08 — evaluation.** DONE (2026-09-12).
- **09 — security/privacy.** DONE (2026-09-12).
- **10 — case studies.** DONE (2026-09-12). Snapshot date stated in the file; re-run §10.1 to refresh.
- **11 — small projects.** DONE (2026-09-12). Five verified bugs; `memlab` harness is now the
  contract the capstones and the playbook should reference.
- **12 — capstones.** DONE (2026-09-12). Churn study is a snapshot; re-run §12.3 on the target repo.
- **13–14 — practice.** The reading list gets every source cited across 00–12 (including the ones
  the deep passes added: Cursor semsearch, context-rot, sleep-time compute, MemoryGraft, OWASP
  ASI06, RRF, Lucene's IDF variant) plus what to take from each; the playbook gets the
  write-path/temporal/security questions from the new failure tables, the "can your suite fail?"
  question §11.4 earned, and the measurement-protocol questions §12.2 earned.

---

## Standing rules for every chapter (do not drift)

1. **Plain words first.** Every chapter opens with a `X.0 In plain words` section that an engineer
   with zero memory-systems background can read in five minutes. Analogy → naive version →
   where it breaks → what the chapter fixes.
2. **Naive version before the abstraction.** Show the 20-line version and its arithmetic failure
   before introducing the real design.
3. **Every number is cited or derived.** Benchmark claims get a link (arXiv, vendor engineering
   blog). Cost/latency claims show the arithmetic inline. No unsourced "studies show".
4. **Snippets are runnable and small.** Real Python/SQL, no pseudo-code, no framework imports
   before chapter 10.
5. **Three units, always.** Accuracy, tokens, latency. A claim missing one of them is marketing.
6. **Every chapter ends with** a failure-mode table (symptom → cause → fix) and exercises with
   observable acceptance criteria.
7. **One agent, one chapter at a time**, committed separately with a descriptive message.
8. **Verify before you write.** Run the snippet / SQL / arithmetic in a scratch kernel and paste the
   *actual* output into the chapter. Bugs found while verifying (see 4.2's gate regex, 5.2's
   truth-vs-belief divergence) become the most valuable paragraphs in the file.

---

## Change log

| Date | Commit | What |
|---|---|---|
| 2026-09-10 | `832a1f5` | Initial build: 14 chapters, projects, reading list, playbook |
| 2026-09-10 | `fd736a6` | Root README as GitHub landing page |
| 2026-09-10 | `214cbba` | Chapter 00 (why memory) + plain-English on-ramps for 01–02 |
| 2026-09-10 | `cd84d3f` | Chapter 03 deep pass: HNSW tuning, filtered ANN, Cursor evidence |
| 2026-09-10 | `d847689` | Chapter 04 deep pass + this progress tracker |
| 2026-09-12 | `f28dbb1` | Chapter 05 deep pass: two clocks, SQL-verified bitemporal queries, interval repair |
| 2026-09-12 | `017f19d` | Chapter 06 deep pass: procedural recall arithmetic, sleep-time compute, trajectory safety |
| 2026-09-12 | `85964f9` | Chapter 07 deep pass: verified RLS isolation lab, latency composition, capacity/shard/cost arithmetic, deadline bug |
| 2026-09-12 | `dd98090` | Chapter 08 deep pass: verified statistics (CI widths, McNemar power, judge attenuation), metric-definition table, CI gate design |
| 2026-09-12 | `c43276a` | Chapter 09 deep pass: executed poisoning chain, SQLite erasure-cascade residue measurement, trust inheritance |
| 2026-09-12 | `d4b16b4` | Chapter 10 deep pass: executed source audit of six live clones, two probe bugs, Letta repo split + MemFS, Mem0 V3 ADD-only + delete_linked default, priced always-in-context files |
| 2026-09-12 | `97a241e` | Chapter 11 deep pass: `memlab` harness, five verified bugs (guard recall 1/6, negative IDF, zero-score arm, ignored assertions, mutation score 2/5), measured project baselines |
| 2026-09-12 | `pending` | Chapter 12 deep pass: capstone scoping arithmetic, load-test sizing, measured repo churn → hash invalidation, handoff/contamination arithmetic |
