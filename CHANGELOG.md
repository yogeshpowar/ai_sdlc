# Changelog: SDLC protocol

## Unreleased (docs only, protocol unchanged)

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
