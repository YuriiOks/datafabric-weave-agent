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
