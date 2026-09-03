# Lane 04 — LLM gateway, prompts, and the bounded pipeline stages

**Goal:** the three model-facing stages (need profile, per-candidate assessment, synthesis) plus
verification and scoring, built so that **the model can only select and quote** — never invent —
and so that everything runs offline in `replay` and `off` modes for tests and the UI.

**Files owned:** `backend/app/recsys/llm/**`, `backend/app/recsys/prompts/**`,
`backend/app/recsys/pipeline/{profile,assess,verify,score,synthesise}.py`,
`backend/tests/recsys/llm/**`, `backend/tests/recsys/pipeline/**`,
`backend/tests/recsys/fixtures/**` (except `stub/` and `registry/`).

**Read first:** `ARCHITECTURE.md` §3, §5, §7, §12, §13; `docs/recsys/RECON.md` §LLM client.

**Not owned:** `pipeline/service.py` (lane 05 composes your stages). Expose clean async functions.

---

## Task 1 — Gateway protocol and the `off` implementation (`llm/gateway.py`, `llm/off.py`)

```python
class LLMGateway(Protocol):
    name: str                      # e.g. "live:<model>", "replay", "off"
    async def structured(self, *, system: str, user: str, schema: type[T], timeout_s: float) -> T: ...
    async def embed(self, texts: list[str]) -> list[list[float]]: ...

class GatewayError(Exception): code: str        # "llm_unavailable" | "llm_timeout" | "llm_invalid_output" | "replay_miss"
```

`OffGateway.structured` returns: for `NeedProfile` a fixed profile (`domains=[other]`,
`capabilities=[other]`, `integrations=[]`, `summary="(model disabled)"`); for `Assessment`
`fit=none` with empty quotes; for the synthesis schema an echo of the given ids. `embed` returns
deterministic pseudo-vectors (`hash(text)` seeded, dimension 3072) so retrieval still ranks.

`get_gateway(settings) -> LLMGateway` selects by `RECSYS_LLM_MODE`, wrapping `live` with the
recorder when `RECSYS_LLM_RECORD=1`.

Test: `off` returns valid instances for every schema; `get_gateway` respects the mode.

## Task 2 — `replay` and `recorder` (`llm/replay.py`, `llm/recorder.py`)

Key = `sha256(schema.__name__ + "\n" + system + "\n" + user)`. Replay reads
`backend/tests/recsys/fixtures/replay/<key>.json` (validated into the schema); miss → `GatewayError("replay_miss")`
with the key in the message so a missing fixture is easy to record. Recorder wraps any gateway and
writes the file after a successful call (never on failure). Embeddings are recorded too, under
`replay/embed/<sha256(text)>.json`.

Tests: record → replay round trip with a fake inner gateway; miss raises with the key.

## Task 3 — `live` adapter (`llm/live.py`)

Thin wrapper around the client recon identified. Structured output: use the hub's existing
mechanism if it is a real JSON/tool-schema mode; otherwise send the JSON schema
(`schema.model_json_schema()`) in the system prompt, request JSON only, parse, validate; on
`ValidationError` retry **once** with the error text appended to the user message; second failure
→ `GatewayError("llm_invalid_output")`. Apply `timeout_s` with `asyncio.wait_for`; timeout →
`GatewayError("llm_timeout")`. Transport errors → `GatewayError("llm_unavailable")`.

Test with a fake underlying client: invalid JSON first then valid → returns valid, one retry
counted; two invalid → `llm_invalid_output`.

## Task 4 — Prompts (`prompts/`)

`prompts/__init__.py`: `PROMPT_VERSION = "1"`.

**`need_profile.py`** — system: role (analyst classifying an AI use case for reuse), the exact
enum values for domains and capabilities, the integrations vocabulary injected at call time,
rules: "summary ≤ 60 words, in your own words; choose only from the listed values; use `other`
when nothing fits; never invent system names — if the text names a system not in the
vocabulary, put `other` and mention it in `constraints`". User: the input text.

**`assess.py`** — system: role (reviewer judging whether ONE component serves the described
need), the fit definitions (`strong`: the component's stated purpose covers the need's main
capability and integration; `partial`: covers a part or needs unmet preconditions; `none`:
no meaningful overlap), the quote rules: "`evidence_component` must be copied verbatim from the
COMPONENT block; `evidence_need` must be copied verbatim from the NEED block; if you cannot find a
supporting verbatim span, answer `none`". User: `NEED:` block (summary + fields) then
`COMPONENT:` block (name, description, triggers, tools, tags). Nothing else — no other
candidates, no ids beyond the one.

**`synthesise.py`** — system: "order the given recommendations from most to least useful for the
need; write ≤ 30 words of rationale each; you MUST return exactly the given ids, no more, no
fewer". User: need summary + numbered list `[i] id — name — fit — evidence`. Output schema
`SynthesisOut(items: list[SynthesisItem(index: int, rationale: str)])` — **indices, not ids**;
the code maps back.

Store a synthetic example input/output pair per prompt under `fixtures/prompt_examples/` and a
test that each example validates against its schema.

## Task 5 — Stages

`pipeline/profile.py`: `async def build_profile(text: str, vocabulary: list[str], gateway) ->
NeedProfile`. Post-validation: drop integrations not in `vocabulary + ["other"]` (count dropped
into a returned `ProfileStats`), truncate summary to 60 words.

`pipeline/assess.py`: `async def assess_candidates(need, input_text, candidates:
list[(Candidate, ComponentRecord)], gateway, cache, *, concurrency, timeout_s) ->
list[Assessment | AssessFailure]`. Cache key per `ARCHITECTURE.md` §4.5; semaphore; timeouts →
`AssessFailure(component_id, "timeout")`; gateway errors other than timeout propagate (the run
fails as a whole — do not hide an unavailable model behind an empty result).

`pipeline/verify.py`: `verify(assessment, component_text, need_text) -> Assessment` sets
`quotes_verified`; normalisation = lower + collapse whitespace + strip quotes/backticks at the
ends. `fit=none` assessments verify trivially (no quotes required).

`pipeline/score.py`: `score_and_select(assessments, candidates, min_score, max_per_type) ->
(recommendations, near_misses, empty_reason)` per `ARCHITECTURE.md` §7.4. `fused_norm` is min-max
within the type; a single candidate gets `fused_norm = 1.0`.

`pipeline/synthesise.py`: `async def synthesise(need, recs, gateway) -> list[Recommendation]`;
skipped when `len(recs) < 2`; validates the index set equals `range(len(recs))`; on any deviation
or gateway error returns the input order with rationale = `evidence_component`.

## Task 6 — Tests that pin the principle (write these first, they define the lane)

- `test_assess_rejects_unverifiable_quote`: a fake gateway returns `fit=strong` with a quote not in
  the component text → after `verify`, `quotes_verified=False`; after `score_and_select` the
  component is absent and `discarded_unverified == 1`.
- `test_none_fit_yields_empty_with_reason`: all assessments `fit=none` → empty recommendations,
  `empty_reason == "no_candidate_fit"`, near-misses contain the top-3 retrieved.
- `test_synthesis_cannot_add_or_drop_ids`: gateway returns indices `[0, 0]` for two recs → result
  is deterministic order; returns `[0, 1, 2]` for two recs → deterministic order.
- `test_assess_uses_cache`: second call with the same key does not hit the gateway.
- `test_assess_timeout_is_failure_not_fit`: a gateway that sleeps past `timeout_s` produces
  `AssessFailure("timeout")`, and `empty_reason == "assessments_timed_out"` when all fail.
- `test_profile_drops_unknown_integrations`.
- `test_no_raw_text_in_logs`: run `build_profile` and `assess_candidates` with `caplog` at DEBUG
  and assert the input text does not appear in any record.

## Done when

- Tests pass in `replay`/`off` modes with no network.
- Handoff records: which structured-output mechanism the live adapter uses, the model name
  constant it reuses, and the exact command to record fixtures on the workstation
  (`RECSYS_LLM_MODE=live RECSYS_LLM_RECORD=1 …`).
