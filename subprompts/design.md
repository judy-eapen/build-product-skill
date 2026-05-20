# Design (In-Repo)

Read and follow `ai-framework/03-design.md`. The AI implements screens and components directly in the project codebase; you run the app, review, and request changes.

For v0-based design (AI generates prompts, you use v0), use `/design-with-v0` instead.

## Workspace defaults

- **Design catalog save location:** `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`
- **Design tokens save location:** `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-design-tokens.md` (after Phase 1)

## Post-design reminder

After the design catalog is complete and the user approves, remind them:

- Run `/update-prd-from-designs` to sync the PRD with the finalized designs before execution.
- Then run `/execute-plan` to implement backend, API, and data wiring.
