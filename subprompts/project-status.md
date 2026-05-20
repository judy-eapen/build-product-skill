# Project Status

Check where you are in the product development workflow. Scans the `~/Desktop/Resources/PDLC Workflow Docs/` folder and reports what's been done, what's in progress, and what the next recommended step is.

## When to use

- Resuming a project after a break.
- Before starting a new session to remember where you left off.
- To get a quick overview of project progress.

---

## Your process

### 1. Scan the docs folder

Check for the existence and modification dates of files in these locations:

| Folder | What it means |
|--------|--------------|
| `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/` | PRDs exist. Check filenames and last modified dates. |
| `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/` | PRD reviews have been done. |
| `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/` | CTO reviews have been done. |
| `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/` | Design catalogs exist. Check which phases have designs. |
| `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/` | Validation reports exist. Check which phases have been validated. |
| `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/learning/` | Learning reports exist. Check which phases have post-ship reflections. |
| `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/` | Jira story documents exist (Work workflow). |

Also check:
- `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/design-tokens.md` -- design system established?
- Any system design docs in `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/`

### 1b. Check agent transcripts

Scan the agent transcripts folder for recent chats related to each project:

- List files in the agent transcripts directory, sorted by modification date (most recent first).
- Read the most recent 3-5 transcripts. Look for project names, PRD references, or app directory names that match known projects.
- For each project found, note the most recent transcript date and a one-line summary of what was discussed or done (e.g. "Phase 2 design review", "Fixed login bug", "CTO review of PRD").

### 1c. Check git activity (if applicable)

For each project directory that is a git repo:

- Run `git log --oneline -5` to get the 5 most recent commits.
- Note the most recent commit date and message.
- If no git repo exists in the project directory, note "No git history."

### 2. Determine current state

Based on what exists, determine which workflow step the project is at:

| If you find... | Current state |
|---------------|--------------|
| No PRD | Step 1: Start with `/research-idea` or `/create-prd` |
| PRD exists, no reviews | Step 3: Run `/review-prd` and `/cto-review` |
| PRD + reviews, no designs | Step 6: Run `/design` or `/design-with-v0` for Phase 1 |
| PRD + reviews + Phase N designs + Jira stories (`~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/` exists), no validation | Work project: Run `/compare-figma-prd` when designer has created Figma. Then `/execute-plan` when ready to implement. |
| PRD + reviews + Phase N designs, no code | Step 7: Run `/execute-plan` for Phase N |
| PRD + code for Phase N, no validation | Step 8: Run `/validate` for Phase N |
| PRD + validation for Phase N | Step 9: Ship, then `/learn` |
| Learning report for Phase N | Step 6: Start Phase N+1 with `/design` or `/design-with-v0` |

### 3. Report

```markdown
# Project Status: [Project name from PRD title]

**Last updated:** [Date of most recent file change]

## Documents

| Document | Status | Path | Last Modified |
|----------|--------|------|--------------|
| PRD | Exists / Missing | [path] | [date] |
| PRD Review | Done / Not done | [path] | [date] |
| CTO Review | Done / Not done | [path] | [date] |
| Design Catalog (Phase 1) | Done / Not done | [path] | [date] |
| Design Catalog (Phase 2) | Done / Not done | [path] | [date] |
| Design Tokens | Established / Not yet | [path] | [date] |
| Validation (Phase 1) | Done / Not done | [path] | [date] |
| Learning (Phase 1) | Done / Not done | [path] | [date] |
| ... | ... | ... | ... |

## Current Phase

**Phase [N]** — [Phase name from PRD]

## Next Step

**[Command]** — [Why this is the next step]

## Last Activity

| Source | Date | Summary |
|--------|------|---------|
| Docs | [most recent doc change date] | [which doc changed] |
| Agent transcript | [most recent relevant chat date] | [one-line summary] |
| Git | [most recent commit date] | [commit message] |

## Stale Projects

[If any known project has not had doc changes, transcript activity, or git commits in 14+ days, list it here with the last activity date. If all projects are active, write "None -- all projects active."]

## Workflow Progress

[Visual checklist of completed steps]

- [x] Research
- [x] PRD
- [x] PRD Review
- [x] CTO Review
- [x] Apply fixes
- [x] Phase 1 Design
- [x] Phase 1 Execute
- [ ] Phase 1 Validate  <-- you are here
- [ ] Phase 1 Ship + Learn
- [ ] Phase 2 Design
- [ ] Phase 2 Execute
- ...
```

---

## Important

- **Read-only.** This command only scans and reports. It does not create or modify anything.
- **Infer, don't guess.** If a file exists, report it. If it doesn't, say so. Don't assume work was done without evidence.
- **Be specific.** Report file paths and dates so the user can verify.
- **Suggest one next step.** Don't list all possible actions — pick the most logical next step based on what's missing.
