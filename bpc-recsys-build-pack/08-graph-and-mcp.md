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
