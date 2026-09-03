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
