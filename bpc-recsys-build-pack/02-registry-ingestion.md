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
