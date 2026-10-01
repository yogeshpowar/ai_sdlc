# Changelog: SDLC protocol

## 1.4 — 2026-10-02

Moves from "full analysis per stage" to **iterative delivery**:
requirements are assumed *not* to be locked, so the product is built in
thin, functionally complete slices that each go to prod, and the human
steers after every release.

### Added
- **Delivery model (§3.3):** thin vertical slices; walking skeleton first;
  just-in-time refinement; the human steers after every release.
- **Item sizes and tracks (§3.4):** S (≤3 ACs, `story.md`) and M (≤5 ACs,
  short `prd.md`); anything bigger must become an epic. Design agents run
  only when the slice's `triggers` (data, contract, ui, security) call for
  them. Definition of Ready.
- **Roadmap, backlog and WIP (§3.5):** `board/roadmap.md` (bd) for
  direction; the backlog holds only refined items (`refine_ahead`); re-rank
  after every release; `wip_limit`.
- **Release review (§3.6):** after every release the human gets Shipped /
  Try it / Cost / Learned / Proposed next / Questions in `outbox/`, and
  replies via `inbox/`.
- **Epics and feature flags (§3.7):** `EPIC` type; slicing patterns; shipping
  dark behind flags (on in uat/stage, off in prod until approved).
- **Spikes (§3.8):** time-boxed questions whose output is a decision or an
  ADR, never production code.
- Principles 8 and 9; `delivery` section in `config.yaml`; flow metrics
  (cycle time, release frequency, WIP).
- `sdlc-status` shows size, flags and epics, with slices nested under their epic.
- **`sdlc-watch` is now a split-screen view** (python3 `curses`, no
  dependencies): Live + Recent on the left, auto-updating; the board on the
  right, scrollable, with keys for paging, switching panes, hiding released
  items, showing their runs and resizing. It stacks top/bottom below 100
  columns, and `--plain` keeps the old full refresh.
- `TODO.md` shows released items as one line (`runs:<n>`) instead of listing
  every run, so it no longer grows without bound.

### Changed
- `human_approvals` is now per size (`spec: [M]`, `prod_deploy: [S, M]`,
  `flag_on_in_prod`, `destructive_migration`). It replaces the old flat list.
- Gates: REQ → DESIGN is the Definition of Ready; DESIGN → DEV checks only
  the triggered areas; QA checks flag on and off; PROD → RELEASED requires
  the release review.
- Agents: orchestrator (walking skeleton, refinement, WIP, size tracks,
  release-review loop), bd (living roadmap), prd (size, slice, story vs
  PRD, epics), design agents get a `Runs when:` trigger, dev agents stay in
  the slice and respect flags, qa tests with flags on/off, devops manages
  flags, release writes the release review.
- GUIDE.md: "How the work flows", brief as a direction, the release review
  as the human's main job, an updated kickoff prompt and config table.
- **Project knowledge lives in the project (§6.1, principle 10).** The
  architecture, API contract, data model, ADRs, UX flows and screens,
  threat model, glossary and runbook are project files in `docs/` (paths in
  the new `config.yaml → docs` map). Design agents edit them on the item
  branch and they are reviewed with the code. `.sdlc/` keeps only process
  and history, with per-item `design-notes.<agent>.md` linking to the docs
  diff. A new deletion test: without `.sdlc/`, the project must still be
  fully understandable. The reviewer gate checks that docs match the code.
  The walking skeleton creates the `docs/` skeleton. The item branch starts
  at the first project-file change (design or dev).
- **`bin/sdlc-upgrade` (§6.12):** upgrades a project in place without touching
  code, `docs/` or agent records. It replaces untouched protocol files,
  three-way-merges customised ones against the old version's tag (on
  conflict, the project's file is kept and `.sdlc-new`/`.sdlc-merge` copies
  are written), creates new files, never re-creates deleted agents, and
  refuses with a dirty tree or live agent runs. `--dry-run`; `--commit`
  only when nothing is left to reconcile. Writes `.sdlc/UPGRADES.md` and
  an `inbox/` request with the CHANGELOG's manual steps, for the
  Orchestrator to do as a CHORE item.
- Removed from the template: `knowledge/context.md`, `knowledge/glossary.md`,
  `knowledge/decisions/` (now `README`/`docs/architecture.md`,
  `docs/glossary.md`, `docs/adr/`).

### Upgrading from 1.3
Run `bin/sdlc-upgrade <project>`. It merges the new `human_approvals`,
`delivery` and `docs` sections into `config.yaml`, adds `board/roadmap.md`,
and leaves these manual steps in the project's `inbox/` for the
Orchestrator to do as a CHORE item:
1. If `config.yaml` still has `human_approvals: [...]` as a flat list,
   replace it with the per-size form.
2. Move durable knowledge out of `.sdlc/` into the project tree: content
   of `knowledge/context.md` → `README.md` / `docs/architecture.md`;
   `knowledge/glossary.md` → `docs/glossary.md`; `knowledge/decisions/*` →
   `docs/adr/`. The latest `architecture.md`, `api-contract.*`,
   `data-req.md`, `ux-flows.md`, `screens/` and `threat-model.md` under
   `items/` or `archive/` → the matching `docs/` paths. Leave the old item
   files in place as history.
3. Create any `docs/` files that are still missing (one-line stubs).

## 1.3 — 2026-10-02

### Added
- **Commit policy (§6.7, rewritten):** which branch each kind of change goes
  on, and seven mandatory commit points (C1–C7). The key ones: a human
  approval is committed (and pushed) **immediately**; dev agents commit at
  every checkpoint and at least every `git.checkpoint_min` minutes; nobody
  hands off or stops with uncommitted work.
- An approval pins the exact artifact version or commit it covers. Later
  changes void it and need a new request.
- Commit message conventions with `Run-Id`, `Item`, `Approved-By`,
  `Approval` and `Changed-By` trailers; push policy (`git.push`); a list of
  things never to do; the Orchestrator's dirty-tree recovery on start.
- `git` section in `config.yaml`; gates require committed work (DEV → REVIEW)
  and a pinned, committed approval (STAGE → PROD).
- `_common.md`, `orchestrator.md`, `devops.md`, GUIDE.md and the kickoff
  prompt updated to match.

## 1.2.1 — 2026-10-02 (docs only, protocol unchanged)

### Added
- `GUIDE.md`: step-by-step instructions for humans: set up, brief, config,
  agents, the kickoff prompt, watching, approvals, inbox, troubleshooting,
  upgrading. Defines the content of `approvals/` and `inbox/` files.
  `sdlc-init` copies it into each project's `.sdlc/`.

## 1.2 — 2026-10-02

### Added
- **Human status view (§6.11):** `.sdlc/TODO.md`, a generated status report
  in [TODO.MD](https://github.com/doublefreein/TODO.MD) format (header,
  Live, Recent, and a board with each item's runs as subtasks), plus
  `bin/sdlc-watch`, a read-only live terminal view refreshed every 2 s.
- `bin/sdlc-status`: renders the report from `state.json` + `trace/`;
  `--write` regenerates `TODO.md` atomically.
- `progress` heartbeat detail event (`step`, `pct`), required at every
  meaningful step and at least every `tracing.heartbeat_min` minutes.
- The item fields in `state.json` that the view depends on are now documented.
- Orchestrator regenerates `TODO.md` after every change; `sdlc-init` creates
  the first one.
- `sdlc-init --version`.
- License: GPL-3.0-only (`LICENSE`), with SPDX headers in the scripts.

### Changed
- Example IDs in PROTO.md are now generic (`FEAT-0012-user-login`).

## 1.1-template — 2026-10-02 (template only, protocol unchanged)

### Added
- `template/.sdlc/agents/`: generic definitions for all 19 roster agents
  (orchestrator, reviewer, security, bd, triage, prd, data-req, data-flow,
  ux, ui, schema, backend, fe-web, fe-app, docs, qa, devops, release,
  monitor), each with mission, reads/writes, tools, forbidden actions, exit
  criteria, handoff, escalation and spawns.
- `agents/_common.md`: start-up, working, finishing and spawning rules that
  every agent shares (tracing, tokens, permissions).

## 1.1 — 2026-10-02

### Added
- **Run tracing (§6.9):** a run ID for every agent run, a spawn tree via
  `parent_run`, lifecycle events in `trace/runs.jsonl`, per-run detail events,
  gitignored transcripts, orphan detection, `Run-Id` commit trailers, and
  debugging recipes.
- **Token accounting (§6.10):** a required `tokens` object on every terminal
  event, `subtree_total` roll-ups, tokens on every `log.md` line that closes a
  run, per-item/agent/stage totals, and per-run and per-item budgets that
  escalate when exceeded.
- `run_id` in file headers and log lines; a `Spawns:` field in the agent
  template; `tracing` and `tokens` sections in `config.yaml`.

## 1.0 — 2026-10-02

### Added
- Agent roster across REQ, DESIGN, DEV, QA, DEPLOY/RELEASE and a control plane
  (Orchestrator, Reviewer, Security & Compliance).
- Lifecycle state machine with QA as a gate at every environment, routing for
  each kind of failure, and retry budgets.
- Gates, Definition of Done, and a fast lane for bugs and hotfixes.
- The `.sdlc/` protocol directory: separation from project files, layout,
  naming patterns, mandatory front matter, messages, the item log, ownership,
  git policy, and the agent start-up contract.
- Environments and promotion, human approval checkpoints, `config.yaml`, and
  metrics.
