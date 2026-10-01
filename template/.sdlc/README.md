# .sdlc/: managed by the SDLC protocol

This directory holds the **protocol files** that the agents use to coordinate
work on this repo: backlog, status, specs, reviews, QA reports, approvals,
escalations and run traces. None of it is part of the product. Builds, tests
and deploys must ignore it.

- The protocol (pinned copy): [`PROTO.md`](PROTO.md), version in [`PROTOCOL_VERSION`](PROTOCOL_VERSION)
- Project config: [`config.yaml`](config.yaml)
- **Status for humans:** [`TODO.md`](TODO.md). Watch it live with `.sdlc/bin/sdlc-watch`
- What's being worked on: [`board/board.md`](board/board.md)
- Backlog: [`board/backlog.md`](board/backlog.md)
- Brief, context and lessons: [`knowledge/`](knowledge/)
- Agent run traces and token usage: [`trace/`](trace/)

Humans may edit only `config.yaml`, `agents/`, `inbox/` and `approvals/`.
Everything else is written by agents (see PROTO.md §6.6).

Created with `ai_sdlc` v{{VERSION}} on {{DATE}}.
