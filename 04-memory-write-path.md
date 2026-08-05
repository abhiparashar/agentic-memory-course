# 04 — The Write Path

> Goal: decide what is worth remembering, turn it into durable state, and keep that state true as
> the world changes. This is where memory systems are actually won or lost.

Retrieval has 30 years of literature and off-the-shelf answers. The write path does not. Nearly every
"our agent's memory is bad" complaint traces back to something in this chapter.

---

## 4.1 The five decisions

Every write is five decisions, and most implementations collapse them into one LLM call, which is
why they are hard to debug. Separate them:

```
turn ──► [1] SALIENCE   should anything be remembered at all?
         [2] EXTRACT    what atomic claims does it contain?
         [3] RESOLVE    do these refer to entities/memories I already have?
         [4] RECONCILE  ADD / UPDATE / DELETE / NOOP against existing state
         [5] COMMIT     write + index + record provenance
```

Keep them as separate, individually testable functions. When quality is bad you need to know
*which* stage failed, and a monolithic "update my memory" prompt cannot tell you.

---

## 4.2 Salience: what deserves to persist

The naive extremes both fail:

- **Store every turn.** Write amplification, retrieval noise, cost. Your top-k fills with "sure,
  let me help with that".
- **Store only on explicit "remember this".** Users almost never say it. You lose the 90% of signal
  that is stated incidentally.

What actually works is a **typed policy**: define the categories you care about, and extract only
those. Untyped "extract anything interesting" produces a junk drawer.

```python
MEMORY_CATEGORIES = {
    "identity":      {"ttl": None,        "volatility": "low"},    # name, pronouns, languages
    "preference":    {"ttl": None,        "volatility": "low"},    # dietary, tone, formats
    "context":       {"ttl": "365d",      "volatility": "high"},   # employer, city, role
    "relationship":  {"ttl": None,        "volatility": "medium"}, # family, colleagues, pets
    "project":       {"ttl": "180d",      "volatility": "high"},   # what they're working on
    "commitment":    {"ttl": "90d",       "volatility": "high"},   # things agent promised
    "constraint":    {"ttl": None,        "volatility": "low"},    # accessibility, compliance
    "skill_learned": {"ttl": None,        "volatility": "low"},    # procedural: how to do X here
    "correction":    {"ttl": None,        "volatility": "low"},    # user corrected the agent
}
```

Three rules for choosing categories:

1. **A category earns its place only if it changes agent behaviour.** If you cannot name the prompt
   or tool call that would consume it, do not store it.
2. **Volatility drives everything downstream** — TTL, recency weighting, re-confirmation. "Vegetarian"
   is low volatility; "currently working on the payments migration" is high.
3. **`correction` is the highest-value category and the most often missed.** When a user says "no, I
   meant X", that is gold: it is explicit, high-confidence, and prevents a repeat failure. Give it
   its own category and its own boosted retrieval weight.

### Negative categories — what never to store

Encode this as an explicit deny-list, enforced in code, not only in a prompt:

- Sensitive-category personal data unless the user explicitly asked you to remember it (health,
  religion, sexuality, political affiliation, biometrics, precise financial identifiers). Regulated
  in most jurisdictions; treat as a hard rule.
- Secrets: API keys, passwords, tokens. Run a secret scanner on the write path.
- Third-party PII the user mentions in passing.
- Transient state that belongs in session memory, not long-term.
- Anything the user prefixed with "don't remember this" — and this must be honoured mechanically.

```python
def prefilter(text: str) -> tuple[bool, str]:
    if SECRET_SCANNER.detect(text):        return False, "secret_detected"
    if EPHEMERAL_MARKER.search(text):      return False, "user_optout"
    if len(text) < 8:                      return False, "too_short"
    return True, ""
```

---

## 4.3 Extraction

Turn conversational text into atomic, self-contained claims.

```python
EXTRACT = """Extract durable facts about the user from this exchange.

Rules:
- One atomic claim per item. Split compound statements.
- Self-contained: resolve all pronouns. "She likes it" -> "User's sister likes hiking".
- Only what is stated or clearly implied. No inference beyond one obvious step.
- Attribute stance: is this a FACT (user asserted), a PREFERENCE, or an INSTRUCTION to you?
- Include a temporal marker if present ("since March", "used to", "starting next week").
- If nothing durable is present, return an empty list. Empty is a correct and common answer.

Allowed categories: {categories}

Exchange:
{exchange}

Return JSON: [{{"claim": str, "category": str, "stance": "fact|preference|instruction",
                "confidence": 0.0-1.0, "temporal": str|null, "evidence_span": str}}]
"""
```

Details that matter, learned the expensive way:

**"Empty is correct."** Without that line, models feel obliged to produce something and you get
`"User is asking a question"` written to your database forever. This single sentence cuts junk
memories dramatically.

**`evidence_span`** — the verbatim substring supporting the claim. Enables provenance, lets you
verify the extractor did not hallucinate (assert the span is actually in the input), and lets you
show the user *why* you believe something. Make this required and validate it programmatically:

```python
def validate(claim: dict, source: str) -> bool:
    span = claim.get("evidence_span", "")
    return bool(span) and span in source     # cheap, catches real hallucinations
```

**`stance`** — a user saying "I'm vegetarian" (fact about them) is different from "always answer in
bullet points" (instruction to you) and from "I'd prefer morning meetings" (soft preference).
Instructions should be injected with much higher priority and near-100% recall; preferences can be
best-effort. Conflating them makes your prompt injection surface wider, too.

**Atomicity.** "I moved to Mumbai last month for a job at Acme" must become three claims (location,
move date, employer). Compound claims cannot be individually updated, and six months later when the
user changes jobs you cannot invalidate half a sentence.

**Batching.** Extract per exchange (user+assistant pair), not per message — the assistant's reply
often disambiguates the user's message. Do it asynchronously off the response path (see 4.8).

---

## 4.4 Resolution: linking to what you already know

New claim: "User works at Acme." Existing memory: "User is employed by Acme Corp." Same fact.

You need two kinds of linking:

**Entity resolution** — "Acme", "Acme Corp", "ACME Inc." are one entity. Approach:
1. Normalise (case, punctuation, legal suffixes).
2. Blocking: candidate generation via exact alias match + embedding nearest neighbours within the
   tenant, top ~20.
3. Scoring: string similarity + embedding similarity + type match + co-occurrence context.
4. Threshold with a rejection band: high → auto-merge, low → new entity, middle → queue for the
   background job or an LLM adjudication call.

**Never auto-merge in the middle band synchronously.** False merges are much worse than duplicates,
because they are hard to detect and corrupt two entities' histories at once. Duplicates are visible
and fixable.

**Memory deduplication** — is this claim already stored?

```python
def find_duplicates(claim_vec, tenant, threshold=0.92):
    return store.vector_search(claim_vec, filters={"tenant_id": tenant}, limit=10,
                               min_score=threshold)
```

Then a cheap LLM adjudication for the survivors, because 0.92 cosine can still mean "worked at" vs
"works at" — which is *not* a duplicate, it is a supersession.

---

## 4.5 Reconciliation: ADD / UPDATE / DELETE / NOOP

This is the heart of the chapter. The pattern was popularised by Mem0, whose architecture is
explicitly a two-stage extract-then-update design: an LLM extracts salient facts from the exchange,
then a decision step issues ADD, UPDATE, DELETE, or NOOP operations against existing memories to keep
the store consistent.

```python
RECONCILE = """You maintain a user's memory store. Decide what to do with a new claim.

New claim: {claim}   (category={category}, stated_at={ts}, confidence={conf})

Existing related memories:
{existing}   # id, text, category, created_at, valid_from, valid_to

Choose exactly one operation:
- NOOP:   already represented; adds nothing.
- ADD:    genuinely new information.
- UPDATE: refines or corrects an existing memory (same subject+attribute, better/newer value).
- DELETE: existing memory is now false and has no historical value.

Guidance:
- Prefer UPDATE over DELETE+ADD; UPDATE preserves history.
- DELETE only when the memory was WRONG (misextraction, user says "I never said that").
  If it was TRUE and is now merely OUTDATED, that is UPDATE (supersession), not DELETE.
- If the new claim contradicts an existing one and both could be true at different times,
  UPDATE with a validity window.
- If uncertain, ADD and flag for review — never silently destroy.

Return JSON: {{"op": "...", "target_id": str|null, "new_text": str|null, "reason": str}}
"""
```

### The distinction that everything hinges on

> **Outdated ≠ wrong.**
>
> "User lives in Pune" was true in 2024 and is false now. That is **supersession**: close the old
> fact's validity window and open a new one. The old fact is still needed to answer "where did I live
> before?" and to explain past behaviour.
>
> "User lives in Paris" extracted from "I'd love to live in Paris someday" was never true. That is
> **deletion**: it was an extraction error and it should leave no trace in the belief state.

Systems that only support add/delete conflate these and will either lose history or accumulate
contradictions. Graphiti's approach is the reference: when new knowledge conflicts with existing
knowledge, the temporal metadata is used to invalidate rather than discard, preserving historical
accuracy. Copy the semantics even if you do not copy the graph.

Concretely:

```sql
-- UPDATE (supersession) — never destructive
UPDATE memories SET valid_to = $now, superseded_by = $new_id WHERE id = $old_id;
INSERT INTO memories (id, text, valid_from, valid_to, supersedes) VALUES ($new_id, $text, $now, NULL, $old_id);

-- DELETE (retraction) — the belief was never correct
UPDATE memories SET retracted_at = $now, retraction_reason = $reason WHERE id = $old_id;
```

Note that even DELETE is a soft delete here. Hard deletion is a separate, audited path driven by
privacy requests (chapter 09), not by the reconciliation logic. Two very different operations that
must not share code.

### Conflict resolution policy

When two claims conflict, resolve by an explicit precedence order — do not leave it to the LLM:

1. **Explicit user correction** beats everything. ("No, I'm in Bangalore now.")
2. **Direct user statement** beats agent inference.
3. **More recent** beats older, *for volatile categories only*.
4. **Higher confidence** beats lower.
5. **More specific** beats more general.
6. If still tied: keep both with validity windows and let retrieval surface both with timestamps.

That last fallback matters. A memory system that is forced to pick a winner will sometimes pick
wrong and destroy the alternative. Keeping both, timestamped, lets the model reason at read time —
and lets the user correct you.

---

## 4.6 Confidence and provenance

Every memory row should carry:

```python
@dataclass
class Memory:
    id: str
    tenant_id: str
    subject: str                  # 'user' | entity id
    category: str
    text: str
    # provenance
    source_episode_ids: list[str]
    evidence_span: str
    extractor_version: str
    # belief state
    confidence: float
    stance: str                   # fact | preference | instruction
    # temporal
    created_at: datetime          # when we learned it
    valid_from: datetime          # when it became true
    valid_to: datetime | None     # when it stopped being true
    retracted_at: datetime | None
    superseded_by: str | None
    # usage
    last_accessed_at: datetime | None
    access_count: int
    positive_feedback: int
    negative_feedback: int
```

Why each field earns its place:

- **`source_episode_ids` + `evidence_span`**: citation, audit, debugging, and recomputation. Without
  them, when a user asks "why do you think that?", you cannot answer, and you cannot re-extract with
  a better model later.
- **`extractor_version`**: when you improve the extractor you need to know which memories came from
  the bad one, so you can backfill selectively.
- **`confidence`**: feeds retrieval weighting and lets you set a bar for what gets injected. Calibrate
  it against human labels; raw LLM-reported confidence is poorly calibrated but monotonic enough to
  rank with.
- **Usage counters**: input to decay (4.7) and to online quality metrics. `negative_feedback` in
  particular is the signal that a memory is actively harmful.

---

## 4.7 Forgetting

Unbounded memory is not a feature. It costs storage, slows retrieval, and increases the odds of
surfacing something stale or embarrassing. You need a deliberate forgetting policy — with a sharp
line between *ranking-level* forgetting and *actual deletion*.

### Level 1 — Decay in ranking (default)

Do not delete; down-weight. Cheap, reversible, low-risk.

```python
import math

def retention_score(m: Memory, now) -> float:
    age_days     = (now - m.created_at).days
    idle_days    = (now - (m.last_accessed_at or m.created_at)).days
    half_life    = HALF_LIFE_BY_CATEGORY[m.category]        # e.g. identity: inf, project: 90d
    if half_life == math.inf:
        recency = 1.0
    else:
        recency = 0.5 ** (idle_days / half_life)
    usage    = math.log1p(m.access_count) / 5.0
    feedback = 0.2 * m.positive_feedback - 0.5 * m.negative_feedback
    return (0.5 * recency + 0.3 * min(usage, 1.0) + 0.2 * m.confidence + feedback)
```

The spaced-repetition shape (accessed memories decay slower) is a deliberate borrow from human
memory research and works well: memories that keep proving useful stay strong.

### Level 2 — Consolidation

Multiple related memories collapse into one better one.

```
"User mentioned running on Monday"
"User mentioned running on Thursday"       ──►  "User runs regularly, ~3×/week, mornings"
"User asked about running shoes"                (with links to the source memories)
```

This is where memory quality compounds. Run it as a background job over clusters of related
memories, keeping links to the sources so you can still answer episodic questions and can undo a bad
consolidation.

**Danger:** consolidation is lossy and irreversible if you delete the sources. Do not delete sources
until the consolidated memory has survived a review period and shown positive usage. I would keep
sources for at least a quarter.

### Level 3 — Archival tiering

Move cold memories to cheap storage, out of the hot index. Retrievable on demand, not by default.
This is the correct answer for "we have too many memories" 90% of the time — far better than
deleting.

### Level 4 — Hard deletion

Reserved for: TTL expiry on genuinely ephemeral categories, user-requested erasure, and legal
requirements. Must cascade to every derived artefact (chapter 09 — this is harder than it sounds).

### The pruning trigger

Run when *any* holds:
- Tenant memory count exceeds a soft cap (e.g. 10k) → archive the bottom decile by retention score.
- A memory's retention score falls below a threshold and it has never been retrieved.
- TTL expiry for its category.
- Negative feedback exceeds a threshold → quarantine and review, do not silently delete.

Log every eviction with its score and reason. When someone reports "it forgot my X", you need to
answer why in minutes.

---

## 4.8 Where the write path runs: sync vs async

**Do not run extraction on the response path.** It adds an LLM round-trip to every turn for
information that will not be needed until a future session.

```
User turn ──► [respond]  ──────────────────────────────► user sees reply (p95 target)
       └────► [enqueue write job] ──► extract ──► resolve ──► reconcile ──► index
                                     (async, seconds to minutes later)
```

Consequences to handle:

- **Read-your-writes within a session.** If the user says "I'm vegetarian" at turn 3 and asks for a
  restaurant at turn 4, async extraction has not landed. **Session memory covers this** — the raw
  turn is still in the context window. Long-term memory only needs to be correct by the *next
  session*. This is the key insight that makes async safe.
- **Ordering.** Process a session's jobs in order, keyed by session id, or you will apply an UPDATE
  before the ADD it depends on.
- **Idempotency.** Jobs will be retried. Key writes on `(session_id, turn_index, extractor_version)`
  so a retry is a no-op.
- **Backpressure.** Extraction is LLM-bound and therefore rate-limited. Queue with a dead-letter and
  alert on lag; a silently growing extraction backlog is a classic incident.

**Batching wins.** Extracting once per session-end over the whole transcript is cheaper and *more
accurate* than per-turn extraction, because the model sees the resolution of things stated earlier.
Hybrid policy that works well in practice: extract at session end, plus immediately on turns
containing explicit memory markers ("remember", "always", "never", "from now on", "I prefer") or a
detected correction.

---

## 4.9 A complete write pipeline

```python
class MemoryWriter:
    def __init__(self, llm, store, embedder, metrics):
        self.llm, self.store, self.embedder, self.metrics = llm, store, embedder, metrics

    def process_session(self, tenant_id: str, session_id: str, exchanges: list[Exchange]):
        stats = {"extracted": 0, "add": 0, "update": 0, "delete": 0, "noop": 0, "rejected": 0}

        for ex in exchanges:
            ok, reason = prefilter(ex.text)
            if not ok:
                stats["rejected"] += 1; self.metrics.incr("write.prefilter", tag=reason); continue

            claims = self.llm.extract(ex, categories=MEMORY_CATEGORIES)
            for c in claims:
                if not validate(c, ex.text):
                    self.metrics.incr("write.hallucinated_span"); continue
                if c["confidence"] < 0.55:
                    self.metrics.incr("write.low_confidence"); continue

                stats["extracted"] += 1
                vec = self.embedder.encode_document(c["claim"])
                related = self.store.vector_search(vec, filters={"tenant_id": tenant_id},
                                                   limit=8, min_score=0.75)

                decision = self.llm.reconcile(claim=c, existing=related)
                self._apply(tenant_id, session_id, c, decision, vec, ex)
                stats[decision["op"].lower()] += 1

        self.metrics.gauge("write.session_stats", stats)
        return stats

    def _apply(self, tenant_id, session_id, claim, decision, vec, ex):
        op = decision["op"]
        if op == "NOOP":
            self.store.touch(decision["target_id"])                    # still record it was reinforced
        elif op == "ADD":
            self.store.insert(Memory(
                tenant_id=tenant_id, text=claim["claim"], category=claim["category"],
                stance=claim["stance"], confidence=claim["confidence"],
                source_episode_ids=[ex.id], evidence_span=claim["evidence_span"],
                valid_from=parse_valid_from(claim, ex.ts), created_at=ex.ts,
                extractor_version=EXTRACTOR_VERSION), vec)
        elif op == "UPDATE":
            self.store.supersede(decision["target_id"],
                                 new_text=decision["new_text"], vec=vec, at=ex.ts,
                                 reason=decision["reason"], source_episode_ids=[ex.id])
        elif op == "DELETE":
            self.store.retract(decision["target_id"], reason=decision["reason"], at=ex.ts)
```

The `NOOP → touch` line is easy to miss and matters: a repeated statement is *evidence*, even if it
adds no new text. Bump confidence and reset decay. A fact the user has told you four times should
outrank one they mentioned once.

---

## 4.10 Testing the write path

Retrieval you test with recall@k. The write path needs behavioural tests. Build a suite of scripted
scenarios with expected end states — these are the tests I would gate a release on:

```python
SCENARIOS = [
  # 1. simple add
  Scenario(turns=["I'm vegetarian"],
           expect_facts=[{"category": "preference", "match": "vegetarian"}]),

  # 2. supersession — the canonical case
  Scenario(turns=["I live in Pune", "...", "I moved to Mumbai last month"],
           expect_current=[{"match": "Mumbai"}],
           expect_historical=[{"match": "Pune", "valid_to": "not null"}]),

  # 3. correction / retraction
  Scenario(turns=["My sister is a doctor", "Sorry, I meant my cousin is a doctor"],
           expect_absent=[{"match": "sister.*doctor"}],
           expect_facts=[{"match": "cousin.*doctor"}]),

  # 4. hypothetical must NOT be stored as fact
  Scenario(turns=["If I moved to Paris, would you help me find an apartment?"],
           expect_absent=[{"match": "lives in Paris"}]),

  # 5. third-party PII not stored
  Scenario(turns=["My colleague Rahul's number is 98xxxxxx"],
           expect_absent=[{"match": "98xxxxxx"}]),

  # 6. opt-out honoured
  Scenario(turns=["Don't remember this, but I'm job hunting"],
           expect_absent=[{"match": "job hunting"}]),

  # 7. idempotency
  Scenario(turns=["I'm vegetarian", "I'm vegetarian", "As I said, I'm vegetarian"],
           expect_fact_count=1, expect_confidence_gte=0.9),

  # 8. temporal expression parsed
  Scenario(turns=["I started at Acme in March 2025"],
           expect_facts=[{"match": "Acme", "valid_from": "2025-03"}]),
]
```

Scenario 4 is the one that catches the most real bugs. Extractors love turning conditionals,
questions, and fiction into facts. Scenario 7 catches duplicate accumulation, which degrades
retrieval silently over months.

Run this suite on every extractor prompt change, every model upgrade, and in CI. Treat it exactly
like a unit test suite for a database, because that is what it is.

---

## 4.11 Exercises

1. Implement extraction + reconciliation with the prompts above. Run the eight scenarios. Expect to
   fail at least three on the first attempt — 4, 6, and 7 are the usual suspects. Fix by changing the
   prompt, then by changing the code, and note which fixes actually held.
2. Take 20 real conversations and hand-label what *should* be remembered. Measure your extractor's
   precision and recall against that. Precision below ~0.8 means your store is filling with junk.
3. Implement the retention score and run it over a synthetic year of memories. Plot what survives.
   Does anything survive that shouldn't? Does anything you'd want disappear?
4. Write the supersession SQL and prove with a query that you can answer "what did the system believe
   about the user on 2026-03-01?" Keep this query; chapter 05 extends it.

Next: `05-temporal-and-graph-memory.md`.
