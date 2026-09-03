# Lane 03 — Deterministic retrieval

**Goal:** `retrieve(need: NeedProfile, need_embedding: list[float], types: list[ComponentType],
k: int, catalogue) -> dict[ComponentType, list[Candidate]]` — lexical + vector + graph
co-occurrence fused by RRF, reproducible bit-for-bit, no model calls, no database calls inside the
hot path.

**Files owned:** `backend/app/recsys/retrieval/**`, `backend/tests/recsys/retrieval/**`.

**Read first:** `ARCHITECTURE.md` §5 (Candidate, RetrievalScores, NeedProfile), §7.2.

**Independence:** this lane does not depend on lane 02's database code. It works on an
in-memory `Catalogue` object (defined here) that lane 05 will populate from the repository.

---

## Task 1 — In-memory catalogue (`retrieval/__init__.py`)

```python
@dataclass(frozen=True)
class CatalogueItem:
    component_id: str; component_type: ComponentType; retrieval_text: str
    embedding: tuple[float, ...] | None; tokens: tuple[str, ...]   # precomputed

@dataclass
class Catalogue:
    version: str
    items: dict[str, CatalogueItem]
    links: list[tuple[str, str, str]]          # (src_id, dst_id, link_type), resolved only
    def by_type(self, t: ComponentType) -> list[CatalogueItem]: ...
    @classmethod
    def from_rows(cls, version, rows: Iterable[Mapping], links) -> "Catalogue": ...  # tokenises once
```

`tokenise(text)`: lower, split on non-alphanumerics, drop tokens < 2 chars and a 40-word English
stop list, keep order. Test: deterministic, handles hyphenated ids (`ibm-i-impact` → `ibm`, `impact`
and also the joined form `ibmiimpact`? — no: keep it simple, split only).

## Task 2 — Lexical (`retrieval/lexical.py`)

BM25 (`k1=1.5`, `b=0.75`) over `tokens` per type. If `rank_bm25` is importable use it; otherwise
implement: document frequencies computed once per `Catalogue` and memoised on the object
(`catalogue._bm25[type]`). Query tokens = `tokenise(need.summary + " " + " ".join(capabilities)
+ " " + " ".join(integrations))`.

Returns `list[tuple[component_id, score]]` sorted by score desc then id asc.

Tests (write first): an item whose text contains the query terms outranks one that does not;
two items with identical text tie and are ordered by id; empty query returns an empty list.

## Task 3 — Vector (`retrieval/vector.py`)

Cosine similarity via numpy between `need_embedding` and a matrix built once per `Catalogue`
per type (memoised, items without embedding excluded). Return `(id, cosine)` sorted desc, id asc.
If numpy is unavailable (recon says so), a pure-Python dot product is acceptable at this scale —
implement behind the same function signature.

Tests: identical vectors score 1.0; orthogonal 0.0; an item with `embedding=None` never appears;
the matrix is built once (assert via a counter on the memoised builder).

## Task 4 — Graph co-occurrence (`retrieval/graph.py`)

1. Score agents by vector similarity to the need (reuse Task 3 over `ComponentType.agent`).
2. Take the top 5 agents with similarity > 0.2.
3. For each `(agent, target, USES_*)` link, `graph_score[target] += agent_similarity`.
4. For each retrieved MCP (from steps 1–3 or the vector list top-10 of type `mcp`), every skill
   with `ALLOWS_MCP` to it gets `+= 0.3 * mcp_score`.
5. Normalise by the max so scores are in `[0, 1]`.

Return per type `(id, score)` sorted desc, id asc. When no agents pass the threshold, return empty
lists (this is the expected state for a catalogue with no agents in the type set).

Tests: a component used by the two most similar agents outranks one used by one agent; a skill
allowing a retrieved MCP gets a non-zero score; no agents → empty.

## Task 5 — Fusion (`retrieval/fuse.py`)

Reciprocal rank fusion: for each signal list, `1 / (60 + rank)` × weight (lexical 1.0, vector 1.0,
graph 0.7). Sum per id. Keep the raw per-signal scores in `RetrievalScores` (0 when absent).
Sort by fused desc, id asc; take top `k`; assign `rank` from 1.

Tests: an item present in all three lists outranks an item first in one list only; tie-break by
id; `k` respected; per-signal scores are carried through untouched.

## Task 6 — `retrieve()` and a stability property test

`retrieve(...)` composes 2–5 per requested type. Only items with `eligible=True` are ever in the
`Catalogue` (lane 05 filters on load; add an assertion in `from_rows` that rejects rows with
`eligible=False` so the invariant is enforced at the boundary).

Property test (no hypothesis dependency needed — loop 20 times over shuffled input order):
`retrieve` returns identical output regardless of the order rows were inserted in `from_rows`.

## Task 7 — Micro-benchmark note

In the handoff, report the wall time of `retrieve` on a synthetic catalogue of 1 000 items with
3072-dim embeddings (generate random vectors). Expect well under 100 ms. If it is not, say why.

## Done when

- Tests pass; the stability property holds; the handoff carries the timing and a note on which of
  `rank_bm25` / `numpy` were available and which fallback was used.
