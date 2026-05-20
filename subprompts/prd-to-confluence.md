# PRD to Confluence

Optional. Publishes the PRD (and optionally the User Stories Breakdown + Visual Diagram) as a Confluence page so non-Jira stakeholders can browse the feature documentation.

Runs in parallel with Step 11 (Jira Export) when the PM enables Confluence at the Step 11 pre-flight. Also called by `/change-mode` to update the page in place when artifacts change. Available standalone via `/prd-to-confluence`.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 1 — Prerequisite check

Verify the Atlassian MCP is connected by calling `getAccessibleAtlassianResources` (or equivalent). If unavailable:

- Apply Error Type 4 from `error-handling.md`: write the intended Confluence content to a local fallback file:
  ```
  ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-confluence-export.md
  ```
- State to the PM: "Atlassian MCP not connected. Confluence content saved locally for manual publishing or retry."
- Continue the pipeline. Other Step 11 agents (Jira, Drive) proceed independently.

---

## Step 2 — Input collection

If called from Step 11 with pre-flight inputs already collected, skip ahead. If called standalone or missing inputs, ask the PM:

1. **Confluence space** — paste the space key (e.g., `PROD`) or name. The skill will look it up via `getConfluenceSpaces`.
2. **Parent page** (optional) — paste the page title or ID. If provided, the new page lives under this parent. If empty, the page is created at the space root.
3. **Page title** — default is the PRD title from the PRD file. Override if needed.
4. **Update mode** — if a page with this title already exists in the space:
   - Update in place (preserve page ID, increment version)
   - Create new page with a suffix (e.g., `[Title] (v2)`)
   - Ask before doing either

Confirm inputs back to the PM before proceeding.

---

## Step 3 — Compose the page content

Read the PRD from `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`.

Build the Confluence page in markdown (Confluence accepts markdown via the `contentFormat: "markdown"` param). Structure:

```markdown
# [Feature Name] — PRD

**Owner:** [from PRD]
**Status:** [In progress / Tickets created / Shipped]
**Last updated:** [YYYY-MM-DD]

## Quick links
- Jira Epic: [URL — if Step 11a already ran, otherwise omit this line]
- User Stories Breakdown: [Drive URL — if Drive sync also ran, otherwise omit]
- Visual Diagram: [Drive URL or inline Mermaid block if PM prefers]

## Executive Summary
[From PRD Section 1]

## Scope
[From PRD Section 2 — in scope + out of scope]

## Roles & Permissions
[From PRD Section 3, or "Not applicable" with the intake reason]

## Data Model
[From PRD Section 4]

## API Contracts
[From PRD Section 5, or "Not applicable"]

## State Management (Frontend)
[From PRD Section 6]

## Phased Plan
[From PRD Section 7]

## Observability & NFR
[From PRD Section 8]

## Testing Notes
[From PRD Section 9]

## Decision Log
[From PRD Section 10]

## Open Questions
[From PRD Section 11]
```

If the PM also enabled inline diagrams, embed the Mermaid diagram from `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md` in the "Quick links" section. Confluence renders Mermaid natively.

---

## Step 4 — Publish

### Create new page

Use `createConfluencePage` with:
- `cloudId`: from `getAccessibleAtlassianResources`.
- `spaceId`: the resolved space ID.
- `title`: from Step 2.
- `body`: full markdown from Step 3.
- `contentFormat`: `"markdown"`.
- `parentId`: from Step 2 if provided.

### Update existing page

Use `updateConfluencePage` with the resolved `pageId`. Increment the version. Same body content.

---

## Step 5 — End-of-run report

Report to the PM:

```
━━━ Confluence Publish Complete ━━━

Page: [Title]
URL: [Confluence URL]
Space: [Space name]
Parent: [Parent page title or "Space root"]
Status: [Created / Updated to v[N]]

Anyone with access to the space can read at: [URL]
━━━
```

If running as part of Step 11 parallel block, include the Confluence URL in the orchestrator's overall Step 11 summary.

---

## Change-mode integration

When called by `/change-mode` after diff propagation:

- If the PRD changed: re-compose the page content from the updated PRD and call `updateConfluencePage` (preserves the page URL — important so stakeholder bookmarks still work).
- If only the User Stories Breakdown changed (PRD untouched): optionally update the "Quick links" section of the existing Confluence page to point at the new breakdown version. Otherwise no Confluence change.
- If neither the PRD nor links changed: skip Confluence update entirely.

---

## Rules

- **Never create duplicate pages silently.** If a page with the same title exists, ask the PM how to handle it.
- **Preserve the page URL across updates.** Use `updateConfluencePage`, not delete + recreate.
- **Confluence is optional and additive.** Jira Export is the authoritative ticket creator. If Confluence publishing fails, the pipeline continues; the user gets a notification but the rest of the work isn't lost.
- **Respect space access.** Don't publish to a space the PM doesn't have permission for. The MCP will fail; surface the error clearly.
- **Keep the local PRD as the source of truth.** Confluence is a published mirror. All edits should happen in the local PRD first (via the skill or manually) and then publish.
