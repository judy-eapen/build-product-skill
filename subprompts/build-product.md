# Build Product

Orchestrator command that guides you through the Work pipeline: Research → Codebase Review → PRD → Dual Review → Apply Fixes → Gate 1 → System Design → Visual Diagram → Design → Update PRD from Designs → Gate 2 → User Stories Breakdown → Gate 3 → Export (Jira + Drive + Confluence).

---

## Stage 0 — Mode Selection

Ask the PM: **"Run in fast mode or gated mode?"**

| Mode | Behavior |
|------|----------|
| **Fast** (default) | AI chains all auto-steps without waiting. Pauses only at the 3 approval gates: PRD, Designs, and User Stories Breakdown. |
| **Gated** | AI pauses after every step and waits for "continue." Use when you want to review each output before proceeding. |

Default to **fast mode** if the user does not specify.

If the user describes their idea instead of choosing, infer and confirm: "This sounds like a work pipeline feature in fast mode. Correct?"

Do not proceed until mode is confirmed.

---

## Stage 0.5 — Intake

Before any pipeline step, walk through the 7 intake questions defined in `CLAUDE.md` → "Intake Parameters". Ask them in order. Persist every answer to `_pipeline-state.json` under an `intake` object — this is the source of truth that downstream steps (PRD generation, user stories, Jira export) will read.

**Question 3 is open-ended and especially important** — it captures every per-ticket convention the team applies. Do not just ask for labels. Probe with examples so the PM thinks about all of:

- **Labels** they always apply (pod tags, area tags, team names)
- **Title format** (verb-first? prefix for BE/FE? Epic naming convention?)
- **BE/FE split** (separate tickets per layer, or combined?)
- **Default values for custom fields** (e.g., "Testable" = Yes/No on every ticket)
- **Fields they intentionally leave blank** (e.g., Story Points)
- **Link conventions** (e.g., BE↔FE pairs as "Relates to"; "Blocked by" with a note)

Present the examples as a list in your question, so the PM can scan them and add anything else specific to their team. Accept the answer as free-text — bullets, prose, or "we don't have specific conventions yet" are all valid. Do not try to parse the answer into rigid fields; store it as a single string and let downstream steps interpret it at the right moment.

Persist the full intake to `_pipeline-state.json`:

```json
{
  "intake": {
    "feature_name": "...",
    "jira_project": "...",
    "jira_ticket_conventions": "<verbatim free-text from PM>",
    "tech_stack": "...",
    "product_type": "...",
    "permission_model": "yes | no | not_yet_decided",
    "backend_api_surface": "yes | no | not_yet_decided"
  }
}
```

If the PM has run this skill before in the same workspace, look for a prior `_pipeline-state.json` (any feature folder) and offer: "I see you ran this for [other-feature] last time. Reuse the same Jira project, label conventions, and tech stack? (yes / show me the values first / start fresh)". Do not assume — confirm reuse explicitly.

Do not proceed to Step 1 until all 7 answers are captured.

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

━━━ QUALITY CHECK ━━━
[list of flags with severity: WARNING or INFO, or "Quality check passed. No issues found."]
━━━ [N] flags found. You can approve anyway or address these first. ━━━

Progress:
[x] Step 1 — [name]
[x] Step 2 — [name]
...
[ ] Step N — [name]  ← next after approval

[Specific instruction for the user]

Options:
- Say "approved" to continue with no open items.
- Say "approved with conditions: [list]" to advance and resolve these before the next gate.
- Give feedback to revise before advancing.
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
[ ] Step 3 — [name]  ← next
[ ] Step 4 — [name]
...

Next step: [Step N+1 name] — [what it does]

Say "continue" to proceed, or provide additional input.
━━━
```

Do NOT proceed until the user responds.

---

## Work Pipeline

### Step 1 — Research Idea [AUTO after initial clarification]

Read and follow: `ai-framework/01-research-idea.md`

Ask the PM to describe their idea. Run Stage 1 (idea clarification questions) — this requires user input. After the PM answers, run Stages 2–5 automatically and proceed without pausing.

**Fast mode:** `✓ Step 1 — Research complete → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/research/[feature-name]-research.md`
**Gated mode:** Pause. Next step: Step 2 — Codebase Review.

Update `_pipeline-state.json`.

---

### Step 2 — Codebase Review [AUTO]

Read and follow: `ai-framework/00-codebase-review.md`

Output: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/codebase-review/[feature-name]-codebase-review.md`

Pass the one-paragraph handoff summary to Step 3 (Create PRD) as context.
Pass the full codebase review output file path to Step 4b (Technical Review) as an input alongside the PRD.

**Fast mode:** `✓ Step 2 — Codebase review complete → [path]`
**Gated mode:** Pause. Next step: Step 3 — Create PRD.

Update `_pipeline-state.json`.

---

### Step 3 — Create PRD [AUTO]

Read and follow: `ai-framework/02-create-prd.md`

Use both the research output (Step 1) and the codebase review handoff summary (Step 2) as context. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`.

**Fast mode:** `✓ Step 3 — PRD created → [path]`
**Gated mode:** Pause. Next step: Step 4 — Dual Review.

Update `_pipeline-state.json`.

---

### Steps 4a + 4b — Dual Review [PARALLEL AUTO]

Read `ai-framework/05-parallel-rules.md` — Block 1.

Spawn two agents at the same time:

**Step 4a — Product Review:**
Apply the Product Reviewer persona from `ai-framework/personas.md`. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[feature-name]-product-review.md`.

**Step 4b — Technical Review:**
Apply the Technical Reviewer persona from `ai-framework/personas.md`. Inputs: both the PRD and the codebase review output. Evaluate them together — assess whether the PRD is consistent with the codebase review findings. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-technical-review.md`.

Wait for both to complete. Synthesize: identify agreements, conflicts, and single-source findings. Conflicts → structured conflict cards per `error-handling.md` Error Type 2.

**Fast mode:** `✓ Steps 4a + 4b — Dual review complete (parallel) → product-review + technical-review | [N] agreements | [N] conflicts`
**Gated mode:** Pause. Next step: Step 5 — Apply Fixes.

Update `_pipeline-state.json`.

---

### Step 5 — Apply Fixes [AUTO → GATE 1]

Read both review files. Apply all findings to the PRD:

- Agreements: apply immediately.
- Single-source findings: apply with a decision log note.
- Conflicts: present at Gate 1 as judgment calls.

From product review: update user stories, add empty/error/loading states, fix UX gaps, add missing AC.
From technical review: fix data model problems, update API contracts, address security concerns.

Update the PRD in place. Add entries to the decision log.

Self-check (Error Type 3 from `error-handling.md`): does this output contradict any prior PRD decision? If yes, flag to PM before writing.

**Gate 1 — Quality Check (run automatically before presenting):**

- Is the problem statement specific enough that two engineers would build the same thing? If not, flag it.
- Does every user story have a measurable outcome? Flag those that do not.
- Did the Technical Review surface any HIGH risks not resolved in the decision log? If so, list them.
- Is the success metric specific and measurable with a baseline? If vague, flag with a suggestion.
- Is the codebase review handoff note reflected in the PRD? If the codebase review flagged a HIGH risk and the PRD does not address it, flag it.

Format:
```
━━━ QUALITY CHECK ━━━
[list of flags with severity: WARNING or INFO]
━━━ [N] flags found. You can approve anyway or address these first. ━━━
```

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
[List each conflict card]

Progress:
[x] Step 1 — Research
[x] Step 2 — Codebase review
[x] Step 3 — PRD
[x] Steps 4a + 4b — Dual review (parallel)
[x] Step 5 — Apply fixes
[ ] Step 6 — System design (optional)  ← next after approval
[ ] Step 7 — Visual diagram
[ ] Step 8 — Design

Also answer (optional):
- Complex architecture needing a system design doc? (yes/no)

Options:
- Say "approved" to continue with no open items.
- Say "approved with conditions: [list]" to advance and resolve these before Gate 2.
- Give feedback to revise the PRD before advancing.
━━━
```

**If "approved with conditions":** Write to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_open-conditions.md`.

Update `_pipeline-state.json`. Mark Gate 1 Approved with date. Write context checkpoint to `_context-checkpoint.md`.

---

### Step 6 — System Design [AUTO if yes at Gate 1]

Read and follow: `subprompts/system-design.md`

Skip entirely if PM said no at Gate 1.

**Fast mode:** `✓ Step 6 — System design complete → [path]` (or `✓ Step 6 — Skipped (no system design needed)`)

Update `_pipeline-state.json`.

---

### Step 7 — Visual Diagram [AUTO]

Read and follow: `ai-framework/03c-visual-diagram.md`

Output: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`

**Fast mode:** `✓ Step 7 — Visual diagram complete → [path]`
**Gated mode:** Pause. Next step: Step 8 — Design.

Update `_pipeline-state.json`.

---

### Step 8 — Design (Per Phase) [AUTO after tool selection]

Ask the PM: **"v0 or Figma Make?"** — both use the same prompt format. Figma Make can additionally reference your team's Figma design system components directly.

Read and follow: `subprompts/design-prompts.md`

Output: design catalog at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`

**Fast mode:** `✓ Step 8 — Design complete → [path]`
**Gated mode:** Pause. Next step: Step 9 — Update PRD from Designs.

Update `_pipeline-state.json`.

---

### Step 9 — Update PRD from Designs [AUTO → GATE 2]

Read and follow: `ai-framework/03b-update-prd-from-designs.md`

Sync the PRD with the finalized design catalog: design catalog reference, copy/flow changes, AC updates, decision log entries. Overwrite the PRD in place.

Self-check (Error Type 3): does this output contradict any prior PRD decision? If yes, flag to PM before writing.

**Gate 2 — Quality Check (run automatically before presenting):**

- Does the visual diagram cover every user story approved at Gate 1? Flag any stories with no corresponding flow.
- Do the design prompts cover all states: empty state, loading state, error state? Flag any missing states.
- **Open conditions from Gate 1:** Verify each condition is resolved. Flag any that remain unresolved as WARNING.

**Both modes — show Gate 2:**

```
━━━ APPROVAL NEEDED: Gate 2 — Designs ━━━

What was produced:
- Design catalog: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md
- PRD updated to reflect designs

[QUALITY CHECK block]

Open conditions from Gate 1: [N resolved / N unresolved — list any unresolved]

Progress:
[x] Steps 1–5 — PRD (reviewed + fixed)
[x] Step 6 — System design (if applicable)
[x] Step 7 — Visual diagram
[x] Step 8 — Phase [N] design
[x] Step 9 — PRD synced with designs
[ ] Step 10 — User Stories Breakdown  ← next after approval

Options:
- Say "approved" to continue with no open items.
- Say "approved with conditions: [list]" to advance and resolve these before Gate 3.
- Give feedback to revise designs before advancing.
━━━
```

**If "approved with conditions":** Append to `_open-conditions.md`.

Update `_pipeline-state.json`. Mark Gate 2 Approved with date. Write context checkpoint to `_context-checkpoint.md`.

---

### Step 10 — User Stories Breakdown [AUTO → GATE 3]

Read and follow: `ai-framework/06-user-stories.md`

Inputs: the final design-informed PRD (post-Gate 2) + the design catalog + the codebase review.

Output: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md`

**Gate 3 — Quality Check (run automatically before presenting):**

- Every PRD user story appears in the breakdown (no drops).
- Every story has a unique US-ID.
- Every story has at least 2 Gherkin scenarios.
- Every story has at least one edge-case or error-state scenario.
- Every linked FE/BE pair has both sides present.
- No story sized larger than L without a proposed split.
- HIGH risks from the codebase review appear in at least one story's testing notes.
- UX state coverage per FE story: empty / loading / error / populated.
- **Open conditions from Gate 2:** Verify each condition is resolved. Flag any unresolved as WARNING.

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
[ ] Step 11 — Export  ← runs after approval

Review the breakdown and the Build Sequence Map. Once approved, tickets will be created in Jira and cannot be easily undone.

Say "approved" to create tickets, or give feedback to revise.
━━━
```

Update `_pipeline-state.json`. Mark Gate 3 Approved with date.

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

---

### Step 11 — Export (parallel: Jira always + Drive optional + Confluence optional)

Run a **pre-flight question** before the parallel block:

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

Collect answers before spawning the parallel block.

Read `ai-framework/05-parallel-rules.md` — Block 3. Spawn up to three agents simultaneously.

**Step 11a — Jira Export (always runs)**
Read and follow: `subprompts/prd-to-jira.md`

**Step 11b — Google Drive Sync (optional)**
Only if enabled. Read and follow: `ai-framework/07-drive-sync.md`

**Step 11c — Confluence Publish (optional)**
Only if enabled. Read and follow: `subprompts/prd-to-confluence.md`

If a Confluence page URL already exists in `_pipeline-state.json` → `export_urls.confluence_page`, call `updateConfluencePage` instead of creating a new one.

After all agents finish:

```
✓ Step 11 — Export complete
  → Jira: [Epic URL] · [N] tickets created
  → Drive: [folder URL] · [N] files synced  (if enabled)
  → Confluence: [page URL]                   (if enabled)
```

Update `_pipeline-state.json` with export results.

---

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

## Rules

- **Fast mode:** Chain all AUTO steps without pausing. Only stop at the 3 approval gates. Print a one-line `✓` update after each auto-step.
- **Gated mode:** Pause after every step. Never auto-advance.
- **Both modes:** Always read the referenced command or framework file at each step. Do not shortcut or summarize the process.
- **Track state across the conversation.** Write `_pipeline-state.json` at the end of every step per the schema in `SKILL.md`. Write `_context-checkpoint.md` after every gate approval.
- **If the conversation is interrupted** (new session), read `_pipeline-state.json`, run integrity verification, read `_context-checkpoint.md`, and offer to resume.
