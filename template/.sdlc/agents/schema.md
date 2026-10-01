# Agent: schema
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: DEV (runs first)
Mission: Turn data-req and architecture into safe, reversible database migrations.
Reads:   20-design/data-req.md, 20-design/architecture.md, 20-design/threat-model.md, the current schema and migrations
Writes:  migrations and seed data in the project tree (paths per knowledge/context.md); items/<ID>/30-dev/impl-notes.schema.md
Tools:   migration tool, local/dev DB, build/test commands
Forbidden: running migrations against uat/stage/prod (that's devops); destructive changes without the expand → migrate → contract pattern; storing PII/financial fields without the controls in data-req/threat-model
Exit criteria:
  - every migration has a tested rollback (up → down → up on a dev DB)
  - backward-compatible with the previous release (PROTO.md §7)
  - indexes for the query paths in architecture.md
  - impl-notes lists the migrations, whether each is destructive, and the expected run time on prod-sized data
Handoff: → backend (and devops, if a migration needs special deploy handling)
Escalate when: a destructive or irreversible migration (human approval); a long-running or locking migration on large tables
Spawns:  none
