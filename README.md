# ai_sdlc

A project-independent protocol for running a software project with multiple
specialized AI agents (BD, PRD, design, dev, QA, DevOps, release, …), each
working one stage of the SDLC with its own context. Work is delivered
**iteratively**: thin, functionally complete slices go all the way to
production, one after another, and the human steers after every release. They coordinate only
through versioned files in the project's `.sdlc/` directory.

The protocol itself is [`PROTO.md`](PROTO.md). This repo is its canonical home.

## What's here

| Path | What it is |
|---|---|
| **`GUIDE.md`** | **Start here:** step-by-step instructions for humans starting and running a project |
| `PROTO.md` | The protocol: roles, lifecycle, gates, file layout, run tracing, token accounting |
| `VERSION` | Protocol version (currently 1.4) |
| `CHANGELOG.md` | What changed between protocol versions |
| `template/.sdlc/` | Blank `.sdlc/` skeleton that gets copied into each new project |
| `template/.sdlc/agents/` | Generic definitions for every roster agent, plus `_common.md` (rules all agents share) |
| `bin/sdlc-init` | Sets up the protocol in a project |
| `bin/sdlc-upgrade` | Upgrades a project's `.sdlc/` to this version without touching its code, docs or agent records |
| `LICENSE` | GPL-3.0-only |

## Get it

```sh
git clone https://github.com/yogeshpowar/ai_sdlc.git    # anywhere you like
export AI_SDLC="$PWD/ai_sdlc"                            # add these two lines to
export PATH="$AI_SDLC/bin:$PATH"                         # ~/.bashrc or ~/.zshrc
sdlc-init --version                                      # check: prints the protocol version
```

To use a specific release, `git -C "$AI_SDLC" checkout v1.4` (any tag).
To update later, `git -C "$AI_SDLC" pull`. Both scripts work from any
directory, and also through a symlink (e.g. in `~/bin`).

## Starting a new project

> Full walkthrough, including the kickoff prompt, approvals, the inbox and
> troubleshooting: **[GUIDE.md](GUIDE.md)**. `sdlc-init` also copies it into
> each project's `.sdlc/`.

```sh
sdlc-init <path-to-new-project> [project-name]
```

This creates `<new-project>/.sdlc/` with:
- a **pinned copy** of `PROTO.md` and `PROTOCOL_VERSION`, so agents need
  nothing outside the repo and a protocol upgrade never changes a running
  project by surprise
- `config.yaml`, `board/` (backlog, state, board), `knowledge/` (brief,
  context, lessons, glossary), `agents/` (19 generic agent definitions plus
  `_common.md` and `_template.md`), empty `items/`,
  `inbox/`, `outbox/`, `approvals/`, `archive/` and `trace/`
- `bin/sdlc-status` and `bin/sdlc-watch` (the human status view) and a first `TODO.md`
- the project name and date filled in, and `.sdlc/tmp/` and
  `.sdlc/trace/transcripts/` added to the project's `.gitignore`

It refuses to run if `.sdlc/` already exists. `sdlc-init --version` prints the protocol version.

Releases are tagged `v<VERSION>` (e.g. `v1.2`). To start a project on a
specific version, check out its tag before running `sdlc-init`.

Then:
1. Write `.sdlc/knowledge/brief.md` (what, for whom, why, constraints).
2. Fill in `.sdlc/config.yaml`: stack, enabled agents, environments, gates, budgets.
3. Review `.sdlc/agents/`: adjust each enabled agent's `Tools` and project
   paths to your stack, and delete or ignore the disabled ones. Add new roles
   from `_template.md`.
4. Start the Orchestrator. It turns the brief into the backlog and runs items
   through the lifecycle.
5. Watch it work: `.sdlc/bin/sdlc-watch` in a spare terminal shows who is
   working, on what, how far along and at what token cost, refreshed every 2 s.
   `.sdlc/TODO.md` has the same report as a file
   ([TODO.MD](https://github.com/doublefreein/TODO.MD) format).

## Upgrading a project

```sh
git -C "$AI_SDLC" pull                    # get the latest version
sdlc-upgrade <path-to-project> --dry-run  # the plan
sdlc-upgrade <path-to-project> --commit   # do it
```

Projects stay on their pinned version until you run this. It replaces the
protocol files, three-way-merges the ones you customised (your copy is kept
on conflict), creates new files, and never touches code, `docs/`, items,
traces, approvals or the board. It refuses while agents are running or with
uncommitted changes. Manual migration steps go to the project's `inbox/`
for the Orchestrator. Details: PROTO.md §6.12 and GUIDE.md §9.

## Changing the protocol

Edit `PROTO.md` and/or the template here, bump `VERSION`, add a
`CHANGELOG.md` entry, and **tag the release `v<VERSION>`**. The tag is the
merge base that `sdlc-upgrade` uses for projects on that version. If
projects need more than a file update, add an "Upgrading from …" section to
the entry.

## License

Copyright © 2026 Doublefree.in and contributors.

The files in this repository are licensed under the **GNU General Public
License v3.0 only** (GPL-3.0-only). See the [`LICENSE`](LICENSE) file for the
full text.

If you copy or modify code from this repository, including the scripts that
`sdlc-init` copies into a project's `.sdlc/bin/`, follow the GPL accordingly.
Projects that only follow the protocol, without incorporating its
GPL-licensed code, are not covered by the GPL through the protocol text alone.
