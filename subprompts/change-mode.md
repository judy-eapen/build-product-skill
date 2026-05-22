# Change Mode

Propagate a change to an existing feature across every artifact, with a blast-radius report and diff-by-diff approval before anything updates.

Use this when something about an in-flight or shipped feature changes after Gate 1:
- Dev feedback during build
- Design update after Gate 2
- Figma file updated
- Stakeholder input changes requirements
- New user research reveals something
- Scope addition or cut
- **Designs arrived** for stories previously written in DRAFT mode (v2.4.0+) — refreshes every story listed in `user_stories.draft_stories[]` and updates the corresponding Jira tickets in place

Read and follow `ai-framework/05-change-propagation.md`.

## What this command does

1. **Change intake** — asks the PM to describe the change and pick one of seven trigger types.
2. **Impact assessment** — reads every artifact for the feature and produces a blast-radius report. Each artifact is rated HIGH / MEDIUM / LOW / NOT AFFECTED.
3. **Scope approval** — PM picks: update everything affected, update HIGH + MEDIUM only, or select specific artifacts.
4. **Propagation in dependency order** — PRD → user stories → diagram → success metrics → designs → Jira tickets → AC → QA scenarios → stakeholders → changelog. Every change shown as a before/after diff.
5. **Diff review** — PM approves all, approves individually, or reverts specific items.
6. **Changelog + summary** — appended to `changelog/[feature]-changelog.md`. Plain-English summary written to `changelog/[feature]-change-[date]-summary.md`.
7. **Re-entry offer** — determines the earliest Build Mode step affected and offers to re-run the pipeline forward from that point.

## Inputs

The change-propagation prompt asks the PM for:

- **Which feature** are we changing? Asks for the feature name if not in conversation context.
- **What is the change?** Describe in detail or paste reference (e.g., a Figma link if it's a design update).
- **Trigger type** — pick one of seven.

For Figma updates and Designs-arrived triggers, the PM is asked to paste the Figma link / design catalog file path before the impact assessment runs. The Designs-arrived trigger uses a focused propagation that only touches DRAFT stories — see `ai-framework/05-change-propagation.md` Step 4 for the specific flow.

## What it will NOT do

- It will not silently update any artifact. Every change is shown as a diff and confirmed before being applied.
- It will not expand scope as a side effect. If the change implies new scope, the prompt surfaces that explicitly and asks whether to open a new research cycle.
- It will not skip the blast radius step even if the change seems small.
- It will not auto-resolve conflicts between the change and existing artifacts. Conflicts are surfaced for PM decision.

## Where the changelog lives

`~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/changelog/`
- `[feature-name]-changelog.md` — append-only history of every change-mode run.
- `[feature-name]-change-[YYYY-MM-DD]-summary.md` — plain-English summary of this specific run.

## Related commands

- `/reopen-gate-1`, `/reopen-gate-2`, `/reopen-gate-3` — re-open a specific gate when an entire phase's artifacts need to be re-done, not just patched.
- `/build-product` — start a new feature from scratch (does not affect existing features).
