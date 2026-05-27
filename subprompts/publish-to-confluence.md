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
  ├─ Step 6: System Design                              (only if generated)
  ├─ Step 7: Visual Diagram                             (Figma iframe embed)
  ├─ Step 8: Design Catalog — Phase [N]                 (one page per phase)
  ├─ Step 10: User Stories                              (lightweight — Jira Epic index, not full breakdown)
  └─ Step 10½: Timeline                                 (Figma embed + link to HTML Gantt)
```

**Intentionally excluded from Confluence (v2.8.0):**
- **Step 4a: Product Review** and **Step 4b: Technical Review** — internal review artifacts. Kept local in `product-review/` and `technical-review/`, referenced in the PRD's decision log when relevant.
- **Full User Stories Breakdown** (`user-stories/[feature]-user-stories.md`) — too large for Confluence and duplicates what's already in Jira. Replaced by a lightweight Step 10 page with Jira Epic links and story titles only (no Gherkin AC).

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
| Step 6: System Design | `technical-review/[feature]-system-design.md` | `Step 6: System Design` | File exists |
| Step 7: Visual Diagram | `diagrams/[feature]-feature-diagram.md` | `Step 7: Visual Diagram` | File exists |
| Step 8: Design Catalog — Phase N | `design/[feature]-phase-[N]-designs.md` | `Step 8: Design Catalog — Phase [N]` | File exists (one page per phase file) |
| Step 10: User Stories (Jira index) | `user-stories/[feature]-user-stories.md` *(mtime source — page content is composed from `user_stories.epics[]` in state, not from the breakdown file itself)* | `Step 10: User Stories` | `user_stories.epics[]` in state has at least one entry |
| Step 10½: Timeline | `timeline/[feature]-timeline.md` | `Step 10½: Timeline` | File exists |

Skip any artifact whose source file is missing. Note skipped artifacts on the hub page with a `⏸ Not yet generated` status so the hub remains accurate.

**Step 4a (Product Review) and Step 4b (Technical Review) are excluded entirely** — these internal artifacts stay local. On the hub page, list them under a "Kept local — not published" subsection with their local file paths so stakeholders know they exist, with a one-line "Internal review artifact" explanation.

### Per-artifact change detection

For every artifact that exists locally:
1. Read its source file's modification time (`stat -f %m` on macOS, or via the file system API the tool uses).
2. Read `_pipeline-state.json` → `confluence_hub.artifacts.[artifact_key].source_mtime` for the prior published mtime.
3. If no prior published mtime exists → artifact is **new** → publish it.
4. If prior mtime < current mtime → artifact has **changed** → republish it (update existing page).
5. If prior mtime == current mtime → artifact is **unchanged** → skip (do not call updateConfluencePage; preserve existing version).

### Pre-publish drift + comment check (v2.15.0+)

For every artifact in the publish plan that resolves to `update` (not `create`, not `skip`), run a pre-publish check **before presenting the plan to the PM**. This protects against two failure modes: silently overwriting Confluence-side edits made by a stakeholder, and silently orphaning inline comments anchored to body text that's about to be replaced.

Three lightweight MCP calls per `update` artifact:

1. **Drift detection.** Call `getConfluencePage(pageId)` for the artifact's recorded `page_id`. Compare the returned `version.number` against `_pipeline-state.json` → `confluence_hub.artifacts.[artifact_key].last_published_version`. If `current_version > last_published_version`, the page has been edited outside the skill since the last publish. Legacy state files (pre-v2.15.0) without `last_published_version` are treated as `last_published_version = 0` — the first drift check always passes, then the field starts tracking on the next successful publish.
2. **Inline comment fetch.** Call `getConfluencePageInlineComments(pageId)`. These are the risky ones — they're anchored to specific text in the body. `updateConfluencePage` may orphan them if the anchored text changes or moves between composition runs.
3. **Footer comment fetch.** Call `getConfluencePageFooterComments(pageId)`. These are attached to the page itself (not body text) and survive `updateConfluencePage` automatically. Surface them for context but do not gate on them.

Run all three calls in parallel per artifact to keep the pre-flight fast. If any call fails (permissions, rate limit, transient API error), record the failure for that artifact and treat its drift + comment state as `unknown` — surfaced as `❓` in the plan rather than blocking the run.

Build a publish plan and present it to the PM before any update API calls:

```
━━━ Confluence publish plan — [Feature Name] ━━━

Parent hub:    [Existing URL] (will update · v8 → v9) | [New page — will be created]

Artifacts:
  Step 1:   Research                  → unchanged · skip
  Step 2:   Codebase Review           → changed · update     · v3 → v4  · ✓ no comments
  Step 3:   PRD                       → changed · update     · v5 → v6  · 🚨 DRIFT (v5 > last-published v2) · ⚠ 3 inline + 1 footer
                                          └─ inline: Sarah on "60-day TTL on chat-log…", Mike on "Path B saves ~1–1.5 weeks", David ✓ on "Phase 4 cutover…"
                                          └─ footer: 1 thread (David, 2026-05-24, "Approved overall — see inline.")
  Step 6:   System Design             → not generated · skip
  Step 7:   Visual Diagram            → new · create        · 🎨 push diagram to Figma first (no URL in state)
  Step 8:   Design Catalog — Phase 1  → unchanged · skip
  Step 10:  User Stories (Jira index) → new · create
  Step 10½: Timeline                  → new · create        · 🎨 push timeline to Figma first (no URL in state)

Total: 4 page operations (4 create/update, 4 skipped)
Figma pre-push: 2 generations (diagram + timeline)
Pre-flight: 1 drift detected (Step 3 PRD), 3 inline comments at risk, 1 footer thread (safe)

Kept local (not published):
  Step 4a: Product Review
  Step 4b: Technical Review
  Full User Stories Breakdown (Gherkin AC — kept local + attached to Jira Epics)

Per-page actions for pages with ⚠ or 🚨:
  Step 3 PRD:
    1. proceed       — publish overwrites; inline anchors may orphan; drift gets overwritten
    2. skip          — leave this page alone this run; revisit after resolving manually
    3. pull-comments — pull comments into local source (see below) + skip this page this run
    4. show-drift    — show unified diff of current Confluence body vs the last-published body
    5. show-comments — show full comment thread bodies

Global options: yes (proceed all) / no (abort run) / skip figma push / show diff for one of these
```

### Annotations explained

- **`v[N] → v[N+1]`** — Confluence's own page version number; the page is at version N now, the publish will bump it to N+1. Missing for `create` and `skip`.
- **`🚨 DRIFT (v[N] > last-published v[M])`** — current Confluence version is higher than what the skill last published. Someone edited the page outside the skill. Always pair with the `show-drift` per-page action so the PM can see what changed before deciding.
- **`⚠ N inline + M footer`** — N inline comments anchored to body text (at risk of orphaning on overwrite) + M footer threads (safe — survive `updateConfluencePage`). Inline comments are surfaced with `[author] on "[anchor-text excerpt up to 80 chars]"` so the PM can recognize them without leaving the terminal.
- **`✓ no comments`** — page has no comments at all; safe to update.
- **`❓ pre-flight check failed`** — getConfluencePage / getInlineComments / getFooterComments returned an error for this page (permissions, rate-limit, transient). The PM can still choose to proceed but the skill couldn't verify drift or comments. Treat this conservatively: when in doubt, ask the PM to retry the run.

### Per-page resolution mechanics

When the PM picks an action other than `proceed` for a flagged page, here's what happens before publish kicks off:

- **`skip`** — remove this artifact from the publish plan. State is not updated for this page (no new `last_published_version`, no new `source_mtime`). The local file mtime stays in state from prior run, so the next run will still detect it as changed and re-prompt.
- **`pull-comments`** — write a sidecar file to `[feature-workspace]/confluence-feedback/[YYYY-MM-DD]/[step-N]-comments.md` with structured comment dumps (see "Comment sidecar format" below), then `skip` this page from the current publish plan. The PM resolves comments in the local source file, then re-runs `/publish-to-confluence`. For Step 3 (PRD) specifically, the sidecar also includes a pointer line: "To synthesize these into PRD edits, run `/read-feedback`."
- **`show-drift`** — fetch the current Confluence page body (HTML format for readable diff), compare against the last-published body if cached, present a unified diff to the PM, then re-prompt for the per-page action. Note: the skill does not currently cache last-published body content — drift display falls back to "current Confluence body vs the body the skill is about to publish" so the PM can see what's at risk.
- **`show-comments`** — print the full comment thread bodies (already fetched in the pre-flight, no extra API call), then re-prompt.

### Comment sidecar format

When `pull-comments` is picked, write to `[feature-workspace]/confluence-feedback/[YYYY-MM-DD]/[step-N]-comments.md`:

```markdown
# Confluence comments pulled from [Page Title] — [YYYY-MM-DD]

**Source page:** [Confluence URL]
**Pulled at:** [ISO-8601 timestamp]
**Skill:** /publish-to-confluence v2.15.0+ (pre-flight comment pull)

> The Confluence page was not updated this run. Resolve these comments in the local source file ([source_path]), then re-run /publish-to-confluence to push the resolved version. After the next successful publish, mark these comments as resolved in Confluence manually (the skill does not resolve comments — comment-resolution would require write access to comments which is a separate flow).

## Inline comments

### Comment 1 — [Author Name] · [Date] · status: [unresolved | resolved]

**Anchored to:** "[full anchored text from Confluence, verbatim]"

**Body:**
[full comment body, markdown if available]

[If there are replies, render as nested blockquote with author + date prefix]

### Comment 2 — …

## Footer comments

### Thread 1 — [Author Name] · [Date]

**Body:**
[full comment body]

[Replies under the thread]
```

If the artifact is `prd/[feature]-prd.md`, append a pointer line at the bottom: `**Auto-synthesis available:** Run `/read-feedback [step-3-comments.md]` to translate these comments into proposed PRD edits.` (The `/read-feedback` command remains PRD-specific in this release; sidecar dumps for non-PRD pages are read-and-edit-by-hand for v2.15.0.)

### Drift display detail

For pages flagged with `🚨 DRIFT`, the `show-drift` action presents:

```
━━━ Drift detail — Step 3: PRD ━━━

Page ID:                 2042036255
Last published version:  2 (2026-05-24 01:54)
Current version:         5 (last edit 2026-05-24 14:22 by Sarah Chen)

The page has been edited 3 times in Confluence since /publish-to-confluence last
ran on it. The skill does not cache prior page bodies, so a full prior-vs-current
diff is not available. Below is the diff between the CURRENT Confluence body and
the body /publish-to-confluence would publish if you proceed.

[unified diff — current Confluence body vs the freshly composed body]

Choose: proceed / skip / pull-comments / abort
```

Wait for PM confirmation before any update API calls. If the PM picks `proceed` for all flagged pages and resolves any `❓` cases, advance to Step 3 (input collection) or Step 3.5 (Figma auto-push) as the existing flow specifies.

The `🎨 push diagram/timeline to Figma first` annotation appears for Step 7 and Step 10½ only when **all three** conditions hold: (a) the artifact is being created or updated this run, (b) the corresponding URL (`export_urls.figma_diagram_url` for Step 7, `export_urls.figma_timeline_url` for Step 10½) is missing from `_pipeline-state.json`, and (c) the Figma MCP is connected (probed via a lightweight `whoami` or `get_metadata` call at the start of Step 2). If the PM responds with "skip figma push", proceed without the push and fall back to the Mermaid-source note in those pages. If the URL is already in state, no annotation appears and Step 3.5 is skipped for that artifact.

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

## Step 3.5 — Figma auto-push pre-composition (Visual Diagram + Timeline)

Confluence cannot render Mermaid syntax without a third-party plugin, so Mermaid-bearing artifacts (Step 7 Visual Diagram and Step 10½ Timeline) ship as raw code blocks unless a Figma equivalent is available to embed via iframe. This step closes that gap: before composing page content, push Mermaid diagrams to Figma so the iframe-embed branch of Step 4 lights up.

### Eligibility check (per artifact, per run)

Run Step 3.5 only for **Step 7: Visual Diagram** and **Step 10½: Timeline**, and only when **all** of the following hold:

1. The artifact appears in the Step 2 publish plan as `new · create` or `changed · update` (skip for `unchanged · skip` and `not generated · skip`).
2. The corresponding URL in state is null or missing:
   - Visual Diagram: `_pipeline-state.json` → `export_urls.figma_diagram_url`
   - Timeline: `_pipeline-state.json` → `export_urls.figma_timeline_url`
3. The Figma MCP is connected — probed at the start of Step 2 via a single lightweight call (`whoami` or any read-only Figma tool). Cache the probe result for the duration of the run.
4. The PM did not respond "skip figma push" at the Step 2 confirmation prompt.

If any condition fails, skip Step 3.5 for that artifact and continue to Step 4. The Step 4 composition logic already handles both branches (URL present → embed; URL absent → Mermaid-source note), so no further action is needed.

### Push mechanics

For each eligible artifact, call the dedicated generation skill to produce a Figma deliverable and capture the returned URL:

| Artifact | Skill to invoke | Figma surface | What gets pushed |
|---|---|---|---|
| Step 7: Visual Diagram | `/visual-diagram` (Figma-MCP branch) | FigJam file | The architecture diagram(s) from `diagrams/[feature]-feature-diagram(s).md` — §1 high-level and §2 detailed v1 layout at minimum. User-journey sequence diagrams (§3) are pushed if the skill supports them; otherwise the FigJam contains only the architecture views with a note pointing to the Mermaid source for journeys. |
| Step 10½: Timeline | `/timeline` (Figma-MCP branch) | FigJam timeline | The Gantt bars from `timeline/[feature]-timeline.md` — phase windows + epic-level bars + milestone markers + REA cadence overlay. Parameter snapshot stays local (text-only, not a visual). |

The publish-to-confluence skill does **not** reimplement the Figma generation — it delegates to the existing commands so authoring stays in one place. If the dedicated skill fails (Figma MCP rate-limit, generation error, file-creation refusal), capture the error message, persist what was created (if anything), and fall through to the Mermaid-source note for that artifact this run. The PM can retry the push standalone via `/visual-diagram` or `/timeline` and the next `/publish-to-confluence` run picks up the URL via the existing eligibility check.

### State persistence (immediately after each push)

On successful push, write to `_pipeline-state.json` **before composing Step 4 content** so the composition step reads the fresh URL:

```json
"export_urls": {
  "figma_diagram_url": "https://www.figma.com/file/...",
  "figma_diagram_pushed_at": "ISO-8601 timestamp",
  "figma_timeline_url": "https://www.figma.com/file/...",
  "figma_timeline_pushed_at": "ISO-8601 timestamp"
}
```

Also append a one-line entry to `changelog/[feature]-changelog.md` if the feature has a changelog folder convention in use: `[date] · Figma diagram pushed during /publish-to-confluence: [URL]`. This is best-effort — skip if no changelog folder exists.

### PM-visible progress

Surface a short progress line per push so the PM knows what's happening between confirmation and the page-creation API calls:

```
🎨 Pushing diagram to Figma via /visual-diagram (FigJam) … done · [URL]
🎨 Pushing timeline to Figma via /timeline (FigJam) … done · [URL]
```

On failure: `🎨 Diagram push failed (Figma MCP error: [message]). Falling back to Mermaid-source note for this run.`

### Re-push behavior

If the URL is already in state (eligibility condition #2 fails), Step 3.5 is skipped. To force a re-push (e.g., because the underlying diagram changed materially and the existing Figma file is stale), the PM clears the URL from state manually or runs `/visual-diagram` or `/timeline` directly — that's the explicit re-generation path. `/publish-to-confluence` deliberately does not auto-re-push on every diagram-source change, because Figma files are stakeholder-visible artifacts and silent overwrites would surprise designers iterating on them.

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
| 6 | [Step 6: System Design](child-url) | ✅ Published [date] *(or: ⏸ Not generated for this feature)* |
| 7 | [Step 7: Visual Diagram](child-url) | ✅ Published [date] |
| 8 | [Step 8: Design Catalog — Phase 1](child-url) | ✅ Published [date] |
| 10 | [Step 10: User Stories](child-url) | ✅ Published [date] · Lightweight — links to Jira Epics |
| 10½ | [Step 10½: Timeline](child-url) | ✅ Published [date] |

## Kept local — not published to Confluence

These artifacts exist on disk but are intentionally not mirrored here:

- **Step 4a: Product Review** — internal review artifact. Local: `product-review/[feature]-product-review.md`
- **Step 4b: Technical Review** — internal review artifact. Local: `technical-review/[feature]-technical-review.md`
- **Full User Stories Breakdown** (with Gherkin AC) — too large for Confluence; the full doc lives at `user-stories/[feature]-user-stories.md` and is also attached to each Jira Epic. The lightweight Step 10 page above provides Jira Epic navigation.

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

### Step 4a / 4b: Reviews — not published

**Skip.** Product Review and Technical Review are internal artifacts kept local in `product-review/` and `technical-review/`. They do not get Confluence pages. The PRD's decision log surfaces any review findings that ended up locked into the spec — that's the stakeholder-facing record.

### Step 6: System Design

```markdown
# Step 6: System Design — [Feature Name]

**Source:** `technical-review/[feature]-system-design.md`
**Pipeline:** [← Step 3: PRD] · [↑ Parent hub] · [Next: Step 7: Visual Diagram →]

[Full content.]
```

### Step 7: Visual Diagram

```markdown
# Step 7: Visual Diagram — [Feature Name]

**Source:** `diagrams/[feature]-feature-diagram.md`
**Pipeline:** [← Step 6: System Design] · [↑ Parent hub] · [Next: Step 8: Design Catalog →]

[Read _pipeline-state.json → export_urls.figma_diagram_url. By the time Step 4 runs, Step 3.5 has already populated this URL if the Figma MCP was connected and the URL was missing. Three composition branches:]

[Branch A — URL present (most common path post-Step-3.5): embed as Figma iframe.]
<iframe src="https://www.figma.com/embed?embed_host=confluence&url=[figma_diagram_url]" width="800" height="450" allowfullscreen></iframe>

[Below the embed, paste the traceability table from the source file mapping each node to its PRD user story.]

[Branch B — URL absent because Figma MCP unavailable or PM said "skip figma push": state at the top of the page "Diagram is in Mermaid format. View source file at [path] for a renderable version." Do NOT embed Mermaid syntax as a code block — it does not render visually in Confluence without a third-party plugin. Paste the traceability table below the note.]

[Branch C — URL absent because Step 3.5 push failed: state at the top of the page "Figma push attempted but failed during this run ([error message from Step 3.5]). Diagram is in Mermaid format — view source file at [path] for a renderable version, or retry the push via /visual-diagram." Paste the traceability table below the note.]
```

### Step 8: Design Catalog (one page per phase)

For every file matching `design/[feature]-phase-*-designs.md`, create or update a page titled `Step 8: Design Catalog — Phase [N]`. Same breadcrumb pattern.

### Step 10: User Stories (lightweight Jira-index page)

This page replaces the full Gherkin breakdown for Confluence. The full breakdown is kept local (too large) and attached to each Jira Epic. This page is a navigation index that points stakeholders at the right Epic for any story they want to dig into.

**Compose from `_pipeline-state.json` → `user_stories.epics[]`** (not from the breakdown file itself). Each Epic in state should already have a `jira_key` populated by Step 11a Jira Export. If `jira_key` is missing for any Epic (Jira export hasn't run yet, or that Epic's creation failed), label that Epic "(Jira creation pending)" instead of linking.

```markdown
# Step 10: User Stories — [Feature Name]

**Source of truth:** Jira Epics (linked below) and the local breakdown at
`user-stories/[feature]-user-stories.md` (kept local — too large for Confluence).
**Pipeline:** [← Step 8: Design Catalog] · [↑ Parent hub] · [Next: Step 10½: Timeline →]

User stories for this feature are tracked in Jira, one ticket per story under the
appropriate Epic. This page is a navigation index — it links to each Jira Epic and
lists the stories under it. The full Gherkin acceptance criteria, testing notes,
and per-story detail live in the Jira tickets and in the local breakdown file
(attached to each Jira Epic).

## Epics

### Epic 1: [title]
- **Phase:** [N]
- **Theme:** [one-line theme from user_stories.epics[].theme]
- **Stories:** [N] total ([N] FE, [N] BE)
- **Jira Epic:** [JIRA-KEY](Jira-URL) *(or "(Jira creation pending)" if jira_key not set)*

Stories under this Epic:
- US-1.1 [FE] [story title]
- US-1.2 [BE] [story title]
- US-1.3 [FE] [story title]

### Epic 2: [title]
- **Phase:** [N]
- **Theme:** [...]
- **Stories:** [N] total ([N] FE, [N] BE)
- **Jira Epic:** [JIRA-KEY](Jira-URL)

Stories under this Epic:
- US-2.1 [FE] [story title]
- ...

[Repeat per Epic in user_stories.epics[].]

## Notes

- For full acceptance criteria, testing notes, and per-story detail, open the
  Jira ticket for that story or open the local breakdown file.
- This page does NOT list Gherkin scenarios — intentional. Gherkin AC is the
  reading-mode artifact for engineers/QA in Jira; Confluence stakeholders use
  this page for navigation only.
- If the breakdown is updated (`/change-mode`, `/user-stories`, or
  `/change-mode` → "Designs arrived"), this page is re-published automatically
  based on the breakdown file's mtime.
```

**Source mtime for this page:** use the breakdown file's mtime (`user-stories/[feature]-user-stories.md`) as the source-of-change indicator — when the breakdown changes (new stories, refreshed DRAFT stories, etc.), this page should re-publish so the index reflects the new structure.

### Step 10½: Timeline

```markdown
# Step 10½: Timeline — [Feature Name]

**Source:** `timeline/[feature]-timeline.md`
**Pipeline:** [← Step 10: User Stories Breakdown] · [↑ Parent hub]

[Read _pipeline-state.json → export_urls.figma_timeline_url. Step 3.5 populates this URL when the Figma MCP is connected and the URL was missing. Three branches mirroring the Visual Diagram page:]

[Branch A — URL present: embed as Figma iframe.]
<iframe src="https://www.figma.com/embed?embed_host=confluence&url=[figma_timeline_url]" width="800" height="450" allowfullscreen></iframe>

[Branch B — URL absent because Figma MCP unavailable or PM said "skip figma push": state at the top "Figma timeline not yet generated — view source file at [path] (markdown tables) or open timeline/[feature]-timeline.html locally for the interactive Gantt."]

[Branch C — URL absent because Step 3.5 push failed: state at the top "Figma push attempted but failed during this run ([error]). Source markdown + local HTML Gantt remain authoritative — retry via /timeline."]

[Below the iframe or note: parameter snapshot table, proposed-timeline table, traceability mapping. Always include the HTML Gantt pointer: "Open the interactive HTML Gantt: timeline/[feature]-timeline.html (local file — not embedded in Confluence)."]
```

---

## Step 5 — Publish (in dependency order)

API calls run in this sequence — child pages depend on the parent existing first:

1. **Parent hub.**
   - If `confluence_hub.parent_page_id` is null: call `createConfluencePage` with the input from Step 3. Record the returned `pageId` to `confluence_hub.parent_page_id` and the URL to `confluence_hub.parent_page_url`.
   - If `confluence_hub.parent_page_id` exists: call `updateConfluencePage` with the parent body. Preserve the existing pageId.

2. **Each child page in the publish plan**, in step order.
   - If a child has no existing `confluence_hub.artifacts.[key].page_id`: call `createConfluencePage` with `parentId = confluence_hub.parent_page_id`. Record the new `page_id`, `page_url`, `source_mtime` (current file mtime at publish time), and **`last_published_version`** (the `version.number` from the create response — typically `1`).
   - If a child has an existing `page_id`: call `updateConfluencePage` on that pageId. Update `source_mtime` and **`last_published_version`** (the `version.number` from the update response — typically the prior version + 1, but always use what the API actually returned).
   - **Critical:** never delete + recreate. Always update in place to preserve URLs.
   - **Pages skipped from the publish plan** (PM picked `skip` or `pull-comments` at the Step 2 pre-flight): do not call `updateConfluencePage` and do not update state for these pages. Their `last_published_version` and `source_mtime` from the prior run remain authoritative, so the next `/publish-to-confluence` run will detect them as `update` candidates again and re-run the pre-flight.

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
  "last_published_version": 9,  // v2.15.0+ — Confluence version.number after the most recent successful update of the parent hub
  "artifacts": {
    "step_1_research":          { "page_id": "...", "page_url": "...", "source_path": "research/...", "source_mtime": 1700000000.0, "last_published_at": "...", "last_published_version": 1 },
    "step_2_codebase_review":   { "page_id": "...", "page_url": "...", "source_path": "...", "source_mtime": ..., "last_published_at": "...", "last_published_version": 4 },
    "step_3_prd":               { "page_id": "...", "page_url": "...", "source_path": "...", "source_mtime": ..., "last_published_at": "...", "last_published_version": 6 },
    "step_6_system_design":     { "page_id": "...", "page_url": "...", "source_path": "...", "source_mtime": ..., "last_published_at": "...", "last_published_version": 1 },
    "step_7_visual_diagram":    { "page_id": "...", "page_url": "...", "source_path": "...", "source_mtime": ..., "last_published_at": "...", "last_published_version": 1 },
    "step_8_design_catalog": [
      { "phase": 1, "page_id": "...", "page_url": "...", "source_path": "design/[feature]-phase-1-designs.md", "source_mtime": ..., "last_published_at": "...", "last_published_version": 2 },
      { "phase": 2, "page_id": "...", "page_url": "...", "source_path": "design/[feature]-phase-2-designs.md", "source_mtime": ..., "last_published_at": "...", "last_published_version": 1 }
    ],
    "step_10_user_stories":     { "page_id": "...", "page_url": "...", "source_path": "user-stories/[feature]-user-stories.md", "source_mtime": ..., "last_published_at": "...", "last_published_version": 3, "format": "lightweight-jira-index" },
    "step_10_5_timeline":       { "page_id": "...", "page_url": "...", "source_path": "...", "source_mtime": ..., "last_published_at": "...", "last_published_version": 1 }
  }
}
```

**Note (v2.15.0+):** `last_published_version` is a new field. Legacy state files without it are treated as `last_published_version = 0` during the next pre-flight drift check, so the first post-upgrade run never triggers a false drift alarm. After that first successful update, the field is populated from the API response and drift detection becomes authoritative.

**Note (v2.8.0+):** `step_4a_product_review` and `step_4b_technical_review` are no longer published. If they appear in legacy state files from a pre-v2.8.0 pipeline run, the entries are ignored on subsequent runs — the corresponding Confluence pages are **not** deleted automatically (preserving stakeholder bookmarks). If you want to delete the prior review pages from Confluence after upgrading, do so manually in Confluence. The hub's "Kept local — not published" section will note that the reviews are no longer mirrored.

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
- **Pre-flight is mandatory for `update` pages (v2.15.0+).** Every page resolving to `update` runs the drift + comment pre-flight before the PM sees the publish plan. The PM cannot skip the pre-flight, only act on its findings. Pages resolving to `create` skip the pre-flight (no prior version, no prior comments to disturb).
- **Footer comments survive `updateConfluencePage` automatically.** They're attached to the page, not anchored to body text. Surface their count for context but do not gate on them.
- **Inline comments are at risk on every update.** The skill composes pages from scratch rather than surgically updating sections, so even semantically-identical body text may orphan inline anchors. The pre-flight makes this visible per page; the PM decides. There is no automatic "preserve anchors" path — `markdown` contentFormat (current default) is not round-trip safe for inline-comment markers.
- **`pull-comments` is the non-destructive path.** Picking it writes comments to a dated sidecar at `confluence-feedback/[YYYY-MM-DD]/[step-N]-comments.md` and skips that page from the current publish run. The Confluence page is not modified — inline comments stay anchored, the page stays at its current version. PM resolves the comments in the local source file and re-runs `/publish-to-confluence`.
- **Drift never auto-resolves.** When `🚨 DRIFT` is detected on a page, the PM must explicitly pick an action (proceed / skip / pull-comments) — the skill will not proceed-by-default. This protects against silent overwrite of stakeholder edits.
- **Figma push is opt-out, not opt-in.** Step 3.5 fires automatically whenever the Figma MCP is connected and the Figma URL is missing from state for a Mermaid-bearing artifact (Visual Diagram, Timeline). To skip it, the PM answers "skip figma push" at the Step 2 confirmation prompt or clears the Figma MCP connection. This favors fewer manual re-runs of `/publish-to-confluence` over silent fallback to unreadable Mermaid code blocks.
- **Push delegates to dedicated skills.** Step 3.5 invokes `/visual-diagram` and `/timeline`; it does not call the Figma MCP directly. Centralizes Figma authoring so changes to diagram/timeline generation only need to be made in one place.
- **Never re-push silently.** Once a Figma URL is in state, `/publish-to-confluence` treats it as authoritative and does not regenerate even if the local Mermaid source has changed. Re-pushing requires explicit PM action (clear the URL in state OR run `/visual-diagram` / `/timeline` directly). This protects designer iterations on the existing Figma file from being overwritten.
