# PROTO — Multi-Agent SDLC Protocol

A project-independent protocol for running a software project with multiple
specialized agents, one stage of the SDLC each. Set it up in a repo with
`ai_sdlc/bin/sdlc-init <project-dir>`. That creates `.sdlc/` with a pinned
copy of this file. Then fill in `.sdlc/config.yaml` (section 9), and the
agents follow it.
All coordination files live in `.sdlc/` (section 6), apart from the project's own files.

---

## 1. Core principles

1. **One agent, one role, one context.** Each agent keeps its own context and
   never relies on another agent's memory.
2. **Artifacts are the only shared memory.** Agents talk to each other only
   through versioned files in the artifact store (section 6). If it isn't
   written down, the next agent can't see it.
3. **Every stage has a gate.** Work moves forward only when the stage's exit
   criteria pass. Failing a gate sends work back, with a written reason.
4. **Least privilege.** Each agent can touch only what its role needs (QA can't
   edit code, devs can't deploy, only DevOps promotes environments).
5. **Humans approve the irreversible.** Production deploys, schema migrations
   on live data, turning a feature flag on in prod, and anything that costs
   money or reaches users need explicit human approval. Scope sign-off is
   sized to the item (3.4).
6. **Bounded loops.** Every feedback loop has a retry limit. When it runs out,
   the work escalates to a human; agents don't loop forever.
7. **Learn once.** Repeat mistakes go into `.sdlc/knowledge/lessons.md` and every later agent
   reads it before starting.
8. **Ship thin, complete slices.** Requirements are never fully known up
   front. Every item is a small, functionally complete slice that goes all
   the way to prod, with process sized to the item (3.3, 3.4).
9. **The human steers after every release.** What to build next is decided
   from what just shipped, not from a plan made at the start (3.5, 3.6).

---

## 2. Agent roster

### 2.0 Control plane (spans all stages)

| Agent | Responsibility |
|---|---|
| **Orchestrator** | Owns the backlog and the state machine. Picks the next item, sends it to the right stage, enforces gates and retry limits, escalates to a human. Never writes product code. |
| **Reviewer** | Independent code/design review at every gate. Runs in a fresh context and never reviews its own work. |
| **Security & Compliance** | Threat model at design, SAST/dependency/secret scan at dev, config review before prod. Can block any gate. Required for regulated domains (payments, PII, KYC). |

### 2.1 Requirements

| Agent | Input → Output |
|---|---|
| **BD / Market** | Brief, user feedback, release-review replies, Monitor insights → `board/roadmap.md` (themes, epics, one-line candidates), refreshed after every release. For the top candidates, just in time: `opportunity.md` (problem, user, evidence, proposed slice), plus market analysis only for new product bets |
| **Triage** | Bug reports, prod incidents, QA escapes → `bug-<id>.md` (repro, severity, affected area). Bugs take the **fast lane** (section 4.3) and skip market analysis. |

### 2.2 Design

| Agent | Input → Output |
|---|---|
| **PRD** | `opportunity.md` → sizes and slices the item (3.4). S → `story.md` (story + ≤3 testable ACs); M → `prd.md` (goal, non-goals, stories, ≤5 testable ACs, success metric); too big → an `EPIC` of slices. Lists the design `triggers`. *Human sign-off for the sizes in `human_approvals.spec`.* |
| **Data Requirements** | `prd.md` → `data-req.md`: entities, fields, ownership, retention, PII classification |
| **Data Flow / Architecture** | `prd.md`, `data-req.md` → `architecture.md` + `api-contract` (OpenAPI/proto): components, sequence/data-flow diagrams, integration points, ADRs for big decisions |
| **UX** | `prd.md` → `ux-flows.md`: user journeys, states (empty/loading/error), accessibility |
| **UI / Screens** | `ux-flows.md` → `screens/`: screen specs or mockups for every state |

> Design agents run **only when triggered** by the slice (3.4). When several
> are, the order is PRD → (Data Req ‖ UX) → (Data Flow ‖ UI). `‖` = can run in parallel.

### 2.3 Development

| Agent | Input → Output |
|---|---|
| **Schema / DB** | `data-req.md`, `architecture.md` → migrations (forward + rollback), seed data. Runs first. |
| **Backend** | `api-contract`, schema → service code + unit/integration tests |
| **Frontend – Web** | `api-contract`, `screens/` → web app + component tests (against a contract mock) |
| **Frontend – App** *(optional)* | same as Web, for mobile. Enabled in `.sdlc/config.yaml`. |
| **Docs** | All of the above → README, API docs, runbook updates |

> Once `api-contract` is frozen, BE and FE work in parallel. Changing the
> contract mid-build sends the item back to Design.

### 2.4 Testing (QA): a gate, not a stage

| Agent | Responsibility |
|---|---|
| **QA** | Builds the test plan from the PRD acceptance criteria, then runs functional, regression, and e2e tests **in each environment**. Writes `qa-report-<env>.md` with a pass/fail per criterion. Read-only on code. |

### 2.5 Deployment and release

| Agent | Responsibility |
|---|---|
| **DevOps** | CI/CD, infrastructure, promotion `dev → uat → stage → prod`, rollback. The only agent allowed to deploy. |
| **Release** | Release notes, CHANGELOG, version tag, and the **release review** for the human (3.6). |
| **Monitor / SRE** | Watches prod after release (errors, latency, business KPIs vs PRD success metrics). Problems go to Triage, insights go to BD. **This closes the loop.** |

---

## 3. Lifecycle and delivery model

```
BACKLOG → REQ → DESIGN → DEV → REVIEW → QA@dev
        → UAT  (deploy → QA@uat)
        → STAGE(deploy → QA@stage)
        → PROD (human approve → deploy → smoke → monitor)
        → RELEASED → (feedback) → BACKLOG
```

QA is **interleaved**, not a single stage at the end: every promotion is
`deploy → QA → gate`.

This is the path of **one small item**, not of the whole product. REQ and
DESIGN are minutes of work for a typical S slice (3.4). Many items flow
through it one after another, and each release feeds the next choice (3.3–3.6).

### 3.1 Transitions on failure

| Failure | Goes back to | Notes |
|---|---|---|
| Code bug found by QA (any env) | DEV | Defect ticket attached; environment is rolled back if needed |
| Acceptance criterion untestable or ambiguous | DESIGN (PRD agent) | Not a dev problem, so don't bounce it to dev |
| Design infeasible / contract change needed | DESIGN | Reviewer or dev raises `design-issue.md` |
| Requirement wrong / out of date | REQ | Rare; Orchestrator confirms with a human |
| Security blocker | Owning stage | Security agent names the owner |
| Prod smoke/monitor failure | DevOps **rollback** first, then Triage → fast lane | Restore service first, fix second |

### 3.2 Retry budget

- Each item gets up to **3** DEV ↔ QA round-trips per environment.
- After that, the Orchestrator escalates to a human with a summary of all the
  attempts. Repeated rework often means the slice is too big, so the
  escalation proposes a split.

### 3.3 Delivery model: iterative, thin slices

The lifecycle above runs **per item, and items are small**. The protocol is
built for products whose requirements are *not* locked. Nobody, including
the human, knows the whole product up front. You learn what to build by
shipping small, working increments and looking at them.

- **Every item is a thin vertical slice.** It is functionally complete for
  its user (a real person can do one real thing end to end), shippable to
  prod on its own, and within the size limits of 3.4. "Backend for X" or
  "screens for X" are not items. "A shop owner can add one product with a
  name and price" is.
- **Ceremony is sized to the item.** Small items get a short story and go
  straight to dev. Design agents run only when the slice touches their area (3.4).
- **Walking skeleton first.** When `delivery.walking_skeleton` is on, a
  project's first item (`CHORE-0001-walking-skeleton`) takes a trivial
  "hello world" through the real pipeline all the way to prod. That proves
  build, deploy, rollback and the status view before any feature work.
- **Refine just in time.** Only the next `delivery.refine_ahead` items are
  refined in detail. Everything else stays a one-line candidate on the
  roadmap until it gets close (3.5).
- **The human steers after every release.** Each release ends with a short
  release review for the human, and the reply decides what comes next (3.6).
- **Bigger ideas become epics of slices.** Work that doesn't fit one slice
  becomes an `EPIC` split into slices. Unfinished parts that would be
  visible to users ship dark behind feature flags (3.7).
- **Unknowns get spikes, not guesses.** If nobody knows how to build it, or
  whether it's worth building, run a time-boxed `SPIKE` first (3.8).

### 3.4 Item sizes and tracks

The PRD agent sizes every item before design starts. The size picks the
track: which stages run, and which human approvals apply.

| Size | Limits (`config.yaml` → `delivery.sizes`) | Spec | Track |
|---|---|---|---|
| **S** | ≤ 3 acceptance criteria, ≤ `max_tokens`; no new entity, contract or trust boundary | `20-design/story.md` (story + ACs, ~10 lines) | story → *triggered design only* → dev → review → QA@dev → uat → stage → prod |
| **M** | ≤ 5 acceptance criteria, ≤ `max_tokens` | `20-design/prd.md` (short PRD) | prd → *triggered design* → dev → review → QA@dev → uat → stage → prod |
| **too big** | anything over M | — | not allowed as an item. The PRD agent turns it into an `EPIC` with S/M slices (3.7) |

**Design agents are triggered, not mandatory.** For every size, a design
agent runs only when the slice touches its area:

| Agent | Runs when the slice… |
|---|---|
| data-req, schema | adds or changes stored data |
| data-flow | adds or changes an API contract, component or integration |
| ux, ui | adds a screen or changes a user flow (not for copy or style tweaks) |
| security (threat model) | adds a trust boundary, auth/permission logic, or touches PII/financial data (`compliance` ≠ none makes this stricter) |

The PRD agent lists the triggers it found in the spec's front matter
(`triggers: [data, contract, ui, security]`, or `[]`). The Orchestrator
routes by that list. A reviewer who finds a missed trigger sends the item
back to DESIGN.

**Definition of Ready** (REQ → DESIGN gate): the item is a slice (one user,
one outcome), within its size, has testable ACs (or will have them after
the story/PRD), names its triggers, and is in the top `refine_ahead` of the
backlog.

### 3.5 Backlog, roadmap and WIP

- **`board/roadmap.md`** (written by BD) holds the direction: themes, epics
  and one-line candidate items, in rough priority order. It's cheap to
  change, and is expected to change after every release.
- **`board/backlog.md`** (written by the Orchestrator) holds only items that
  are ready or nearly ready. BD and PRD refine the top `refine_ahead`
  candidates from the roadmap into backlog items, just in time.
- **Re-rank after every release.** The Orchestrator re-orders the backlog
  from the human's release-review reply (3.6), Monitor findings and new
  `inbox/` requests. Bugs with Sev1/Sev2 go to the top.
- **Limit work in progress.** At most `delivery.wip_limit` items may be
  between DEV and PROD at once (default 1). Finish before starting:
  the Orchestrator doesn't pull a new item into DEV while the limit is reached.

### 3.6 Release review with the human

After every prod release, the release agent writes
`70-release/release-review.md` and a matching
`outbox/<ts>-<ID>-release-review.md`:

```markdown
## Shipped        <what a user can now do, in one or two sentences> (v<X.Y.Z>)
## Try it         <URL / command and 3–5 steps to see it working>
## Cost           <tokens for this item, rework share>
## Learned        <surprises, Monitor findings, lessons.md entries added>
## Proposed next  <top 3 backlog items with one line each on why, plus any epic progress>
## Questions      <decisions only the human can make>
```

The human replies in `inbox/` (`type: answer`). They might keep the order,
re-rank, add or drop items, or change direction. The Orchestrator commits
the reply (C2), BD updates the roadmap, and the backlog is re-ranked before
the next item is pulled. If `delivery.release_review: wait`, the
Orchestrator waits for the reply. With `notify` (the default), it carries
on with the proposed next item and picks up the reply whenever it arrives.

### 3.7 Epics, slicing and feature flags

- **`EPIC-NNNN-<slug>`** items group slices. An epic never enters the
  lifecycle itself. It has an `items/<ID>/item.md` listing its slices, a
  goal, and its own done criterion. Each slice has `epic: EPIC-NNNN` in its
  state entry. The epic is done when its slices are released and its goal is met.
- **Slice by user value, not by layer.** Good patterns: one happy path
  first, then edge cases; one user role at a time; one data variant at a
  time; manual first, automated later; read before write.
- **Main is always releasable.** A slice that is complete but not yet
  useful (or not yet safe) to show users ships **behind a feature flag**
  (`feature_flag: <name>` in the spec). The flag is on in uat and stage for
  QA, and off in prod until the human approves turning it on. Turning a flag
  on in prod is a prod change: approval (C1), deploy record, monitor window.
- Flags are temporary. The epic's last slice removes them, and a
  `CHORE` is raised for any flag older than `delivery.flag_max_age_days`.

### 3.8 Spikes

`SPIKE-NNNN-<slug>` items answer a question: "can we integrate with X?",
"which approach is cheaper?", "do users even want Y?".

- They are time-boxed (`delivery.spike_max_tokens`) and the output is a
  **decision**: an ADR in `knowledge/decisions/` and/or new roadmap
  candidates. It is never production code. Throwaway code stays on the
  spike branch and is not merged.
- Track: `10-req/question.md` → the agent best suited to answer it
  (data-flow, bd, backend, …) → `70-release/spike-result.md` → a short
  review for the human, same as 3.6.

---

## 4. Gates (exit criteria)

### 4.1 Standard gates

| Gate | Must be true to pass |
|---|---|
| **REQ → DESIGN** | **Definition of Ready** (3.4): one user, one outcome; problem and success signal stated; within size limits or split into an epic; duplicate check against backlog done |
| **DESIGN → DEV** | Spec (`story.md`/`prd.md`) approved per `human_approvals.spec`; every AC testable; for each **triggered** area only: contract frozen, data-req done, threat model done, screens cover all states |
| **DEV → REVIEW** | Builds; lint/vet clean; unit tests pass; coverage ≥ target; migrations have rollback; docs updated; **all work committed on the item branch (and pushed, if a remote is set)** |
| **REVIEW → QA** | Reviewer approves against the PRD; security scan clean; `lessons.md` checklist ticked |
| **QA@env → next env** | 100% of acceptance criteria pass (with the feature flag on, and nothing changed with it off); no open Sev1/Sev2; regression suite green |
| **STAGE → PROD** | All of the above **plus** explicit human approval (committed, and pinned to the commit SHA being deployed) and a rollback plan |
| **PROD → RELEASED** | Smoke tests pass; monitor window clean (e.g. 30 min); CHANGELOG published; release review sent to the human (3.6) |

### 4.2 Definition of Done
The slice is in prod and **functionally complete for its user**. Nothing
half-built is visible unless it's behind an off flag. Its success signal is
instrumented, docs and CHANGELOG are updated, the release review has been
sent, and the item is closed with links to every artifact.

### 4.3 Fast lane (bugs and hotfixes)
`Triage → DEV (fix + regression test that reproduces the bug) → REVIEW → QA →
UAT → STAGE → PROD`. Skips BD and Design unless the fix changes behaviour or the
contract. Sev1 hotfixes may compress UAT/STAGE only with human approval.

---

## 5. Agent definition template

Each agent is defined in `.sdlc/agents/<name>.md`:

```markdown
# Agent: <name>
Stage: <REQ|DESIGN|DEV|QA|DEPLOY|CONTROL>
Mission: <one sentence>
Runs when: <always | only when the slice's `triggers` include …> (3.4)
Reads:   <artifacts it must read before starting, always incl. knowledge/lessons.md>
Writes:  <protocol files under .sdlc/items/<ID>/<NN-stage>/ + project paths it may touch>
Tools:   <allowed tools / commands>
Forbidden: <what it must never do, e.g. "edit src/", "deploy", "merge">
Exit criteria: <checklist the agent self-verifies before handing off>
Handoff: <next agent(s) + the handoff note format>
Escalate when: <conditions that go to Orchestrator/human>
Spawns:  <sub-agents it may spawn, or "none">; it mints and traces their run IDs (6.9)
```

---

## 6. Protocol directory `.sdlc/`: communication and status files

### 6.1 The separation rule

Files in a repo fall into exactly one of two kinds:

| Kind | What it is | Where it lives | Who writes it |
|---|---|---|---|
| **Project files** | The product: source, tests, migrations, configs, README, user docs, CHANGELOG | Anywhere in the project tree **except** `.sdlc/` | Humans and agents |
| **Protocol files** | How the work is coordinated: backlog, state, specs, handoffs, reviews, QA reports, approvals, escalations | **Only** inside `.sdlc/` | Agents (humans only in the places listed in 6.6) |

- An agent **never** writes a coordination file outside `.sdlc/`. No `PRD.md`,
  `REVIEW.md`, `TODO.md` or `STATUS.md` at the repo root.
- Nothing inside `.sdlc/` is part of the product. Builds, tests, packaging and
  deploys must ignore it. Deleting `.sdlc/` loses the history but never breaks
  the product.
- When a protocol file needs to point at a project file (e.g. a review of
  `internal/api/user.go`), it **links to it by repo-relative path** and never
  copies it.
- Deliverables that agents produce (code, migrations, CHANGELOG entries) are
  project files. They go in the project tree, and the item's `log.md` records
  their paths.

### 6.2 Directory layout

```
.sdlc/
  PROTO.md                          # this protocol (lives here, not at the repo root)
  TODO.md                           # human status report, TODO.MD format, generated (6.11)
  bin/
    sdlc-status                     # renders TODO.md / the live view from state.json + trace/
    sdlc-watch                      # live terminal view for humans (read-only)
  README.md                         # generated: "this dir is managed by PROTO.md, don't hand-edit"
  PROTOCOL_VERSION                  # e.g. 1.0, so agents can detect an out-of-date layout
  config.yaml                       # project config (section 9)
  agents/                           # agent definitions (section 5)
    <agent>.md
  board/
    roadmap.md                      # direction: themes, epics, one-line candidates (bd; 3.5)
    backlog.md                      # ready / nearly-ready items, ranked (Orchestrator; 3.5)
    state.json                      # machine state: item → stage, owner, retries, env versions
    board.md                        # generated human-readable view of state.json
  knowledge/
    lessons.md                      # learned mistakes and checks, read by every agent
    glossary.md                     # domain terms
    decisions/                      # project-wide ADRs
      ADR-0001-<slug>.md
  items/
    <ITEM-ID>/                      # one folder per backlog item (6.3)
      item.md                       # header card: title, type, priority, current stage, links
      10-req/
      20-design/
      30-dev/
      40-review/
      50-qa/
      60-deploy/
      70-release/
      messages/                     # agent ↔ agent handoffs, questions, rejections (6.5)
      log.md                        # append-only audit trail for this item
  inbox/                            # human → agents: new requests, answers to questions
  outbox/                           # agents → human: approval requests, escalations
  approvals/                        # human decisions on outbox requests
  trace/                            # agent run tracing (6.9)
    runs.jsonl                      # lifecycle index: spawned/started/completed/failed/… per run
    runs/<YYYY-MM-DD>/<RUN-ID>.jsonl  # detail events from inside each run
    transcripts/<RUN-ID>.md         # raw agent transcripts (gitignored)
  archive/<YYYY>/<ITEM-ID>/         # released or cancelled items moved here
  tmp/                              # scratch space, gitignored, safe to delete
```

### 6.3 Naming patterns

**Item ID:** `<TYPE>-<NNNN>-<slug>`
- `TYPE` ∈ `FEAT` (feature slice), `BUG`, `CHORE` (tech debt, infra), `SPIKE` (time-boxed question, 3.8), `HOTFIX`, `EPIC` (a group of slices that never enters the lifecycle itself, 3.7)
- `NNNN` is a zero-padded number that only increases and is unique across all types
- `slug` is kebab-case, at most 5 words
- e.g. `FEAT-0012-user-login`, `BUG-0014-login-timeout`

**Stage folders** have a numeric prefix so they sort in lifecycle order:
`10-req`, `20-design`, `30-dev`, `40-review`, `50-qa`, `60-deploy`, `70-release`.
The gaps leave room to add stages later (e.g. `45-security`).

**Artifact file:** `<artifact>[.<qualifier>].md` inside its stage folder

| Stage folder | Files |
|---|---|
| `10-req/` | `opportunity.md`, `market-analysis.md` (new product bets only), `bug-report.md`, or `question.md` (spikes) |
| `20-design/` | `story.md` (S) or `prd.md` (M), `data-req.md`, `architecture.md`, `api-contract.yaml`, `ux-flows.md`, `threat-model.md`, `screens/<screen-slug>.md` |
| `30-dev/` | `impl-notes.<agent>.md` (e.g. `impl-notes.backend.md`), `test-summary.<agent>.md` |
| `40-review/` | `review.r<N>.md`, `security-scan.r<N>.md` |
| `50-qa/` | `test-plan.md`, `qa-report.<env>.r<N>.md` (e.g. `qa-report.uat.r2.md`) |
| `60-deploy/` | `deploy.<env>.r<N>.md`, `rollback-plan.md` |
| `70-release/` | `release-notes.md`, `release-review.md` (3.6), `monitor-report.md`, `spike-result.md` (spikes) |

- `r<N>` is the **round** (attempt number). A failed QA round doesn't overwrite
  the report. `r2` sits next to `r1`, so the history of each round-trip stays
  visible. The retry budget counts these.
- Specs (`prd.md`, `architecture.md`, …) are **single living files**. Their
  history is git plus the `version` field in the header. Reports are
  **immutable once written**, and a new round creates a new file.

**Message file** (`messages/`): `<SEQ>-<from>-to-<to>-<kind>.md`
- `SEQ` is a 4-digit sequence within the item, so files sort in conversation order
- `kind` ∈ `handoff`, `question`, `answer`, `reject`, `escalation`, `approval-request`
- e.g. `0007-qa-to-backend-reject.md`, `0008-backend-to-qa-handoff.md`

**Inbox, outbox and approvals:** `<YYYYMMDD-HHMM>-<ITEM-ID|new>-<kind>.md`
- e.g. `outbox/20261002-1430-FEAT-0012-approval-request.md`, then
  `approvals/20261002-1430-FEAT-0012-approval.md` with the same stem, so a
  request and its decision always pair up

**General rules:** lowercase kebab-case, `.md` for prose, `.yaml`/`.json` only
for machine-read files, no spaces, no dates in spec file names (dates go in the
header).

### 6.4 Mandatory header (front matter)

Every Markdown protocol file starts with:

```yaml
---
id: FEAT-0012-user-login/20-design/prd       # item/stage/artifact
type: prd                     # artifact | message kind
item: FEAT-0012-user-login
stage: design
author: prd-agent             # agent name, or human:<handle>
run_id: RUN-20261002T143000Z-prd-9c2e   # the run that last wrote this file (6.9); null for humans
status: draft | in-review | approved | rejected | superseded
version: 3                    # specs only; reports use the r<N> in the file name
round: 2                      # reports and messages only
created: 2026-10-02T14:30:00Z
updated: 2026-10-02T16:05:00Z
inputs:                       # what this was derived from (traceability)
  - 10-req/opportunity.md@v2
supersedes: null
---
```

Agents check the header before trusting a file. For example, Dev won't start
from a `prd.md` whose status isn't `approved`.

### 6.5 Messages and the log

- **Messages** are how agents talk. Each one is a separate file, written once
  and never edited. Answering is a new file. Body:

  ```markdown
  ## Summary        <2–3 lines: what was done / what is being asked>
  ## Artifacts      <paths, protocol and project, that changed>
  ## Gate           PASS | FAIL | n/a — <criteria results>
  ## Needs          <what the recipient must do>
  ## Open risks     <list or "none">
  ```

- **`log.md`** is the item's append-only audit trail, one line per event,
  written by the Orchestrator:

  ```
  2026-10-02T14:30Z | design→dev | gate PASS | prd-agent → backend,fe-web | msg 0005 | run RUN-20261002T141800Z-prd-9c2e | tokens 61.4k (in 52.0k / out 9.4k) | item Σ 61.4k
  2026-10-02T18:10Z | qa@uat     | gate FAIL | qa → backend | round 1 | msg 0007 | run RUN-20261002T175900Z-qa-7f3a | tokens 54.3k (in 48.2k / out 6.1k) | item Σ 312.9k
  ```

  Every line that closes a run **must** show that run's tokens and the item's
  running total (`item Σ`). Tokens are the unit of cost for this protocol (6.10).

  `log.md` is the human-readable summary. The full run detail (spawn, timing,
  reads and writes, failures) is in `trace/` (6.9), joined by the `run` ID.

- **`board/state.json`** is the single source of truth for status, and only
  the Orchestrator writes it. `board.md` is regenerated from it and never
  hand-edited.

### 6.6 Ownership and permissions

| Path | Writer | Everyone else |
|---|---|---|
| `board/*` (except roadmap), `items/*/log.md`, `items/*/item.md` | Orchestrator | read |
| `board/roadmap.md` | bd | read |
| `items/*/<NN-stage>/*` | that stage's agents | read |
| `items/*/messages/*` | the sending agent (create only) | read |
| `knowledge/lessons.md` | Reviewer and QA append; Orchestrator curates | read (mandatory) |
| `agents/*`, `config.yaml` | humans | read |
| `inbox/`, `approvals/` | humans | read |
| `outbox/` | agents | humans read |
| `trace/runs.jsonl` | the spawner (normally Orchestrator), append only | read |
| `trace/runs/<date>/<RUN-ID>.jsonl` | that run only, append only | read |
| `trace/transcripts/` | the agent harness | humans read when debugging |
| `TODO.md` | Orchestrator, only by running `bin/sdlc-status --write` | humans read / watch |
| `bin/*` | humans (copied from ai_sdlc) | everyone runs |
| `tmp/` | anyone | not trusted, not read across agents |

Upstream artifacts are read-only for downstream agents. To change one, send a
`question` or `reject` message to the owner.

### 6.7 Git and commit policy

Commit early and often. Work that isn't committed doesn't exist: a crashed
run, a closed laptop or a new session must never lose more than a few
minutes of work. **A human approval is committed the moment it is made.**

#### Where things are committed

| What | Branch | Committed by |
|---|---|---|
| Product code, tests, migrations, docs for an item | the item branch `feature/<ITEM-ID>` (or `fix/` / `hotfix/`), never the main branch directly | the dev agent doing the work |
| Everything under `.sdlc/` | the **main branch** only (`config.yaml` → `git.main_branch`) | the Orchestrator (the single writer, so no conflicts) |
| Merge of an item branch | main branch, `--no-ff`, only after `review` is approved | the Orchestrator |
| Release tag `v<X.Y.Z>` | main branch, on the deployed commit | release |

- Never commit `.sdlc/` changes on an item branch. Stage agents write their
  `.sdlc/` files in the main checkout, and the Orchestrator commits them.
- Commit all of `.sdlc/` **except** `tmp/` and `trace/transcripts/`. Add both
  to `.gitignore`. `trace/*.jsonl` **is** committed, because it's the debugging record.

#### When to commit (mandatory commit points)

| # | Moment | Who | Commit |
|---|---|---|---|
| C1 | **A human approval or rejection lands in `approvals/`** | Orchestrator, **immediately**, before spawning anything else | the approval file, plus the resulting `state.json`, `log.md`, `TODO.md`. Subject `sdlc(<ID>): <decision> <what> by <human>`, trailers `Approved-By: human:<handle>` and `Approval: approvals/<file>` |
| C2 | A human writes a brief, config or agent change, or an `inbox/` file | Orchestrator, on its next start (or the human, directly) | `sdlc: <what changed>`, trailer `Changed-By: human:<handle>` |
| C3 | A dev agent reaches a **checkpoint**: an AC is implemented and its tests are green, a migration is written, or `git.checkpoint_min` minutes have passed since the last commit | the dev agent, on its item branch | `<type>(<ID>): <summary>` (`wip(<ID>): …` is allowed on the branch) |
| C4 | **Before any handoff**: a dev agent may not hand off with uncommitted work | the dev agent | its final commit. The handoff message names the commit SHA |
| C5 | **Every terminal event** (completed, failed, escalated, timed_out, cancelled) | Orchestrator | that run's `.sdlc/` outputs, plus `state.json`, `log.md`, `TODO.md`, `trace/`. Subject `sdlc(<ID>): <stage> r<N> <result>` |
| C6 | Before a deploy | devops | nothing new. It **refuses a dirty tree** and deploys only a committed and (for prod) tagged commit |
| C7 | Before stopping, pausing or escalating, or at the end of a session | every running agent, then the Orchestrator | everything in progress, as `wip(<ID>): …` on the item branch and `sdlc: checkpoint` on main, so a resume loses nothing |

#### Approvals pin what was approved

An approval commit (C1) records the **exact version that was approved**:
the approval file names the artifact and its `version` (e.g. `prd.md@v3`) or
the commit SHA (for a prod deploy). If the artifact changes afterwards, the
approval no longer applies. The Orchestrator must ask again (new
`approval-request`), and the gate treats the old approval as void.

#### Commit messages

- Every agent commit carries trailers `Run-Id: <RUN-ID>` (6.9) and
  `Item: <ITEM-ID>`, so any line of code traces back to the run, item and
  approval that produced it.
- Product commits: `<type>(<ITEM-ID>): <summary>`, where `type` is one of
  `feat` | `fix` | `test` | `docs` | `refactor` | `chore` | `wip`.
- Protocol commits: `sdlc(<ITEM-ID>): <summary>`, or `sdlc: <summary>` when no item applies.
- Keep product and protocol changes in **separate commits**.

#### Pushing

If `config.yaml` → `git.remote` is set, push according to `git.push`:
`on_commit` (default: after every commit point), `on_approval` (C1, C4, C5
and tags only), or `never`. **Approval commits (C1) are pushed immediately**
under every setting except `never`, so the approval record is safe
off-machine. A failed push is retried, then escalated. It is never ignored.

#### Never

Force-push or rewrite history on the main branch; amend or rebase another
run's commits; commit secrets, credentials or `.env` files (the security
agent's scan blocks it); commit with failing gates on the main branch; leave
the working tree dirty at the end of a session.

#### Start-up check

On every start, the Orchestrator runs `git status`. If the tree is dirty
(e.g. left over from a crashed run), it commits the leftovers as
`wip(<ID>): recovered from <RUN-ID>` on the right branch, records a
`decision` event, and only then continues. If it can't tell who owns the
changes, it escalates instead of guessing.

#### Archiving

When an item reaches `RELEASED` or is cancelled, the Orchestrator moves
`items/<ITEM-ID>/` to `archive/<YYYY>/` and commits the move (C5).

### 6.8 Agent start-up contract

Before doing anything, an agent:
1. Takes the `RUN-ID` its spawner gave it. If it has none, it stops: untraced runs aren't allowed.
   A dev agent then checks out its item branch and confirms the tree is clean (6.7).
2. Reads `.sdlc/PROTOCOL_VERSION` and stops if it doesn't support that version.
3. Reads its own `agents/<agent>.md`, `config.yaml`, and `knowledge/lessons.md`.
4. Reads the latest message addressed to it in `items/<ITEM-ID>/messages/`.
5. Reads only the artifacts listed in its `Reads`, checking each one's header status.
6. Writes the `started` event (6.9) and logs each read as a `read` detail event.

Before exiting, it commits its work (6.7, C4/C7) and writes exactly one terminal event (`completed`, `failed`
or `escalated`). If it spawns sub-agents, it mints their run IDs and writes
their `spawned` events itself.

Nothing else is assumed. If something it needs isn't in `.sdlc/` or the
project tree, it sends a `question` message.

### 6.9 Run tracing (debugging the SDLC itself)

`log.md` tells you **what** happened to an item. `trace/` tells you **which
agent run did it, who spawned that run, when it started and ended, what it
read and wrote, and how it finished**. Every agent run is traced, with no
exceptions, including the Orchestrator's own runs.

**Run ID:** `RUN-<YYYYMMDDTHHMMSSZ>-<agent>-<4 hex>`
- e.g. `RUN-20261002T143000Z-qa-7f3a`
- The **spawner** mints the ID and hands it to the agent at spawn time. An
  agent never makes up its own ID.
- A retry is a **new run** with a new ID. It points back with `retry_of`.

**Spawn tree:** every run except the root Orchestrator run records
`parent_run`. Following `parent_run` upwards rebuilds who spawned whom, e.g.

```
RUN-…-orchestrator-01ab                     (root, picks FEAT-0012)
├── RUN-…-prd-9c2e                          completed  12m   61.4k tok
├── RUN-…-backend-44f1                      completed  31m  142.8k tok
├── RUN-…-reviewer-d07a                     completed   6m   28.1k tok
├── RUN-…-qa-7f3a                           failed      9m   54.3k tok  (qa@uat r1 FAIL)
├── RUN-…-backend-b812   retry_of 44f1      completed  14m   67.0k tok
└── RUN-…-qa-e5c9                           completed   8m   49.7k tok  (qa@uat r2 PASS)
```

#### Files

| File | Written by | Contents |
|---|---|---|
| `trace/runs.jsonl` | Orchestrator (spawner) | **Lifecycle index**: one line per lifecycle event for every run in the project |
| `trace/runs/<YYYY-MM-DD>/<RUN-ID>.jsonl` | the run itself | **Detail events** from inside that run |
| `trace/transcripts/<RUN-ID>.md` | the agent harness | Raw prompt/response transcript. Gitignored, may be large, may be pruned |

All three are append-only JSON Lines, one event per line. Never rewrite or
delete a line. A correction is a new event.

#### Lifecycle events (`trace/runs.jsonl`)

| `event` | When | Required extra fields |
|---|---|---|
| `spawned` | Spawner creates the run | `parent_run`, `agent`, `item`, `stage`, `round`, `task` (one line), `inputs[]`, `retry_of?` |
| `started` | Agent has passed the start-up contract (6.8) | `protocol_version`, `model` |
| `completed` | Agent finished and handed off | `status: ok`, `outputs[]`, `message`, `gate?`, **`tokens`** |
| `failed` | Agent hit an error it couldn't recover from, or its gate failed | `status: error\|gate_fail`, `reason`, **`tokens`** |
| `timed_out` | Spawner gave up waiting (`config.yaml` → `tracing.run_timeout_min`) | `reason`, **`tokens`** (last known, `"estimated": true`) |
| `escalated` | Run handed the problem to a human (`outbox/`) | `reason`, `outbox` path, **`tokens`** |
| `cancelled` | Spawner stopped the run (e.g. upstream artifact changed) | `reason`, **`tokens`** |

Every terminal event carries a `tokens` object (6.10). A run that ended
without reporting tokens is a protocol violation, and the Orchestrator flags it.

Every run must end with exactly **one** terminal event (`completed`, `failed`,
`timed_out`, `escalated`, `cancelled`). If a run has `spawned` but no terminal
event, it is **orphaned**. The Orchestrator checks for orphans on every start
and writes `timed_out` for any run older than the timeout.

Example lines:

```json
{"ts":"2026-10-02T14:30:00Z","event":"spawned","run_id":"RUN-20261002T143000Z-qa-7f3a","parent_run":"RUN-20261002T090000Z-orchestrator-01ab","agent":"qa","item":"FEAT-0012-user-login","stage":"qa@uat","round":1,"task":"Run acceptance tests on uat","inputs":["20-design/prd.md@v2","60-deploy/deploy.uat.r1.md"]}
{"ts":"2026-10-02T14:30:04Z","event":"started","run_id":"RUN-20261002T143000Z-qa-7f3a","protocol_version":"1.1","model":"<model-id>"}
{"ts":"2026-10-02T14:39:12Z","event":"failed","run_id":"RUN-20261002T143000Z-qa-7f3a","status":"gate_fail","reason":"AC-2 failed: account not locked after 5 bad attempts","outputs":["50-qa/qa-report.uat.r1.md"],"message":"messages/0007-qa-to-backend-reject.md","duration_ms":552000,"tokens":{"input":48210,"output":6120,"cache_read":31500,"cache_write":4200,"total":54330,"subtree_total":54330},"commit":"a1b2c3d"}
```

#### Detail events (`trace/runs/<date>/<RUN-ID>.jsonl`)

`progress` (heartbeat: `step` = what the agent is doing now in plain words,
optional `pct`; required at every meaningful step and at least every
`tracing.heartbeat_min`), `read` (path@version), `wrote` (path, protocol or project), `tool_call`
(tool, short args, exit code, duration), `gate_check` (criterion, pass/fail),
`message_sent`, `question_asked`, `decision` (a non-obvious choice and why),
`error`. Keep each line short and put large outputs in the transcript.

#### Common fields (every event, both files)

`ts` (UTC ISO-8601), `event`, `run_id`. Lifecycle events also repeat
`agent` and `item`, so you can `grep` a single line without joining.

#### Linking everything together

- Every protocol file header has `run_id` (6.4). Every artifact and message
  points back to the exact run that wrote it.
- Every `log.md` line ends with the `run` that caused it (6.5).
- Every commit an agent makes carries a `Run-Id: <RUN-ID>` git trailer, so
  `git log --grep` finds the code a run produced.

#### Debugging recipes

```sh
# Everything that happened to one item, in order
grep '"item":"FEAT-0012' .sdlc/trace/runs.jsonl

# Runs that never finished (orphans)
jq -s 'group_by(.run_id)[] | select(all(.event!="completed" and .event!="failed"
       and .event!="timed_out" and .event!="escalated" and .event!="cancelled"))
       | .[0]' .sdlc/trace/runs.jsonl

# Who wrote this artifact, and what was that run told to do?
grep run_id .sdlc/items/FEAT-0012-user-login/50-qa/qa-report.uat.r1.md

# Commits made by a run
git log --grep 'Run-Id: RUN-20261002T143000Z-qa-7f3a'
```

#### Privacy

Never write secrets, credentials or personal data into a trace. Mask them
(`****`) at the point of logging. Transcripts are gitignored for exactly
this reason, and `config.yaml` → `tracing.transcripts` can turn them off.

### 6.10 Token accounting

Tokens are the unit of cost and effort for every agent. Each run reports
exactly how many it used, and those numbers roll up to items, stages, agents
and the project.

**The `tokens` object** (on every terminal event in `trace/runs.jsonl`):

```json
"tokens": {
  "input": 48210,          // fresh input tokens
  "output": 6120,          // generated tokens
  "cache_read": 31500,     // prompt-cache hits (0 if not supported)
  "cache_write": 4200,     // prompt-cache writes (0 if not supported)
  "total": 54330,          // input + output: this run only
  "subtree_total": 54330,  // total + every descendant run's subtree_total
  "model": "<model-id>",
  "estimated": false       // true only when the harness couldn't report real counts
}
```

- Counts come from the model API or harness usage report, **never** from a
  guess. If real counts aren't available, set `estimated: true`.
- `total` covers this run only. `subtree_total` adds every sub-agent it
  spawned, so the root Orchestrator run's `subtree_total` is the whole
  item's cost.
- A run that spawns sub-agents waits for their terminal events before
  writing its own, so its `subtree_total` is complete.
- Retries count. A failed QA round's tokens stay on the item's bill, which
  makes rework visible.

**Where tokens show up:**

| Place | What's shown |
|---|---|
| `trace/runs.jsonl` terminal event | the full `tokens` object (source of truth) |
| `items/<ID>/log.md` | `tokens <total> (in / out) \| item Σ <running total>` on every line that closes a run |
| `items/<ID>/item.md` | a **Tokens** table: one row per agent (runs, total), one per stage, and the item total |
| `board/state.json` | `items.<ID>.tokens: { total, by_agent: {...}, by_stage: {...} }`, kept up to date by the Orchestrator |
| `board/board.md` | tokens per in-progress item, plus the project total |
| `70-release/release-notes.md` | final item cost: tokens per stage, rework share (tokens spent on retries) |

**Budgets** (`config.yaml` → `tokens`):
- `per_run_max`: a run that hits this limit stops and writes `escalated`
  (reason `token_budget`).
- `per_item_max`: when an item's running total passes this limit, the
  Orchestrator stops spawning runs for it and escalates to a human with the
  token breakdown.
- `warn_at_pct`: the Orchestrator writes a `warning` line in `log.md` and an
  `outbox/` note when an item passes this percentage of its budget.

### 6.11 Human status view: `TODO.md` and `sdlc-watch`

Agents can work for a long time. The human watching should always be able
to see **who is working, on what, how far along it is, and what it has cost**,
without reading JSON.

**`.sdlc/TODO.md`** is the human status report, in
[TODO.MD](https://github.com/doublefreein/TODO.MD) format. It is
**generated, never hand-edited**: `bin/sdlc-status --write` builds it from
`board/state.json` and `trace/`. The Orchestrator regenerates it after every
lifecycle event it writes and every state change. The write is atomic, so a
reader never sees half a file. Sections:

1. **Header:** number of agents working, the current item and stage, project
   tokens, environment versions, last update time.
2. **Live:** one line per running agent: agent, item, stage and round, time
   elapsed, `pct`, and its latest `progress` step. A run that has gone quiet
   for more than 5 minutes is flagged ⚠️.
3. **Recent:** the last 10 spawn and terminal events (▶️ ✅ ❌ ⌛ 🙋 ⏹️), with
   tokens and reasons.
4. **Board:** one `## <section>` per backlog section. Each item is a TODO.MD
   task line, and its agent runs are subtasks under it. An `EPIC` shows
   `slices:<released>/<total>`, with its slices nested under it as subtasks:

```
* [u] (p0) 2026-10-02 +shop Cart and checkout @qa id:FEAT-0002-cart-checkout stage:uat round:1 tok:250.1k
    * [X] 2026-10-02 2026-10-02 +shop design r1: Write PRD @prd run:RUN-…-prd-0001 took:12m00s tok:61.4k
    * [X] 2026-10-02 2026-10-02 +shop dev r1: Implement cart API @backend run:RUN-…-0002 took:37m00s tok:142.8k
    * [o] 2026-10-02 +shop qa@uat r1: Run acceptance tests on uat @qa run:RUN-…-qa-0003 took:3m00s
```

TODO.MD fields as used here: status marker; `(p0…p4)` = item priority;
creation date, then completion date; `+<project>`; title; `@<agent>` = the
current owner or the run's agent; `k:v` = `id`, `stage`, `round`, `tok`,
`run`, `took`, `release`, `retry_of`.

| Item stage | Marker | | Run end | Marker |
|---|---|---|---|---|
| BACKLOG | `[ ]` | | running | `[o]` |
| REQ | `[.]` | | completed | `[X]` |
| DESIGN, DEV | `[o]` | | failed, timed_out, cancelled | `[f]` |
| REVIEW, QA | `[O]` | | escalated | `[B]` |
| READY (passed QA@dev) | `[x]` | | | |
| UAT / STAGE / PROD | `[u]` / `[s]` / `[p]` | | | |
| RELEASED | `[X]` | | | |
| blocked (waiting on a human) | `[B]` | | | |
| failed gate in UAT / STAGE / PROD / elsewhere | `[uf]` / `[sf]` / `[pf]` / `[f]` | | | |

**`bin/sdlc-watch [seconds]`** is the live show: it re-renders the same
report every 2 s (default) straight from `state.json` and `trace/`, so it
updates between Orchestrator writes too. It is read-only and needs only
bash + python3. Run it in a spare terminal, or use
`watch -n 2 cat .sdlc/TODO.md` for the last snapshot.

**What feeds it**, and is therefore required:
- `progress` events from every running agent (6.9), written at least every
  `tracing.heartbeat_min`. Without them the Live line can only show the task.
- Item fields in `board/state.json`, maintained by the Orchestrator:

```json
"items": {
  "FEAT-0002-cart-checkout": {
    "title": "Cart and checkout", "section": "Storefront", "priority": "p0",
    "created": "2026-10-02", "released": null,
    "stage": "UAT", "flag": null,            // null | blocked | failed
    "owner": "qa", "round": 1, "release": null,
    "type": "FEAT", "size": "S",              // S | M (EPIC items have no size)
    "epic": "EPIC-0001-storefront",           // parent epic, or null
    "feature_flag": null,                     // flag name if it ships dark (3.7)
    "note": null,                            // one line shown under the item (e.g. what it's blocked on)
    "tokens": { "total": 250100, "by_agent": {}, "by_stage": {} }
  }
}
```

---

## 7. Environments and promotion

| Env | Purpose | Who deploys | Gate |
|---|---|---|---|
| dev | per-branch builds, unit and integration tests | CI | DEV → REVIEW |
| uat | functional acceptance against the PRD | DevOps | QA@uat |
| stage | prod-like data and config, perf and regression | DevOps | QA@stage + human |
| prod | live | DevOps | smoke + monitor |

- Promote the **same build** from one environment to the next; never rebuild
  per environment.
- Every deploy has a tested rollback. Schema migrations need to be
  backward-compatible across one release.

---

## 8. Human-in-the-loop checkpoints

1. **Release review after every release** (3.6). This is the main way the
   human steers: re-rank, add, drop, or change direction.
2. Spec sign-off for the sizes listed in `human_approvals.spec` (default: M only)
3. Destructive or irreversible migrations
4. STAGE → PROD promotion for the sizes in `human_approvals.prod_deploy`,
   and turning a feature flag on in prod
5. Any escalation from a used-up retry budget, a token budget, or a security blocker
6. Optionally, approving a new roadmap theme or epic

Every human decision is committed immediately (6.7, C1), and it is pinned
to the exact artifact version or commit that was approved.

---

## 9. `.sdlc/config.yaml` (what makes this protocol project-independent)

```yaml
project: <name>
stack: { backend: <lang/framework>, web: <...>, app: <none|ios|android|rn|flutter>, db: <...> }
agents:
  enabled: [orchestrator, reviewer, security, bd, triage, prd, data-req,
            data-flow, ux, ui, schema, backend, fe-web, docs, qa, devops,
            release, monitor]
  disabled: [fe-app]            # e.g. a CLI or API-only project
environments: [dev, uat, stage, prod]
environment_mapping:            # how "deploy" is realised in this project
  uat: <k8s namespace | git branch | url>
  prod: <...>
gates:
  coverage_min: 80
  retry_budget: 3
  monitor_window_min: 30
human_approvals:                # who signs off what, per item size
  spec: [M]                     # story/PRD sign-off: [] | [M] | [S, M]
  prod_deploy: [S, M]           # drop S once you trust the pipeline
  flag_on_in_prod: true
  destructive_migration: true
compliance: [none | pci-dss | gdpr | rbi | hipaa]
delivery:                       # 3.3–3.8
  walking_skeleton: true        # first item = hello world through the real pipeline to prod
  sizes:
    S: { max_acs: 3, max_tokens: 300000 }
    M: { max_acs: 5, max_tokens: 800000 }
  refine_ahead: 3               # only this many backlog items are refined in detail
  wip_limit: 1                  # max items between DEV and PROD at once
  release_review: notify        # notify (carry on) | wait (block until the human replies)
  feature_flags: true
  flag_max_age_days: 30
  spike_max_tokens: 200000
git:                            # 6.7
  main_branch: main
  item_branch: feature/<ITEM-ID>  # fix/<ITEM-ID> for BUG, hotfix/<ITEM-ID> for HOTFIX
  remote: origin                # or null for local-only
  push: on_commit               # on_commit | on_approval | never
  checkpoint_min: 30            # a dev agent commits at least this often while working
tracing:
  run_timeout_min: 60           # spawner marks a silent run timed_out after this
  heartbeat_min: 2              # max gap between an agent's `progress` events
  detail_events: true           # write trace/runs/<date>/<RUN-ID>.jsonl
  transcripts: true             # keep raw transcripts (gitignored)
tokens:                         # 6.10 (always recorded; these are the budgets)
  per_run_max: 200000
  per_item_max: 1500000
  warn_at_pct: 80
```

Small projects can merge roles (e.g. Data Req + Data Flow + Schema → one
"Data" agent; UX + UI → one "Design" agent). Larger ones can split them further.
The protocol stays the same and only the roster changes.

---

## 10. Metrics (to evaluate the agent system itself)

- **Flow** (3.3–3.5): cycle time per item (DEV → RELEASED) by size;
  releases per week; WIP over time; share of items that are S; items split
  after starting (slicing misses); time from release review to the human's reply
- Lead time per item (BACKLOG → RELEASED) and time spent at each stage
- Gate failure rate per stage and **escape rate** (bugs found in a later env
  than they should have been)
- DEV ↔ QA round-trips per item
- Human escalations per item
- Commit hygiene: time between commits during runs, uncommitted-work
  recoveries (6.7 start-up check), time from human approval to its commit
- Repeat-mistake rate (the same `lessons.md` entry triggered again)
- From `trace/`: runs per item, run duration per agent (p50/p95), failed or
  timed-out run rate per agent, orphaned runs, retries per stage
- **Tokens** (6.10): tokens per item, per agent (p50/p95 per run), per stage;
  **rework tokens** (spent on retries and failed rounds) as a share of the
  total; tokens per released item over time (is the system getting cheaper?)
