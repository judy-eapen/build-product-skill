# Timeline (Gantt)

Runs after Gate 3 (User Stories Breakdown approval) and before Step 11 (Export) when called from the orchestrator, OR can be called standalone via `/timeline` when a PM has a PRD and an approved user-stories breakdown and wants a Gantt-style timeline without re-running the full pipeline.

Produces a high-level timeline at the **Epic + Phase** level (not individual stories). Two outputs:

- **Figma FigJam timeline** — created via the Figma MCP. Lives in Figma, can be shared with stakeholders, and embeds in Confluence.
- **Interactive HTML Gantt** — self-contained HTML file (vanilla HTML/CSS/JS, no external dependencies) the PM can open locally or share as a file. Includes hover details, today-marker, and a printable view.

Estimation is **hybrid**: the skill proposes durations from story sizing + a default velocity, then the PM tunes per-epic before the final Gantt is rendered.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Mode dispatch (run this first)

If the PM invoked the prompt with `apply [path-to-json]` (or "apply this plan: [path]", or pasted JSON content), this is **apply mode** — skip the normal generation flow and jump to the dedicated **Apply Mode** section at the end of this file. Otherwise, continue with Step 0.

Apply mode is triggered when:
- The PM message contains `/timeline apply [path]` or `apply [path]`
- The PM pasted JSON matching the `build-product-timeline-plan-v1` schema (with a `"schema"` field set to that value)
- The PM said "apply my edits" or "save the plan" along with a path or pasted JSON

If you're not sure whether this is an apply call, ask: "Are you applying an edited plan JSON, or generating a fresh timeline?"

---

## Step 0 — Input Check (gracefully handle standalone calls)

Before doing anything else, determine whether you have the required inputs.

**Required input:**
- The user-stories breakdown (provides sizing + FE/BE split + phase grouping + dependencies).

**Recommended input:**
- The PRD (provides phase order and any explicit dates / milestones the PM has already committed to).

### If running inside the orchestrator (called from `/build-product`)

All inputs are already in conversation context:
- User stories: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md`
- PRD: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`
- Pipeline state: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_pipeline-state.json`

Skip Step 0 and proceed to Step 1.

### If running standalone (called via `/timeline`)

Ask the PM:

> "To build a timeline I need a user-stories breakdown at minimum, ideally also the PRD.
>
> **Required:**
> - Paste the user-stories breakdown or give me the file path (e.g., `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md`).
>
> **Recommended:**
> - PRD file path or pasted content — gives me phase order and any committed milestones.
>
> If you only have a PRD (no breakdown yet), I can still produce a phase-level timeline, but the per-epic durations will be coarser. Tell me what you have and I'll work with that."

Wait for the PM's response. Also ask for the feature name (used for the output file paths). Apply the standard feature-name derivation rule (spaces → hyphens, lowercase).

If the PM provides neither a breakdown nor a PRD, stop. State: "I need at least a PRD or user-stories breakdown to produce a timeline. Please paste one, give me a file path, or run `/build-product` / `/user-stories` first."

---

## Step 1 — Extract structure from inputs

From the user-stories breakdown, extract:
- **Phases** in PRD order (Phase 1, Phase 2, ...).
- For each phase, the **Epics** if explicitly grouped; otherwise treat each phase as a single epic.
- Per epic: list of stories with `Type` (FE / BE), `Size` (S / M / L / L+), and `Depends on` references.

Convert sizes to a baseline number of working days using these defaults (the PM can override at Step 2):

| Size | Default days |
|------|--------------|
| S    | 1            |
| M    | 3            |
| L    | 5            |
| L+   | 8            |

For each epic, compute two raw totals:
- **FE-days** = sum of days across all FE stories in the epic.
- **BE-days** = sum of days across all BE stories in the epic.

Carry phase-level dependencies forward: Phase N+1 cannot start until Phase N's last epic ends, unless the PRD explicitly notes parallel phase execution.

If the user-stories breakdown is missing (PRD-only fallback), use phase descriptions as a proxy and estimate epic-level effort as a single bucket per phase. Note this limitation in the output: "Estimated from PRD only — no story-level sizing available."

---

## Step 2 — Gather timeline parameters from the PM

Ask the PM the following questions in one turn (group them so the PM can answer all at once):

> "To compute the timeline I need a few inputs. Answer all at once or one at a time:
>
> 1. **Start date** — when does the work begin? (default: today, [YYYY-MM-DD])
> 2. **Sprint length** — 1 week / 2 weeks / 3 weeks? (default: 2 weeks)
> 3. **Team composition** — how many FE devs and BE devs are assigned? (default: 1 FE + 1 BE)
> 4. **Velocity** — how many person-days per dev per sprint? (default: 8 — i.e., a 10-day sprint with 2 days of overhead)
> 5. **Buffer** — what percentage padding for unknowns / sick days / context switching? (default: 15%)
> 6. **Target launch date** — optional. If you have a committed launch, share it and I'll flag any gap.
> 7. **Known off-time / holidays** — optional. List date ranges to exclude (e.g., '2026-12-20 to 2026-12-31').
> 8. **Size-to-days overrides** — optional. The defaults are S=1, M=3, L=5, L+=8. Override any that don't fit your team's reality.
>
> Anything you skip uses the default in parentheses."

Wait for the PM's response. Apply defaults for anything not specified. Persist all parameters to `_pipeline-state.json` → `timeline.parameters` so subsequent runs (e.g., after `/change-mode`) can reuse them.

---

## Step 3 — Compute the proposed timeline

For each epic, compute its duration in working days:

```
fe_capacity_per_day = fe_devs        # 1 dev-day per FE dev per working day
be_capacity_per_day = be_devs
fe_epic_days        = fe_days / fe_capacity_per_day   # 0 if no FE work
be_epic_days        = be_days / be_capacity_per_day
epic_days           = max(fe_epic_days, be_epic_days) # FE and BE run in parallel within an epic
epic_days_buffered  = ceil(epic_days * (1 + buffer_pct))
```

Lay epics out **sequentially within a phase** unless the breakdown's Build Sequence Map shows independent epics — in that case, lay them in parallel. Phases run sequentially unless the PRD says otherwise.

Apply the sprint length to align epic start/end dates to sprint boundaries (round each epic's end date up to the next sprint-end). This produces clean sprint-aligned milestones.

Skip weekends. Skip any date ranges the PM listed as off-time.

Compute three derived values per epic:
- **Start date** (working-day-adjusted).
- **End date** (working-day-adjusted, sprint-aligned).
- **Duration in calendar days** (for display) and **working days** (for math).

Compute one rollup per phase:
- Phase start = earliest epic start in that phase.
- Phase end = latest epic end in that phase.

Compute the overall feature timeline:
- Feature start = Phase 1 start.
- Feature end = last phase end.
- If the PM provided a target launch date, compute the gap: `gap_days = target_launch - feature_end`. Positive = ahead of target, negative = behind target.

---

## Step 4 — Present the proposed timeline for PM tuning

Show the PM a compact table for review **before** generating any visuals. Format:

```
Proposed timeline — [feature-name]
Start: [YYYY-MM-DD] · End: [YYYY-MM-DD] · Sprints: [N] · Working days: [N]
[Target: [YYYY-MM-DD] · Gap: [+/- N days]]   ← only if PM gave a target

PHASE 1 — [Phase name]   ([start] → [end], [N] working days)
  Epic 1.1 — [Epic name]              [start] → [end]   FE [N]d / BE [N]d   buffered [N]d
  Epic 1.2 — [Epic name]              [start] → [end]   FE [N]d / BE [N]d   buffered [N]d

PHASE 2 — [Phase name]   ([start] → [end], [N] working days)
  Epic 2.1 — [Epic name]              [start] → [end]   FE [N]d / BE [N]d   buffered [N]d
  ...

Tune anything before I render the Gantt:
  • "Epic 1.2 should be 10 days, not 6" — overrides one epic
  • "Phase 2 starts 2026-08-01" — pins a phase start
  • "Add a 5-day buffer between Phase 1 and Phase 2" — inserts gap
  • "Looks good" — proceeds to render

What needs to change?
```

Wait for the PM's response. Apply any overrides and recompute. Re-present the updated table. Loop until the PM says "looks good" or equivalent.

If the PM provided a target launch date and the computed gap is negative (behind target), explicitly call this out at the top of the table:

```
⚠ TIMELINE EXCEEDS TARGET by [N] working days. Options:
  1. Compress scope (drop stories or move to a later phase) — re-run /change-mode
  2. Increase team size (add FE or BE devs)
  3. Accept the slip and update the target date
```

Do **not** silently fit-to-target by shrinking durations. The math should be honest.

---

## Step 5 — Generate the Figma FigJam timeline

Load the `/figma-generate-diagram` skill before calling the Figma MCP. This skill is mandatory before calling `generate_diagram`.

Call `generate_diagram` with a structured description of a horizontally-laid-out timeline. Use this prompt template (substitute the bracketed values from Step 4):

```
Create a horizontal timeline / Gantt chart in FigJam.

Title: "[feature-name] — Timeline"
Subtitle: "Start [start-date] · End [end-date] · [N] sprints"

X-axis: calendar weeks from [start-date] to [end-date], labeled by week-of and sprint number.

Rows (top to bottom):
- Row 1 (header): "PHASE 1 — [name]" — a header band spanning [phase-1-start] to [phase-1-end].
  - Bar: "Epic 1.1 — [name]" spanning [start] → [end], color blue.
  - Bar: "Epic 1.2 — [name]" spanning [start] → [end], color blue.
- Row 2 (header): "PHASE 2 — [name]" — header band spanning [phase-2-start] to [phase-2-end].
  - Bar: "Epic 2.1 — [name]" spanning [start] → [end], color green.
  - ...

Add a vertical dashed line for "Today" at [today-date].
If a target launch was given, add a vertical red line labeled "Target: [target-date]".

Use FigJam shapes (sticky notes, rectangles, connectors) — this should be readable as a sticky-note board, not a precision chart.
```

After the diagram is created:
- Record the Figma timeline URL in `_pipeline-state.json` → `export_urls.figma_timeline_url`.
- Store the URL as a variable for use in Step 7 (output file) and downstream steps (Confluence embed).

If `generate_diagram` fails or the Figma MCP is not available, do not block the step. Skip the Figma output, note in the HTML output: "Figma FigJam timeline not generated — Figma MCP unavailable." Continue to Step 6.

---

## Step 6 — Generate the interactive HTML Gantt

Build a self-contained HTML file (no external dependencies, no CDN, opens offline) that the PM can **edit interactively** — drag bars to shift dates, drag the right edge to resize duration, with auto-cascade of downstream bars by default. Edits persist to localStorage and can be exported as a JSON plan that round-trips back to the skill via `/timeline apply [path]`.

### HTML structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>[feature-name] — Timeline</title>
  <style>/* see CSS section below */</style>
</head>
<body data-feature="[feature-name]" data-start-date="[YYYY-MM-DD]" data-working-days="[N]" data-px-per-day="24">
  <header>
    <h1>[feature-name] — Timeline</h1>
    <p class="meta">Generated [date] · Start <span id="hdr-start">[start]</span> · End <span id="hdr-end">[end]</span> · [N] sprints · <span id="hdr-days">[N]</span> working days</p>
    <p class="target">[Target launch: [date] · Gap: <span id="hdr-gap">+/- [N] days</span>]</p>
  </header>

  <div class="toolbar">
    <button id="btn-reset" type="button" title="Discard edits and restore the skill-computed plan">↺ Reset to original</button>
    <button id="btn-save" type="button" title="Save plan JSON directly to the skill's timeline folder (Chrome/Edge)">💾 Save to skill</button>
    <button id="btn-export" type="button" title="Download current plan as JSON for /timeline apply">⬇ Export plan</button>
    <span class="hint">Drag a bar to shift · Drag right edge to resize · Hold <kbd>Shift</kbd> to lock other bars</span>
    <span id="edit-indicator" class="edit-indicator" hidden>● Unsaved edits</span>
    <span id="save-status" class="save-status" hidden></span>
  </div>

  <div class="legend">
    <span class="legend-item phase-1">Phase 1</span>
    <!-- ... per phase ... -->
    <span class="legend-item today">▎ Today</span>
    <span class="legend-item target">▎ Target</span>
  </div>

  <div class="gantt" id="gantt">
    <div class="time-axis"><!-- one column per working day, week labels above --></div>

    <div class="row phase-row" data-id="P1" data-kind="phase" data-phase="1">
      <div class="row-label">PHASE 1 — [name]</div>
      <div class="bar-cells">
        <div class="bar phase" data-start="[N]" data-span="[N]">PHASE 1</div>
      </div>
    </div>
    <div class="row epic-row" data-id="E1" data-kind="epic" data-phase="1" data-depends-on="">
      <div class="row-label">Epic 1.1 — [name]</div>
      <div class="bar-cells">
        <div class="bar epic" data-start="[N]" data-span="[N]"
             data-hover="Start: [date] · End: [date] · FE [N]d · BE [N]d · Buffered [N]d">
          <span class="bar-label">Epic 1.1</span>
          <span class="resize-handle" aria-hidden="true"></span>
        </div>
      </div>
    </div>
    <!-- ... more epics ... -->

    <!-- IMPORTANT: Every `.bar.phase`, `.bar.epic`, `.bar.early-start` MUST be wrapped
         in a `<div class="bar-cells">` container. The container owns the per-row day grid
         (`grid-template-columns: repeat(var(--days), var(--px-per-day))`); the bar inside
         positions itself in that grid via its `data-start` / `data-span` attributes.
         Skipping the container is a known regression — bars then position against the
         row's outer 2-column grid (label + data) and visibly clamp to the right edge. -->

    <div class="today-line" data-day="[N]"></div>
    <div class="target-line" data-day="[N]"></div>
  </div>

  <footer>
    <p>Source: <code>[user-stories-file-path]</code></p>
    <p>Re-render: <code>/timeline</code> · Apply edits to skill state: <code>/timeline apply [path]</code></p>
  </footer>

  <script>/* see JS section below */</script>
</body>
</html>
```

### CSS rules (embed inline)

- **Layout via three-level CSS Grid.** The structure is `gantt → row → bar-cells → bar` — each level handles one concern.
  - `.gantt { display: grid; grid-template-columns: 320px 1fr; }` — outer 2-column grid: label gutter + data gutter.
  - `.row  { display: grid; grid-template-columns: 320px 1fr; }` — each row mirrors `.gantt`'s columns so `.row-label` sits in column 1 and `.bar-cells` sits in column 2.
  - `.row .bar-cells { grid-column: 2 / 3; display: grid; grid-template-columns: repeat(var(--days), var(--px-per-day, 24px)); position: relative; }` — **the per-row day grid lives here**, not on `.row` itself. `--days` is total working days.
  - `.row .bar { grid-column: calc(var(--start) + 1) / span var(--span); }` — bars position within `.bar-cells`'s day grid, driven by `data-start` / `data-span` attributes (read into CSS custom properties by JS on render).
- **Why the container layer is mandatory.** If `.bar` is placed as a direct child of `.row` (no `.bar-cells` wrapper), its `grid-column` resolves against `.row`'s 2-column outer grid — so any `grid-column: N` with N > 2 clamps to the right edge of the row and every epic bar visibly collapses to the right while the row labels look cut off. This is a recurring regression; the wrapper is not optional.
- **Phase header bars** span the full epic group inside their phase and use a slightly darker hue than the epics. Distinct hue per phase (use a 5-color palette cycling per phase index).
- **Bars are draggable**: `.bar { cursor: grab; }` and `.bar.dragging { cursor: grabbing; opacity: 0.7; }`. Edited bars get a subtle outline: `.bar.edited { outline: 2px solid currentColor; }`.
- **Resize handle**: the `.resize-handle` is an 8px-wide strip on the right edge of every epic bar with `cursor: col-resize` and high z-index so it captures pointer events before the bar's drag handler.
- **Ghost preview during drag**: a faintly-outlined clone of the bar follows the cursor in its candidate position. Implemented with a CSS `::after` pseudo or a sibling `.ghost` element positioned via JS.
- **Today line / target line**: absolutely positioned vertical bars using `left: calc(var(--day) * var(--px-per-day, 24px))`. Today is dashed grey; target is solid red.
- **Toolbar**: sticky to top of viewport, white background, buttons styled minimally. `.edit-indicator` hidden until an edit happens.
- **Print stylesheet** — `@media print`: hide toolbar, hide hover tooltips, hide ghost previews, force a single-page-wide layout, keep colors.

### JavaScript behavior (embed inline, no external dependencies)

The JS implements the editing model. Write it inline at the bottom of `<body>`. Keep under ~250 lines. Vanilla JS only — no library, no CDN.

#### Data model

```javascript
// Read-only baseline written by the skill (parsed once at load from data-* attributes).
const baseline = readBaselineFromDOM();
// Shape:
// {
//   bars: [
//     { id: "E1", kind: "epic", phase: 1, baseStart: 0, baseSpan: 12, dependsOn: [] },
//     { id: "E2", kind: "epic", phase: 1, baseStart: 12, baseSpan: 6, dependsOn: ["E1"] },
//     ...
//   ],
//   phaseRows: [{ id: "P1", phase: 1 }, ...],   // header rows, derived not edited
//   totalDays: [N],
//   featureName: "...",
//   startDate: "[YYYY-MM-DD]"
// }

// Editable delta layer — start at zeros, mutate as the user drags.
let edits = loadFromLocalStorage() || {}; // { [barId]: { startDelta: int, spanDelta: int } }
```

#### Effective positions

`effectiveStart(barId)` = `baseStart + (edits[barId]?.startDelta ?? 0)`
`effectiveSpan(barId)` = `baseSpan + (edits[barId]?.spanDelta ?? 0)` (minimum 1)
Phase header bars are NOT editable directly — their position recomputes from `min(effectiveStart of epics in that phase)` to `max(effectiveStart + effectiveSpan of epics in that phase)`. After every edit, recompute all phase bars and rewrite their `data-start`/`data-span` and CSS custom properties.

#### Drag-to-shift (mouse on bar body)

1. On `mousedown` on a `.bar.epic` (NOT on its `.resize-handle`): record `startX = e.clientX`, `originalDelta = edits[id]?.startDelta ?? 0`. Add `.dragging` class. Set `document.body.style.userSelect = 'none'`.
2. On `mousemove`: compute `dxPx = e.clientX - startX`, `dxDays = Math.round(dxPx / pxPerDay)`. Compute candidate `newDelta = originalDelta + dxDays`. Render the bar's ghost at the candidate position. Show a small floating label with the new start date next to the cursor.
3. On `mouseup`: commit the change: set `edits[id].startDelta = newDelta`. Then **cascade** unless Shift is held: for every other bar whose original `baseStart >= bar.baseStart + bar.baseSpan` (i.e., bars that started after this bar's original end), apply the same `dxDays` shift to their `startDelta`. (Phase headers do not get a direct delta — they recompute from their epics.) Save to localStorage. Re-render all affected bars. Update header totals (end date, working days). Show the edit indicator.
4. Bound the resulting position: a bar cannot start before day 0. Clamp negative effectiveStart to 0.

#### Drag-to-resize (mouse on `.resize-handle`)

1. On `mousedown` on `.resize-handle`: same setup as drag-to-shift, but tracking `spanDelta` instead of `startDelta`.
2. On `mousemove`: compute `dxDays` and propose `newSpan = baseSpan + originalSpanDelta + dxDays`. Minimum is 1 day. Render ghost showing the new right edge.
3. On `mouseup`: commit `edits[id].spanDelta = newSpanDelta`. Cascade unless Shift held: same logic as shift — bars that originally started after this bar's original end shift by the resize delta. Save, re-render, show edit indicator.

#### Cascade rule (precise)

When bar `X` moves or resizes by `delta` working days (positive or negative):
- If Shift is held: only `X` is updated. No downstream changes.
- Otherwise: for every bar `Y` (excluding phase headers) where `Y.baseStart >= X.baseStart + X.baseSpan`, set `edits[Y.id].startDelta += delta`. Phase headers recompute automatically from the new epic positions.

This means dependencies are inferred by **original position**, not by an explicit `dependsOn` list. A bar that originally came after `X` is treated as downstream of `X` and cascades when `X` moves. This matches the visual mental model ("everything after this point shifts") and avoids requiring the PM to maintain a dependency graph.

(If the PM wants more surgical cascade behavior, they can hold Shift on the first drag, then drag each downstream bar manually. The "Export plan" output captures the final state regardless of how it was reached.)

#### Toolbar buttons

- **`#btn-reset`**: confirm via native `confirm()` then clear `edits = {}`, clear localStorage, re-render everything from baseline. Hide the edit indicator.
- **`#btn-export`**: build a JSON object (see "Export shape" below) and trigger a download via a temporary `<a download>` element. Filename: `[feature-name]-plan-[YYYYMMDD-HHMM].json`. Do not clear edits on export — they remain in localStorage until reset.
- **`#btn-save`** (File System Access API — Chrome/Edge): writes the plan JSON directly into the feature's `timeline/` folder so the PM doesn't have to deal with a Downloads file. See "Save to skill" below.

#### Save to skill (File System Access API)

The "💾 Save to skill" button uses the modern File System Access API (`window.showSaveFilePicker`, `FileSystemFileHandle`) to write the plan JSON directly to disk — no Downloads round-trip. Browser support: Chromium-based browsers (Chrome, Edge, Brave, Opera). Safari and Firefox don't support it; the button falls back to the same behavior as Export Plan in those browsers.

##### Feature detection

```javascript
const fsaSupported = 'showSaveFilePicker' in window && 'storage' in navigator;
if (!fsaSupported) {
  // Re-label the button to "💾 Save (download)" and have it dispatch to the same handler as #btn-export.
  document.getElementById('btn-save').textContent = '💾 Save (download)';
  document.getElementById('btn-save').title = 'Browser does not support direct save — will download a JSON file instead';
}
```

##### First-click flow

1. Build the plan JSON object (same shape as Export Plan).
2. Call `window.showSaveFilePicker({ suggestedName: '[feature-name]-plan-current.json', types: [{ description: 'Timeline plan JSON', accept: { 'application/json': ['.json'] } }] })`.
3. The browser shows a native save dialog. The PM picks a target — recommend they save into the feature's `timeline/` folder (mention this in the button's title attribute).
4. Receive the `FileSystemFileHandle`. Persist it to IndexedDB so subsequent saves reuse the same target without re-prompting. See "IndexedDB handle storage" below.
5. Get a writable stream: `const writable = await handle.createWritable();`
6. Write the JSON: `await writable.write(JSON.stringify(plan, null, 2));`
7. Close the stream: `await writable.close();`
8. Show success toast in `#save-status` for 4 seconds: `✅ Saved to [filename]. Run /timeline apply in chat to refresh the sidecar.`

##### Subsequent-click flow

1. Read the stored `FileSystemFileHandle` from IndexedDB.
2. Verify the permission is still granted: `if (await handle.queryPermission({ mode: 'write' }) !== 'granted') { ... }`. If revoked or expired, call `await handle.requestPermission({ mode: 'write' })` — this requires a user gesture (which we have, the button click).
3. If permission is denied, fall back to first-click flow (prompt the picker again).
4. Otherwise: get writable stream, write, close, toast.

##### IndexedDB handle storage

Store under a database named `build-product-timeline` with an object store `handles`. Key: feature name. Value: the `FileSystemFileHandle` (handles are structured-cloneable in modern browsers, so they can be persisted in IDB directly).

Minimal helper code:

```javascript
async function idbStoreHandle(featureName, handle) {
  const db = await openIDB();
  const tx = db.transaction('handles', 'readwrite');
  await tx.objectStore('handles').put(handle, featureName);
  await tx.done;
}
async function idbReadHandle(featureName) {
  const db = await openIDB();
  return db.transaction('handles').objectStore('handles').get(featureName);
}
```

Use the IDB Promised API or hand-roll a tiny Promise wrapper around the native API — no external library.

##### Error handling

- User cancels the picker: silently no-op. Do not show an error.
- Permission denied: show `⚠ Save permission denied — using download instead` and fall back to the download flow.
- Write fails (disk full, file locked): show `⚠ Save failed: [error message]` and offer "Download instead?" via a button in the toast.

##### Why "Save to skill" doesn't update the markdown sidecar

The HTML can only write files the user has granted it access to. It writes the plan JSON, but it cannot rewrite the markdown sidecar (`[feature]-timeline.md`) because the markdown formatting logic lives in the skill, not in the JS. After Save-to-skill, the PM must still run `/timeline apply` once in chat to rebuild the sidecar from the saved JSON. The auto-discovery feature (next section) makes that a one-line command — no path argument needed.

#### localStorage

Key: ``build-product-timeline:${baseline.featureName}``. Value: `JSON.stringify(edits)`. Save on every successful `mouseup` that committed a change. Load once at page init.

#### Export shape (JSON)

```json
{
  "schema": "build-product-timeline-plan-v1",
  "feature_name": "[feature]",
  "exported_at": "ISO-8601 UTC",
  "baseline_start_date": "[YYYY-MM-DD]",
  "px_per_day": 24,
  "edits": [
    {
      "epic_id": "E1",
      "base_start_day": 0,
      "base_span_days": 12,
      "edited_start_day": 0,
      "edited_span_days": 12,
      "start_delta_days": 0,
      "span_delta_days": 0
    },
    {
      "epic_id": "E2",
      "base_start_day": 12,
      "base_span_days": 6,
      "edited_start_day": 17,
      "edited_span_days": 8,
      "start_delta_days": 5,
      "span_delta_days": 2
    }
  ],
  "phase_bands_recomputed": [
    { "phase": 1, "start_day": 0, "end_day": 25 },
    { "phase": 2, "start_day": 26, "end_day": 47 }
  ]
}
```

`base_start_day` and `base_span_days` reflect the original skill-computed plan; `edited_start_day` and `edited_span_days` reflect the final edited plan. Both are included so the skill can apply edits idempotently and compute new calendar dates from working days.

### Accessibility & ergonomics

- Every bar has a `tabindex="0"` and responds to `←`/`→` arrow keys to shift by 1 day, and `Shift+←`/`→` to resize by 1 day (when focused on the bar but not on the resize handle). Arrow-key edits also cascade (or lock with held Shift) — same rules as mouse drag.
- Hover tooltip shows the current effective dates and computed FE/BE breakdown.
- All buttons have visible focus states.
- The today/target lines have `aria-label`s.

### Working-day math

All shifting and resizing is done in **working days** (the unit the grid columns are sized in). Calendar-date computation (for display in tooltips and the JSON export's `edited_start_date`) skips weekends and any off-time ranges from `_pipeline-state.json` → `timeline.parameters.off_time`. The skill should embed those off-time ranges as a `data-off-time` attribute on `<body>` if present, so the JS can compute calendar dates correctly.

---

## Step 7 — Write outputs

Confirm the file paths to the PM before writing:

```
Writing:
  ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/timeline/[feature-name]-timeline.html
  ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/timeline/[feature-name]-timeline.md
```

The `.md` file is a markdown sidecar that contains:
- The parameter snapshot (start date, sprint length, team comp, velocity, buffer, target).
- The proposed-timeline table from Step 4 (final, post-tuning).
- The Figma FigJam URL (if generated).
- A path to the HTML Gantt: `./[feature-name]-timeline.html`.
- A traceability table mapping every Gantt bar to its source epic and contributing stories from the user-stories breakdown.

Write both files. If the parent folders do not exist, create them automatically.

Update `_pipeline-state.json`:
- `timeline.parameters` — the final parameter values.
- `timeline.computed` — start, end, sprints, working_days, gap_days (if applicable).
- `timeline.outputs.html_path` — path to the HTML file.
- `timeline.outputs.markdown_path` — path to the markdown sidecar.
- `export_urls.figma_timeline_url` — set only if Figma succeeded.

---

## Step 8 — Validation

Ask exactly one question:

"Does this timeline reflect a plan the team can commit to? (yes / needs changes / regenerate)"

- **yes** → if running inside the orchestrator, proceed to Step 11 (Export). If standalone, end the command.
- **needs changes** → ask what to change; return to Step 4 to re-tune.
- **regenerate** → ask what should be different about the inputs (sizing defaults, velocity, team comp); return to Step 2.

---

## Rules

- **Source of truth is the user-stories breakdown** (or the PRD if breakdown is unavailable). Do not invent epics or stories the breakdown does not have.
- **Granularity is Epic + Phase.** Do not render one bar per story — that produces a Gantt no one will read.
- **Phases run sequentially** unless the PRD's decision log explicitly says otherwise.
- **Within an epic, FE and BE run in parallel** (epic duration = max of FE-days and BE-days, divided by capacity).
- **Within a phase, epics run sequentially** unless the breakdown's Build Sequence Map shows independent epics.
- **Math must be honest.** Never fit the timeline to a target by shrinking durations behind the scenes. If the gap is negative, surface it and let the PM decide between scope cut, team increase, or slip.
- **Hybrid estimation.** The skill proposes; the PM tunes; the skill recomputes. Do not lock in the math before the PM has seen the proposal.
- **Hover details on every bar.** The HTML output must expose start, end, FE-days, BE-days, and buffered total on hover.
- **No external CDN.** The HTML must open offline. No `<script src="...">` to a remote host.
- **Self-check (Error Type 3 in error-handling.md):** does this timeline contradict any decision recorded in the PRD's decision log (e.g., a committed phase order)? If yes, flag to the PM before writing.
- **Update `_pipeline-state.json`** at the end of this step regardless of whether Figma succeeded.

---

## Apply Mode — round-trip edits from the HTML back into the skill

Triggered when the PM runs `/timeline apply [path]` (or pastes a `build-product-timeline-plan-v1` JSON, or says "apply my edits"). This mode promotes interactive HTML edits into the official skill state and re-renders the markdown sidecar.

### A-1 — Locate and read the JSON

Three input paths:

**(a) Explicit path:** PM gave a file path like `/timeline apply ~/Downloads/feature-plan-20260521-1342.json`. Read that file directly.

**(b) Inline paste:** PM pasted JSON content in the message. Parse it directly.

**(c) No argument — auto-discovery:** PM just typed `/timeline apply` (with no path and no pasted JSON). Scan for plan files in this order:

1. **Feature's own `timeline/` folder** (where "💾 Save to skill" writes by default): `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/timeline/*-plan-*.json` — sorted by mtime descending.
2. **Downloads folder** (where "⬇ Export plan" writes): `~/Downloads/[feature-name]-plan-*.json` — sorted by mtime descending. Filter by feature-name prefix to avoid picking up plans for other features.

For each candidate, briefly read it to verify it parses as a `build-product-timeline-plan-v1` JSON and that its `feature_name` matches the current context.

Presentation rules:

- **Zero candidates found:** ask: "I couldn't find a plan JSON in `[feature]/timeline/` or `~/Downloads/`. Paste the JSON content, or give me an explicit path."
- **Exactly one candidate found:** show the path and modified time, ask: "Apply `[path]` (modified [time ago])? (yes / no)". If yes, proceed with that file.
- **Multiple candidates found:** list them with mtimes (newest first), most recent pre-selected:
  ```
  Multiple plan files found:
    1. ~/Desktop/Resources/PDLC Workflow Docs/[feature]/timeline/[feature]-plan-current.json   (2 minutes ago) ← latest
    2. ~/Downloads/[feature]-plan-20260521-1342.json                                            (1 hour ago)
    3. ~/Downloads/[feature]-plan-20260521-1015.json                                            (yesterday)

  Apply #1 (default) or pick another?
  ```

Once a JSON is selected (via any of the three paths), validate it has `"schema": "build-product-timeline-plan-v1"`. If it doesn't, refuse and explain — the file isn't an export from this skill's Gantt. Do not attempt to interpret arbitrary JSON.

Validate that `feature_name` matches the current feature (when running inside the orchestrator, use the context feature; standalone, ask if it doesn't match: "This plan is for `[plan-feature]` but the active feature is `[ctx-feature]`. Apply to `[plan-feature]` instead?").

### A-2 — Compute new calendar dates from the edits

For each entry in `edits[]`:
- Read `edited_start_day` and `edited_span_days` (post-edit values).
- Convert `edited_start_day` (working-day offset) to a calendar date by adding `edited_start_day` working days to `baseline_start_date`, skipping weekends and any off-time ranges from `_pipeline-state.json` → `timeline.parameters.off_time`.
- Convert `edited_span_days` to an end date the same way (add `edited_span_days - 1` working days to the start date).
- Result per epic: `{ epic_id, new_start_date, new_end_date, new_duration_working_days }`.

For each phase band in `phase_bands_recomputed`:
- Same conversion: working-day offset → calendar date.
- Result per phase: `{ phase, new_start_date, new_end_date }`.

Compute the new overall feature end (latest epic end). Compute the new gap vs. target launch (if PM set one): `new_gap = target_launch - new_feature_end`. Flag prominently if the gap is now negative (timeline exceeds target).

### A-3 — Show the diff before writing

Present a compact diff:

```
━━━ Apply plan — [Feature Name] ━━━

Edited 4 of 7 epics. Total feature end moved from 2026-08-12 to 2026-08-19 (+5 working days).
[Gap vs target 2026-08-15: was +2 days, now -2 days — timeline exceeds target.]

Per-epic changes:
  Epic 1.1 (Listing FE)        unchanged
  Epic 1.2 (Listing BE)        2026-06-10 → 2026-06-15  (+5 working days, span unchanged)
  Epic 1.3 (Management FE)     2026-06-20 → 2026-06-25  (+5 working days, span +2 days)
  Epic 1.4 (Management BE)     2026-06-20 → 2026-06-25  (+5 working days, span unchanged)
  Epic 2.1 (Notifications)     2026-07-05 → 2026-07-10  (+5 working days, span unchanged)
  ...

Phase bands:
  Phase 1                      ends 2026-07-02 (+5 days)
  Phase 2                      starts 2026-07-05, ends 2026-08-19 (+5 days)

Apply this plan? (yes / no / show full per-epic detail)
```

If the PM says no, stop. Do not modify state or files.

### A-4 — Update `_pipeline-state.json`

On approval:
- Update `timeline.computed` with the new `start_date`, `end_date`, `working_days`, `sprints`, `gap_days` (if target).
- Update `timeline.parameters` epic durations to match `edited_span_days` per epic.
- Add an entry to `timeline.applied_edits[]`:
  ```json
  {
    "applied_at": "ISO-8601",
    "source_file": "[path or 'inline-paste']",
    "epics_changed": [N],
    "feature_end_shift_days": [signed int],
    "new_gap_days": [signed int or null]
  }
  ```

### A-5 — Re-render the markdown sidecar

Rewrite `[feature]/timeline/[feature]-timeline.md` to reflect the new dates. Keep the same structure (parameter snapshot, proposed-timeline table, Figma URL, HTML path, traceability) but with **post-edit** values throughout. Preserve any unchanged sections verbatim.

Note at the top of the file:

```markdown
> Plan applied [YYYY-MM-DD HH:MM] from `[path]`.
> Feature end shifted [+/- N] working days from baseline. See `_pipeline-state.json` → `timeline.applied_edits` for history.
```

### A-6 — Re-render the HTML

Rewrite `[feature]/timeline/[feature]-timeline.html` so the **edited positions become the new baseline**. After re-render:
- `data-start` and `data-span` on each bar reflect the edited values (not the original baseline).
- localStorage edits should be cleared (the JS reads `data-start` / `data-span` as the new ground truth).
- The HTML is functionally identical to a fresh `/timeline` run, but with the new dates.

This means after `/timeline apply`, the PM can open the HTML again and it shows the applied plan as the new baseline — they can iterate further from there.

### A-7 — Confluence + transcript awareness

If Confluence has been published for this feature (`confluence_hub.parent_page_id` exists), inform the PM:

```
The Step 10½: Timeline child page in Confluence still reflects the prior plan.
Run /publish-to-confluence to update it with the applied edits. (per-file mtime
detection means only the timeline page will re-publish.)
```

Do not auto-republish — PM should decide when to share the change.

### A-8 — Report

```
✅ Plan applied — [Feature Name]

  Feature start:  [date]
  Feature end:    [date]   (was [old], shift: [+/- N] working days)
  Working days:   [N]      (was [old])
  Target:         [date or "none set"]
  Gap:            [+/- N days] [⚠ if negative]

State updated:
  _pipeline-state.json → timeline.computed
  _pipeline-state.json → timeline.parameters (per-epic durations)
  _pipeline-state.json → timeline.applied_edits[] (this run logged)

Files re-rendered:
  timeline/[feature]-timeline.md   (sidecar, with applied-from header)
  timeline/[feature]-timeline.html (new baseline = applied edits)

Next step (optional):
  /publish-to-confluence  — refresh the Timeline child page on the hub
```

### Apply-mode rules

- **Never modify the user-stories breakdown** as part of apply mode. The breakdown is the source of truth for what gets built; the timeline only governs when. Scope changes still require `/change-mode`.
- **Never silently overwrite the markdown sidecar without showing the diff.** Always Step A-3 → Step A-4.
- **Idempotent re-apply.** If the PM applies the same JSON twice, the second run should be a no-op (state and files already match). Detect this and tell the PM: "Plan is already applied — nothing to update."
- **Schema gate.** Refuse any JSON that doesn't have `"schema": "build-product-timeline-plan-v1"`. Future schema versions will require a migration step here; today there is only v1.
