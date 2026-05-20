# Codebase Review

Run a standalone codebase review for an existing or new feature. Use when:
- You want to evaluate a codebase before committing to a PRD (sanity-check what's reusable, what's risky).
- You want a written codebase review you can hand to engineering for early input.
- You're not running the full pipeline yet but want the codebase-grounded view that Step 2 of `/build-product` provides.

Read and follow `ai-framework/00-codebase-review.md`.

## Inputs

The underlying prompt accepts one of four things:

1. **Folder paths on disk (preferred).** Paste the absolute path(s) to the frontend and/or backend codebase. The skill reads directly from those folders — no copy-paste required.
   - Frontend: `/Users/[name]/Desktop/my-app-frontend/`
   - Backend: `/Users/[name]/Desktop/my-app-backend/`
   - Monorepo: a single path covering both.

   Typical workflow: download the repo as a zip from GitHub, extract to Desktop, paste the extracted folder's path.

2. **Paste code inline** — for specific files or snippets when folder access isn't practical.

3. **Describe the architecture in plain English** — when code isn't accessible at all.

4. **Confirm greenfield** — no existing codebase.

No prior pipeline run is required.

## Output

`~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/codebase-review/[feature-name]-codebase-review.md`

The skill will ask for the feature name if it's not already in conversation context.

## When to use the full pipeline instead

If you want the codebase review to feed directly into a PRD and Technical Review (with the codebase-review file passed as input to those steps), run `/build-product` and pick the Work pipeline. The codebase review there hands off automatically.
