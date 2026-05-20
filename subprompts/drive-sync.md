# Google Drive Sync

Sync a feature's pipeline artifacts to Google Drive so they're shareable, durable, and accessible to stakeholders who don't have access to the PM's local machine. Use when:

- You ran `/build-product` without enabling Drive sync at Step 11, and now want to share the artifacts.
- A feature is older and you want to push its artifacts to a shared Drive folder for stakeholder review.
- You want to re-sync after manually editing local files.

Read and follow `ai-framework/07-drive-sync.md`.

## Prerequisite

A **Google Drive MCP** must be installed and authenticated in Claude Code. If it's not:

- Add a Drive MCP from your MCP marketplace (Anthropic-supported one is recommended).
- Authenticate with the Google account that owns the destination Drive folder.
- Re-run `/drive-sync`.

## Inputs

The underlying prompt collects:

- **Feature name** — which feature folder to sync.
- **Drive folder location** — paste a folder URL, ID, or name. Default: create "PDLC Workflow Docs" in your My Drive.
- **Optional team-share** — comma-separated emails to share the new feature folder with (Comment access by default).
- **File format** — `.md` only (default) or `.md` + Google Doc versions for the PRD.

## Output

A mirrored folder structure in Drive:

```
[your folder]/PDLC Workflow Docs/[feature-name]/
├── research/
├── codebase-review/
├── prd/
├── product-review/
├── technical-review/
├── diagrams/
├── design/
├── user-stories/
├── jira-export/
├── changelog/
├── _pipeline-state.md
└── _FEATURE_SUMMARY.md   ← single-page stakeholder summary with quick links
```

The skill returns the Drive folder URL at the end so you can share it directly.

## When this command runs

- **Standalone**: a full feature folder sync. Walks every file.
- **Inside `/build-product` Step 11**: runs in parallel with Jira Export and (optionally) Confluence Publish.
- **Inside `/change-mode`**: surgical re-sync of only the changed files identified in the blast-radius report.

## Rules

- The local folder is the source of truth. Drive is a mirror.
- Never deletes files from Drive automatically.
- Always overwrites existing files at matching paths (no duplicate `_v2` files).
- Optional, additive — never required for the rest of the pipeline to succeed.
