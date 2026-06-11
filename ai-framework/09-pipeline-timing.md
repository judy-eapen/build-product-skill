# Pipeline Timing Report

Generates a timing report for a feature's pipeline run — both **wall-clock time** (total elapsed including lunch breaks and overnight gate waits) and **active-work time** (excluding time the model was waiting on the PM).

Two data sources, in order of preference:

1. **Instrumented timestamps** in `_pipeline-state.json` → `step_timings` (written by the orchestrator at each step's start and end). Precise.
2. **JSONL fallback** parsed from `~/.claude/projects/[your-profile]/[session-uuid].jsonl`. Approximate — uses message timestamps and the gap-detection heuristic below.

Both sources can be combined for a single run: instrumented data for steps that have it, JSONL fallback for any step missing instrumentation. The report notes which source produced each row.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Input Check

**If running inside the orchestrator (called from `/build-product` at the end, or from `/export-transcript`):** feature name is in context. Proceed.

**If running standalone (called via `/pipeline-timing`):** ask the PM:

> "Which feature should I generate a timing report for?
> 1. **The most recently active feature** — derived from the most recently modified `_pipeline-state.json` under `~/Desktop/Resources/PDLC Workflow Docs/`.
> 2. **A specific feature** — give me the feature name.
>
> (Default: 1)"

---

## Step 1 — Read `_pipeline-state.json`

Read `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_pipeline-state.json` and capture:

- `pipeline_started_at` — pipeline kickoff timestamp.
- `pipeline_completed_at` — last step finished (or null if still running).
- `step_timings` — dict keyed by step id. Each entry can contain:
  - `started_at`, `completed_at`, `duration_seconds` (auto step).
  - For gate steps: also `presented_at`, `approved_at`, `gate_wait_seconds`.
- Mode (`fast` or `gated`) and pipeline type (`work`).

If `step_timings` is empty or missing, jump to Step 3 (JSONL fallback). Otherwise, continue to Step 2.

---

## Step 2 — Compute totals from instrumented data

For each step that has both `started_at` and `completed_at`:

```
duration_seconds = completed_at - started_at
```

For each gate, additionally:

```
gate_wait_seconds = approved_at - presented_at
```

**Wall-clock total** = `pipeline_completed_at - pipeline_started_at` (or `now - pipeline_started_at` if still running).

**Active-work total** = `wall_clock_total - sum(gate_wait_seconds for all gates)`.

If a step has no `completed_at` (still running or instrumentation interrupted), mark its duration as "in progress (started [time])" and exclude from totals.

---

## Step 3 — JSONL fallback for un-instrumented steps

For any step missing from `step_timings`, or if `step_timings` is entirely empty:

1. Locate the session JSONL file (`ls -t ~/.claude/projects/*/*.jsonl | head -1`). The subfolder under `~/.claude/projects/` is derived from your home directory path (slashes replaced with hyphens); run `ls ~/.claude/projects/` to see yours.
2. Parse it. Each `user` and `assistant` line has a `timestamp` (ISO-8601).
3. Filter to the window `[pipeline_started_at, pipeline_completed_at]` (or "now" if still running).
4. For each step, infer boundaries:
   - **Step transitions** are marked by orchestrator output mentioning the next step's name (e.g., "Step 3 — Create PRD") or the previous step's banner (e.g., "✓ Step 2 — Codebase Review complete"). Scan assistant messages for these banners and use their timestamps as step boundaries.
   - **Gate waits** are detected as long gaps between an assistant message containing the gate-presentation banner (e.g., `━━━ APPROVAL NEEDED: Gate 1`) and the next user message. Default threshold: any gap > 60 seconds is treated as a wait.
   - **Generic idle time** is detected as gaps between an assistant message ending in a question and the next user message. Same 60-second threshold.

5. Compute durations from inferred timestamps.

Note: JSONL-derived timings are approximate. The report should explicitly mark these rows with `~` (e.g., `~12m 30s — inferred from session log`).

---

## Step 4 — Compose the report

Write to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/timing/[feature-name]-timing.md`:

```markdown
# Pipeline Timing Report — [Feature Name]

**Pipeline started:** [pipeline_started_at — formatted as "2026-05-21 09:00 PDT"]
**Pipeline completed:** [pipeline_completed_at, or "still running"]
**Mode:** [Fast / Gated]
**Generated:** [now]

## Summary

| Metric | Time |
|---|---|
| **Wall-clock total** (elapsed from start to finish, includes all waits) | **4h 23m 12s** |
| **Active-work total** (excludes time waiting on you at gates) | **1h 12m 04s** |
| **Total gate-wait time** (you reviewing artifacts before approving) | **3h 11m 08s** |

You spent [X%] of pipeline time reviewing artifacts — the rest was the model working.

## Per-step breakdown

| # | Step | Started | Duration | Notes |
|---|---|---|---|---|
| 1 | Research Idea          | 09:00 | 12m 04s |  |
| 2 | Codebase Review        | 09:12 | 04m 31s |  |
| 3 | Create PRD             | 09:17 | 18m 22s |  |
| 4 | Dual Review (parallel) | 09:35 | 06m 18s | parallel block |
| 5 | Apply Fixes → Gate 1   | 09:41 | **1h 28m** | includes 1h 22m gate wait |
| 6 | System Design          | 11:09 | 11m 02s |  |
| 7 | Visual Diagram         | 11:20 | 03m 41s |  |
| 8 | Design Prompts         | 11:24 | 09m 17s |  |
| 9 | Update PRD → Gate 2    | 11:33 | **42m 11s** | includes 38m gate wait |
| 10 | User Stories → Gate 3 | 12:15 | **1h 10m** | includes 1h 04m gate wait |
| 10.5 | Timeline             | 13:25 | 05m 18s |  |
| 11 | Export (parallel)     | 13:30 | 08m 47s | Jira + Drive + Confluence |
| 12 | Export Transcript     | 13:39 | 00m 24s |  |
| ~ | Total                  | 09:00 → 13:23 | **4h 23m 12s** | wall-clock |

## Gates — how long each took you to approve

| Gate | Presented | Approved | You took |
|---|---|---|---|
| Gate 1: PRD                        | 09:47 | 11:09 | **1h 22m** |
| Gate 2: Designs                    | 11:38 | 12:16 | **38m**    |
| Gate 3: User Stories Breakdown     | 13:15 | 14:19 | **1h 04m** |

## Data sources

- Instrumented step_timings entries used: [N]
- JSONL-inferred entries used: [N] (marked with ~ in the table)
- Session JSONL: `[path]`
```

If a step's data source is JSONL fallback, prefix its row's Duration with `~` and add a note column entry. The "Data sources" footer should clearly state how many entries came from each.

---

## Step 5 — Update state

Write the computed totals back to `_pipeline-state.json` → `timing_report`:

```json
"timing_report": {
  "last_generated_at": "ISO-8601",
  "wall_clock_seconds": 15792,
  "active_work_seconds": 4324,
  "total_gate_wait_seconds": 11468,
  "report_path": ".../timing/[feature]-timing.md",
  "instrumented_entries": 11,
  "jsonl_inferred_entries": 2
}
```

This lets downstream consumers (Confluence hub publish, transcript export) include the totals without re-running the report.

---

## Step 6 — End-of-run output

Print a compact summary:

```
━━━ Pipeline timing — [Feature Name] ━━━

Wall-clock: 4h 23m 12s   (from 2026-05-21 09:00 to 13:23)
Active work: 1h 12m 04s   (model working, excludes gate waits)
Gate waits:  3h 11m 08s   (you reviewing — Gate 1: 1h22m, Gate 2: 38m, Gate 3: 1h04m)

Full report: [path]
━━━
```

If called as part of `/export-transcript` or the end-of-pipeline summary, the caller may include just the table portion (without the banner) inline.

---

## Rules

- **Wall-clock and active-work are computed from the same data**, never from separate sources. Whichever data source produces wall-clock must also produce gate-wait to keep the math consistent.
- **Mark inferred entries with `~`** in the report. Don't pretend JSONL-derived timings are precise.
- **Never modify `pipeline_started_at`** — it's set once at pipeline start and is authoritative.
- **If `pipeline_completed_at` is null** (still running), use `now` as the upper bound and label the report `(pipeline in progress)`.
- **Update `_pipeline-state.json`** at the end so other steps can read the totals without re-running.
- **Self-check:** if active-work + total-gate-wait significantly diverges from wall-clock (>5% off), surface the discrepancy at the bottom of the report with a note. The math should be tight; large divergence usually means an instrumented step was missing its `completed_at`.
