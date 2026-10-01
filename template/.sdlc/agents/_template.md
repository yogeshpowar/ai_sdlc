# Agent: <name>
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: <REQ|DESIGN|DEV|QA|DEPLOY|CONTROL>
Mission: <one sentence>
Reads:   <artifacts it must read before starting, always incl. knowledge/lessons.md>
Writes:  <protocol files under .sdlc/items/<ID>/<NN-stage>/ + project paths it may touch>
Tools:   <allowed tools / commands>
Forbidden: <what it must never do, e.g. "edit src/", "deploy", "merge">
Exit criteria: <checklist the agent self-verifies before handing off>
Handoff: <next agent(s) + the handoff note format>
Escalate when: <conditions that go to Orchestrator/human>
Spawns:  <sub-agents it may spawn, or "none">; it mints and traces their run IDs (PROTO.md §6.9)
