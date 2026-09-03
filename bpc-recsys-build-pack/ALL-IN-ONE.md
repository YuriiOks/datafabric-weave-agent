# Component Recommendation PoC — build pack (all files concatenated)

Lead: split this back into docs/recsys-build/<name>.md on the FILE markers before starting.


<!-- ===== FILE: docs/recsys-build/ARCHITECTURE.md ===== -->

# Component Recommendation Service — Architecture and Build Contract

> **For the lead agent:** this is the single source of truth for the build. Read it fully before
> reading any lane brief. Lane briefs (`00`–`09`) never override this document; when they
> conflict, this document wins and the lane reports the conflict in its handoff.

**Goal:** an open pull request on branch `feat/component-recommendations-poc` in the AI Playbooks
Hub repository that adds a working, clickable, end-to-end proof of concept which — given a use
case (free text or an existing best-practice card) — recommends reusable **skills, MCP servers and
agents** from the bank's Agent Hub catalogue, with structural guarantees against hallucination, a
registry browser for the catalogue itself, and an evaluation harness with a draft golden set.

**Architecture in three sentences:** a deterministic **registry** is built from two sources (the
Agent Hub metadata export and the skills marketplace repository) into versioned Postgres tables
with quality flags; a six-stage **recommendation pipeline** turns a use case into a controlled-
vocabulary need profile, retrieves candidates deterministically (lexical + vector + graph
co-occurrence), lets the model **select and justify from an enumerated candidate list only**,
verifies every quoted piece of evidence by exact substring match, and returns an empty result as a
first-class outcome; a FastAPI router and a React/Ant Design UI expose the pipeline and the
registry, and an evaluation script measures precision, false-positive rate on deliberately empty
cases, and stability.

**Tech stack (existing, reuse only):** FastAPI backend under `backend/`, Postgres, Redis, FalkorDB
(graph + 3072-dim vector index), the hub's existing LLM and embedding gateway (GAIP via the
internal API gateway), React 19 / Vite / Ant Design 6 / Tailwind 4 frontend. **No new services, no
new infrastructure, no new graph model.** New code lives in one backend package and one frontend
feature folder.

---

## 0. Why this exists (context the agent must carry)

The AI Playbooks Hub stores best-practice cards for AI use cases in a graph, offers search with a
novelty verdict (NOVEL / PARTIAL / COVERED), and scores card completeness. Leadership wants the hub
to also recommend **reusable components** — skills, MCP servers, agents — so that teams stop
rebuilding what exists. The catalogue of those components lives in Agent Hub, whose metadata was
exported to a four-sheet spreadsheet; skills additionally have a source repository
(`gaip-agenthub-skills-marketplace`) with one `SKILL.md` per skill.

What is known about the data, and what the design must survive:

| Fact | Consequence for design |
|---|---|
| MCP identifiers in the export are not unique (the same id appears up to four times, sometimes with contradictory name/description) | Identity is derived by us, never trusted from the source; duplicates are flagged, not silently merged |
| The knowledge-source sheet contains obvious test rows (`test`, `a`, `upload test`, misspelt PoC names) | Exclusion heuristics with a visible flag, off-by-default inclusion |
| Nine skills in the repo have no front matter; their raw file content leaks into the export's description column | `raw_yaml_description` and `missing_frontmatter` flags; such rows are ineligible by default |
| No schema validation exists anywhere in the skills pipeline; `version` is absent in 59 %; the same field is spelt two ways (`user-invocable` / `user-invokable`); metadata lives in two places (top level vs `metadata.*`) | The SKILL.md parser is tolerant by design: it normalises known variants, records which variant it saw, and never drops a value silently |
| Twelve skills have folder name ≠ front-matter `name`; one `name` is declared in two folders | Canonical skill identity = **folder slug**; `name` kept as alias; the two-folder case is flagged `duplicate_name_across_folders` |
| Dependencies are declared by 11 % of skills, in five different fields, with no URI form | We record links where they exist, mark unresolved targets, and treat co-occurrence in the agent sheet as the primary signal rather than declared dependencies |
| 291 skills in repo vs ~245 in the export; the portal database is the authoritative registry, the repo is the content source | Two-source merge with `source` recorded per record; a skill present in only one source is still a component, with `only_in_repo` / `only_in_export` recorded in `external_ids` |
| The hub's card ingestion parser silently drops unrecognised front-matter keys | Our parsers **log every unconsumed key** with a count in the ingest report; same class of bug, deliberately made visible |
| The hub's existing vector index is dropped and rebuilt on each card ingestion | We do **not** depend on the hub's index for component embeddings in the PoC; embeddings are stored in our own table and compared in memory (catalogue is < 1 000 items) |
| Leadership rejected a proposal to build a new graph/data model | We add nodes/edges to the **existing** graph only as an optional projection behind a flag, using the hub's own MERGE-only ingestion utilities and the `EMPLOYS_AGENT` edge as precedent |

**Political framing that must show in the code and the PR:** this is an *extension* of the hub
(new package, new router, new UI feature, additive tables), not a replacement for anything.
Nothing in the existing search, novelty verdict, completeness scoring or card ingestion changes.

---

## 1. Non-goals

- No change to the existing search, novelty verdict, or completeness scoring.
- No new graph model; no replacement of FalkorDB, Postgres or Redis.
- No authentication or authorisation work. The hub's existing middleware protects the new router
  exactly as it protects the others. Ingestion is a **CLI**, not an endpoint.
- No live MCP transport unless the `mcp` Python package is already available in the environment;
  if it is not, the MCP adapter lane documents the HTTP equivalents and stops (see lane 08).
- No production rollout. `RECSYS_ENABLED` defaults to `false`; the PoC is exercised on a
  workstation and in the PR's review environment.
- No CxO or MRM integration. Governance data is consumed later, never reproduced here.

---

## 2. System overview

```
                       ┌──────────────────────────────────────────────────────┐
  Agent Hub export     │  REGISTRY (backend/app/recsys/registry)               │
  (.xlsx or 4 .csv) ──►│  parse ─► normalise ─► identity ─► flags ─► embed     │
  skills repo          │  ─► write recsys_component / recsys_component_link    │
  (SKILL.md × N)   ───►│  ─► recsys_catalogue_version (sha over inputs)        │
                       └───────────────┬──────────────────────────────────────┘
                                       │ read-only at query time
                       ┌───────────────▼──────────────────────────────────────┐
  use case text        │  PIPELINE (backend/app/recsys/pipeline)              │
  or card_id      ────►│  S1 need profile (LLM, controlled vocab)             │
                       │  S2 retrieve: lexical + vector + graph → RRF → top-K │
                       │  S3 assess: one LLM call per candidate, index-only   │
                       │  S4 verify quotes (substring) + threshold            │
                       │  S5 synthesise ordering (LLM may not add ids)        │
                       │  S6 present: recommendations + near-misses + empty   │
                       └───────┬───────────────────────────┬──────────────────┘
                               │                           │
                ┌──────────────▼───────────┐   ┌───────────▼───────────────┐
                │ FastAPI router /api/recsys│   │ MCP adapter (optional)    │
                │ submit → poll, registry   │   │ 3 tools over same service │
                └──────────────┬───────────┘   └───────────────────────────┘
                               │
                ┌──────────────▼───────────────────────────────────────────┐
                │ React feature frontend/src/features/recsys                │
                │ /recommend  /registry  /registry/:id  /registry/status   │
                │ + "Suggested components" panel on the card page          │
                └──────────────────────────────────────────────────────────┘
                               ▲
                ┌──────────────┴───────────────────────────────────────────┐
                │ EVAL (backend/app/recsys/eval): golden set + metrics      │
                └──────────────────────────────────────────────────────────┘
```

---

## 3. The one principle everything else follows from

**The model never generates an identifier. It selects from a list we give it, by index, and it
justifies the selection with quotes we can verify.**

Concretely:

1. Retrieval is deterministic code. The model sees at most K candidates per type, each shown as
   `[i] name — description (truncated)`; the model's output schema refers to candidates by `i`.
2. The assessment step is **one candidate per call**. The model cannot trade candidates off
   against each other in one context, so it cannot invent a "better" one.
3. Every assessment must contain two verbatim quotes: one from the component's record, one from the
   need profile / input. The verification layer normalises whitespace and case and checks exact
   substring membership. An assessment whose quotes do not verify is discarded, and the discard is
   counted in the run manifest.
4. "Nothing suitable" is a first-class result. The threshold is applied *after* assessment; an
   empty list with `empty_reason` is a success, not an error.
5. The synthesis step may reorder and write one-line rationales; it is validated to contain only
   ids that were in its input. Any extra id fails validation and the run falls back to the
   deterministic order.

Everything else — prompts, weights, K — is a tunable. This is not.

---

## 4. Data model (additive, Postgres)

All tables prefixed `recsys_`. Use the backend's existing ORM and migration tool (recon decides:
SQLAlchemy + Alembic is the expectation; if the backend uses raw SQL or another tool, follow it).

### 4.1 `recsys_component`

| column | type | notes |
|---|---|---|
| `component_id` | text PK | canonical, `"{type}:{slug}"`, e.g. `mcp:confluence`, `skill:ibm-i-impact-analyzer`, `agent:tenant-ellipse/some-agent` |
| `component_type` | text enum | `skill` \| `mcp` \| `agent` \| `knowledge_source` |
| `name` | text | display name as in source |
| `description` | text | as in source, untouched |
| `source` | text enum | `agenthub_export` \| `skills_repo` \| `merged` |
| `source_ref` | text | structure only: `sheet=Skill List;row=17` or `path=skills/<folder>/SKILL.md` |
| `external_ids` | jsonb | `{ "portal_id": ..., "folder": ..., "frontmatter_name": ..., "only_in_repo": true }` |
| `tenants` | jsonb (list) | from Agent Usage sheet for agents; propagated to components an agent uses |
| `version` | text nullable | first non-empty of top-level `version`, `metadata.version` |
| `triggers` | jsonb (list) | trigger phrases extracted from description ("Use when …", "Trigger:", etc.) |
| `tools` | jsonb (list) | `allowed-tools` / `tools` for skills; tool names for MCP if present |
| `tags` | jsonb (list) | `category`, `visibility`, free tags, normalised lower-case |
| `quality_flags` | jsonb (list) | see §4.4 |
| `eligible` | boolean | derived: no blocking flag (see §4.4) |
| `retrieval_text` | text | derived: `name + description + triggers + tools + tags` |
| `embedding` | jsonb (list of float) nullable | 3072-dim, same embedder as the hub uses for cards |
| `catalogue_version` | text FK | version this row belongs to |

Rows are written **per catalogue version**; a re-ingest creates a new version and new rows; old
versions are kept (the PoC never deletes). Queries always filter on the current version.

### 4.2 `recsys_component_link`

| column | type |
|---|---|
| `src_id` | text |
| `dst_id` | text |
| `link_type` | `USES_SKILL` \| `USES_MCP` \| `USES_KNOWLEDGE` \| `DEPENDS_ON_SKILL` \| `ALLOWS_MCP` |
| `source` | text (`agent_usage_sheet` \| `frontmatter:<field>`) |
| `resolved` | boolean — false when `dst_id` did not match any component |
| `catalogue_version` | text |

`USES_*` come from the Agent Usage sheet (comma-separated id columns). `DEPENDS_ON_SKILL` from
`metadata.dependencies.skills`. `ALLOWS_MCP` from `metadata.allowed_mcp` (plain names → resolved by
slug; empty strings ignored and counted).

### 4.3 `recsys_catalogue_version`

| column | notes |
|---|---|
| `version` | `sha256(sorted input file hashes + ingest_code_version)[:16]` |
| `created_at` | |
| `ingest_code_version` | constant in `registry/__init__.py`, bump when parsing rules change |
| `counts` | jsonb: per type, eligible / ineligible |
| `flag_histogram` | jsonb: flag → count |
| `unconsumed_keys` | jsonb: front-matter key → count (the "silent loss made visible" report) |
| `inputs` | jsonb: list of `{ "kind": "export|repo", "hash": ..., "rows|files": n }` — **no paths, no names** |
| `is_current` | boolean, exactly one true |

### 4.4 Quality flags

| flag | blocks eligibility | rule |
|---|---|---|
| `duplicate_id` | yes | same source id appears more than once in a sheet; every duplicate carries the flag, records are kept distinct with suffix `#2`, `#3` |
| `name_description_mismatch` | yes | name mentions system X, description mentions a different system from a small list (Confluence/Jira/GitHub/ServiceNow/Slack/Teams/SharePoint) |
| `test_data` | yes | name or description matches `^(test|a|tmp|dummy|sample|upload test|poc test)$` (case-insensitive) or length < 3, or contains "hallucition" |
| `missing_frontmatter` | yes | SKILL.md has no leading `---` block |
| `raw_yaml_description` | yes | export description starts with `---` or contains `\nname:` |
| `empty_description` | yes | description empty after strip |
| `no_version` | no | no version in either location |
| `folder_name_mismatch` | no | folder slug ≠ slugified front-matter `name` |
| `duplicate_name_across_folders` | no | same front-matter `name` in two folders |
| `unresolved_dependency` | no | at least one declared link target did not resolve |
| `nested_skill` | no | SKILL.md more than one level below `skills/` |
| `spelling_variant` | no | any known spelling variant was normalised (`user-invokable` → `user-invocable`) |

`eligible = not any(blocking flags)`. The API accepts `include_ineligible=true` for the registry
browser; the recommendation pipeline never retrieves ineligible components.

### 4.5 Run and cache tables

`recsys_run`: `run_id` (uuid), `status` (`queued|running|done|failed`), `catalogue_version`,
`prompt_version`, `model`, `input_sha256`, `input_length`, `input_kind` (`text|card`),
`card_id` nullable, `result` jsonb nullable (the `RecommendationResult`), `error_code` nullable,
`timings` jsonb, `created_at`, `finished_at`.

**The raw input text is never stored and never logged.** Only its hash, its length and the derived
need profile (which contains a ≤ 60-word model-written summary and controlled-vocabulary fields)
are persisted. Verified evidence quotes (≤ 200 chars each) are stored inside `result`.

`recsys_assessment_cache`: `cache_key` PK = `sha256(input_sha256 + component_id +
catalogue_version + prompt_version + model)`, `assessment` jsonb, `created_at`. Hit before calling
the model; write after verification succeeds.

---

## 5. Contracts (Pydantic, `backend/app/recsys/contracts.py`; mirrored in
`frontend/src/features/recsys/types.ts`)

These are written in wave 0 and are frozen for the duration of the build. Lanes may add fields
only by appending optional fields, and must note the addition in their handoff.

```python
class ComponentType(str, Enum):
    skill = "skill"; mcp = "mcp"; agent = "agent"; knowledge_source = "knowledge_source"

class QualityFlag(str, Enum):  # values exactly as in §4.4

class ComponentSummary(BaseModel):
    component_id: str
    component_type: ComponentType
    name: str
    description: str            # may be truncated to 600 chars in list responses
    tenants: list[str] = []
    version: str | None = None
    quality_flags: list[QualityFlag] = []
    eligible: bool

class ComponentDetail(ComponentSummary):
    source: str
    source_ref: str
    external_ids: dict[str, str | bool]
    triggers: list[str] = []
    tools: list[str] = []
    tags: list[str] = []
    used_by: list[ComponentSummary] = []      # agents that USES_* this component
    uses: list[ComponentSummary] = []         # for agents
    depends_on: list[ComponentSummary] = []
    catalogue_version: str

class Domain(str, Enum):
    lending = "lending"; payments = "payments"; compliance = "compliance"; kyc_aml = "kyc_aml"
    trading = "trading"; risk = "risk"; hr = "hr"; it_operations = "it_operations"
    customer_service = "customer_service"; data_engineering = "data_engineering"
    software_engineering = "software_engineering"; document_processing = "document_processing"
    other = "other"

class Capability(str, Enum):
    summarisation = "summarisation"; extraction = "extraction"; classification = "classification"
    qa_over_documents = "qa_over_documents"; code_generation = "code_generation"
    code_review = "code_review"; ticket_automation = "ticket_automation"
    workflow_orchestration = "workflow_orchestration"; search_retrieval = "search_retrieval"
    translation = "translation"; conversation = "conversation"; monitoring = "monitoring"
    testing = "testing"; other = "other"

class NeedProfile(BaseModel):
    summary: str = Field(max_length=600)          # ≤ 60 words, model-written
    domains: list[Domain]
    capabilities: list[Capability]
    integrations: list[str]                        # constrained to registry vocabulary + "other"
    data_kinds: list[str]                          # free but short, ≤ 6 items
    constraints: list[str] = []                    # e.g. "on-prem only", "no external calls"

class RetrievalScores(BaseModel):
    lexical: float = 0.0; vector: float = 0.0; graph: float = 0.0; fused: float

class Candidate(BaseModel):
    component_id: str
    scores: RetrievalScores
    rank: int

class Fit(str, Enum):
    strong = "strong"; partial = "partial"; none = "none"

class Assessment(BaseModel):
    component_id: str
    fit: Fit
    evidence_component: str = Field(max_length=200)   # verbatim from component record
    evidence_need: str = Field(max_length=200)        # verbatim from need summary or input
    unmet_preconditions: list[str] = []
    unknowns: list[str] = []
    quotes_verified: bool = False
    from_cache: bool = False

class Recommendation(BaseModel):
    component: ComponentSummary
    fit: Fit
    score: float                    # 0..1, see §7.4
    rank: int
    rationale: str = Field(max_length=240)
    evidence_component: str
    evidence_need: str
    unmet_preconditions: list[str] = []

class NearMiss(BaseModel):
    component: ComponentSummary
    fit: Fit
    reason: str                     # "partial fit; preconditions unmet: …" or "retrieved but no fit"
    scores: RetrievalScores

class RunTimings(BaseModel):
    profile_ms: int; retrieve_ms: int; assess_ms: int; synth_ms: int; total_ms: int
    llm_calls: int; cache_hits: int; discarded_unverified: int; assess_timeouts: int

class RecommendationRequest(BaseModel):
    text: str | None = Field(default=None, max_length=8000)
    card_id: str | None = None
    component_types: list[ComponentType] = [skill, mcp, agent]
    max_per_type: int = Field(default=5, ge=1, le=10)
    # exactly one of text / card_id must be set — validator

class RecommendationResult(BaseModel):
    run_id: str
    status: Literal["queued", "running", "done", "failed"]
    catalogue_version: str
    prompt_version: str
    model: str
    input_kind: Literal["text", "card"]
    card_id: str | None
    need_profile: NeedProfile | None
    recommendations: dict[ComponentType, list[Recommendation]] = {}
    near_misses: dict[ComponentType, list[NearMiss]] = {}
    empty_reason: str | None = None          # set when every type list is empty
    timings: RunTimings | None = None
    error_code: str | None = None
    created_at: datetime
```

Registry status:

```python
class CatalogueStatus(BaseModel):
    version: str; created_at: datetime; is_current: bool
    counts: dict[ComponentType, dict[Literal["eligible", "ineligible"], int]]
    flag_histogram: dict[QualityFlag, int]
    unconsumed_keys: dict[str, int]
    link_counts: dict[str, int]              # link_type → count, plus "unresolved"
```

---

## 6. Registry ingestion (lane 02)

Inputs:

- Export: `data/agenthub-export/*.xlsx` (one or two files: PPD and Prod) **or** a directory of
  four CSVs named `agent_usage.csv`, `skill_list.csv`, `mcp_list.csv`, `knowledge_source_list.csv`.
  Sheet names as exported: `Agent Usage`, `Skill List`, `MCP List`, `Knowledge Source List`.
  Column expectations for Agent Usage: `Tenant`, `Agent Name`, `Description`, `Skill ID`,
  `MCP ID`, `Knowledgebase ID` (last three comma-separated). Other sheets: an id column, a name
  column, a description column — exact headers are confirmed in recon and recorded in `RECON.md`.
- Skills repo: sibling checkout, default `../gaip-agenthub-skills-marketplace`, skills under
  `skills/<folder>/SKILL.md`; the `skills/scripts/` folder and repo-template files are skipped.

Pipeline: `parse → normalise → resolve identity → flag → link → embed → write version`.

Rules that matter (details in the lane brief):

- Identity: skill slug = folder name lower-cased; MCP slug = slugified source id, with `#n` suffix
  on duplicates; agent slug = `tenant/slugified-name`; knowledge source slug = source id.
- Merge for skills: export row and repo folder are matched by `slug(export name) == folder` first,
  then `slug(export name) == slug(frontmatter name)`; unmatched rows keep `source` of their origin.
- Front-matter parsing is tolerant: a small explicit alias table, and **every key that is not
  consumed is counted** into `unconsumed_keys` — never dropped in silence.
- Embedding: `retrieval_text` embedded with the hub's existing embedding client, batched, cached
  on `sha256(retrieval_text)` in a local JSON cache under `data/recsys-cache/` (git-ignored) so
  re-ingestion is cheap and offline re-runs are possible.
- Output report `docs/recsys/INGEST-REPORT.md` with counts, flag histogram, unconsumed keys,
  unresolved links, merge statistics. **Structure only** — no component names or descriptions in
  the report beyond counts.

CLI: `python -m app.recsys.ingest --export data/agenthub-export --skills-repo
../gaip-agenthub-skills-marketplace [--no-embed] [--dry-run]`.

---

## 7. Recommendation pipeline (lanes 03, 04, 05)

### 7.1 S1 — need profile

One LLM call, structured output into `NeedProfile`. The `integrations` field is constrained to a
vocabulary generated at ingest time (normalised MCP names + tool names + a fixed list of common
systems) and `other`. Card input: the card's title, summary/description and capabilities fields
are concatenated (recon confirms which card fields exist and how to read them via the hub's
existing card repository — **do not** re-parse card files).

### 7.2 S2 — retrieval (deterministic)

Three signals, each producing a ranked list over eligible components of the requested types:

- **Lexical**: BM25 over tokenised `retrieval_text` (use `rank_bm25` if present in the environment;
  otherwise implement BM25 in ~40 lines — no new dependency is required). Query = need summary +
  capabilities + integrations.
- **Vector**: cosine between the embedded need summary and stored component embeddings, computed
  in memory with numpy (catalogue < 1 000). Load embeddings once per catalogue version into a
  process-level cache.
- **Graph co-occurrence**: find the top-5 agents by vector similarity to the need; every component
  those agents `USES_*` gets a graph score proportional to the agent's similarity; additionally
  skills `ALLOWS_MCP` a retrieved MCP get a smaller boost. This reads `recsys_component_link` only
  — FalkorDB is not queried in the PoC path.

Fusion: reciprocal rank fusion with `k = 60`, weights lexical 1.0 / vector 1.0 / graph 0.7.
Ties broken by `component_id` ascending so results are reproducible. Top-K per type, K = 8
(configurable `RECSYS_ASSESS_K`).

### 7.3 S3 — per-candidate assessment

For each candidate, one structured LLM call producing `Assessment`. Input: the need profile and
the single component record (name, description, triggers, tools, tags — never the embedding, never
employee identifiers). Cache lookup first. Calls run concurrently under `asyncio.Semaphore(4)`
with a 30 s timeout each; a timeout marks the candidate `assess_timeout` and it is excluded.

### 7.4 S4 — verification and scoring

- Quote verification: `normalise(q) in normalise(text)` where `normalise` lower-cases and
  collapses whitespace; `evidence_component` against the component's `retrieval_text`,
  `evidence_need` against `need_profile.summary + "\n" + input_text`. Failure → assessment
  discarded, counted in `discarded_unverified`.
- Score: `0.6 * fit_weight + 0.4 * fused_norm` where `fit_weight ∈ {strong: 1.0, partial: 0.5,
  none: 0.0}` and `fused_norm` is the fused retrieval score min-max normalised within the type.
- Include as recommendation if `fit != none` and `score ≥ 0.45` (configurable
  `RECSYS_MIN_SCORE`). Cap at `max_per_type`.
- Near-miss if `fit == partial` and excluded, or `fit == none` and retrieval rank ≤ 3.
- If every type list is empty, set `empty_reason` from a fixed set:
  `no_candidates_retrieved` | `no_candidate_fit` | `all_assessments_unverified` |
  `assessments_timed_out`.

### 7.5 S5 — synthesis (optional, bounded)

One LLM call given the included recommendations (id, name, fit, evidence) asking for an ordering
and a ≤ 30-word rationale each. Validation: the set of ids returned must equal the set given; any
deviation → keep deterministic order (by score desc) and use the assessment's
`evidence_component` as the rationale. Skipped entirely when fewer than 2 recommendations exist.

### 7.6 Prompts

Live in `backend/app/recsys/prompts/` as Python constants with a `PROMPT_VERSION = "1"` in
`prompts/__init__.py`; bump on any change. Each prompt has a **sanitised** example pair in
`tests/recsys/fixtures/` (synthetic component records only, never real catalogue text).

### 7.7 LLM gateway

`backend/app/recsys/llm/gateway.py` defines:

```python
class LLMGateway(Protocol):
    async def structured(self, *, system: str, user: str, schema: type[BaseModel], timeout_s: float) -> BaseModel: ...
    async def embed(self, texts: list[str]) -> list[list[float]]: ...
```

Three implementations selected by `RECSYS_LLM_MODE`:

- `live` — adapter over the hub's existing client(s) (recon finds them; reuse model names and
  auth exactly as the hub does; structured output via the same mechanism the hub already uses, or
  JSON mode + Pydantic validation + one retry with the validation error appended).
- `replay` — reads `tests/recsys/fixtures/replay/<sha256(system+user)>.json`; raises
  `ReplayMiss` when absent. Used by tests and by eval when offline.
- `off` — returns a canned `NeedProfile` and `fit=none` for everything; lets the UI and API be
  exercised without any model access.

`live` is also wrapped by a **recorder** when `RECSYS_LLM_RECORD=1`, which writes the replay file
for every call — this is how the eval fixtures are produced once on the workstation.

---

## 8. API (lane 05)

Router `backend/app/recsys/api/router.py`, mounted at `/api/recsys` only when
`RECSYS_ENABLED=true`. Follows the hub's existing router registration pattern and error envelope
(recon records both).

| method | path | body / params | returns |
|---|---|---|---|
| POST | `/recommendations` | `RecommendationRequest` | `{ run_id, status }` 202 |
| GET | `/recommendations/{run_id}` | | `RecommendationResult` (status may be `queued/running`) |
| GET | `/registry/status` | | `CatalogueStatus` |
| GET | `/registry/components` | `type?`, `q?`, `flag?`, `include_ineligible?`, `limit` (≤ 200), `offset` | `{ items: ComponentSummary[], total }` |
| GET | `/registry/components/{component_id}` | | `ComponentDetail` |
| GET | `/registry/vocabulary` | | `{ domains, capabilities, integrations }` |
| GET | `/cards/{card_id}/suggested-components` | | last `done` result for that card in the current catalogue version, or 404 |

Execution: `POST` creates a `recsys_run` row and schedules the pipeline as a background task
(FastAPI `BackgroundTasks` is sufficient for the PoC unless recon finds the hub's own job runner is
trivially reusable). Poll interval suggested to the UI: 1 s.

Errors: 400 for request validation (neither/both of `text`/`card_id`), 404 for unknown card or
component, 409 for `no_current_catalogue`, 503 for `llm_unavailable` (live mode, gateway raised).

---

## 9. UI (lane 06)

Feature folder `frontend/src/features/recsys/` (recon confirms the frontend's feature/page
convention and routing library; follow them). Ant Design 6 components; Tailwind only for layout
spacing where the rest of the app does the same.

Routes and states:

- `/recommend` — left: input card with a textarea (max 8 000 chars, counter) **or** a card
  picker (search existing cards by title via the hub's existing cards endpoint), type checkboxes
  (Skills / MCP / Agents), max-per-type select, Submit. Right: result panel with states
  `idle → submitting → running (spinner + elapsed) → done | failed`. Done renders the need profile
  as tags (domains, capabilities, integrations), then three tabs (Skills / MCP servers / Agents),
  each a list of `RecommendationCard`: name, fit badge (strong = green, partial = gold), score bar,
  rationale, two evidence quotes rendered as quoted blocks with the *component* quote highlighted
  inside the component description (expandable), unmet preconditions as a warning list, and a link
  to `/registry/:id`. Below: "Near misses" collapsible with reason. Empty: a distinct
  `EmptyResult` component that states the `empty_reason` in plain words and links to the registry
  browser. Footer line: catalogue version, prompt version, model, total ms, LLM calls, cache hits.
- `/registry` — table: name, type, tenants, version, flags (chips, colour by blocking/non-blocking),
  eligible; filters: type, flag, eligibility toggle (default eligible only), free-text search
  (server-side `q`); pagination; row click → detail.
- `/registry/:componentId` — detail: all fields, flags with one-line explanations, "Used by"
  agents, "Uses" (for agents), "Depends on"; a "Recommend for a use case like this" button that
  prefills `/recommend` with the component's description (demonstrates the near-duplicate check).
- `/registry/status` — catalogue version, created, counts table per type, flag histogram (bar
  list), unconsumed keys table, link counts incl. unresolved, and a short "how to re-ingest" note.
- Card page panel — `SuggestedComponentsPanel` mounted on the existing card view (recon finds the
  component and a non-invasive insertion point): shows the last result for the card, or a
  "Generate suggestions" button that submits `card_id` and polls.

Data access: a single `api.ts` module with typed fetchers and a `useRecommendationRun(run_id)`
polling hook; a `mock.ts` adapter that serves fixture JSON so the UI lane can build before the API
lane finishes. Switch by `VITE_RECSYS_MOCK=1`.

No new global state library. No employee identifiers anywhere in the UI (the API never returns
them; the UI never asks).

---

## 10. Evaluation (lane 07)

Golden set at `backend/app/recsys/eval/golden/*.yaml`, one file per case:

```yaml
case_id: g-001
kind: positive | empty_control | near_duplicate_pair | stability
input:
  card_id: UC0001234        # or text: "…"   (text cases must be synthetic)
expected:
  skill: [ "skill:…", … ]   # ids that a correct system should include
  mcp:   [ … ]
  agent: [ … ]
forbidden:                  # ids that must not appear (optional)
  mcp: [ … ]
labelled_by: draft-agent    # human name once confirmed
confidence: low | medium | high
notes: "why these expectations"
```

Draft golden set produced by the lane: 20 positive cases from cards whose linked agents (via the
Agent Usage sheet) give ground truth for MCP/skill expectations; 6 empty controls (synthetic
inputs describing needs with no plausible component, e.g. a physical-branch procedure); 5
near-duplicate pairs (two cards known to be related via the hub's `related_use_case_ids` field,
expecting overlapping recommendations); 5 stability cases (same input run 3× expecting identical
top-3). All marked `labelled_by: draft-agent`, `confidence: low` unless the ground truth is
mechanical (agent-usage links).

Metrics (`python -m app.recsys.eval.run --mode replay|live --report docs/recsys/EVAL-REPORT.md`):

- precision@k and recall against `expected` per type (k = max_per_type)
- false-positive rate on `empty_control` (fraction that returned any recommendation)
- forbidden-hit rate
- near-duplicate overlap (Jaccard of top-3 across the pair)
- stability (fraction of stability cases with identical top-3 across 3 runs)
- verification discard rate and timeout rate (from run timings)

Targets for the PoC (report, do not gate): precision@5 ≥ 0.6 on `high`/`medium` cases, empty-control
FP rate ≤ 0.2, stability ≥ 0.8, discard rate ≤ 0.1.

---

## 11. Graph projection and MCP adapter (lane 08, both optional)

- **Graph projection** (`RECSYS_GRAPH_PROJECTION_ENABLED`, default false): after ingest, MERGE
  `(:Component {component_id, component_type, name, catalogue_version})` nodes and
  `USES_*`/`DEPENDS_ON_SKILL`/`ALLOWS_MCP` edges into FalkorDB using the hub's own ingestion
  helpers (MERGE-only, matching the hub's style). Never creates a vector index; never touches
  existing labels or edges; never deletes. Purpose: make components visible in the hub's existing
  graph views next to `EMPLOYS_AGENT` edges.
- **MCP adapter**: only if `import mcp` succeeds in the backend environment. A stdio server in
  `backend/app/recsys/mcp_server.py` exposing three tools that call the service in-process:
  `check_duplicate(text) → {verdict, top_cards}` (uses the hub's existing search),
  `suggest_components(text | card_id) → RecommendationResult`, `get_card(card_id) → card`. Tool
  descriptions are written as routing prompts ("Use this when the user describes a new AI use case
  and wants to know what to reuse …"). If `mcp` is absent, the lane writes
  `docs/recsys/MCP-ADAPTER.md` describing the HTTP equivalents and stops.

---

## 12. Protections (what we keep; everything else is assumed to be handled by the platform)

1. **Identifier discipline** (§3): index-only selection, substring-verified quotes, validated
   synthesis, empty as success.
2. **No raw input at rest or in logs**: hash + length + derived profile only. Log lines carry
   `run_id`, `stage`, `component_type`, counts and durations — never text, never names of
   components (component ids are fine).
3. **No employee identifiers**: the registry never ingests owner/contributor identifier fields;
   the card reader used for `card_id` input copies only title/summary/capabilities.
4. **Bounded model usage**: K per type, one call per candidate, semaphore 4, 30 s timeout, cache
   keyed on catalogue + prompt + model versions, synthesis skipped for < 2 results.
5. **Feature-flagged and additive**: router mounted only under `RECSYS_ENABLED`; new tables only;
   one additive migration with a working downgrade; existing hub code is touched in at most three
   places (router registration, frontend route registration, card-page panel mount) and each
   touch is a single guarded line/block.
6. **Versioned everything**: every result carries `catalogue_version`, `prompt_version`, `model`.
7. **No secrets**: configuration through the hub's existing settings mechanism; nothing hardcoded;
   `.env` files never committed; the local embedding cache and export data directories are
   git-ignored.

Not done, on purpose: rate limiting, authN/Z, tenant scoping of recommendations, PII scanning of
inputs, prompt-injection hardening beyond structured outputs. The platform protects the process;
the PR description says so in one sentence.

---

## 13. Configuration keys

| key | default | meaning |
|---|---|---|
| `RECSYS_ENABLED` | `false` | mount router |
| `RECSYS_LLM_MODE` | `live` | `live` \| `replay` \| `off` |
| `RECSYS_LLM_RECORD` | `0` | record replay fixtures while live |
| `RECSYS_ASSESS_K` | `8` | candidates assessed per type |
| `RECSYS_MIN_SCORE` | `0.45` | inclusion threshold |
| `RECSYS_ASSESS_CONCURRENCY` | `4` | semaphore |
| `RECSYS_ASSESS_TIMEOUT_S` | `30` | per assessment call |
| `RECSYS_GRAPH_PROJECTION_ENABLED` | `false` | lane 08 |
| `RECSYS_EMBED_CACHE_DIR` | `data/recsys-cache` | git-ignored |

---

## 14. Repository layout of the change

```
backend/app/recsys/
  __init__.py
  contracts.py                 # §5
  config.py                    # §13, reads hub settings
  registry/
    __init__.py                # INGEST_CODE_VERSION
    excel_reader.py            # xlsx or csv → row dicts (structure-checked)
    skill_md_parser.py         # tolerant front matter + unconsumed key counting
    normalise.py               # slugs, spelling variants, trigger extraction
    identity.py                # canonical ids, duplicate suffixing, merge
    flags.py                   # §4.4
    links.py                   # §4.2
    embed.py                   # batch embed with file cache
    writer.py                  # versioned write
    report.py                  # INGEST-REPORT.md
  ingest.py                    # CLI entry (python -m app.recsys.ingest)
  retrieval/
    lexical.py  vector.py  graph.py  fuse.py  __init__.py (retrieve())
  llm/
    gateway.py  live.py  replay.py  off.py  recorder.py
  prompts/
    __init__.py (PROMPT_VERSION)  need_profile.py  assess.py  synthesise.py
  pipeline/
    profile.py  assess.py  verify.py  score.py  synthesise.py  service.py (RecommendationService)
  api/
    router.py  schemas.py (thin re-exports)  deps.py
  db/
    models.py  repository.py  migration (per hub tooling)
  eval/
    golden/*.yaml  build_golden.py  run.py  metrics.py
  mcp_server.py                # optional
  graph_projection.py          # optional

backend/tests/recsys/          # mirrors the package; fixtures/ holds synthetic data + replay
frontend/src/features/recsys/  # types.ts api.ts mock.ts hooks/ pages/ components/
docs/recsys/                   # RECON.md INGEST-REPORT.md EVAL-REPORT.md README.md MCP-ADAPTER.md
docs/recsys-build/handoffs/    # one file per lane, written by the lane, read by the lead
```

---

## 15. Definition of done (the PR is opened only when every line is true)

- [ ] `python -m app.recsys.ingest …` ran against the real export and the sibling repo on the
      workstation; `docs/recsys/INGEST-REPORT.md` is committed and shows counts per type, the flag
      histogram, and the unconsumed-key table.
- [ ] Backend test suite for `tests/recsys/` passes; the existing hub suite still passes.
- [ ] Frontend type-check and production build pass; existing frontend tests still pass.
- [ ] Lint/format for both halves pass with the repo's configured tools.
- [ ] With `RECSYS_ENABLED=true RECSYS_LLM_MODE=live`: `/recommend` returns recommendations for a
      real card; an empty-control input returns the empty state; `/registry` lists components with
      flags; `/registry/status` shows the current version; the card-page panel works.
- [ ] With `RECSYS_LLM_MODE=off`: the whole UI is navigable and every state renders.
- [ ] `docs/recsys/EVAL-REPORT.md` is committed with metrics over the draft golden set (live or
      replay, stated which).
- [ ] The migration downgrades cleanly (`downgrade` then `upgrade` on a scratch database).
- [ ] `git diff --stat main...HEAD` shows changes to existing hub files in ≤ 3 places, each guarded.
- [ ] A grep for employee-identifier field names, `.env` files, absolute home paths and raw
      catalogue descriptions across `docs/recsys/` and tests returns nothing.
- [ ] PR body follows lane 09 §5; branch pushed; PR URL reported to the user.

---

## 16. Known unknowns — recon must resolve before wave 1 starts

1. Backend ORM and migration tool; how existing tables are declared and migrated.
2. How routers are registered and how settings/feature flags are read.
3. The existing LLM client: module, call signature, structured-output mechanism, model names,
   timeout handling. The existing embedding client: module, batch size, dimension (expect 3072).
4. How a card is read by id in code (repository/service), and which fields hold title, summary,
   capabilities, related use cases.
5. Whether the hub has a reusable background job/polling mechanism worth reusing for runs.
6. Frontend: router library, feature/page folder convention, API client convention, how routes and
   navigation entries are registered, the card page component and a safe mount point, test runner.
7. Test runners and lint/format commands for backend and frontend; how tests get a database.
8. Availability of `openpyxl` (or `pandas`), `rank_bm25`, `numpy`, `mcp` in the backend environment.
9. Where the export files are on the workstation and their exact sheet/column headers.
10. The 41-vs-213 card count question: how many cards the local checkout actually ingests.
11. Whether `gh` is installed and authenticated for PR creation; the remote's default branch name.

---

## 17. Glossary (for the agent)

- **Card** — a best-practice card describing one AI use case; lives in the hub's graph.
- **Component** — a reusable building block: skill, MCP server, agent, or knowledge source.
- **Agent Hub** — the bank's internal platform where components are registered; the export is its
  metadata dump.
- **Skills marketplace repo** — the git repository holding `SKILL.md` files; the portal syncs from it.
- **Need profile** — the controlled-vocabulary description of what a use case needs.
- **Eligible** — a component with no blocking quality flag.
- **Near miss** — a retrieved component the assessment rejected or that fell below threshold.
- **Catalogue version** — a hash identifying one ingestion of the two sources.


<!-- ===== FILE: docs/recsys-build/00-LAUNCH.md ===== -->

# 00 — Launch: how the swarm runs

This file is for the **lead agent** (the Claude Code session the human starts) and for the human
who starts it. Lanes `01`–`09` are briefs the lead hands to subagents. `ARCHITECTURE.md` is the
contract every brief refers to.

---

## 1. Human pre-flight (before starting Claude Code)

1. Hub repository checked out and on a clean `main` (or the repo's default branch):
   `cd ~/bp-card-matured && git status` shows nothing to commit.
2. Skills repository checked out as a sibling: `~/gaip-agenthub-skills-marketplace`.
3. Agent Hub export files placed under `~/bp-card-matured/data/agenthub-export/` — the `.xlsx`
   file(s) as received, **or** four CSVs exported from the sheets and named `agent_usage.csv`,
   `skill_list.csv`, `mcp_list.csv`, `knowledge_source_list.csv`. Add `data/` to `.gitignore` if it
   is not already ignored (the lead checks this in wave 0 and will do it if needed).
4. Backend and frontend dependencies installed the way the repo's README says; the backend can
   reach its database; the hub's LLM gateway credentials are configured the way the hub already
   expects them (the PoC reuses the hub's client, it has no credentials of its own).
5. This pack copied to `~/bp-card-matured/docs/recsys-build/` (all eleven files). If pasting is the
   only way in, paste `ALL-IN-ONE.md` and the lead will split it.
6. Claude Code started in `~/bp-card-matured` with a permission mode that allows unattended file
   edits and the commands the build needs. Suggested: `acceptEdits` plus an allow-list for
   `python`, `pytest`, `npm`, `npx`, `git add/commit/checkout/switch/push`, `gh`. Do **not**
   allow `git push --force`, `git reset --hard`, `rm -rf` outside the working tree, or anything
   touching `.env`.

## 2. The launch prompt (paste verbatim)

```
You are the lead engineer for the build described in docs/recsys-build/ARCHITECTURE.md.
Read ARCHITECTURE.md fully, then docs/recsys-build/00-LAUNCH.md. Follow 00-LAUNCH.md §3–§7
exactly: run wave 0 yourself, then dispatch wave 1 and wave 2 lanes as subagents in parallel
using the lane briefs 01–09 as their complete instructions, review each handoff, commit per
lane, run wave 3, and finish with an open pull request. Work autonomously; do not stop to ask
questions unless a stop condition in 00-LAUNCH.md §8 is hit. Report progress in
docs/recsys-build/PROGRESS.md after every wave.
```

## 3. Lead responsibilities

The lead never implements a lane itself except wave 0 (recon + contracts) and wave 3 integration.
The lead:

1. Creates the branch: `git switch -c feat/component-recommendations-poc`.
2. Runs lane `01` personally (recon + contracts + scaffolding + stubs). Commits.
3. Dispatches wave 1 (lanes `02`, `03`, `04`, `06`, `07`) **in parallel**, each subagent given:
   the path to `ARCHITECTURE.md`, the path to `docs/recsys/RECON.md`, the path to its lane brief,
   and the sentence "You own only the paths listed in your brief's *Files owned* section. Do not
   edit anything else. Do not run git commit. Write your handoff to
   `docs/recsys-build/handoffs/<lane>.md` when done."
4. On each handoff: reads it, runs the lane's own test command, runs one read-only review
   subagent with the prompt in §6, applies review findings by sending them back to the same lane
   subagent (or a fresh one with the handoff + findings), then **stages only that lane's owned
   paths** and commits: `git add <paths> && git commit -m "<lane>: <summary>"`.
5. Dispatches wave 2 (lanes `05`, `08`) once `02`, `03`, `04` are committed; re-dispatches `06`
   with "wire to the real API" once `05` is committed (the `06` brief covers both phases).
6. Runs wave 3 (lane `09`) personally, with subagents for the mechanical parts it names.
7. Keeps `docs/recsys-build/PROGRESS.md` current: wave, lane, status, commit sha, open issues.

Model choice for subagents: implementation lanes on the fastest capable model available; the
review pass on the same or stronger. The lead itself should be the strongest model available.

## 4. Waves and dependencies

```
wave 0  lead      01 recon + contracts + stubs           → RECON.md, contracts.py, types.ts, stub router, package skeleton
wave 1  parallel  02 registry ingestion                  ← contracts
                  03 retrieval                           ← contracts (uses an in-memory fake registry in tests)
                  04 llm stages + prompts + gateway      ← contracts, RECON §LLM
                  06 frontend (phase A: mock adapter)    ← types.ts, stub router fixtures
                  07 eval harness + draft golden set     ← contracts (golden built from export/cards read directly)
wave 2  parallel  05 service + API + runs                ← 02 03 04 committed
                  08 graph projection + MCP adapter      ← 02 committed (optional lane; may end early)
        then      06 phase B: wire to real API           ← 05 committed
wave 3  lead      09 integration, real-data run, eval, verification, docs, PR
```

## 5. File ownership (no two lanes touch the same file)

| lane | owns |
|---|---|
| 01 | `backend/app/recsys/{__init__,contracts,config}.py`, `backend/app/recsys/api/router.py` (stub only), `backend/app/recsys/db/`, `backend/tests/recsys/conftest.py`, `frontend/src/features/recsys/types.ts`, `docs/recsys/RECON.md`, the ≤ 3 guarded touches in existing hub files, `.gitignore` additions |
| 02 | `backend/app/recsys/registry/**`, `backend/app/recsys/ingest.py`, `backend/tests/recsys/registry/**`, `docs/recsys/INGEST-REPORT.md` |
| 03 | `backend/app/recsys/retrieval/**`, `backend/tests/recsys/retrieval/**` |
| 04 | `backend/app/recsys/llm/**`, `backend/app/recsys/prompts/**`, `backend/app/recsys/pipeline/{profile,assess,verify,score,synthesise}.py`, `backend/tests/recsys/llm/**`, `backend/tests/recsys/pipeline/**`, `backend/tests/recsys/fixtures/**` |
| 05 | `backend/app/recsys/pipeline/service.py`, `backend/app/recsys/api/**` (replaces stub), `backend/tests/recsys/api/**` |
| 06 | `frontend/src/features/recsys/**` except `types.ts`, frontend route/nav registration touch (coordinated with 01's guarded touch), `frontend` tests for the feature |
| 07 | `backend/app/recsys/eval/**`, `docs/recsys/EVAL-REPORT.md` |
| 08 | `backend/app/recsys/graph_projection.py`, `backend/app/recsys/mcp_server.py`, `docs/recsys/MCP-ADAPTER.md`, their tests |
| 09 | `docs/recsys/README.md`, `docs/recsys-build/PROGRESS.md`, PR body |

If a lane needs a change outside its ownership, it writes the exact requested change into its
handoff under **Requests to other lanes** and continues with a local workaround (e.g. a fake).
The lead routes the request.

## 6. Review pass prompt (read-only subagent, one per lane)

```
Review the uncommitted changes under <owned paths> against docs/recsys-build/ARCHITECTURE.md
and the lane brief <brief path>. Report, in order of severity: (1) any violation of
ARCHITECTURE.md §3 (model generating ids, unverified quotes, empty result treated as error),
§12 (raw input stored/logged, employee identifiers, secrets, unguarded touches to existing
files), or §5 contracts; (2) silent failures — except/pass, broad excepts, defaults that hide
missing data; (3) tests that do not test behaviour (mock-only, no negative case); (4) anything
that would break on the real export/repo given the facts in ARCHITECTURE.md §0. Do not edit.
Output a numbered list with file:line and a one-line fix each.
```

## 7. Handoff format (every lane ends with this file)

`docs/recsys-build/handoffs/<lane>.md`:

```
# Lane <nn> handoff
Status: done | done-with-gaps | blocked
Test command: <exact command>   Result: <pass count / fail count>
Files created/modified: <list>
Contract additions: <none | field list>
Requests to other lanes: <none | list>
Known gaps: <none | list, each with why and suggested owner>
Facts learned about the repo/data (structure only): <list>
```

## 8. Stop conditions (the lead stops and writes the reason to PROGRESS.md)

- Export files missing or their headers do not match any expectation in `RECON.md` after one
  attempt at auto-detection.
- The hub's LLM/embedding client cannot be located, or calling it fails for a non-transient
  reason; the build continues in `RECSYS_LLM_MODE=off`/`replay` **and** the stop is recorded —
  the PR can still open, with the gap stated.
- A migration cannot be generated with the repo's tool.
- Any command would require credentials, `.env` edits, force-push, history rewriting, or pushing
  to the default branch.
- Existing hub tests fail after a lane's change and the cause is in the lane's own guarded touch
  that cannot be made non-breaking.

Everything else is handled inside the lane (fakes, workarounds, documented gaps).

## 9. Commit and git rules

- One branch, lead commits only, one commit per accepted lane plus small fix-up commits.
- Conventional prefixes: `recsys(registry): …`, `recsys(api): …`, `recsys(ui): …`, `recsys(eval): …`,
  `docs(recsys): …`.
- Never `--amend` a commit that has been pushed; never rebase; never touch the default branch.
- Push only in wave 3, after the definition of done is fully met.

## 10. What the human does after the PR opens

1. Pull the branch on the workstation; set `RECSYS_ENABLED=true`; start backend and frontend as
   usual.
2. Walk `docs/recsys/README.md` §"Manual test script" — ~15 clicks covering every UI state.
3. Read `INGEST-REPORT.md` and `EVAL-REPORT.md`; the numbers there are the stand-up material.
4. Replace `labelled_by: draft-agent` in golden cases you personally confirm; re-run eval.


<!-- ===== FILE: docs/recsys-build/01-recon-and-contracts.md ===== -->

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


<!-- ===== FILE: docs/recsys-build/02-registry-ingestion.md ===== -->

# Lane 02 — Registry ingestion

**Goal:** `python -m app.recsys.ingest --export data/agenthub-export --skills-repo
../gaip-agenthub-skills-marketplace` builds a versioned, flagged, linked, embedded component
catalogue in Postgres and writes `docs/recsys/INGEST-REPORT.md`. Deterministic, idempotent,
tolerant of the data defects listed in `ARCHITECTURE.md` §0, and **loud** about anything it could
not consume.

**Files owned:** `backend/app/recsys/registry/**`, `backend/app/recsys/ingest.py`,
`backend/tests/recsys/registry/**`, `docs/recsys/INGEST-REPORT.md`.

**Read first:** `ARCHITECTURE.md` §4, §6, §12; `docs/recsys/RECON.md` §Export files, §Dependencies,
§Embedding client.

**Test data rule:** every fixture is synthetic. Write your own tiny spreadsheet/CSV and SKILL.md
files under `backend/tests/recsys/fixtures/registry/`. Never copy a real row, name or description.

---

## Task 1 — Reader (`registry/excel_reader.py`)

Reads either `.xlsx` (openpyxl if available; if not, and pandas is absent too, raise
`ExportFormatUnavailable` with the message "convert the workbook to four CSVs, see README") or a
directory of four CSVs. Returns `ExportTables(agent_usage: list[dict], skills: list[dict],
mcps: list[dict], knowledge: list[dict])` with header normalisation: strip, lower, spaces→`_`.

Header expectations are a **mapping with alternatives**, e.g. `{"agent_name": ["agent name",
"agent", "name"], "skill_id": ["skill id", "skill ids", "skills"]}`. Unmatched required header →
`ExportSchemaError` listing what was found (headers only). Unrecognised extra headers → counted
into `unconsumed_keys["export:<sheet>:<header>"]`.

Tests (write first): CSV dir with expected headers parses; a renamed header that is in the
alternatives parses; a missing required header raises with the found-headers list; an extra header
is counted, not dropped.

## Task 2 — SKILL.md parser (`registry/skill_md_parser.py`)

`parse_skill_md(text: str, folder: str) -> ParsedSkill | NoFrontMatter`. Front matter = the block
between a leading `---` line and the next `---` line; parse with PyYAML if available, else a
minimal `key: value` / nested two-level parser (enough for this repo's shapes: scalars, lists,
one level of `metadata:` nesting).

Consumed keys and their canonical destinations:

| canonical | accepted spellings / locations |
|---|---|
| `name` | `name` |
| `description` | `description` |
| `version` | `version`, `metadata.version` (top-level wins if both) |
| `category` | `category`, `metadata.category` |
| `visibility` | `visibility`, `metadata.visibility` |
| `owner` | `owner`, `metadata.owner` |
| `user_invocable` | `user-invocable`, `user-invokable`, `user_invocable` (flag `spelling_variant` for the second) |
| `allowed_tools` | `allowed-tools`, `tools` |
| `allowed_mcp` | `metadata.allowed_mcp` (list; empty strings dropped and counted as `empty_allowed_mcp`) |
| `dependencies_skills` | `metadata.dependencies.skills` |
| `policy_refs` | `metadata.policy_refs` |
| `tags` | `tags`, `metadata.tags` |

**Every other key** (top-level or under `metadata`) is returned in `unconsumed: dict[str, Any]`
and later aggregated into the catalogue's `unconsumed_keys`. This is the deliberate opposite of a
silent fallback.

`ParsedSkill.body` = markdown after the front matter (used for trigger extraction only).

Tests: full front matter with both spellings; `metadata.version` only; both versions; no front
matter → `NoFrontMatter`; unknown key lands in `unconsumed`; empty `allowed_mcp` entries counted.

## Task 3 — Normalisation (`registry/normalise.py`)

- `slug(s)`: lower, non-alphanumerics → `-`, collapse, strip.
- `extract_triggers(description, body)`: sentences starting with "Use when", "Use this when",
  "Trigger", "Invoke when", "When the user"; plus lines under a heading matching `/trigger|when to
  use/i` in the body. Return ≤ 8 strings, each ≤ 200 chars.
- `split_ids(cell)`: split on `,` and `;`, strip, drop empties.
- `is_test_like(name, description)`: the `test_data` rule from `ARCHITECTURE.md` §4.4.
- `mismatched_systems(name, description)`: returns True when name and description each mention
  exactly one system from the fixed list and they differ.

Tests for each with positive and negative cases.

## Task 4 — Identity and merge (`registry/identity.py`)

`build_components(tables: ExportTables, skills: list[ParsedSkill | NoFrontMatter]) ->
list[ComponentRow]` applying `ARCHITECTURE.md` §6 rules:

- MCP rows: `mcp:<slug(id)>`; on repeated slug, `#2`, `#3`, … in order of appearance; all
  members of a duplicate group get `duplicate_id`.
- Knowledge rows: `knowledge_source:<id>`; duplicates as above.
- Agent rows: `agent:<tenant>/<slug(name)>`; tenants recorded.
- Skills: repo folders first → `skill:<folder>`; export rows matched by `slug(name) == folder`,
  then `slug(name) == slug(frontmatter_name)`; matched rows become `source=merged` with export
  description kept in `external_ids["export_description_hash"]` (not the text) if it differs;
  unmatched export rows → `skill:<slug(name)>` with `external_ids["only_in_export"]=True`;
  folders without an export match → `external_ids["only_in_repo"]=True`.
- `NoFrontMatter` → component from folder name with `missing_frontmatter`, description empty.

Tests: duplicate MCP ids get suffixes and flags; a skill matched by folder; one matched by
front-matter name; one only-in-export; one only-in-repo; two folders with the same front-matter
name → both flagged `duplicate_name_across_folders`.

## Task 5 — Flags and links (`registry/flags.py`, `registry/links.py`)

`apply_flags(rows)` sets every flag in §4.4 and `eligible`. `build_links(tables, rows)` produces
`ComponentLink` rows; targets resolved by exact id, then by slug over the component's type; an
unresolved target sets `resolved=False` and flags the source `unresolved_dependency`.

Tests: each flag has one positive and one negative fixture row; a link to an unknown id is kept
with `resolved=False`.

## Task 6 — Embedding with file cache (`registry/embed.py`)

`embed_components(rows, gateway, cache_dir)`: for each eligible row, key `sha256(retrieval_text)`;
cache file `<cache_dir>/<key>.json`; batch misses in groups of the size recon recorded; write
cache after each batch. `--no-embed` leaves `embedding=None`. Ineligible rows are never embedded
(cost).

Test with a fake gateway that returns `[float(i)] * 8` and asserts the second call does not hit
the gateway.

## Task 7 — Writer and report (`registry/writer.py`, `registry/report.py`)

`write_version(session, rows, links, inputs, unconsumed)`: computes the version hash from the
sorted input file hashes and `INGEST_CODE_VERSION`; if that version already exists, exits with
"unchanged" unless `--force`; otherwise inserts rows, links, the version row, and flips
`is_current`.

`write_report(version_row, path)`: markdown with: version, created, inputs (kind + hash + row
counts — **no file names**), counts per type (eligible/ineligible), flag histogram, unconsumed
keys table, link counts and unresolved count, skills merge statistics (matched by folder / by
name / only-in-export / only-in-repo), MCP duplicate group count and largest group size.

Test: report contains the section headings and no line longer than 120 chars containing a
component description (assert no fixture description text appears).

## Task 8 — CLI (`ingest.py`)

Argparse: `--export`, `--skills-repo`, `--no-embed`, `--dry-run` (prints the report to stdout,
writes nothing), `--force`, `--cache-dir`. Logs one line per stage with counts. Exit codes:
0 ok, 2 schema error, 3 gateway error.

Smoke test: run against the synthetic fixture directory with `--dry-run --no-embed`; assert exit 0
and the report text contains "eligible".

## Done when

- All tests under `backend/tests/recsys/registry/` pass.
- `python -m app.recsys.ingest --export <fixture dir> --skills-repo <fixture repo> --dry-run
  --no-embed` prints a report.
- Handoff lists any header names from the real export (recorded in RECON) that needed adding to
  the alternatives mapping, and the exact command the lead should run in wave 3 for the real
  data.


<!-- ===== FILE: docs/recsys-build/03-retrieval.md ===== -->

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


<!-- ===== FILE: docs/recsys-build/04-llm-stages.md ===== -->

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


<!-- ===== FILE: docs/recsys-build/05-api-and-orchestration.md ===== -->

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


<!-- ===== FILE: docs/recsys-build/06-frontend.md ===== -->

# Lane 06 — Frontend: clickable recommendation and registry UI

**Goal:** four routes and one panel, every state reachable and rendered, built first against a
mock adapter (phase A, wave 1) and then switched to the real API (phase B, after lane 05).

**Files owned:** `frontend/src/features/recsys/**` except `types.ts`; feature tests; the route/nav
registration block lane 01 created (you may edit that block only).

**Read first:** `ARCHITECTURE.md` §5, §8, §9, §12; `docs/recsys/RECON.md` §Frontend;
`frontend/src/features/recsys/types.ts`; stub fixtures under `backend/tests/recsys/fixtures/stub/`.

**Conventions:** follow whatever the app already does for pages, data fetching, and tests. Ant
Design 6 components throughout; Tailwind only where neighbouring pages use it for spacing. No new
state library. No new dependencies.

---

## Phase A — build against the mock

### Task 1 — API module and mock

`api.ts`: typed functions for the seven endpoints, using the app's fetch wrapper and base URL
convention; errors surfaced as `{ code, message }` from the hub envelope. `mock.ts`: same
signatures, serving the stub fixtures copied into `features/recsys/fixtures/` (they are
synthetic), with a simulated `queued → running → done` progression over ~2 s for
`getRecommendation`, and a special-case: input text containing `EMPTY` returns the empty result
fixture; containing `FAIL` returns `status: "failed", error_code: "llm_unavailable"`.
`index.ts` exports `api` chosen by `import.meta.env.VITE_RECSYS_MOCK === "1"`.

### Task 2 — Hooks

`hooks/useRecommendationRun.ts`: `submit(request)` → stores `run_id`, polls every 1 000 ms until
`done | failed`, exposes `{ state, result, error, elapsedMs, reset }` with state
`idle | submitting | running | done | failed`. Stops polling on unmount. Test with fake timers:
transitions in order; unmount cancels.

`hooks/useRegistry.ts`: list with filters + pagination; `useComponent(id)`; `useCatalogueStatus()`;
`useVocabulary()`.

### Task 3 — Pages

`pages/RecommendPage.tsx` — two-column layout (stacks on narrow widths):

- `InputPanel`: segmented control *Free text | Existing card*; textarea with counter (8 000);
  card picker = `AutoComplete` backed by the hub's existing card search endpoint (recon names it;
  fall back to a plain text input for the card id if none exists); checkboxes Skills / MCP
  servers / Agents (all on); `max per type` select 3/5/10; Submit (disabled when empty or
  running); a small "Try an example" link that fills a synthetic example text.
- `ResultPanel` by state: idle hint; running → `Spin` + "Assessing candidates… {elapsed}s";
  failed → `Alert` with a plain-English message per `error_code` (`llm_unavailable`: "The model
  service is not reachable. The registry still works." + link); done → `NeedProfileTags`
  (domains/capabilities/integrations as `Tag`s, constraints as text) then `Tabs` per type with
  counts in labels, each rendering `RecommendationCard` list or `EmptyResult`.
- `RecommendationCard`: name (link to detail), `FitBadge` (strong=green, partial=gold),
  `Progress` bar for score, rationale, `EvidenceBlock` ×2 (quoted; component quote highlighted
  inside the expandable full description using a simple split-and-`<mark>` on the normalised
  match), `unmet_preconditions` as a warning list, footer with `score.toFixed(2)`.
- `NearMisses`: `Collapse` listing name, fit, reason, and the three raw retrieval scores.
- `EmptyResult`: icon + the `empty_reason` mapped to a sentence:
  `no_candidates_retrieved` → "No catalogue component matched this need at all." …
  plus "Browse the registry" link.
- `RunFooter`: catalogue version, prompt version, model, total ms, LLM calls, cache hits,
  discarded quotes count.

`pages/RegistryPage.tsx` — `Table` with server-side pagination; columns name, type (`Tag`),
tenants, version, flags (`FlagChips`: red for blocking, grey for informational, tooltip with the
one-line rule from `ARCHITECTURE.md` §4.4), eligible (`Badge`). Filter bar: type `Select`, flag
`Select` (from the enum), eligibility `Switch` (default eligible only), search `Input.Search`
(debounced 300 ms → server `q`). Row click → detail. URL query params reflect filters.

`pages/ComponentDetailPage.tsx` — `Descriptions` for all fields; flags with explanations; three
lists *Used by*, *Uses*, *Depends on* as link lists; button "Recommend for a need like this" →
navigates to `/recommend?text=<description>` (prefilled, not auto-submitted).

`pages/CatalogueStatusPage.tsx` — `Statistic` cards per type (eligible / ineligible), flag
histogram as a horizontal bar list (plain divs, no chart library), unconsumed keys `Table`,
link counts, and a `Typography` block "To re-ingest, run … (see docs/recsys/README.md)".

`components/SuggestedComponentsPanel.tsx` — replaces the placeholder from wave 0: on mount calls
`GET /cards/{id}/suggested-components`; 404 → "Generate suggestions" button which submits
`{ card_id }` and uses the run hook; otherwise renders a compact three-column summary (top 3 per
type, name + fit badge) with a link to the full result on `/recommend?run_id=…` (RecommendPage
loads an existing run when `run_id` is present).

### Task 4 — Tests (phase A)

With the mock: RecommendPage submit → running → done renders three tabs and cards; `EMPTY` text →
`EmptyResult` visible with reason sentence; `FAIL` text → alert with the llm_unavailable
sentence; RegistryPage filter toggle changes the request params (assert on the mock spy);
detail renders lists; status page renders counts. Type-check passes.

## Phase B — wire to the real API (after lane 05 is committed)

### Task 5

Run backend with `RECSYS_ENABLED=true RECSYS_LLM_MODE=off` and the seeded test catalogue (lane 05
handoff names the seed command) and the frontend with `VITE_RECSYS_MOCK` unset. Fix any
contract mismatch by changing the **frontend** (the backend contract is frozen); if the backend
is wrong, write a *Request to lane 05* in the handoff. Confirm every state renders against the
real API, including the empty state (off gateway → `no_candidate_fit`).

### Task 6 — Accessibility and polish (bounded)

Every interactive element keyboard-reachable; `aria-live="polite"` on the result panel state
text; colour is never the only carrier of fit (badge text present). Nothing more.

## Done when

- Phase A tests pass and type-check/build pass; phase B verified against the real API in `off`
  mode; handoff lists any endpoint whose real shape differed from the stub fixtures.


<!-- ===== FILE: docs/recsys-build/07-eval-and-golden-set.md ===== -->

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


<!-- ===== FILE: docs/recsys-build/08-graph-and-mcp.md ===== -->

# Lane 08 — Graph projection and MCP adapter (optional, bounded)

**Goal:** two thin adapters over work other lanes did. Both are optional: each has an explicit
"stop early" condition, and stopping early with a clear note is a valid, complete outcome.

**Files owned:** `backend/app/recsys/graph_projection.py`, `backend/app/recsys/mcp_server.py`,
`docs/recsys/MCP-ADAPTER.md`, `backend/tests/recsys/test_graph_projection.py`,
`backend/tests/recsys/test_mcp_server.py`.

**Read first:** `ARCHITECTURE.md` §11, §12; `docs/recsys/RECON.md` §Dependencies; lane 02 handoff.

**Hard rules:** never create an index in FalkorDB; never touch existing labels, edges or the
hub's ingestion code; MERGE only; everything behind flags; the recommendation path never reads
the graph.

---

## Part A — Graph projection

### Task A1 — Find the hub's graph write helper

Locate how the hub's ingestion writes nodes and edges (the MERGE helper, the client/session
object, how a transaction is opened). Record `path:line` in the handoff. If there is no reusable
helper and writing would require duplicating connection handling, **stop Part A** and write the
finding into the handoff; do not build a second graph client.

### Task A2 — Projection

`project_catalogue(session, graph, version) -> ProjectionStats`:

- For each **eligible** component: `MERGE (c:Component {component_id: $id}) SET c.component_type,
  c.name, c.catalogue_version` — no descriptions, no embeddings, no tenants beyond a joined
  string, no employee identifiers.
- For each resolved link: `MERGE (a)-[:USES_SKILL|USES_MCP|USES_KNOWLEDGE|DEPENDS_ON_SKILL|
  ALLOWS_MCP {catalogue_version}]->(b)` between `Component` nodes.
- Return counts of nodes/edges merged.

Wire it into the ingest CLI as `--project-graph` only when `RECSYS_GRAPH_PROJECTION_ENABLED` is
true (edit request to lane 02 goes in the handoff — do not edit `ingest.py` yourself; expose
`project_catalogue` and let the lead route it).

Test with a fake graph object recording the queries: exactly one MERGE per component and per
link; no `CREATE INDEX` string anywhere; ineligible components absent.

## Part B — MCP adapter

### Task B1 — Availability gate

`python -c "import mcp"` in the backend environment. If it fails and the package cannot be
installed with the repo's dependency tooling offline, **stop Part B**: write
`docs/recsys/MCP-ADAPTER.md` with the three tool contracts below, their HTTP equivalents from
`ARCHITECTURE.md` §8, and a note on how a future stdio server would wrap them. Done.

### Task B2 — Server (only if `mcp` imports)

`mcp_server.py`: stdio server with three tools calling the service **in-process** (no HTTP hop):

| tool | input | output | description (this text is the routing prompt — write it carefully) |
|---|---|---|---|
| `check_duplicate` | `text` | `{ verdict, top_cards: [{card_id, title, score}] }` | "Use FIRST when the user describes a new AI use case. Returns whether a similar best-practice card already exists (NOVEL / PARTIAL / COVERED) with the closest cards. Do not call for questions about existing cards." |
| `suggest_components` | `text` or `card_id`, optional `component_types` | `RecommendationResult` (done) | "Use when the user wants to know which existing skills, MCP servers or agents could be reused for a use case. Returns only catalogue components with verified evidence; an empty list means nothing suitable exists — say so, do not invent alternatives." |
| `get_card` | `card_id` | card summary | "Use to read one best-practice card by id after `check_duplicate` returned it." |

`check_duplicate` calls the hub's existing search service (recon names it; if the call requires
the async submit/poll path, poll internally with a 20 s cap). `suggest_components` calls
`RecommendationService.submit` and awaits completion with a 90 s cap.

Tests: tool list contains exactly the three names; each tool's JSON schema validates its input;
`suggest_components` with a fake service returns the result; the empty case returns an empty
result rather than raising.

`docs/recsys/MCP-ADAPTER.md`: how to run (`python -m app.recsys.mcp_server`), a sample client
config block (structure only, no hostnames), and the note that transport, auth and distribution
are platform concerns outside this PoC.

## Done when

- Either both parts built and tested, or each stopped part has a one-paragraph reason in the
  handoff and (for B) the adapter doc exists.


<!-- ===== FILE: docs/recsys-build/09-integration-verification-pr.md ===== -->

# Lane 09 — Integration, real-data run, verification, docs, PR (wave 3, run by the lead)

**Goal:** every line of `ARCHITECTURE.md` §15 true, then an open pull request.

**Files owned:** `docs/recsys/README.md`, `docs/recsys-build/PROGRESS.md`, the PR body. The lead
may make small fix-up edits anywhere in `backend/app/recsys/**` and
`frontend/src/features/recsys/**` during integration, committing them as `recsys(fix): …`.

---

## Task 1 — Integrate (branch state check)

`git log --oneline main..HEAD` shows one commit per lane (01–08, some optional). Run the full
backend test suite and the frontend test/type-check/build. Fix red before anything else; if a
failure is in existing hub tests, bisect to the lane commit and repair the guarded touch.

## Task 2 — Real-data ingestion

```
python -m app.recsys.ingest --export data/agenthub-export --skills-repo ../gaip-agenthub-skills-marketplace
```

Expect: exit 0; `docs/recsys/INGEST-REPORT.md` written. Read it. Sanity checks against the facts
in `ARCHITECTURE.md` §0 (numbers are expectations, not gates):

- skills ≈ 290 folders in repo; merged with export ≈ 245; `missing_frontmatter` ≈ 9;
  `no_version` a majority; `folder_name_mismatch` ≈ 12; `duplicate_name_across_folders` = 2;
- MCP `duplicate_id` groups present (the largest ≈ 4);
- `test_data` non-zero on knowledge sources;
- `unconsumed_keys` non-empty (this is expected; list length is informative).

A number wildly off (e.g. zero skills merged) means a header or slug mismatch — fix in lane 02's
code via a fix-up commit, re-run with `--force`. If embedding fails for a non-transient reason,
re-run with `--no-embed`, note it in PROGRESS.md, and continue (retrieval degrades to lexical +
graph; the report must say so).

Commit `INGEST-REPORT.md` after confirming it contains no component names/descriptions and no
paths.

## Task 3 — Live smoke and fixture recording

With `RECSYS_ENABLED=true RECSYS_LLM_MODE=live RECSYS_LLM_RECORD=1`, start the backend and run
three requests via curl: one real card id, one synthetic empty control, one free text describing
a Confluence/Jira-style workflow. Confirm: done status; recommendations for the third; empty
state for the second; `timings.llm_calls` bounded by `types × K + 2`. If live is unavailable,
record the stop condition and continue in `off`.

## Task 4 — Evaluation

```
python -m app.recsys.eval.run --mode live --record --report docs/recsys/EVAL-REPORT.md
```
(or `--mode replay` if fixtures exist and live is unavailable; state which in the report header).
Commit the report and the replay fixtures **only if** they contain no raw card text — the
recorder stores prompt/response pairs, and prompts include card text for card cases. Therefore:
commit fixtures for synthetic-text cases only; for card cases keep fixtures local (add
`backend/tests/recsys/fixtures/replay/card-*` to `.gitignore` via a naming convention that lane
04's recorder applies when `input_kind == "card"`). If lane 04 did not implement that prefix,
add it now as a fix-up.

## Task 5 — Verification (all must be green; record outputs in PROGRESS.md)

1. Backend tests: full suite.
2. Frontend: lint, type-check, production build, feature tests.
3. Migration: on a scratch database, `upgrade → downgrade → upgrade`.
4. `git diff --stat main...HEAD -- ':!backend/app/recsys' ':!frontend/src/features/recsys'
   ':!backend/tests/recsys' ':!docs'` — the remaining changed files are the guarded touches;
   there must be ≤ 3 plus `.gitignore` and the dependency file (if a dependency was added).
5. Leak grep across `docs/recsys/`, `backend/tests/recsys/fixtures/` and the golden set for:
   employee-identifier field names (`owner_employee_id`, `ps id`, `contributors`), `.env`,
   absolute home paths, and any description string longer than 120 chars that also appears in the
   real export (compare hashes: compute sha256 of each real description at run time and of each
   fixture string; intersection must be empty). Write the command and its empty output into
   PROGRESS.md.
6. Manual UI walk in `off` mode (all states) and, if available, `live` mode (a real result).
   Screenshot names listed in PROGRESS.md (screenshots stay local; not committed).

## Task 6 — `docs/recsys/README.md`

Sections: What this is (three sentences); How it avoids hallucination (five bullets from
`ARCHITECTURE.md` §3); Running it (flags, ingest command, start commands, mock mode for the UI);
**Manual test script** (numbered clicks: submit example → running → done tabs → expand evidence
→ near-misses → empty control → registry filters → flag chips → detail → "recommend like this"
→ status page → card panel generate; expected observation per step); Evaluation (commands, how
to read the report, the draft-label caveat); What is deliberately not here (§1 and §12's "not
done" list); Files map (§14).

## Task 7 — Pull request

Branch pushed: `git push -u origin feat/component-recommendations-poc` (never to the default
branch). PR via `gh pr create` if authenticated, else print the compare URL and the body for the
human to paste.

Title: `Component recommendation service (skills / MCP / agents) — proof of concept`

Body structure (plain prose, no internal doc names, no people, no ticket ids):

```
## Problem
Teams building AI use cases cannot see which existing skills, MCP servers or agents they could
reuse; the catalogue metadata has duplicate ids, test rows and unvalidated skill files, so any
recommendation built directly on it would be unreliable.

## Solution
A feature-flagged recommendation service and registry browser inside the hub. The catalogue is
ingested into versioned tables with visible quality flags; recommendations are produced by
selecting from a deterministic candidate list with quotes verified against the source record,
and "nothing suitable" is a normal outcome. Nothing in existing search, novelty verdicts,
completeness scoring or card ingestion changes; the router is mounted only when the flag is on.

## How to verify
1. …numbered product steps from README's manual test script, including the empty case and the
   model-off fallback…

## Review focus
- the selection/verification rules in pipeline/verify.py and score.py
- the identity and flag rules in registry/identity.py and flags.py against the ingest report
- the three guarded touches to existing files
- the migration's downgrade

## Not in this PR
Authentication, rate limiting, tenant scoping, production rollout, CxO/MRM data — the platform's
existing protections apply; these are listed for a follow-up.
```

Append the ingest and eval headline numbers (counts and rates only) as a PR comment, not in the
body.

## Task 8 — Final PROGRESS.md and handoff to the human

PROGRESS.md ends with: PR URL, commit list, the verification outputs, stop conditions hit (if
any), and a "what to look at first" list of three items.
