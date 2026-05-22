v2.3.0 — 2026-05-21

Three new features:

1. Confluence Publish now writes the entire feature workspace — parent hub page + one numbered child page per artifact (Research, Codebase Review, PRD, Product/Technical Review, System Design, Visual Diagram, Design Catalog per phase, User Stories, Timeline). Per-file mtime change detection republishes only the pages whose source actually changed. The command is `/publish-to-confluence`; the older `/prd-to-confluence` was removed in this release.

2. New Step 12 / `/export-transcript` writes the full back-and-forth between the PM and the model for a feature's pipeline run to two markdown files (clean reading version + full forensic version with tool calls). Reads the live Claude Code session JSONL on disk.

3. Pipeline timing instrumentation + `/pipeline-timing` report. Every step records start/end timestamps; gate steps record presented/approved times. Report shows wall-clock total, active-work total (gate waits excluded), per-step breakdown, and per-gate wait time. Embedded in the transcript and Confluence hub page automatically.

See CHANGELOG.md for full version history.
