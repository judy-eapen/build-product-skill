v2.4.0 — 2026-05-21

Two new features in the User Stories Breakdown step:

1. **Multi-epic support.** Step 10 proposes an Epic grouping (one Epic per PRD phase by default, sub-epics for functional clusters); PM accepts or adjusts. Step 11a Jira Export creates one Jira Epic per group with stories assigned correctly. Per-Epic descriptions scoped to that Epic's stories only.

2. **DRAFT mode for stories without finalized designs.** New Step 0.5 design-availability check. If no finalized designs exist, the PM can write stories from the PRD alone — design-dependent stories tagged `Status: DRAFT — needs design`, sized with `*` suffix, exempt from UX state coverage at Gate 3, labeled `draft` in Jira. New `/change-mode` trigger "Designs arrived" refreshes every DRAFT story (AC, sizing, UX coverage) and updates the Jira tickets in place when designs land.

See CHANGELOG.md for full version history.
