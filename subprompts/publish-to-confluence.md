# Publish to Confluence (full feature)

Publishes the **entire feature workspace** to Confluence as a parent hub page with one numbered child page per artifact. Stakeholders get a browsable feature record — research through Jira export — that mirrors the local PDLC workspace.

Runs in parallel with Step 11 (Jira Export) when the PM enables Confluence at the Step 11 pre-flight. Also called by `/change-mode` to update changed pages in place. Available standalone via `/publish-to-confluence`.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Page hierarchy

```
[Feature Name]                                          ← parent hub
  ├─ Step 1: Research
  ├─ Step 2: Codebase Review
  ├─ Step 3: PRD
  ├─ Step 4a: Product Review
  ├─ Step 4b: Technical Review
  ├─ Step 6: System Design                              (only if generated)
  ├─ Step 7: Visual Diagram                             (Figma iframe embed)
  ├─ Step 8: Design Catalog — Phase [N]                 (one page per phase)
  ├─ Step 10: User Stories Breakdown
  └─ Step 10½: Timeline                                 (Figma embed + link to HTML Gantt)
```

Steps 5, 9, 11 do not get pages — they are gate-application / export steps that update the PRD in place or push to Jira. The PRD page (Step 3) reflects all post-gate updates.

---

## Step 1 — Prerequisite check

Verify the Atlassian MCP is connected by calling `getAccessibleAtlassianResources`. If unavailable:

- Apply Error Type 4 from `error-handling.md`: write the intended hub-and-children content to a local fallback file:
  ```
  ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-confluence-export.md
  ```
  Structure the file as one big markdown document with `## Page: [title]` headers separating what would become each Confluence page. Include the full hierarchy so the PM can recreate it manually if needed.
- State to the PM: "Atlassian MCP not connected. Confluence content saved locally for manual publishing or retry."
- Continue the pipeline. Other Step 11 agents (Jira, Drive) proceed independently.

---

## Step 2 — Detect artifacts and decide what to publish

Read `_pipeline-state.json` and the feature workspace folder to determine which artifacts exist:

| Artifact | Source path | Page title | Publish if |
|---|---|---|---|
| Step 1: Research | `research/[feature]-research.md` | `Step 1: Research` | File exists |
| Step 2: Codebase Review | `codebase-review/[feature]-codebase-review.md` | `Step 2: Codebase Review` | File exists |
| Step 3: PRD | `prd/[feature]-prd.md` | `Step 3: PRD` | File exists |
| Step 4a: Product Review | `product-review/[feature]-product-review.md` | `Step 4a: Product Review` | File exists |
| Step 4b: Technical Review | `technical-review/[feature]-technical-review.md` | `Step 4b: Technical Review` | File exists |
| Step 6: System Design | `technical-review/[feature]-system-design.md` | `Step 6: System Design` | File exists |
| Step 7: Visual Diagram | `diagrams/[feature]-feature-diagram.md` | `Step 7: Visual Diagram` | File exists |
| Step 8: Design Catalog — Phase N | `design/[feature]-phase-[N]-designs.md` | `Step 8: Design Catalog — Phase [N]` | File exists (one page per phase file) |
| Step 10: User Stories Breakdown | `user-stories/[feature]-user-stories.md` | `Step 10: User Stories Breakdown` | File exists |
| Step 10½: Timeline | `timeline/[feature]-timeline.md` | `Step 10½: Timeline` | File exists |

Skip any artifact whose source file is missing. Note skipped artifacts on the hub page with a `⏸ Not yet generated` status so the hub remains accurate.

### Per-artifact change detection

For every artifact that exists locally:
1. Read its source file's modification time (`stat -f %m` on macOS, or via the file system API the tool uses).
2. Read `_pipeline-state.json` → `confluence_hub.artifacts.[artifact_key].source_mtime` for the prior published mtime.
3. If no prior published mtime exists → artifact is **new** → publish it.
4. If prior mtime < current mtime → artifact has **changed** → republish it (update existing page).
5. If prior mtime == current mtime → artifact is **unchanged** → skip (do not call updateConfluencePage; preserve existing version).

Build a publish plan and present it to the PM before any API calls:

```
━━━ Confluence publish plan — [Feature Name] ━━━

Parent hub:    [Existing URL] (will update) | [New page — will be created]

Artifacts:
  Step 1:  Research                  → unchanged · skip
  Step 2:  Codebase Review           → changed · update
  Step 3:  PRD                       → changed · update
  Step 4a: Product Review            → unchanged · skip
  Step 4b: Technical Review          → unchanged · skip
  Step 6:  System Design             → not generated · skip
  Step 7:  Visual Diagram            → new · create
  Step 8:  Design Catalog — Phase 1  → unchanged · skip
  Step 10: User Stories Breakdown    → new · create
  Step 10½: Timeline                 → new · create

Total: 4 page operations (4 create/update, 6 skipped)

Proceed? (yes / no / show diff for one of these)
```

Wait for PM confirmation before any API calls. If the PM says "show diff for X", show a unified diff between the local file and the published version (fetch via `getConfluencePage`), then re-present the plan.

---

## Step 3 — Input collection (first run only)

If this is the first Confluence publish for the feature (no `confluence_hub.parent_page_id` in state), ask the PM:

1. **Confluence space** — paste the space key (e.g., `PROD`) or name. The skill will look it up via `getConfluenceSpaces`.
2. **Parent location** (optional) — paste a parent page title or ID if the feature hub should live under another page. If empty, the hub is created at the space root.
3. **Hub page title** — default is the feature name from `_pipeline-state.json` → `intake.feature_name`. Override if needed.

If a `confluence_hub.parent_page_id` already exists in state, skip these questions — reuse the existing hub.

### Legacy migration

If state has `export_urls.confluence_page` set (legacy single-PRD page from a pre-v2.3.0 run) but no `confluence_hub.parent_page_id`:

```
A legacy Confluence PRD page exists for this feature:
  [URL]

Migrate to the new hub model? The legacy page becomes "Step 3: PRD" under a new
parent hub page. The legacy URL is preserved so existing bookmarks keep working.

(yes / no — keep legacy single-page model)
```

If yes: create the new parent hub, reparent the legacy PRD page under it (call `updateConfluencePage` with the new `parentId`), then proceed with the publish plan for the remaining artifacts. Record both the new `confluence_hub.parent_page_id` and the migrated page's ID in `confluence_hub.artifacts.step_3_prd.page_id`.

If no: bail out of this run — do not mix the two models. Tell the PM they can run `/publish-to-confluence` again later if they change their mind.

---

## Step 4 — Compose page content per artifact

For each artifact in the publish plan that is **create** or **update**, compose its page content:

### Parent hub page

```markdown
# [Feature Name]

**Owner:** [from PRD]
**Pipeline status:** [Gate N approved · YYYY-MM-DD · current step]
**Jira Epic:** [URL — if Step 11a already ran, otherwise "Not yet created"]
**Drive folder:** [URL — if Drive sync ran, otherwise omit]
**Last updated:** [YYYY-MM-DD]

## What is this feature?
[One-paragraph summary from PRD Executive Summary.]

## Pipeline timing

[If `_pipeline-state.json` → `timing_report` has data, include this section. Otherwise omit.]

| Metric | Time |
|---|---|
| **Wall-clock total** | [Xh Ym] |
| **Active work** (model time) | [Xh Ym] |
| **Gate-wait time** (you reviewing) | [Xh Ym] |

[Full per-step breakdown lives at `timing/[feature]-timing.md` in the local workspace.]

## Pipeline artifacts

| Step | Artifact | Status |
|---|---|---|
| 1 | [Step 1: Research](child-url) | ✅ Published [date] |
| 2 | [Step 2: Codebase Review](child-url) | ✅ Published [date] |
| 3 | [Step 3: PRD](child-url) | ✅ Published [date] |
| 4a | [Step 4a: Product Review](child-url) | ✅ Published [date] |
| 4b | [Step 4b: Technical Review](child-url) | ✅ Published [date] |
| 6 | Step 6: System Design | ⏸ Not generated for this feature |
| 7 | [Step 7: Visual Diagram](child-url) | ✅ Published [date] |
| 8 | [Step 8: Design Catalog — Phase 1](child-url) | ✅ Published [date] |
| 10 | [Step 10: User Stories Breakdown](child-url) | ✅ Published [date] |
| 10½ | [Step 10½: Timeline](child-url) | ✅ Published [date] |

## Decision log
[From PRD Section 10 — locked decisions only, most recent first.]

## Open questions
[From PRD Section 11.]
```

Update the parent hub on **every** run (even when no child changed) so the `Last updated` date and any child URLs stay current.

### Step 1: Research

```markdown
# Step 1: Research — [Feature Name]

**Source:** `research/[feature]-research.md` · **Last updated:** [YYYY-MM-DD from file mtime]
**Pipeline:** [← Parent hub] · [Next: Step 2: Codebase Review →]

[Full content of the research document, preserving headings.]
```

### Step 2: Codebase Review

```markdown
# Step 2: Codebase Review — [Feature Name]

**Source:** `codebase-review/[feature]-codebase-review.md`
**Pipeline:** [← Step 1: Research] · [↑ Parent hub] · [Next: Step 3: PRD →]

[Full content.]
```

### Step 3: PRD

The PRD page keeps the existing PRD-page structure but adds the breadcrumb at the top:

```markdown
# Step 3: PRD — [Feature Name]

**Owner:** [from PRD] · **Status:** [In progress / Tickets created / Shipped]
**Source:** `prd/[feature]-prd.md` · **Last updated:** [YYYY-MM-DD]
**Pipeline:** [← Step 2: Codebase Review] · [↑ Parent hub] · [Next: Step 4a: Product Review →]

## Quick links
- Jira Epic: [URL or omit]
- Drive folder: [URL or omit]
- Visual diagram: [Figma URL or omit]

## Executive Summary
[PRD Section 1]

## Scope
[PRD Section 2]

## Roles & Permissions
[PRD Section 3, or "Not applicable"]

## Data Model
[PRD Section 4]

## API Contracts
[PRD Section 5, or "Not applicable"]

## State Management (Frontend)
[PRD Section 6]

## Phased Plan
[PRD Section 7]

## Observability & NFR
[PRD Section 8]

## Testing Notes
[PRD Section 9]

## Decision Log
[PRD Section 10]

## Open Questions
[PRD Section 11]
```

### Step 4a: Product Review / Step 4b: Technical Review

```markdown
# Step 4a: Product Review — [Feature Name]
*(or "Step 4b: Technical Review")*

**Source:** `product-review/[feature]-product-review.md`
**Pipeline:** [← Step 3: PRD] · [↑ Parent hub] · [Next: Step 4b: Technical Review →]

[Full content of the review document.]
```

### Step 6: System Design

```markdown
# Step 6: System Design — [Feature Name]

**Source:** `technical-review/[feature]-system-design.md`
**Pipeline:** [← Step 4b: Technical Review] · [↑ Parent hub] · [Next: Step 7: Visual Diagram →]

[Full content.]
```

### Step 7: Visual Diagram

```markdown
# Step 7: Visual Diagram — [Feature Name]

**Source:** `diagrams/[feature]-feature-diagram.md`
**Pipeline:** [← Step 6: System Design] · [↑ Parent hub] · [Next: Step 8: Design Catalog →]

[If Figma diagram URL is available from _pipeline-state.json → export_urls.figma_diagram_url, embed it:]
<iframe src="https://www.figma.com/embed?embed_host=confluence&url=[figma_diagram_url]" width="800" height="450" allowfullscreen></iframe>

[Below the embed, paste the traceability table from the source file mapping each node to its PRD user story.]

[If no Figma URL available, omit the embed entirely. Do not embed Mermaid syntax — it does not render visually in Confluence without a third-party plugin. State at the top of the page: "Diagram is in Mermaid format. View source file at [path] for a renderable version."]
```

### Step 8: Design Catalog (one page per phase)

For every file matching `design/[feature]-phase-*-designs.md`, create or update a page titled `Step 8: Design Catalog — Phase [N]`. Same breadcrumb pattern.

### Step 10: User Stories Breakdown

```markdown
# Step 10: User Stories Breakdown — [Feature Name]

**Source:** `user-stories/[feature]-user-stories.md`
**Pipeline:** [← Step 8: Design Catalog] · [↑ Parent hub] · [Next: Step 10½: Timeline →]

[Full content — Build Sequence Map, per-story sections, traceability appendix.]
```

### Step 10½: Timeline

```markdown
# Step 10½: Timeline — [Feature Name]

**Source:** `timeline/[feature]-timeline.md`
**Pipeline:** [← Step 10: User Stories Breakdown] · [↑ Parent hub]

[If Figma timeline URL is available from _pipeline-state.json → export_urls.figma_timeline_url:]
<iframe src="https://www.figma.com/embed?embed_host=confluence&url=[figma_timeline_url]" width="800" height="450" allowfullscreen></iframe>

[Below: parameter snapshot table, proposed-timeline table, traceability mapping. Note the HTML Gantt is local-only: "Open the interactive HTML Gantt: timeline/[feature]-timeline.html (local file — not embedded in Confluence)."]
```

---

## Step 5 — Publish (in dependency order)

API calls run in this sequence — child pages depend on the parent existing first:

1. **Parent hub.**
   - If `confluence_hub.parent_page_id` is null: call `createConfluencePage` with the input from Step 3. Record the returned `pageId` to `confluence_hub.parent_page_id` and the URL to `confluence_hub.parent_page_url`.
   - If `confluence_hub.parent_page_id` exists: call `updateConfluencePage` with the parent body. Preserve the existing pageId.

2. **Each child page in the publish plan**, in step order.
   - If a child has no existing `confluence_hub.artifacts.[key].page_id`: call `createConfluencePage` with `parentId = confluence_hub.parent_page_id`. Record the new `page_id`, `page_url`, and `source_mtime` (current file mtime at publish time).
   - If a child has an existing `page_id`: call `updateConfluencePage` on that pageId. Update `source_mtime`.
   - **Critical:** never delete + recreate. Always update in place to preserve URLs.

3. **API call format** — for every create/update, use:
   - `cloudId`: from `getAccessibleAtlassianResources`.
   - `spaceId`: from input collection (Step 3) or stored in state.
   - `title`: from the title table in Step 2.
   - `body`: the composed markdown from Step 4.
   - `contentFormat`: `"markdown"`.
   - `parentId`: parent hub id for children; the user-provided parent (or none) for the hub itself.

If any API call fails partway through:
- Stop further publishing for this run.
- Persist whatever IDs and mtimes have been recorded so far to state.
- Report which artifacts succeeded vs failed.
- Apply Error Type 4 for the unfinished artifacts: write their composed markdown to the local fallback file.

---

## Step 6 — Update state

Write to `_pipeline-state.json`:

```json
"confluence_hub": {
  "space_id": "...",
  "space_name": "...",
  "parent_page_id": "...",
  "parent_page_url": "...",
  "last_published_at": "ISO-8601 timestamp",
  "artifacts": {
    "step_1_research":          { "page_id": "...", "page_url": "...", "source_path": "research/...", "source_mtime": 1700000000.0, "last_published_at": "..." },
    "step_2_codebase_review":   { "page_id": "...", "page_url": "...", "source_path": "...", "source_mtime": ..., "last_published_at": "..." },
    "step_3_prd":               { ... },
    "step_4a_product_review":   { ... },
    "step_4b_technical_review": { ... },
    "step_6_system_design":     { ... },
    "step_7_visual_diagram":    { ... },
    "step_8_design_catalog": [
      { "phase": 1, "page_id": "...", "page_url": "...", "source_path": "design/[feature]-phase-1-designs.md", "source_mtime": ..., "last_published_at": "..." },
      { "phase": 2, "page_id": "...", "page_url": "...", "source_path": "design/[feature]-phase-2-designs.md", "source_mtime": ..., "last_published_at": "..." }
    ],
    "step_10_user_stories":     { ... },
    "step_10_5_timeline":       { ... }
  }
}
```

For unchanged-and-skipped artifacts, do **not** touch their existing entry in state — leave the prior `source_mtime` and `last_published_at` intact.

Also keep `export_urls.confluence_page` set to the **Step 3: PRD** child URL (backward-compat for anything still reading that field). Add a new `export_urls.confluence_hub` pointing at the parent.

---

## Step 7 — End-of-run report

Report to the PM:

```
━━━ Confluence Publish Complete ━━━

Hub:    [Parent hub URL]
Space:  [Space name]

Published this run:
  ✅ Step 7: Visual Diagram (new)
  ✅ Step 10: User Stories Breakdown (new)
  ✅ Step 10½: Timeline (new)
  🔄 Step 2: Codebase Review (updated to v[N])
  🔄 Step 3: PRD (updated to v[N])

Skipped (unchanged): 5 artifacts
Skipped (not generated): 1 artifact (Step 6: System Design)

Stakeholders can browse the full feature record at:
  [Parent hub URL]
━━━
```

If running as part of Step 11 parallel block, include the hub URL in the orchestrator's overall Step 11 summary.

---

## Change-mode integration

When called by `/change-mode` after diff propagation:

- For every artifact that `/change-mode` updated, the file mtime advances automatically. The next `/publish-to-confluence` run picks up the change and republishes only those pages.
- The parent hub is **always** updated by change-mode runs, since the pipeline status (current step, decision log) typically advances.
- No explicit per-artifact instruction needed — mtime comparison does the right thing.

---

## Rules

- **Never create duplicate pages silently.** If a title collision happens at create time (page exists in the space at the same parent), pull its pageId and update it instead — never make a `(v2)` page.
- **Preserve URLs across updates.** Always use `updateConfluencePage`, never delete + recreate. Stakeholder bookmarks must keep working.
- **Mtime comparison is per-file, not per-feature.** A change to one source file should only re-publish that one artifact's page (plus the parent hub for status freshness).
- **Confluence is optional and additive.** Jira Export is the authoritative ticket creator. If Confluence publishing fails entirely, the pipeline continues. Partial failures are reported per-artifact, not as a global rollback.
- **Respect space access.** Don't publish to a space the PM doesn't have permission for. The MCP will fail; surface the error clearly with the artifact name.
- **Keep the local files as source of truth.** Confluence is a published mirror. Edits should happen in the local files first (via the skill or manually); then `/publish-to-confluence` syncs them up.
- **Numbered titles are exact.** Use the literal page-title strings from the Step 2 table (`Step 1: Research`, `Step 4a: Product Review`, `Step 10½: Timeline`, etc.) so cross-feature search in Confluence ("step 3: prd") finds them consistently.
