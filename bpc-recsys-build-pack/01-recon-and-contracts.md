# Lane 01 — Recon, contracts, scaffolding (wave 0, run by the lead)

**Goal:** answer every item in `ARCHITECTURE.md` §16, write the frozen contracts, create the
package skeleton, the migration, a stub router serving fixtures, and the ≤ 3 guarded touches to
existing hub files — so that five lanes can start in parallel without touching each other.

**Files owned:** see `00-LAUNCH.md` §5, lane 01.

**Do not** implement ingestion, retrieval, prompts, or UI here. Stubs only.

---

## Task 1 — Recon (read-only, ~30 min)

Write `docs/recsys/RECON.md` with a heading per item below and a **verified** answer (cite
`path:line`). "Unknown" is acceptable only with the command you tried.

1. **Backend framework & layout.** Confirm FastAPI; the app factory; where routers are imported and
   included (`include_router` call sites). Record the pattern used to mount a router conditionally,
   if one exists.
2. **Settings.** How configuration is read (pydantic-settings? env? a settings module?). Record how
   an existing boolean flag such as the bounded-search flag is declared and read. `config.py` will
   copy that pattern.
3. **ORM & migrations.** SQLAlchemy? Declarative base location; session dependency; the migration
   tool and the command to autogenerate and to apply; where the test database comes from.
4. **LLM client.** Module, class, method signature for a chat completion; how structured/JSON
   output is requested today (tool call? `response_format`? free text + parse?); model name
   constants; timeout and retry behaviour; whether calls are sync or async. Same for the
   **embedding client**: batch limits and dimension — run a one-text embed in a Python shell and
   record `len(vector)` (expect 3072).
5. **Cards.** The repository/service that returns a card by id; the field names for title,
   summary/description, capabilities, related use cases; how many cards the local checkout has
   (`ls "best practise cards" | wc -l` and the count the hub's own ingestion reports). Write both
   numbers; if they differ from 213, say so plainly — do not resolve, just record.
6. **Background work.** Is there a job table / polling mechanism used by search? Signature and
   whether it is generic enough to reuse. Default recommendation: use `BackgroundTasks` +
   `recsys_run` unless reuse is a five-line change.
7. **Frontend.** Router library and route table location; page/feature folder convention; API
   client convention (fetch wrapper, generated client, base URL handling); how navigation entries
   are added; the card page component path and a component boundary where a panel can be
   mounted; test runner and command; type-check and build commands.
8. **Toolchain.** Backend test command, lint, format; frontend lint/type-check/build; pre-commit
   hooks present?
9. **Dependencies available.** In the backend environment: `python -c "import openpyxl"`, `pandas`,
   `rank_bm25`, `numpy`, `mcp`, `yaml`. Record which import succeeds. Adding a dependency is
   allowed only if the repo's dependency file is edited and the install command succeeds offline;
   otherwise lanes use the fallbacks named in their briefs.
10. **Export files.** `ls data/agenthub-export/`; for each file, list sheet names and the header
    row of each sheet (structure only — no data rows in the recon file). If CSVs, list headers.
11. **Git and PR tooling.** Default branch name; `gh --version` and `gh auth status`; remote URL
    host (record host only, no tokens).
12. **Existing conventions to copy.** Error envelope shape for API errors; logging setup (logger
    name convention, structured fields); a representative existing test file to mirror.

Commit nothing yet.

## Task 2 — Package skeleton and config

Create `backend/app/recsys/__init__.py` (empty docstring), `config.py` following the settings
pattern from recon with every key in `ARCHITECTURE.md` §13 and its default, and empty packages
`registry/`, `retrieval/`, `llm/`, `prompts/`, `pipeline/`, `api/`, `db/`, `eval/` each with an
`__init__.py`. Create `backend/tests/recsys/__init__.py` and `conftest.py` that provides:

- a `db_session` fixture using the repo's existing test-database pattern;
- a `fake_components()` factory producing synthetic `ComponentSummary`/row objects (names like
  `Alpha Ticket Helper`, descriptions in plain English, never real catalogue text);
- an `off_gateway` fixture returning the `off` gateway once lane 04 exists (import guarded with
  `pytest.importorskip` until then).

## Task 3 — Contracts

Write `backend/app/recsys/contracts.py` **exactly** as in `ARCHITECTURE.md` §5, plus the validator
on `RecommendationRequest` (exactly one of `text`/`card_id`). Test first:

`backend/tests/recsys/test_contracts.py`
```python
def test_request_requires_exactly_one_input():
    with pytest.raises(ValidationError): RecommendationRequest()
    with pytest.raises(ValidationError): RecommendationRequest(text="x", card_id="UC1")
    assert RecommendationRequest(text="x").text == "x"

def test_result_round_trips_json():
    r = RecommendationResult(run_id="r", status="done", catalogue_version="v", prompt_version="1",
                             model="m", input_kind="text", card_id=None, need_profile=None,
                             created_at=datetime.now(timezone.utc))
    assert RecommendationResult.model_validate_json(r.model_dump_json()) == r
```
Run → fail → implement → pass.

Mirror to `frontend/src/features/recsys/types.ts`: one TS type per Pydantic model, enums as
string-literal unions, `datetime` as `string`. Add a comment at the top: `// Mirrors
backend/app/recsys/contracts.py — change both or neither.`

## Task 4 — DB models and migration

`backend/app/recsys/db/models.py`: the five tables from `ARCHITECTURE.md` §4 using the repo's
declarative base. Indexes: `recsys_component (catalogue_version, component_type, eligible)`,
`recsys_component_link (catalogue_version, src_id)`, `(catalogue_version, dst_id)`,
`recsys_run (card_id, catalogue_version, status)`. Exactly one `is_current` row: enforce in the
repository, not the schema (PoC).

`backend/app/recsys/db/repository.py`: `get_current_version()`, `list_components(version, type,
q, flag, include_ineligible, limit, offset)`, `get_component(version, id)`, `links_for(version,
id)`, `create_run(...)`, `update_run(...)`, `get_run(id)`, `latest_done_run_for_card(card_id,
version)`, `cache_get(key)`, `cache_put(key, assessment)`. Tests with the `db_session` fixture:
insert two synthetic components, list with each filter, get one, 404 case.

Generate the migration with the repo's tool; open it and check the downgrade drops the five
tables and nothing else. Apply + downgrade + apply on the test database.

## Task 5 — Stub router and guarded touches

`backend/app/recsys/api/router.py`: every endpoint from `ARCHITECTURE.md` §8 returning fixture
data from `backend/tests/recsys/fixtures/stub/*.json` (synthetic), with a module-level
`STUB = True` comment so lane 05 knows to replace it. Mount it in the app factory behind
`RECSYS_ENABLED` — **this is guarded touch 1**; keep it to a single `if settings.recsys_enabled:`
block.

Frontend: register routes `/recommend`, `/registry`, `/registry/:componentId`,
`/registry/status` pointing at placeholder page components under `features/recsys/pages/` that
render their own name — **guarded touch 2** (a single block in the route table, plus one nav
entry if the app has a nav). Card page panel mount point: add a single `<SuggestedComponentsPanel
cardId={…} />` line where recon found the boundary, with the component being a placeholder that
renders nothing until lane 06 replaces it — **guarded touch 3**.

`.gitignore`: add `data/agenthub-export/`, `data/recsys-cache/` if not covered.

Test: backend app starts with `RECSYS_ENABLED=true` and `GET /api/recsys/registry/status` returns
the stub 200; with `false` it returns 404. Frontend type-check passes.

## Task 6 — Commit

`git add` the owned paths; commit `recsys(scaffold): contracts, migration, stub router, recon`.
Write `docs/recsys-build/PROGRESS.md` with wave 0 done and the commit sha.

## Done when

- `RECON.md` has all 12 headings answered with citations.
- Contract tests pass; migration up/down/up works; stub endpoints respond; frontend builds.
- Handoff file written per `00-LAUNCH.md` §7.
