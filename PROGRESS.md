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
| 05 | `05-temporal-and-graph-memory.md` | 🟡 draft | 405 | Needs: plain-words opener on two clocks, bi-temporal SQL worked example, Graphiti invalidation walkthrough, entity-resolution numbers |
| 06 | `06-procedural-and-reflective.md` | 🟡 draft | 355 | Needs: reflection loop opener, Generative Agents + sleep-time-compute evidence, a runnable reflection job |
| 07 | `07-systems-design.md` | 🟡 draft | 415 | Needs: latency budget table with real p99 arithmetic, sharding worked example, backfill runbook |
| 08 | `08-evaluation.md` | 🟡 draft | 331 | Needs: LoCoMo/LongMemEval/BEAM specifics with links, a runnable harness, CI gate thresholds |
| 09 | `09-security-privacy-governance.md` | 🟡 draft | 348 | Needs: injection→memory-poisoning worked attack, GDPR cascade checklist across derived state |
| 10 | `10-case-studies.md` | 🟡 draft | 344 | Needs: re-verified snapshots (this file churns fastest), architecture diagrams per system |
| 11 | `11-projects-small.md` | 🟡 draft | 191 | Add acceptance tests that reuse ch04 scenario runner |
| 12 | `12-projects-capstone.md` | 🟡 draft | 255 | Fine as-is; revisit after 07/08 deep pass |
| 13 | `13-reading-list.md` | 🟡 draft | 186 | Add the papers cited in the deep passes (Mem0, Zep, Cursor semsearch) |
| 14 | `14-design-review-playbook.md` | 🟡 draft | 190 | Add write-path questions from 04.12 failure table |

---

## Resume here

**Next chapter to deepen: `05-temporal-and-graph-memory.md`.**

It is the natural continuation — chapter 04 ends by promising that `valid_from` and `created_at`
become two independent clocks, and exercise 5 hands chapter 05 a query to extend.

Target shape for the 05 deep pass (same as 03 and 04):

1. `5.0 In plain words` — the two clocks (when it was true vs when we learned it), explained with a
   single everyday example before any SQL.
2. A bi-temporal table + the four canonical queries (current belief, belief as of date T, truth as
   of date T, full audit trail) with runnable SQL.
3. Graphiti/Zep edge invalidation walked through step by step, citing
   [arXiv:2501.13956](https://arxiv.org/abs/2501.13956).
4. Entity resolution at graph scale — where 4.4's scoring is reused, and what changes.
5. When a graph is *not* worth it (the honest section; most systems do not need one).
6. Failure-mode table + exercises that build on the 04 store.

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

---

## Change log

| Date | Commit | What |
|---|---|---|
| 2026-09-10 | `832a1f5` | Initial build: 14 chapters, projects, reading list, playbook |
| 2026-09-10 | `fd736a6` | Root README as GitHub landing page |
| 2026-09-10 | `214cbba` | Chapter 00 (why memory) + plain-English on-ramps for 01–02 |
| 2026-09-10 | `cd84d3f` | Chapter 03 deep pass: HNSW tuning, filtered ANN, Cursor evidence |
| 2026-09-10 | _this_ | Chapter 04 deep pass + this progress tracker |
