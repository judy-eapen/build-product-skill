# Google Drive Sync

Optional. Mirrors the local feature folder to Google Drive so every artifact is shareable, durable, and accessible to stakeholders who don't have access to the PM's local machine.

Runs in parallel with Step 11 (Jira Export) when the PM enables Drive at the Step 11 pre-flight. Also called by `/change-mode` after diff propagation to re-sync only the changed files. Available standalone via `/drive-sync`.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 1 — Prerequisite check

Verify the Google Drive MCP is installed and authenticated. Test by calling a small read operation (e.g., list root folders). If the MCP is unavailable:

- State: "Google Drive MCP not available. Drive sync skipped. To enable: install a Google Drive MCP in Claude Code, authenticate it, and re-run."
- Apply Error Type 4 (preserve artifacts locally, which they already are; no further action needed).
- Continue the pipeline. Jira Export proceeds independently.

---

## Step 2 — Input collection

Ask the PM (if not already provided at the Step 11 pre-flight):

1. **Drive folder location.** Either:
   - Paste a Google Drive folder URL or folder ID (the skill creates the feature subfolder inside it).
   - Or paste a folder NAME and let the skill search for it.
   - Default fallback: ask whether to create a top-level folder named "PDLC Workflow Docs" in the PM's My Drive.

2. **Optional team-share email** — should the new feature folder be auto-shared with anyone? Comma-separated email list. Default: none.

3. **File format**: upload as `.md` (default, preserves source) or auto-convert PRD to a Google Doc as well (`.md` + `.gdoc` versions for the PRD only). Default: `.md` only.

Confirm inputs back to the PM before proceeding.

---

## Step 3 — Folder structure

In the target Drive folder, create (or update if existing):

```
[target folder]/
└── PDLC Workflow Docs/                    ← created once, top-level
    └── [feature-name]/                    ← created per feature
        ├── research/
        ├── codebase-review/
        ├── prd/
        ├── product-review/
        ├── technical-review/
        ├── diagrams/
        ├── design/
        ├── user-stories/
        ├── jira-export/                   ← includes manifest + export file
        ├── changelog/                     ← populated by /change-mode runs
        └── _pipeline-state.md             ← uploaded at the end of the sync
```

If "PDLC Workflow Docs" already exists at the target location, reuse it. If `[feature-name]` already exists, this is an update — overwrite changed files, leave others alone.

---

## Step 4 — Upload artifacts

Walk the local feature folder at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/`. For each file:

1. Read the local file.
2. Upload to the matching Drive path.
3. If a file with the same name already exists at the target path, **overwrite** it (don't create duplicates with numbered suffixes).
4. Track success/failure per file.

If the PM opted into the Google Doc version for the PRD: after uploading the `.md` PRD, also create a `.gdoc` version using the MCP's "convert markdown to Google Doc" capability if available. Skip silently if not supported.

**Do not** upload:
- `node_modules/`, `.git/`, `.DS_Store`, or other dev artifacts (shouldn't exist in the feature folder anyway, but defensive).
- Files with extensions other than `.md`, `.txt`, `.pdf`, `.png`, `.jpg`, `.svg` (the skill should only produce markdown today, but defensive).

---

## Step 5 — Generate the root summary

At the feature folder root in Drive, create or update a file named `_FEATURE_SUMMARY.md` (or convert to a Google Doc `_FEATURE_SUMMARY` if the PM enabled .gdoc conversion).

The summary doc should contain:

```markdown
# [Feature Name] — Pipeline Summary

**Owner:** [from PRD]
**Status:** [In progress at Step N / Tickets created / Shipped]
**Last synced:** [YYYY-MM-DD HH:MM]

## Quick links
- **PRD:** [link to PRD file in this Drive folder]
- **User Stories Breakdown:** [link to breakdown]
- **Jira Epic:** [Epic URL if Step 11 already ran]
- **Visual Diagram:** [link to diagram]
- **Design Catalog:** [link]

## What this feature is
[Two-sentence summary from PRD Executive Summary]

## Build status
- Phase 1: [N] tickets — [status]
- Phase 2: [N] tickets — [status]
- ...

## Artifacts in this folder
- research/
- codebase-review/
- prd/
- ...

## Changelog
Most recent change-mode runs surface here. Full history at `changelog/[feature]-changelog.md`.

- [YYYY-MM-DD] [trigger type]: [one-line description]
- [YYYY-MM-DD] ...
```

This becomes the single page stakeholders open first.

---

## Step 6 — Share (optional)

If the PM provided a team-share email list in Step 2, share the **feature folder** (not the root "PDLC Workflow Docs") with each email:

- Permission: Comment (default — stakeholders can read and comment but not edit; PM stays the source of truth).
- Send notification: true.

If sharing fails for one email, continue with the others. Report any failures to the PM.

---

## Step 7 — End-of-run summary

Report to the PM:

```
━━━ Drive Sync Complete ━━━

Folder: [Drive URL]
Files uploaded: [N created] + [N updated]
Failures: [N] (see list below if any)
Summary doc: [URL to _FEATURE_SUMMARY]
Shared with: [N] team members

Stakeholders can now open: [folder URL]
━━━
```

If running as part of Step 11 (parallel with Jira Export), include the Drive URL in the orchestrator's overall Step 11 summary.

---

## Change-mode integration

When called by `/change-mode` after diff propagation:

- Input: the list of changed file paths from the blast-radius report.
- Action: re-upload only those changed files to their matching Drive paths (overwriting).
- Also: update the `_FEATURE_SUMMARY.md` (or `_FEATURE_SUMMARY` gdoc) with a fresh "Last synced" timestamp and prepend the latest change-mode entry to the changelog section.
- Skip full-folder walk. This is a surgical sync.

---

## Rules

- **Never delete files** from Drive automatically. If a local file is removed, it stays on Drive unless the PM explicitly asks to clean up. Drive is a record; deletions should be deliberate.
- **Always check the MCP is available** before doing anything. If it's missing, skip cleanly — never crash.
- **Drive sync is optional and additive.** Jira Export is the authoritative ticket creator. If Drive sync fails, the pipeline continues. The user gets a notification but the rest of the work isn't lost.
- **Never share folders broader than needed.** Default permission is Comment, not Edit. The PM is the owner.
- **Keep the local folder as the source of truth.** Drive is a mirror, not a separate workflow target. All edits should happen locally first (via the skill or manually) and then sync.
