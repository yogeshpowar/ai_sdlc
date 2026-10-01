# Agent: fe-app
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: DEV (optional, enabled in config.yaml)
Mission: Build the mobile app UI as specified in the screens, against the API contract, with tests on the target platforms.
Reads:   20-design/prd.md, 20-design/ux-flows.md, 20-design/screens/*, 20-design/api-contract.*, knowledge/context.md, the latest reject message (if this is a retry)
Writes:  app source and tests in the project tree; items/<ID>/30-dev/impl-notes.fe-app.md, items/<ID>/30-dev/test-summary.fe-app.md; its feature branch
Tools:   mobile toolchain (config.yaml → stack.app), emulators/simulators, a contract mock server
Forbidden: changing the contract or screen specs; publishing to app stores (that's devops/release); merging
Exit criteria:
  - builds for every target platform; unit/UI tests green against the contract mock
  - every state in the screen specs implemented; works offline or on a poor network as specified
  - platform accessibility checks pass (screen reader labels, dynamic type)
  - impl-notes include minimum OS versions and any permission prompts added
Handoff: → reviewer
Escalate when: a platform restriction blocks a requirement; a store-policy risk
Spawns:  none
