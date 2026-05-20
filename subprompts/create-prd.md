# Create PRD

Read and follow `ai-framework/02-create-prd.md`. Start with clarification questions, then generate the PRD.

## Save location

- `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`
- **Check first:** Look for an existing PRD for the same feature in `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/` before creating. If one exists, ask whether to update it or create a new one.

## Tech stack

The skill does not assume a default tech stack. At intake (see Intake Parameters in `CLAUDE.md`) the PM either names their stack, says "I don't know", or describes the product type — and the skill proposes defaults appropriate to that product type only at that point. Do not propose defaults in this file.

## PRD structure

The framework (`ai-framework/02-create-prd.md`) defines the PRD sections. **Include Section 9 — Testing Notes** in every PRD: positive cases (happy path, valid inputs), negative cases (invalid input, auth failures, validation errors), and edge cases (empty state, first use, limits, offline). Make each case testable (preconditions, action, expected result).

## Quality bar

Match the level of detail your team expects for a PRD. If you have prior PRDs in `~/Desktop/Resources/PDLC Workflow Docs/`, open one as a reference for tone, structure, and depth.
