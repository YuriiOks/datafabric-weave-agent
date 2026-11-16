# PR test plans: one per pull request

Every pull request carries its own test plan: a small YAML file that says how a person would check what this pull request changes. An agent reads the plan, walks the journeys in the running application, and reports pass or fail per journey with screenshots. A person can read the same file as a checklist.

A PR test plan covers one pull request, not the whole feature. It does not replace automated tests.

## 1. Where the plan lives

| Rule | Detail |
|---|---|
| One folder per pull request | `.github/testplans/<pr-number>/plan.yaml` |
| Folder name | The pull request number and nothing else |
| `pr:` inside the file | Must equal the folder name |
| Template folders | Names starting with `_` (for example `_example`) are not tied to a pull request |
| Own folder only | A pull request changes its own folder and `_` folders, never the folder of another pull request |
| After merge | The folder stays as the record of how that pull request was checked |

The path follows from the number alone. Nobody searches for the plan, and two runs on the same pull request read the same file.

## 2. Two shapes

### 2.1 Nothing a user can see

For a change with no screen to walk (a refactor, a pipeline, documentation), the plan is three lines:

```yaml
pr: 87
title: Role lookup moved into one module
no_ui: backend refactor only; no screen or API response changes
```

`no_ui` must say why there is nothing to walk. An empty reason is rejected.

### 2.2 Journeys

For a change a user can see, the plan lists ordered journeys. At most six. More than six means the plan describes the feature and not the pull request.

```yaml
# Workspace creation with roles: a signed-in user creates a workspace,
# a user without the role is refused. Checked by walking the three
# journeys below in the running application.
pr: 88
title: Create a workspace with role checks
owner: knowledge-team
covered_by_e2e:
  - tests/e2e/test_workspace_create.py
journeys:
  - name: owner-creates-workspace
    goal: A domain owner creates a workspace and sees it in the list
    ship_gate: true
    steps:
      - goto: /workspaces
      - click: New workspace
      - fill: Name => Legislation review
      - press: Enter
      - expect: the new workspace is in the list with the owner shown
      - screenshot: workspace-created
  - name: reader-is-refused
    goal: A user without the creator role cannot create a workspace
    steps:
      - do: sign in as a user who has only the reader role
      - goto: /workspaces
      - expect: the New workspace button is absent or disabled
      - do: send the create request directly, without the screen
      - expect: the request is refused and a readable message is shown
      - screenshot: refused
  - name: session-ends-cleanly
    goal: An expired session sends the user to sign-in and loses no saved work
    skip_if: the covering automated test exists and passes on this branch
    steps:
      - do: let the session expire while the workspace page is open
      - click: any workspace in the list
      - expect: the sign-in step appears, no blank page, no raw error text
      - screenshot: session-expired
fixtures:
  - one user with the domain owner role and one with only the reader role
checks:
  - no errors in the browser console on any journey
  - layout intact at 1440px width
report:
  comment: one pull request comment with pass or fail per journey and screenshots
```

## 3. Schema

| Field | Required | Meaning |
|---|---|---|
| `pr` | yes | Pull request number, equal to the folder name |
| `title` | yes | What this pull request changes, in plain words |
| `owner` | no | Team that owns the change |
| `no_ui` | one of the two | Reason there is nothing to walk |
| `journeys` | one of the two | Ordered scenarios, at most six |
| `covered_by_e2e` | no | Paths of automated tests that already cover the area; each path must exist |
| `fixtures` | no | Data the journeys assume; if it is missing the journey is reported as blocked |
| `checks` | no | Things that must hold on every journey |
| `report` | no | What the runner publishes |

Each journey:

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | Short id in kebab-case |
| `goal` | yes | One sentence: what a pass proves |
| `steps` | yes | Ordered steps, using only the verbs below |
| `ship_gate` | no | `true` (bare, not quoted) means the runner may never skip this journey |
| `skip_if` | no | Condition for skipping; allowed only when `covered_by_e2e` is not empty |
| `selectors` | no | Test ids the runner should prefer |

Step verbs, and no others:

| Verb | Use |
|---|---|
| `goto` | Open a path |
| `click` | Click something, described in words |
| `fill` | `field => text to type` |
| `press` | Press a key |
| `expect` | What must be visibly true |
| `do` | Any other action, described in words |
| `screenshot` | Name of the screenshot to take |

Format: two-space indent, no anchors, no aliases, no multi-line values.

## 4. Rules

1. The plan goes into the same push that opens the pull request, not into a later commit.
2. Write journeys for what the pull request changes, not for the whole feature.
3. No ticket ids anywhere in the file. The ticket belongs in the pull request title.
4. If an automated test already covers a journey, list the test under `covered_by_e2e` and mark the journey with `skip_if`. The runner checks that the test file exists before skipping.
5. `no_ui` and `journeys` never appear together.
6. Verdict per journey is one of: PASS, FAIL, BLOCKED (a fixture is missing), SKIPPED (covered by an automated test).
7. When a journey fails, the developer who fixes it also adds an automated test at the lowest level that catches the bug. The plan is not a substitute for that test.
8. Desktop width (1440px) is checked by default. A narrow width is checked only when the plan asks for it in `checks` or in a step.

## 5. How a plan is run

1. The runner takes the pull request number and opens `.github/testplans/<pr-number>/plan.yaml`.
2. It starts or opens the application built from that branch.
3. It walks each journey step by step and takes the named screenshots.
4. It skips a journey only when `skip_if` holds and the covering test file exists.
5. It publishes one result table: journey, verdict, one line of evidence, screenshot.

## 6. What to decide before using this

| Question | Suggested answer |
|---|---|
| Folder location | `.github/testplans/` as above, unless the repository owner prefers `docs/testplans/` |
| Who runs the plan | The author's agent, before asking for review |
| Where the result goes | The pull request description, under a screenshots heading |
| Is it enforced | Start as a convention; add a check in the pipeline later |
