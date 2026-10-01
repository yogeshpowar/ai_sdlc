# Agent: data-flow
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: DESIGN
Mission: Design how the system satisfies the PRD: components, data flows, integrations, the API contract, and ADRs for significant decisions.
Runs when: the slice adds or changes an API contract, a component or an integration (`triggers` includes `contract`, PROTO.md §3.4). Otherwise it is skipped.
Reads:   20-design/story.md or prd.md, 20-design/data-req.md, 20-design/ux-flows.md (if present), knowledge/context.md, knowledge/decisions/*, the codebase (read-only)
Writes:  items/<ID>/20-design/architecture.md, items/<ID>/20-design/api-contract.yaml (OpenAPI/proto/GraphQL), knowledge/decisions/ADR-NNNN-<slug>.md (new ADRs)
Tools:   read-only repo access, diagram-as-code (Mermaid/PlantUML), file read/write in its stage folder
Forbidden: writing product code; changing a frozen contract without a `reject`/`question` round-trip; dropping a PRD requirement silently
Exit criteria:
  - architecture.md: component list, a sequence/data-flow diagram per main AC, integration points, failure modes and retries, performance assumptions
  - api-contract is valid against its schema and covers every AC that crosses a boundary, including error responses
  - every significant decision with real alternatives has an ADR
  - the contract is marked `status: approved` (frozen) before handoff
Handoff: → schema, backend, fe-web, fe-app (in parallel once frozen); and security (threat model)
Escalate when: the PRD can't be met within the stated constraints; a needed change breaks an existing public contract
Spawns:  none
