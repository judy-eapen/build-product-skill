---
name: build-product
description: End-to-end product development pipeline. Use for new products, features, or bug fixes. Handles Personal (Full/Medium/Light) and Work pipelines with parallel reviews, validation, and exports. Starts with project type selection.
---

# Build Product (Parallel Orchestrator)

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

End-to-end product development pipeline with parallelized steps for maximum throughput. Handles Personal and Work projects across Full, Medium, and Light pipelines.

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
- `ai-framework/03-design.md` / `subprompts/design-with-v0.md` — load only at Step 7
- `ai-framework/06-user-stories.md` — load only at Step 10 (Work pipeline)
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
- Type: [Personal Full / Personal Medium / Personal Light / Work]
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
| Validation Phase N | [path or N/A] |

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
├── success-metrics/
├── diagrams/
├── design/
├── validation/
├── jira-export/
├── qa-scenarios/
├── stakeholders/
├── changelog/
└── learning/
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
| `03-design.md` / `design-with-v0.md` | `design/[feature]-phase-[N]-designs.md` |
| `03b-update-prd-from-designs.md` | `prd/[feature]-prd.md` (overwrite in place) |
| `validate.md` | `validation/[feature]-phase-[N]-validation.md` |
| `04b-update-prd-from-build.md` | `prd/[feature]-prd.md` (overwrite in place) |
| `prd-to-jira.md` | `jira-export/[feature]-jira-export.md` |
| `prd-to-jira.md` (manifest) | `jira-export/[feature]-jira-manifest.md` |
| `06-user-stories.md` | `user-stories/[feature]-user-stories.md` |
| `05-change-propagation.md` (changelog) | `changelog/[feature]-changelog.md` (append) |
| `05-change-propagation.md` (summary) | `changelog/[feature]-change-[date]-summary.md` |
| `learn.md` | `learning/[feature]-phase-[N]-learning.md` |
| User stories | `user-stories/[feature]-user-stories.md` |
| Success metrics | `success-metrics/[feature]-success-metrics.md` |
| QA scenarios | `qa-scenarios/[feature]-qa-scenarios.md` |
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

## Stage 0 — Project Type, Pipeline, and Mode Selection

Ask the user: **"Personal or work project?"**

| Type | Behavior |
|------|----------|
| **Personal** | Full implementation. AI executes, validates, ships. |
| **Work** | Planning and design only. No implementation. User stories go to Jira. Ends after design approval. |

Then for **Personal**, ask pipeline:

| Pipeline | When to use |
|----------|------------|
| **Full** | New product, greenfield app, major feature with new data model |
| **Medium** | New feature in existing app, significant scope |
| **Light** | Bug fix, small UI change, config change |

Then ask mode:

| Mode | Behavior |
|------|----------|
| **Fast** (default) | Chains all auto-steps without pausing. Stops only at 3 approval gates: PRD, Designs, Ship. |
| **Gated** | Pauses after every step. Use when reviewing each output matters. |

If the user describes their idea instead of choosing, infer and confirm: "This sounds like a [personal/work] [full/medium/light] project in fast mode. Correct?"

Do not proceed until project type, pipeline, and mode are confirmed.

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

## Personal — Full Pipeline

### Step 1 — Research Idea [AUTO after initial clarification]

Read and follow: `ai-framework/01-research-idea.md`

Ask the user to describe their idea. Run Stage 1 (clarification questions) — this requires user input. After the user answers, run Stages 2–5 (market scan, strategic evaluation, 10x test, decision gate) and proceed to PRD creation.

Output path: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/research/[feature-name]-research.md`

If research was already completed in the current conversation (the user has already described and explored the idea in depth), confirm with the user: "We've covered a lot of research already — should I use this conversation as the research output, or do you want to run the full research stage?" If they confirm existing research is sufficient, save a research summary to the path above and proceed.

**Fast mode:** `✓ Step 1 — Research complete → [path]`
**Gated mode:** Pause. Next step: Step 2 — Create PRD.

Update `_pipeline-state.md` at the end of this step.

---

### Step 2 — Create PRD [AUTO]

Read and follow: `ai-framework/02-create-prd.md` (includes Stage 0 Knowledge Base Check).

Use research output from Step 1 as input. Run the full PRD generation process — clarification, generation, validation. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`.

**Fast mode:** `✓ Step 2 — PRD created → [path]`
**Gated mode:** Pause. Next step: Step 3 — Dual Review.

Update `_pipeline-state.md`.

---

### Steps 3a + 3b — Dual Review [PARALLEL AUTO]

Run both reviews simultaneously. See `ai-framework/05-parallel-rules.md` — Block 1.

Spawn two agents at the same time:

**Step 3a — Product Review:**
Review the PRD from a product lens. Apply the **Product Reviewer** persona from `ai-framework/personas.md`. Cover document quality (structure, problem definition, success metrics, scope, user stories, AC) and product quality (problem-solution fit, workflows, edge cases, empty/error/loading states, onboarding, phasing). Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[feature-name]-product-review.md`.

**Step 3b — Technical Review:**
Review the PRD from a technical lens. Apply the **Technical Reviewer** persona from `ai-framework/personas.md`. In the Personal Full pipeline there is no codebase review file; proceed with the PRD only. Cover architecture, data model, API contracts, security, phasing, dependencies, technical risks, cost vs value. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-technical-review.md`.

Wait for both to complete. Synthesize: identify agreements, conflicts, and single-source findings. Conflicts go to a structured conflict card per `error-handling.md` Error Type 2.

**Fast mode:** `✓ Step 3a + 3b — Dual review complete (parallel) → product-review + technical-review | [N] agreements | [N] conflicts`
**Gated mode:** Pause. Show synthesis summary. Next step: Step 4 — Apply Fixes.

Update `_pipeline-state.md`.

---

### Step 4 — Apply Fixes [AUTO → GATE 1]

Read both review files. Apply all findings to the PRD:

- Agreements (both reviews agree): apply immediately.
- Single-source findings: apply with a decision log note.
- Conflicts: present to user at Gate 1 as judgment calls (use the conflict card format from `error-handling.md`).

From the product review: update user stories, add empty/error/loading states, fix UX gaps, add missing acceptance criteria.
From the technical review: fix data model problems, update API contracts, address security concerns, correct architectural decisions.

Update the PRD file in place at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`. Add entries to the decision log.

Self-check (Error Type 3): does this output contradict any prior PRD decision? If yes, flag to PM before writing.

**Gate 1 — Quality Check (before presenting the gate):**

Run all of the following checks automatically:

- Is the problem statement specific enough that two engineers would build the same thing? If not, flag it.
- Does every user story have a measurable outcome? If not, flag which ones do not.
- Did the Technical Review surface any HIGH risks that have not been resolved in the decision log? If so, list them.
- Is the success metric a specific measurable metric with a current baseline, or is it a vague statement? If vague, flag it with a suggestion.
- (Work pipeline only) Is the codebase review handoff note reflected in the PRD? If the codebase review flagged a HIGH risk and the PRD does not address it, flag it.

Format the quality check output as:

```
━━━ QUALITY CHECK ━━━
[list of flags with severity: WARNING or INFO]
━━━ [N] flags found. You can approve anyway or address these first. ━━━
```

If zero flags are found, print: `Quality check passed. No issues found.`

**Both modes — show Gate 1:**

```
━━━ APPROVAL NEEDED: Gate 1 — PRD ━━━

What was produced:
- PRD (reviewed + fixed): ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md
- Product review: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[feature-name]-product-review.md
- Technical review: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-technical-review.md
- Fixes applied: [N] from reviews

[QUALITY CHECK block]

Conflicts requiring your decision:
[List each conflict card from Step 3a/3b synthesis]

Progress:
[x] Step 1 — Research
[x] Step 2 — PRD
[x] Step 3a + 3b — Dual review (parallel)
[x] Step 4 — Apply fixes
[ ] Step 5 — System design (optional)  ← next after approval
[ ] Step 6 — Visual diagram
[ ] Step 7 — Design

Also answer (optional):
- Complex architecture needing a system design doc? (yes/no)

Options:
- Say "approved" to continue with no open items.
- Say "approved with conditions: [list your conditions]" to advance the pipeline now and resolve these items before Gate 2.
- Give feedback to revise the PRD before advancing.
━━━
```

**If "approved with conditions":** Write the PM's conditions to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_open-conditions.md`:

```markdown
# Open Conditions

## From Gate 1 — approved [YYYY-MM-DD]
- [ ] [Condition 1 — what must be resolved before Gate 2]
- [ ] [Condition 2]
```

Then proceed. The Gate 2 quality check will automatically verify all Gate 1 conditions are resolved before advancing.

Update `_pipeline-state.json`. Mark Gate 1 as Approved with date. Write context checkpoint to `_context-checkpoint.md`.

---

### Steps 5a + 5b — System Design + Phase 1 Screen Inventory Prep [PARALLEL, if yes at Gate 1]

Only if the user said yes at Gate 1.

Spawn two agents simultaneously:

**Step 5a — System Design:**
Read and follow `subprompts/system-design.md`. Generate a system design document with: Overview, High-level Architecture, Data Structures, APIs, Feature-by-Feature Implementation, Key Algorithms, Tech Stack, Build Order, Open Technical Decisions. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-system-design.md`.

**Step 5b — Phase 1 Screen Inventory Prep:**
Read the PRD Phase 1 user stories. Produce a draft screen inventory: list of all distinct screens, the user story each maps to, and the states needed (empty / loading / error / populated). This is prep work for the design step — it is not final. Save as a note in the conversation context (do not write to file yet).

Wait for both. Proceed to Step 6.

**Fast mode:** `✓ Step 5a + 5b — System design + screen inventory prep complete (parallel) → [system design path]`

Update `_pipeline-state.md`.

---

### Step 6 — Visual Diagram [AUTO between Gate 1 and design]

Read and follow: `ai-framework/03c-visual-diagram.md`

Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`.

**Fast mode:** `✓ Step 6 — Visual diagram complete → [path]`
**Gated mode:** Pause. Next step: Step 7 — Design.

Update `_pipeline-state.md`.

---

### Step 7 — Design (Per Phase) [AUTO after mode selection]

Ask:

**"In-repo design (AI builds screens in code) or v0 (AI generates prompts for v0)?"**

Then read and follow:
- In-repo: `ai-framework/03-design.md`
- v0: `subprompts/design-with-v0.md`

Use the screen inventory from Step 5b if available — skip the screen discovery phase.

After Phase 1: also create the design tokens file at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-design-tokens.md`.

**Fast mode:** `✓ Step 7 — Design complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`
**Gated mode:** Pause. Next step: Step 8 — Update PRD from Designs.

Update `_pipeline-state.md`.

---

### Step 8 — Update PRD from Designs (Per Phase) [AUTO → GATE 2]

Read and follow: `ai-framework/03b-update-prd-from-designs.md`

Sync the PRD with the finalized design catalog: design catalog reference, copy/flow changes, AC updates, decision log entries, frontend task list updates. Overwrite the PRD in place at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`.

Self-check (Error Type 3): does this output contradict any prior PRD decision? If yes, flag to PM before writing.

**Gate 2 — Quality Check (before presenting the gate):**

Run all of the following checks automatically:

- Does the visual diagram cover every user story approved at Gate 1? Flag any stories with no corresponding flow.
- Did the compliance check surface any HIGH risk items that have not been addressed in the designs? If so, list them.
- Do the design prompts or mockups cover all states: empty state, loading state, error state? Flag any missing states.
- **Open conditions from Gate 1:** If `_open-conditions.md` exists and contains unchecked Gate 1 conditions, verify each one. For each condition, assess whether it has been addressed in the design or updated PRD. Flag any that remain unresolved as WARNING.

Format and present per the Gate 1 quality check format.

**Both modes — show Gate 2:**

```
━━━ APPROVAL NEEDED: Gate 2 — Designs ━━━

What was produced:
- Design catalog: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md
- PRD updated to reflect designs

[QUALITY CHECK block]

Open conditions from Gate 1: [N resolved / N unresolved — list any unresolved]

Progress:
[x] Step 1–4 — PRD (reviewed + fixed)
[x] Step 5a/5b — System design (if applicable)
[x] Step 6 — Visual diagram
[x] Step 7 — Phase [N] designs
[x] Step 8 — PRD synced with designs
[ ] Step 9 — Execute Phase [N]  ← next after approval

[Instructions for reviewing designs based on method used]

Options:
- Say "approved" to begin implementation with no open items.
- Say "approved with conditions: [list]" to advance and resolve these before Gate 3.
- Give feedback to revise designs before advancing.
━━━
```

**If "approved with conditions":** Append to `_open-conditions.md`:

```markdown
## From Gate 2 — approved [YYYY-MM-DD]
- [ ] [Condition 1 — what must be resolved before Gate 3]
```

Update `_pipeline-state.json`. Mark Gate 2 as Approved with date. Write context checkpoint to `_context-checkpoint.md`.

---

### Step 9 — Execute (Per Phase) [AUTO with internal parallelism]

Read and follow: `ai-framework/04-execute-plan.md`

Execute the current phase. Implementation order:
1. Data layer and migrations
2. Backend services
3. API endpoints
4. Frontend screens (reference design catalog)
5. Error handling and loading states
6. Logging and observability
7. Tests

**Internal parallelism during execute:**
After each layer is implemented, spawn a background agent to write tests for that layer while the main agent continues to the next layer. Do not wait for tests before continuing — tests are parallelized with the next implementation layer.

**Phase pipelining (if multi-phase product):**
After frontend implementation is complete but before tests finish, spawn a background agent to: read Phase N+1 scope from the PRD and draft the Phase N+1 screen inventory. This will be available at Gate 3.

**Fast mode:** `✓ Step 9 — Phase [N] implementation complete`
**Gated mode:** Pause. Next step: Step 10 — Validate.

Update `_pipeline-state.md`.

---

### Steps 10a + 10b + 10c — Validate (Per Phase) [PARALLEL AUTO]

Run three validation checks simultaneously. See `ai-framework/05-parallel-rules.md` — Block 2.

Spawn three agents at the same time:

**Step 10a — AC Compliance:**
Check every acceptance criterion in the PRD for the current phase. Verdict per criterion: PASS / PARTIAL / FAIL.

**Step 10b — Design Match:**
Check current UI against the design catalog. If no catalog exists, skip. Note structural deviations (not style preferences).

**Step 10c — NFR Check:**
Check performance thresholds, error states, loading states, empty states, auth enforcement, mobile layout, logging.

Wait for all three. Merge into a single report at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/[feature-name]-phase-[N]-validation.md`.

If any FAIL items: `⚠ [N] failures found — resolving` and fix inline before proceeding.

**Fast mode:** `✓ Step 10a + 10b + 10c — Validation complete (parallel) → validation report | [N] pass | [N] notes | [N] failures`
**Gated mode:** Pause. Show full report. Next step: Step 11 — Update PRD from Build.

Update `_pipeline-state.md`.

---

### Step 11 — Update PRD from Build (Per Phase) [AUTO → GATE 3]

Read and follow: `ai-framework/04b-update-prd-from-build.md`

Sync the PRD with what was actually built: data model changes, implementation notes, deferred scope, copy/flow changes, decision log entries. Overwrite the PRD in place.

Self-check (Error Type 3): does this output contradict any prior PRD decision? If yes, flag to PM before writing.

**Gate 3 — Quality Check (before presenting the gate):**

Run all of the following checks automatically:

- Does every ticket have at least 2 acceptance criteria? Flag any that do not.
- Does every ticket have at least one edge case or error state documented? Flag any that do not.
- (Work pipeline only) Are there any HIGH risks from the codebase review that do not appear in any ticket? Flag them.
- Does the build sequence have any circular dependencies? Flag if found.
- **Open conditions from Gate 2:** If `_open-conditions.md` exists and contains unchecked Gate 2 conditions, verify each one. Flag any that remain unresolved as WARNING.

Format and present per the Gate 1 quality check format.

**Both modes — show Gate 3:**

```
━━━ APPROVAL NEEDED: Gate 3 — Ship ━━━

Phase [N] is built and validated.

What was produced:
- Implementation: Phase [N] complete
- Validation: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/[feature-name]-phase-[N]-validation.md ([pass/fail])
- PRD synced with build

[QUALITY CHECK block]

Open conditions from prior gates: [N resolved / N unresolved — list any unresolved]

Phase N+1 design prep: [available / not started]

Progress:
[x] Step 7 — Phase [N] design
[x] Step 9 — Phase [N] execute
[x] Step 10a–c — Phase [N] validate
[x] Step 11 — PRD synced
[ ] Ship Phase [N]  ← you are here
[ ] Step 12 — Learn

Deployment: [guidance based on tech stack]

Say "shipped" when deployed, or "skip ship" to go straight to the learning report.
━━━
```

Update `_pipeline-state.md`. Mark Gate 3 as Approved with date.

---

### Step 12 — Learn (Per Phase) [AUTO after "shipped"]

Read and follow: `subprompts/learn.md` instructions.

Collect feedback, check live app, review success metrics, identify what struggled and what worked. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/learning/[feature-name]-phase-[N]-learning.md`.

After the learning report is written, the Knowledge Base Update section in `learn.md` appends a structured entry to `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md`.

After the learning report, ask:

```
━━━ Phase [N] complete ━━━

Learning report: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/learning/[feature-name]-phase-[N]-learning.md
Knowledge base updated: ~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md

What's next?
- "continue" — Start Phase [N+1] (loops back to Step 7 — Phase N+1 screen inventory is already prepped)
- "done" — End pipeline. Show final summary.
- "revise PRD" — Update PRD based on learnings, then continue.
━━━
```

Update `_pipeline-state.md`.

---

## Personal — Medium Pipeline

### Step 1 — Create PRD [AUTO after clarification]

Read and follow: `ai-framework/02-create-prd.md`. Same as Full Pipeline Step 2.

### Steps 2a + 2b — Dual Review [PARALLEL AUTO]

Spawn parallel review agents (same as Full Pipeline Steps 3a + 3b — same persona files, same output paths). Synthesize.

### Step 3 — Apply Fixes [AUTO → GATE 1]

Same as Full Pipeline Step 4. Apply quality check, present Gate 1.

### Step 4 — Execute [AUTO]

Read and follow: `ai-framework/04-execute-plan.md`. Same as Full Pipeline Step 9 (with internal parallelism).

### Steps 5a + 5b + 5c — Validate [PARALLEL AUTO]

Run parallel validation (same as Full Pipeline Steps 10a + 10b + 10c).

### Step 6 — Update PRD from Build [AUTO → end]

Read and follow: `ai-framework/04b-update-prd-from-build.md`. After syncing, show final progress and end.

---

## Personal — Light Pipeline

### Step 1 — Execute

Read and follow: `ai-framework/04-execute-plan.md`

Ask the user to describe the bug fix or change. Lock scope and implement. When done, show final progress and end.

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

Update `_pipeline-state.md`.

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

Update `_pipeline-state.md`.

### Step 7 — Visual Diagram [AUTO]

Read and follow: `ai-framework/03c-visual-diagram.md`. Output: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`.

**Fast mode:** `✓ Step 7 — Visual diagram complete → [path]`

Update `_pipeline-state.md`.

### Step 8 — Design (Per Phase) [AUTO after mode selection]

Same as Personal Full Step 7. Ask in-repo vs v0; follow `ai-framework/03-design.md` or `subprompts/design-with-v0.md`. Output: design catalog at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`.

**Fast mode:** `✓ Step 8 — Design complete → [path]`

Update `_pipeline-state.md`.

### Step 9 — Update PRD from Designs [AUTO → GATE 2]

Same as Personal Full Step 8. Sync the PRD with design catalog: catalog reference, copy/flow changes, AC updates, decision log entries. Quality checks run before Gate 2 (see Personal Full Step 8 for the check list).

After Gate 2 approval, proceed to Step 10.

Update `_pipeline-state.md`. Mark Gate 2 as Approved with date.

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

Update `_pipeline-state.md` with the export results, including the Drive folder URL and Confluence page URL if they were enabled.

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

**Engineer picking up a feature:** Run `/feature-kickoff [feature-name]` for a role-specific briefing, then `/execute-plan`.

**Designer joining mid-pipeline:** Run `/feature-kickoff [feature-name]` with role = Designer, deliver in Figma, PM runs `/compare-figma-prd`.

**QA validating a phase:** Run `/validate-parallel` directly, or `/feature-kickoff` with role = QA.

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
  "pipeline": "Full | Medium | Light | Work",
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
    "validations": [
      { "phase": 1, "path": "string", "size_bytes": 0 }
    ],
    "jira_manifest": { "path": "string | null", "size_bytes": 0 },
    "learnings": [
      { "phase": 1, "path": "string", "size_bytes": 0 }
    ]
  },
  "export_urls": {
    "jira_epic": "string | null",
    "drive_folder": "string | null",
    "confluence_page": "string | null — PRD page, created at Gate 1 share or Step 11c",
    "confluence_breakdown_page": "string | null — user stories breakdown page, created at Gate 3 share",
    "pr_urls": [
      { "phase": 1, "pr_number": 1, "url": "string | null" }
    ]
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

Pipeline: [Full / Medium / Light / Work]
Mode: [Fast / Gated]
Feature: [Name]
Output folder: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/
Owner: [Name]

Documents created:
- Research: [path]
- Codebase Review: [path or N/A]
- PRD: [path]
- Product Review: [path]
- Technical Review: [path]
- System Design: [path or N/A]
- Visual Diagram: [path]
- Design Catalog(s): [paths per phase]
- User Stories Breakdown: [path]
- Validation Reports: [paths per phase]
- Learning Reports: [paths per phase]
- Jira tickets: [Epic URL or local fallback path]
- Jira manifest: [path]
- Changelog: [path if any change-mode runs occurred]

Phases completed: [list]
Phases remaining: [list or "none — product complete"]

Gates:
- Gate 1: [Approved YYYY-MM-DD]
- Gate 2: [Approved YYYY-MM-DD or N/A]
- Gate 3: [Approved YYYY-MM-DD or N/A]
━━━
```
