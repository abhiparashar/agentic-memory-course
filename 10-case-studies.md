# 10 — Case Studies

> Goal: read real architectures, recognise the patterns from chapters 01–09 in them, and be able to
> make a build/buy decision you can defend to someone who will have to live with it for three years.
>
> **Snapshot: 2026-09-12.** Every number, licence, version, file path and line number in this chapter
> was taken from the live GitHub API and from shallow clones made on that date, on a machine with no
> cached copies. The commands are in §10.1 so you can re-run them and diff. Assume the numbers are
> stale the moment you read them; the *method* is the durable part.
>
> This chapter is deliberately the least trustworthy in the course, and it says so up front. Two of
> the five systems below changed something architecturally load-bearing since the paper you would
> read about them was published, and one of them no longer ships the code the paper describes.

---

## 10.0 In plain words

### Buying filing cabinets

You need a filing system for an office. Five vendors show up. Every brochure says the same four
things: *remembers everything, finds it instantly, scales, trusted by thousands.*

The brochures are useless because they all say the same thing. What actually differs is behind the
panels:

- Does a drawer keep the old card when you file a corrected one, or does it shred it?
- If you ask for a document to be destroyed, does it also come out of the index, the photocopies,
  and the summary binder the intern made?
- Can two tenants' cards end up in the same drawer?
- Can a human open a drawer and read it without a special tool?

Nobody writes those on a brochure. You find them by opening the cabinet and looking at the hinges.

### The naive version

Here is the evaluation almost every team actually performs. It is about twenty lines of effort:

```python
# the 20-line vendor evaluation
import requests

CANDIDATES = ["letta-ai/letta", "mem0ai/mem0", "getzep/graphiti",
              "langchain-ai/langgraph", "topoteretes/cognee"]

def evaluate(repo):
    r = requests.get(f"https://api.github.com/repos/{repo}").json()
    return {"repo": repo, "stars": r["stargazers_count"], "licence": r["license"]["spdx_id"]}

best = max((evaluate(c) for c in CANDIDATES), key=lambda d: d["stars"])
print("picked:", best)          # then: read the README, read the paper, start integrating
```

Read the README, skim the paper, pick the most-starred option with a permissive licence, start
integrating. It feels responsible. It is not.

### The arithmetic that kills it

**1. Popularity does not predict the properties you need.** Live counts, 2026-09-12:

```
mem0ai/mem0            65,135 ★    Apache-2.0
langchain-ai/langgraph 41,472 ★    MIT
getzep/graphiti        30,812 ★    Apache-2.0
topoteretes/cognee     30,641 ★    Apache-2.0
letta-ai/letta         24,698 ★    Apache-2.0
langchain-ai/langmem    1,660 ★    MIT
```

Spread between most and least starred: **39×**. Now the property that decides whether your CRM agent
gives correct answers about a customer who changed jobs — bi-temporal validity fields on the stored
fact (chapter 05) — is present in exactly one of them, and it is the one with **less than half** the
star count of the leader (§10.1, probe P1). The ranking you get for free is uncorrelated with the
ranking you need.

**2. The paper you read may describe code that no longer exists.** `letta-ai/letta` is the MemGPT
repo. On 2026-09-12 it contains **12 files, none of them source**: a README, a licence, policies, and
CI. The Python server the paper describes was moved to an `archive` branch and active development
moved to a *different repository in a different language*:

```
letta-ai/letta         @ main     →   0 source files  (12 files total: docs, licence, CI)
letta-ai/letta         @ archive  → 716 source files  (the retired V1 API server)
letta-ai/letta-code    @ main     → 1,897 source files (TypeScript, npm @letta-ai/letta-code)
```

An integration plan that began "pip install letta" was obsolete before it was written. The stars are
on the empty repo.

**3. The switching cost is not the integration cost.** A memory layer is a deep dependency: the data
is in *its* schema, extracted by *its* prompts. Moving means re-deriving. At an extraction cost of
~1,000 input tokens per stored memory and $3 per million input tokens:

```
   100,000 memories × 1,000 tok = 1.0e8 tok →     $300
 1,000,000 memories × 1,000 tok = 1.0e9 tok →   $3,000
10,000,000 memories × 1,000 tok = 1.0e10 tok →  $30,000   + re-embedding + the eval to prove parity
```

Re-embedding is cheap by comparison (1.0e10 tokens at $0.02/M ≈ $200), but it is the *proof of
parity* that costs weeks — see chapter 07's re-embedding runbook and chapter 08's fair-comparison
harness. The point: choose as if migration costs a quarter of engineering time, because it does.

**4. Churn is measurable, so measure it.** Every repo here was pushed to within 72 hours of the
snapshot. That is healthy, and it also means the interface you integrate against this week is not the
one you will maintain. Two of the five changed their core write-path semantics since publication
(§10.2, §10.3).

### What fixes what

| Failure of the naive evaluation | Fixed by |
|---|---|
| Chose by stars; the property you needed wasn't there | §10.1 — the five-question audit, executed against source |
| The paper describes retired code | §10.2 — Letta's repo split and what survived it |
| Assumed the write path reconciles the way the paper says | §10.3 — Mem0's move from update-in-place to append-and-link |
| Assumed `delete()` deletes | §10.3 `delete_linked=False`, §10.5 the one real FK cascade, §10.8 the vendor's own erasure warning |
| Assumed temporal correctness because the README says "temporal" | §10.4 — the four timestamps, in the schema, with line numbers |
| Built a vector pipeline for knowledge that must apply every time | §10.7 — files win, and here is exactly what files cost |
| Cannot answer "is it working?" | §10.10 — measure before you adopt, not after |
| Grep said the mechanism was there and it wasn't | §10.1 — the two bugs my own audit hit |

**The one-sentence takeaway:** brochures and star counts are noise; open the repository, run five
mechanical probes against the schema and the write path, read the files the probes point at, and
price the migration before you write the first import.

---

## 10.1 The five-question audit, executed

Chapter 10 used to *end* with five questions to ask of any memory system. Asking is cheap; this
section answers them mechanically, against cloned source, and then shows why the mechanical answer
is only a triage signal.

### Setting up

```bash
mkdir -p /tmp/ch10 && cd /tmp/ch10
for u in letta-ai/letta mem0ai/mem0 getzep/graphiti \
         langchain-ai/langgraph topoteretes/cognee langchain-ai/langmem; do
  git clone -q --depth 1 --single-branch "https://github.com/$u.git" "$(basename $u)"
done
git clone -q --depth 1 --single-branch https://github.com/letta-ai/letta-code.git letta-code
git clone -q --depth 1 --single-branch --branch archive \
          https://github.com/letta-ai/letta.git letta-archive
```

Real output — disk size, landed commit:

```
164M cognee/ c0d18c8 2026-09-09    90M letta-code/ (HEAD 2026-09-10)
 55M mem0/   c7ee362 2026-09-11    34M letta-archive/
 29M graphiti/ 19d7245 2026-09-11  18M langgraph/  e539ac1 2026-09-09
232K letta/  5bcdd17 2026-09-10  ← 12 files, no source
```

### The corpus

```python
import pathlib
ROOT  = "/tmp/ch10"
REPOS = {"letta (archive)": "letta-archive", "letta-code": "letta-code", "mem0": "mem0",
         "graphiti": "graphiti", "langgraph": "langgraph", "cognee": "cognee"}
EXT  = {".py", ".ts", ".tsx", ".sql"}
SKIP = {"node_modules", ".git", "dist", "build", "vendor", "test", "tests",
        "__pycache__", "docs", "examples", "cookbook"}

def files(repo):
    for p in pathlib.Path(ROOT, repo).rglob("*"):
        if p.suffix in EXT and not any(part in SKIP for part in p.parts):
            yield p

corpus = {n: [(p, p.read_text(errors="replace")) for p in files(r)] for n, r in REPOS.items()}
for n, docs in corpus.items(): print(f"{n:16} {len(docs):6,} source files")
```

```
letta (archive)     716 source files
letta-code        1,897 source files
mem0                583 source files
graphiti            186 source files
langgraph           248 source files
cognee            2,069 source files
```

Note the `SKIP` set excludes `tests`, which understates letta-code badly — that project has test
files sitting next to sources rather than in a `tests/` tree, so a lot of its 1,897 files *are*
tests. Raw file counts are not a quality metric; they are here only to size the grep.

### The probes

Each probe looks for a **mechanism**, not a topic. The negative lookarounds matter; §"the two bugs"
below explains why they had to be added.

```python
import re

PROBES = {
 "P1 validity fields on the fact": r"(?<![A-Za-z_])valid_at(?![A-Za-z_])|(?<![A-Za-z_])invalid_at(?![A-Za-z_])",
 "P2 supersede, not delete":       r"(?<![A-Za-z_])superseded(_by)?(?![A-Za-z_])",
 "P3 write-decision vocabulary":   r'"event"\s*:\s*"(ADD|UPDATE|DELETE|NONE)"|ADD/UPDATE/DELETE',
 "P4 namespace as a key type":     r"namespace:\s*tuple\[str",
 "P5 delete cascades to derived":  r"(?<![A-Za-z_])delete_linked(?![A-Za-z_])|ondelete=|ON DELETE CASCADE",
}

for name, docs in corpus.items():
    row, first = [], {}
    for q, pat in PROBES.items():
        rx = re.compile(pat)
        hits = [(str(p).replace(ROOT + "/", ""), i + 1, ln.strip()[:70])
                for p, t in docs
                for i, ln in enumerate(t.splitlines()) if rx.search(ln)]
        row.append(len(hits))
        if hits: first[q] = hits[0]
    print(f"{name:16} " + " ".join(f"{c:4d}" for c in row))
    for q, (f, i, ln) in first.items():          # the part that decides anything
        print(f"{'':18}{q.split()[0]}  {f}:{i}\n{'':22}{ln}")
```

Real output:

```
system             P1   P2   P3   P4   P5
letta (archive)     0    0    0    0  165
letta-code          0    7    0    0    0
mem0                0    5   27    0   22
graphiti          238    2    0    0    0
langgraph           0    0    0   29    2
cognee             19   13    0    0   50
```

And the first-hit evidence, which is the part that actually matters:

```
graphiti   P1  graphiti/server/graph_service/zep_graphiti.py:134    valid_at=edge.valid_at,
cognee     P1  cognee/cognee/modules/migration/cogx.py:113          Temporal validity (valid_at/invalid_at) is always carried…
cognee     P2  cognee/…/tasks/graph/resolve_temporal_contradictions.py:7   …tags the older ones as superseded.
mem0       P3  mem0/mem0-ts/src/oss/src/prompts/index.ts:138        "event" : "NONE"
mem0       P5  mem0/mem0-ts/src/client/mem0.types.ts:52             Off by default. Serialized as `delete_linked`.
langgraph  P4  langgraph/libs/checkpoint-postgres/…/store/postgres/base.py:1278  def _namespace_to_text(namespace: tuple[str, ...])
langgraph  P5  langgraph/libs/checkpoint-postgres/…/store/postgres/base.py:113   FOREIGN KEY (prefix, key) REFERENCES store(prefix, key) ON DELETE CASCADE
letta (a)  P5  letta-archive/alembic/versions/bff040379479_add_block_history_tables.py:47  ondelete="CASCADE"
```

Read the matrix as a **specialisation map**, not a scoreboard. Graphiti's 238 P1 hits and zero P4
hits are not a weakness; they say it is a temporal-fact engine, and tenancy is your problem.
LangGraph's 29 P4 hits and zero P1 hits say it is a namespaced store primitive with no opinion about
truth over time. A system scoring on every probe would be suspicious, not excellent.

### The two bugs the audit hit

**Bug 1 — `valid_to` matches inside `invalid_tool_message`.** My first pass used
`invalid_at|valid_to|valid_until|invalidate_|supersed|\bNOOP\b`, and LangGraph's top "conflict
policy" file came back as `libs/prebuilt/langgraph/prebuilt/tool_node.py` with six hits. There is no
conflict policy in a tool-dispatch node. The substring:

```python
>>> import re
>>> pat = r"invalid_at|valid_to|valid_until|invalidate_|supersed|\bNOOP\b"
>>> for s in ["invalid_tool_message", "INVALID_TOOL_NAME_ERROR_TEMPLATE",
...           "edge.valid_to", "superseded_by"]:
...     m = re.search(pat, s, re.I); print(f"{s:36} -> {m.group(0) if m else None}")
invalid_tool_message                 -> valid_to
INVALID_TOOL_NAME_ERROR_TEMPLATE     -> VALID_TO
edge.valid_to                        -> valid_to
superseded_by                        -> supersed
```

`in|valid_to|ol`. The `\b` on `NOOP` was there; the identifier-ish tokens had none, and
`\b` would not have helped anyway — `_` is a word character, so `\bvalid_to\b` still matches inside
`invalid_tool`. The fix is an explicit non-identifier lookaround, `(?<![A-Za-z_])…(?![A-Za-z_])`,
which is what §"the probes" uses.

This is the same class of bug as chapter 04.2's gate regex and it has the same lesson: **a
substring match on an identifier is not a match on a symbol.** It produced a false positive that
flattered a system on a property it does not have, in the direction that would have made me choose
wrong.

**Bug 2 — grep finds prose, not mechanism.** With the fixed pattern, mem0's five P2 hits are all
*docstrings and type comments*:

```
mem0/mem0/client/main.py:389   superseded (the v3 ``linked_memory_ids`` chain), transitively.
mem0/mem0-ts/src/client/mem0.types.ts:50  When `true`, also delete the older memories this one superseded…
```

The word is in the file. Whether the behaviour is in the file is a different question — and here it
turned out to be more interesting than the count (§10.3). Meanwhile letta-code's seven P2 hits
include `"perm-stream-superseded"`, a websocket auth-lifecycle test fixture with nothing to do with
memory.

**Discipline that follows:** the probe ranks files to read; it never decides. Budget one hour per
candidate to *read the top hit of each probe*. An hour of reading source told me more about these
five systems than their combined documentation.

---

## 10.2 MemGPT / Letta — memory as an operating system, which became a git repo

**Paper:** *MemGPT: Towards LLMs as Operating Systems*
([arXiv:2310.08560](https://arxiv.org/abs/2310.08560)) · **Repos:** `letta-ai/letta` (12 files,
Apache-2.0, 24,698★, last release `0.16.8` on 2026-05-14), `letta-ai/letta` @ `archive` (the retired
Python V1 server, 716 source files), `letta-ai/letta-code` (active, TypeScript, Apache-2.0, installed
as `npm i -g @letta-ai/letta-code`).

### The original idea

Treat the context window as physical memory and external storage as virtual memory, with the LLM as
the operating system paging between them:

```
main context: system instructions │ memory blocks (agent-editable, char-capped:
              persona, human)     │ recent messages
                   │ tool calls
     ┌─────────────┴──────────────┬────────────────────────────┐
  recall storage                archival storage
  full message history          embedding-based store
  conversation_search           archival_memory_insert/_search
```

In-context blocks are edited with `core_memory_append` / `core_memory_replace`; replying is itself a
tool call (`send_message`); *heartbeats* let the agent chain calls without user input. Chapter 06.4
has the self-editing discussion.

### What the audit found instead

The interesting artefact in the archive branch is not the tool list, it is a migration:
`letta-archive/alembic/versions/bff040379479_add_block_history_tables.py`. Memory blocks are
versioned, append-only, and attributed:

```python
op.create_table("block_history",
    sa.Column("label", sa.String(), nullable=False),
    sa.Column("value", sa.Text(), nullable=False),
    sa.Column("limit", sa.BigInteger(), nullable=False),
    sa.Column("actor_type", sa.String(), nullable=True),   # who edited
    sa.Column("actor_id", sa.String(), nullable=True),
    sa.Column("block_id", sa.String(), nullable=False),
    sa.Column("sequence_number", sa.Integer(), nullable=False),
    sa.Column("organization_id", sa.String(), nullable=False),
    sa.ForeignKeyConstraint(["block_id"], ["block.id"], ondelete="CASCADE"), …)
op.create_index("ix_block_history_block_id_sequence", "block_history",
                ["block_id", "sequence_number"], unique=True)
op.add_column("block", sa.Column("current_history_entry_id", sa.String()))
op.add_column("block", sa.Column("version", sa.Integer(), server_default="1", nullable=False))
```

That is chapter 04's supersession pattern and chapter 09's audit requirement, in one table: a
unique `(block_id, sequence_number)` so history cannot fork, a pointer to the current entry, an
`actor_id` so you know whether the agent or a human made the edit, and a cascade so history dies with
its block. If you build self-editing in-context memory, copy this schema. `core_memory_replace`
without a history table is an unauditable overwrite.

### The evolution that matters: sleep-time compute, then MemFS

First split: the original design bundled conversation, tools, and memory management into one agent.
Letta separated them — a **primary agent** that talks and can search memory but holds no edit tools,
and a **sleep-time agent** that holds the edit tools and rewrites shared blocks in the background
(chapter 06.3 has the arithmetic; the underlying result is
[arXiv:2504.13171](https://arxiv.org/abs/2504.13171): ~5× less test-time compute for equal accuracy,
up to +13%/+18% accuracy when the offline work is scaled, 2.5× lower average cost per query when
amortised).

Second split, and the one nobody's architecture diagram shows yet: **core memory is now a
git-backed filesystem.** In `letta-code`, memory is "MemFS" — a per-agent repository with a working
tree, commits, and a remote:

```
letta-code/src/types/protocol_v2.ts
  ReadMemoryFileCommand    { agent_id, path, encoding: "utf8"|"base64" }
  WriteMemoryFileCommand   { agent_id, path, content, encoding, commit_message? }
     "Write a file into the agent's MemFS and commit. Path is relative to the
      memory root and is rejected if it escapes the root."
  DeleteMemoryFileCommand  { agent_id, path, commit_message? }
     "Idempotent: if the file is already absent, the handler returns success
      without producing a commit."
```

Details worth stealing, all verified in source:

- **Every memory write is a commit.** `src/tools/memory-tool-assets.ts` distinguishes the remote case
  ("the harness pushes clean committed memory changes after the turn for remote MemFS agents") from
  local ("no Letta remote; memory changes are committed locally"). Memory has a `git log`.
- **Path escape is rejected at the protocol layer**, not asked for in a prompt — chapter 09's rule
  about controls living in code.
- **The format has a version with different budget semantics.**
  `src/agent/system-prompt-size.ts`: "MemFS v1 counts Markdown files recursively under `system/`.
  MemFS v2 counts root-level Markdown files. A root `MEMORY.md` selects v2 only for API-backed
  memory; local MemFS remains v1." The same file exports `estimateSystemPromptSize()`, returning a
  **per-file token breakdown** of core memory — a token budget instrument for the always-in-context
  tier (chapter 06.1's `render_rules` budget, productised).
- **Sub-agents can be forced stateless.** `src/headless-memfs-policy.ts` is twenty lines and encodes
  one decision: a newly created sub-agent in a sub-agent role must not enable, clone, or sync MemFS.
  Chapter 09's blast-radius containment as a pure function.
- Blocks did not die: `memory_blocks` with labels (`persona`, `project`, `onboarding`) and shared
  `block_ids` are still the agent-creation API (`src/agent/create-agent-request.ts`).

**The arc is the lesson.** The system that introduced the OS metaphor for memory — paging, virtual
context, interrupts — converged on *markdown files in a git repository with a token budget*. That is
the coding-agent pattern of §10.7, arrived at from the opposite direction. Two independent lineages,
same answer: for the always-in-context tier, files with history beat a database.

```
Letta today
┌──────────────────────────────────────────────────────────┐
│ context: memory blocks (labelled, capped, versioned in   │
│          block_history) + MemFS markdown (root *.md)     │
└───────┬──────────────────────────────────────────────────┘
        │ memory / memory_apply_patch tools → git commit → push after turn
┌───────▼──────────┐  ┌──────────────────┐  ┌──────────────────────────┐
│ MemFS git repo   │  │ recall storage   │  │ sleep-time agent         │
│ per agent, v1/v2 │  │ message history  │  │ holds the edit tools     │
└──────────────────┘  └──────────────────┘  └──────────────────────────┘
```

**Use it when:** you want stateful agents with durable, auditable, human-readable memory and are
willing to run a stateful agent server. **Be careful about:** the repo/language split — pin what you
integrate against and read the archive branch only for archaeology; and self-editing memory is still
hard to audit at the *semantic* level even with a git log, so keep the agent-proposes /
system-validates hybrid from 06.4 in regulated domains.

---

## 10.3 Mem0 — from reconcile-in-place to append-and-link

**Paper:** *Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory*
([arXiv:2504.19413](https://arxiv.org/abs/2504.19413)) · **Repo:** `mem0ai/mem0`, Apache-2.0,
65,135★, pushed 2026-09-11.

### What the paper describes

A two-stage pipeline that is exactly chapter 04's write path: **extraction** (an LLM pulls salient
facts from the new exchange plus context) then **update** (a decision step issues operations against
existing memories). Published results on LOCOMO: **+26% relative** on the LLM-as-a-Judge metric over
OpenAI's memory, the graph variant ~**+2%** above the base configuration, **91% lower p95 latency**
and **>90% token savings** versus passing the full conversation. That is the triple from rule 5 —
accuracy, tokens, latency — reported together, which is why chapter 08 holds them up as the standard.

### Correction to a claim this chapter used to make

The reconciliation vocabulary is **ADD / UPDATE / DELETE / NONE**, not `NOOP`. From
`mem0/mem0/configs/prompts.py:182-185`:

```
- ADD: Add it to the memory as a new element
- UPDATE: Update an existing memory element
- DELETE: Delete an existing memory element
- NONE: Make no change (if the fact is already present or irrelevant)
```

A trivial-looking fix with a non-trivial consequence: if you copy the vocabulary into your own
prompt and your parser expects `NOOP`, every no-change decision falls through to your default
branch. Chapter 04.2 made the same class of mistake with a regex.

### The architectural change the paper does not describe

The default `add()` path is no longer the reconcile-in-place pipeline. From
`mem0/mem0/memory/main.py:916` onward it is a **V3 phased batch pipeline** whose only operation is
ADD:

```
# === V3 PHASED BATCH PIPELINE ===
Phase 0  context gathering      db.get_last_messages(session_scope, limit=10)
Phase 1  existing retrieval     vector_store.search(..., top_k=10, filters={user_id|agent_id|run_id})
         uuid → integer mapping "# Map UUIDs to integers (anti-hallucination)"
Phase 2  single LLM call        ADDITIVE_EXTRACTION_PROMPT  (+ AGENT_CONTEXT_SUFFIX if agent-scoped)
Phase 3  batch embed            embed_batch(mem_texts, "add"), per-item fallback on failure
```

The prompt at `prompts.py:464` is titled *"V3 Additive Extraction Prompt (ADD-only with memory
linking)"* and says so plainly: *"Your sole operation is ADD."* Instead of mutating an existing row,
the new memory carries `linked_memory_ids` pointing at the memories it relates to or supersedes.

Three things in there are worth stealing outright:

1. **UUID → small-integer mapping before the LLM sees existing memories.** The model is shown ids
   `"0"`, `"1"`, `"2"`; the mapping back to UUIDs happens in code. Models hallucinate UUIDs; they do
   not hallucinate `"3"` into existence as often, and when they do, the lookup fails loudly instead
   of writing a link to a memory that does not exist. Chapter 04's extraction snippets should have
   done this.
2. **Append-and-link instead of update-in-place.** This is chapter 04/05's argument — never destroy
   the prior assertion, record the new one and mark the relationship — adopted by the most widely
   deployed memory layer in the ecosystem, after the paper was published. If your own design still
   does in-place `UPDATE`, this is the strongest available evidence that you will change your mind.
3. **A deliberate loud failure**, with the reasoning left in the source
   (`main.py`, Phase 2 error branch):

   > *"Re-raise so callers can implement provider fallback / retry. The original silent `return []`
   > made upstream callers unable to distinguish 'LLM unavailable' (429/5xx/timeout) from 'LLM
   > extracted no facts' — both surfaced as an empty list."*

   That is chapter 08's silent-failure trap named and fixed in a shipping library. An extractor that
   returns `[]` on a rate limit produces an eval run that looks like "nothing was memorable today".

### The default that will bite you

`mem0/mem0/client/main.py:382`:

```python
def delete(self, memory_id: str, delete_linked: bool = False) -> Dict[str, Any]:
    """Delete a specific memory by ID.

    delete_linked: When True, also delete the older memories this one
        superseded (the v3 ``linked_memory_ids`` chain), transitively.
        This is the delete-side counterpart of ``latest_only`` — it
        stops a superseded memory from resurfacing after you delete the
        current one. Defaults to False (only the given memory is deleted).
    """
```

Read it against chapter 09.6. Append-and-link means a corrected fact does not remove the wrong one;
it links to it. So `delete(current_memory_id)` with the default `delete_linked=False` deletes the
*current* assertion and leaves the superseded chain in place — where the read side will happily
return it, because `latest_only` is also off by default (`mem0/mem0/client/types.py:61`:
`latest_only: Optional[bool] = Field(default=None, …)`).

Concretely: user says "I'm vegetarian", later "I eat fish now", then asks you to delete the second.
Default behaviour restores the first as the only surviving answer. That is defensible as a design
(the history is real) and indefensible as a *default* if your deletion path is wired to a GDPR
erasure request. **If you use Mem0 for anything with an erasure obligation, pass
`delete_linked=True` and add the chapter 08.5 `must_not_contain` assertion that proves it.**

Both parameters existing at all is a point in Mem0's favour — most stores give you no cascade
vocabulary whatsoever. The criticism is precisely and only about which way the default points.

```
Mem0 write path (V3 default)
new turn ──► last 10 msgs ─┐
                           ├─► one LLM call (ADD-only) ─► facts + linked_memory_ids
top-10 existing (uuid→int)─┘                                     │
                                                    batch embed ─┴─► vector store
read: search(filters=user/agent/run, latest_only=?)   delete: delete(id, delete_linked=?)
```

**Use it when:** conversational workload, you want production memory in days, managed dependency is
acceptable. **Be careful about:** extraction is lossy by construction (keep the episode log if users
ask verbatim-recall questions), LLM-in-the-write-path means write cost scales with conversation
volume (batch at session end), and the two defaults above.

---

## 10.4 Zep / Graphiti — bi-temporal, and it is really in the schema

**Paper:** *Zep: A Temporal Knowledge Graph Architecture for Agent Memory*
([arXiv:2501.13956](https://arxiv.org/abs/2501.13956)) · **Repo:** `getzep/graphiti`, Apache-2.0,
30,812★, `v0.30.2` released 2026-09-08.

Covered in depth in chapter 05; this is the comparison entry plus the audit evidence.

Published numbers: **DMR 94.8% vs MemGPT's 93.4%**, and on LongMemEval accuracy improvements of up
to **18.5%** with **90% lower response latency**. Chapter 08.8 is the required reading before you
quote the first of those: a 1.4pp gap on a few hundred items is inside the noise band unless the
comparison was paired. The LongMemEval pair — big accuracy delta *and* a latency reduction — is the
claim that carries information.

### The audit evidence

Probe P1 returned **238 hits**, more than all other systems combined, and the definition is exactly
where it should be — on the edge that carries the fact
(`graphiti/graphiti_core/edges.py:271-281`):

```python
class EntityEdge(Edge):
    name: str            # relation name
    fact: str            # fact representing the edge and the nodes it connects
    fact_embedding: list[float] | None
    episodes: list[str]  # episode ids that reference this edge
    expired_at: datetime | None   # "datetime of when the node was invalidated"
    valid_at:   datetime | None   # "datetime of when the fact became true"
    invalid_at: datetime | None   # "datetime of when the fact stopped being true"
    reference_time: datetime | None  # reference timestamp from the producing episode
    attributes: dict[str, Any]
```

Plus `created_at` from the base `Edge`. Four timestamps, two clocks: `created_at`/`expired_at` is
**when the system believed it** (transaction time); `valid_at`/`invalid_at` is **when it was true in
the world** (valid time). That is chapter 05.2's table, in a Pydantic model, with `episodes` giving
you the bidirectional fact↔episode index for citation and incremental update. `reference_time`
is the extra one worth noting — it pins the fact to the episode's own notion of "now", which is how
you resolve "last Tuesday" correctly when the episode is ingested three days late.

### Architecture

```
episodes  (raw, non-lossy, timestamped)
    │ extract entities + edges, resolve duplicates, detect contradictions
    ▼
semantic entities & facts  (edges carry valid_at / invalid_at / created_at / expired_at)
    │ label propagation
    ▼
communities  (clusters + generated summaries)

retrieval: cosine ∪ BM25 ∪ breadth-first traversal → fuse (RRF/MMR/cross-encoder)
           → context constructor renders facts WITH their validity ranges
```

**What to steal:** bi-temporal edges (the single most transferable idea in the field); the
episode↔fact indices; rendering validity ranges into the prompt (nearly free, disproportionate
quality gain); and the observation that agent memory is millions of small, mostly-cold graphs — a
different systems problem from one large graph (chapter 07.5).

**Be careful about:** write-path cost — entity extraction plus resolution plus contradiction
detection is several LLM calls per episode (05.8 has the mitigations); operational weight if you
self-host a graph database; and the divergence between the open-source engine and the hosted service.

---

## 10.5 LangGraph Store + LangMem — namespaces, TTL, and the one real cascade

**Repos:** `langchain-ai/langgraph` (MIT, 41,472★), `langchain-ai/langmem` (MIT, 1,660★).

### The core idea

Two scopes, cleanly separated, and the separation *is* the lesson:

- **Thread state (short-term)** — checkpointed state of one graph run. Durable, resumable, scoped to
  the thread.
- **Store (long-term)** — namespaced key-value with optional vector indexing, shared across threads.

The store's type surface (`libs/checkpoint/langgraph/store/base/__init__.py`) is more opinionated
than "key-value with vectors" suggests:

```python
class Item:
    __slots__ = ("value", "key", "namespace", "created_at", "updated_at")
    namespace: tuple[str, ...]     # hierarchical path, e.g. ("user_123", "preferences")

GetOp(namespace=("users", "profiles"), key="user123", refresh_ttl=True)
SearchOp(namespace_prefix=("documents",), …)
ListNamespacesOp(match_conditions=…, max_depth=…, limit=…)
# NamespacePath supports wildcards: ("documents", "*")
# MatchType is "prefix" | "suffix"
```

```python
store.put(("user_123", "preferences"), key="diet", value={"text": "vegetarian"})
results = store.search(("user_123", "preferences"), query="what do they eat", limit=5)
```

Three things fall out of the tuple namespace that a flat `user_id` column does not give you:
enumerable hierarchy (`ListNamespacesOp` with `max_depth`), prefix/suffix wildcard matching, and a
deletion boundary that is a *path*, not a predicate. Chapter 07.2 makes the case for
`(tenant, subject, category)` as a first-class key structure; this is the clearest implementation of
it available.

### The cascade that actually exists

Probe P5 found only two hits in LangGraph, and one of them is the single best line in the whole
audit (`libs/checkpoint-postgres/langgraph/store/postgres/base.py:113`):

```sql
CREATE TABLE IF NOT EXISTS store_vectors (
    prefix text NOT NULL,
    key text NOT NULL,
    field_name text NOT NULL,
    embedding vector(<dims>),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (prefix, key, field_name),
    FOREIGN KEY (prefix, key) REFERENCES store(prefix, key) ON DELETE CASCADE
);
```

Deleting the row deletes its vectors, enforced by the database rather than by remembering to call a
second function. Chapter 09.6 measured seven residue sites after a naive `DELETE` against a store
that had a vector table, a lexical index, derived memories and a cached context; this schema closes
the first of them *structurally*. It does not close derived memories or summaries — nothing here
knows what `derived_from` means — but a foreign key you cannot forget is worth more than a deletion
runbook you can.

TTL support (`refresh_ttl` on `GetOp` and `SearchOp`) is the other production-shaped detail: it makes
the chapter 04.7 decay tier a property of the store rather than a cron job you write.

### LangMem

LangMem layers memory *management* on the store: extraction utilities, prompt optimisation from
feedback, hot-path memory tools the agent can call mid-conversation, and a background memory manager
— the foreground/background split of chapter 06.3, with native integration into the store. At
1,660★ against LangGraph's 41,472 it is a much thinner community; treat it as a set of primitives to
read and adapt rather than a dependency to lean on.

**Use it when:** you are already on LangGraph, or you want a memory *primitive* rather than a memory
*opinion*. **Be careful about:** the store has no notion of truth over time (P1: zero hits) — the
whole of chapter 05 is yours to build.

---

## 10.6 Cognee — pipeline memory, and cardinality you must declare

**Repo:** `topoteretes/cognee`, Apache-2.0, 30,641★, `v1.5.4` released 2026-09-04, 2,069 source
files — the largest surface here.

Memory as an **ingestion pipeline** producing a hybrid graph + vector store. The `cognee/api/v1/`
directory is the fastest way to read the design, because the verbs are the architecture:

```
add   cognify  memify   recall  remember  search   forget  prune
datasets  permissions  ontologies  skills  proposals  export  visualize
```

`add` → `cognify` (build the graph) → `memify`/`remember` → `recall`/`search`, with `forget` and
`prune` as first-class inverse operations. Naming the inverse operations at the same level as the
write operations is a design choice most memory systems skip, and chapter 09 is a long argument for
why that is the wrong thing to skip.

### The most honest file in the audit

`cognee/cognee/tasks/graph/resolve_temporal_contradictions.py` is an opt-in task, and its docstring
is the best statement of chapter 05's hardest problem I have found in production code:

> *"Runs after the graph has been written. For the relationships the caller declares **functional** —
> single-valued, like a company's current CEO — it inspects the region of the graph this ingestion
> touched and, wherever a subject ended up holding more than one target for such a relationship,
> keeps the most recent assertion and tags the older ones as superseded.*
>
> *Nothing is deleted: a superseded edge stays in the graph with its provenance, tagged
> (`superseded`, `superseded_by`, `supersession_reason`) so the current fact can be told apart from
> the history it replaced.*
>
> *The task is a no-op unless `functional_relationships` is given. Cognee's LLM-extracted
> relationships are mostly many-valued (`knows`, `mentions`) and carry no cardinality metadata, so
> which ones hold a single target cannot be inferred — only declared."*

Unpack the last sentence, because it is the whole difficulty of automated contradiction detection.
"Priya works at Acme" and "Priya works at Globex" is a contradiction only if `works_at` is
single-valued. "Priya knows Sam" and "Priya knows Lee" is not. **An extractor that emits relation
names without cardinality cannot tell those cases apart**, so any system that claims automatic
contradiction resolution over free-text-extracted relations is either guessing or has a hidden list
of functional predicates. Cognee makes the list explicit and defaults to doing nothing. That is the
correct default, and it is the opposite of impressive-sounding.

Two more details verified in that file: supersession is computed against **the stored
neighbourhood**, not the in-flight batch, so today's fact supersedes last month's; and it relies on
deterministic entity ids (`Entity:<name>`) so a re-mentioned subject keeps the id its earlier facts
hang off. That second one is chapter 05.6's entity-resolution problem solved by fiat — cheap,
robust, and wrong the moment two different people share a name. Know which trade you are accepting.

**What to steal:** memory as an ETL pipeline with named, individually re-runnable stages (chapter
04's "separate the five decisions", applied at system level); `forget`/`prune` as first-class verbs;
multiple retrieval modes exposed rather than hidden behind one `search()`; and declared cardinality
for contradiction resolution.

**Use it when:** you are ingesting heterogeneous sources (documents + conversations + structured
data) and want a batteries-included pipeline. **Be careful about:** surface area — 2,069 source files
and ~30 API verbs is a lot of system to operate, and the temporal correctness you may assume from
"knowledge graph" is opt-in and requires you to enumerate your functional predicates.

---

## 10.7 Coding agents — files win, and here is what files cost

Claude Code, Cursor, the Devin-likes and now Letta itself converged on something the
memory-framework world underweights: **for procedural and project knowledge, files beat databases.**
Chapter 06.1 has the table; this section prices it.

The stack, roughly:

```
persistent, always in context:
  CLAUDE.md / AGENTS.md / rules files    ← conventions, human-editable, in git
per-task, written by the agent:
  PLAN.md, PROGRESS.md, NOTES.md         ← externalised working memory
per-session, automatic:
  compaction of the transcript           ← summarise, reinitialise
  tool-result clearing                   ← lightest-touch compaction
per-subtask:
  sub-agents with isolated windows       ← return condensed summaries only
```

Anthropic's own account of extended-horizon work names exactly three techniques — **compaction**,
**structured note-taking** (the agent writes notes persisted outside the context window and reads
them back later), and **sub-agent architectures** where each sub-agent may burn tens of thousands of
tokens but returns "often 1,000–2,000 tokens"
([Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents),
2025-09-29). Two specifics from that post worth carrying into your own design: tool-result clearing
is called out as "one of the safest lightest touch forms of compaction", and the motivation is
**context rot** — recall degrades as the window fills
([research.trychroma.com/context-rot](https://research.trychroma.com/context-rot)), which is why a
bigger window is not a substitute for curation.

The platform primitive matching this is the **memory tool**
([docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)), and its shape is
instructive:

```json
{"type": "memory_20250818", "name": "memory"}
```

- **Client-side.** The model only *requests* operations — `view`, `create`, `str_replace`, `insert`,
  `delete`, `rename` — against a `/memories` prefix that your handler maps onto real storage. "Memory
  lives entirely in your application."
- **Path traversal protection is your job**, and the docs say so twice: reject any path outside
  `/memories`, and reject a `rename` whose `old_path` is the memory root.
- One sharp edge: in `str_replace`, `new_str` is optional, and **omitting it deletes `old_str`**. A
  handler that treats a missing field as an empty replacement silently makes deletion the default for
  malformed input.
- Available on Claude 4 and later; SDK helpers exist (`BetaAbstractMemoryTool`, `betaMemoryTool`,
  `BetaLocalFilesystemMemoryTool`) so the storage backend is a subclass, not a fork.

Note what that design deliberately does not do: no embeddings, no ranking, no extraction. It is
just-in-time retrieval over a directory the agent navigates itself — file names and directory
structure *are* the index.

### What files cost, measured

Files are not free; they are the one memory tier that costs tokens on **every single turn**. Measured
with `tiktoken` (`cl100k_base`) against the real instruction files in the repos cloned in §10.1,
priced at $3.00 per million input tokens:

```python
import tiktoken, pathlib
enc = tiktoken.get_encoding("cl100k_base")
for f in ["letta-code/AGENTS.md", "cognee/CLAUDE.md", "mem0/AGENTS.md",
          "graphiti/CLAUDE.md", "langgraph/AGENTS.md", "letta/AGENTS.md"]:
    t = pathlib.Path("/tmp/ch10", f).read_text()
    n = len(enc.encode(t))
    print(f"{f:24} {t.count(chr(10)):6,} {n:8,} {n*3/1e6:9.5f} {n*3/1e6*1000:11.2f}")
```

```
file                      lines   tokens    $/turn  $/1k turns
letta-code/AGENTS.md        957   10,992   0.03298       32.98
cognee/CLAUDE.md            866   12,226   0.03668       36.68
mem0/AGENTS.md              175    3,122   0.00937        9.37
graphiti/CLAUDE.md          181    1,739   0.00522        5.22
langgraph/AGENTS.md          65      478   0.00143        1.43
letta/AGENTS.md              48      735   0.00220        2.21
```

Two orders of magnitude between the leanest and the fattest. Chapter 06.1 suggested ~1,800 tokens as
the point where a rule file needs consolidation; the two largest files here are **6× and 7× that**,
and they sit in front of every request. A 40-turn session carrying `letta-code/AGENTS.md`
uncached:

```
uncached :    439,680 token-equivalents  $1.32
cached   :     56,609 token-equivalents  $0.17     (1.25× write once + 0.10× read ×39)
ratio    : 7.8x cheaper with a warm cache
break-even turn count where caching wins: 2
```

**So the honest version of "files win" is: files win *and* they are the tier you must budget,
instrument, and cache.** Recall 1.0 is worth paying for (chapter 06.0: at recall 0.9, twenty tasks
give an 87.8% chance of missing a must-apply rule at least once), but 12,000 tokens of conventions
on every turn is a real line item, and it is also 12,000 tokens of context rot pressure. Letta
shipping `estimateSystemPromptSize()` with a per-file breakdown (§10.2) is the right response:
measure the tier, per file, and make the biggest file justify itself.

**What to steal:** ask whether a markdown file solves 80% of the problem before building a vector
pipeline; tool-result clearing before full compaction; sub-agent isolation for anything
search-shaped; a `dead_ends` section in whatever carries state across compactions; and a token
budget report for your always-in-context tier, in CI.

---

## 10.8 Consumer assistant memory — and the vendor's own erasure warning

Consumer assistants have converged on a product shape worth cataloguing, because it encodes lessons
about *users* rather than systems. ChatGPT's memory documentation
([Memory FAQ](https://help.openai.com/en/articles/8590148-memory-faq), retrieved 2026-09-12) is the
most explicit public description, and it has moved:

- **From discrete memories to a synthesis.** The current system is a **memory summary**: "a
  continually updated synthesis of context from your past chats", shown with a "last updated"
  timestamp, editable by typing what you want changed or by highlighting text to correct it. The
  legacy **saved memories** system — discrete items the model wrote and the user could delete
  individually — is still selectable in Settings.
- **Why they moved**, in their words: the previous system "often became stale and relied on users to
  manually manage updates. Memories could also contradict one another, such as 'I'm training for a
  marathon' and 'I sprained my ankle', which made personalization less accurate." That is chapter
  04's reconciliation problem and chapter 05's validity problem, stated as a product regression, with
  the canonical example.
- **Provenance surfaced to the user.** A book icon under a response lists the sources that
  personalised it — custom instructions, past chats, files, memories — and tapping a memory explains
  *why* it was used. Chapter 09.5's "show your work" control, shipped. The docs also concede it "may
  not show every factor".
- **Scoping.** Temporary Chats "do not use existing memories or create new memories".
  **Project-only memory** confines reference to one project in both directions: chats inside can
  reference each other, chats outside cannot reach in. That is chapter 07.2's namespace boundary as a
  user-facing feature.
- **Conservative defaults where it counts.** In ChatGPT for Healthcare and Enterprise with Regulated
  Workspace, improved memory is **disabled by default**, must be granted per role, and is "not
  covered under your BAA" with explicit instruction not to enter PHI.

And then the sentence every engineer building memory should have pinned above their desk:

> *"To fully delete something ChatGPT may know about you, you'll need to delete every source where it
> appears, including past chats, archived chats, files, the memory summary, and disconnect any
> connected apps that may contain that information."*

That is chapter 09.6's erasure cascade, admitted by a vendor in their own help centre, enumerating
five residue sites. Chapter 09 measured seven sites surviving a naive `DELETE` in a toy SQLite store;
the production version of the same problem is worse, because one of the sites is a *synthesis* —
deleting the source text does not remove the inference drawn from it until the synthesis is
regenerated. The docs also note "we may retain a log of deleted Saved Memories for up to 30 days for
safety and debugging purposes", and that turning memory off does not delete past chats, so turning it
back on may re-derive memories from history.

The product table stakes, then:

| Pattern | Why it exists | Chapter |
|---|---|---|
| Explicit + implicit channels | trust, not capability | 09.5 |
| Management UI: list, view source, edit, delete, disable | best privacy control *and* best quality feedback loop | 09.5 |
| Per-conversation opt-out | users need an unrecorded mode | 09.5 |
| Visible write moments ("I've saved that") | silent formation is the fastest route to a creepiness complaint | 09.2 |
| Scoped memory (project-only) | blast radius | 07.2 |
| Conservative defaults in regulated tiers | the BAA does not cover it | 09.5 |
| Synthesis over item list | items go stale and contradict each other | 04.4, 05.2 |

Skipping the management UI and the visible write moment is what turns a memory feature into a trust
problem. And do not infer engineering quality from benchmark rows here: assistant memory optimises
latency, cost, privacy conservatism and domain breadth simultaneously, which is a different objective
from a benchmark-maximising configuration (chapter 08.3).

---

## 10.9 Cross-system comparison, honestly

Published numbers, with all three units where the source reports them. **These are not comparable to
each other** — different actor models, judges, retrieval configs and subsets; chapter 08.3 explains
why, and §10.10 step 3 is how you get numbers that *are* comparable.

| System | Accuracy claim | Tokens | Latency | Source |
|---|---|---|---|---|
| Mem0 | +26% relative (LLM-as-Judge) vs OpenAI memory on LOCOMO; graph variant ~+2% over base | >90% saved vs full-context | 91% lower p95 vs full-context | [arXiv:2504.19413](https://arxiv.org/abs/2504.19413) |
| Zep / Graphiti | DMR 94.8% vs MemGPT 93.4%; LongMemEval up to +18.5% | not reported | 90% lower response latency | [arXiv:2501.13956](https://arxiv.org/abs/2501.13956) |
| Letta sleep-time | +13% (Stateful GSM-Symbolic), +18% (Stateful AIME) | ~5× less test-time compute for equal accuracy; 2.5× lower avg cost/query amortised | shifted off the critical path | [arXiv:2504.13171](https://arxiv.org/abs/2504.13171) |
| LangGraph Store | none published | none published | none published | — |
| Cognee | none published in-repo | — | — | — |

Two readings of that table. First, the honest one: **three of five publish no accuracy/token/latency
triple at all**, which is not a scandal — LangGraph Store is a primitive, not a memory strategy, and
a primitive has nothing to benchmark. Second: the 1.4pp DMR gap is the weakest number in the table
and the most quoted, while the "90% lower latency alongside higher accuracy" is the strongest and the
least quoted, because latency wins are unglamorous.

What the audit adds that no benchmark shows:

| Property | Letta | Mem0 | Graphiti | LangGraph Store | Cognee |
|---|---|---|---|---|---|
| Validity fields on the fact (P1) | no | no | **yes** (4 timestamps) | no | partial (carried) |
| Supersede rather than destroy (P2) | yes (`block_history`) | yes (`linked_memory_ids`) | yes (invalidate) | no | yes (opt-in, declared) |
| Explicit write-decision vocabulary (P3) | — | **yes** (ADD/UPDATE/DELETE/NONE) | — | — | — |
| Namespace as a key type (P4) | org + agent | user/agent/run filters | group_id | **yes** (tuple + wildcards) | datasets + permissions |
| Delete cascades to derived (P5) | FK cascade on history | opt-in `delete_linked` | — | **FK cascade to vectors** | `forget`/`prune` verbs |
| Always-in-context tier | **git-backed MemFS** | — | — | — | — |
| Human-readable without a tool | **yes** (markdown) | no | no | no | no |

No row wins everything, which is the finding. Pick by the row that maps to your failure mode: CRM
correctness → P1; erasure obligations → P5; multi-tenant SaaS → P4; coding/ops agents → the last two
rows.

---

## 10.10 Build vs. buy

**Buy (managed memory layer) when:** memory is not your differentiator; the workload is conversational
and mainstream; you need to ship in weeks; your compliance posture tolerates a third party holding
personal data.

**Self-host open source when:** data residency or regulation rules out SaaS; you need to modify
write-path behaviour (custom categories, custom reconciliation); per-call pricing at your volume
exceeds the cost of running it.

**Build when:** memory *is* the product; your domain has structure that generic extraction destroys
(medical, legal, financial); you need guarantees — deletion, audit, isolation — that no vendor will
put in a contract; your scale makes per-call pricing absurd.

**The pragmatic path, and the one I would defend in a design review:**

1. **Weeks 1–2.** Session summarisation + a facts table in Postgres. Chapter 04's gate cascade,
   chapter 05's two validity columns. No framework.
2. **Weeks 3–6.** Hybrid retrieval (chapter 03) and an explicit write-decision vocabulary — use
   **ADD/UPDATE/DELETE/NONE**, verbatim, so your prompts are comparable to the literature.
3. **Weeks 7–8.** Your own eval harness (chapter 08): 100+ cases weighted toward knowledge update
   and abstention, bootstrap CIs, accuracy/tokens/latency on every run.
4. **Only then** evaluate frameworks, using step 3's harness and the fair-comparison rules of 08.9.

Teams that adopt a memory framework at step 0 usually cannot answer "is it working?" at step 4,
because they never built the measurement — and a memory layer is a deep dependency with a
five-figure migration (§10.0, arithmetic 3). Weight governance and interface stability heavily: in
the eighteen months before this snapshot, one of the five systems here moved its entire source tree
to a different repository in a different language, and another changed its default write semantics
from update-in-place to append-only.

Before you sign, run this list against the candidate:

- [ ] The five probes of §10.1, plus one hour reading the files they point at.
- [ ] `delete()` semantics, tested: write a fact, correct it, delete the correction, then search.
      Assert the superseded value does **not** come back (chapter 08.5 `must_not_contain`).
- [ ] Tenancy: is it a filter or a boundary? Try to read another tenant's row with a forged id
      (chapter 07.2).
- [ ] Exported data format. Can you reconstruct your store without the vendor's code?
- [ ] Extraction cost per stored memory, measured on your own traffic, times your projected volume.
- [ ] Licence *and* repo activity on the artefact you will actually import, not the one the paper
      names.

---

## 10.11 Reading the code

Read one codebase: **Graphiti** — 186 source files, legible bi-temporal logic
(`graphiti_core/edges.py`, then `utils/maintenance/edge_operations.py`). Two: add **Mem0's**
`configs/prompts.py` and `memory/main.py:916`. Three: add
`cognee/tasks/graph/resolve_temporal_contradictions.py`, the best-written statement of why automatic
contradiction detection is hard.

The five questions, mapped to probes and chapters:

| Question | Probe | Read | Fails as |
|---|---|---|---|
| Where is the write path? | — | extraction prompt + reconciliation decision | ch 04 |
| What is the conflict policy? | P1, P2 | the fact schema | LongMemEval update category |
| How is tenancy enforced? | P4 | filter you must pass vs. boundary the DB enforces | ch 07.2 cross-tenant leak |
| What runs in the background? | — | job definitions; nothing ⇒ quality plateaus | ch 06.3 |
| What is the deletion path? | P5 | then **test** it (§10.3) | ch 09.6 residue |

An hour with those tells you more than a week of documentation. Both bugs in §10.1 are why the hour
includes *reading*, not just grepping.

---

## 10.12 Failure modes when adopting someone else's memory

| Symptom | Root cause | Fix |
|---|---|---|
| "We picked the popular one and it can't answer temporal questions" | selected on stars; P1 property absent | §10.1 probes before shortlisting |
| Integration guide doesn't match the repo | paper describes retired code | §10.2 pin the artefact you import; check for archive branches |
| Deleted memories reappear in answers | supersession chain left behind by default | §10.3 `delete_linked=True`, `latest_only`; ch 09.6 cascade |
| Every no-change decision hits the default branch | copied `NOOP` instead of `NONE` | §10.3 verify vocabulary against source |
| Eval run says "no facts were memorable" | extractor swallowed 429/5xx as `[]` | §10.3 raise on LLM failure; ch 08 silent-failure trap |
| Vector index still returns a deleted row | no FK from vectors to rows | §10.5 `ON DELETE CASCADE`; ch 09.6 |
| "Knowledge graph" system still contradicts itself | functional predicates never declared | §10.6 declare cardinality; ch 05.5 |
| Context cost doubled after adopting a rules file | always-in-context tier unmeasured and uncached | §10.7 token budget per file + prompt cache |
| Users say memory is creepy | no write-moment, no management UI | §10.8 table |
| Framework swap stalls for a quarter | migration means re-extraction, unpriced | §10.0 arithmetic 3; ch 07 re-embedding runbook |
| Grep "proved" a mechanism that isn't there | substring match on an identifier | §10.1 bug 1, non-identifier lookarounds |
| Audit passed, production failed | probe hits were docstrings | §10.1 bug 2, read the files |

---

## 10.13 Exercises

1. **Re-run the audit.** Execute §10.1 verbatim today and diff against the numbers printed here.
   *Acceptance:* a table of every changed value (stars, releases, file counts, probe counts) plus one
   sentence per change saying whether it would alter a build/buy decision. If nothing changed, say so
   — that is also a finding.
2. **Add a sixth probe.** Design one for a property *your* product needs that none of P1–P5 covers
   (e.g. per-memory encryption, audit log of reads, confidence scores). *Acceptance:* the probe
   returns zero false positives across all six repos after you inspect every hit by hand; document
   any lookaround you needed and why.
3. **Break a probe on purpose.** Write a pattern that produces a false positive as convincing as
   `valid_to` inside `invalid_tool_message`. *Acceptance:* a one-line `re.search` demo, and the
   corrected pattern.
4. **Test the delete semantics.** Against any one candidate: store "user is vegetarian", store "user
   eats fish now", delete the second, then search for diet. *Acceptance:* written record of what came
   back, and a `must_not_contain` assertion (chapter 08.5) that fails before your fix and passes
   after.
5. **Price your always-in-context tier.** Run the §10.7 script against your own `CLAUDE.md` /
   `AGENTS.md` / rules file. *Acceptance:* tokens per turn, $ per 1,000 turns uncached and cached,
   and either a justification for every file above 1,800 tokens or a consolidation commit.
6. **Reproduce one published number.** Pick Mem0's LOCOMO result or Zep's LongMemEval result and try
   to reproduce it with your own harness. *Acceptance:* your number, their number, and a list of
   every reason they differ (actor model, judge, subset, retrieval config, prompt). The list will be
   long; that is chapter 08.3's lesson, felt rather than read.
7. **Write the migration plan you hope never to execute.** For the system you are most likely to
   adopt: how do you get your data *out*? *Acceptance:* a document with the export format, the
   re-extraction token cost at your volume, the re-embedding cost, and the eval that proves parity.
   If you cannot write it, you have not finished evaluating.
8. **Declare your functional predicates.** List every relation your extractor emits and mark each
   single- or multi-valued (§10.6). *Acceptance:* the list, plus at least one relation you discovered
   you were treating as functional when it is not — or proof there are none.

Next: `11-projects-small.md`.
