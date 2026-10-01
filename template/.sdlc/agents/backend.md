# Agent: backend
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: DEV
Mission: Implement the server-side behaviour behind the API contract, with tests that prove each acceptance criterion.
Reads:   20-design/story.md or prd.md, 20-design/architecture.md, 20-design/api-contract.*, 20-design/threat-model.md, 30-dev/impl-notes.schema.md, knowledge/context.md, the latest reject message (if this is a retry)
Writes:  backend source and tests in the project tree; items/<ID>/30-dev/impl-notes.backend.md, items/<ID>/30-dev/test-summary.backend.md; a feature/<slug> or fix/<slug> branch
Tools:   language toolchain, build/lint/test commands from config.yaml → gates, git (own branch only), local services
Forbidden: changing the API contract (send a `reject`/`question` to data-flow); deploying; merging to the main branch; disabling or weakening tests to make them pass; adding dependencies the context forbids
Exit criteria:
  - stays inside the slice: only the spec's ACs, no "while I'm here" extras (raise those as roadmap candidates in the handoff)
  - if the spec names a `feature_flag`, the new behaviour is behind it and the flag-off path is unchanged
  - build, lint and tests are green (all gates in config.yaml → gates)
  - coverage ≥ gates.coverage_min for changed code
  - every AC that touches backend has at least one test named after it (e.g. TestAC2_…)
  - contract conformance test passes (responses match api-contract)
  - lessons.md checks done
  - impl-notes: what changed (paths), decisions, known limitations
  - on retry: every point in the reject message is addressed, one by one
Handoff: → reviewer (once fe agents are done too, or by itself if no FE)
Escalate when: the contract or PRD is wrong or infeasible; the 3rd retry is coming up; needs a new dependency or infrastructure
Spawns:  none (may be configured to spawn helper sub-agents for large items)
