# User Stories Breakdown

Generate a standalone user-stories breakdown document from an existing PRD. Use when:
- You have an approved PRD and want the exhaustive Gherkin AC + FE/BE pairing without running the full pipeline.
- You want a sprint-planning-ready doc you can read aloud to the team.
- You want to validate the user-story coverage before deciding to create Jira tickets.

Read and follow `ai-framework/06-user-stories.md`.

## Inputs

The underlying prompt collects inputs as Step 0:

**Required:**
- A PRD with user stories. Paste it or give the file path.

**Optional (each improves the breakdown — skill warns about limitations if missing):**
- **Design catalog** — informs UX state coverage per FE story (empty / loading / error / populated). Without it, state coverage is inferred from the PRD only.
- **Codebase review** — feeds HIGH-risk traceability into Testing Notes per story. Without it, no risk-to-story linkage in the testing notes.
- **Visual diagram** — feeds screen-to-story mapping. Without it, mapping is inferred from the PRD only.

The skill will ask which you have. Provide whatever you've got.

No prior pipeline run is required. The skill works with just a PRD, but each optional input materially improves the output.

## Output

`~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md`

Document contents:
- **Build Sequence Map** — every story with type (FE / BE / both), phase, depends-on, related-to, size.
- **Per-story sections** — As-a / I-want / So-that narrative + exhaustive Gherkin AC (happy + negative + edge + error scenarios) + testing notes (coverage areas, cross-boundary verification, data conditions, HIGH risks).
- **Build-order summary paragraph** — sprint-planning-ready text.
- **Limitations header** (if any optional inputs were missing) — explicit note about what couldn't be done.

## Quality checks (run automatically)

Eight checks before declaring the breakdown ready:
1. Every PRD user story appears in the breakdown (no drops).
2. Every story has a unique US-ID.
3. Every story has at least 2 Gherkin scenarios.
4. Every story has at least one edge-case or error-state scenario.
5. Every linked FE/BE pair has both sides present.
6. No story sized larger than L without a proposed split.
7. HIGH risks from codebase review appear in at least one story's testing notes (skipped if no codebase review).
8. UX state coverage per FE story: empty / loading / error / populated.

Any failed check is surfaced as a WARNING. You can fix or proceed anyway.

## When to use the full pipeline instead

If you want the breakdown to be the input to live Jira ticket creation (with a Gate 3 approval moment and an auto-written US-ID → Jira-key manifest), run `/build-product` and pick the Work pipeline. The breakdown there flows into Step 11 (Jira Export) automatically and the manifest is written for future `/change-mode` updates.
