# Common rules for every agent

Every agent definition in this folder includes these rules. If an agent's
own file conflicts with this one, the agent's file wins, except on tracing,
tokens and permissions, which are never relaxed.

## Start-up (PROTO.md §6.8)
1. Take the `RUN-ID` your spawner gave you. If you have none, stop.
2. Check `.sdlc/PROTOCOL_VERSION`. Stop if you don't support it.
3. Read this file, your own `agents/<you>.md`, `config.yaml` and
   `knowledge/lessons.md`.
4. Read the latest message addressed to you in `items/<ITEM-ID>/messages/`.
5. Read only the artifacts in your `Reads`. Check each header's `status`, and
   don't build on a `draft` or `rejected` upstream artifact.
6. Write the `started` event. Log every read as a `read` detail event.

## While working
- **Two homes, never mixed (PROTO.md §6.1).** Durable knowledge, meaning the
  architecture, API contract, data model, ADRs, UX flows and screens, threat
  model, glossary, runbook, README and CONTRIBUTING, lives in the **project
  tree** (`config.yaml → docs`) and is edited on the item branch, like code.
  `.sdlc/` holds only process and history (specs, design notes, reviews,
  reports, messages, traces). A design note in `.sdlc/` summarises a change
  and links to the docs diff. It is never the only copy.
- Read the project's own docs (README, CONTRIBUTING, the docs your `Reads`
  names) the same way a human developer would, before relying on anything
  in `.sdlc/`.
- Write protocol files **only** in your own stage folder, and only under
  `.sdlc/`. Write project files only in the paths your `Writes` allows.
- Every protocol file you write has the full front matter (§6.4) with your
  `run_id`.
- Upstream artifacts are read-only for you. To change one, send a `question`
  or `reject` message to its owner.
- Record non-obvious choices as `decision` detail events, with the reason.
- **Keep the human informed:** write a `progress` detail event (`step` = what
  you are doing now, in plain words a human understands, plus optional `pct`)
  at each meaningful step and at least every `tracing.heartbeat_min` minutes.
  This feeds the Live view in `TODO.md` and `bin/sdlc-watch` (PROTO.md §6.11).
- Never put secrets, credentials or personal data in any protocol file,
  trace or transcript. Mask them as `****`.
- **Commit early and often (PROTO.md §6.7).** Product work goes on the item
  branch (`feature/<ITEM-ID>`), never the main branch, and never with `.sdlc/`
  changes in it. Commit at every checkpoint: an AC done and green, a
  migration written, or at least every `git.checkpoint_min` minutes. Messages
  are `<type>(<ITEM-ID>): …` with trailers `Run-Id: <RUN-ID>` and
  `Item: <ITEM-ID>`. `.sdlc/` files are committed by the Orchestrator.
- Never force-push, rewrite or amend shared history, and never commit secrets.
- Stay inside `tokens.per_run_max`. If you are about to exceed it, stop and
  escalate (reason `token_budget`).

## Finishing
0. Commit all your work (no dirty tree, ever). Push it if `git.remote` is set.
   Your handoff message names the final commit SHA.
1. Self-check your **Exit criteria**. If any fail, don't hand off. Fix them,
   or end with `failed` and a clear `reason`.
2. Write exactly one message (§6.5): a `handoff` on success, or a `reject`,
   `question` or `escalation` otherwise.
3. Write exactly one terminal event (`completed` | `failed` | `escalated`)
   with the full `tokens` object (§6.10). If you spawned sub-agents, wait for
   their terminal events first and include them in `subtree_total`.

## Spawning sub-agents
Only spawn agents listed in your `Spawns`. For each one: mint its `RUN-ID`,
write its `spawned` event (with `parent_run` = your run), give it its task
and inputs, and wait for its terminal event.
