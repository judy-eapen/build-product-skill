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

## Editing the timeline interactively (v2.5.0+)

The HTML Gantt is **editable in the browser** — no skill round-trip needed for quick what-if exploration.

**Drag a bar** to shift the start date. Downstream bars cascade automatically (every Epic that originally started after this bar's original end moves by the same amount). Hold **Shift** while dragging to lock other bars — only the bar you're moving will shift.

**Drag the right edge** of a bar to resize its duration. Same cascade rules apply.

**Arrow-key edits**: focus a bar (Tab to it) and use `←` / `→` to shift by 1 working day. Add Shift to resize instead of shift. Add Alt to lock other bars (suppress cascade).

**Auto-saved to localStorage** on every edit. Reload the page and your edits persist.

**Toolbar buttons**:
- **↺ Reset to original** — discards your edits and restores the skill-computed baseline. Confirms first.
- **💾 Save to skill** (v2.6.0+, Chrome/Edge) — writes the plan JSON directly into the feature's `timeline/` folder via the File System Access API. First click pops a one-time permission prompt; subsequent clicks save silently. Falls back to ⬇ Export plan behavior in Safari/Firefox.
- **⬇ Export plan** — downloads a JSON file with your edits, ready to round-trip back into the skill.

### Round-trip edits to the skill

Casual edits in the browser are great for "what if?" exploration, but they don't update the skill's official state (so `/change-mode`, the Confluence Timeline child page, and downstream tools still see the old plan) until you apply them. There are now three ways to do it — pick whichever fits the moment:

**Fastest (v2.6.0+, Chrome/Edge):**
```
1. Edit in browser.
2. Click 💾 Save to skill. First time only: grant write permission for the timeline/ folder.
3. In chat, just type: /timeline apply
   (no path needed — auto-discovers the latest plan JSON)
4. Skill diffs, you confirm, sidecar + HTML re-render.
```

**Cross-browser (works in Safari, Firefox, anywhere):**
```
1. Edit in browser.
2. Click ⬇ Export plan → downloads [feature]-plan-YYYYMMDD-HHMM.json
3. In chat, type: /timeline apply
   (still no path — auto-discovery scans ~/Downloads/ too)
4. Skill diffs, you confirm, sidecar + HTML re-render.
```

**Explicit (if you've moved the file or want to be precise):**
```
/timeline apply ~/path/to/specific-plan.json
```

You can also paste the JSON content inline in chat if you don't want to deal with file paths at all — the skill detects the `build-product-timeline-plan-v1` schema and handles it the same way.

### Where Save-to-skill writes (Chrome/Edge)

First click of 💾 Save to skill opens a native save dialog. **Recommended:** navigate to your feature's timeline folder (`~/Desktop/Resources/PDLC Workflow Docs/[feature]/timeline/`) and save as `[feature]-plan-current.json`. After the first save, the browser remembers the file handle (stored in IndexedDB) so subsequent saves go straight to that file with no dialog.

If you ever want to point Save-to-skill at a different file, use ↺ Reset to original first (clears edits and the saved handle) or click 💾 Save to skill from a fresh browser profile.

### What gets updated when you apply

| Updated | Not updated |
|---|---|
| `timeline.computed` (start, end, working_days, gap) | User stories breakdown (scope unchanged) |
| `timeline.parameters` (per-epic durations) | Jira ticket sizing |
| `timeline.applied_edits[]` (history of applies) | PRD |
| Markdown sidecar (re-rendered with new dates) | Figma FigJam timeline (run `/timeline` to regenerate) |
| HTML (re-rendered; edited positions become new baseline) | Confluence Timeline page (run `/publish-to-confluence`) |

Scope changes still go through `/change-mode`. The Timeline is a "when," not a "what."

## When to use the full pipeline instead

If you have only a feature idea, run `/build-product`. The Timeline step runs automatically after Gate 3 (User Stories Breakdown approval) and before Step 11 (Export), and the resulting Figma URL is embedded in the Confluence publish.
