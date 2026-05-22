# Change Propagation Pipeline

Standalone pipeline triggered by typing `/change-mode` at any point after Gate 1. Does not replace or interrupt the build pipeline. Runs alongside it.

When the PM types `/change-mode`, load and follow this file regardless of where the build pipeline is currently paused.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 1 — Change Intake

Ask the PM to describe the change. Then ask where it came from. Present exactly seven trigger types:

1. **Dev feedback** — an engineer flagged something during build.
2. **Design change** — designs were updated after approval (e.g., a finalized design swapped out for a new version).
3. **Figma update** — a Figma file has been updated and tickets need to reflect it.
4. **Stakeholder input** — leadership or a stakeholder changed requirements.
5. **New user research** — new data or interviews changed the understanding of the problem.
6. **Scope change** — something is being added or cut.
7. **Designs arrived** — finalized designs are now available for stories previously written in DRAFT mode (v2.4.0+). Refreshes every story listed in `_pipeline-state.json` → `user_stories.draft_stories[]` and updates the corresponding Jira tickets in place.

### If trigger type is Figma update OR Designs arrived

Ask the PM to paste the Figma link or share the relevant Figma frames / design catalog file path. Confirm receipt before proceeding.

For **Designs arrived** specifically, before proceeding to Step 2:
- Read `_pipeline-state.json` → `user_stories.draft_stories[]` to know which stories need refresh. If the list is empty, tell the PM "No DRAFT stories on this feature — nothing to refresh. Did you mean 'Design change' instead?" and let them re-select.
- Read the new design catalog file(s) (`design/[feature]-phase-*-designs.md`) — the source the refresh will draw from.
- Present a summary: "Found [N] DRAFT stories across [N] Epics. I'll walk through each one with the new designs and refresh AC, sizing, UX state coverage, and the corresponding Jira ticket. Proceed? (yes / no)"

### For all other trigger types

Ask the PM to paste or describe the change in full detail. Confirm receipt before proceeding.

Do not proceed to Step 2 until trigger type and change detail are both confirmed.

---

## Step 2 — Impact Assessment (Blast Radius Report)

Before touching any artifact, produce a full blast radius report.

Read all existing artifacts for this feature from the output folder. For each artifact, assess whether it is affected and rate the impact.

Artifacts to evaluate:

- **PRD:** which sections are affected and how.
- **User stories:** which stories need to change, be added, or removed.
- **Feature flow diagram:** does the flow change.
- **Success metrics:** does the baseline or target change.
- **Design files and mockups:** which screens or flows are affected.
- **Jira tickets:** which specific tickets need to change. List by ticket ID and title.
- **Acceptance criteria:** which AC items need to change.
- **QA test scenarios:** which tests need to be updated or added.
- **Stakeholder and dependency list:** does this change affect who needs to know.
- **Release note draft:** does the user-facing description change.
- **Changelog:** needs a new entry regardless.

Rate each artifact:
- **HIGH** — significant rewrite required.
- **MEDIUM** — targeted edits required.
- **LOW** — minor update or reference change only.
- **NOT AFFECTED** — no change needed.

Present the full blast radius table to the PM before changing anything:

| Artifact | File Path | Impact | What Changes |
|----------|-----------|--------|--------------|
| ... | ... | HIGH / MEDIUM / LOW / NOT AFFECTED | ... |

---

## Step 3 — Scope Approval

Ask the PM to choose exactly one of three options:

1. **Update all affected artifacts.**
2. **Update HIGH and MEDIUM only, skip LOW.**
3. **Select specific artifacts manually** — PM names which ones.

Wait for explicit confirmation. Do not infer or assume. Do not proceed to Step 4 until the PM has selected an option.

---

## Step 4 — Propagation

Apply changes in strict dependency order:

1. PRD
2. User stories
3. Feature flow diagram
4. Success metrics
5. Design files and mockup descriptions
6. Jira tickets
7. Acceptance criteria
8. QA test scenarios
9. Stakeholder list
10. Release note and changelog

### For Figma update triggers

At the design artifacts step (step 5 of the propagation order), compare the Figma content provided in Step 1 against existing design descriptions and ticket acceptance criteria. List every specific difference. Apply only the differences the PM confirmed in Step 3.

### For Designs arrived triggers (DRAFT story refresh)

This trigger has a focused propagation that operates only on DRAFT stories. Skip the full propagation order. Instead:

1. **For each story in `_pipeline-state.json` → `user_stories.draft_stories[]`:**
   - Read the corresponding section of `user-stories/[feature]-user-stories.md`.
   - Read the matching screens/components from the new design catalog (e.g., the design for US-1.1 from `design/[feature]-phase-1-designs.md`).
   - Compose a refreshed version of the story:
     - Change `Status: ⚠ DRAFT — needs design` → remove (story is now complete).
     - Rewrite the Gherkin AC to include the full set of scenarios (happy path, negative, edge — including UX state coverage for FE: empty / loading / error / populated).
     - Update the sizing — remove the `*` suffix (e.g., `M*` → `M`) and adjust if the design reveals more or less complexity than estimated.
     - Update Testing Notes — fill in the UX state coverage row, refresh edge-case list, add design-specific data conditions.
     - Remove the "Known design gaps" section from the story.
2. **Show the refreshed version as a diff** in the standard Step 5 format. PM approves per-story.
3. **For each approved story**, update the corresponding Jira ticket via `updateJiraIssue`:
   - Update the **User Story** and **Acceptance criteria** custom fields with the refreshed ADF payloads.
   - Update the **labels** array — **remove `draft`**.
   - Update the **Description** — remove the "⚠ DRAFT — needs design refresh" note.
4. **Update `_pipeline-state.json`**: remove each refreshed story from `user_stories.draft_stories[]`. When the list is empty, also update `user_stories.mode` from `DRAFT` (or `MIXED`) to `full` and record the timestamp under `user_stories.draft_resolved_at`.
5. **Skip the broader artifact-propagation order** (PRD, system design, etc.) unless the PM explicitly says the designs imply scope changes — in that case, flag and ask if a separate `/change-mode` "Scope change" run is needed.

### Writing

Write each updated artifact back to its original file path. Do not create new files or rename files (the changelog and change-summary files in Step 6 are the only exceptions).

Before writing each artifact, print:

```
Writing: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/[subfolder]/[filename]
```

### Conflict handling

If a conflict is found between what the change requires and what an existing artifact says, surface the conflict explicitly. Do not resolve conflicts autonomously. Pause propagation and ask the PM how to resolve.

### Scope expansion check

If during propagation a change implies new scope (a feature, story, or component that did not exist before), stop and flag it explicitly. Ask whether to open a new research cycle. Do not silently expand scope.

---

## Step 5 — Diff Review

For every artifact updated, show a before and after comparison in this exact format:

```
ARTIFACT: [name and file path]
BEFORE: [original content of changed section only]
AFTER: [new content]
REASON: [one sentence explaining why this changed]
```

Present all diffs together as a single review block.

Ask the PM to choose:
- **Approve all** — finalize every change shown.
- **Approve individually** — go through each artifact and confirm yes / no.
- **Revert specific items** — name which to discard.

Nothing is finalized until the PM explicitly approves. If the PM reverts an item, restore the original content and remove any downstream changes that depended on it.

---

## Step 6 — Changelog and Summary

### Changelog entry

Write a changelog entry to:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/changelog/[feature-name]-changelog.md
```

This file is append-only. Never overwrite. If the file does not exist, create it with a header line and append the first entry.

Each entry must include exactly:
- Date and time.
- Trigger type and source.
- One-sentence description of what changed.
- List of every artifact updated with a one-line summary of what changed in each.
- Name of the PM who approved (ask if not known).

### Change summary

Write a per-change summary file to:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/changelog/[feature-name]-change-[YYYY-MM-DD]-summary.md
```

The summary must include:
- What changed (plain English).
- What was updated (list of artifacts).
- Current state of the feature (one paragraph).
- Any open questions created by this change that need resolution before build continues.

---

## Step 6.5 — Re-sync external mirrors (Drive + Confluence, if enabled)

If Google Drive was enabled for this feature when Step 11 originally ran, re-sync the changed files to Drive now. The Drive sync prompt (`ai-framework/07-drive-sync.md`) supports a surgical mode where it takes the list of changed paths and re-uploads only those — no full-folder walk.

If Confluence was enabled and any artifact was changed, re-publish the affected pages in place via `subprompts/publish-to-confluence.md` (uses per-file mtime detection to update only the changed child pages, preserving URLs — important for stakeholder bookmarks).

If neither was enabled originally, skip this step.

If either MCP is unavailable now, apply Error Type 4: notify the PM, preserve content locally, continue.

Report a final external-sync block in the propagation summary:

```
External mirrors updated:
- Drive: [N] files re-synced (Drive URL)
- Confluence: page updated (Confluence URL)
```

---

## Step 7 — Re-entry Offer

After completing, determine the earliest Build Mode step affected by the changes and offer to re-run the pipeline forward from that point.

Do not restart from Step 1 unless the research itself changed.

State exactly:

```
Change propagation complete.

Earliest affected Build Mode step: Step [N] — [step name].
Reason: [one sentence on why this step is the re-entry point].

Re-run the pipeline from Step [N]? (yes / no / specify a different step)
```

If change propagation reveals a gate needs to be re-approved, tell the PM which gate (Gate 1, Gate 2, or Gate 3) and why.

---

## Rules

- Never change an artifact silently. Every change must be shown as a diff and confirmed.
- Never add new features or expand scope as a side effect of propagating a change. If the change implies new scope, flag it explicitly and ask whether to open a new research cycle.
- Never skip the blast radius step even if the change seems small.
- If a conflict is found between what the change requires and what an existing artifact says, surface the conflict explicitly. Do not resolve conflicts autonomously.
- If change propagation reveals a gate needs to be re-approved, tell the PM which gate and why.
- Update the `_pipeline-state.md` file at the end of every step in this pipeline.
