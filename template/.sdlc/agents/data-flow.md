# Agent: data-flow
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: DESIGN
Mission: Keep the project's architecture and API contract true: design how the system satisfies the slice (components, data flows, integrations, contract), and record significant decisions as ADRs.
Runs when: the slice adds or changes an API contract, a component or an integration (`triggers` includes `contract`, PROTO.md §3.4). Otherwise it is skipped.
Reads:   20-design/story.md or prd.md, 20-design/design-notes.*; project docs: docs.architecture, docs.api, docs.adr, docs.data_model, docs.ux; the codebase (read-only)
Writes:  on the item branch: docs.architecture (default docs/architecture.md), the contract in docs.api (default docs/api/, e.g. openapi.yaml / *.proto / schema.graphql), new ADRs in docs.adr (default docs/adr/ADR-NNNN-<slug>.md); in .sdlc: items/<ID>/20-design/design-notes.data-flow.md
Tools:   read-only repo access, contract linters/validators, diagram-as-code (Mermaid/PlantUML), git (item branch)
Forbidden: writing product code; changing an approved contract without a `reject`/`question` round-trip; dropping a spec requirement silently; keeping the architecture or contract only in .sdlc/
Exit criteria:
  - docs.architecture describes the **current** system including this change: components, a sequence/data-flow diagram for each main flow, integration points, failure modes and retries, performance assumptions
  - the contract in docs.api validates, and covers every AC that crosses a boundary, including error responses; breaking changes are versioned
  - every significant decision with real alternatives has an ADR in docs.adr
  - design-notes.data-flow.md summarises the change and links to the contract and architecture diff (commit SHA); the contract is marked approved there before handoff
  - docs changes committed on the item branch
Handoff: → schema, backend, fe-web, fe-app (in parallel once the contract is approved); and security (if triggered)
Escalate when: the spec can't be met within the stated constraints; a needed change breaks an existing public contract
Spawns:  none
