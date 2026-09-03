# Lane 07 — Evaluation harness and draft golden set

**Goal:** a golden set that is honest about its own confidence, a runner that works offline in
`replay` mode and online in `live` mode, and a metrics report the human can read out in a
stand-up without qualification.

**Files owned:** `backend/app/recsys/eval/**`, `docs/recsys/EVAL-REPORT.md`.

**Read first:** `ARCHITECTURE.md` §10, §5; `docs/recsys/RECON.md` §Cards, §Export files.

**Independence:** in wave 1 the service does not exist yet. Build the golden set and the metrics
code against the contracts; the runner calls `RecommendationService` through a small
`Runner` protocol that lane 09 will bind to the real service. Provide a `FakeRunner` for tests.

---

## Task 1 — Golden case schema (`eval/schema.py`)

Pydantic model matching the YAML in `ARCHITECTURE.md` §10. Validation: `kind` enum; exactly one
of `input.card_id` / `input.text`; `text` inputs must contain the marker `synthetic:` as the first
word of `notes` (a cheap guard against pasting real text); `expected` and `forbidden` ids must
match `^(skill|mcp|agent|knowledge_source):`; `confidence` enum; `labelled_by` non-empty.

Test: a valid case loads; each invalid variant raises.

## Task 2 — Golden set builder (`eval/build_golden.py`)

Reads the **export** directly (reuse lane 02's `excel_reader` — read-only import) and the hub's
cards (via the reader lane 01 documented), and writes draft YAML cases:

- **Positive (20)**: pick cards whose title/description mention a system or capability that
  also appears in the description of an eligible MCP or skill (simple normalised-token overlap ≥
  3 distinct tokens after stop-word removal). Expected ids = those overlapping components (cap 3
  per type). `confidence: low`, `labelled_by: draft-agent`, `notes:` explains the overlap tokens
  (tokens only, not sentences).
- **Agent-grounded positives (up to 10 of the 20)**: where an agent's description overlaps a
  card the same way, its `USES_MCP`/`USES_SKILL` targets become expected ids with
  `confidence: medium` (mechanical ground truth from the usage sheet).
- **Empty controls (6)**: synthetic texts written here, describing needs with no plausible
  catalogue match (e.g. a manual branch-opening checklist, a physical security patrol schedule, a
  catering order form, a request explicitly asking for no automation, a single-sentence greeting,
  a text in a language the catalogue does not use). `expected: {}`; `notes: "synthetic: …"`.
- **Near-duplicate pairs (5)**: pairs of cards linked by the hub's related-use-case field (recon
  names it); if fewer than 5 exist locally, pairs with the highest title-token overlap. Case
  `kind: near_duplicate_pair`, `input.card_id` for the first, `pair_card_id` for the second;
  expectation is overlap, not specific ids.
- **Stability (5)**: five of the positives re-declared with `kind: stability`.

Output: `eval/golden/g-NNN.yaml` and `eval/golden/INDEX.md` (case id, kind, confidence, whether a
human has confirmed). Never write card text into YAML — card cases carry `card_id` only.

Test: run the builder against synthetic fixtures (a tiny fake export + three fake cards) and
assert counts per kind and that every file validates.

## Task 3 — Metrics (`eval/metrics.py`)

Pure functions over `(GoldenCase, RecommendationResult | list[RecommendationResult])`:

- `precision_at_k`, `recall` per type; `forbidden_hits`; `empty_control_fp` (bool per case);
  `pair_overlap` (Jaccard of top-3 ids across the pair's two results per type);
  `stability` (all three runs share the same ordered top-3 per type);
  `aggregate(cases_results) -> EvalSummary` with per-kind and per-confidence breakdowns and the
  discard/timeout rates pulled from `timings`.

Tests with hand-built results: each metric with one exact numeric expectation.

## Task 4 — Runner (`eval/run.py`)

```
python -m app.recsys.eval.run --mode replay|live [--only g-001,g-002] [--kinds positive,empty_control]
       --report docs/recsys/EVAL-REPORT.md [--record]
```

`Runner` protocol: `async def recommend(request) -> RecommendationResult`. The module resolves
the real service when importable (`app.recsys.pipeline.service`) else exits 2 with "service not
built yet — use FakeRunner in tests". Stability cases run 3×. `--record` sets the recorder so a
live run produces the replay fixtures for subsequent offline runs. Concurrency: 2 cases at a
time (the service already bounds per-run calls).

Report sections: header (mode, catalogue version, prompt version, model, date, cases run /
skipped with reason); summary table vs the PoC targets from `ARCHITECTURE.md` §10 with a
met/not-met column; per-kind tables; per-case table (case id, kind, confidence, P@k, hits,
misses count — ids only, never names or text); "How to read this" paragraph stating that
`confidence: low` cases are draft labels and should not be quoted as accuracy figures.

Test: `FakeRunner` returning canned results → the report file is written and contains the
summary table and the target rows.

## Done when

- Golden directory has ≥ 36 validated cases with the kind counts above (fewer only if the local
  card count makes it impossible — then say so in INDEX.md).
- Metrics tests pass; runner works with `FakeRunner`.
- Handoff tells lane 09 the exact live and replay commands and that the report's numbers on
  `low`-confidence cases are not to be quoted as accuracy.
