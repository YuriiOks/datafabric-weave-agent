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
