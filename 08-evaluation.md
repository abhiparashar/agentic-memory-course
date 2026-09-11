# 08 — Evaluation

> Goal: know whether your memory system is good, know when it regresses, and be able to read a
> benchmark claim without being fooled by it.
>
> Evaluation is the difference between a memory system that improves and one that drifts. Build the
> harness before you build the third feature.
>
> Every number in this chapter that is not a citation was produced by running the code in it. The
> simulations are labelled as simulations.

---

## 8.0 In plain words

### The driving test

Think about how you would certify a driver. You would not watch someone drive around a car park for
five minutes and issue a licence. You would test specific abilities — hill start, parallel park,
emergency stop, reading a sign at distance — because a single "did they crash?" score tells you
nothing about *which* skill is missing, and because the easy manoeuvres would dominate the average.

Memory evaluation is the same, and it has one extra wrinkle: **the examiner is also a language
model**, which is to say the examiner is sometimes wrong.

### The naive version

Here is the memory evaluation nearly every team starts with, and it is completely reasonable as a
first move:

```python
# the 20-line eval
cases = load_json("eval_cases.json")          # 20 questions somebody wrote on a Tuesday
correct = 0
for c in cases:
    ingest_history(c["history"])
    answer = agent(c["question"])
    if judge(f"Is '{answer}' the same as '{c['gold']}'?") == "yes":
        correct += 1
print(f"accuracy: {correct}/{len(cases)}")    # "18/20, ship it"
```

### The arithmetic that kills it

**1. Twenty questions is a coin toss dressed as a measurement.** Bootstrap the 95% confidence
interval for an observed accuracy of ~0.80 at several eval-set sizes:

```
  n=50    observed 0.820  95% CI [0.700,0.920]  width 22.0pp
  n=100   observed 0.800  95% CI [0.720,0.880]  width 16.0pp
  n=200   observed 0.795  95% CI [0.735,0.850]  width 11.5pp
  n=500   observed 0.786  95% CI [0.750,0.822]  width  7.2pp
  n=1000  observed 0.788  95% CI [0.763,0.812]  width  4.9pp
  n=2000  observed 0.800  95% CI [0.782,0.817]  width  3.4pp
```

At n=100 your error bar is **±8 percentage points**. Every "we improved memory by 4%" claim built on
a hundred examples is indistinguishable from noise. (And yet — §8.8 shows how to detect a 3pp change
with 500 examples, because there is a right way to do this.)

**2. "Recall@5" is five different numbers.** Run one retrieval over one fixture of five queries, and
compute every metric that people call recall@5 in conversation:

```
=== same run, five numbers people all call 'recall@5' ===
  recall_macro     0.750      mean of per-query recall           <- usually what you want
  recall_micro     0.778      pooled hits / pooled relevant
  precision_at_k   0.280      hits / (k x queries)
  hit_rate_at_k    0.800      fraction of queries with >=1 hit
  mrr              0.700
  ndcg             0.704
```

**0.28 and 0.80 describe the same retrieval run.** If your dashboard says "recall@5 = 0.8" and the
vendor's paper says 0.28, you may be looking at identical systems. Nobody is lying; nobody defined
the metric.

**3. Your examiner is noisy, and it shrinks whatever you are trying to measure.** Simulate a judge
that agrees with a human grader a fraction `q` of the time, on a true 5.0pp quality gap:

```
  judge agreement 1.00 -> measured gap +5.01pp   (theory: (2q-1)x5.0 = 5.00pp)
  judge agreement 0.98 -> measured gap +4.85pp   (theory: (2q-1)x5.0 = 4.80pp)
  judge agreement 0.95 -> measured gap +4.51pp   (theory: (2q-1)x5.0 = 4.50pp)
  judge agreement 0.90 -> measured gap +3.97pp   (theory: (2q-1)x5.0 = 4.00pp)
  judge agreement 0.80 -> measured gap +2.98pp   (theory: (2q-1)x5.0 = 3.00pp)
```

Symmetric judge error **attenuates** every measured difference by `(2q − 1)`. A judge at 80%
agreement reports 3.0pp for a real 5.0pp win — it will not flip your sign, but it will make real
improvements look marginal and it wrecks your statistical power. This is not a reason to abandon
LLM judges. It is a reason to measure your judge before you trust your results.

**4. The exam sets the system.** If your 20 questions are all "what did the user say about X", you
will build a system that is excellent at simple recall and humiliating at "I moved last month" — and
you will find out from users, not from CI.

### What fixes what

| Failure of the naive eval | Fixed by |
|---|---|
| "18/20, ship it" | §8.8 confidence intervals, paired testing, minimum set sizes |
| Five numbers called recall@5 | §8.4 one definition, written down, in code |
| Noisy judge silently shrinking results | §8.7 judge calibration and the attenuation factor |
| All-recall eval set | §8.6 ability categories with mandated weights |
| Great in CI, bad in production | §8.5 four layers, §8.9 online metrics and A/B traps |
| Vendor benchmark claims | §8.2 what each benchmark measures, §8.3 what none of them measure |
| "We can't tell if this PR made it worse" | §8.10 CI gates that are actually enforceable |
| Architecture chosen on accuracy alone | §8.11 the six-dimension comparison |

**The one-sentence takeaway:** pick one metric definition, build an eval set weighted by the
abilities that break in production (updates and abstention, not recall), run both arms on the *same*
items and use a paired test, calibrate your judge, and always report accuracy with tokens and
latency.

---

## 8.1 The three-number rule

**Never report memory quality as a single number.** Always report a triple:

```
(accuracy, tokens per query, latency p95)
```

A system at 95% accuracy consuming 40k tokens per query is, for almost every production deployment,
worse than one at 89% at 5k tokens — chapter 07.8 priced that difference at roughly **$144,000/mo
versus $38,400/mo at 1M MAU**, and the bigger context is also closer to the context-rot cliff
(ch 02). The published numbers worth anything report cost alongside accuracy; the ones that do not
are marketing.

The corollary for reading vendor claims: **pair every number with its partner.** Pair accuracy with
token cost. Pair a single-session score with a multi-session one. Pair a long-context probe with an
actual memory eval. A number without its pair is not evidence.

Mem0's paper is a good model here precisely because it reports the whole triple: 26% relative
improvement on the LLM-as-a-judge metric over the OpenAI memory baseline, **and** 91% lower p95
latency, **and** >90% token cost saving versus full-context
([arXiv:2504.19413](https://arxiv.org/abs/2504.19413)). You can disagree with the setup; you cannot
accuse it of hiding the cost axis.

---

## 8.2 The public benchmarks

Know these four: what they test, what their numbers actually mean, and where each falls short.

### LoCoMo — *Evaluating Very Long-Term Conversational Memory of LLM Agents*

Multi-session dialogues built by a machine–human pipeline: LLM personas grounded on temporal event
graphs generate conversations, which human annotators then verify and edit for long-range
consistency. The released dataset is **"very long-term conversations, each encompassing 300 turns and
9K tokens on avg., over up to 35 sessions"**, with QA, event summarisation, and multi-modal dialogue
tasks; the paper's headline finding is that long-context LLMs and RAG both "still substantially lag
behind human performance" ([arXiv:2402.17753](https://arxiv.org/abs/2402.17753)).

*Use it for:* a reproducible baseline. It is the most widely reported memory number, so it is your
only cheap route to external comparability.

*Limitations, and they are serious:*
- **9K tokens on average fits in every current context window.** A benchmark whose average instance
  can be pasted wholesale into the prompt cannot, by construction, discriminate between memory
  architectures on the *scale* axis. It measures retrieval and reasoning quality, not memory at
  scale. This is why full-context is a competitive baseline in every LoCoMo table you will read.
- It does not isolate **knowledge updates** — the supersession behaviour from chapter 04 that
  dominates real complaints.

Treat LoCoMo as a floor, not a bar.

### LongMemEval

500 curated questions embedded in freely scalable chat histories, designed around **five core
long-term memory abilities: information extraction, multi-session reasoning, temporal reasoning,
knowledge updates, and abstention.** The headline result is that "commercial chat assistants and
long-context LLMs show a 30% accuracy drop on memorizing information across sustained interactions"
([arXiv:2410.10813](https://arxiv.org/abs/2410.10813)). It ships in two sizes: `_S` (~115k-token
histories) and `_M` (hundreds of sessions).

> Draft correction, since this course is meant to be checkable: an earlier version of this chapter
> said "six ability categories". The paper's abstract names **five** core abilities; the released
> dataset then splits question *types* more finely (single-session-user, single-session-assistant,
> single-session-preference, multi-session, temporal-reasoning, knowledge-update, plus abstention
> variants of each). Abilities and question types are different taxonomies and people conflate them
> constantly, including in comparison tables.

Two design choices make it the more useful benchmark:

1. **Knowledge-update questions** directly test "I moved" handling — the single most
   production-relevant category in any public benchmark.
2. **Abstention questions** ask about events that never happened, and score the system on correctly
   saying "you never told me that." **This is enormously important and almost universally ignored.**
   A memory system that confidently invents a plausible memory is worse than one with no memory,
   because the user cannot tell which mode it is in.

*Limitations:* still chat-shaped. No tool-using agents, no procedural memory (ch 06.1), no
trajectory memory (ch 06.5).

### BEAM — *Beyond a Million Tokens*

Built for the scale problem LoCoMo cannot reach: **100 coherent, multi-domain conversations at
128K / 500K / 1M / 10M tokens, with 2,000 human-validated probing questions across 10 memory
abilities** — the LongMemEval set plus **contradiction resolution, event ordering, and instruction
following** ([arXiv:2510.27246](https://arxiv.org/abs/2510.27246), ICLR 2026;
[code](https://github.com/mohammadtavakoli78/BEAM)). The companion method, LIGHT, reports average
gains of **3.50%–12.69%** over strong baselines — note the modest size of those gains, which tells
you how far from saturated this benchmark is.

*Use it for:* stress-testing an architecture intended to hold years of history. Nobody saturates it,
which is exactly what you want from a benchmark you are using to choose infrastructure.

### DMR (Deep Memory Retrieval)

Established by the MemGPT team as their primary metric and still cited in comparisons — Zep reports
**94.8% vs MemGPT's 93.4%** on it ([arXiv:2501.13956](https://arxiv.org/abs/2501.13956)). Note what
that comparison is worth: a 1.4pp gap. Per §8.8, on a few hundred items that is inside the noise
band unless the evaluation was paired. Zep's more interesting claim is on LongMemEval, where it
reports accuracy improvements up to 18.5% alongside **90% lower response latency** — again, the
triple.

### The moving frontier

The space is expanding into exactly the gaps above: memory consistency and hallucination, project
-oriented multi-week interaction, long-horizon embodied agents, and framework-native suites from the
tool vendors. Expect the frontier to keep moving toward **agentic, tool-using, multi-week** settings,
because that is where all four benchmarks above are weakest. When you read a new one, ask the four
questions in §8.3 before you read its leaderboard.

---

## 8.3 Why public benchmarks are not enough

Read reported scores with these five caveats. Published numbers on the *same* benchmark vary
enormously across sources — different actor models, retrieval configurations, and judging prompts —
so cross-source comparisons are routinely apples-to-oranges.

1. **Different actor models.** A memory system evaluated with a frontier model looks better than the
   same system with a small one. The number measures the *pair*, not the memory layer. Any table
   that compares "Mem0 vs Zep vs Letta" without fixing the actor is measuring three different
   experiments.
2. **Judge variance.** Most are LLM-judged, so §8.7's attenuation applies to every published gap you
   read. Always report the judge model and prompt; treat papers that do not as unreplicable.
3. **They do not measure cost.** See §8.1.
4. **They do not measure the write path in isolation.** A system can score well by storing everything
   verbatim and retrieving hard — excellent on the benchmark, unaffordable in production (ch 07.8),
   and a privacy liability (ch 09).
5. **They do not test what production actually breaks on:** poisoning resistance, deletion
   correctness, cross-tenant isolation (ch 07.3), staleness, cold start, or user-perceived
   creepiness.

**Conclusion:** use public benchmarks for architecture selection and sanity-checking. Use your own
harness for everything else.

---

## 8.4 Define your metric once, in code

Before any harness, settle the definition argument permanently. Here is the whole thing, runnable:

```python
import math

def dcg(rels): return sum(r / math.log2(i + 2) for i, r in enumerate(rels))

def retrieval_metrics(cases, k=5):
    """cases: [{'gold': set[str], 'ranked': list[str]}]"""
    per_q_recall, rr, ndcg = [], [], []
    hits, gold_total = 0, 0
    for c in cases:
        top = c["ranked"][:k]
        h = len(set(top) & c["gold"])
        hits += h; gold_total += len(c["gold"])
        per_q_recall.append(h / len(c["gold"]))
        rank = next((i + 1 for i, d in enumerate(top) if d in c["gold"]), None)
        rr.append(1 / rank if rank else 0.0)
        rels = [1 if d in c["gold"] else 0 for d in top]
        ideal = sorted(rels, reverse=True)
        ndcg.append(dcg(rels) / dcg(ideal) if any(ideal) else 0.0)
    n = len(cases)
    return {
        "recall_macro":   sum(per_q_recall) / n,     # mean of per-query recall
        "recall_micro":   hits / gold_total,         # pooled
        "precision_at_k": hits / (k * n),
        "hit_rate_at_k":  sum(1 for c in cases if set(c["ranked"][:k]) & c["gold"]) / n,
        "mrr":            sum(rr) / n,
        "ndcg":           sum(ndcg) / n,
    }
```

On the five-query fixture from §8.0 this prints:

```
  recall_macro     0.750
  recall_micro     0.778
  precision_at_k   0.280
  hit_rate_at_k    0.800
  mrr              0.700
  ndcg             0.704
```

Which one should you actually gate on? Depends on what the retrieval feeds:

| If your reader… | Gate on | Why |
|---|---|---|
| needs **one** fact to answer (most memory queries) | `hit_rate_at_k` or `MRR` | you only need the fact present, and early |
| synthesises across **several** facts (ch 05 multi-hop) | `recall_macro` | missing one of four golds is a real loss |
| has a tight token budget (ch 07.8) | `precision_at_k` | every non-relevant slot is money |
| is graded by position (rerank tuning, ch 03) | `nDCG@k` | rewards getting the good one to rank 1 |

**Use macro, not micro, for anything user-facing.** Micro-averaging weights queries by how many gold
items they have, so your metric is dominated by a handful of broad queries while the 200 one-fact
queries that make up your traffic barely move it. Macro gives every user question one vote — which
is the thing you actually care about.

And write the definition into the metric *name* in your dashboards: `recall_macro@5`, not `recall`.
Six months later, nobody remembers.

---

## 8.5 Building your own eval harness

Four layers. Build them in this order; each is cheap and catches different bugs.

### Layer 1 — Unit tests on the write path

The scenario suite from chapter 04.10. Fast, deterministic, runs in CI on every change. This is your
regression gate and it catches the failures that retrieval metrics cannot see.

```python
def test_supersession():
    mem = fresh_store()
    mem.ingest("I live in Pune", ts="2025-01-01")
    mem.ingest("I moved to Mumbai last month", ts="2026-03-01")
    assert mem.query_current(subject="user", predicate="lives_in").object == "Mumbai"
    assert mem.query_at(subject="user", predicate="lives_in",
                        t="2025-06-01").object == "Pune"          # ch 05.2 query 2

def test_belief_history_is_preserved():
    """ch 05.2 query 3: what did we believe then, not what is true now."""
    ...
    assert mem.query_as_believed(at="2026-02-01", predicate="lives_in").object == "Pune"

def test_future_fact_is_not_current():
    mem.ingest("I'm starting at Beta Corp next month", ts="2026-03-15")
    assert mem.query_current(predicate="works_at").object == "Acme"   # not yet
```

These are *assertions about behaviour*, and each maps to a named production failure from the ch 04
and ch 05 failure tables. Do not write tests that assert your extractor emitted a particular JSON
shape; assert what a consumer observes.

### Layer 2 — Retrieval metrics on labelled data

§8.4's function over a fixed, version-controlled set of 200–500 labelled queries. No LLM at eval
time, so it is fast, free, and **deterministic** — which has a statistical consequence people get
wrong, covered in §8.10.

### Layer 3 — End-to-end QA on your own conversations

The real test. Build it from your own traffic:

```python
from dataclasses import dataclass, field

@dataclass
class MemoryEvalCase:
    case_id: str
    setup_sessions: list["Session"]   # history that must be ingested first
    query: str                        # the question, in a NEW session
    expected: str                     # gold answer, or "NOT_STATED" for abstention
    category: str                     # recall|update|temporal|multi_hop|abstention|preference|deletion
    must_not_contain: list[str] = field(default_factory=list)
    days_between: int = 0             # simulated gap between setup and query
```

```python
def run_case(system, case, judge) -> dict:
    system.reset_tenant(case.case_id)
    for s in case.setup_sessions:
        system.ingest_session(s)
    system.flush_write_path()                     # do not measure a race (ch 07.7 freshness)

    t0 = time.monotonic()
    out = system.answer(case.query, new_session=True)
    latency_ms = (time.monotonic() - t0) * 1000

    grade = judge(case.query, case.expected, out.text, category=case.category)
    leaked = [s for s in case.must_not_contain if s.lower() in out.text.lower()]
    return {
        "case_id": case.case_id, "category": case.category,
        "correct": grade == "CORRECT" and not leaked,
        "leaked": leaked,
        "injected_tokens": out.memory_tokens,      # the second of the three numbers
        "latency_ms": latency_ms,                  # the third
        "memories_injected": len(out.memory_ids),
        "memories_cited": len(out.cited_ids),      # ch 07.10 usage rate
    }
```

Four properties are non-negotiable:

- **The query runs in a NEW session**, so the context window cannot answer it. Otherwise you are
  testing your context window, not your memory. This is the single most common harness bug and it
  makes everything look great.
- **`flush_write_path()` before querying.** Your write path is async (ch 07.7). If the harness does
  not wait for indexing, your eval measures a race condition and gets a different score on a loaded
  CI machine — which people then blame on the model.
- **Abstention cases, ~15% of the set.** Ask about things never mentioned; the correct answer is an
  admission of not knowing. If you build only positive cases you will optimise your system into a
  confident confabulator and find out from users.
- **`must_not_contain` assertions.** After "forget my old address", assert the old address appears
  in no answer. This is how you test deletion *behaviourally* rather than at the storage layer — and
  it is chapter 09's erasure cascade with an observable pass/fail.

Note that `run_case` returns all three of the §8.1 numbers per case. Make that structural: a harness
that can only report accuracy will only ever be used to report accuracy.

### Layer 4 — Online metrics

The ground truth. Offline evals correlate imperfectly with user experience; these do not.

- **User correction rate** — how often users say "no, that's wrong". Best single proxy for quality.
- **Memory usage/citation rate** — chapter 07.10. Low usage = wasted tokens.
- **Explicit feedback** on memory items, if your UI exposes them (it should).
- **Deletion rate**, segmented by category — separates "creepy" from "wrong".
- **Session-over-session task success** — the actual business metric.

---

## 8.6 Test design: the categories that matter

Structure your eval set by *ability*, not by topic, so failures are diagnostic. Minimum viable
distribution:

| Category | % | What it tests | Typical root cause when it fails |
|---|---|---|---|
| Simple recall | 20% | Single fact from one past session | Retrieval miss (ch 03) |
| Multi-session synthesis | 15% | Combine facts across sessions | Fusion / k too small (ch 03.7) |
| Knowledge update | 20% | Superseded facts | Write path has no supersession (ch 04.5, 05.7) |
| Temporal reasoning | 15% | "What was true when", durations, ordering | No bi-temporal model (ch 05.2) |
| Abstention | 15% | Never-stated facts | Overconfident generation |
| Preference application | 10% | Implicit use of stored preferences | Gate / query construction (ch 04.3) |
| Negative/deletion | 5% | Deleted facts must not surface | Incomplete erasure cascade (ch 09) |

**Weight knowledge update and abstention heavily.** They are where real systems fail and where public
benchmarks under-test — LoCoMo especially. If your eval set is 80% simple recall, you will ship a
system that is great at simple recall and embarrassing at everything users notice.

One discipline that pays for itself: **every production incident becomes an eval case.** A user
complaint is a free, perfectly-targeted test case, and a suite grown this way converges on your
actual failure distribution rather than on what was easy to write on a Tuesday.

---

## 8.7 The judge, and how much it is lying to you

Most end-to-end grading is LLM-based. Getting the judge right matters more than people assume,
because §8.0's arithmetic shows judge error **attenuates every effect you are trying to measure**.

```python
JUDGE = """Compare the system's answer to the gold answer.

Question: {question}
Gold answer: {gold}
System answer: {answer}

Grade CORRECT if the system answer conveys the gold information, even if worded differently
or with extra context. Grade INCORRECT if it contradicts the gold, omits the key fact,
or is unhelpfully vague.

For abstention questions (gold = "NOT_STATED"), grade CORRECT only if the system explicitly
indicates it does not have that information. A plausible invented answer is INCORRECT.
A hedged answer that still asserts the invented fact is INCORRECT.

Return JSON: {{"grade": "CORRECT|INCORRECT", "reason": "one sentence"}}
"""
```

### The attenuation result

Simulating a judge with symmetric error rate `1−q` over 2,000 items, 200 repeats, against a system
pair with a true 5.0pp quality gap:

```
  judge agreement 1.00 -> measured gap +5.01pp   (theory: (2q-1)x5.0 = 5.00pp)
  judge agreement 0.98 -> measured gap +4.85pp   (theory: (2q-1)x5.0 = 4.80pp)
  judge agreement 0.95 -> measured gap +4.51pp   (theory: (2q-1)x5.0 = 4.50pp)
  judge agreement 0.90 -> measured gap +3.97pp   (theory: (2q-1)x5.0 = 4.00pp)
  judge agreement 0.80 -> measured gap +2.98pp   (theory: (2q-1)x5.0 = 3.00pp)
```

The simulation matches the closed form exactly: **measured gap = (2q − 1) × true gap.** Three
consequences:

1. **Judge quality has a multiplicative effect on your ability to detect improvements.** Going from
   a 90% judge to a 98% judge is worth more than doubling your eval set — the 98% judge recovers
   4.85pp of a 5pp effect, and it costs you one afternoon of rubric work.
2. **Attenuation is toward zero, not toward a random direction.** A noisy judge makes you
   *conservative*: you will discard real wins, not ship fake ones. That is the good news, and it is
   why LLM judges are usable at all.
3. **It compounds with §8.8's power problem.** Judge noise shrinks the effect *and* inflates the
   variance, so the required sample size grows faster than the attenuation alone suggests.

### Judge discipline

- **Calibrate against humans.** Hand-grade 100 cases and measure agreement. Below ~90%, fix the
  rubric — usually by adding explicit rules for the specific cases you disagreed on. Report
  agreement alongside your results the way you report the judge model.
- **Use a different model family than the actor** where possible, to reduce self-preference bias.
- **Pin the judge version** and record it with every result. A judge upgrade invalidates your
  history; when you change judges, re-run the baseline before you compare anything.
- **Randomise answer position** in pairwise comparisons; position bias is real and large.
- **Log reasons and read the INCORRECTs weekly.** Judge failures cluster, and the clusters are the
  rubric's missing rules.
- **Grade abstention separately.** Judges are systematically bad at it — "I don't have that
  information, but typically people…" is a failure that reads like a success.

---

## 8.8 Statistics: how many cases, and which test

This section is the difference between an eval that steers the project and one that generates
arguments.

### Sample size

From §8.0: at n=100 the 95% CI on an 80% accuracy is ±8pp. Rules of thumb that follow:

| Question you are asking | Minimum n |
|---|---|
| "Roughly how good are we?" | 100 (±8pp — say "about 80%", never "80.4%") |
| "Did this PR change anything?" | 300–500 **paired** (below) |
| "Is A better than B by a few points?" | 500+ paired, or 2,000 unpaired |
| "Is this per-category score real?" | 100 *per category*, not 100 total |

That last row is the one that bites. A 500-case suite split across seven categories gives ~70 cases
per category, so your per-category numbers — the diagnostic ones you actually act on — carry ±10pp
error bars. Either size the suite for the categories or stop reporting per-category deltas.

### Paired beats unpaired by an order of magnitude

Both arms should answer **the same questions**, and then you test the *disagreements* (McNemar's
test) rather than the two averages. Simulated 300 experiments per row, true improvement +3.0pp
(the new system fixes 23% of the baseline's errors and breaks 2% of its successes):

```
=== power to detect a TRUE +3.0pp improvement (300 simulated experiments each) ===
   n     mean observed delta   unpaired z-test    paired McNemar
  100        +2.91pp              0.0%             9.7%
  200        +3.07pp              0.0%            31.0%
  500        +3.08pp              6.0%            75.3%
  1000       +3.06pp             31.7%            95.0%
  2000       +3.01pp             85.0%           100.0%
```

At **n=500, the paired test finds the improvement 75% of the time; the unpaired test finds it 6% of
the time.** To get the unpaired test to 85% power you need 2,000 cases — a 4× more expensive
evaluation to learn the same fact. One concrete experiment from that run:

```
  one n=1000 experiment: base-only-correct b=18, new-only-correct c=46, McNemar p=0.00062
  same data, unpaired: 0.805 vs 0.833
```

The unpaired view is "0.805 vs 0.833, eh, maybe". The paired view is "46 fixes against 18
regressions, p = 0.0006, ship it". Same data.

```python
def mcnemar_p(base: list[int], new: list[int]) -> tuple[float, int, int]:
    """Exact two-sided McNemar over paired per-case correctness."""
    b = sum(1 for x, y in zip(base, new) if x == 1 and y == 0)   # only base correct
    c = sum(1 for x, y in zip(base, new) if x == 0 and y == 1)   # only new correct
    n = b + c
    if n == 0:
        return 1.0, b, c
    p = 2 * sum(math.comb(n, i) * 0.5 ** n for i in range(min(b, c) + 1))
    return min(p, 1.0), b, c
```

Two operational implications:

- **Store per-case results, not aggregates.** You cannot run a paired test on two accuracy numbers.
  If your harness logs `accuracy: 0.83` and throws the rows away, you have destroyed 90% of your
  statistical power for the sake of a smaller file.
- **Look at `b`, the regressions, not just the net.** A change with b=18 / c=46 and one with b=2 /
  c=30 have similar deltas and very different risk profiles. The first one broke eighteen things that
  used to work — go read them; they cluster, and the cluster is usually a category you did not intend
  to touch.

### Do not peek

Running the eval after every prompt tweak and stopping when it looks good is the oldest way to fool
yourself. Fix the eval set, fix the number of runs, decide the threshold in advance, and keep a
**held-out set** that you run monthly at most. Your development set will be overfit within a
quarter — that is not a moral failing, it is what happens when you optimise against a fixed target,
and the held-out set is the only thing that tells you it happened.

---

## 8.9 A/B testing memory in production

Memory A/B tests have four traps that make naive experiments misleading.

**Trap 1: Memory effects are delayed.** A user's first session has no memory to use. Benefits appear
in sessions 3–20. A two-week experiment on a weekly-active product is measuring the *cost* of memory
with none of its benefit.
*Fix:* run long, and analyse by **session index**, not by user-day.

**Trap 2: Novelty and creepiness both distort early data.** Users react to being remembered — some
delight, some discomfort. Both fade.
*Fix:* discard the first week; look at week 3+ for steady state.

**Trap 3: Averages hide harm.** Memory can help 90% of users slightly and harm 10% badly (a wrong or
embarrassing memory).
*Fix:* look at the tail — p5 of satisfaction, complaint rate, deletion rate — not just the mean. A
memory feature that improves the mean and doubles the complaint rate is not a win.

**Trap 4: The treatment leaks into the control.** Memory written during the experiment persists. If
you roll a user back to control, their memory store still exists, and if you later re-randomise,
your "control" users have memories. Memory experiments are **not cleanly reversible**, which makes
them closer to a migration than to a button-colour test.
*Fix:* randomise once, at the user level, hold the assignment for the whole experiment, and decide
before you start what happens to the control group's accumulated data.

Metrics I would gate a launch on:

```
PRIMARY:   task success rate (sessions 3+)
GUARDRAIL: user correction rate           must not increase
GUARDRAIL: memory deletion rate           must not increase
GUARDRAIL: p95 latency                    must not increase > 50ms   (ch 07.4 budget)
GUARDRAIL: complaint / thumbs-down rate   must not increase
GUARDRAIL: cross-tenant incidents         must be zero               (ch 07.3)
SECONDARY: tokens per turn, session length, return rate
```

---

## 8.10 Regression gates in CI

What actually runs on every PR:

```yaml
memory_ci:
  fast (every PR, < 3 min):
    - write-path scenario suite              # ch 04.10 — must be 100%
    - cross-tenant isolation + RLS config    # ch 07.3 — must be 100%
    - deletion cascade test                  # ch 09   — must be 100%
    - retrieval recall_macro@20, fixed 300-query set   # must not drop at all (see below)
    - injected tokens p50 on the same set    # must not increase > 10%

  nightly (< 45 min):
    - end-to-end eval, 500 paired cases, all categories, McNemar vs main
    - per-category breakdown (report, do not gate — n too small per category)
    - cost per query, latency p50/p95/p99
    - LoCoMo subset (200 q) for external comparability

  weekly:
    - full LongMemEval_S
    - poisoning red-team suite (ch 09)
    - counterfactual memory-usage ablation (ch 07.10)
    - held-out eval set (never used for development)
```

**The must-be-100% items** are the ones where a single failure is a bug, not a metric regression:
isolation, deletion, and the write-path scenarios. Never let those become "we're at 97% and trending
up."

### The subtlety in the Layer-2 gate

"Recall must not drop at all" looks absurd next to §8.8's ±11.5pp error bars at n=200. It is not,
and the reason is worth understanding because it decides how you write every gate:

- **Layer 2 is deterministic and paired against a fixed set.** The same 300 queries, the same
  embedding model, the same index. There is no sampling noise *relative to that set*: if
  `recall_macro@20` moves at all, the system changed. So gate it tightly — even `> 0.5%` is
  meaningful as a *change detector*.
- **What you may not do is call that number your accuracy.** The 300 queries are a sample of the
  query population, so the gate says "this PR changed retrieval behaviour", not "retrieval is 78.3%
  good". Two different claims, two different error bars.
- **Layer 3 is stochastic** (sampling + generation + judge), so it gets a statistical gate
  (McNemar p < 0.05 on regressions) rather than a threshold, and it runs nightly rather than
  per-PR because it costs real money.

Get this distinction wrong in either direction and CI becomes useless: loose gates on the
deterministic layer let real regressions through, and tight thresholds on the stochastic layer
produce a flaky pipeline that everyone learns to re-run until green.

### What an eval run costs

Budget it, or it will not run. Using chapter 07.8's rates (`gpt-4.1-mini`, $0.40/1M in, $1.60/1M
out): 500 end-to-end cases × (~8k tokens of setup ingestion + ~3k tokens of query context + ~300
output) ≈ 5.5M input + 150k output ≈ **$2.44 per nightly run**, plus the judge at ~500 × 700 tokens
≈ $0.14. The full weekly LongMemEval_S is roughly an order of magnitude more. In other words: the
harness is not what costs money. There is no budget excuse for not running it.

---

## 8.11 Comparing architectures fairly

If you are choosing between memory architectures — build vs. Mem0 vs. Zep vs. LangGraph store (ch 10)
— here is the harness that makes the comparison honest:

1. **Fix the actor model and the judge.** Same for all arms, pinned versions. Report both.
2. **Fix the eval set** — your own data, plus one public benchmark for external comparability.
3. **Pair the cases across arms** and use McNemar (§8.8). Every arm answers the same questions.
4. **Measure the triple** for each arm: accuracy, tokens/query, p95 latency.
5. **Measure the write cost too** — tokens and dollars per session ingested. Graph-based systems can
   be an order of magnitude more expensive to write (ch 05.8), which never appears in accuracy
   tables and always appears in your bill.
6. **Run the operational tests:** deletion correctness, tenant isolation, restore from backup,
   behaviour on a 100k-memory tenant (ch 07.3's whale), and freshness lag under load.
7. **Score the non-functionals:** licence, self-host support, data export, governance model,
   dependency risk. Vendors in this space have changed licences, retired community editions, and
   moved features behind hosted platforms with little notice — a memory layer is a deep dependency
   and migrating it is a quarter of work, so weight this heavily.

Write the result up as a table with all seven dimensions. A decision made on accuracy alone will be
revisited within a year.

---

## 8.12 Failure modes of evaluation itself

| Symptom | Root cause | Fix |
|---|---|---|
| "We improved 4%" that nobody can reproduce | n≈100, unpaired | §8.8 paired McNemar, 500+ cases |
| Your recall doesn't match the vendor's | different metric under the same name | §8.4 one definition, named in the metric |
| Real improvements keep looking marginal | judge agreement ~0.85, effects attenuated | §8.7 calibrate the rubric to >0.95 |
| Scores great offline, users complain | eval queries run in the same session | §8.5 `new_session=True` |
| Eval score varies run to run on identical code | harness races the async write path | §8.5 `flush_write_path()` |
| Model confabulates memories, eval is 92% | no abstention cases | §8.6 15% abstention, graded strictly |
| Category deltas flip sign every week | ~70 cases per category | §8.8 size per category or stop reporting |
| Dev set score rises, production flat | overfitting to the dev set | §8.8 held-out set, monthly |
| CI is flaky, everyone re-runs until green | statistical gate written as a hard threshold | §8.10 deterministic vs stochastic layers |
| Deletion "works" but old data resurfaces | tested at the storage layer only | §8.5 `must_not_contain`, ch 09 cascade |
| Benchmark win does not survive contact with prod | benchmark avg fits in the context window | §8.2 LoCoMo's 9K-token average; use BEAM |

---

## 8.13 Exercises

1. **Reproduce the metric ambiguity.** Take 20 real queries from your system, label the gold set, and
   compute all six numbers from §8.4. Acceptance: you can state which one your dashboard has been
   showing and whether it is the one you want. (In my experience the answer is "precision@k,
   mislabelled".)
2. **Build the Layer-3 harness** with 100 cases from your own history, 15 of them abstention.
   Acceptance: the harness emits accuracy, injected tokens, and latency per case, and your abstention
   number is the worst of the seven categories. That is normal; it is also the most actionable
   finding you will get this month.
3. **Calibrate a judge.** Hand-grade 100 cases, measure agreement, then iterate the rubric.
   Acceptance: agreement > 0.95, and you can state — using `(2q−1)` — how much a real 5pp improvement
   would be attenuated before and after your fix.
4. **Prove the paired-test result on your own data.** Run two variants of your system (e.g. k=8 vs
   k=4) on the same 300 cases. Acceptance: a b/c contingency count, a McNemar p-value, and the
   unpaired z-test for comparison. State which one you would have believed.
5. **Find your regressions, not your delta.** For the same experiment, list the cases the new system
   broke. Acceptance: you can name a *category* for at least half of them. If they look random, your
   eval set is too small or your categories are wrong.
6. **Run a LoCoMo subset** and compare to published numbers. Acceptance: a written list of every
   reason your number is not comparable to theirs — actor model, judge, retrieval config, subset
   choice, prompt. The list will be long; that is the lesson of §8.3.
7. **Cost your harness.** Compute the per-run dollar cost of your nightly eval at your own rates.
   Acceptance: a number, and a decision about how often it runs that is based on that number rather
   than on vibes.
8. **Design the A/B test** for adding memory to an existing product. Write metric definitions,
   guardrails, the analysis plan by session index, the handling of control-group memory (trap 4), and
   the stopping rule — all before looking at any data. Acceptance: a one-page plan a sceptical
   colleague signs off on.

Next: `09-security-privacy-governance.md` — the attack that starts with a web page and ends with your
agent acting on a fact nobody ever told it, and the erasure cascade that most systems get wrong.
