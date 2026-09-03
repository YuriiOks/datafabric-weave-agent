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
