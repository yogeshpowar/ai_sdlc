# .sdlc/: managed by the SDLC protocol

This directory holds the **protocol files** that the agents use to coordinate
work on this repo: backlog, status, specs, reviews, QA reports, approvals,
escalations and run traces. None of it is part of the product. Builds, tests
and deploys must ignore it.

**Looking for how the product works?** Read `README.md` and `docs/` at the
repo root: architecture, API contract, data model, ADRs, runbook. Those are
the project's living docs, for everyone. This folder is the agents'
process record.

- **How to use this as a human:** [`GUIDE.md`](GUIDE.md)
- The protocol (pinned copy): [`PROTO.md`](PROTO.md), version in [`PROTOCOL_VERSION`](PROTOCOL_VERSION)
- Project config: [`config.yaml`](config.yaml)
- **Status for humans:** [`TODO.md`](TODO.md). Watch it live with `.sdlc/bin/sdlc-watch`
- What's being worked on: [`board/board.md`](board/board.md)
- Backlog: [`board/backlog.md`](board/backlog.md)
- Brief and lessons for agents: [`knowledge/`](knowledge/)
- Agent run traces and token usage: [`trace/`](trace/)

Humans may edit only `config.yaml`, `agents/`, `inbox/` and `approvals/`.
Everything else is written by agents (see PROTO.md §6.6).

Managed by `ai_sdlc`. The protocol version is in `PROTOCOL_VERSION`.
