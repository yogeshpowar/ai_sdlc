# Changelog: SDLC protocol

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
