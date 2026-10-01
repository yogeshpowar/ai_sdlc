# Guide: running a new project with ai_sdlc

This guide is for **you, the human**. It walks through starting a project,
getting the agents working, and what your job is while they run. The rules
the agents follow are in [`PROTO.md`](PROTO.md). You don't need to read it
to start, but section numbers (§) below point into it.

---

## 0. What you need

- `git`, `bash` and `python3` (for `sdlc-init`, `sdlc-status` and `sdlc-watch`)
- An **agent runner that can read and write files, run commands and spawn
  sub-agents**, e.g. Claude Code (`claude`). The examples below use Claude
  Code, but any runner that can do those three things works.
- A clear idea of what you want to build. Ten good sentences are enough.

---

## 1. Create the project (2 minutes)

```sh
mkdir -p ~/working/my-app && cd ~/working/my-app
git init
~/working/ai_sdlc/bin/sdlc-init . my-app
git add -A && git commit -m "sdlc: initialise protocol"
```

This gives you `.sdlc/`, the agents' workspace. Your product code will live
everywhere else in the repo, and the agents never mix the two (§6.1).

To pin a specific protocol version instead of the latest:
`git -C ~/working/ai_sdlc checkout v1.2` before running `sdlc-init`.
`sdlc-init --version` shows which version you have.

---

## 2. Write the brief (10–20 minutes, the most important step)

Edit **`.sdlc/knowledge/brief.md`**. Everything the agents do traces back to
this file. Answer:

1. **What** are we building? (one paragraph)
2. **For whom?** (the users, and what they're trying to do)
3. **Why?** (what problem it solves; what success looks like, in numbers if you can)
4. **Scope of the first release:** must-haves, plus what is explicitly *out*.
5. **Constraints:** stack, hosting, budget, deadlines, compliance
   (e.g. PCI-DSS, GDPR, RBI), integrations, anything non-negotiable.

Example:

```markdown
# Project brief: my-app
A web app where small shop owners list products and take UPI payments.
Users: shop owners (non-technical, mobile-first) and their customers.
Success: a shop goes live in < 15 minutes; 95% of checkouts succeed.
First release: product listing, cart, UPI checkout, order list for the owner.
Out of scope: inventory sync, delivery tracking, multiple currencies.
Constraints: Go backend, React web, Postgres, deploy on our k8s; RBI rules
for payment data; no card data stored by us.
```

Set `status: approved` in its front matter when you're happy with it.

---

## 3. Configure the project (10 minutes)

Edit **`.sdlc/config.yaml`**:

| Setting | What to put |
|---|---|
| `stack` | Languages, frameworks, DB. Use `none` for parts you don't have |
| `agents.enabled` / `disabled` | Turn off what you don't need. A CLI tool doesn't need `ux`, `ui`, `fe-web`, `fe-app` or `monitor`. A project with no database doesn't need `schema`. |
| `environment_mapping` | What uat, stage and prod *actually are* for you: git branches, k8s namespaces, URLs |
| `gates` | The real commands for build, lint, test and smoke, e.g. `go test ./...` or `npm test` |
| `human_approvals` | Where you want to say yes before agents continue. The default is PRD sign-off and prod deploys. |
| `compliance` | Any regulations that apply. The security agent checks against these. |
| `tokens` | Budgets per run and per item. Start generous and tighten once you've seen real numbers. |

---

## 4. Review the agents (10 minutes)

`.sdlc/agents/` has a definition for every role. For each **enabled** agent,
skim its file and adjust:
- **Tools:** the real commands for your stack (e.g. `go`, `npm`, `kubectl`).
- **Writes:** where its code goes in *your* repo (e.g. `backend/`, `web/`).

Delete the files of disabled agents if you like. To add a role, copy
`_template.md`. Leave `_common.md` alone: it holds the shared rules
(tracing, tokens, permissions).

Optionally fill in `.sdlc/knowledge/context.md` (architecture, conventions).
If you leave it, the agents fill it in as they go.

Commit:

```sh
git add -A && git commit -m "sdlc: brief, config and agents for my-app"
```

---

## 5. Start the Orchestrator

Open your agent runner **in the project root** and give it this kickoff
prompt. With Claude Code: `cd ~/working/my-app && claude`, then paste:

```text
You are the Orchestrator for this project. Follow .sdlc/PROTO.md exactly,
acting as the agent defined in .sdlc/agents/orchestrator.md (and
.sdlc/agents/_common.md).

Your run ID is RUN-<YYYYMMDDTHHMMSSZ>-orchestrator-<4 hex>. Mint it now and
write your spawned/started events to .sdlc/trace/runs.jsonl.

1. Read .sdlc/knowledge/brief.md and .sdlc/config.yaml.
2. If the backlog is empty, spawn the bd agent to turn the brief into
   opportunities, then build .sdlc/board/backlog.md from them.
3. Take the first todo item through the lifecycle, spawning each stage's
   agent as a sub-agent with its own run ID, in its own fresh context.
4. After every event, update state.json, the item's log.md, and run
   .sdlc/bin/sdlc-status --write.
5. Stop and tell me whenever a human approval is needed (outbox/), and
   continue when I say so.
```

To continue later, use the same prompt. The Orchestrator rebuilds where it
was from `.sdlc/board/state.json` and `trace/`, so you never have to
re-explain anything.

---

## 6. Watch the show

In a second terminal:

```sh
cd ~/working/my-app && .sdlc/bin/sdlc-watch
```

You'll see, refreshed every 2 seconds:
- **Live:** which agent is working, on which item, for how long, how far
  along, and its current step in plain words
- **Recent:** the last spawns and finishes (▶️ ✅ ❌ ⌛ 🙋), with token costs
- **Board:** every item in [TODO.MD](https://github.com/doublefreein/TODO.MD)
  format: `[o]` building, `[O]` in review or QA, `[u]` in UAT, `[s]` in
  staging, `[p]` in prod, `[X]` done, `[B]` **waiting for you**, `[f]` failed

The same report is saved in **`.sdlc/TODO.md`**, so you can read it later
or from your phone in the git host.

---

## 7. Your job while agents work

You are the **approver and tie-breaker**. Agents never guess on your behalf.
They write to `outbox/` and wait.

### Approve or reject something
When `TODO.md` shows `[B]`, or the Orchestrator tells you, open the request
in `.sdlc/outbox/`, e.g. `20261002-1430-FEAT-0001-approval-request.md`. Then
create the file **with the same name, but `approval` instead of
`approval-request`**, in `.sdlc/approvals/`:

```markdown
---
type: approval
item: FEAT-0001-product-list
decision: approved          # approved | rejected | changes-requested
by: human:<your-name>
date: 2026-10-02
---
Approved. Keep the listing page under 1s on 3G.
```

Then tell the Orchestrator "continue" (or just restart it with the kickoff prompt).

The usual approval points (`config.yaml` → `human_approvals`):
- **PRD sign-off.** Read `items/<ID>/20-design/prd.md` and check the
  acceptance criteria are what you want. This is the cheapest moment to
  change your mind.
- **Prod deploy.** Check `50-qa/qa-report.stage.*.md` and
  `60-deploy/rollback-plan.md`.
- **Destructive migrations, accepted security risks, used-up retry or token
  budgets.** The request says what happened and what the options are.

### Ask for something new, or report a bug
Drop a file in **`.sdlc/inbox/`**, named `<YYYYMMDD-HHMM>-new-<kind>.md`:

```markdown
---
type: request            # request | bug | answer
by: human:<your-name>
date: 2026-10-02
---
Bug: checkout button does nothing on Safari 17 (iPhone). Steps: add any
item → cart → Checkout. Expected: payment sheet. Actual: nothing.
```

The Orchestrator routes requests to bd, bugs to triage (the fast lane), and
answers to whichever agent asked.

### Answer an agent's question
Questions show up in `outbox/` (or in the runner's chat). Answer with an
`answer` file in `inbox/` that names the question's file, or reply in the chat.

### What *not* to do
- Don't hand-edit anything under `.sdlc/` except `brief.md`, `config.yaml`,
  `agents/`, `inbox/` and `approvals/` (§6.6). Everything else is the
  agents' record. If it looks wrong, say so in `inbox/`.
- Don't fix agents' code on their branches yourself. File it as a bug or
  review comment so the loop (and `lessons.md`) learns from it.

---

## 8. When something looks wrong

| You see | What it means | What to do |
|---|---|---|
| ⚠️ "quiet for 10m" on a Live line | The agent stopped sending heartbeats | Wait for `run_timeout_min`. The Orchestrator marks it `timed_out` and retries. If it keeps happening, check that agent's transcript in `trace/transcripts/` |
| The same item bouncing `[f]` → `[o]` | Dev ↔ QA rework | After `retry_budget` rounds it escalates to you. Read the `qa-report.*.r<N>.md` files |
| Tokens climbing fast | A loop or an oversized task | Check `item.md` → Tokens by agent. Lower `tokens.per_item_max` or split the item |
| An agent did something odd | Debug the run | `grep <RUN-ID> .sdlc/trace/runs.jsonl`, then read `trace/runs/<date>/<RUN-ID>.jsonl` (§6.9 has ready-made commands) |
| A mistake keeps coming back | The lesson isn't recorded | Check `knowledge/lessons.md`. Add the entry via `inbox/` if the reviewer missed it |

---

## 9. Upgrading a project to a newer protocol

```sh
git -C ~/working/ai_sdlc log --oneline --decorate   # see what's new (CHANGELOG.md)
cp ~/working/ai_sdlc/PROTO.md ~/working/ai_sdlc/GUIDE.md .sdlc/
cp ~/working/ai_sdlc/template/.sdlc/bin/* .sdlc/bin/
cat ~/working/ai_sdlc/VERSION > .sdlc/PROTOCOL_VERSION
git add -A && git commit -m "sdlc: upgrade protocol to $(cat .sdlc/PROTOCOL_VERSION)"
```

Read the CHANGELOG entry first. A new version may add fields to
`config.yaml` or `_common.md` that you should copy over as well.

---

## Quick reference

| I want to… | Do this |
|---|---|
| Start a project | `sdlc-init <dir> <name>`, then write `brief.md`, `config.yaml`, review `agents/`, commit |
| Kick off or resume the agents | Paste the kickoff prompt (§5 of this guide) in the project root |
| See what's happening | `.sdlc/bin/sdlc-watch`, or read `.sdlc/TODO.md` |
| Approve or reject | Same-named `approval` file in `.sdlc/approvals/` |
| Request a feature or report a bug | A file in `.sdlc/inbox/` |
| See an item's whole story | `.sdlc/items/<ID>/` (`item.md`, `log.md`, `messages/`) |
| See what it cost | `item.md` → Tokens; the header of `TODO.md` |
| See what we've learned | `.sdlc/knowledge/lessons.md` |
