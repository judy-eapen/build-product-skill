# Compare Figma to PRD and Jira

Compare the designer's Figma output to the PRD and Jira user stories, identify gaps and changes, then update the PRD and Jira tickets accordingly.

## When to use

- Work pipeline has completed (steps 1-6b done).
- Designer has created their version in Figma.
- You want to sync PRD and Jira with what was actually designed.

You can run this in the same session as the work pipeline or later. Use `/project-status` to see where you left off; when it suggests "Run /compare-figma-prd", run this command.

---

## Your process

### 1. Get inputs

- **Figma link or node ID:** User provides URL (e.g. `https://figma.com/design/...?node-id=1-2`), or use Figma MCP if the file is open. Extract node ID from URL if needed (e.g. `node-id=1-2` → `1:2`).
- **PRD path:** Infer from `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/` (most recent or match by name) or ask.
- **Jira context:** Epic key, or path to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-export.md`, or use Jira MCP to fetch existing stories. If user provides Epic key or Jira link, use it.

### 2. Fetch Figma design

- Use Figma MCP `get_design_context` and/or `get_screenshot` for the relevant node(s).
- Extract screens, components, copy, and flows from the design output.

### 3. Compare

- Load PRD user stories and acceptance criteria.
- Load Jira stories (from MCP or `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/`).
- Diff:
  - What is in Figma but not in PRD/Jira?
  - What is in PRD/Jira but not in Figma?
  - What changed (copy, flows, behavior)?

### 4. Present diff

- Show a structured comparison (additions, removals, changes).
- Ask user to confirm before applying.

### 5. Update PRD

- Apply changes per `ai-framework/03b-update-prd-from-designs.md`: design catalog reference, copy/flows, acceptance criteria, decision log.
- Add a design catalog reference to the Figma file if not already present (e.g. link or path to the Figma design).

### 6. Update Jira

- **If Jira MCP available:** Edit existing tickets (summary, description, User Story field, Acceptance criteria field) with the changes.
- **If not:** Regenerate `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-export.md` with updated content for manual sync. Tell the user.

### 7. Confirm

- "PRD and Jira updated from Figma comparison. Ready for implementation when you run `/execute-plan`."

---

## Important

- **Read-only until confirmed.** Do not update PRD or Jira until the user confirms the diff.
- **Figma MCP:** Requires Figma Desktop with the file open, or a valid node ID from a shareable link.
- **Jira MCP:** If unavailable, create/update the fallback document in `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/` so the user can sync manually.
