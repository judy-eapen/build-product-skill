# Pull Designs from Figma — Refresh Catalog & Sync Downstream Artifacts

Reads the current state of a Figma file (the one `push-to-figma` seeded, or any file the designer iterated on) and pulls the latest frames, screenshots, metadata, and tokens back into the feature workspace. Refreshes the design catalog with real screenshots + URLs, then offers to sync the PRD and user stories with any changes the designer made.

**Standalone — not part of the auto-run pipeline.** Designers iterate asynchronously (days or weeks after `push-to-figma`), so this runs manually whenever designs are ready to be pulled back.

**Prerequisite:** Figma MCP must be connected (`claude.ai Figma`). If it isn't, ask the PM to install it before proceeding.

**Companion to `/push-to-figma`.** Typical flow: `/design-prompts` → `/push-to-figma` → *designer iterates in Figma* → `/pull-from-figma` → *PRD + stories synced*.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Input Check

Before doing anything, confirm:

1. **Feature name + workspace.** Ask the PM which feature this pull is for. Expect `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/` to exist. If not, ask for the path.
2. **Figma file URL.** Check `_pipeline-state.json` → `figma_generation.target_file_url`. If present, confirm with the PM: "I'll pull from [file URL]. Is this still the right file, or did the designer move to a new file?" If not present in state, ask for the URL.
3. **Figma MCP is connected.** Quick check: `whoami()` on the Figma MCP. If it fails, ask the PM to connect and retry.
4. **PRD path.** Default: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`. Confirm exists.
5. **User stories path (optional).** Default: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md`. If absent, skip the user-stories sync at Step 6.

Ask the PM:
1. **Scope.** Pull *all* pages/frames from the file, or a subset (e.g., "just page A", "just the screens with the @needs-pull tag")?
2. **Depth.** **Full pull** (screenshots + metadata + tokens + variable changes) or **light pull** (URLs + thumbnails only). Default: full.
3. **Sync downstream?** After the catalog refresh, do you want me to (a) run a PRD diff and offer updates, (b) run a user-stories diff and offer AC updates, (c) both, or (d) catalog refresh only? Default: both.

Wait for answers.

---

## Step 1 — Load Required Skills + Resolve File

**MANDATORY before any Figma MCP read** — load the `figma-use` skill from `skill://figma/figma-use/SKILL.md`. (Skills are needed for read calls that may involve component metadata.)

Then:

1. Extract `file_key` from the file URL.
2. Call `get_metadata({ fileKey })` — captures top-level structure (pages, frame names, last-modified, last-modified-by).
3. Cross-check against `_pipeline-state.json` → `figma_generation.frame_ids`:
   - **Frames still present** with same names → matched, will pull.
   - **Frames renamed or moved** → present in metadata but key changed. Flag as `renamed`.
   - **Frames deleted** since push → present in state but not in file. Flag as `deleted`.
   - **New frames** the designer added → present in file but not in state. Flag as `added`.

Save discoveries to a scratch object — not yet to state.

Present the inventory to the PM:

```
Pulling from [file URL]
Last modified: [timestamp] by [designer name]

Matched frames:   N
Renamed frames:   N  (list)
Deleted frames:   N  (list)
New frames:       N  (list)

Proceed with pull?
```

Wait for approval.

---

## Step 2 — Fetch Frames (Per Page)

For each page in scope, walk the frames and pull data. Group calls by page for throughput.

**Per frame, depending on depth setting:**

| Data | Tool | Notes |
|---|---|---|
| Direct URL | derive from `file_key` + `node-id` | always included |
| Screenshot | `get_screenshot({ nodeId, fileKey })` | save PNG to `design/figma-pulls/[YYYY-MM-DD]/[frame-name].png` |
| Layout/structure metadata | `get_design_context({ nodeId, fileKey })` | full pull only — captures hierarchy, copy, layout |
| Variable bindings | `get_variable_defs({ nodeId, fileKey })` | full pull only — captures which tokens are bound where; lets you spot when designer swapped in a different token |

**Throughput:** expect ~1–2 MCP calls per frame for light, ~3–4 for full. For ~30 frames at full depth, plan on 90–120 calls. Communicate progress per page.

**Save screenshots locally** under `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/figma-pulls/[YYYY-MM-DD]/` so the catalog can reference them by relative path. Confirm the directory before writing the first screenshot.

---

## Step 3 — Refresh Design Catalog

The catalog at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-figma-catalog.md` was written by `push-to-figma` with the initial frames. Now overwrite it with the post-iteration state.

Confirm the path before writing: `Writing: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-figma-catalog.md`

Updated catalog structure:

```markdown
# Design Catalog — [Feature Name] (Figma — pulled via MCP)

**Tool used:** Figma (read via Figma MCP)
**Phase:** [N or "All v1"]
**Initial generation:** [date from state.figma_generation.generated_at]
**Last pulled:** [YYYY-MM-DD HH:MM]
**Last modified in Figma:** [timestamp from get_metadata] by [designer name]
**PRD:** [path]
**Source file:** [target Figma file URL]

## Change summary since initial generation

- Matched frames: N
- Renamed frames: N — list
- Deleted frames: N — list
- New frames: N — list
- Token swaps detected: N — list (full pull only)

## [Category A name]

| Frame | URL | Screenshot |
|---|---|---|
| A.1 [Frame name] | [Direct node URL] | ![A.1](figma-pulls/[date]/A.1-[name].png) |
| A.2 [Frame name] | ... | ... |

[Repeat per category. Mark new/renamed/deleted with badges: 🆕 NEW, ✏️ RENAMED (was: [old name]), 🗑️ DELETED]

## Design tokens applied (post-pull)

| Token | Variable key | Usage | Changed since push? |
|---|---|---|---|
| [name] | [key] | [usage] | yes/no |

## Open design questions

[Pull from any annotations or notes the designer left in Figma if accessible; otherwise carry forward from the prior catalog.]

## Notes for the PM

- This catalog reflects what the designer has produced as of [last pulled timestamp].
- Frames marked 🆕 NEW or ✏️ RENAMED suggest the designer expanded or restructured beyond the original PRD scope — review the diff in the next step.
- Screenshots are PNG exports stored under `figma-pulls/[date]/`. Re-running `/pull-from-figma` writes a new dated subfolder; older snapshots are preserved for audit.

## Provenance

This catalog was refreshed via the Figma MCP read APIs (`get_metadata`, `get_design_context`, `get_screenshot`, `get_variable_defs`). The frames remain editable in the source Figma file.
```

---

## Step 4 — PRD Diff (Optional)

Skip if PM chose "catalog refresh only" at Step 0.

Otherwise, run the same logic as `/compare-figma-prd`:

1. Load the PRD.
2. For each frame, compare:
   - **Copy** in the design vs. acceptance criteria / user-story narrative in the PRD.
   - **Flow** implied by frame order/links vs. the user-flow section of the PRD.
   - **Scope** — are new frames covering scope not in the PRD? Are deleted frames removing flows the PRD still calls out?
3. Build a structured diff:
   ```
   ## Additions in Figma not in PRD
   - [Frame A.7 — Confirmation modal] — new screen, no AC in PRD covers it
   - [Frame B.3 — Settings] — adds toggle for X, PRD doesn't mention X

   ## Removals in PRD not in Figma
   - [Story 4.2 — Bulk delete] — PRD describes; no frame depicts

   ## Changes
   - [Frame A.1 — Welcome] — CTA copy changed from "Get Started" to "Create Account"
   - [Frame B.2 — List view] — sort order changed from date-desc to relevance
   ```
4. Present the diff. **Do not write to the PRD yet.** Ask: "Apply these to the PRD? (a) yes, all; (b) walk through one by one; (c) skip — I'll handle manually."
5. If yes or walkthrough: invoke `update-prd-from-designs.md` logic (read `ai-framework/03b-update-prd-from-designs.md`) and apply approved changes. Add a decision-log entry per change.

---

## Step 5 — User Stories Diff (Optional)

Skip if PM chose "catalog refresh only" or if the user-stories file doesn't exist.

1. Load `[feature]-user-stories.md`.
2. For each story, check whether the relevant frames still exist, were renamed, or were materially changed.
3. Surface findings:
   ```
   ## Stories affected by design changes
   - [US-12] FE: Display welcome screen — frame A.1 copy changed → AC needs update
   - [US-08] BE: Sort listings by date → Figma now shows relevance sort → BE contract may need update
   - [US-15] FE: Bulk delete → no frame depicts this anymore → story may be dropped or deferred
   ```
4. Ask the PM: "Update affected AC inline? (a) yes; (b) walk through; (c) skip."
5. If yes/walkthrough, edit the stories file in place. Preserve story IDs. **Do not add a `## Change log` section to the stories file** — per `ai-framework/style-preferences.md` § Artifact Conventions, change history stays out of reader artifacts. The record of these edits goes to the centralized changelog only (Step 6 below appends to `changelog/[feature]-changelog.md` → "User Stories Breakdown" section).
6. **If Jira MCP is connected**, also offer to push the AC updates to the corresponding Jira tickets. If not, note in the report that the user-stories markdown is updated but Jira tickets still need manual sync (or `/compare-figma-prd` for the Jira leg).

---

## Step 6 — Save State

Update `_pipeline-state.json` → `figma_generation`:

```json
{
  "figma_generation": {
    "target_file_url": "...",
    "target_file_key": "...",
    "page_ids": { ... },
    "frame_ids": {
      "A.1 Welcome": "9:10",
      "A.7 Confirmation modal": "9:90"
    },
    "catalog_path": "~/Desktop/...",
    "generated_at": "YYYY-MM-DD HH:MM",
    "fidelity": "production-trace",
    "frame_count": N,
    "last_pulled_at": "YYYY-MM-DD HH:MM",
    "last_pull_summary": {
      "matched": N,
      "renamed": N,
      "deleted": N,
      "added": N,
      "token_swaps": N
    },
    "screenshot_dir": "design/figma-pulls/[YYYY-MM-DD]/"
  }
}
```

Append to the feature's `changelog/[feature]-changelog.md` (the single feature-level file, grouped by artifact — append a dated line under each artifact section this pull touched, e.g. `## Design Catalog`, `## PRD`, `## User Stories Breakdown`; create a section if missing):

```
## Design Catalog
- [YYYY-MM-DD] — Pulled from Figma: N frames from [file URL]; designer changes [matched / renamed / deleted / added counts].

## PRD
- [YYYY-MM-DD] — Pulled from Figma: [count] PRD updates applied (or "no PRD change").

## User Stories Breakdown
- [YYYY-MM-DD] — Pulled from Figma: [count] AC/story updates applied (or "no story change").
```

---

## Step 7 — Report Back

Output to the PM:

```
✓ Pulled N frames from Figma → [target file URL]
Catalog refreshed: [path]
Screenshots: [screenshot dir]

Designer changes since last push:
  Renamed: N
  Deleted: N
  Added:   N
  Token swaps: N

Downstream sync:
  PRD updates applied: [count or "skipped"]
  User-story updates applied: [count or "skipped"]
  Jira sync: [pushed via MCP / manual export pending / skipped]

Next:
  - Review the refreshed catalog
  - If PRD was updated, consider re-running /validate-prd
  - If user stories were updated, consider re-running /validate-user-stories
```

---

## Rules

- **Read-only on the Figma file.** This skill never writes back to Figma — only reads. All writes are local (catalog, PRD, stories, state, changelog) or via Jira MCP if connected.
- **No mutation without approval.** Catalog refresh is automatic once the PM approves the pull inventory at Step 1. PRD and user-story edits require a second approval at Step 4 / Step 5.
- **Mandatory skill load.** Load `figma-use` from MCP resources before any Figma MCP read call. Pass `resource:figma-use` in `skillNames` if a `use_figma` call is needed (e.g., for any post-pull annotation).
- **Snapshots preserved.** Each pull writes screenshots to a new dated subfolder under `figma-pulls/`. Old snapshots are never overwritten — gives an audit trail of what the design looked like over time.
- **Catalog is replaceable; PRD is editable in place.** The figma-catalog.md is fully overwritten each pull. The PRD is edited in place via the `update-prd-from-designs` logic, with decision-log entries appended (never overwritten).
- **Token swaps are signal.** If `get_variable_defs` shows a variable changed (e.g., primary action color now bound to a different token), surface it explicitly — designers swap tokens for a reason and it often implies brand or accessibility intent the PRD should capture.
- **No re-push.** This command does not re-push to Figma. If the PRD changes from this sync trigger further design work, that's a separate `/push-to-figma` or `/design-prompts` run.

---

## Limitations

- **Frame name is the join key.** Matching push-state to pull-state relies on frame names being stable. If a designer renames frames *and* changes IDs (e.g., by deleting and recreating), the skill flags them as deleted+added rather than renamed. The PM can correct this interactively when reviewing the inventory.
- **Copy diff is heuristic.** Comparing Figma copy to PRD AC is a fuzzy text match plus model judgment. Subtle wording changes may not be flagged; the PM should still spot-check the screenshots.
- **No designer-comment ingestion.** The Figma MCP read API does not currently expose Figma comments/threads, so designer rationale in comments isn't pulled. Ask the designer to surface key decisions in frame names or a "Notes" frame if they want them captured.
- **Image fills not pulled.** Same constraint as push — `get_screenshot` captures rendered pixels but `get_design_context` can't extract original image assets. If real photos/logos need to land in the PRD, the designer should provide them separately.
