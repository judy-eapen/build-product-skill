# Pipeline Doctor

A diagnostic command that scans the skill's own files **and** any active feature workspace for inconsistencies — the kind of drift that causes silent stalls, missing slash commands, broken state files, or stale features stuck mid-pipeline.

Use when:
- The pipeline mysteriously stops or skips a step.
- A slash command doesn't show up in Claude Code's menu after install/update.
- `_pipeline-state.json` looks weird or doesn't match what's on disk.
- You're returning to a feature after weeks and want a "what's the state?" report.
- You've just modified the skill (added a step, renamed a command, changed a schema) and want to verify nothing is dangling.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Scope selection

Ask the PM:

```
What should I check?

1. Everything — skill self-consistency + every feature in the workspace
2. Just the skill — no per-feature checks
3. One specific feature — give me the feature name
4. Stale features only — flag pipelines that haven't moved in 30+ days

(Default: 1)
```

Apply the standard feature-name derivation rule (spaces → hyphens, lowercase) if the PM gives a feature name freely.

---

## Step 1 — Run checks

Each check produces one of three severities:

- **CRITICAL** — pipeline cannot run correctly until fixed. Examples: a step in `pipeline-configs.yaml` references an instruction file that doesn't exist; a slash command file points at a renamed/removed subprompt.
- **WARNING** — pipeline may stall or behave unexpectedly. Examples: a step in `pipeline-configs.yaml` has no prose in `subprompts/build-product.md` (the bug that caused the Step 10.5 stall); state has DRAFT stories but the breakdown file doesn't mark them.
- **INFO** — informational, not actionable. Examples: a feature hasn't been touched in 14 days; the Drive sync was disabled at pre-flight so Drive folder is null.

Run checks in this order. Skip categories the scope doesn't include.

### Category A — Skill self-consistency

Read `ai-framework/pipeline-configs.yaml`, `SKILL.md`, and `subprompts/build-product.md` for these checks.

| ID | Check | Severity if failed |
|---|---|---|
| **A1** | Every step in `pipelines.work.steps[]` has an `instruction:` field that resolves to a real file (resolve relative paths from the skill root). | CRITICAL |
| **A2** | Every quality_check ID referenced in any `gate.quality_checks[]` is defined in the top-level `quality_checks:` section. | CRITICAL |
| **A3** | Every step ID in `pipelines.work.steps[]` has a matching `### Step [N]` (or `Step [N.M]`) heading in `subprompts/build-product.md`. *This is the check that catches the bug Judy hit: Step 10.5 was in the config but missing from the orchestrator subprompt.* | WARNING |
| **A4** | Every step ID has a matching `### Step [N]` heading in `SKILL.md` Work Pipeline section. | WARNING |
| **A5** | Every step block in `subprompts/build-product.md` ends with an explicit "Next:" handoff (or is itself the final step, or is a gate that pauses by design). Scan for `### Step` headings and the prose between each pair; flag any block that has no "Next:" / "proceed to" / "continue automatically" language. | WARNING |
| **A6** | For each gate, the gate's `name` and `after_step` reference a valid step. | CRITICAL |
| **A7** | All conditional steps (with a `condition:` field) reference a state path that the relevant gate writes. (E.g., `gate_1.answer.system_design == true` requires the Gate 1 prompt to ask the system-design question and write the answer.) | INFO |

### Category B — Feature-state consistency (per feature in scope)

For each feature folder under `~/Desktop/Resources/PDLC Workflow Docs/`, read its `_pipeline-state.json` and run:

| ID | Check | Severity if failed |
|---|---|---|
| **B1** | `_pipeline-state.json` parses as valid JSON. | CRITICAL |
| **B2** | Required keys present: `feature_name`, `mode`, `current_step`, `gates`, `artifacts`. (Either legacy `pipeline: "Work"` or modern `pipeline_started_at` is acceptable.) | CRITICAL |
| **B3** | For each artifact entry in `artifacts.[key]` with non-null `path`: the file exists on disk. Flag missing files per artifact. | WARNING |
| **B4** | If `current_step` is at or after a step that comes after Gate N, then `gates.gate_N` must be "Approved [date]". Mismatches (e.g., current_step says Step 10 but Gate 2 is "Pending") indicate state corruption. | WARNING |
| **B5** | If `user_stories.draft_stories[]` is non-empty, read the user-stories breakdown file and confirm each `us_id` is present and marked with `Status: ⚠ DRAFT — needs design`. | WARNING |
| **B6** | If `user_stories.epics[]` has entries with `jira_key` set, verify those Epic keys are referenced in the Jira manifest at `jira-export/[feature]-jira-manifest.md` (if the manifest exists). | INFO |
| **B7** | If `confluence_hub.parent_page_id` is set, `confluence_hub.artifacts` must be populated (non-empty dict). | WARNING |
| **B8** | For each `confluence_hub.artifacts.[key]` with a `source_mtime` set: the mtime should be ≤ "now" and ≥ `pipeline_started_at` (if available). Future mtimes or pre-pipeline mtimes indicate clock drift or migration issues. | INFO |
| **B9** | If `timeline.applied_edits[]` has entries, `timeline.computed.end_date` must be set and consistent with the most recent applied edit. (Verify by re-reading the markdown sidecar's end-date if available.) | INFO |
| **B10** | `pipeline_completed_at` is set only if the last step's `step_timings[step].completed_at` is also set. Mismatches indicate a partial-completion state. | INFO |
| **B11** | The Jira manifest, if present, references the same `feature_name` as state. (Cross-feature pollution check.) | WARNING |

For each finding, include the feature name in the report so the PM knows which workspace is affected.

### Category C — Slash command coverage

| ID | Check | Severity if failed |
|---|---|---|
| **C1** | For every `subprompts/*.md` file (excluding `build-product.md` itself, which is registered as the orchestrator entry point): there should be a matching `~/.claude/commands/[name].md` registration. List any missing. | WARNING |
| **C2** | For every `~/.claude/commands/*.md` file whose content references `~/.claude/skills/build-product/subprompts/[name].md`: that target file must exist. List any broken pointers. | CRITICAL |
| **C3** | The orchestrator subprompt (`subprompts/build-product.md`) is registered (matching file in `~/.claude/commands/`). | CRITICAL |

### Category D — Stale features

Scan every feature folder for staleness:

| ID | Check | Severity if failed |
|---|---|---|
| **D1** | `pipeline_started_at` > 30 days ago AND `pipeline_completed_at` is null. List with the days-since-start count. | INFO |
| **D2** | `_pipeline-state.json` mtime hasn't changed in 14+ days AND `pipeline_completed_at` is null. List with the days-since-update count. | INFO |
| **D3** | Feature folder exists but no `_pipeline-state.json` is present. Indicates an orphaned partial run. | WARNING |

---

## Step 2 — Compose the report

Build two outputs: an **inline chat summary** (concise, scannable) and a **full markdown report** written to disk.

### Inline chat summary

```
━━━ Pipeline Doctor — [YYYY-MM-DD] ━━━

Scope: [Everything | Skill only | [feature-name] | Stale features only]

CATEGORY A — Skill self-consistency
  ✓ A1: All step instruction files exist
  ✓ A2: All quality_checks defined
  ⚠ A3: Step 10.5 in pipeline-configs but missing in subprompts/build-product.md
  ✓ A4: All steps documented in SKILL.md
  ⚠ A5: Step 7 has no explicit "Next:" handoff
  ✓ A6: Gate references valid
  ✓ A7: Conditional state paths valid

CATEGORY B — Feature state ([feature-name])
  ✓ B1–B2: state schema valid
  ⚠ B3: 1 artifact path missing on disk: codebase-review/[feature]-codebase-review.md
  ✓ B4: gate states coherent with current_step
  ✓ B5–B11: no findings

CATEGORY C — Slash command coverage
  ✓ C1: All 18 subprompts have registered slash commands
  ✓ C2: No broken command pointers
  ✓ C3: /build-product registered

CATEGORY D — Stale features
  ℹ D1: 'old-experiment' started 47 days ago, never completed
  ℹ D2: 'paused-feature' state file unchanged for 21 days

Summary: 2 CRITICAL · 3 WARNING · 2 INFO · 35 passed

Full report: ~/Desktop/Resources/PDLC Workflow Docs/_pipeline-doctor-report-[YYYY-MM-DD-HHMM].md
```

### Full markdown report

Path: `~/Desktop/Resources/PDLC Workflow Docs/_pipeline-doctor-report-[YYYY-MM-DD-HHMM].md`

Structure:

```markdown
# Pipeline Doctor Report — [YYYY-MM-DD HH:MM]

**Scope:** [...]
**Skill version:** [from VERSION.md]
**Files scanned:** [N]

## Summary

| Severity | Count |
|---|---|
| CRITICAL | [N] |
| WARNING | [N] |
| INFO | [N] |
| Passed | [N] |

[If any CRITICAL: lead with a "Fix these first" block. Otherwise skip.]

## Findings

### CRITICAL

#### [Check ID] — [One-line title]
- **Where:** [file path or feature name]
- **What:** [precise description]
- **Fix:** [concrete suggested action — file path + change to make]

### WARNING

[Same structure]

### INFO

[Same structure, may be terse]

## Passed checks ([N])

[List in compact form: A1, A2, A4, B1, B2, ... — for transparency, not detail]

## How to act on this report

- **CRITICAL items** must be resolved before running `/build-product` again. They will cause the pipeline to fail or stall.
- **WARNING items** should be reviewed. They may not break the next run but indicate drift between intent and reality.
- **INFO items** are diagnostic — read for context, act if relevant.
- For drift between skill files (Category A), see `subprompts/build-product.md` and `SKILL.md` to add missing prose.
- For feature-state inconsistencies (Category B), `/change-mode` is usually the right way to bring state and files back in sync. For unrecoverable corruption, the safest move is to restart the feature from the relevant gate via `/reopen-gate-N`.
- For stale features (Category D), decide: archive (move to a `_archive/` subfolder), complete (run `/build-product` to advance), or abandon.
```

---

## Step 3 — Present findings

Print the inline summary to chat first. Then ask:

```
Want me to walk through the [N] CRITICAL/WARNING findings and propose fixes one at a time?
(yes / no / just the CRITICAL ones)
```

If yes: for each finding (CRITICAL first, then WARNING), show:
- The full finding text
- A concrete proposed fix (e.g., "Add this section to `subprompts/build-product.md`: [text]")
- Ask: "Apply this fix? (yes / skip / show me the file context first)"

Apply approved fixes one at a time. After each batch, re-run the affected checks to verify the fix took.

Do not apply fixes silently — always confirm per-finding.

---

## Step 4 — Write the report file

Always write the full markdown report to disk, even if the PM declined the fix walkthrough. The report is the historical record of what was checked and when. Keep prior reports in place (don't overwrite — use the timestamped filename).

Update `_pipeline-doctor-history.md` at the workspace root (`~/Desktop/Resources/PDLC Workflow Docs/_pipeline-doctor-history.md`) — append a one-line entry per run:

```markdown
- 2026-05-22 14:30 — Everything scope, 2 CRITICAL, 3 WARNING, 2 INFO, 35 passed — [report path]
```

This gives the PM a running log of when checks were run and how the skill has trended over time.

---

## Rules

- **Read-only by default.** The doctor scans and reports. It does not modify files until the PM approves a specific fix at Step 3.
- **Skill-relative paths.** Resolve all `pipeline-configs.yaml` instruction paths relative to the skill root (`~/.claude/skills/build-product/`), not to `cwd`.
- **No external network calls.** Doctor only reads local files. It does not query Jira, Confluence, Figma, or any MCP — those are out of scope. (A future `/pipeline-doctor --remote` could check Confluence URLs and Jira tickets, but not in v1.)
- **Idempotent.** Running the doctor twice in a row should produce identical findings. No side effects between runs except the timestamped report file.
- **Concise per-finding.** Each finding fits a single screen. PMs reading this are usually triaging — long explanations slow them down.
- **Self-aware.** If the doctor itself has a bug, its own state is what's broken. Surface this gracefully: if a check fails to execute (e.g., a file can't be read), report `EXEC ERROR` not a fake CRITICAL.
