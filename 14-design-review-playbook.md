# 14 — Design Review Playbook

> The questions I actually ask when someone brings a memory design to review, what a weak and a
> strong answer sound like, the follow-up that catches a bluff, and a rubric you can score.
>
> Use it three ways: to pressure-test your own design before review, to run a review in a fixed time
> box, and to prepare for system-design interviews on this topic (it is now a common one).

---

## 14.0 In plain words

### The home inspection

When you buy a house you walk through it and it looks nice. An inspector ignores the rooms. They go
to the basement, the roof, the electrical panel, and the place where the water comes in — because
those are the defects that are expensive, invisible from the hallway, and impossible to fix after
you have moved the furniture in.

A memory design review is a home inspection. The kitchen is retrieval: enjoyable to build, fun to
demo, cheap to change later. The foundation is deletion, isolation, provenance and the episode log.
Nobody volunteers to look at the foundation, which is exactly why it is where the defects are.

### The naive review

```
"Walk me through the architecture."   (designer talks for 25 minutes)
"Nice. Does it work?"                 ("yes, here's a demo")
"Do you have tests?"                   ("yes, all green")
"Ship it."
```

Three failure modes in four lines. The designer chose the route through the house. "It works" was
demonstrated at n=1. And "all green" was accepted without asking whether the suite *can* fail.

### The arithmetic that kills it

**1. First implementations are defective, and we have the record to prove it.** This course
implemented nine components from scratch and verified each by running it. Every single one was
wrong on the first attempt:

| Area | Where | Defects in the first implementation |
|---|---|---|
| Salience gate | 04.2 | 2 — negation via contraction, elided subject |
| Tenant isolation (RLS) | 07.3 | 2 — table-owner bypass, unset GUC not fail-closed |
| Deadline propagation | 07.4 | 1 — 901ms against a 120ms budget |
| Erasure cascade | 09.6 | 7 residue sites + 1 contentless-FTS5 ordering bug |
| Framework audit probes | 10.1 | 2 — `valid_to` matched inside `invalid_tool_message`; grep hit docstrings |
| Identifier guard | 11.2 | 1 — guard recall 1/6, passing while five identifiers were lost |
| Lexical retrieval arm | 11.3 | 2 — negative Robertson IDF, zero-score arm voting in fusion |
| Scenario runner | 11.4 | 2 — 3 of 9 assertion keys ignored; mutation score 2/5 |
| Citations | 13.1 | 3 — wrong page count, missing identifier, failed-lookup false negative |

**Nine areas, nine defective first attempts, 23 defects.** That is not a comment on the author; it
is the base rate. A review that assumes the design is probably fine and looks for surprises is
calibrated wrong. Assume roughly one defect per area and go find it.

**2. A short unstructured review cannot cover the surface.** There are ~25 reviewable areas in a
memory design. If `d` of them are defective and you sample `q` at random:

```
q\d         1        2        4        8
5       20.0%    36.7%    61.7%    88.4%
8       32.0%    54.7%    81.2%    97.8%
12      48.0%    74.0%    94.3%    99.9%
17      68.0%    90.7%    99.4%   100.0%
25     100.0%   100.0%   100.0%   100.0%
```

A 30-minute review is about 8 questions. With 4 defects present, random sampling finds at least one
81.2% of the time — and a designer-led walkthrough is *worse than random*, because the route is
chosen by the person most confident about the parts they chose to show you. [INFERENCE, but the
mechanism is not subtle.] The fix is not more time; it is **choosing the 8 questions from the defect
record above** rather than letting the architecture diagram choose them (§14.4).

**3. The cheap questions and the expensive questions are not the same questions.** Using ch 07.8's
workload of 60M new memories per month:

```
one month shipped without `derived_from` on derived memories
  = 60,000,000 rows that cannot be correctly cascade-deleted
retrofit requires re-deriving each from its source episodes
  -> if episodes were not kept (04.6), the retrofit is IMPOSSIBLE, not expensive
```

Compare with "we should add a reranker": an afternoon, any time, forever. **Review order should
follow retrofit cost, not interest.** Provenance, the episode log, the tenancy boundary and the
validity columns are the four things that cannot be added later; everything else is a schedule
decision.

### What fixes what

| Review failure | Section |
|---|---|
| Designer chooses the route | 14.2 time-boxed script, you choose |
| Ran out of time before deletion | 14.4 the eight questions, in order |
| "All tests green" accepted | 14.3 Q28 — can your suite fail? |
| "It works" at n=1 | 14.3 Q30–Q33 measurement protocol |
| Two reviewers, two verdicts | 14.5 rubric + agreement attenuation |
| Claim accepted because it had a citation | 14.3 Q34–Q35 evidence hygiene |
| Nits blocked the review, foundations passed | 14.5 starred rows are the only blockers |
| Design approved, then unfixable | 14.0 retrofit-cost ordering |

---

## 14.1 The opening question

> **"Walk me through what happens between the user saying 'I'm vegetarian' and the agent, three
> weeks later, recommending a restaurant."**

One question that exercises the whole system. Ask it first, let it run for five minutes, and take
notes on what they *skip* — the omissions are the review.

**Weak answer:** "We embed the message and store it, then do similarity search later." Five
decisions collapsed into one, and no model of the write path.

**Strong answer** traces: salience decision → extraction into an atomic typed claim with provenance
→ dedup and reconciliation against existing memories → async indexing with a freshness SLO → three
weeks later a retrieval gate fires → query construction (noting that "recommend a restaurant" has
**no lexical overlap** with "vegetarian", so slot-based retrieval or expansion is required) →
tenant-scoped hybrid retrieval → fusion → budget → rendering with validity and a
data-not-instructions fence.

The "no lexical overlap" observation is the tell: candidates who spot it have built this, candidates
who have not have read about it. §11.3 shows that case measured — BM25 scores **0.000** for every
document on that query, and the arm still votes in fusion unless you stop it.

**The follow-up that catches a bluff:** *"Where in that path does the 'I'm vegetarian' text
physically live three weeks later — name every table and index."* A real implementation answers with
five or six places (facts row, vector for each `model_id`, lexical index, episode log, any derived
summary, cache). An imagined one answers with one. That list is also the deletion cascade, which is
why this single follow-up predicts the answer to Q19 before you ask it.

---

## 14.2 Running the review in a fixed time box

You choose the route. Announce the structure at the start so nobody feels ambushed.

**30 minutes** — the eight questions in §14.4, in that order. No architecture walkthrough; you will
reconstruct the architecture from the answers, and gaps show up faster this way.

**60 minutes**

```
0:00-0:05   the opening question (14.1), uninterrupted
0:05-0:10   the follow-up: name every place the text lives
0:10-0:22   write path + time              (Q1-Q9, Q10-Q12)
0:22-0:32   read path                      (Q13-Q17)
0:32-0:44   systems + safety               (Q18-Q27; do the four starred ones first)
0:44-0:55   testing, evaluation, evidence  (Q28-Q37)
0:55-1:00   score the rubric out loud, name the blockers, agree the next artefact
```

**90 minutes** — the above, plus a live demonstration. Ask for three things on screen: the
cross-tenant isolation test failing closed with an unset GUC; the deletion of one fact followed by
a grep for its text across every store; and one mutation of a safety gate showing a test go red.
Fifteen minutes of watching beats an hour of describing.

**Scoring out loud at the end is not optional.** It converts "some concerns" into "isolation is a 1
and that is a blocker", which is the only form of feedback that gets acted on.

---

## 14.3 The question set

Each question: what you are listening for, and where the answer lives in this course.

### On the write path

**Q1. What decides that something is worth remembering?**
Weak: "the LLM decides." Strong: a typed policy with categories and volatility, an LLM only inside
it. → 04.2
*Follow-up:* "What is your gate's false-negative rate on a labelled set?" If there is no labelled
set, the gate is untested and 04.2's two regex bugs are probably still in it.

**Q2. What happens when the user contradicts a stored memory?**
Listening for: supersession distinct from retraction, "outdated ≠ wrong". → 04.5, 05.7

**Q3. Show me the ADD vs UPDATE vs DELETE vs NOOP decision.**
Listening for: DELETE reserved for facts that were never true; NOOP bumping confidence. → 04.5

**Q4. Is extraction synchronous?**
Should be no. *Follow-up:* "Then how do you handle read-your-writes inside a session?" Correct
answer: session memory covers it; long-term memory only has to be right by the next session. → 04.8

**Q5. What is your freshness SLO and how do you measure it?**
Listening for: `episode_ts → indexed_ts`, p99, measured from the rows rather than from the queue's
own metrics. → 07.7

**Q6. How do you prevent duplicate accumulation?**
Listening for: write-time dedup, a `fact_key`, and a **measured** duplicate rate. → 04.4
*Follow-up:* "What cosine threshold, and how did you pick it?" Anything copied from a blog post is a
finding — 04.4's calibration script exists because the right threshold moves with the model.

**Q7. What is the blast radius of one destructive turn?**
Listening for: a destructive budget and a review queue. "Forget everything" must not be able to
empty a profile. → 04.5

**Q8. Where does confidence come from, and is it calibrated?**
Listening for: user-sourced reinforcement only, and a reliability curve. If the agent's own
restatements raise confidence, hallucinations become certainties. → 04.6

**Q9. What do you do with a memory derived from another memory?**
Listening for: `derived_from`, a separate retrieval lane, re-derivation when sources change. This is
the retrofit-impossible column from §14.0. → 06.2, 09.6

### On time

**Q10. Can you answer "what did the system believe about the user on 1 March?"**
Listening for: two clocks. If `created_at` is the only timestamp, retroactive statements are all
wrong. → 05.2
*Follow-up:* "Show me a date where 'what was true' and 'what we believed' differ." §11.5 has that
case; a design that cannot produce one has not exercised its own schema.

**Q11. "How long was the user at Acme?" — can your schema express that?**
Listening for: a duration computed from a validity window. §11.5 measured a flat schema answering
**1 of 6** temporal questions, the other five *not expressible* — which is a schema defect, not a
quality issue, and no prompt fixes it. → 05.2

**Q12. Who enforces "one current value per single-valued predicate"?**
Listening for: a database constraint, not application code. → 05.2's partial unique index.

### On the read path

**Q13. Do you retrieve on every turn?**
Listening for: a gate, with a measured skip rate *and* a measured cost of skipping wrongly. → 04.2,
07.10

**Q14. How many memories do you inject, and how do you know that is the right number?**
Listening for: the usage/citation metric. **If they have never measured whether injected memories
are used, that is the single most common gap in real designs** — and it is also the largest line
item in the cost model (07.8: injected tokens at 11× all extraction). → 07.10

**Q15. What happens when retrieval returns contradictory facts?**
Listening for: validity filtering *before* ranking, and rendering with timestamps. → 05.2, 09.5

**Q16. Dense only, or hybrid?**
Listening for: BM25 for identifiers, and awareness that short templated memory text is exactly where
dense retrieval is weakest. → 03.2
*Follow-up:* "What is `df/N` for your most common token?" Memory rows all start with "User", so
textbook Robertson IDF goes **negative** and punishes documents for containing the query term —
§11.3 measured the gold memory ranking below an irrelevant one with a negative score.

**Q17. What is your latency budget, and what gets dropped first under pressure?**
Listening for: a documented skip order and deadline propagation. → 07.4
*Follow-up:* "Did you compose the percentiles or add them?" Adding per-stage p95s overstated a real
composed p95 by **26%** in §11.6; and 07.4's `TaskGroup` deadline bug measured **901ms against a
120ms budget** in the obvious implementation.

### On systems

**Q18. How is tenant isolation enforced?** ★
Listening for: something below the application layer — RLS or physical partitioning — plus a CI
test. *Follow-up:* "What happens when the tenant GUC is unset?" It must fail closed; 07.3 found it
failing open, plus a table-owner bypass. → 07.3

**Q19. Walk me through deleting one fact.** ★
Listening for: the full cascade — vectors across all `model_id`s, lexical index, graph edges, derived
memories **regenerated** rather than unlinked, session summaries, caches, analytics, backups. 09.6
measured **7 residue sites** after the deletion everyone ships. → 09.6
*Follow-up:* "Which store must you read *before* the delete?" Correct answer: any contentless
external index (09.6's FTS5 bug) — once the row is gone you cannot clean its index entry.

**Q20. What happens to the tenant with 500k memories?**
Listening for: working-set caps, separate background queue lanes, an explicit product quota. → 07.3

**Q21. How do you re-embed when you change embedding models?**
Listening for: `model_id` per vector, dual serving, per-tenant cutover, rollback. → 07.9

**Q22. What is the single largest line item in your cost model?**
Correct at almost every scale: injected tokens. "The vector database" means the spreadsheet does not
exist. → 07.8

### On safety

**Q23. Can content from a web page the agent read become a durable memory?** ★
Listening for: trust levels and a hard rule that untrusted content cannot become an
instruction-stance memory, enforced in code. → 09.4

**Q24. Can a memory reduce a confirmation requirement?** ★
Must be an immediate, unqualified no, with the confirmation logic not reading from memory at all.
→ 06.5, 09.3

**Q25. How would you detect that a tenant's memory has been poisoned?**
Listening for: behavioural drift monitoring against a fixed probe set — content scanning cannot see
sleepers, which look benign in isolation. → 09.8

**Q26. Can you turn memory off right now, without a deploy?**
Must be yes, at global, tenant and category granularity. → 07.11

**Q27. In a multi-agent system, what trust does a derived memory carry?**
Listening for: the minimum of its inputs. §12.4's simulation: without that rule one poisoned
low-trust memory reaches the **highest-privilege agent 85% of the time in ~2 hops**; with it, 0%.
→ 09.6, 12.4

### On testing and evaluation

**Q28. Can your test suite fail?** ★
This is the question the deep passes earned. Listening for: a mutation score. §11.4 had a suite
reporting **12/12 while three safety gates were deletable** — they passed because the extractor was
too weak to produce the forbidden fact. Every negative assertion needs a paired positive control.
→ 11.4

**Q29. How do you know it works?**
Listening for: the triple — accuracy, tokens/query, p95 — on an eval set built from their own
traffic. → 08.1

**Q30. How many cases, and what is the interval?**
Listening for: a number and a CI. At n=100 an 80% accuracy has a ±8pp interval; at n=10 the interval
is 50pp wide and the better of two systems measures worse 23% of the time (§11.0). → 08.8

**Q31. Is the comparison paired?**
Listening for: same cases through both configurations. Unpaired 20-run comparisons produce
[0.60, 0.95] against [0.50, 0.90] and settle nothing (§12.0). → 08.8

**Q32. What is your judge's agreement with human labels, and what did you do about it?**
Listening for: a number, and the attenuation `(2q − 1)` applied. A judge at 75% agreement halves a
real gap. → 08.7

**Q33. What is your load test's sample count?**
Listening for: enough for the percentile being claimed. A 100-request test **passes a system whose
true p99 is 164ms against a 150ms SLO 44% of the time** (§12.0). A p99 claim without a duration is
not a claim. → 12.2

**Q34. What fraction of your eval set is abstention?**
Listening for: ~15%. Zero means they will ship a confabulator. → 08.6

**Q35. What broke last time you changed the extractor prompt?**
Listening for: any specific answer. "We don't know" means there is no regression suite, and 04.11's
scenario suite is the missing artefact. *Follow-up:* "Which of your fixes were prompt fixes and
which were code fixes?" Prompt fixes regress at the next model upgrade; code fixes do not.

### On evidence

**Q36. That benchmark number — what was the actor model, the judge, and the token cost?**
Any missing one makes the number incomparable, including to its own previous version. → 08.3

**Q37. Where does that claim come from?**
Listening for: an identifier, not a name. §13.1 found a paper cited by name whose venue claim was
*correct* and still unverifiable, and a page count asserted from memory that was simply wrong.
Unverifiable-but-true is indistinguishable from unverifiable-and-false. → 13.1

---

## 14.4 If you only have thirty minutes

Eight questions, ordered by the §14.0 defect record and by retrofit cost. Do not reorder them to
follow the design's structure; that is how the foundation goes unexamined.

```
1. Walk me through deleting one fact.                       (Q19 ★)  retrofit: impossible
2. How is tenant isolation enforced, and what happens
   when the tenant id is unset?                             (Q18 ★)  retrofit: expensive
3. Can your test suite fail? Show me a mutation.            (Q28 ★)  retrofit: cheap, but nothing
                                                                      else you hear is trustworthy
                                                                      until this is answered
4. Can untrusted content become an instruction-stance
   memory, and can a memory reduce a confirmation?          (Q23,Q24 ★) retrofit: expensive
5. What did the system believe about the user on 1 March?   (Q10)    retrofit: impossible
6. What decides what is worth remembering, and what is
   the gate's false-negative rate?                          (Q1)     retrofit: cheap
7. How many memories do you inject, and what fraction
   does the model actually use?                             (Q14)    retrofit: cheap, largest cost line
8. How many eval cases, paired or unpaired, and what is
   the interval?                                            (Q30,Q31) retrofit: cheap
```

Question 3 is placed third deliberately. Everything after it is a claim about behaviour, and claims
about behaviour are only as good as the suite behind them — a design whose suite cannot fail has no
verified properties at all, only intentions.

---

## 14.5 Rubric

Score each dimension 0–3, and record the **evidence** you saw. Below 2 on any starred row is a
blocker, not a nit.

| Dimension | 0 | 1 | 2 | 3 | Evidence that distinguishes 2 from 3 |
|---|---|---|---|---|---|
| Write path decomposition | one LLM call does everything | extraction separated | + reconciliation with 4 ops | + typed policy, provenance, confidence | the 12-scenario suite running, with a mutation score |
| Temporal correctness ★ | no time modelling | `created_at` only | validity windows | bi-temporal, point-in-time queries | a date where belief ≠ truth, on their data |
| Retrieval | dense only, fixed k | + filters | hybrid + fusion | + rerank, gate, measured usage | citation rate for injected memories |
| Isolation ★ | app-level filter | + code review discipline | RLS or partitioning | + CI test + prod audit sampling | the unset-GUC test failing closed |
| Deletion ★ | row delete | + vectors | + derived regenerated | + caches, analytics, backups, tested | a grep across every store after a delete |
| Injection safety ★ | none | prompt fence | + trust levels enforced in code | + drift monitoring + rollback | a mutation test on the trust rule |
| Evaluation | vibes | manual spot checks | offline eval set | + abstention, CI gates, online metrics | intervals and judge agreement on every number |
| Measurement discipline ★ | numbers without units | the triple reported | + intervals | + paired comparisons, stated sample counts | the load test's sample count |
| Cost awareness | unknown | rough estimate | modelled | + largest line item with a reduction plan | sensitivity to block size |
| Operability | no metrics | basic latency | full metric set | + runbook, kill switch, snapshots | a kill switch flipped live |
| Background tier | none | summarisation | + consolidation | + reflection, contradiction sweep, decay | p95 **and** quality both improved |

Scoring it mechanically keeps reviews comparable across designs and reviewers:

```python
STARRED = {"temporal", "isolation", "deletion", "injection_safety", "measurement"}

def score(review: dict) -> dict:
    """review: {dimension: (score 0-3, evidence str)} — evidence is required, not optional."""
    missing = [d for d, (s, ev) in review.items() if s >= 2 and not ev.strip()]
    blockers = sorted(d for d in STARRED if review.get(d, (0, ""))[0] < 2)
    total = sum(s for s, _ in review.values())
    return {"total": total, "max": 3 * len(review),
            "blockers": blockers,
            "unevidenced_claims": missing,          # a 2+ with no evidence is a 1
            "verdict": "blocked" if blockers else ("ship" if total >= 2 * len(review) else "revise")}
```

`unevidenced_claims` is the part that earns its keep. A score of 2 recorded because someone *said*
they had RLS is a 1; the rubric should not let you forget which.

**A design scoring 2 across the board is shippable. A design scoring 3 on retrieval and 0 on
deletion is a lawsuit**, and that exact profile is the default outcome of an enjoyable project
(§12.5). Check the starred rows first, every time.

**One caution on the rubric itself.** Reviewers disagree, and disagreement attenuates exactly as
LLM-judge disagreement does (08.7). If two reviewers agree with the truth only `q` of the time, a
real one-point quality gap between two designs measures as `(2q − 1)`:

```
q=0.70 -> a true 1.0-point gap measures 0.40
q=0.80 -> 0.60
q=0.90 -> 0.80
q=0.95 -> 0.90
```

So do not use rubric totals to rank teams. Use them to find blockers, which is a threshold judgement
and far more robust than a difference.

---

## 14.6 Red and green flags

Phrases that should trigger deeper questions:

- **"The LLM decides what to remember."** Fine as a component, alarming as the whole policy.
- **"We just use cosine similarity."** No hybrid, no filtering, no temporal handling.
- **"We store everything and retrieve top-20."** No salience policy, and over-retrieval degrades
  quality via context rot while being the largest cost line.
- **"Deletion just marks a row."** The cascade is missing; 09.6 says there are about seven places.
- **"All our tests pass."** Ask Q28. §11.4's suite passed 12/12 with three gates deletable.
- **"We'll add evaluation later."** It will not be added later, and by then nobody will know whether
  a change helped.
- **"Memory is in the system prompt so it's trusted."** Directly inverted threat model.
- **"We picked [framework] because it scored highest on [benchmark]."** Ask Q36.
- **"It's basically RAG."** Reveals no model of the write path, which is the hard half.
- **"The TTL is 30 days."** Ask where the 30 came from. §12.3's churn study shows a mean-rate TTL
  over-invalidating by 2.1–2.6× because churn concentrates in a tenth of the files.
- **"We measured it in staging with a quick load test."** Ask Q33.

Green flags, conversely:

- They can state the query distribution the design is optimised for, **and what it is bad at.**
- They have a number for how often injected memories are actually used.
- They chose *not* to use a graph, with a reason.
- They can describe a failure they had and the test they added afterwards — and the test is a code
  fix, not a prompt fix.
- They volunteer a mutation score, or the recall of one of their own guards.
- They know which of their fixes will regress at the next model upgrade.

---

## 14.7 Pressure-test your own design

Answer these in writing before review. If an answer is longer than a paragraph or shorter than a
sentence, that part of the design is not ready.

1. Draw the 06.6 tier diagram for your system, naming the owning module for each box. Add the
   instruments row from §11.8 — guards, suite, mutation score, red-team scoreboard.
2. Answer all 37 questions from §14.3 in writing. The ones you answer with a paragraph of context
   are the ones you do not understand yet.
3. Fill in 07.11's failure table for your architecture, with a time-to-detect per row.
4. Compute your cost model and identify the largest line item, plus its sensitivity to block size.
5. Write the deletion cascade as an enumerated list of every place a fact leaves a trace. Then find
   the ones you missed by grepping for every site where memory text is written — and then delete one
   fact and grep for its text, which finds the sites your list forgot.
6. Measure one of your own detectors — a gate, a guard, a PII scanner — against a labelled set of at
   least ten positives in shapes you did not have in mind when you wrote it. §11.2's guard scored
   1/6 on that test.
7. Mutate five mechanisms and confirm a test goes red for each. A mutation score below 4/5 means
   your review will be about intentions.
8. List the three query types your system handles worst, and what you would build if they mattered.

Item 8 is what senior reviewers care about most. Every design has a shape it is bad at; knowing
yours — and saying so unprompted — is the difference between a design review and a defence.

---

## 14.8 Failure modes of design reviews

| Symptom | Root cause | Fix |
|---|---|---|
| Review ran out of time before deletion | designer-led route | 14.2 time box, 14.4 fixed order |
| Everyone agreed it looked good, it failed in month two | reviewed the kitchen, not the foundation | 14.0 retrofit-cost ordering |
| "All green" accepted as evidence | no mutation testing | 14.3 Q28 |
| Approved a p99 claim that was false | load test too short | 14.3 Q33, 12.2 |
| Approved an accuracy claim inside its own noise | no interval, unpaired | 14.3 Q30–Q31 |
| Review became an argument about the reranker | nits treated as blockers | 14.5 only starred rows block |
| Two reviewers, opposite verdicts | totals compared instead of thresholds | 14.5 agreement attenuation |
| Team fixed the nits and shipped the blocker | verdict never said the word "blocker" | 14.2 score out loud, name blockers |
| A cited number turned out wrong | citation by name, not identifier | 14.3 Q37, 13.1 |
| Same defect class found in three consecutive reviews | findings never fed back into the checklist | 14.9 ex. 5 — add a row per defect |

---

## 14.9 Exercises

1. **Review a real system, blind.** Pick one framework from chapter 10, answer §14.4's eight
   questions from its source code alone, and score the rubric. Acceptance: eight answers with file
   and line citations, a rubric score, and a named blocker or an explicit "none".
2. **Review your own S3.** Run §14.7 against the fact store from project S3. Acceptance: every
   question answered in writing, and at least two items you could not answer — those are your real
   gaps, and the chapter reference for each tells you what will break.
3. **Time-box it.** Run a 30-minute review on a colleague's design using §14.4, then a 60-minute one
   using §14.2. Acceptance: the defects found in each, and whether the extra 30 minutes found
   anything the eight questions missed. If it did not, keep using the eight.
4. **Calibrate two reviewers.** Have two people score the same design independently. Acceptance: the
   per-dimension agreement rate, and the attenuation `(2q − 1)` it implies for any comparison you
   were planning to make with those scores.
5. **Grow the checklist.** For every defect you find in a real review, add a row to §14.3 with the
   question that would have caught it. Acceptance: after five reviews your question set has grown and
   at least one of the new questions has caught a defect a second time. That second catch is the
   whole point of a playbook.
6. **Prove the coverage arithmetic on yourself.** Seed a design doc with three deliberate defects,
   hand it to a reviewer who has not read this chapter, and count how many they find in 30 minutes.
   Acceptance: the count, and a comparison against the §14.0 table. Then have them re-review with
   §14.4 and compare.

---

## 14.10 A closing note

The systems in this course are not hard because the ideas are hard. Extraction, retrieval and
validity windows are all conceptually simple. They are hard because memory is **stateful,
long-lived, personal, and adversarially reachable** — four properties that individually make systems
difficult and jointly make them unforgiving.

The engineering discipline that matters, in priority order:

1. **Provenance on everything.** Every memory knows where it came from. That one property makes
   deletion, audit, debugging, citation and re-derivation possible. Skipping it is the decision you
   will most regret, because it is the one you cannot reverse — 60M rows a month, unattributable
   forever (§14.0).
2. **Never destroy the source.** Facts are derived; episodes are truth. You will want to re-extract
   with a better model, and you can only do that if you kept the raw material.
3. **Measure before optimising.** The triple, always, with intervals. Most systems inject far more
   memory than the model uses, and nobody knows because nobody measured.
4. **Assume the memory is wrong.** Design read paths so a bad memory degrades an answer rather than
   corrupting an action. Hints, never permissions.
5. **Measure your instruments.** This is the discipline the deep passes added, and it is the one
   nobody writes down. Every guard, gate, judge, suite and link checker in this course was wrong on
   first use — 9 areas, 23 defects, all found by pointing a test at the test. A green light nobody
   has tried to turn red is not evidence of anything.

Get those five right and the rest is refinement.

---

*End of the course. If you have done S1–S6 and one capstone, you have built every component in
chapters 01–09 from scratch, measured each one, and found your own versions of the defects
documented here. That is the qualification — not the reading.*
