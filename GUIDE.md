# Guide: running a new project with ai_sdlc

This guide is for **you, the human**. It walks through starting a project,
getting the agents working, and what your job is while they run. The rules
the agents follow are in [`PROTO.md`](PROTO.md). You don't need to read it
to start, but section numbers (§) below point into it.

---

## How the work flows (read this first)

You **don't** need to know the whole product up front. The agents build it
in **thin slices**. Each slice is a small thing a real user can do end to
end, and it goes all the way to production before the next one starts.

1. **Slice 1 is a walking skeleton:** a "hello world" deployed to prod
   through the real pipeline, which proves everything works.
2. Each slice after that adds one small, complete capability.
3. **After every release you get a short review** in `.sdlc/outbox/`: what
   shipped, how to try it, what it cost, and what the agents propose next.
   Your reply decides what comes next.

Small slices (S) go straight from a 10-line story to code. Medium ones (M)
get a short PRD that you sign off. Anything bigger is split into an
**epic** of slices. Design agents (data, API, UX/UI, security) only join a
slice when it touches their area. See PROTO.md §3.3–3.8.

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

This gives you `.sdlc/`, the agents' workspace. Everything else in the
repo is **the project**: code, plus its living documentation in `README.md`,
`CONTRIBUTING.md` and `docs/` (architecture, API contract, data model, ADRs,
UX, security, runbook). The agents keep those docs up to date for every
future developer, and never hide them in `.sdlc/` (§6.1). The first item
(the walking skeleton) creates that `docs/` skeleton.

| Where | What | For whom |
|---|---|---|
| project tree (`src/`, `docs/`, `README.md`, …) | the product and how it works **now** | everyone, forever |
| `.sdlc/` | how it was built: backlog, specs, reviews, QA, traces, approvals | the agents and you, for audit and debugging |

The test: delete `.sdlc/`, and a new developer can still understand, build,
run and change the product.

To pin a specific protocol version instead of the latest:
`git -C ~/working/ai_sdlc checkout v1.2` before running `sdlc-init`.
`sdlc-init --version` shows which version you have.

---

## 2. Write the brief (10–20 minutes, the most important step)

Edit **`.sdlc/knowledge/brief.md`**. Everything the agents do traces back to
this file. It's a **direction, not a specification**: you'll refine it after
every release, so don't try to get everything right now. Answer:

1. **What** are we building? (one paragraph)
2. **For whom?** (the users, and what they're trying to do)
3. **Why?** (what problem it solves; what success looks like, in numbers if you can)
4. **The first few things a user should be able to do,** smallest first.
   Say what is explicitly *out* for now.
5. **Constraints:** stack, hosting, budget, deadlines, compliance
   (e.g. PCI-DSS, GDPR, RBI), integrations, anything non-negotiable.
6. **What you're unsure about.** The agents turn these into time-boxed
   spikes instead of guessing.

Example:

```markdown
# Project brief: my-app
A web app where small shop owners list products and take UPI payments.
Users: shop owners (non-technical, mobile-first) and their customers.
Success: a shop goes live in < 15 minutes; 95% of checkouts succeed.
First things a user can do: an owner adds a product; a customer sees the
list; a customer pays by UPI; the owner sees the order.
Out for now: inventory sync, delivery tracking, multiple currencies.
Unsure: which UPI gateway; whether owners want photos on day one.
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
| `human_approvals` | Where you want to say yes before agents continue, **per item size**. By default you sign off M specs (not S stories) and every prod deploy. Once you trust the pipeline you can drop S from `prod_deploy`. |
| `delivery` | Slice sizes (max ACs and tokens for S and M), the walking skeleton, how many items are refined ahead, the WIP limit (default 1: finish before starting), and whether the agents wait for your release-review reply (`wait`) or carry on (`notify`) |
| `compliance` | Any regulations that apply. The security agent checks against these. |
| `tokens` | Budgets per run and per item. Start generous and tighten once you've seen real numbers. |
| `git` | Your main branch name, the remote (`origin`, or `null` if local only) and when to push. Agents commit regularly whatever you choose (see below) |

---

## 4. Review the agents (10 minutes)

`.sdlc/agents/` has a definition for every role. For each **enabled** agent,
skim its file and adjust:
- **Tools:** the real commands for your stack (e.g. `go`, `npm`, `kubectl`).
- **Writes:** where its code goes in *your* repo (e.g. `backend/`, `web/`).

Delete the files of disabled agents if you like. To add a role, copy
`_template.md`. Leave `_common.md` alone: it holds the shared rules
(tracing, tokens, permissions).

Check the `docs:` paths in `config.yaml` (where architecture, the API
contract, ADRs and so on live in *your* repo). If you have existing docs,
point the paths at them, and the agents will keep them up to date.

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
2. Work iteratively (PROTO.md §3.3–3.8): if this is a new project, the
   first item is the walking skeleton. Have bd keep a light roadmap, and
   refine only the next few items into small, functionally complete
   slices (S or M; split anything bigger into an epic).
3. Take the top item through its size track, spawning only the agents it
   needs, each as a sub-agent with its own run ID in a fresh context.
   Respect the WIP limit. After each release, send me the release review
   and re-rank the backlog from my reply.
4. After every event, update state.json, the item's log.md, and run
   .sdlc/bin/sdlc-status --write.
5. Commit as PROTO.md §6.7 requires: code on item branches at every
   checkpoint, .sdlc/ on main after every run, and each of my approvals
   immediately when it lands.
6. Stop and tell me whenever a human approval is needed (outbox/), and
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

You are the **product owner, approver and tie-breaker**. Agents never guess
on your behalf. They write to `outbox/` and wait.

### Steer after every release (your most important job)
After each slice reaches prod, a **release review** lands in
`.sdlc/outbox/<ts>-<ID>-release-review.md`: what shipped, how to try it,
what it cost, what was learned, and the top 3 proposed next items. **Try
it**, then reply with an `answer` file in `.sdlc/inbox/`:

```markdown
---
type: answer
answers: outbox/20261002-1730-FEAT-0003-release-review.md
by: human:<your-name>
date: 2026-10-02
---
Works. Next: payments before photos. Drop "bulk import" for now.
Customers asked for search, so add it to the roadmap.
```

The agents update the roadmap and re-rank the backlog from your reply. By
default (`release_review: notify`) they carry on with their proposal until
your reply arrives. Set `wait` if you want to approve every next step.

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
approves: items/FEAT-0001-product-list/20-design/prd.md@v3   # exactly what the request named (or a commit SHA for prod)
by: human:<your-name>
date: 2026-10-02
---
Approved. Keep the listing page under 1s on 3G.
```

Then tell the Orchestrator "continue" (or just restart it with the kickoff prompt).
**It commits your approval immediately**, before doing anything else
(`sdlc(<ID>): approved … by human:<you>`, with `Approved-By:` and `Approval:`
trailers) and pushes it if a remote is set. You can also commit the file
yourself. Your approval covers the exact version named in the request
(e.g. `prd.md@v3`). If the agents change it afterwards, they have to ask again.

The usual approval points (`config.yaml` → `human_approvals`):
- **Spec sign-off (M items by default).** Read `items/<ID>/20-design/prd.md`
  and check the acceptance criteria are what you want. This is the cheapest
  moment to change your mind. S stories don't wait for you unless you add
  `S` to `human_approvals.spec`.
- **Turning a feature flag on in prod.** A finished epic's slices may have
  shipped dark. Turning them on for users is your call.
- **Prod deploy.** Check `50-qa/qa-report.stage.*.md` and
  `60-deploy/rollback-plan.md`.
- **Destructive migrations, accepted security risks, used-up retry or token
  budgets.** The request says what happened and what the options are.

### Commits you'll see
The agents commit regularly, so nothing is lost if a session dies (§6.7):
- **Code** goes on `feature/<ID>` branches: at each finished acceptance
  criterion, at least every 30 minutes, and always before handoff. These
  branches merge to main only after review.
- **`.sdlc/` records** go on the main branch: after every agent run finishes,
  and **immediately after each of your approvals**.
- `git log --grep "Approved-By"` lists every decision you've made.
- `git log --grep "Item: FEAT-0001"` lists everything done for one item.

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
- Don't leave your own edits to `brief.md`, `config.yaml` or `agents/`
  uncommitted for long. Commit them, or the Orchestrator will (as
  `Changed-By: human:…`) on its next start.
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
| `wip(<ID>): recovered from RUN-…` commits | A run crashed with uncommitted work, and the Orchestrator saved it | Nothing. The next run picks it up. Frequent ones mean runs are dying, so check their transcripts |
| Slices keep getting split or bouncing back | Items are too big or too vague | Answer the agents' questions sooner, and in release reviews favour the smallest next step. Check `delivery.sizes` |
| Lots of design work on tiny changes | Triggers are too eager | Look at the `triggers:` in the item's `story.md` and tell the reviewer via `inbox/` |
| A mistake keeps coming back | The lesson isn't recorded | Check `knowledge/lessons.md`. Add the entry via `inbox/` if the reviewer missed it |

---

## 9. Upgrading a project to a newer protocol

You can upgrade while the project is in active development. **The upgrade
never touches your code, your `docs/`, or the agents' records** (items,
traces, approvals, inbox/outbox, board, lessons). See PROTO.md §6.12.

```sh
git -C ~/working/ai_sdlc pull                     # get the new version (read CHANGELOG.md)
cd ~/working/my-app
# 1. stop the Orchestrator and wait for running agents to finish (sdlc-watch shows "nobody is working")
# 2. commit or stash anything uncommitted
~/working/ai_sdlc/bin/sdlc-upgrade . --dry-run    # see exactly what will change
~/working/ai_sdlc/bin/sdlc-upgrade . --commit     # do it
```

What you'll see in the plan:
- `update`: protocol files you never edited, replaced with the new version.
- `merge`: files you customised (usually `config.yaml`, `agents/*.md`).
  Your edits are kept and the new changes are merged in.
- `CONFLICT`: your edit and the new version touch the same lines. **Your
  file is left exactly as it was**, and `<file>.sdlc-new` (the new version)
  and `<file>.sdlc-merge` (with conflict markers) appear next to it. Edit
  your file, delete the two copies, and commit. `--commit` waits until
  you've done this.
- `create`: files new in this version (e.g. `board/roadmap.md`).
- `skip`: an agent you deleted stays deleted.
- `kept`: a file the new template no longer has. It stays put, and a
  migration step says where its content should go.

If the new version needs real migration work (e.g. 1.4 moves architecture
notes from `.sdlc/` into `docs/`), the tool leaves a request in
`.sdlc/inbox/`. The next time you start the Orchestrator, it turns that into
a `CHORE` item and does the work through the normal flow: branch, review,
commits. Every upgrade is recorded in `.sdlc/UPGRADES.md`.

Changed your mind before committing? `git checkout -- .sdlc .gitignore && git clean -fd .sdlc`.

---

## Quick reference

| I want to… | Do this |
|---|---|
| Start a project | `sdlc-init <dir> <name>`, then write `brief.md`, `config.yaml`, review `agents/`, commit |
| Kick off or resume the agents | Paste the kickoff prompt (§5 of this guide) in the project root |
| See what's happening | `.sdlc/bin/sdlc-watch`, or read `.sdlc/TODO.md` |
| Decide what comes next | Reply to the release review in `.sdlc/outbox/` with an `answer` file in `.sdlc/inbox/` |
| See where the product is going | `.sdlc/board/roadmap.md` |
| Understand how the system works | `README.md` and `docs/`, the same docs the agents read |
| Approve or reject | Same-named `approval` file in `.sdlc/approvals/` |
| Request a feature or report a bug | A file in `.sdlc/inbox/` |
| See an item's whole story | `.sdlc/items/<ID>/` (`item.md`, `log.md`, `messages/`) |
| See what it cost | `item.md` → Tokens; the header of `TODO.md` |
| See what we've learned | `.sdlc/knowledge/lessons.md` |
| Upgrade the protocol | stop agents, commit, then `ai_sdlc/bin/sdlc-upgrade . --dry-run`, then `--commit` |
