# Update PRD from Build

Read and follow `ai-framework/04b-update-prd-from-build.md`.

Sync the PRD with what was actually implemented so the PRD stays the single source of truth for the next phase and future validation.

## When to use

- After `/validate` for a phase (workflow step 8b).
- Before starting the next phase's design or execution.

## Your process

1. **Get inputs.** Ask:
   - Which PRD? (e.g. `@~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`)
   - Which phase was just built?

2. **Check for validation context.** Look for a validation report at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/[feature-name]-phase-[N]-validation.md`. If one exists, read it to understand what passed, what failed, and what was adjusted during execution.

3. **Follow the framework file.** Read `ai-framework/04b-update-prd-from-build.md` and apply all updates: data model changes, implementation notes, scope/AC that changed, copy/flow updates, decision log.

4. **Save and confirm.** Save the PRD. State:
   - "PRD updated for Phase [N] from build. Ready for next phase."
