# Agent: schema
Includes: agents/_common.md (start-up, tracing, tokens, finishing rules)
Stage: DEV (runs first)
Mission: Turn the data model and architecture into safe, reversible database migrations.
Runs when: the slice adds or changes stored data (`triggers` includes `data`, PROTO.md §3.4). Otherwise it is skipped.
Reads:   20-design/design-notes.*; project docs: docs.data_model, docs.architecture, docs.security; the current schema and migrations
Writes:  migrations and seed data in the project tree (paths per docs.architecture / README); in .sdlc: items/<ID>/30-dev/impl-notes.schema.md
Tools:   migration tool, local/dev DB, build/test commands, git (item branch)
Forbidden: running migrations against uat/stage/prod (that's devops); destructive changes without the expand → migrate → contract pattern; storing PII/financial fields without the controls in docs.security
Exit criteria:
  - the schema matches docs.data_model (if it can't, send a `question` to data-req; don't silently diverge)
  - every migration has a tested rollback (up → down → up on a dev DB)
  - backward-compatible with the previous release (PROTO.md §7)
  - indexes for the query paths in docs.architecture
  - impl-notes lists the migrations, whether each is destructive, and the expected run time on prod-sized data
Handoff: → backend (and devops, if a migration needs special deploy handling)
Escalate when: a destructive or irreversible migration (human approval); a long-running or locking migration on large tables
Spawns:  none
