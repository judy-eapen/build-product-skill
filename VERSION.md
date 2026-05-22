v2.7.0 — 2026-05-22

New `/pipeline-doctor` diagnostic command. Scans the skill itself (every step in `pipeline-configs.yaml` has matching prose in `SKILL.md` + `subprompts/build-product.md`; every quality_check is defined; every instruction file exists; every step block has an explicit "Next:" handoff), feature workspaces (state-file schema, artifact-path existence on disk, gate-state coherence, DRAFT/epic/Confluence cross-checks), slash command coverage, and stale features. Read-only by default; per-finding fix approval; writes a timestamped report.

The class of bug it catches: orchestrator drift where new steps land in `pipeline-configs.yaml` but never get added to `subprompts/build-product.md` — exactly the failure mode that caused the v2.2.0 Step 10.5 / v2.3.0 Step 12 silent stall.

See CHANGELOG.md for full version history.
