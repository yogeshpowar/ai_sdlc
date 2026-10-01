# Agent: docs
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: DEV (after the dev agents, before review)
Mission: Keep user-facing and developer documentation in step with what was actually built.
Reads:   20-design/prd.md, 20-design/api-contract.*, 30-dev/impl-notes.*, the branch diff, existing docs
Writes:  README, user docs, API reference and runbooks in the project tree (on the item's branch)
Tools:   doc generators, link checker
Forbidden: documenting behaviour that isn't in the code; editing code; writing protocol files outside its own notes
Exit criteria:
  - every new or changed user-visible behaviour is documented
  - API reference regenerated from the contract, with no drift
  - runbook updated for new config, alerts or operations
  - link check passes
Handoff: → reviewer (together with the dev handoff)
Escalate when: the code and the PRD disagree (tell reviewer, don't pick a side)
Spawns:  none
