# Pipeline Timing Report

Generate a timing report for a feature's pipeline run with both **wall-clock time** (total elapsed including gate waits) and **active-work time** (model working, gate waits excluded).

Use when:
- The pipeline just finished and you want to see how long the whole thing actually took.
- You're estimating effort for the next feature and want a baseline from prior runs.
- You want to surface how much of pipeline time is the model working vs. you reviewing.

Read and follow `ai-framework/09-pipeline-timing.md`.

## Inputs

The underlying prompt collects inputs as Step 0:

- **Required:** feature name (asked if not in conversation context).
- **Auto-detected:** the pipeline state file at `[feature]/_pipeline-state.json` and the current Claude Code session JSONL.

No prior pipeline run is strictly required, but the report is most accurate when one exists.

## Output

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/timing/[feature-name]-timing.md
```

The report includes:
- **Summary table** — wall-clock total, active-work total, total gate-wait time, percent of time you spent reviewing.
- **Per-step breakdown** — every pipeline step with its start time, duration, and notes (gate steps call out how much of the duration was gate wait).
- **Per-gate table** — each of Gate 1 / 2 / 3 with the time it was presented to you, the time you approved, and how long you took.
- **Data sources footer** — how many entries came from instrumented `step_timings` (precise) vs. JSONL fallback (approximate, marked with `~`).

The report's totals are also written back to `_pipeline-state.json` → `timing_report` so the transcript export and the Confluence hub page can include the numbers without re-running the report.

## How it computes the numbers

Two data sources, in order of preference:

1. **Instrumented timestamps in `_pipeline-state.json` → `step_timings`** — the orchestrator writes `started_at` when a step begins and `completed_at` when it finishes (plus `presented_at` / `approved_at` for gates). Precise.
2. **JSONL fallback** parsed from the live Claude Code session log — uses message timestamps and a 60-second gap-detection heuristic to infer step boundaries. Approximate.

Both sources can be combined: instrumented for steps that have it, JSONL-inferred for any step missing instrumentation. Inferred rows are marked with `~`.

## When to run

- **Automatically at the end of `/build-product`** — included in the Step 11 end-of-export summary and embedded at the top of the Step 12 transcript output.
- **Standalone via `/pipeline-timing`** — re-generate any time, including for completed pipelines from prior sessions.

## What "active-work time" means

The math is:

```
wall_clock     = pipeline_completed_at - pipeline_started_at
active_work    = wall_clock - sum(gate_wait_time for all gates)
gate_wait_time = approved_at - presented_at   (per gate)
```

Active-work time captures the model's working time, excluding the periods where the pipeline was paused waiting for your approval at a gate. It does not separate model thinking time from tool execution time — those both count as active work.
