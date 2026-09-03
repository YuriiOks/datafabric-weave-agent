# Lane 05 — Service, runs, and the real API

**Goal:** `RecommendationService` composes lanes 02/03/04 into the six-stage run, persists it as a
`recsys_run`, and the router from `ARCHITECTURE.md` §8 replaces the wave-0 stub. Submit-then-poll,
feature-flagged, every result versioned.

**Files owned:** `backend/app/recsys/pipeline/service.py`, `backend/app/recsys/api/**` (replace the
stub; keep the fixture files for the frontend mock), `backend/tests/recsys/api/**`.

**Read first:** `ARCHITECTURE.md` §7, §8, §12; `docs/recsys/RECON.md` §Cards, §Background work,
§Error envelope; handoffs of lanes 02, 03, 04.

---

## Task 1 — Catalogue loader

`service.py`: `load_catalogue(session) -> Catalogue` — reads the current version's **eligible**
rows and resolved links via the lane-01 repository and builds lane 03's `Catalogue`. Memoise
per process keyed on `catalogue_version`; invalidate when `get_current_version()` changes (check
on each request — one cheap query).

Also `integration_vocabulary(catalogue) -> list[str]`: normalised MCP names + all `tools` values
+ a fixed list (`confluence, jira, github, servicenow, slack, teams, sharepoint, email, sql
database, rest api, s3`), deduplicated, sorted.

Test: two versions inserted, the current one is loaded; an ineligible row never reaches the
catalogue; vocabulary contains the fixed list and the MCP names.

## Task 2 — Card input

`read_card_text(card_id) -> str | None` using the hub's card repository from recon; concatenates
title, summary/description, capabilities (as recon named them), nothing else — explicitly not
owner/contributor fields. Returns `None` for an unknown id. Test with the hub's own card test
fixture if one exists, else mock the repository at its import path.

## Task 3 — The run

```python
async def run(self, run_id: str) -> None:
    # 1 load run row → input (text passed in-memory by the submitter; NOT read back from DB)
```

Careful: the raw text must not be persisted, so the submitter keeps it in memory and passes it
to the background task directly. Implement `submit(request) -> run_id` that creates the row
(`input_sha256`, `input_length`, `input_kind`, `card_id`) and schedules `execute(run_id, text)`.

`execute`: status→running; S1 `build_profile`; embed the summary; S2 `retrieve`; S3
`assess_candidates` with the cache from the repository; S4 `verify` + `score_and_select`; S5
`synthesise`; assemble `RecommendationResult` with `timings`; status→done and store `result`.
Any `GatewayError` → status→failed with `error_code = e.code`; any other exception → failed with
`error_code = "internal"` and the traceback logged with `run_id` only.

Logging: one structured line per stage: `run_id, stage, component_type (if any), count,
duration_ms`. Test with `caplog`: the input text and component names never appear.

## Task 4 — Router (replace stub)

Endpoints exactly per `ARCHITECTURE.md` §8. Use the hub's error envelope. Dependencies:
`get_session`, `get_service` (singleton per app). `POST /recommendations` → 202 `{run_id,
status: "queued"}`; validates via the contract; `card_id` unknown → 404 before scheduling;
no current catalogue → 409 `no_current_catalogue`.

`GET /registry/components` passes filters straight to the repository; `q` is a case-insensitive
substring over `name` and `description` (Postgres `ILIKE`); `flag` filters on the jsonb list.
`GET /registry/vocabulary` returns enum values + `integration_vocabulary`.
`GET /cards/{card_id}/suggested-components` → latest done run for the card and current version,
else 404.

## Task 5 — API tests (`tests/recsys/api/`)

Use the app's test client with `RECSYS_ENABLED=true` and `RECSYS_LLM_MODE=off`, database seeded
with 6 synthetic components (2 per type) and a current version:

- submit text → 202; poll until done (loop ≤ 50 × 0.1 s) → `status == "done"`, `empty_reason ==
  "no_candidate_fit"` (off gateway returns none), `catalogue_version` set, near-misses non-empty.
- submit with both `text` and `card_id` → 400; unknown card → 404.
- registry list default excludes ineligible; `include_ineligible=true` includes; `flag=test_data`
  filters; `q` matches name substring; detail returns `used_by` for a linked component;
  unknown id → 404.
- status returns counts equal to the seed.
- with `RECSYS_ENABLED=false` every path is 404.
- with `RECSYS_LLM_MODE=replay` and a recorded fixture set for one synthetic case (write it by
  hand — it is JSON), a full run produces one `strong` recommendation with verified quotes. This
  is the one test that exercises the whole pipeline end to end.

## Task 6 — OpenAPI check

`GET /openapi.json` includes the seven paths with the contract schemas; write a test asserting the
`RecommendationResult` schema is present and has `empty_reason`.

## Done when

- API tests pass; the stub `STUB = True` marker is gone; fixture JSON under
  `tests/recsys/fixtures/stub/` still exists for the frontend mock and matches the contracts
  (add a test that validates each fixture file against its model).
- Handoff records the exact poll interval the UI should use and any deviation from §8.
