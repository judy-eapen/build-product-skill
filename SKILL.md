---
name: build-product
description: Work pipeline for PM product workflows. Research → Codebase Review → PRD → dual review → designs → user stories → Jira export. Planning and design only, no implementation.
---

# Build Product (Work Pipeline Orchestrator)

---

## SETUP CHECK — runs once at session start

Before any pipeline routing, run this check. It must run exactly once per session.

1. Verify the following folders exist relative to this skill root:
   - `ai-framework/`
   - `subprompts/`

2. Verify the following critical files exist:
   - `ai-framework/rules.md`
   - `ai-framework/personas.md`
   - `ai-framework/error-handling.md`
   - `ai-framework/02-create-prd.md`
   - `subprompts/cto-review.md`
   - `subprompts/review-prd.md`

3. If any folders or files are missing, print:

   ```
   Skill is incomplete. Missing: [list of missing files].
   The skill may not have installed correctly.
   Please reinstall or contact the skill owner.
   ```

   Then stop. Do not attempt to run the pipeline.

4. If everything is present, proceed normally and never mention the setup check to the PM unless something is wrong.

---

Work pipeline for PM product workflows with parallelized review steps. Research → Codebase Review → PRD → dual review → designs → user stories → Jira export.

**Pipeline routing is config-driven.** After the PM selects a pipeline type, read `ai-framework/pipeline-configs.yaml` to determine the exact steps, gates, quality checks, and conditions for that pipeline. Do not hardcode step logic — derive it from config. If a pipeline type the PM describes is not listed in the config file, say so and offer the closest match.

Read `ai-framework/05-parallel-rules.md` before executing any parallel block.

**Global rules inheritance.** Every step in every pipeline must read both `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## CONTEXT BUDGET RULES

Long pipelines consume significant context. Follow these rules to prevent mid-pipeline context exhaustion.

### Lazy loading — load only what you need

At any given step, load ONLY:
1. `ai-framework/rules.md` (always)
2. `ai-framework/error-handling.md` (always)
3. The current step's instruction file (e.g. `ai-framework/02-create-prd.md`)

Do NOT pre-load all framework files at the start of the session. Load other files on demand:
- `ai-framework/05-parallel-rules.md` — load only when about to execute a parallel block
- `ai-framework/personas.md` — load only when composing agent prompts for a review step
- `subprompts/design-prompts.md` — load only at Step 8 (design)
- `ai-framework/06-user-stories.md` — load only at Step 10
- `ai-framework/06b-timeline.md` — load only at Step 10.5 (Timeline)
- `ai-framework/07-drive-sync.md` — load only if Drive export was enabled at Step 11 pre-flight

### Gate context checkpoints — write after every gate approval

After every gate is approved, write a compact context checkpoint to:
```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_context-checkpoint.md
```

Overwrite this file each time (only the most recent checkpoint is needed).

The checkpoint captures all decisions and artifact paths in compressed form so that subsequent sessions can reconstruct context without re-reading every artifact in full:

```markdown
# Context Checkpoint — [feature-name]
Last updated: [YYYY-MM-DD] after Gate [N] approval

## Pipeline
- Type: Work
- Mode: [Fast / Gated]
- Current phase: [N]

## Key decisions (Decision Log summary)
- [Decision 1 — one line]
- [Decision 2 — one line]
- [Add one line per locked decision from the PRD decision log]

## Open conflicts awaiting PM resolution
- [Conflict N — one line summary, or "none"]

## Artifact paths
| Artifact | Path |
|----------|------|
| PRD | [path] |
| Research | [path or N/A] |
| Codebase review | [path or N/A] |
| Product review | [path or N/A] |
| Technical review | [path or N/A] |
| System design | [path or N/A] |
| Visual diagram | [path or N/A] |
| Design catalog Phase N | [path or N/A] |
| User stories | [path or N/A] |

## Gate status
- Gate 1: [Approved YYYY-MM-DD / Pending]
- Gate 2: [Approved YYYY-MM-DD / Pending / N/A]
- Gate 3: [Approved YYYY-MM-DD / Pending / N/A]

## Next step
[Step number and name]
```

### Session resumption with checkpoint

When resuming a session, prefer reading `_context-checkpoint.md` over reading every artifact file from scratch. The checkpoint gives you all decisions and paths without loading the full PRD and all reviews into context.

Only read the full artifact files when the current step requires processing their content (e.g. applying fixes, syncing, validating). For routing and state decisions, the checkpoint is sufficient.

---

## OUTPUT CONVENTIONS

All pipeline outputs write to:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/
```

Rules:
- Feature name confirmed with PM at intake (see "Feature name confirmation" below).
- Root folder (`~/Desktop/Resources/PDLC Workflow Docs/`) created automatically if it does not exist.
- Feature subfolder created automatically when the first file is about to be written.
- Subfolders inside the feature folder are created only when the first file for that stage is about to be written, not all at once upfront.
- Every step confirms the full file path before writing by printing: `Writing: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/[subfolder]/[filename]`.
- No step writes to `docs/` or any other path. The new root replaces the old path entirely for all new runs.
- Existing files under the old `docs/` path are not touched by new runs.
- Changelog folder appends, never overwrites.
- `_pipeline-state.json` is written at the end of every step (overwrite each step) to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_pipeline-state.json`.
- `_pipeline-state.json` is read at the start of every new conversation before anything else. Integrity verification runs before any step routing.
- Knowledge base lives at the root (not inside any feature folder): `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md`. Append-only.

### Subfolder structure within each feature folder

```
[feature-name]/
├── research/
├── codebase-review/
├── prd/
├── product-review/
├── technical-review/
├── user-stories/
├── diagrams/
├── design/
├── jira-export/
├── stakeholders/
└── changelog/
```

### File paths per step

| Source prompt | Output path |
|---|---|
| `00-codebase-review.md` | `codebase-review/[feature]-codebase-review.md` |
| `01-research-idea.md` | `research/[feature]-research.md` |
| `02-create-prd.md` | `prd/[feature]-prd.md` |
| `review-prd.md` | `product-review/[feature]-product-review.md` |
| `cto-review.md` | `technical-review/[feature]-technical-review.md` |
| `system-design.md` | `technical-review/[feature]-system-design.md` |
| `03c-visual-diagram.md` | `diagrams/[feature]-feature-diagram.md` |
| `design-prompts.md` | `design/[feature]-phase-[N]-designs.md` |
| `03b-update-prd-from-designs.md` | `prd/[feature]-prd.md` (overwrite in place) |
| `prd-to-jira.md` | `jira-export/[feature]-jira-export.md` |
| `prd-to-jira.md` (manifest) | `jira-export/[feature]-jira-manifest.md` |
| `06-user-stories.md` | `user-stories/[feature]-user-stories.md` |
| `06b-timeline.md` (HTML) | `timeline/[feature]-timeline.html` |
| `06b-timeline.md` (sidecar) | `timeline/[feature]-timeline.md` |
| `05-change-propagation.md` (changelog) | `changelog/[feature]-changelog.md` (append) |
| `05-change-propagation.md` (summary) | `changelog/[feature]-change-[date]-summary.md` |
| Stakeholder list | `stakeholders/[feature]-stakeholders.md` |
| Pipeline state | `_pipeline-state.json` (overwrite each step) |
| Context checkpoint | `_context-checkpoint.md` (overwrite each gate) |
| Open conditions | `_open-conditions.md` (append per gate, only if conditions set) |
| Knowledge base (root) | `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md` (append) |

### Feature name confirmation at intake

After the PM provides the feature name at the start of a pipeline run, display the derived folder name and ask for confirmation before creating anything:

```
Feature name entered: [what PM typed]
Output folder will be: ~/Desktop/Resources/PDLC Workflow Docs/[derived-name]/

Confirm? (yes / use a different name)
```

Derivation rule: replace spaces with hyphens, all lowercase. Example: `Customer Onboarding Flow` → `customer-onboarding-flow`.

---

## SESSION RESUMPTION

At the very top of every new conversation, before any step routing:

1. Read `_pipeline-state.json` from the feature folder. If it does not exist, start fresh.
2. Run integrity verification (see State Tracking Rules above). Warn the PM about any unverified artifacts before proceeding.
3. Read `_context-checkpoint.md` to reconstruct decisions and key context without loading full artifact files.
4. Print to the PM:

```
Resuming [feature-name] from [step number] — [step name].
Pipeline: [type] | Mode: [Fast/Gated] | Phase: [N]

Artifact integrity: [N] verified | [N] unverified (listed below if any)
[List any unverified artifacts with paths]

Gates: Gate 1 [status] | Gate 2 [status] | Gate 3 [status]

Shall I continue from here? (yes / start over / pick a different step)
```

Only after the PM confirms should you continue.

---

## ENTRY POINTS

In addition to running through the pipeline linearly, the following commands are available at any time:

| Command | Action |
|---|---|
| `/change-mode` | Load and follow `ai-framework/05-change-propagation.md`. Available at any time after Gate 1. Does not interrupt or reset the build pipeline. |
| `/reopen-gate-1` | Flag all artifacts produced after Gate 1 as `DRAFT`. Re-run the steps between research and Gate 1. Re-present Gate 1 with a full quality check. |
| `/reopen-gate-2` | Flag all artifacts produced after Gate 2 as `DRAFT`. Re-run the steps between Gate 1 and Gate 2. Re-present Gate 2 with a full quality check. |
| `/reopen-gate-3` | Flag all artifacts produced after Gate 3 as `DRAFT`. Re-run the steps between Gate 2 and Gate 3. Re-present Gate 3 with a full quality check. |

Reopen-gate behavior is defined in `ai-framework/error-handling.md` (Error Type 1).

---

## Stage 0 — Mode Selection and Intake

Ask the user: **"Fast mode or gated mode?"**

| Mode | Behavior |
|------|----------|
| **Fast** (default) | Chains all auto-steps without pausing. Stops only at 3 approval gates: PRD, Designs, User Stories Breakdown. |
| **Gated** | Pauses after every step. Use when reviewing each output matters. |

Default to fast mode if the user does not specify.

Do not proceed until mode is confirmed.

After confirmation, ask for the **feature name** and run the Feature name confirmation block from OUTPUT CONVENTIONS above.

Then ask about optional review lenses. Read `ai-framework/personas.md` — "Optional Review Lenses" section. Based on the product type and feature description, suggest relevant additional lenses and ask whether to activate them. Do not activate lenses without PM confirmation. Record activated lenses in `_pipeline-state.json` under a `review_lenses` array (always includes `"product"` and `"technical"`; add any optional ones the PM confirmed).

---

## Output Patterns

### Fast mode — auto-step (no pause)
```
✓ Step [N] — [step name] — [one-line summary] → [output file path]
```
Proceed immediately to the next step.

### Fast mode — parallel block completion
```
✓ Step [N]a/b/c — [block name] (parallel) — [summary of what ran]
  → [file path Na]
  → [file path Nb]
  [N] agreements | [N] conflicts | [N] single-source findings
```

### Fast mode — approval gate (pause required)
```
━━━ APPROVAL NEEDED: Gate [N] — [Gate Name] ━━━

What was produced:
[Bullet list of outputs with file paths]

━━━ QUALITY CHECK ━━━
[list of flags with severity: WARNING or INFO, or "Quality check passed. No issues found."]
━━━ [N] flags found. You can approve anyway or address these first. ━━━

Progress:
[x] Completed steps
[ ] Next step after approval

[Specific instruction for the user]

Say "approved" to continue, or give feedback to revise.
━━━
```

### Gated mode — step pause
```
━━━ Step [N] complete: [Step Name] ━━━

What was done: [1-2 sentence summary]
Output saved to: [file path]

Progress: [checklist with sequential step numbers]

Next step: Step [N+1] — [name — what it does]

Say "continue" to proceed, or provide additional input.
━━━
```

---

## Work Pipeline

### Step 1 — Research Idea [AUTO after initial clarification]

Same as Personal Full Step 1.

### Step 2 — Codebase Review [AUTO]

Read and follow: `ai-framework/00-codebase-review.md`

Output: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/codebase-review/[feature-name]-codebase-review.md`

Pass the one-paragraph handoff summary to Step 3 (Create PRD) as context.
Pass the full codebase review output file path to Step 4b (Technical Review) as an input alongside the PRD.

**Fast mode:** `✓ Step 2 — Codebase review complete → [path]`

Update `_pipeline-state.json`.

### Step 3 — Create PRD [AUTO]

Same as Personal Full Step 2. Use both the research output (Step 1) and the codebase review handoff summary (Step 2) as context. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`.

### Steps 4a + 4b — Dual Review [PARALLEL AUTO]

Spawn two agents at the same time:

**Step 4a — Product Review:**
Same as Personal Full Step 3a. Apply Product Reviewer persona. Save to `product-review/[feature-name]-product-review.md`.

**Step 4b — Technical Review:**
Apply Technical Reviewer persona from `ai-framework/personas.md`. **Inputs:** both the PRD file path AND the codebase review output file path from Step 2. Evaluate them together. Specifically assess whether the PRD is consistent with the codebase review findings, whether HIGH risks from the codebase review are addressed in the PRD, and whether the proposed build approach aligns with existing patterns identified in the codebase review. Save to `technical-review/[feature-name]-technical-review.md`.

Synthesize. Conflicts → structured conflict cards per `error-handling.md` Error Type 2.

### Step 5 — Apply Fixes [AUTO → GATE 1]

Same as Personal Full Step 4, plus the Work-specific quality check item:
- Is the codebase review handoff note reflected in the PRD? If the codebase review flagged a HIGH risk and the PRD does not address it, flag it.

At Gate 1, also ask:
1. Complex architecture needing a system design doc? (yes/no)

### Step 6 — System Design [AUTO if yes at Gate 1]

Same as Personal Full Step 5a (without the Phase 1 screen inventory prep agent — Work pipeline does not implement). Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-system-design.md`. Skip entirely if PM said no at Gate 1.

**Fast mode:** `✓ Step 6 — System design complete → [path]` (or `✓ Step 6 — Skipped (no system design needed)`)

Update `_pipeline-state.json`.

### Step 7 — Visual Diagram [AUTO]

Read and follow: `ai-framework/03c-visual-diagram.md`. Output: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`.

**Fast mode:** `✓ Step 7 — Visual diagram complete → [path]`

Update `_pipeline-state.json`.

### Step 8 — Design (Per Phase) [AUTO after tool selection]

Ask the PM: **"v0 or Figma Make?"** — both use the same prompt format. Figma Make can additionally reference your team's Figma design system components directly.

Read and follow: `subprompts/design-prompts.md`

Output: design catalog at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`.

**Fast mode:** `✓ Step 8 — Design complete → [path]`

Update `_pipeline-state.json`.

### Step 9 — Update PRD from Designs [AUTO → GATE 2]

Same as Personal Full Step 8. Sync the PRD with design catalog: catalog reference, copy/flow changes, AC updates, decision log entries. Quality checks run before Gate 2 (see Personal Full Step 8 for the check list).

After Gate 2 approval, proceed to Step 10.

Update `_pipeline-state.json`. Mark Gate 2 as Approved with date.

### Step 10 — User Stories Breakdown [AUTO → GATE 3]

Read and follow: `ai-framework/06-user-stories.md`.

Inputs: the final design-informed PRD (post-Gate 2) + the design catalog + the codebase review.

Output: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md`. Standalone document with:
- Build Sequence Map (every story, type FE/BE, phase, depends-on, related-to, size).
- Per-story sections: As-a/I-want/So-that + exhaustive Gherkin AC (happy + negative + edge + error) + testing notes (coverage areas, cross-boundary verification, edge cases, data conditions).

**Gate 3 — Breakdown approval — Quality Check (before presenting the gate):**

Run all of the following:

- Every PRD user story appears in the breakdown (no drops).
- Every story has a unique US-ID.
- Every story has at least 2 Gherkin scenarios.
- Every story has at least one edge-case or error-state scenario.
- Every linked FE/BE pair has both sides present.
- No story sized larger than L without a proposed split.
- HIGH risks from the codebase review appear in at least one story's testing notes.
- UX state coverage per FE story: empty / loading / error / populated.

Format and present per the Gate 1 quality check format.

**Both modes — show Gate 3:**

```
━━━ APPROVAL NEEDED: Gate 3 — User Stories Breakdown ━━━

What was produced:
- User stories breakdown: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md
- Build Sequence Map with [N] stories ([N] FE, [N] BE)

[QUALITY CHECK block]

Progress:
[x] Step 9 — PRD synced with designs (Gate 2 approved)
[x] Step 10 — User Stories Breakdown
[ ] Step 11 — Jira Export  ← runs after approval

Review the breakdown and the Build Sequence Map. Once approved, tickets will be created in Jira and cannot be easily undone.

Say "approved" to create tickets, or give feedback to revise.
━━━
```

Update `_pipeline-state.json`. Mark Gate 3 as Approved with date.

**Optional — publish breakdown to Confluence and share for engineering review:**
After Gate 3 approval, offer once:
```
Breakdown approved. Want to publish it to Confluence and share it with the engineering
team for sizing review before Jira tickets are created? (yes / skip)
```

If yes:
1. Publish the user stories breakdown as a Confluence page — read and follow
   `subprompts/prd-to-confluence.md` using the breakdown file as the source document.
   Record the URL in `_pipeline-state.json` → `export_urls.confluence_breakdown_page`.
2. Then read and follow `subprompts/share-for-review.md` using that URL.

Note: Step 11c (Confluence publish) runs next and will publish or update the full PRD
page. The breakdown page created here is a separate page, not a duplicate.

### Step 10.5 — Timeline (Gantt) [AUTO]

Read and follow: `ai-framework/06b-timeline.md`.

Inputs: the approved user-stories breakdown from Step 10 (primary), the PRD (for phase order and any committed milestones), and `_pipeline-state.json` (for prior timeline parameters if re-running).

The step gathers timeline parameters from the PM in one turn (start date, sprint length, team composition, velocity, buffer, optional target launch and off-time), proposes durations at the Epic + Phase level using a hybrid estimation model (sizing × velocity, PM tunes), and produces two outputs:

- **Figma FigJam timeline** — via Figma MCP `generate_diagram`. URL stored in `_pipeline-state.json` → `export_urls.figma_timeline_url`. If the Figma MCP is unavailable, the step skips this output, notes it in the markdown sidecar, and continues.
- **Interactive HTML Gantt** — self-contained HTML file at `timeline/[feature-name]-timeline.html`. Opens offline. Hover details on every bar.

Granularity is Epic + Phase, not story-level. A 60-story feature produces a roadmap with one bar per epic grouped under phase headers, not 60 bars.

Math is honest: if the PM provided a target launch date and the computed end exceeds it, the step surfaces the gap and offers scope cut / team increase / slip — it does not silently shrink durations.

No gate. After the PM accepts the timeline ("yes" at Step 8 of the underlying prompt), proceed to Step 11.

Update `_pipeline-state.json`:
- `timeline.parameters`, `timeline.computed`, `timeline.outputs.html_path`, `timeline.outputs.markdown_path`.
- `export_urls.figma_timeline_url` (only if Figma succeeded).

### Step 11 — Export (parallel: Jira always + Drive optional + Confluence optional)

After Gate 3 approval, run a **pre-flight question** before the parallel block:

```
━━━ Step 11 — Export pre-flight ━━━

Jira ticket creation will run automatically (primary export).

Two optional exporters can run in parallel:

[ ] Sync all artifacts to Google Drive
    (Mirrors the local feature folder. Useful for stakeholder sharing.)

[ ] Publish PRD as a Confluence page
    (For teams that read documentation in Confluence.)

Enable either, both, or neither. (Type "jira only" / "jira + drive" /
"jira + confluence" / "all three" / or list them.)

If you enable Drive, I'll need: Drive folder URL/ID or name; optional team-share emails.
If you enable Confluence, I'll need: Confluence space + (optional) parent page.
━━━
```

Collect the answers and any required inputs before spawning the parallel block.

### Parallel block

Spawn up to three agents simultaneously based on what the PM enabled. Apply `ai-framework/05-parallel-rules.md` — Block 3.

#### Step 11a — Jira Export (always runs)

Read and follow: `subprompts/prd-to-jira.md`.

Inputs: the approved user-stories breakdown from Step 10 (primary), the PRD (fallback).

Creates one Jira ticket per story in the breakdown. Composes the Epic description from PRD content (never local file paths). Attaches the PRD file to the Epic. Sets all custom fields (User Story ADF, Acceptance Criteria ADF with verbatim Gherkin, Testable, FE/BE labels, sequence labels `seq-01...`, size labels `size-S/M/L`, team labels). Parent: Epic key or created Epic. Relates-to: linked FE/BE pair counterpart.

Writes a manifest file: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-manifest.md` mapping US-ID → Jira issue key. Used by `/change-mode` for future updates.

If the Jira MCP is unavailable, applies Error Type 4 from `error-handling.md`: writes the entire intended export locally for manual creation.

#### Step 11b — Google Drive Sync (optional)

Only runs if the PM enabled Drive at the pre-flight.

Read and follow: `ai-framework/07-drive-sync.md`.

Inputs: the Drive folder location and (optional) team-share emails collected at pre-flight.

Mirrors the local feature folder to Drive: research/, codebase-review/, prd/, product-review/, technical-review/, diagrams/, design/, user-stories/, jira-export/, changelog/. Generates a `_FEATURE_SUMMARY.md` at the feature folder root with quick links to every artifact.

If the Drive MCP is unavailable, skips cleanly with a notification. Pipeline continues — Jira Export proceeds in parallel.

#### Step 11c — Confluence Publish (optional)

Only runs if the PM enabled Confluence at the pre-flight.

Read and follow: `subprompts/prd-to-confluence.md`.

Inputs: Confluence space + (optional) parent page collected at pre-flight.

**If a Confluence page already exists** (the PM used `/share-for-review` earlier and
a page URL is recorded in `_pipeline-state.json` → `export_urls.confluence_page`):
call `updateConfluencePage` to update the existing page rather than creating a new one.
Add a quick-links section at the top pointing to the Jira Epic (Step 11a) and Drive
folder (Step 11b) once those are available. Do not create a duplicate page.

**If no page exists yet:** publish fresh — same behavior as before.

If the Atlassian MCP is unavailable, applies Error Type 4: writes intended page content locally. Pipeline continues.

### Synthesis

After all enabled agents finish, the orchestrator returns a single summary:

```
✓ Step 11 — Export complete
  → Jira: [Epic URL] · [N] tickets created
  → Drive: [folder URL] · [N] files synced  (if enabled)
  → Confluence: [page URL]                   (if enabled)
```

If any of the three failed, the failure is reported but the others still succeed independently — there is no global rollback.

Update `_pipeline-state.json` with the export results, including the Drive folder URL and Confluence page URL if they were enabled.

### Work Pipeline Complete

```
━━━ Work Pipeline Complete ━━━

Planning and design phase done. No implementation in this pipeline.

What was produced:
- Research: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/research/[feature-name]-research.md
- Codebase review: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/codebase-review/[feature-name]-codebase-review.md
- PRD: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md
- Product review: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[feature-name]-product-review.md
- Technical review: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-technical-review.md
- System design: [path or N/A]
- Visual diagram: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md
- Design catalog: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md
- User stories breakdown: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md
- Jira tickets: [Epic URL or local fallback path]
- Jira manifest: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-manifest.md
- Google Drive folder: [Drive URL]   (if Drive sync was enabled)
- Confluence page: [Confluence URL]    (if Confluence publishing was enabled)

Next step:
When the designer delivers Figma updates, run /compare-figma-prd to sync Figma with the PRD and Jira.
Run /team-status to see where this product sits in the portfolio.
If anything changes after any gate, run /change-mode for safe propagation.
━━━
```

---

## Team Usage

**PM starting a product:** Run `/build-product` → Work pipeline → Jira tickets created automatically.

**Designer joining mid-pipeline:** Run `/feature-kickoff [feature-name]` with role = Designer, deliver in Figma, PM runs `/compare-figma-prd`.

**Seeing the full team's work:** Run `/team-status`.

**Propagating a change after Gate 1:** Run `/change-mode`.

**Reopening an approved gate:** Run `/reopen-gate-1`, `/reopen-gate-2`, or `/reopen-gate-3`.

---

## State Tracking Rules

- `_pipeline-state.json` is the source of truth for pipeline state. Write it at the end of every step. It is machine-parseable JSON, not prose markdown.
- On every new session start, read `_pipeline-state.json` before anything else. Verify artifact integrity before trusting the state (see "Integrity verification" below).
- If interrupted, the user can run `/project-status` to see where they left off. The orchestrator reads `_pipeline-state.json` first and offers to resume.
- If the user provides a previous research summary or PRD, treat those as completed steps and skip to the appropriate point.

### `_pipeline-state.json` schema

Write this file at the end of every step, overwriting the previous version:

```json
{
  "feature_name": "string",
  "pipeline": "Work",
  "mode": "Fast | Gated",
  "current_phase": 1,
  "current_step": "string — step number and name, e.g. '3b — Technical Review'",
  "next_step": "string — step number and name, e.g. '4 — Apply Fixes'",
  "gates": {
    "gate_1": "Approved YYYY-MM-DD | Pending | N/A",
    "gate_2": "Approved YYYY-MM-DD | Pending | N/A",
    "gate_3": "Approved YYYY-MM-DD | Pending | N/A"
  },
  "artifacts": {
    "research": { "path": "string | null", "size_bytes": 0 },
    "codebase_review": { "path": "string | null", "size_bytes": 0 },
    "prd": { "path": "string | null", "size_bytes": 0 },
    "product_review": { "path": "string | null", "size_bytes": 0 },
    "technical_review": { "path": "string | null", "size_bytes": 0 },
    "system_design": { "path": "string | null", "size_bytes": 0 },
    "visual_diagram": { "path": "string | null", "size_bytes": 0 },
    "design_catalogs": [
      { "phase": 1, "path": "string", "size_bytes": 0 }
    ],
    "user_stories": { "path": "string | null", "size_bytes": 0 },
    "jira_manifest": { "path": "string | null", "size_bytes": 0 }
  },
  "export_urls": {
    "jira_epic": "string | null",
    "drive_folder": "string | null",
    "confluence_page": "string | null — PRD page, created at Step 11c",
    "confluence_breakdown_page": "string | null — user stories breakdown page, created at Gate 3 share",
    "figma_diagram_url": "string | null — FigJam diagram URL, null if Mermaid fallback was used"
  },
  "open_conflicts": [],
  "review_requests": [
    {
      "artifact": "string — e.g. 'PRD', 'Design catalog Phase 1'",
      "confluence_url": "string | null",
      "slack_channel": "string | null",
      "slack_ts": "string | null — Slack message timestamp, null if MCP unavailable",
      "deadline": "string | null",
      "reviewers": ["string"],
      "posted_at": "YYYY-MM-DDTHH:MM:SSZ",
      "feedback_read": false,
      "feedback_applied_at": "YYYY-MM-DDTHH:MM:SSZ | null",
      "edits_applied": 0
    }
  ],
  "last_updated": "YYYY-MM-DDTHH:MM:SSZ"
}
```

For `size_bytes`: after writing each artifact, record the file size in bytes. Use `wc -c [path]` or equivalent. Use `0` only if the file has not been written yet.

### Integrity verification on session resumption

When reading `_pipeline-state.json` at the start of a new session, verify each artifact that the state claims exists:

1. Check that the file path actually exists on disk.
2. Check that the recorded `size_bytes` is non-zero and plausible (> 100 bytes for any real document).
3. If a file is missing or its size is 0, mark that step as `UNVERIFIED` in your working state and warn the PM:

```
⚠ State integrity issue: [artifact name] listed in _pipeline-state.json but file not found at [path].
  This step may need to be re-run. Proceeding from the last verified step instead.
```

Do not trust a gate as approved if any artifact that gate depended on fails verification.

---

## Final Summary

When the pipeline ends, output:

```
━━━ Pipeline Complete ━━━

Pipeline: Work
Mode: [Fast / Gated]
Feature: [Name]
Output folder: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/
Owner: [Name]

Documents created:
- Research: [path]
- Codebase Review: [path]
- PRD: [path]
- Product Review: [path]
- Technical Review: [path]
- System Design: [path or N/A]
- Visual Diagram: [path]
- Design Catalog(s): [paths per phase]
- User Stories Breakdown: [path]
- Jira tickets: [Epic URL or local fallback path]
- Jira manifest: [path]
- Google Drive folder: [URL or N/A]
- Confluence page: [URL or N/A]
- Changelog: [path if any change-mode runs occurred]

Gates:
- Gate 1: [Approved YYYY-MM-DD]
- Gate 2: [Approved YYYY-MM-DD]
- Gate 3: [Approved YYYY-MM-DD]
━━━
```
