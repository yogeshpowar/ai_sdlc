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
5. **Humans approve the irreversible.** Scope sign-off, production deploys,
   schema migrations on live data, and anything that costs money or reaches
   users need explicit human approval.
6. **Bounded loops.** Every feedback loop has a retry limit. When it runs out,
   the work escalates to a human; agents don't loop forever.
7. **Learn once.** Repeat mistakes go into `.sdlc/knowledge/lessons.md` and every later agent
   reads it before starting.

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
| **BD / Market** | Market signals, competitors, user feedback → `opportunity.md` (problem, target user, competitive analysis, sizing, proposed feature/product) |
| **Triage** | Bug reports, prod incidents, QA escapes → `bug-<id>.md` (repro, severity, affected area). Bugs take the **fast lane** (section 4.3) and skip market analysis. |

### 2.2 Design

| Agent | Input → Output |
|---|---|
| **PRD** | `opportunity.md` → `prd.md`: goal, non-goals, user stories, **testable** acceptance criteria, success metrics. *Human sign-off gate.* |
| **Data Requirements** | `prd.md` → `data-req.md`: entities, fields, ownership, retention, PII classification |
| **Data Flow / Architecture** | `prd.md`, `data-req.md` → `architecture.md` + `api-contract` (OpenAPI/proto): components, sequence/data-flow diagrams, integration points, ADRs for big decisions |
| **UX** | `prd.md` → `ux-flows.md`: user journeys, states (empty/loading/error), accessibility |
| **UI / Screens** | `ux-flows.md` → `screens/`: screen specs or mockups for every state |

> Order: PRD → (Data Req ‖ UX) → (Data Flow ‖ UI). `‖` = can run in parallel.

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
| **Release** | Release notes, CHANGELOG, version tag, announcement. |
| **Monitor / SRE** | Watches prod after release (errors, latency, business KPIs vs PRD success metrics). Problems go to Triage, insights go to BD. **This closes the loop.** |

---

## 3. Lifecycle state machine

```
BACKLOG → REQ → DESIGN → DEV → REVIEW → QA@dev
        → UAT  (deploy → QA@uat)
        → STAGE(deploy → QA@stage)
        → PROD (human approve → deploy → smoke → monitor)
        → RELEASED → (feedback) → BACKLOG
```

QA is **interleaved**, not a single stage at the end: every promotion is
`deploy → QA → gate`.

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
  attempts.

---

## 4. Gates (exit criteria)

### 4.1 Standard gates

| Gate | Must be true to pass |
|---|---|
| **REQ → DESIGN** | Problem, target user, and success metric are stated; duplicate check against backlog done |
| **DESIGN → DEV** | PRD human-approved; every acceptance criterion testable; API contract frozen; threat model done; screens cover all states |
| **DEV → REVIEW** | Builds; lint/vet clean; unit tests pass; coverage ≥ target; migrations have rollback; docs updated |
| **REVIEW → QA** | Reviewer approves against the PRD; security scan clean; `lessons.md` checklist ticked |
| **QA@env → next env** | 100% of acceptance criteria pass; no open Sev1/Sev2; regression suite green |
| **STAGE → PROD** | All of the above **plus** explicit human approval and a rollback plan |
| **PROD → RELEASED** | Smoke tests pass; monitor window clean (e.g. 30 min); CHANGELOG published |

### 4.2 Definition of Done
The item is in prod, its success metric is instrumented, docs and CHANGELOG are
updated, and the backlog item is closed with links to every artifact.

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
    backlog.md                      # ordered list of item IDs + status (Orchestrator only)
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
- `TYPE` ∈ `FEAT` (feature), `BUG`, `CHORE` (tech debt, infra), `SPIKE` (research), `HOTFIX`
- `NNNN` is a zero-padded number that only increases and is unique across all types
- `slug` is kebab-case, at most 5 words
- e.g. `FEAT-0012-user-login`, `BUG-0014-login-timeout`

**Stage folders** have a numeric prefix so they sort in lifecycle order:
`10-req`, `20-design`, `30-dev`, `40-review`, `50-qa`, `60-deploy`, `70-release`.
The gaps leave room to add stages later (e.g. `45-security`).

**Artifact file:** `<artifact>[.<qualifier>].md` inside its stage folder

| Stage folder | Files |
|---|---|
| `10-req/` | `opportunity.md`, `market-analysis.md`, or `bug-report.md` |
| `20-design/` | `prd.md`, `data-req.md`, `architecture.md`, `api-contract.yaml`, `ux-flows.md`, `threat-model.md`, `screens/<screen-slug>.md` |
| `30-dev/` | `impl-notes.<agent>.md` (e.g. `impl-notes.backend.md`), `test-summary.<agent>.md` |
| `40-review/` | `review.r<N>.md`, `security-scan.r<N>.md` |
| `50-qa/` | `test-plan.md`, `qa-report.<env>.r<N>.md` (e.g. `qa-report.uat.r2.md`) |
| `60-deploy/` | `deploy.<env>.r<N>.md`, `rollback-plan.md` |
| `70-release/` | `release-notes.md`, `monitor-report.md` |

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
| `board/*`, `items/*/log.md`, `items/*/item.md` | Orchestrator | read |
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

### 6.7 Git policy

- Commit all of `.sdlc/` **except** `tmp/` and `trace/transcripts/`. Add both
  to `.gitignore`. `trace/*.jsonl` **is** committed, because it's the debugging record.
- Agent commits carry a `Run-Id: <RUN-ID>` trailer (6.9).
- Protocol changes go in their own commits with an `sdlc:` prefix, e.g.
  `sdlc(FEAT-0012): qa@uat r1 FAIL`, so they don't mix with product commits.
- When an item reaches `RELEASED` or is cancelled, the Orchestrator moves
  `items/<ITEM-ID>/` to `archive/<YYYY>/` and keeps the board lean.

### 6.8 Agent start-up contract

Before doing anything, an agent:
1. Takes the `RUN-ID` its spawner gave it. If it has none, it stops: untraced runs aren't allowed.
2. Reads `.sdlc/PROTOCOL_VERSION` and stops if it doesn't support that version.
3. Reads its own `agents/<agent>.md`, `config.yaml`, and `knowledge/lessons.md`.
4. Reads the latest message addressed to it in `items/<ITEM-ID>/messages/`.
5. Reads only the artifacts listed in its `Reads`, checking each one's header status.
6. Writes the `started` event (6.9) and logs each read as a `read` detail event.

Before exiting, it writes exactly one terminal event (`completed`, `failed`
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
   task line, and its agent runs are subtasks under it:

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

1. Approving a new opportunity into the backlog (optional, per project)
2. PRD sign-off
3. Destructive or irreversible migrations
4. STAGE → PROD promotion
5. Any escalation from a used-up retry budget or a security blocker

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
human_approvals: [prd, prod_deploy, destructive_migration]
compliance: [none | pci-dss | gdpr | rbi | hipaa]
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

- Lead time per item (BACKLOG → RELEASED) and time spent at each stage
- Gate failure rate per stage and **escape rate** (bugs found in a later env
  than they should have been)
- DEV ↔ QA round-trips per item
- Human escalations per item
- Repeat-mistake rate (the same `lessons.md` entry triggered again)
- From `trace/`: runs per item, run duration per agent (p50/p95), failed or
  timed-out run rate per agent, orphaned runs, retries per stage
- **Tokens** (6.10): tokens per item, per agent (p50/p95 per run), per stage;
  **rework tokens** (spent on retries and failed rounds) as a share of the
  total; tokens per released item over time (is the system getting cheaper?)
