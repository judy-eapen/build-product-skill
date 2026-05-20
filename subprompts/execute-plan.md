# Execute Plan

Read and follow `ai-framework/04-execute-plan.md`.

## Workspace defaults

- **Default mode:** Solo mode (unless the user says "team mode" or this is a team/production project).
- When the framework asks "Is this a solo/learning project or a team/production project?" -- default to solo and confirm: "Defaulting to solo mode. Say 'team mode' if this is a team/production project."

## Design catalog check

Before starting any frontend tasks, check for a design catalog:

- Look for `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`
- If one exists, use it as the visual reference for all frontend implementation.
- If one does not exist, ask: "No design catalog found for this phase. Proceed without designs, or run `/design` or `/design-with-v0` first?"

## Post-execution reminders

After the phase scope is complete and the stop condition is reached, remind the user:

1. Run `/validate` to verify the build against the PRD and designs.
2. After validation, run `/update-prd-from-build` to sync the PRD with what was actually implemented.
