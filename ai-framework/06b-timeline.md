# Timeline (Gantt)

Runs after Gate 3 (User Stories Breakdown approval) and before Step 11 (Export) when called from the orchestrator, OR can be called standalone via `/timeline` when a PM has a PRD and an approved user-stories breakdown and wants a Gantt-style timeline without re-running the full pipeline.

Produces a high-level timeline at the **Epic + Phase** level (not individual stories). Two outputs:

- **Figma FigJam timeline** — created via the Figma MCP. Lives in Figma, can be shared with stakeholders, and embeds in Confluence.
- **Interactive HTML Gantt** — self-contained HTML file (vanilla HTML/CSS/JS, no external dependencies) the PM can open locally or share as a file. Includes hover details, today-marker, and a printable view.

Estimation is **hybrid**: the skill proposes durations from story sizing + a default velocity, then the PM tunes per-epic before the final Gantt is rendered.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

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

Build a self-contained HTML file (no external dependencies, no CDN, opens offline) with the following structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>[feature-name] — Timeline</title>
  <style>
    /* Embedded CSS — see template below */
  </style>
</head>
<body>
  <header>
    <h1>[feature-name] — Timeline</h1>
    <p class="meta">Generated [date] · Start [start] · End [end] · [N] sprints · [N] working days</p>
    <p class="target">[Target launch: [date] · Gap: +/- [N] days]</p>  <!-- only if target given -->
  </header>

  <div class="legend">
    <span class="legend-item phase-1">Phase 1</span>
    <span class="legend-item phase-2">Phase 2</span>
    <!-- ... per phase ... -->
    <span class="legend-item today">▎ Today</span>
    <span class="legend-item target">▎ Target</span>
  </div>

  <div class="gantt">
    <div class="time-axis">
      <!-- One column per week, labeled "W1 / Apr 1" etc. -->
    </div>

    <div class="row phase-row">
      <div class="row-label">PHASE 1 — [name]</div>
      <div class="bar phase" style="--start: [N]; --span: [N]; --color: phase-1;">PHASE 1</div>
    </div>
    <div class="row epic-row">
      <div class="row-label">Epic 1.1 — [name]</div>
      <div class="bar epic" style="--start: [N]; --span: [N]; --color: phase-1;"
           data-hover="Start: [date] · End: [date] · FE [N]d · BE [N]d · Buffered [N]d">
        Epic 1.1
      </div>
    </div>
    <!-- ... -->

    <div class="today-line" style="--day: [N];"></div>
    <div class="target-line" style="--day: [N];"></div>  <!-- only if target given -->
  </div>

  <footer>
    <p>Source: <code>[user-stories-file-path]</code></p>
    <p>Re-render this timeline: <code>/timeline</code> from this feature folder.</p>
  </footer>
</body>
</html>
```

**CSS template** — embed inline. Use CSS Grid with one column per working day, `--start` and `--span` as CSS custom properties driving `grid-column`. Use distinct hues per phase. Hover state on each bar shows the `data-hover` content as a tooltip. Today and Target lines are absolutely positioned vertical bars.

**Print stylesheet** — `@media print`: hide hover tooltips, force a single-page-wide layout, keep colors.

**Today logic** — compute the column index for today's date at render time using the start date and a working-days-elapsed calculation. Embed the index as a CSS custom property on the `.today-line` element so it does not need JavaScript.

**Minimal JS** — none required unless adding bar-click-to-expand. If included, keep it inline and under 30 lines. Default: no JS.

The bars must be sized in **working days** to keep the math consistent with Step 3. Weekends are simply not rendered as columns.

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
