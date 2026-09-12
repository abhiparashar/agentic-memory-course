# 11 — Small Projects

> Goal: six projects that make you *feel* every failure mode in chapters 02–09, each ending in a
> number rather than a demo. Every measurement in this chapter was executed before it was written;
> the outputs below are real, including the five bugs the verification run found.

---

## 11.0 In plain words

### The logbook

Pilots do not learn engine failure on a passenger flight. They log simulator hours, and the whole
point of the simulator is that the first time the left engine quits, it quits at 3,000 feet in a
machine bolted to a warehouse floor. The hours are boring. They are also the difference between
"I have read about this" and "my hands know what to do".

These six projects are simulator hours. Each one stages exactly one emergency from an earlier
chapter — the fact that falls out of the window, the retriever that penalises the query term, the
gate that drops a negation, the clock that cannot explain itself — in a rig small enough to fit in
one file, with an instrument panel that tells you when you have crashed.

### The naive version

Here is how nearly everyone "learns" agent memory, in twenty lines:

```python
history, memories = [], []

def chat(user_text):
    history.append(("user", user_text))
    hits = sorted(memories, key=lambda m: cosine(embed(user_text), m["vec"]))[-5:]
    reply = llm(system="Known facts:\n" + "\n".join(m["text"] for m in hits),
                messages=history)
    history.append(("assistant", reply))
    if llm_says_worth_remembering(user_text):
        memories.append({"text": user_text, "vec": embed(user_text)})
    return reply

chat("I'm vegetarian")            # ✓ works
chat("What do I eat?")            # ✓ works
chat("Recommend a restaurant")    # ✓ works — ship it
```

It works. It works on the three examples you typed, and on any example you would think to type,
because you are testing the thing you just wrote with the cases you had in mind while writing it.

### The arithmetic that kills it

Three numbers, all measured in this chapter's verification run rather than asserted:

**1. Your demo has a sample size of three, and three cases cannot see anything.** Bootstrap 95%
confidence intervals for an observed 80% accuracy (20,000 resamples each):

```
n=10    CI [0.50, 1.00]   width 50.0pp
n=30    CI [0.67, 0.93]   width 26.7pp
n=100   CI [0.72, 0.88]   width 16.0pp
n=300   CI [0.75, 0.84]   width  9.0pp
n=1000  CI [0.78, 0.82]   width  4.9pp
```

Worse, at demo scale the *ordering* inverts. Simulating a true 60% system against a true 80% system:
at **n=10 the better system measures worse-or-equal 23% of the time**; at n=30, 5%; at n=100, 0%.
A ten-case acceptance bar is a coin you are reading as a thermometer (§8.8 has the theory; this
chapter's §11.7 has the code).

**2. Your demo never runs 500 turns, which is where the cost lives.** A real 500-exchange session,
counted with `tiktoken` (`cl100k_base`), 44 tokens per exchange, 21,981 tokens of transcript:

```
strategy            billed input tokens   $/session @ $0.40/1M   fact recalled at turn 480?
SendAll                       5,500,245                  $2.20   yes
SlidingWindow(20 msgs)          231,871                  $0.09   NO  (fact left at turn 15)
SlidingWindow(20 turns)         444,861                  $0.18   NO  (fact left at turn 25)
Window + summary-from-log       329,299                  $0.13   yes
```

`SendAll` is **23.7× the tokens** of the naive window and **16.7×** the window-plus-summary, for the
same answer; the sliding window is cheapest and *wrong*. Nothing in a three-turn demo can show you
this, and the accounting error hiding in plain sight — "window of 20" meaning messages in one place
and exchanges in another — is a silent **1.92× cost** difference.

**3. Your assertions pass because nothing checks them.** Running the chapter-04 scenario suite
against a from-scratch extractor: the naive version scored **8/12**. Then two audits:

```
assertion keys the runner silently ignores:
   2 supersession   key='current'     in {'match': 'Mumbai', 'current': True}
   2 supersession   key='closed'      in {'match': 'Pune',   'closed':  True}
   8 temporal       key='valid_from'  in {'match': 'Acme',   'valid_from': '2025-03'}
=> claimed 8/12, actually verified: 6/12 fully

mutation score of the fixed 12/12 suite: 2/5
   gate: opt-out disabled           -> SURVIVED
   gate: untrusted fence removed    -> SURVIVED
   gate: secret scan removed        -> SURVIVED
```

Three safety gates could be **deleted entirely** with the suite still reporting 12/12 green. Full
derivation in §11.4; the fix takes four lines and raises the mutation score to 5/5.

### What fixes what

| The thing you cannot feel from reading | Project | Chapter it comes from |
|---|---|---|
| Cost and recall trade off non-obviously across 500 turns | S1 §11.2 | 02 |
| Lexical and dense retrieval fail on *different* queries | S2 §11.3 | 03 |
| BM25 can penalise a document for containing your query term | S2 §11.3 | 03.2 |
| "recall@5" names six different numbers | S2 §11.3 | 08.4 |
| Write-path bugs are invisible without behavioural tests | S3 §11.4 | 04.11 |
| A green suite can be testing nothing | S3 §11.4 | 04.11, 08.12 |
| Flat memory cannot *express* five of six temporal questions | S4 §11.5 | 05.2 |
| Background work wins latency and quality simultaneously | S5 §11.6 | 06.3, 07.4 |
| p99s do not add, and adding them overstates your budget | S5 §11.6 | 07.4 |
| A poisoned memory survives the session that planted it | S6 §11.7 | 09.2 |
| Deleting a row does not delete the fact | S6 §11.7 | 09.6 |

**Ground rules, unchanged from the first draft because they are the right ones:**

- **No memory frameworks.** Postgres, SQLite, an embedding model, an LLM API, standard library. You
  are allowed `numpy` and a tokenizer. Frameworks start in chapter 10 and stay there.
- **Every project ends in a measurement, not a demo.** The deliverable is a table.
- **Always report the triple** — accuracy, tokens, latency (standing rule 5). Two of three is
  marketing.
- **One repo.** S3 imports S2's retriever; S6 evaluates all of them.

Budget: ~33 hours total. S3 is the one that matters most; if you only do one, do S3.

---

## 11.1 The lab: one harness all six projects share

Build this first, in `memlab/`. Every project reports through it, which is what makes the six
projects comparable instead of six unrelated demos. It is about 120 lines and there is no excuse for
not having it.

```python
# memlab/harness.py  — stdlib + numpy + tiktoken only
import time, math, re, statistics
import numpy as np, tiktoken

ENC = tiktoken.get_encoding("cl100k_base")
def ntok(s: str) -> int: return len(ENC.encode(s))

# ---- 1. the triple ------------------------------------------------------------
class Run:
    """Accumulates the three numbers for one configuration. Never report fewer."""
    def __init__(self, name): self.name, self.rows = name, []
    def record(self, correct: bool, in_tok: int, out_tok: int, latency_ms: float):
        self.rows.append((bool(correct), in_tok, out_tok, latency_ms))
    def report(self, price_in=0.40/1e6, price_out=1.60/1e6) -> dict:
        c, i, o, l = zip(*self.rows)
        return {"name": self.name, "n": len(c),
                "accuracy": sum(c)/len(c),
                "ci95": boot_ci(sum(c), len(c)),
                "tokens_in_per_q": sum(i)/len(i),
                "p50_ms": float(np.percentile(l, 50)),
                "p95_ms": float(np.percentile(l, 95)),
                "usd_per_1k_q": (sum(i)*price_in + sum(o)*price_out) / len(c) * 1000}

# ---- 2. never report accuracy without an interval -----------------------------
def boot_ci(k, n, iters=20_000, seed=0):
    r = np.random.default_rng(seed)
    obs = np.concatenate([np.ones(k), np.zeros(n - k)])
    m = r.choice(obs, size=(iters, n), replace=True).mean(axis=1)
    return (round(float(np.percentile(m, 2.5)), 3), round(float(np.percentile(m, 97.5)), 3))

# ---- 3. retrieval metrics, defined ONCE (see 08.4) ----------------------------
def recall_at_k(gold, ranked, k):      # denominator = |gold|, uncapped
    gold = set(gold)
    return len(gold & set(ranked[:k])) / len(gold) if gold else float("nan")
def ndcg_at_k(gold, ranked, k):
    gold = set(gold)
    dcg  = sum((1 if d in gold else 0)/math.log2(r+2) for r, d in enumerate(ranked[:k]))
    idcg = sum(1/math.log2(r+2) for r in range(min(len(gold), k)))
    return dcg/idcg if idcg else float("nan")

# ---- 4. did the fix survive contact? -----------------------------------------
def mutation_score(suite_fn, mutants: dict) -> float:
    """mutants: {name: callable that breaks one thing and returns a restore callable}"""
    killed = 0
    for name, apply_mutant in mutants.items():
        restore = apply_mutant()
        try:    killed += bool(suite_fn())      # suite_fn returns list of failures
        finally: restore()
    return killed / len(mutants)

def timed(fn, *a, **kw):
    t0 = time.perf_counter(); out = fn(*a, **kw)
    return out, (time.perf_counter() - t0) * 1000
```

Two design decisions in there are the whole point of the harness.

**`Run.report` cannot return accuracy without a confidence interval.** Make the honest version the
only version available and you stop shipping "we improved from 78% to 81%" on 50 cases. Verified
widths at the top of §11.0: at n=100 that 3pp "improvement" sits inside a 16pp interval.

**`recall_at_k` is defined once.** §8.4 showed six different numbers all called `recall@5`; here is
that spread reproduced on a single run with `|gold| = 7` and top-5 `[8, 1, 0, 2, 4]` (hits: 0, 8):

```
hit@5                              1.000
recall@5 = |gold ∩ top5| / |gold|  0.286
recall@5 capped = / min(|gold|,5)  0.400
precision@5                        0.400
MRR@5                              1.000
nDCG@5 (binary)                    0.509
spread: 0.286 .. 1.000
```

**3.5× between the smallest and largest number describing one identical result.** Any of the six is
defensible; choosing per-experiment is not. Import the function, never re-implement it.

---

## 11.2 S1 — The Goldfish (ch 02) · ~3 hours

**Build:** a chat loop that survives 500 turns inside a fixed token budget.

**Requirements**

- A token counter and a `ContextBudget` with named regions and hard caps. Overflow raises; it does
  not silently truncate (silent truncation is how identifiers disappear).
- Three history strategies behind one interface: `SendAll`, `SlidingWindow(n)`, `WindowPlusSummary`.
- **Summaries regenerate from the durable log, never from the previous summary.** This is the single
  most important line in the project (02.3).
- Per-region token occupancy logged every turn.
- State your unit in the class name: `SlidingWindowMessages(20)` or `SlidingWindowExchanges(20)`.

**Test:** a 500-turn conversation where `my account number is ACC-77213` appears at turn 4 and is
asked for at turn 480.

### Measured, so you know what to expect

Executed with the tokenizer, not estimated. Per-exchange mean 44 tokens; full transcript 21,981
tokens; "fact retained" means the identifier is present in the rendered context at turn 480, which
is an upper bound on answering correctly:

| Strategy | Billed input tokens | Peak context | $/session @ $0.40/1M | Fact retained at 480 |
|---|---|---|---|---|
| `SendAll` | 5,500,245 | 21,966 | $2.20 | yes |
| `SlidingWindow(20 messages)` | 231,871 | 469 | $0.09 | **no** — left at turn 15 |
| `SlidingWindow(20 exchanges)` | 444,861 | 909 | $0.18 | **no** — left at turn 25 |
| `WindowPlusSummary(20, 200)` | 329,299 | 675 | $0.13 | yes |

Rates are `gpt-4.1-mini` input at $0.40/1M
([OpenAI pricing](https://developers.openai.com/api/docs/pricing), retrieved 2026-09-12).

Read the table twice. `WindowPlusSummary` is **16.7× cheaper than `SendAll` and answers the same
question**; it is 1.42× dearer than the naive window and the naive window *cannot answer at all*.
That 1.42× is the price of memory, and it is the cheapest thing you will buy all chapter.

Also: `SendAll` crosses an 8k context limit at **turn 187** — so a "128k context, we don't need
memory" plan is simply a plan to pay 23.7× for the privilege of also hitting context rot (02.1,
[Chroma context-rot report](https://research.trychroma.com/context-rot)).

### The bug the verification run found: your compaction guard is decoration

02.3 recommends a `compaction_guard` that extracts identifiers from the log and asserts they survive
summarisation. Here is the guard everyone writes first:

```python
ID_RE = re.compile(r"\b[A-Z]{2,}-\d{3,}\b")          # ACC-77213 ✓
def compaction_guard(log_text, context_text):
    lost = set(ID_RE.findall(log_text)) - set(ID_RE.findall(context_text))
    assert not lost, f"compaction dropped identifiers: {lost}"
```

Run it on a conversation containing six identifiers in the shapes real users actually type:

```
compaction_guard(ID_RE) says lost: []  ->  PASS
actually missing from context : ['INV/2026/0042', 'acct 77213', 'a3f2c1d9', '#77213', 'user_id=90218']
guard recall on identifier shapes: 1/6
```

**The guard passes while five of six identifiers are gone.** It recognised `ACC-77213` and nothing
else, so it measured its own regex rather than the summariser. The wider pattern finds all six and
correctly reports the five losses:

```python
WIDE = re.compile(r"(?:\b[A-Z]{2,}[-/][A-Za-z0-9/\-]*\d[A-Za-z0-9/\-]*"
                  r"|\b(?:acct|account|ticket|order|user_id|id)[\s:#=]*\d{3,}"
                  r"|#\d{3,}|\b[0-9a-f]{7,40}\b)")
# WIDE says lost: ['INV/2026/0042', 'a3f2c1d9', 'acct 77213', 'ticket #77213', 'user_id=90218']
```

**The lesson, and it recurs in S3 and S6: a guard needs its own recall measured against a labelled
set, or it is a green light wired to nothing.** This is the same failure as 04.2's gate regex missing
`I don't drink` — the detector was never tested as a classifier. Before trusting any guard, write ten
positive examples in shapes you did not have in mind when you wrote the pattern, and assert the guard
finds all ten.

**You are done when**

- [ ] You can state the exact turn at which your window drops the fact (15 for messages, 25 for
      exchanges on this transcript) and your log shows it.
- [ ] `WindowPlusSummary` preserves `ACC-77213` through ≥ 10 summarisation rounds, with the summary
      regenerated from the log each time.
- [ ] Your identifier guard scores ≥ 9/10 recall on a labelled set of identifier shapes, and the
      set is committed to the repo.
- [ ] The triple is reported per strategy: fact-retention rate, tokens/session, p95 turn latency.

**Stretch:** implement summary-of-summary deliberately and plot identifier survival per round. At a
per-round survival of 0.9 — generous — ten rounds retains `0.9^10 = 34.9%`. That arithmetic is why
"regenerate from the log" is a hard rule, not a preference.

---

## 11.3 S2 — Hybrid Retriever (ch 03) · ~6 hours

**Build:** a retrieval stack over conversation history, with real IR metrics.

**Requirements**

- Postgres + pgvector: messages with embeddings and a `tsvector`.
- Three arms: dense (pgvector), lexical (BM25 — implement it yourself first, 25 lines, then swap in
  `tsvector`), recency.
- RRF fusion at `k=60`, per-arm weights
  ([Cormack et al., SIGIR 2009](https://dl.acm.org/doi/10.1145/1571941.1572114)).
- Optional cross-encoder rerank; MMR diversification on the final selection.
- All metrics from `memlab.harness`, never re-implemented locally.

**Test:** 100 labelled queries over your own chat export with gold message ids (03.9 labelling
recipe, hand-check 10%).

### Bug 1: BM25 penalises documents for containing your query term

Implement BM25 from the textbook and you will write Robertson's IDF:

```python
def idf_robertson(t): return math.log((N - df[t] + 0.5) / (df[t] + 0.5))
```

Now run it on a **memory store**, where every row begins with "User":

```
df('user') = 11/12   idf_robertson = -2.037   idf_lucene = +0.123
df('vegetarian') = 2   idf_robertson = +1.435

query = 'user vegetarian'   (gold = doc 0: "User is vegetarian and avoids dairy")
 Robertson (negative IDF)   ['1.451 d8 "Vegetarian restaurants near Bandra"',
                             '-0.609 d0 "User is vegetarian and avoids dairy"',
                             '-1.715 d2 "User works at Beta Corp as a tech lead"']
 Lucene (+1 inside log)     ['1.791 d0 "User is vegetarian and avoids dairy"',
                             '1.667 d8 "Vegetarian restaurants near Bandra"',
                             '0.143 d1 "User lives in Mumbai"']
```

The gold document scores **negative** and loses to a document about restaurants. Any term appearing
in more than half the corpus gets a negative IDF, so a document is *punished* for containing it. Fix:
the variant Lucene ships, `log(1 + (N − df + 0.5)/(df + 0.5))`, which is asymptotically the same and
never negative.

This is not a toy corner case — it is the default state of a memory store. Extracted memories are
templated ("User is…", "User prefers…"), so the templating tokens are near-universal and the
textbook formula turns them into penalties. Postgres `ts_rank` and Lucene are both safe; a
hand-rolled BM25 over your own memory table is not. Check `df/N` for your ten most common tokens
before you trust any lexical arm.

### Bug 2: a zero-score arm still votes in RRF

A query with no lexical overlap with the gold memory — exactly the case 14.1 asks about, and the
reason dense retrieval exists:

```
query: 'where should I eat dinner tonight'
 BM25 top3: ['0.000 d0', '0.000 d1', '0.000 d2']
 overlap with gold d0 terms: set()   -> gold BM25 score 0.0
```

Every score is zero. But `sorted()` is stable, so the arm still emits a ranked list — document order
— and RRF happily fuses it:

```
 BM25 run : [0, 1, 2, 3]            ← meaningless: all scores 0.0
 dense run: [0, 8, 10, 3]
 RRF fused: [0, 3, 1, 8]            ← d3 "prefers three bullets" promoted to rank 2
 RRF fused (zero-score docs dropped): [0, 8, 10, 3]      ← d3 falls to rank 4
```

An irrelevant memory reaches rank 2 purely from tie-order inside a dead arm. The fix is one line —
`[i for s, i in scored if s > 0]` before fusion — and the general rule is: **an arm that abstains
must abstain, not shuffle.** Then note the corollary, visible in the same output: RRF's rank-1 to
rank-2 weight ratio at `k=60` is `(1/61)/(1/62) = 1.016`, i.e. RRF is deliberately almost
rank-insensitive. That flatness is why it is robust across arms with incomparable scores, and also
why it cannot rescue you from a noisy arm — it will average the noise in with everything else.

### Measure

| Config | Recall@20 | nDCG@5 | Tokens/query | p95 latency |
|---|---|---|---|---|
| dense only | | | | |
| BM25 only (Lucene IDF) | | | | |
| BM25 only (Robertson IDF) | | | | |
| RRF(dense, bm25) | | | | |
| RRF + rerank | | | | |
| RRF + rerank + MMR | | | | |

Include the Robertson row. Watching a "correct" implementation of a published formula underperform
on your data is the lesson.

**You are done when**

- [ ] You can name one query only BM25 finds (an identifier, a rare proper noun) and one only dense
      finds (a paraphrase with zero term overlap), and explain each failure.
- [ ] Your BM25 arm reports zero negative IDFs on your corpus, asserted in a test.
- [ ] Fusion drops abstaining arms, with a test that fails if a zero-score arm votes.
- [ ] All metrics come from `memlab.harness`; every accuracy number carries its bootstrap CI.

**Stretch:** sweep `ef_search` from 10 to 400 and plot recall against p95 (03.3 has the shape to
expect), then compare with brute force on the same data and find the corpus size where the index
starts winning. It is later than you think.

---

## 11.4 S3 — The Fact Store (ch 04) · ~8 hours · *the most important one*

**Build:** extraction + reconciliation with ADD / UPDATE / DELETE / NOOP.

**Requirements**

- Typed memory categories with volatility and TTL (04.2).
- Extraction into atomic claims with `category`, `stance`, `confidence`, `evidence_span`,
  `temporal`. **Validate that `evidence_span` occurs literally in the source and reject if not** —
  the cheapest hallucination check in the course.
- The gate cascade before any model call: opt-out marker, hypothetical detector, untrusted-content
  fence, secret/third-party scanner, minimum length (04.2).
- Reconciliation against the top-8 related memories, emitting one of the four ops.
- Soft supersession (`valid_to`, `superseded_by`) distinct from retraction (`retracted_at`) — 04.5.
- `NOOP` bumps confidence and resets decay.
- A `fact_key` (subject–predicate) with a unique index on single-valued predicates (04.4).

### Make the suite run without an API key

The twelve scenarios in 04.11 are the acceptance bar, and they are worthless if they only run when
someone exports a key. Write a **deterministic reference extractor** — regex rules, no model — and
run the same suite against both it and your real extractor. The reference version runs in
milliseconds on every commit and pins the *contract*; the LLM version is then measured against a
bar that already exists.

Here is the naive reference extractor's real score against the twelve scenarios:

```
naive extractor: 8/12 scenarios pass
  FAIL 3 retraction       leaked  'sister'      facts="User's sister is a doctor | User's cousin is a doctor"
  FAIL 4 hypothetical     leaked  'Paris'       facts='User lives in Paris'
  FAIL 5 third-party PII  leaked  '9876543210'  facts='User number 9876543210'
  FAIL 11 injection       leaked  'admin'       facts='User is admin'
```

Four failures, and every one is a category of bug rather than a typo: no correction handling, no
mood detection ("if I moved to Paris"), no third-party boundary, no trust boundary. Exactly the four
the chapter warned about. Fix them with the gate cascade and correction handling, and the suite goes
to 12/12.

### Bug 3: your runner is ignoring a third of your assertions

Before celebrating 12/12, audit the runner against the scenario spec:

```python
KNOWN = {"match"}            # keys the runner actually implements
def audit_assertions(scen, known=KNOWN):
    return [(s.name, k, e) for s in scen for e in s.expect_facts + s.expect_absent
            for k in e if k not in known]
```

```
assertion keys the runner silently ignores:
   2 supersession   key='current'     in {'match': 'Mumbai', 'current': True}
   2 supersession   key='closed'      in {'match': 'Pune',   'closed':  True}
   8 temporal       key='valid_from'  in {'match': 'Acme',   'valid_from': '2025-03'}
scenarios passing partly on unchecked assertions: 2
=> claimed 8/12, actually verified: 6/12 fully
```

The canonical supersession scenario — the one the whole chapter is about — was passing on a substring
match while the `valid_to` behaviour it exists to test went unchecked. **Make the runner assert on its
own vocabulary:**

```python
def run_strict(scenarios):
    known = {"match", "current", "closed", "valid_from"}
    for s in scenarios:
        for e in s.expect_facts + s.expect_absent:
            bad = set(e) - known
            assert not bad, f"scenario {s.name!r}: runner cannot check {bad}"
    ...
```

Four lines. Now an assertion you have not implemented is a loud error instead of a free pass. Any
test DSL without this check is quietly optional.

### Bug 4: three safety gates can be deleted and the suite stays green

A 12/12 suite still tells you nothing until you check that it *can* fail. Mutate one gate at a time
and see which scenario catches it:

```
gate: opt-out disabled           -> SURVIVED  caught by -- nothing --
gate: hypothetical disabled      -> KILLED    caught by ['4 hypothetical']
gate: untrusted fence removed    -> SURVIVED  caught by -- nothing --
gate: secret scan removed        -> SURVIVED  caught by -- nothing --
retraction detection removed     -> KILLED    caught by ['3 retraction']
mutation score: 2/5
```

Three of five gates are dead code as far as the suite is concerned. The reason is subtle and it
applies to every `expect_absent` assertion ever written: **scenarios 5, 6 and 11 passed because the
extractor was too weak to produce the forbidden fact in the first place.** "No job-hunting memory was
stored" is satisfied equally by a working opt-out gate and by an extractor that cannot parse
"I'm job hunting".

The fix is a **positive control**: prove the extractor *would* have produced the fact if the gate
were off. Broaden the reference extractor to be realistically greedy — as an LLM extractor is — and
re-run:

```
suite with greedy extractor + gates: 12/12 pass
gate: opt-out disabled           -> KILLED   caught by ['6 opt-out']
gate: hypothetical disabled      -> KILLED   caught by ['4 hypothetical']
gate: untrusted fence removed    -> KILLED   caught by ['11 injection']
gate: secret scan removed        -> KILLED   caught by ['5 third-party PII']
retraction detection removed     -> KILLED   caught by ['3 retraction']
mutation score: 5/5
```

Same twelve scenarios, same 12/12, and now the green light is load-bearing. **Every negative
assertion in your suite needs a paired positive control, and the cheapest proof that a suite works is
that a broken system fails it.** This generalises past memory systems, but memory systems are where
it bites hardest, because most of the safety properties are negative ones.

### Measure

- Scenario pass rate (target 12/12) **and mutation score over ≥ 5 mutants** (target 5/5).
- Gate false negatives on the 04.11 `GATE_SET` (target 0 — non-negotiable).
- Extractor precision/recall against your own hand-labels on 20 real conversations (targets ≥ 0.85
  and ≥ 0.70).
- Duplicate rate: fraction of within-tenant memory pairs with cosine > 0.95 (target < 2%).
- Write-path cost per session in tokens, split extraction vs reconciliation (04.10).

**You are done when**

- [ ] Scenarios 4 (hypothetical), 6 (opt-out), 7 (idempotency) and 9 (negation) pass — all four
      will fail on your first attempt.
- [ ] Mutation score is 5/5 and each `expect_absent` scenario has a named positive control.
- [ ] `run_strict` raises on an assertion key it cannot check, proven by a test that adds a bogus key.
- [ ] For each fix you wrote down whether it was a prompt fix or a code fix, and re-ran the suite
      after a model change. Prompt fixes regress; code fixes do not. That record is the exercise.

---

## 11.5 S4 — Time Machine (ch 05) · ~5 hours

**Build:** bi-temporal fact storage with point-in-time queries.

**Requirements**

- The 05.2 `facts` schema, both clocks: `valid_from/valid_to` (world) and
  `recorded_at/expired_at` (system).
- Both database-level invariants from 05.2: `CHECK (valid_to IS NULL OR valid_to > valid_from)` and
  the partial unique index giving single-valued predicates at most one current row.
- Temporal extraction for absolute, relative and implicit expressions against `t_ref`, including
  *future* validity ("starting next month").
- A declared `CARDINALITY` table driving deterministic invalidation; LLM adjudication only for
  multi-valued predicates (the contradiction 10.6 found in Cognee is the warning here).
- All four canonical queries, plus duration.

### The comparison that justifies the chapter

Both schemas loaded with the same six facts from the 05.9 worked example, run in SQLite:

```
Q1 true now           : [('has_role','tech lead'), ('lives_in','Mumbai'), ('works_at','Beta Corp')]
Q2 true on 2026-03-20 : [('has_role','tech lead'), ('lives_in','Mumbai'), ('works_at','Acme')]
Q3 believed 2026-03-20: [('has_role','tech lead'), ('lives_in','Pune'),   ('works_at','Acme')]
Q4 duration           : [('Acme', 38), ('Beta Corp', 4)]     ← months, from the validity window
```

Q2 and Q3 pin the *same date* and disagree: Mumbai versus Pune. Q2 is truth as now understood, Q3 is
the belief the agent acted on — the answer to "why did you send it to Pune?". Then the same six
questions against a flat `(subject, predicate, object, created_at)` table with overwrite-on-update:

```
  current employer       Beta Corp
  employer in Mar 2026   NOT EXPRESSIBLE
  belief on 2026-03-20   NOT EXPRESSIBLE
  months at Acme         NOT EXPRESSIBLE
  city on 2025-06-01     NOT EXPRESSIBLE
  why we said Pune       NOT EXPRESSIBLE
  flat answers 1/6; bitemporal answers 6/6
```

**Not "less accurate" — not expressible.** No prompt, model upgrade or retrieval tweak recovers a
column you did not write. That is the difference between a schema problem and a quality problem, and
it is the reason this project exists at hour 20 of 33 rather than as a stretch goal.

### Measure

- Correctness on 20 hand-written temporal cases across: current state, historical state, duration,
  ordering, future-dated facts, retroactive statements. Report accuracy **with CI** (n=20 gives a
  ~35pp interval — say so, and treat the categories as a checklist rather than a score).
- The flat baseline on the same 20, split into *wrong* and *not expressible*. That split is the
  finding.
- Tokens and p95 for each of the four queries; the partial index from 05.2 should keep Q1 under a
  millisecond at project scale.

**You are done when**

- [ ] `Q2 ≠ Q3` on your own data, and you can explain the divergence to someone in one sentence.
- [ ] "How long were they at Acme?" returns a number computed from a validity window, not a string.
- [ ] Inserting a retroactive fact ("I actually moved in January") produces **three** rows — expire
      the old open-ended belief, re-assert it closed, add the backdated new fact (05.2, row 6).
- [ ] The partial unique index fires: a reconciler that forgets to close a window fails at INSERT.
      Prove it with a deliberately broken write.
- [ ] Retraction behaves differently from supersession, with a test for each.

---

## 11.6 S5 — The Night Shift (ch 06) · ~5 hours

**Build:** the background maintenance tier.

**Requirements**

- A job runner (queue + worker, or a scheduler) with **per-tenant ordering** — out-of-order
  supersession corrupts validity windows.
- Four jobs: consolidation, reflection with the 06.2 usefulness filter, decay scoring and archival
  tiering, and a contradiction sweep.
- Consolidated and reflected memories record `derived_from` — required for chapter 09's erasure
  cascade, and you cannot retrofit it.
- A procedural memory file maintained by the agent, with `human-pinned` entries immune to pruning,
  enforced in code (06.1).

### The latency argument, measured

20,000 simulated turns, stage medians from the 07.4 budget (retrieve 35ms, generate 1,400ms,
extract 900ms, reconcile 1,100ms), lognormal with σ 0.35–0.55:

```
background (async write)   p50   1433ms  p95   2515ms  p99   3134ms
inline (sync write)        p50   3679ms  p95   5835ms  p99   7173ms
p95 delta: 3320ms (2.32x)
naive 'add the p95s': 7348ms vs measured 5835ms -> overstates by 1513ms
```

Two results. **Inline memory management costs 2.32× p95** — the reason 06.3 splits the agent in two.
And **adding per-stage p95s overstates the composed p95 by 1,513ms (26%)**, because stages do not
peak together; §7.4 has the derivation. If you size your budget by adding p95s you will buy hardware
you do not need, and if you *only* learn that rule you will still be wrong in the other direction
under fan-out, where the tail gets worse rather than better (7.4, mistake 2).

### A caveat that is itself a lesson

Consolidation on 300 synthetic memories:

```
memories before 300  after 46  reduction 85%
lives_in contradictions before 2  after 0
```

**Do not put that 85% in a report.** The generator produced 300 memories from 11 note templates and
3 cities, so the reduction measures the *generator's* duplication rate, not the consolidator's
skill. Synthetic-data metrics that depend on redundancy measure the data. Use synthetic data to test
the *mechanism* (did contradictions reach zero? did `derived_from` get written? did a human-pinned
rule survive?) and measure compression on real memories only.

### Measure

- Count before/after consolidation on **real** memories; hand-rate 20 consolidated entries for
  information loss (1–5). Any 1 or 2 means consolidation is lossy and must be reverted.
- Reflection actionability, 20 insights rated 1–5. Mean below 3 means the *filter* is too weak —
  iterate on `is_useful_insight`, not the prompt (06.2).
- Contradictions before/after the sweep, by predicate.
- Conversation p50/p95 inline vs background, and the chapter-04 scenario suite for both.

**You are done when** background processing improves **both** latency and quality — p95 down and
scenario pass rate not down. A win on one axis only means the background job is doing the wrong work.

- [ ] Per-tenant ordering proven: shuffle two supersessions for one tenant and show the queue
      serialises them and the final validity windows are correct.
- [ ] A human-pinned procedural rule survives an agent pruning pass over an oversized rule file.
- [ ] `derived_from` is populated on every derived memory, asserted by a NOT NULL constraint.

---

## 11.7 S6 — Break It (ch 08, 09) · ~6 hours

**Build:** the eval harness and the red team. This is where the previous five projects get graded.

### Part A — Eval harness

- 100 cases from your own data: 20 recall, 15 multi-session, 20 knowledge update, 15 temporal,
  15 abstention, 10 preference, 5 deletion (08.6).
- **Queries run in a new session** so session memory cannot answer them. Without this, you are
  testing the context window.
- An LLM judge with the 08.6 abstention rule, calibrated against 100 hand-graded cases.
- Report the triple per category through `memlab.Run`.

Two numbers to internalise before you pick your set size:

```
observed 0.80 accuracy, 95% bootstrap CI:   n=30 -> ±13pp    n=100 -> ±8pp    n=300 -> ±4.5pp
true 60% vs true 80%: better system measures worse-or-equal   n=10 -> 23%   n=30 -> 5%   n=100 -> 0%
```

And use **paired** comparisons — the same cases through both configurations. With n=100 and a true
20pp gap, the paired sign of the difference favoured the better system in **100% of 20,000
simulated runs**, because per-case difficulty cancels. §8.8 quantifies the unpaired version of the
same comparison at a fraction of the power. Pairing is free; not pairing costs you cases.

Judge calibration is not optional either: at judge agreement `q` with truth, a measured difference
is attenuated by `(2q − 1)`, so a judge at 90% agreement shrinks a real 10pp gap to 8pp, and one at
75% shrinks it to 5pp (08.7). Report judge agreement next to every judged number.

### Part B — Red team

- Ten injection payloads across three delivery channels: direct user statement, content inside a
  document the agent reads, and a tool result. All aim to plant a durable instruction (09.3, T1).
- Measure how many get **written**, then how many **influence a later session** — the second number
  is the one that matters, and it is the one 09.2's executed chain demonstrates.
- Implement provenance + trust levels (09.4) and re-measure.
- Implement the erasure cascade with derived-memory regeneration, then go looking for residue. 09.6
  found **seven residue sites** after a naive `DELETE`, including a contentless-FTS5 index that
  cannot be cleaned after the row is gone. Count yours; the count is the deliverable.

| | Before controls | After controls |
|---|---|---|
| Payloads written to memory (of 10) | | |
| Payloads that influenced a later session (of 10) | | |
| Deleted-fact residue sites found | | |
| Tokens/query (control overhead) | | |
| p95 latency (control overhead) | | |

The last two rows exist because a control you cannot afford will be switched off in three months.

**You are done when**

- [ ] Zero payloads become instruction-stance memories, and you can point at the code line that
      blocks each channel (not a prompt instruction).
- [ ] Deletion cascade at 100%: a deleted fact appears in no memory row, no vector for any
      `model_id`, no lexical index, no derived summary, no cache, no analytics extract.
- [ ] Residue sites enumerated by grepping for every place memory text is written — then the list
      is committed as a test.
- [ ] Judge agreement reported alongside every judged accuracy, with the attenuation applied.
- [ ] Every headline accuracy carries its bootstrap CI, and no acceptance bar rests on n < 30.

---

## 11.8 Wiring it together

After S1–S6 you have a complete memory system. Spend an hour drawing the 06.6 tier diagram for
*your* system, naming the file and function that owns each box:

```
IN CONTEXT (always)      procedural file (S5) · core blocks (S5) · recent window (S1)
IN CONTEXT (retrieved)   facts w/ validity (S3+S4) · episodes (S2) · reflections (S5)
ON DISK                  episode log (S1) · fact store (S3) · bitemporal history (S4)
BACKGROUND               extraction (S3) · consolidation, reflection, decay (S5)
GATES                    salience (S3) · trust (S6) · erasure cascade (S6)
INSTRUMENTS              triple + CI + mutation score (11.1) · red-team scoreboard (S6)
```

Any box you cannot name is a gap, and the chapter it belongs to predicts the failure: no procedural
tier means flaky repeated mistakes (06.0); no background tier means quality plateaus (06.3); no
episode log means you can never re-derive (04.6); no mutation score means you do not know whether
any of the above is tested.

The instruments row is the addition this pass makes. Five of the bugs in this chapter were found by
instruments, not by reading code: a guard measured against a labelled set, a runner that asserts on
its own vocabulary, a mutation sweep, an IDF sanity check, an abstaining-arm test. Build the
instruments and the bugs surface on their own.

---

## 11.9 Failure modes of doing these projects

| Symptom | Root cause | Fix |
|---|---|---|
| "It works" but you cannot say how well | no `Run`/triple, no CI | 11.1 harness; accuracy only ever ships with a CI |
| Two configs "improved" by 3pp on 50 cases | interval wider than the effect | 11.0 CI table, 11.7 n≥100 and paired |
| Suite is green, production is broken | assertions the runner ignores | 11.4 `run_strict` raises on unknown keys |
| Suite is green, a gate is dead code | negative assertion with no positive control | 11.4 mutation score, paired positive control |
| Compaction guard never fires | guard regex recall never measured | 11.2 labelled identifier shapes, ≥9/10 |
| Lexical arm ranks the gold memory last | negative Robertson IDF on templated rows | 11.3 Lucene IDF, assert no negative IDFs |
| Irrelevant memory at rank 2 after fusion | zero-score arm still emits ranks | 11.3 drop abstaining arms before RRF |
| "recall@5" differs between two of your scripts | metric defined per-experiment | 11.1 one definition, imported |
| Consolidation reports 85% compression | metric measures the synthetic generator | 11.6 mechanism on synthetic, compression on real |
| Capacity plan is 26% too pessimistic | per-stage p95s added | 11.6 / 7.4 compose, don't add |
| Temporal question cannot be answered at all | flat schema, not a quality issue | 11.5 bi-temporal columns, 1/6 → 6/6 |
| Deleted fact reappears in a summary | no `derived_from`, cascade stops at the row | 11.6 constraint, 11.7 cascade + residue count |
| S6 red team scores 0/10 on the first run | extractor too weak to be attacked | 11.4 positive controls — your "pass" is vacuous |

---

## 11.10 Exercises

1. **Break your own guard.** Take the S1 identifier guard and write ten identifier shapes you did
   *not* have in mind when writing the regex (locale formats, lowercase prefixes, slashes, short
   hashes). Acceptance: the guard's recall on that set is measured and reported, the set is
   committed, and recall ≥ 9/10 after fixing.
2. **Mutate everything.** Extend the S3 mutation sweep to ten mutants including: supersession
   replaced by append, `fact_key` uniqueness dropped, dedup threshold set to 1.0, confidence
   reinforcement counting the agent's own restatements. Acceptance: mutation score ≥ 9/10, with each
   surviving mutant either killed by a new scenario or documented as intentionally untested.
3. **Find your negative IDF.** Compute `df/N` for the 20 most frequent tokens in your own memory
   table. Acceptance: you can state how many tokens exceed `df/N = 0.5`, and a test asserts your
   lexical arm produces no negative term weights.
4. **Six numbers, one run.** Implement all six definitions of recall@5 from §11.1 and run them on one
   retrieval result from S2. Acceptance: you report the spread on your own data and state — in your
   README — which definition your project uses and why.
5. **Make the clocks disagree.** In S4, construct a case where Q2 and Q3 return different answers for
   the same date from your own conversation history. Acceptance: both queries committed as tests, and
   a one-sentence explanation of which one you would show a user asking "why did you do that?".
6. **Price your window.** Re-run the §11.2 table with your own transcript and your own model rates.
   Acceptance: a four-row table with tokens, cost per session, and fact-retention, plus the turn
   number at which your window drops the planted fact.
7. **Pay for your controls.** Measure tokens/query and p95 with and without the S6 trust checks and
   erasure cascade. Acceptance: the overhead is a number, and you can say whether it is affordable at
   your 07.8 scale. A control whose cost you have not measured will be removed by someone who has.
8. **Regress on purpose.** After S3 passes 12/12, change your extraction model (or temperature) and
   re-run. Acceptance: you can list which failures were prompt-sensitive and which were code-enforced.
   The ratio is the single best predictor of how your system ages.

Next: `12-projects-capstone.md` — the same discipline at three-to-six-week scale, where the
deliverable is a design doc, a working system, and an eval report.
