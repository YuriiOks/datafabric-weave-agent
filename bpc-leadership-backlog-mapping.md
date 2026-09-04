# BPC / AI Playbooks Hub — Leadership Backlog Mapping and Brainstorm Prep

**Owner:** Yurii Oksamytnyi
**Date:** 6 August 2026
**Status:** Internal working artifact for brainstorm and stand-up preparation — not a Confluence publication page
**Sources:** Leadership backlog and customer-feedback tables (Confluence, 29 Jul – 06 Aug 2026); canonical IDs from `architecture-improvement-review.md` (v3.1)

## 1. Orientation

About 70 percent of the leadership backlog is already covered by the v3.1 review under different names. The genuinely new work concentrates in three areas: a component-level corpus (Agent Hub assets: agents, skills, MCP servers), automated playbook creation from existing metadata repositories, and deterministic diagram extraction. Everything else maps onto IDs the review already defines.

Positioning for stand-ups: the follow-on programme in the review is the delivery vehicle for this backlog, not a competing plan. `Prod readiness` (DTAA-444, end of September) is the clearest example — Outcome C of the review is that baseline, already written.

## 2. Decoder — what Agent, Skill, and MCP mean in this backlog

The vocabulary comes from the internal Agent Hub / marketplace initiative (Elaine, 03 Aug: the solution should be linked to the agent inventory being developed as part of Agent Hub). The definitions are the standard agent-ecosystem ones, catalogued as enterprise assets:

| Term in the backlog | What it is | Engineering translation |
| --- | --- | --- |
| **Agent** | A deployed AI agent registered in the Agent Inventory (Agent Hub) with owner, capability, model, and infrastructure metadata | A running LLM-plus-tools service, catalogued as a reusable enterprise asset |
| **Skill** | A reusable packaged capability (prompt + procedure + resources) published to the marketplace (DTAA-1169) | A versioned instruction/resource bundle an agent can load |
| **MCP** | A Model Context Protocol server — a reusable tool/data connector, catalogued internally | A standard MCP server exposing tools or data behind the approved gateway |

What the three recommendation items (`Agent recommendation`, `Skill recommendations`, `MCP recommendations`) actually require: extend BPC retrieval from the use-case level to the component level. Today the platform matches: your idea resembles UC-X. The ask is: for your idea, agent A, skill S, and MCP M already exist — use them directly (Ross Stokoe, 06 Aug: reusable artifacts instead of guidance text).

This is not a new system. The graph already carries `EMPLOYS_AGENT` edges. The work is: new node types (Agent, Skill, MCPServer), metadata ingestion from the Agent Hub inventory, and the same bounded retrieval channels pointed at the new corpus — the RET-001/CARD-001 machinery over a new node population. Supporting evidence from the feedback itself (Elaine/Sofia, 03 Aug): the focus should be on agent and skills metadata rather than use cases, because similar use cases are unlikely to be exact matches. Component-level matching is where retrieval precision actually lives.

## 3. Ongoing Monitoring (Top priority) — a five-idea maturity ladder

The gap being closed: today the duplicate check is user-initiated — it only fires when someone asks. Prevention means the platform watches continuously. The motivating scenario from Marcin: a developer finds an existing prototype, builds a parallel one anyway, and nothing records that this happened.

The five ideas are not competitors — they form a ladder. Each rung catches what the previous one missed.

| # | Idea | Role | Integration point | Catches |
| --- | --- | --- | --- | --- |
| 1 | Draft-time copilot | Good cop | Ideation form / intake draft | Duplication before it hardens; measures whether advice changes behaviour |
| 2 | Submission-time gate | Bad cop | AIRCO/AIG submission (SNOW or GAIP process) | Duplicates at the formal boundary, with forced justification |
| 3 | Continuous corpus re-scan | Monitor | Nightly job over the existing graph | Convergence between in-flight initiatives that passed earlier checks |
| 4 | Ignored-advice ledger | Auditor | Verdict + evidence log, governance councils | The exact scenario Marcin described — advice shown, divergence chosen |
| 5 | Estate monitoring | Outside-in | Agent Inventory / GAIP registrations / SNOW demand / repo creation | Initiatives that never asked the platform at all |

**Idea 1 — Draft-time copilot (good cop).** At ideation, a non-blocking advisory: similar existing use cases, reusable agents/skills. Logged with the TEL-001 principle (category and IDs, never raw query text). The measurable question: did teams that saw the hint change course? That metric is the business case for the rest of the ladder.

**Idea 2 — Submission-time gate (bad cop).** Every new use-case submission automatically runs the matcher; a PARTIAL/COVERED verdict attaches a duplication report to the AIG record and requires a divergence-justification field before approval proceeds. This is §5.12 (structured AIRCO draft matching) wired into the submission system. David Rice proposed exactly this shape (05 Aug), including the acceptance that initial friction drives users to seek guidance earlier. Deterministic, auditable, MRM-friendly.

**Idea 3 — Continuous corpus re-scan.** A nightly job recomputes pairwise similarity over new/changed use cases and component metadata — productizing the `SIMILAR_TO` edges that already exist — and emits a convergence report: clusters of overlapping in-flight initiatives, notified to owners and AIG. Catches drift that slipped past a point-in-time check.

**Idea 4 — Ignored-advice ledger.** When a team is shown an existing prototype (verdict + evidence) and proceeds anyway, record that fact. Periodic report to governance: N teams proceeded despite similarity above threshold, with links. Sold as a reuse-leakage metric for councils, not as automated enforcement — enforcement framing would push users to route around the tool. Tracked at use-case-ID level, never at query-text level (PRIV-001).

**Idea 5 — Estate monitoring (outside-in).** Monitor the exhaust of the estate rather than submissions: new registrations in the Agent Inventory, GAIP model registrations, SNOW demand tickets, repository creation. Each event runs through the matcher; alert when something appears that never went through any duplicate check. The truest prevention and the heaviest integration — propose as a pilot with one source (Agent Inventory is the most accessible and the most aligned with the component pivot).

## 4. DIAG-001 — deterministic diagram extraction from PPTX (sketch)

The complaint: PowerPoint decks degrade in ingestion and diagrams cannot be reconstructed; the wish is interactive, animated diagrams. The right architecture is extraction at ingestion time, not diagram understanding at query time.

- **PPTX is XML, not pixels.** Shapes, connectors, groups, z-order, and text are structured OOXML. A deterministic extractor (python-pptx level) recovers a graph from most architecture slides: boxes become nodes, connectors become edges, grouping becomes containment. No LLM in the primary path.
- **Pipeline:** PPTX → typed intermediate representation (JSON: nodes, edges, groups, labels, source anchor) → rendered as a self-contained interactive HTML artifact per card. Animation is cheap presentation glaze once the IR exists.
- **The hidden jackpot:** the IR also lands in the graph, making diagrams queryable and linkable to each other. The CiB AIG leads asked for precisely this (03 Aug): different architectural diagrams do not link together without a proper extraction tool. The IR is that linking layer. The same IR feeds `Automate Playbook Creation` (pre-filling playbooks from existing artifacts).
- **Honest caveats, stated up front:** real decks are messy — detached connectors, diagrams pasted as images, SmartArt. The pipeline is deterministic-first, with a vision-model fallback through the approved gateway for image-only diagrams (output marked lossy/unverified), and a human-confirm step before the IR is trusted.
- **Proposal shape:** a one-week spike on ten real decks with an extraction-fidelity metric, then an evidence-gated decision. No product commitment before the spike numbers exist.

## 5. Backlog mapping table

| Backlog item | Priority | Review ID(s) | Verdict | Notes |
| --- | --- | --- | --- | --- |
| Playbooks Scope Extension (~90 → all PoC/Pilot/Prod) | Top | §7 scaling ladder, DATA-001 | Covered as method | Scale evidence ladder is written; corpus onboarding is the new work |
| Ongoing Monitoring (prevention) | Top | §5.12, PROD-001, TEL-001 | Core covered | Five-idea ladder above is the brainstorm deliverable |
| Operating Model (AIG process, SNOW or GAIP) | Top | §1.2 MRM gate, §5.11 review workflow | Partial | Process integration itself is new; the governance skeleton exists |
| Agent recommendation | H | RET-001 + CARD-001 machinery | New corpus, same architecture | Needs Agent Hub inventory access |
| Skill recommendations (DTAA-1169) | H | RET-001 + CARD-001 machinery | New corpus, same architecture | Marketplace metadata schema needed |
| Enhanced report (artifacts over guidance text) | M/H | RET-001, §5.4 | Partial | Component-level result rendering |
| Data product recommendations (DTAA-1168) | M/H | — | New | Governed data products as a further node type; owners and access paths |
| MCP recommendations | M/H | RET-001 + CARD-001 machinery | New corpus, same architecture | Catalogue source TBC |
| Expert routing | L/M | REV-001, owner metadata | Partial | People-graph on top of existing ownership data |
| Architecture library | L/M | CARD-001 (schema discipline) | New | Reference patterns as a first-class corpus |
| Prod readiness (DTAA-444, end of September) | H | §3.5, Outcome C entire | Covered | The review is the baseline document for this item |
| Automate Playbook Creation | H | — (candidate APC-001) | New, large | Feeds on DIAG-001 IR + governance artifacts (Group Infra WG, 05 Aug) |
| CxO Integration (DTAA-1137) | M/H | — | New | CxO data assets as baseline input |
| Enhanced scoring mechanism (DTAA-449) | M | EVID-001, CAL-001 | Covered almost verbatim | Metadata-repository-informed quality scoring |
| Playbook Self Service (DTAA-1136) | L/M | CARD-001, §5.10 | Partial | In-app authoring on top of the validated schema |
| HSBC search and retrieve capability (Helios etc.) | M/H | §7, RET-001 | Covered as pattern | The bounded pipeline is the exportable pattern |
| AI Playbooks Hub Documentation | M/H | DOC-002 | Partial | In-app roadmap visibility; Tech Airco roadmap ask (06 Aug) |

## 6. Customer-feedback digest — five themes

1. **Integration over standalone (the loudest theme).** Josh/Mervin (GAIP end-to-end lifecycle), Dave/Lucy (Group AI platform first), Sofia/Elaine/Stephen (inside the enterprise process), David Rice (SNOW analyzer). Nobody wants another portal.
2. **Component-level reuse over use-case matching.** Doug and the AIG team (real value is component-level reuse), Elaine/Sofia (agent and skills metadata over use cases). Confirms the Section 2 pivot.
3. **Single version of truth for metadata; generate, do not ask.** Elaine (align with the Agent Hub inventory; do not ask delivery teams for more documentation — generate playbooks from existing governance artifacts), Group Infra WG (UC0004442 MRM artifacts and AI Review Council minutes as sources).
4. **Strategic signals.** James Fisher (29 Jul): the AI use case concept may disappear with IT service releases — the durable unit of the corpus is the component, which strengthens theme 2. David Rice: expand beyond data AI to all AI use cases organization-wide.
5. **Quality and trust of the knowledge base.** Roshini (strategic-alignment scoring, process-mapping requirements at ideation, document uploads with standardized templates), Dave/Lucy (data completeness and convergence strategy).

## 7. Tensions and open questions for Marcin

1. **SNOW vs Group AI platform for the gate.** David Rice suggested a ServiceNow analyzer (05 Aug); the Dave/Lucy group agreed to prioritize the Group AI platform over ServiceNow (04 Aug). Which system is the system of record for the submission gate (Idea 2)?
2. **The durable corpus unit.** If the AI use case concept can disappear (James Fisher), should new ingestion investment target components (agents, skills, MCP servers, data products) as the primary node type, with use cases as one lens over them?
3. **Access prerequisites.** Agent Hub inventory API and metadata schema; DTAA Jira visibility (DTAA-444/449/1136/1137/1168/1169 are authentication-walled in the shared table); knowledge-layer alignment contact (Annmarie).
4. **Ownership of the divergence-justification policy** for the bad-cop gate: AIG, MRM, or the platform team?
5. **Sequencing of scope extension** (~90 pilot → all PoC/Pilot/Prod → beyond data AI): which expansion happens before September production readiness?

## 8. Candidate IDs for a future v3.2 (only if adopted)

| Candidate | Scope | Feeds from |
| --- | --- | --- |
| MON-001 | The five-rung monitoring ladder as one programme with per-rung gates | §5.12, TEL-001, PRIV-001 |
| COMP-001 | Component-level corpus: Agent / Skill / MCPServer / DataProduct node types + inventory ingestion | RET-001, CARD-001 |
| APC-001 | Automated playbook creation from metadata repositories and governance artifacts | COMP-001, DIAG-001 |
| DIAG-001 | Deterministic PPTX diagram extraction: typed IR + interactive HTML render + graph linking | One-week spike first |

The v3.1 review stays untouched until Marcin adopts any of these; adopted items enter as a v3.2 delta with the same evidence-gated discipline as the rest of the backlog.



Structural analysis of the AI Playbooks Hub repository. I need the shape of the code
and the data, not the code itself. Do not paste source, card content, or real use-case
text into the report.

1. Completeness scoring
   - Does completeness scoring live in this repository at all? If yes, give the module
     path and the entry point. If no, say so explicitly and report any evidence of where
     it runs instead (API calls, config, service names).
   - What is the input contract: which field, on which entity, read through which parse
     path. Give the field name and its expected type/shape.
   - How is a zero produced? Enumerate every code path that can result in a completeness
     score of zero, and state for each whether the input was empty, unparsed, unmatched,
     or absent.
   - Is the score persisted with any provenance — number of items parsed, catalogue or
     taxonomy version, source field, computation timestamp? List what is stored alongside
     the score and what is not.
   - When does scoring run: at ingestion, on demand, on a schedule, or on card update?
     Is there a recompute trigger when a card changes after its first score?
   - Is there a standard taxonomy or reference list that completeness is measured against?
     Where does it live and is it versioned?

2. Card schema
   - What front-matter fields does the ingestion parser actually read from a card?
     Give the full list with types.
   - Is the parser a fixed field list or does it accept arbitrary keys? Quote the
     mechanism, not the code.
   - What happens to a field present in the file but unknown to the parser — dropped,
     stored, or an error?
   - Is there any schema validation of cards at ingestion, and what does it do on failure?

3. Graph schema
   - List every node label and every edge type the ingestion writes.
   - Are labels and edge types a hardcoded enumeration, or derived from data or config?
     This determines whether adding a component node type is a config change or a code
     change — answer that question directly.
   - How are properties written onto nodes: fixed set or open?

4. Card corpus statistics
   - Total number of cards.
   - For each front-matter field, the number and percentage of cards populating it.
   - Distribution of card body length, so we can see what share are substantially
     populated versus stubs.
   - How many cards carry any description of the components, tools, agents or services
     the use case was built from. This is the single most important number in the report.

5. One specific card
   - For use case uc0003818: does it exist in the repository, and what is present in the
     capabilities field — its shape, item count, and whether it parses. Describe the
     structure only. Do not reproduce the content.
   - Pick two cards with a healthy completeness score and compare the shape of their
     capabilities field against uc0003818. Report whether the shapes differ.

6. Search
   - There is a feature flag selecting a deterministic search path instead of the default.
     Confirm the flag name, what it switches between, and its default value.
   - Is input validated before a novelty verdict is produced? If so, where and what does
     it check?

Output a markdown report. Field names, paths, counts, percentages and flag names are
fine to include. Card content, source code and real use-case text are not.







  Нажми последний Copy to Clipboard (base64 «user:password» — Nexus уже посчитал) и в bash-терминал одной строкой, вставив значение вместо PASTE:


⏺️ Demo polish pass on the registry UI. Dev server stays up; work in small commits, tsc and
  eslint on src/features/recsys after each group, then tell me when the dev server has
  reloaded. No backend changes unless an item needs a field the API already has.

  Status page (/registry/status)
  1. Sort the quality-flag histogram by count, descending.
  2. Colour blocking flags red and informational flags grey — same palette as the list —
     and add a one-line legend under the heading: "Blocking flags exclude a component from
     recommendations; informational flags do not."
  3. Collapse "Unconsumed front-matter keys" by default; put the count in the heading.
  4. Add a one-sentence summary under the version line, computed from the counts:
     "2,232 components across four types; 1,499 eligible for recommendations."
  
  Registry list (/registry)
  5. Add a summary line above the table: total components, eligible count, and the range
     shown on this page (e.g. "Showing 1–25 of 2,232 · 1,499 eligible").
  6. Humanise flag labels everywhere (empty_description → "Empty description"); keep the
     raw key as the tooltip. Same for type labels if any are still raw.
  7. Make the whole row clickable, not only the name.
  
  Component detail (/registry/:id)
  8. De-duplicate "Used by", "Uses" and "Depends on" by component id — an agent that
     references two tools of one MCP server currently appears twice.
  9. Humanise "Source reference": "sheet=Agent Usage;row=1513" → "Agent Usage sheet,
     row 1513"; "skills repo" references likewise.
  10. Add a "Back to registry" link above the title that preserves the previous filters.
  11. Hide rows whose value is "—" behind a "Show empty fields" toggle, off by default, so
      the card reads as five or six populated rows.
  
  Recommend page (/recommend), off mode
  12. Check the empty state copy. It should say plainly that the model is not configured in
      this environment and that the registry is still browsable, with a link to /registry.
      No stack traces, no internal reason codes on screen — keep those in the details
      expander.
      
  When done: commit as recsys(ui): demo polish (one or several commits), run the frontend
  tests for the feature, and report what changed and anything you chose not to do.
  

 
