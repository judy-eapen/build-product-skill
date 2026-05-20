# Build Product

Orchestrator command that guides you through the entire product development pipeline.

---

## Stage 0 — Project Type, Pipeline, and Mode Selection

Ask the user:

**"Personal or work project?"**

| Type | Behavior |
|------|----------|
| **Personal** | Full implementation. No user stories. AI can execute, validate, ship, learn. |
| **Work** | Planning and design only. No implementation. User stories go to Jira. Ends after design. Manual Figma handoff follows. |

Then:

**For Personal:** Ask pipeline and mode:

**"What are we building? Choose a pipeline:"**

| Pipeline | When to use | Steps |
|----------|------------|-------|
| **Full** | New product, greenfield app, major feature with new data model | Research -> PRD -> Review -> CTO Review -> Fix -> Design -> Execute -> Validate -> Ship -> Learn |
| **Medium** | New feature in existing app, significant scope | PRD -> Review and Fix -> Execute -> Validate -> Update PRD |
| **Light** | Bug fix, small UI change, config change | Execute (scope lock + implement) |

**For Work:** Single flow (steps 1-6b). Ask mode only.

**"Run in fast mode or gated mode?"**

| Mode | Behavior |
|------|----------|
| **Fast** (recommended) | AI chains all auto-steps without waiting. Pauses only at 3 real decision gates: PRD approval, design approval, and ship. |
| **Gated** | AI pauses after every step and waits for "continue." Use when you want to review each output before proceeding. |

Default to **fast mode** if the user does not specify a mode.

If the user describes their idea instead of choosing, infer and confirm: "This sounds like a [personal/work] project, [full/medium/light] pipeline in fast mode. Correct?"

Do not proceed until project type, pipeline (for Personal), and mode are confirmed.

---

## Output Patterns

### Fast mode: auto-step output (no pause)

After completing an auto-step, print one line and immediately move to the next step:

```
✓ [Step name] — [one-line summary] → [output file path, if any]
```

Do NOT wait for user input. Proceed immediately.

### Fast mode: approval gate (pause required)

Only used at the 3 gates. Output this block and wait:

```
━━━ APPROVAL NEEDED: [Gate Name] ━━━

What was produced:
[Bullet list of outputs with file paths]

Progress:
[x] Step 1 — [name]
[x] Step 2 — [name]
...
[ ] Step N — [name]  <-- next after approval

[Specific instruction for the user — e.g. "Review the PRD at ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[name].md.
Say 'approved' to continue, or give feedback to revise."]
━━━
```

Do NOT proceed until the user responds.

### Gated mode: step pause (used after every step)

```
━━━ Step [N] complete: [Step Name] ━━━

What was done: [1-2 sentence summary]
Output saved to: [file path, if applicable]

Progress:
[x] Step 1 — [name]
[x] Step 2 — [name]
[ ] Step 3 — [name]  <-- next
[ ] Step 4 — [name]
...

Next step: [Step N+1 name] — [what it does]

Say "continue" to proceed, or provide additional input.
━━━
```

Do NOT proceed until the user responds.

---

## Personal — Full Pipeline

When the user selects Personal and "Full," execute these steps. Each step is marked **AUTO** (chains without pause in fast mode) or **GATE** (always pauses for human decision).

### Step 1 — Research Idea [AUTO after initial clarification]

Read and follow: `ai-framework/01-research-idea.md`

Ask the user to describe their idea. Run Stage 1 (idea clarification questions) — this requires user input. After the user answers, run Stages 2–5 (market scan, strategic evaluation, 10x test, decision gate) automatically and proceed to PRD creation without pausing.

**Fast mode:** Print `✓ Research complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/research/[name]-clarity.md` and proceed.
**Gated mode:** Pause after research. Show progress. Next step: Create PRD.

### Step 2 — Create PRD [AUTO]

Read and follow: `ai-framework/02-create-prd.md`

Use the research output from Step 1 as input. Run the full PRD generation process (clarification, generation, validation). Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[name].md`.

**Fast mode:** Print `✓ PRD created → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[name].md` and proceed to Step 3.
**Gated mode:** Pause. Show progress. Next step: Review PRD.

### Step 3 — Review PRD (Product Lens) [AUTO]

Read and follow: `subprompts/review-prd.md`

Review the PRD. Produce the full review covering document quality and product quality. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[name].md`.

**Fast mode:** Print `✓ Product review complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[name].md` and proceed to Step 4.
**Gated mode:** Pause. Show progress. Next step: CTO Review.

### Step 4 — CTO Review (Technical Lens) [AUTO]

Read and follow: `subprompts/cto-review.md`

Review the PRD from a technical lens. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[name].md`.

**Fast mode:** Print `✓ CTO review complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[name].md` and proceed to Step 5.
**Gated mode:** Pause. Show progress. Next step: Apply Fixes.

### Step 5 — Apply Fixes [AUTO → GATE 1]

Read both review files from Steps 3 and 4. Apply all findings to the PRD:

- Product review: update user stories, add empty states, fix UX gaps, add missing acceptance criteria.
- CTO review: fix data model problems, update API contracts, address security concerns, correct architectural decisions.

Update the PRD file in place. Update the decision log.

**Both modes:** After applying fixes, show **GATE 1**:

```
━━━ APPROVAL NEEDED: PRD ━━━

What was produced:
- PRD (reviewed + fixed): ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[name].md
- Product review: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[name].md
- CTO review: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[name].md
- Changes applied: [N] fixes from reviews

Progress:
[x] Research
[x] PRD
[x] Product review
[x] CTO review
[x] Apply fixes
[ ] Design Phase 1  <-- next after approval

Review the PRD at ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[name].md.

Also answer one optional question: Complex architecture needing a system design doc? (yes/no — use yes for multiple external integrations or multi-service dependencies)

Say "approved" (with any answer) to continue, or give feedback to revise the PRD.
━━━
```

### Step 5c — System Design (Optional) [AUTO if yes]

Only if the user said yes to complex architecture at Gate 1.

Read and follow: `subprompts/system-design.md`. Generate a system design document.

**Fast mode:** Print `✓ System design complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[name]-system-design.md` and proceed.
**Gated mode:** Pause. Show progress. Next step: Design Phase 1.

### Step 6 — Design (Per Phase) [AUTO after mode selection]

Ask the user (at Gate 1 approval or at the start of this step if returning from a phase loop):

**"How do you want to design this phase — in-repo (AI builds screens in code) or v0 (AI generates prompts, you use v0)?"**

Then read and follow the appropriate command:
- In-repo: `subprompts/design.md`
- v0: `subprompts/design-with-v0.md`

If this is the first time reaching Step 6, design Phase 1. If returning from the phase loop, design the next phase. After Phase 1, also create the design tokens file.

**Fast mode:** Print `✓ Design complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[app]-phase-[N]-designs.md` and proceed to Step 6b.
**Gated mode:** Pause. Show progress. Next step: Update PRD from designs.

### Step 6b — Update PRD from Designs (Per Phase) [AUTO → GATE 2]

Read and follow: `subprompts/update-prd-from-designs.md`

Sync the PRD with the finalized design catalog.

**Both modes:** After syncing, show **GATE 2**:

```
━━━ APPROVAL NEEDED: Designs ━━━

What was produced:
- Design catalog: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[app]-phase-[N]-designs.md
- PRD updated to reflect designs

Progress:
[x] Research
[x] PRD (reviewed + fixed)
[x] Phase [N] designs
[x] PRD synced with designs
[ ] Execute Phase [N]  <-- next after approval

Review the designs:
[Instructions based on design method — e.g. "Run the app and navigate to [routes]" or "Review the v0 outputs in the design catalog"]

Say "approved" to begin implementation, or give feedback to revise the designs.
━━━
```

### Step 7 — Execute (Per Phase) [AUTO]

Read and follow: `subprompts/execute-plan.md`

Execute the current phase in solo mode. The design catalog from Step 6 is the visual reference.

**Fast mode:** Print `✓ Phase [N] implementation complete` and proceed to Step 8.
**Gated mode:** Pause. Show progress. Next step: Validate.

### Step 8 — Validate (Per Phase) [AUTO]

Read and follow: `subprompts/validate.md`

Validate the phase against PRD acceptance criteria and designs. Save report to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/[app]-phase-[N]-validation.md`.

**Fast mode:** Print `✓ Validation complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/[app]-phase-[N]-validation.md` and proceed to Step 8b. If validation finds failures (not just notes), print `⚠ Validation found [N] failures — fixing before proceeding` and resolve them before continuing.
**Gated mode:** Pause. Show progress. Next step: Update PRD from build.

### Step 8b — Update PRD from Build (Per Phase) [AUTO → GATE 3]

Read and follow: `subprompts/update-prd-from-build.md`

Sync the PRD with what was actually implemented.

**Both modes:** After syncing, show **GATE 3**:

```
━━━ APPROVAL NEEDED: Ship ━━━

Phase [N] is built and validated.

What was produced:
- Implementation: Phase [N] complete
- Validation: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/[app]-phase-[N]-validation.md ([pass/fail summary])
- PRD synced with build

Progress:
[x] Phase [N] design
[x] Phase [N] execute
[x] Phase [N] validate
[x] PRD synced
[ ] Ship Phase [N]  <-- you are here
[ ] Learn

Deployment:
[Guidance based on tech stack — e.g. "Run `vercel deploy` or push to main"]

Say "shipped" when deployed, or "skip ship" to go straight to the learning report.
━━━
```

### Step 10 — Learn (Per Phase) [AUTO after "shipped"]

Read and follow: `subprompts/learn.md`

Collect feedback and produce the learning report. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/learning/[app]-phase-[N]-learning.md`.

**Both modes:** After the learning report is complete, ask:

```
━━━ Phase [N] complete ━━━

Learning report: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/learning/[app]-phase-[N]-learning.md

What's next?
- "continue" — Start Phase [N+1] (loops back to Step 6 — design)
- "done" — End the pipeline. Show final summary.
- "revise PRD" — Loop back to update the PRD based on learnings, then continue.
━━━
```

---

## Personal — Medium Pipeline

When the user selects Personal and "Medium," execute these steps:

### Step 1 — Create PRD [AUTO after clarification]

Read and follow: `ai-framework/02-create-prd.md`. Same as Full Pipeline Step 2.

**Fast mode:** Proceed directly to Step 2 after PRD is saved.
**Gated mode:** Pause. Next step: Review and Fix.

### Step 2 — Review and Fix [AUTO → GATE]

Read and follow: `subprompts/review-and-fix.md`

Runs product review + CTO review + applies fixes in one pass.

**Both modes:** After fixes are applied, show the same GATE 1 approval block as the Full Pipeline (PRD approval gate). After approval, proceed to Execute.

### Step 3 — Execute [AUTO]

Same as Full Pipeline Step 7. Read `subprompts/execute-plan.md`.

**Fast mode:** Print `✓ Implementation complete` and proceed to Step 4.
**Gated mode:** Pause. Next step: Validate.

### Step 4 — Validate [AUTO]

Same as Full Pipeline Step 8. Read `subprompts/validate.md`.

**Fast mode:** Print `✓ Validation complete` and proceed to Step 5.
**Gated mode:** Pause. Next step: Update PRD from Build.

### Step 5 — Update PRD from Build [AUTO → end]

Same as Full Pipeline Step 8b. Read `subprompts/update-prd-from-build.md`.

After updating the PRD, show final progress and end. No design step, no ship/learn loop.

---

## Personal — Light Pipeline

When the user selects Personal and "Light," execute one step:

### Step 1 — Execute

Read and follow: `subprompts/execute-plan.md`

Ask the user to describe the bug fix or small change. Lock scope and implement.

When complete, show final progress and end. No PRD, no design, no review.

---

## Work Pipeline

When the user selects "Work," execute steps 1-6b. No implementation. User stories go to Jira. Ends after design approval.

### Steps 1-5 — Same as Full Pipeline

Research, Create PRD, Product Review, CTO Review, Apply Fixes. Same commands and output paths.

### Gate 1 — PRD Approval

Same structure as Full Pipeline Gate 1. Ask:
1. Import user stories into Jira? (yes/no)
2. Complex architecture needing a system design doc? (yes/no)

### Step 5b — Jira Import (Optional) [AUTO if yes]

Only if the user said yes to Jira import at Gate 1.

Read and follow: `subprompts/prd-to-jira.md`. Create Jira stories from the PRD.

**Fast mode:** Print `✓ Jira import complete` and proceed.
**Gated mode:** Pause. Show progress.

### Step 5c — System Design (Optional) [AUTO if yes]

Same as Full Pipeline. Read `subprompts/system-design.md`.

### Steps 6, 6b — Design and Update PRD from Designs

Same as Full Pipeline. Read `subprompts/design.md` or `subprompts/design-with-v0.md`, then `subprompts/update-prd-from-designs.md`.

### Gate 2 — Design Approval (END)

After syncing PRD from designs, show Gate 2. Then show this block and **end** (no Execute, Validate, Ship, Learn):

```
━━━ Work Pipeline Complete ━━━

Planning and design phase done. No implementation in this pipeline.

What was produced:
- PRD: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[name].md
- Jira user stories: [project/Epic link or ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[slug]-jira-stories.md]
- Design catalog: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[app]-phase-[N]-designs.md

Next step (now or later):
When the designer has created their version in Figma, run `/compare-figma-prd` to compare Figma to the PRD and Jira, then update both.

You can run `/project-status` anytime to see where you left off. When you see "Run /compare-figma-prd", run that command to complete the sync.

When ready to implement, use `/execute-plan` with the updated PRD and designs.
━━━
```

---

## Final Summary

When the pipeline ends (user says "done" or pipeline completes), output:

**For Personal pipelines:**
```
━━━ Pipeline Complete ━━━

Pipeline: [Full / Medium / Light]
Mode: [Fast / Gated]
Product: [Name from PRD or description]

Documents created:
- PRD: [path]
- Product Review: [path]
- CTO Review: [path]
- System Design: [path or N/A]
- Design Catalog: [paths per phase]
- Validation Reports: [paths per phase]
- Learning Reports: [paths per phase]

Phases completed: [list]

[Any remaining phases or next steps]
━━━
```

**For Work pipeline:** The Work Pipeline Complete block (above) serves as the final summary. No validation or learning reports.

---

## Rules

- **Fast mode:** Chain all AUTO steps without pausing. Only stop at the 3 approval gates (PRD, designs, ship). Print a one-line `✓` update after each auto-step so the user can see progress.
- **Gated mode:** Pause after every step. Never auto-advance.
- **Both modes:** Always read the referenced command or framework file at each step. Do not shortcut or summarize the process.
- **Validation failures:** In fast mode, if validation finds failures (not just suggestions), fix them before proceeding to Gate 3. Print `⚠ [N] failures found — resolving` and resolve inline.
- **Track state across the conversation.** Write `_pipeline-state.json` at the end of every step per the schema in `SKILL.md`. Write `_context-checkpoint.md` after every gate approval.
- **If the conversation is interrupted** (new session), read `_pipeline-state.json`, run integrity verification, read `_context-checkpoint.md`, and offer to resume. The user can also run `/project-status` to see where they left off.
