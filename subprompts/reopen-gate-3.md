# Reopen Gate 3

Re-open the **User Stories Breakdown approval gate** for an existing feature.

Use this when:
- After approving Gate 3, the breakdown turns out to have missing stories, incorrect FE/BE pairing, or thin acceptance criteria (Gherkin or plain English).
- Tickets have been created in Jira but the underlying breakdown was wrong, so the tickets are wrong.
- The PM wants to revise sizing, sequencing, or testing notes before the engineering team starts work.

Read and follow Error Type 1 in `ai-framework/error-handling.md`.

## What this command does

1. **Identify the feature.** Ask the PM which feature folder to re-open Gate 3 for. Confirm the path: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/`.
2. **Flag downstream artifacts as DRAFT.** Every file produced after the original Gate 3 approval gets a `STATUS: DRAFT — Gate 3 reopened on [date]` header line added to the top:
   - Jira export file (the fallback / local copy)
   - Jira manifest file
   - Any in-flight tickets already created in Jira (do not delete — flag them in description as "breakdown reopened on [date]; review and update")
3. **Re-run Step 10** (User Stories Breakdown) using the existing breakdown as starting context:
   - Read the existing breakdown file.
   - Ask the PM what specifically needs to change (missing story? wrong AC? wrong sizing? wrong FE/BE split?).
   - Update only the affected stories. Do not regenerate the entire breakdown from scratch.
   - Re-run the 8 quality checks.
4. **Re-present Gate 3** with the updated breakdown and quality-check report.
5. **On approval**: update the pipeline state file. The Jira export step (Step 11) needs to be re-run to apply the updated breakdown to actual tickets. Offer to run it now or wait.

## What it will NOT do

- It will not auto-delete or auto-update Jira tickets. If tickets already exist, the prompt:
  - Lists which tickets correspond to changed stories (using the manifest file).
  - Asks the PM whether to update them via `/change-mode` (preferred — diff-based, safe) or via direct Jira edits.
  - Never bulk-recreates tickets without explicit PM approval.
- It will not regenerate every story from scratch. Surgical updates only.

## Related commands

- `/reopen-gate-1` — re-open the PRD approval gate (for fundamental PRD issues that need to flow through everything).
- `/reopen-gate-2` — re-open the Design approval gate.
- `/change-mode` — propagate a smaller change across Jira tickets after the breakdown has been updated.
