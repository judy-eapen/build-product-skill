# Timeline (Gantt)

Generate a Gantt-style timeline for an existing PRD and user-stories breakdown, or for a described feature. Use when:
- You have an approved user-stories breakdown and want a roadmap to share with stakeholders or commit a delivery date against.
- You want a timeline for an existing feature without re-running the full pipeline.
- You want to regenerate the timeline after a scope or team-composition change.

Read and follow `ai-framework/06b-timeline.md`.

## Inputs

The underlying prompt collects inputs as Step 0:

- **Required:** a user-stories breakdown (paste, file path, or skip if PRD-only).
- **Recommended:** the PRD, for phase order and any committed milestones.
- Feature name (asked if not in conversation context).
- Timeline parameters (start date, sprint length, team composition, velocity, buffer, optional target launch date and off-time ranges). The skill asks all of these in one turn at Step 2 and applies defaults for any the PM skips.

No prior pipeline run is required, but the timeline quality is highest when a user-stories breakdown exists. PRD-only fallback produces a coarser phase-level view and notes the limitation in the output.

## Output

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/timeline/
├── [feature-name]-timeline.html   ← interactive Gantt, opens offline
└── [feature-name]-timeline.md     ← parameter snapshot + traceability table + Figma URL
```

The HTML file is a self-contained Gantt with hover details per bar, a today marker, and a print stylesheet. The markdown sidecar holds the parameter snapshot, the proposed-timeline table (post-PM tuning), the Figma FigJam URL (if generated), and a traceability table mapping each Gantt bar to its source epic and stories.

If the Figma MCP is connected, a **Figma FigJam timeline** is also generated and the URL is recorded in `_pipeline-state.json` → `export_urls.figma_timeline_url`. If Figma is unavailable, the step still produces the HTML output and notes the Figma skip in the markdown sidecar.

## Granularity

The Gantt shows **Epics + Phases**, not individual stories. A feature with 60 stories does not produce a 60-bar chart — it produces a clean roadmap with one bar per epic, grouped under phase headers. Story-level detail lives in the user-stories breakdown.

## Estimation

Hybrid:
1. The skill proposes durations from the user-stories breakdown's `Size` column and default velocity (8 person-days per dev per sprint).
2. The PM sees the proposed table and tunes any epic ("Epic 1.2 should be 10 days, not 6") or pins phase starts.
3. The skill recomputes and re-presents until the PM accepts.

Math is honest: if the computed end date misses a target launch, the skill surfaces the gap and offers scope cut / team increase / slip as the three options — it does not shrink durations to fit.

## When to use the full pipeline instead

If you have only a feature idea, run `/build-product`. The Timeline step runs automatically after Gate 3 (User Stories Breakdown approval) and before Step 11 (Export), and the resulting Figma URL is embedded in the Confluence publish.
