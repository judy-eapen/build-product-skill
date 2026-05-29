# Changelog

All notable changes are documented here. Follows [Semantic Versioning](https://semver.org): MAJOR.MINOR.PATCH.

---

## v2.18.0 — 2026-05-29

### Changed

- **No inline change history in artifacts — enforced by heading name** (`ai-framework/style-preferences.md` § Artifact Conventions). The rule now forbids any section titled `## Change log` / `## Changelog` / `## Revision history` / `## Refactor history` / `## Refactor summary` (by any wording), plus `**v0.X (date):**` version-history blocks and "Document version" / "Last updated" stamp lines. A version number in the title line + a one-line changelog pointer is all an artifact carries.
- **Centralized changelog is grouped by artifact.** `changelog/[feature]-changelog.md` is a single feature-level file organized into one `## ` section per artifact (PRD, System Design, User Stories Breakdown, Design Catalog, Timeline, Decision Log, …), each append-only and dated. `/change-mode` Step 6 (`ai-framework/05-change-propagation.md`) writes one dated line under each affected artifact's section per run.
- **Decision log moved out of the PRD into a sidecar** (`decisions/[feature]-decision-log.md`). The PRD's § 10 (`ai-framework/02-create-prd.md`) is now a one-line pointer. All producers append to the sidecar — apply-fixes (`subprompts/build-product.md`), `/update-prd-from-designs` (`ai-framework/03b-…`), `/read-feedback`, conflict resolution (`ai-framework/error-handling.md`), single-source review findings (`ai-framework/05-parallel-rules.md`), `/create-prd` Category-A pre-population. All consumers read from it — `/validate-prd`, `/exec-summary`, `/review-prd`, `/publish-to-confluence`, and the diagram/timeline/user-stories self-checks.

### Added

- **Two leaks closed.** The User Stories Breakdown's Appendix B ("Refactor history / changelog") is now a one-line pointer to the centralized changelog (`ai-framework/06-user-stories.md`), and `/pull-from-figma` no longer writes a `## Change log` into the stories file (`subprompts/pull-from-figma.md`).
- **`/validate-user-stories` Check 8b.** Flags inline change-history / refactor sections anywhere in the breakdown (not just above the per-story blueprints), recommending removal in favor of the centralized changelog + Appendix B pointer.
- **`decisions/` folder** added to the output tree (`SKILL.md`) and the Google Drive sync mirror (`ai-framework/07-drive-sync.md`, `subprompts/drive-sync.md`).

---

## v2.17.0 — 2026-05-29

### Changed

- **Intake Q3 (Jira ticket conventions) is now a suggested-defaults checklist, not a blank prompt** (`subprompts/build-product.md` Stage 0.5, `CLAUDE.md` Intake Parameters). The skill presents proposed conventions the PM confirms / edits / removes, with a concrete suggestion next to each: every-ticket labels (pod/area tag), per-layer labels (`backend` on BE, `frontend` on FE), title format (verb-first + `[BE]`/`[FE]` layer prefix, `"Feature - Sub-feature"` for Epics — explicitly asks how BE vs. FE should be marked in the title so each ticket's layer is obvious), BE/FE split (separate tickets), Testable Yes/No on every ticket, Story Points left blank, and BE↔FE "Relates to" / "Blocked by" link conventions. Saying "no conventions yet" makes the suggestions the working defaults.
- **Intake Q2 now captures the board, not just the project** (`intake.board`) — for teams using a shared board with Components as swim lanes.

### Added

- **Durable conventions profile at `~/Desktop/Resources/PDLC Workflow Docs/_jira-conventions.json`.** Jira project, board, and ticket conventions (Q2 + Q3) are saved once and **confirm-reused** on every future `/build-product` run: intake shows the saved values and asks "reuse / edit / start fresh" instead of re-asking from scratch. The profile is rewritten with the latest agreed values after each run, and the full per-feature snapshot still lives in that feature's `_pipeline-state.json` → `intake`. Replaces the prior best-effort scavenge of another feature's state file.

---

## v2.16.0 — 2026-05-26

### Added

- **Meat-first layout for user-stories breakdowns (`ai-framework/06-user-stories.md` Step 5).** Per-story blueprints now appear near the top of the document. Operational metadata (ID Stability Policy, refactor history, format conventions, full per-story dependency table, vendor sprint alignment, PRD cross-reference) moves to Appendices A–F at the end. New explicit rule: nothing below `## ` level appears above the first `### US-` blueprint header except front matter, At-a-glance, How-to-read, Epic outlines, and the small Wave overview table.
- **Source-of-truth count rule.** The number of unique `### US-` blueprint headers in the document is the single computed value. Every prose count claim (At-a-glance, "Total v1 stories: N", per-epic "N stories" openers, "All N stories written" footers) reads from that count. `/user-stories`, `/change-mode`, and `/validate-user-stories` recompute and reconcile in the same pass that adds or cuts stories. Hand-narrated counts that drift are not allowed.
- **`/validate-user-stories` Check 8 — layout (meat-first / appendix).** Scans every `## ` (level-2) header above the first `### US-` blueprint header. Flags `## ID Stability Policy`, `## Refactor summary` / `Refactor lineage` / `Refactor authority`, `## Format Conventions` / story archetype reference, `## Pass [N] status`, and full per-story dependency tables sitting above the meat. WARNING per section; CRITICAL if cumulative front-matter exceeds 100 lines.
- **`/validate-user-stories` Check 9 — count parity (prose ↔ blueprints).** Counts unique `### US-X.Y` blueprint headers (total + per-epic), extracts every numeric story-count claim from prose, and reports each mismatch with line number, quote, claimed number, actual number, and delta. CRITICAL for top-of-document drift or delta >5%; WARNING otherwise. Proposes corrected numbers; batch-apply allowed since the source-of-truth basis is unambiguous.
- **`/sync-artifacts` Agent C internal-consistency hop.** Beyond PRD-vs-breakdown drift, Agent C now also runs count parity and meat-first layout checks within the breakdown itself. HIGH findings route the user to `/validate-user-stories` for the per-line walkthrough.
- **`/change-mode` preserves layout + counts (`subprompts/change-mode.md` "What it preserves" section).** When change-mode propagates an add/cut/rewrite that touches the user-stories doc, new content lands in the meat (under its `## Epic [N]:` header) or in the named appendix; never above the first blueprint. After every run, count claims are recomputed from blueprint headers and reconciled in the same pass.
- **Workspace-level CLAUDE.md reminder** under "User-Stories Document Layout (across all PDLC features)" — matches the pattern of the existing Diagram Rendering and ID Stability Policy reminders. Captures the rule once at the workspace level so it survives every feature regeneration; points back at `ai-framework/06-user-stories.md` Step 5 as load-bearing.

### Why this matters

The nestfully-ai v1.2 breakdown accreted 460 lines of front-matter through repeated `/change-mode` runs — refactor lineage from v0.9 through v1.2, full ID Stability Policy, full Refactor summary, complete per-story sequence table, format conventions — pushing the first per-story blueprint to line 463 of a 6042-line document. For sponsors, designers, and engineers (the document's primary audience), the doc became unreadable: they had to scroll past hundreds of lines of maintenance content to find anything actionable. Simultaneously, prose count claims drifted across versions — top-of-document "Total v1 stories: 99" alongside "all 93 v1 stories" in an adjacent paragraph alongside an actual table containing 106 IDs.

v2.16.0 fixes both at the source rather than per-document: the layout spec enforces meat-first structure at write time, the count-parity rule makes the blueprint-header count the only source of truth, the validation checks catch drift before it ships, and the workspace reminder keeps the rule visible across every regeneration step.

### Changed

- **`ai-framework/06-user-stories.md`** — Step 5 output structure rewritten (meat-first + appendix layout + source-of-truth count rule); Step 4 quality checks gain Check 12 (layout) and Check 13 (count parity).
- **`subprompts/validate-user-stories.md`** — Step 1 expanded from seven to nine checks; inline summary template updated; Step 3 fix-walkthrough adds layout and count-parity fix patterns.
- **`subprompts/change-mode.md`** — new "What it preserves" section documenting layout and count discipline during propagation.
- **`sync-artifacts/SKILL.md`** — Agent C expanded with internal-consistency hop (checks 6 and 7 within the breakdown itself).

### Migration

No data migration. Existing breakdown files generated under v2.15.0 and earlier will trip the new checks on the next `/validate-user-stories` run — that's expected and surfaces accumulated drift. Per-finding fix walkthrough in Step 3 of validate-user-stories lets the PM apply corrections incrementally; `/change-mode` runs going forward respect the new layout.

For an existing feature with heavy front-matter accretion (nestfully-ai is the canonical example), the recommended path is: (a) run `/validate-user-stories` to see the full drift report, (b) apply Check 9 count fixes via batch confirmation, (c) apply Check 8 layout fixes section-by-section so internal cross-references are preserved. Backups in the feature's `user-stories/` folder (the existing `.bak-before-*` naming pattern) cover rollback.

### Validated on

Designed against the nestfully-ai v1.2 breakdown (106 actual blueprints, 99 / 93 / 70+27 / 97 conflicting count claims, 460 lines of front-matter above first blueprint). Validation pending re-application via `/validate-user-stories` then optional regeneration via `/change-mode`.

---

## v2.15.0 — 2026-05-25

### Added

- **`/publish-to-confluence` — pre-flight drift + comment check (Step 2 expanded).** Every page resolving to `update` (not `create`, not `skip`) now runs three lightweight MCP calls in parallel before the publish plan is presented to the PM: `getConfluencePage` (drift), `getConfluencePageInlineComments` (orphan risk), `getConfluencePageFooterComments` (safe — context only). Findings are surfaced per-page in the publish plan with three new annotations:
  - **`🚨 DRIFT (v[N] > last-published v[M])`** — Confluence's current version is higher than what the skill last published. Someone edited the page outside the skill. Always pair with `show-drift` so the PM can see what's changed before deciding.
  - **`⚠ N inline + M footer`** — N inline comments anchored to body text (at risk of orphaning when `updateConfluencePage` replaces the body) + M footer threads (safe). Inline comments include `[author] on "[anchor-text excerpt]"` so the PM can recognize them without leaving the terminal.
  - **`✓ no comments`** — page has no comments at all; safe to update.
  - **`❓ pre-flight check failed`** — getConfluencePage / getInlineComments / getFooterComments returned an error; PM can still proceed but the skill couldn't verify drift or comments.
- **Per-page resolution actions for flagged pages.** Beyond the global `yes / no / skip figma push`, the PM can pick per page:
  - **`proceed`** — publish overwrites; inline anchors may orphan; drift gets overwritten.
  - **`skip`** — leave this page alone this run; state is not updated for it (so the next run re-runs the pre-flight).
  - **`pull-comments`** — write a structured comment dump to the sidecar `[feature-workspace]/confluence-feedback/[YYYY-MM-DD]/[step-N]-comments.md` and skip the publish for this page. Resolves comments-vs-overwrite by deferring the publish until the PM has addressed them in the local source. For Step 3 PRD specifically, the sidecar includes a pointer line to run `/read-feedback` for auto-synthesis.
  - **`show-drift`** — print Confluence version metadata + a unified diff between the current Confluence body and the body the skill would publish, then re-prompt.
  - **`show-comments`** — print the full comment thread bodies (already fetched), then re-prompt.
- **New state field: `confluence_hub.artifacts.[key].last_published_version`** — Confluence's `version.number` captured from every successful create/update API response and persisted per artifact (plus on the parent hub). Drift detection compares this against the live page's current version.

### Changed

- **`subprompts/publish-to-confluence.md`** — Step 2 pre-publish drift + comment subsection (~80 lines), publish-plan output format, per-page action mechanics, sidecar format spec, drift display detail, Step 5 publish-step capture of `version.number` from API responses, Step 6 state schema with `last_published_version`, Rules section additions (pre-flight is mandatory for `update`; footer comments survive; inline comments are at risk; `pull-comments` is the non-destructive path; drift never auto-resolves).

### Why this matters

PMs (Judy on nestfully-ai) raised the question of what happens to inline comments when `/publish-to-confluence` republishes a page that's already had stakeholder feedback. The answer pre-v2.15.0 was bad: footer comments survived but inline comments could orphan silently, and the skill never even checked whether someone had edited the page in Confluence before overwriting it. Pre-flight v2.15.0 makes both signals visible before any overwrite, and gives the PM a non-destructive path (`pull-comments`) when comments need to be addressed in the local source first. This is a meaningful step toward bidirectional discipline without taking on full bidirectional merge (which the skill's compose-from-scratch architecture makes lossy in the Confluence-to-local direction).

### Limitations

- **No automatic anchor preservation.** The skill still composes pages from scratch and sends `contentFormat: "markdown"`. Even if a page is updated with semantically identical content, inline-comment anchors may orphan because the markdown→storage conversion can change anchor markers. Pre-flight surfaces the risk; it does not eliminate it.
- **Sidecar dump is read-and-edit-by-hand (except PRD).** `pull-comments` writes a structured markdown dump but does not auto-apply the comments to the local source. For PRD specifically, the sidecar points at `/read-feedback` which synthesizes comments into proposed PRD edits. For other artifacts, the PM reads the sidecar and edits the source file directly.
- **Drift display has no cached prior body.** The skill does not cache the body it published in the prior run, so `show-drift` shows the diff between the current Confluence body and the body the skill is *about to publish*, not a true three-way diff against the last-published common ancestor. Good enough to see what's at risk; not a full merge tool.
- **Pre-flight adds 3 MCP calls per `update` page.** For a feature with all 8 children updating, that's 24 extra calls before the publish kicks off. Each call is cheap, but the prompt-to-display latency is a few seconds longer than v2.14.0.
- **Confluence comment resolution is not in scope.** Even after the PM resolves comments in the local source file and re-publishes, the inline comments in Confluence remain in their unresolved state. Comment resolution requires write access to comments (a separate API surface); this release does not touch it.

### Migration

No data migration. Existing features without `last_published_version` in state are handled gracefully:

- On the next `/publish-to-confluence` run, missing `last_published_version` is treated as `0`. The drift check (`current_version > 0`) trivially passes, so every page registers `🚨 DRIFT (v[N] > last-published v0)` — but the publish plan also includes the comment-check output, which is the actually-useful signal. PMs should treat the v0 baseline drift as "expected on first post-upgrade run" and proceed.
- After the first post-upgrade run completes successfully, `last_published_version` is populated for every published page. Subsequent runs use it for genuine drift detection.

### Validated on

Designed against the `nestfully-ai` feature workspace after its v2.13.0 initial publish + v2.14.0 Figma-auto-push. Pre-flight validation: re-running on a feature where a stakeholder has added inline comments + edited the page in Confluence directly should show all three signals (`🚨 DRIFT`, `⚠ inline comments`, `✓ footer thread`) and offer the per-page resolution UX. Validation pending real stakeholder comment activity on a published feature.

---

## v2.14.0 — 2026-05-24

### Added

- **`/publish-to-confluence` — Step 3.5: Figma auto-push pre-composition.** New step inserted between input collection (Step 3) and content composition (Step 4) that closes the rendering gap for Mermaid-bearing artifacts.
  - **Eligibility-gated.** Fires only for **Step 7: Visual Diagram** and **Step 10½: Timeline**, and only when (a) the artifact is `new · create` or `changed · update` this run, (b) the corresponding URL in `_pipeline-state.json` → `export_urls.figma_diagram_url` / `figma_timeline_url` is missing, and (c) the Figma MCP is connected (probed once via `whoami` at the start of Step 2).
  - **Surfaced in the publish plan.** The Step 2 plan output now shows `🎨 push diagram/timeline to Figma first (no URL in state)` next to eligible artifacts, plus a `Figma pre-push: N generations` line and a new `skip figma push` confirmation option.
  - **Delegates to dedicated skills.** `/visual-diagram` (FigJam — architecture) and `/timeline` (FigJam — Gantt). `/publish-to-confluence` does not reimplement Figma generation — centralizes authoring in one place.
  - **State persistence is immediate.** URLs are written to state **before** Step 4 composes the page, so the existing iframe-embed branch picks them up automatically. `figma_diagram_pushed_at` and `figma_timeline_pushed_at` timestamps are also written.
  - **Graceful fallback on failure.** Push failure (MCP error, generation refusal, rate-limit) falls through to a Mermaid-source note for that artifact this run, with a "retry via `/visual-diagram` or `/timeline`" pointer. Other artifacts continue publishing — no global rollback.
- **Step 4 composition — three explicit branches for Visual Diagram and Timeline:**
  - **Branch A — URL present** (most common post-Step-3.5): Figma iframe embed.
  - **Branch B — URL absent because Figma MCP unavailable / PM said "skip figma push"**: "Diagram is in Mermaid format. View source file at [path]." note. No Mermaid code block (it doesn't render).
  - **Branch C — URL absent because Step 3.5 push failed**: explicit "Figma push attempted but failed during this run ([error])" note with retry pointer.

### Changed

- **`subprompts/publish-to-confluence.md`** — Step 2 publish-plan output, new Step 3.5 (~80 lines), Step 4 Visual Diagram + Timeline composition sections, Rules section additions (Figma push is opt-out / delegates to dedicated skills / never re-pushes silently).

### Why this matters

PMs (Judy on nestfully-ai) discovered post-publish that Step 7 and Step 10½ rendered as raw Mermaid code blocks in Confluence — unreadable to non-engineer stakeholders without a third-party plugin the Confluence admin would need to install. The prior workflow required manually chaining `/visual-diagram` → `/timeline` → `/publish-to-confluence`, with the publish skill silently falling back to Mermaid notes if those upstream pushes didn't happen. Step 3.5 makes the push automatic when the Figma MCP is available and the URL is missing, so the iframe-embed branch is the default outcome. Opt-out is one word at the confirmation prompt ("skip figma push") for PMs who specifically want the Mermaid fallback (e.g., when iterating quickly and don't want to clutter Figma).

### Limitations

- **One Figma file per artifact per feature.** Step 3.5 only creates a Figma file when the URL is missing from state — it never overwrites an existing URL. Stakeholder bookmarks and designer iterations are preserved by design. PMs who want to regenerate must clear the URL in state manually or run `/visual-diagram` / `/timeline` directly.
- **Delegates to skills that may themselves fail.** If `/visual-diagram` or `/timeline` is buggy or the Figma MCP rate-limits, Step 3.5 inherits those failures. Behavior on failure is the graceful fallback (Branch C), not a retry loop.
- **No mtime-based re-push.** The auto-push fires on URL-missing, not on diagram-source-changed. A material change to the Mermaid source does not trigger a Figma regeneration; the PM clears the URL or invokes `/visual-diagram` / `/timeline` explicitly. This is deliberate — silent overwrites of a designer-iterated Figma file would surprise designers.
- **Step 3.5 only handles Visual Diagram + Timeline.** Other Mermaid blocks (e.g., the architecture diagrams embedded inside System Design or PRD pages) are not pushed to Figma. Those pages are already composed as section-by-section summaries with the full Mermaid source kept local; embedding fidelity is acceptable for the structural-summary use case.

### Migration

No data migration. Existing features:

- If a Figma URL already exists in `_pipeline-state.json` → `export_urls.figma_diagram_url` / `figma_timeline_url` (populated by a prior `/visual-diagram` or `/timeline` run), Step 3.5 sees the URL and skips — same behavior as today.
- If the URL is missing and the Figma MCP is connected (most common case for features that haven't run the dedicated Figma commands yet), Step 3.5 pushes on next `/publish-to-confluence`. The Step 2 plan output flags this explicitly, so no surprise.
- If the URL is missing and the Figma MCP is **not** connected, behavior is identical to pre-v2.14.0 — the Mermaid-source note is published.

### Validated on

Designed against the `nestfully-ai` feature after its initial `/publish-to-confluence` run (v2.13.0) shipped Step 7 + Step 10½ as raw Mermaid blocks. Re-running `/publish-to-confluence` on v2.14.0 with the Figma MCP connected is the validation path; URLs land in state, both pages re-publish with iframe embeds.

---

## v2.13.0 — 2026-05-23

### Added

- **`/pull-from-figma` standalone command.** Reverse direction of `/push-to-figma`. Pulls the post-iteration state of a Figma file back into the feature workspace after the designer has refined the pushed frames. New file: `subprompts/pull-from-figma.md`.
  - **Diffs against the push-state.** Reads `_pipeline-state.json` → `figma_generation.frame_ids` and cross-checks with `get_metadata`. Surfaces designer changes since the last push: matched frames, renamed (same node ID, new name), deleted (no longer in the file), and added (frames the designer created beyond the original scope). Token swaps are also surfaced — when `get_variable_defs` shows a frame is now bound to a different design system variable.
  - **Catalog overwrite.** The `design/[feature]-figma-catalog.md` file is rewritten with refreshed URLs, screenshots, change summary, and post-pull token bindings. Change badges (🆕 NEW, ✏️ RENAMED, 🗑️ DELETED) make designer changes visible at a glance.
  - **Optional downstream sync.** After the catalog refresh, the PM is offered three sync options: (a) PRD diff + updates via `update-prd-from-designs` logic, (b) user-stories diff + AC updates, (c) both. Both are skippable — catalog refresh is the minimum.
  - **Snapshots preserved.** Screenshots are written to a dated `design/figma-pulls/[YYYY-MM-DD]/` subfolder, so each pull is a separate snapshot. Old snapshots are never overwritten — gives an audit trail of how the design evolved over time.
  - **Standalone only.** Not part of the auto-run pipeline. Designers iterate asynchronously (days or weeks after `/push-to-figma`), so the PM invokes this manually when designs are ready to be pulled back. No coupling to `/build-product` sequencing.
  - **Two depth modes.** Full pull (screenshots + design context + variable defs — ~3–4 MCP calls per frame, captures structure and token changes). Light pull (URLs + thumbnails — ~1–2 MCP calls per frame). Default: full.
  - **Read-only on Figma.** The skill never writes back to Figma. All writes are local (catalog, PRD, stories, state, changelog) or to Jira via MCP if connected. Two approval gates: pull inventory at Step 1, downstream edits at Step 4 / 5.
- **Round-trip loop closed.** `/design-prompts` → `/push-to-figma` → designer iterates → `/pull-from-figma` → PRD + user stories synced. PMs no longer need to hand-reconcile designer changes; the skill compares the iterated Figma file against the push-state and offers structured updates.

### Why this matters

`/push-to-figma` (v2.12.0) handled the outbound direction — prompts to frames. The pull side was still manual: when a designer iterated on the pushed frames (renamed, added, deleted, or swapped tokens), the PM had to either ignore the changes or hand-update the catalog and PRD. `/pull-from-figma` closes that loop programmatically: it reads the iterated file via the Figma MCP, computes a structured diff, and offers to propagate the design changes through the catalog, PRD, and user stories. Designer rationale is preserved because the catalog is regenerated from the live file rather than hand-edited.

### Validated on

Designed against the `nestfully-ai` Figma file (v2.12.0 output: 29 mobile frames, 13 categories, 🐦 Nestfully Mobile library). Pull-side validation pending designer iteration.

### Limitations

- **Frame name is the join key.** Matching push-state to pull-state relies on frame names being stable. If a designer renames a frame *and* changes its node ID (e.g., by deleting and recreating), the skill flags it as deleted+added rather than renamed. The PM can correct this when reviewing the inventory at Step 1.
- **Copy diff is heuristic.** Comparing Figma copy to PRD AC is a fuzzy text match plus model judgment. Subtle wording changes may not surface; PM should still spot-check screenshots.
- **No designer-comment ingestion.** The Figma MCP read API does not expose Figma comments/threads, so designer rationale in comments isn't pulled. Workaround: ask the designer to surface key decisions in frame names or a "Notes" frame.
- **No re-push.** This skill does not re-push to Figma. If the PRD changes from the sync trigger further design work, that's a separate `/push-to-figma` or `/design-prompts` run.

### Migration

No data migration. `/pull-from-figma` is purely additive. Features that ran `/push-to-figma` on v2.12.0 already have the `figma_generation` block in `_pipeline-state.json` that `/pull-from-figma` reads; no state schema change. Features that didn't use `/push-to-figma` can still run `/pull-from-figma` by passing the Figma file URL manually.

---

## v2.12.0 — 2026-05-23

### Added

- **`/push-to-figma` standalone command.** Generates real, editable Figma frames programmatically from a feature's design prompts file via the Figma MCP. New file: `subprompts/push-to-figma.md`.
  - **Wires every frame to the team's design system.** Color tokens are referenced by variable binding (not hardcoded hex). When the source library updates a color, every generated frame updates automatically. Components are imported by key where they fit; primitives bound to variables are used otherwise.
  - **One page per category.** The skill reads the prompts file's category-level structure (A, B, C, …) and creates one Figma page per category. Sub-frames within a category land horizontally on the same page.
  - **Mobile vs. desktop dimensions** are picked from `intake.product_type`: mobile = 390 × 844 (iPhone 14 Pro), web = 1440 × 900.
  - **Companion to `/design-prompts`.** Typical flow: `/design-prompts` to generate the text prompts → PM review → `/push-to-figma` to produce real Figma frames from those prompts. Both commands run standalone.
  - **State persistence.** Writes `figma_generation` block to `_pipeline-state.json` with file URL, page IDs, frame IDs, and discovered variable/component keys — so `/change-mode` and re-runs can reference the existing file rather than recreating it.
  - **No v0 equivalent.** v0 is a browser chat product with no public API for programmatic prompt submission; if PM wants v0 output, run `/design-prompts` and paste manually.
- **Output catalog at `design/[feature]-figma-catalog.md`.** Distinct from `/design-prompts`'s `design/[feature]-phase-[N]-designs.md` so the two outputs don't collide. The Figma catalog includes every frame's direct node URL, the design tokens applied (with variable keys), and the open design questions carried forward from the prompts file.

### Why this matters

PMs (Judy on nestfully-ai) ran the prompts-to-Figma workflow manually for v1 — every screen had to be generated one by one via the MCP. That worked but took dozens of tool calls and was non-repeatable. The new command captures the workflow as a real pipeline step: variable discovery, file creation, page setup, per-frame generation, screenshot validation, catalog write-out, and state save are all built in.

The output is meaningfully better than v0 for teams with a real design system: v0 produces components from scratch (off-brand by default); `/push-to-figma` produces frames already wired to the team's actual color variables and component library, so even mid-fi output looks like the team's product.

### Validated on

`nestfully-ai` — 29 mobile frames across 13 categories generated against the 🐦 Nestfully Mobile library. Catalog at `~/Desktop/Resources/PDLC Workflow Docs/nestfully-ai/design/nestfully-ai-figma-catalog.md`. Generated Figma file: https://www.figma.com/design/1iX0hpldO5R9QrCL5mEzyi.

### Limitations

- **Image fills not supported** — `use_figma` can't fetch external URLs, so listing photos / hero images land as placeholder rectangles. For web apps the designer can run `generate_figma_design` in parallel to capture pixel-perfect images (see the `figma-generate-design` MCP skill).
- **Production-trace ≠ Gate 2 spec.** Output is mid-fi: real IA, real tokens, illustrative copy from the PRD. Typography hierarchy, illustrations, and per-state polish are the designer's job. The catalog says this explicitly.
- **Re-runs create new frames by default.** Idempotent overwrite is supported only if PM explicitly requests it (then existing frames are modified in place by name, never deleted-and-recreated, to preserve any designer edits).

### Migration

No data migration. PMs can keep using `/design-prompts` standalone; `/push-to-figma` is purely additive. Existing features can run `/push-to-figma` against their existing prompts file at any time.

---

## v2.11.0 — 2026-05-22

### Added

- **`/exec-summary` standalone command.** Synthesizes a feature's PRD + system design + user-stories breakdown + timeline + decision log into a single ~20 KB / 5–7 PDF page executive summary. Fixed 9-section structure: masthead, vision, use cases table, moving pieces (external vendors / internal Bright systems / cross-cutting work), teams & roles, critical timeline alignment, open decisions, risks, the ask, what's next. Outputs markdown + PDF + DOCX side-by-side in the feature's root folder. New file: `subprompts/exec-summary.md`.
- **Idempotent overwrite.** Re-running on the same feature overwrites all three output files. No version history embedded inline; change history lives in `changelog/[feature]-changelog.md`.
- **Graceful degradation.** PRD is required; system design / user stories / timeline / decision log are opportunistic. Missing artifacts produce a thinner section with `[NOTE: ...]` rather than failing the run. PDF/DOCX tooling is best-effort — markdown always lands.
- **Standalone only.** Not auto-invoked by `/build-product`. PMs run on demand. Pipeline state's `artifacts.exec_summary` block is updated on each run with `path`, `pdf_path`, `docx_path`, `generated_against_prd_version`, `generated_at`.

### Changed

- **`ai-framework/style-preferences.md` — new "Artifact Conventions" section.** Generated artifacts must not embed inline version-history blocks. Version number lives in the title line; full change history lives in `changelog/[feature]-changelog.md`. Decision logs, content-level product version concepts (e.g., "v1 ships in October"), and cross-artifact source pointers are explicitly distinguished from this rule and remain in artifacts.
- **`ai-framework/05-change-propagation.md` Step 4 Writing — reinforcing rule.** Future `/change-mode` runs must not append `**v0.X (date):** ...` bullets to artifacts during propagation. The changelog entry written by Step 6 of the pass is the single audit-trail record per change-mode run.

### Why this matters

PMs (Judy on nestfully-ai) flagged that downstream artifacts were becoming unreadable as `/change-mode` passes layered inline version bullets onto the same files. The same change history was being recorded in three places (the artifact, the `changelog/` folder, and pipeline state); two of those was redundant. The Artifact Conventions rule eliminates the redundancy and the `/exec-summary` skill gives PMs a way to produce an exec-readable synthesis without manually re-creating a format from scratch every time.

### Migration

No data migration needed. The rule applies prospectively to all generative skills and to future `/change-mode` runs. Retroactive cleanup of an existing feature's inline changelogs is a manual edit (or use `/change-mode` with the cleanup framed as a scope_change). The skill itself does not auto-clean prior artifacts.

---

## v2.10.0 — 2026-05-22

### Added

- **Wave sequencing in the User Stories Breakdown.** `ai-framework/06-user-stories.md` gains a new **Step 2.5 — Wave Sequencing** between Step 2 (Build Sequence Map) and Step 3 (Per-story sections). The step:
  - **Computes a Wave assignment** for every story via topological sort on the `Depends On` column. Wave 1 = stories with no dependencies; Wave N = stories whose dependencies are all in Waves 1..N-1.
  - **Wave numbering is global** (W1, W2, ... W[N]) — not reset per phase or epic. Phase ordering is respected as an implicit constraint.
  - **FE/BE pairs are NOT treated as hard dependencies** by default (they commonly land in the same wave). Pair links live in the `Related To` column, not `Depends On`. Override only when an AC explicitly requires the BE deployed before the FE.
  - **Cycle detection**: if the dependency graph contains a cycle, the step surfaces it as a CRITICAL Gate 3 finding with the US-IDs in the cycle, and refuses to advance until the PM resolves.
- **Build Sequence Map gains a Wave column.** Column order: `US-ID | Title | Type | Epic | Phase | Wave | Depends On | Related To | Size | DRAFT?`.
- **New Wave Summary section** in the breakdown file — one row per wave with theme, story_ids, and annotations for the **critical convergence wave** (a wave whose completion unlocks a large downstream block) and the **launch gate wave** (the last wave before release).
- **State schema adds `user_stories.waves[]`** — one entry per wave with `wave`, `theme`, `story_ids[]`, `critical_convergence`, `launch_gate` flags.
- **Two new Gate 3 quality checks** added to `pipeline-configs.yaml`:
  - `every_story_has_wave` (WARNING) — every story must have a wave assignment.
  - `dependency_graph_acyclic` (CRITICAL when a cycle is found) — flags cycles with the US-IDs involved.

### Why this matters

`/validate-user-stories` Check 7 ("Wave / dependency sanity") was added in v2.9.0 expecting waves to exist — but the creation step (`/user-stories`) didn't actually produce them. PMs (like Judy on nestfully-ai) had been adding waves manually. v2.10.0 closes that loop: waves are computed automatically, persisted to state, and validated at Gate 3 + by `/validate-user-stories`.

The cycle-detection check is the most impactful piece. Cycles in the dependency graph are real bugs — they mean engineering can never start because every starting point depends on something else. Catching them at Gate 3 (before Jira export) saves a sprint of confusion.

### Not changed

- Existing user_stories.epics[] and user_stories.draft_stories[] schemas unchanged.
- Existing Gate 3 quality checks unchanged — wave checks are additive.
- `/timeline` (Step 10.5) doesn't yet read waves explicitly (it operates at Epic + Phase granularity). Future work could surface waves on the Gantt as a finer-grained view; out of scope for this release.

---

## v2.9.0 — 2026-05-22

### Added

- **`/validate-prd`** — a semantic content validator for the PRD (different from `/pipeline-doctor`, which checks structural integrity). Six checks:
  1. **Internal consistency** — sections contradict each other (data model vs. API contracts, scope vs. phases, decision log vs. body, roles referenced but not defined, NFR scope vs. API endpoints).
  2. **Hallucinated data** — numeric claims / statistics / market references without a source. Cross-checks against the research output file.
  3. **Completeness** — `[TBD]` / `[TODO]` / `[FIXME]` markers, empty required sections, single-sentence placeholders.
  4. **VOC traceability** — does the PRD use language from the research output, or did it drift to generic AI prose? Catches invented user quotes.
  5. **NFR measurability** — non-functional requirements have concrete thresholds and bounded scopes vs. vague aspirations ("fast", "scalable").
  6. **Scope coherence** — in-scope vs. out-of-scope contradictions, scope-creep references, success metrics that depend on out-of-scope capabilities.
- **`/validate-user-stories`** — a semantic content validator for the user-stories breakdown. Seven checks:
  1. **Story ↔ PRD traceability** — every story maps to a PRD section; no orphans either direction; FE/BE pair coverage.
  2. **AC duplication / contradiction across stories** — same Gherkin scenario in multiple stories, or contradicting `Then` clauses for the same screen / endpoint.
  3. **FE/BE pair coherence** — endpoint contracts, error paths, permissions, and naming match between paired stories.
  4. **AC specificity** — catches vague Gherkin ("it works", "appropriate error", "if needed") that won't survive QA. DRAFT stories exempt.
  5. **Sizing sanity** — scenario count + distinct surface count vs. size label (S/M/L/L+). Catches under-sized and over-sized stories. DRAFT stories' `*` sizing gets gentle treatment.
  6. **DRAFT consistency** — state and breakdown agree on which stories are DRAFT; markers (`Status: ⚠ DRAFT`, `*` sizing, design-gaps list) are present where expected.
  7. **Wave / dependency sanity** — acyclic dependencies, wave ordering (no story depends on later-wave work), phase ordering, FE→BE direction sanity.
- **Same UX as `/pipeline-doctor`** — inline summary → per-finding walkthrough with approve/skip → full markdown report saved to `[feature]/validation/[feature]-validate-{prd|stories}-[YYYY-MM-DD-HHMM].md`. A `_validate-{prd|stories}-history.md` log accumulates one-line entries per run.

### Why this matters

`/pipeline-doctor` catches structural drift (missing files, broken state schemas, stale features). The new validators catch **content drift** — the PRD that no longer agrees with itself after 17 review-and-fix cycles; the 75-story breakdown where Story 2.4 and Story 3.1 specify different empty-state copy for the same screen; the L-sized story that's actually 3 trivial scenarios because the PM rushed it. These are invisible to mechanical gate quality checks and to skim-reading. At 75 stories / 11,220 lines, a PM cannot catch them by hand.

### Cost note

`/validate-user-stories` is the most expensive command in the skill. On the nestfully-ai 614KB breakdown, expect 1–3 minutes of runtime and significant token consumption. Run on-demand, not auto-run at gates.

### Not changed

- Existing pipeline gates (1, 2, 3) and their mechanical quality checks remain. Validators are additive on-demand checks, not replacements.
- All existing commands (`/pipeline-doctor`, `/change-mode`, `/build-product`, etc.) unchanged.

---

## v2.8.1 — 2026-05-22

### Fixed

- **DRAFT-mode Gate 2 now records a distinct "Deferred" state instead of "Pending".** Pre-v2.8.1, when the PM proceeded past Step 9 in DRAFT mode (no finalized designs), `gates.gate_2` was left as `"Pending"` — indistinguishable from "haven't reached Gate 2 yet" in state, which caused the `/pipeline-doctor` B4 check to falsely flag the feature as state-corrupted. The orchestrator now writes `gates.gate_2 = "Deferred — DRAFT mode (no finalized designs as of YYYY-MM-DD)"` instead. Gate 2 will still be formally approved later when designs arrive (via re-running Step 9 or via `/change-mode` → "Designs arrived").
- **`/pipeline-doctor` B4 check updated** to recognize the deferred pattern. If `current_step` is past Step 9 AND `user_stories.mode == "DRAFT"` (or `"MIXED"`) AND `artifacts.design_catalogs` is empty, the doctor treats `gate_2 = "Pending"` as a deferred-but-not-yet-recorded state and proposes updating it to the explicit `"Deferred — DRAFT mode (...)"` string (INFO, not WARNING). Mismatches outside this pattern still WARN.
- **`SKILL.md` state schema documentation** updated — `gates.gate_2` now lists `"Deferred — DRAFT mode (...)"` as a valid value alongside `"Approved YYYY-MM-DD"`, `"Pending"`, and `"N/A"`.

### State files affected

- `nestfully-ai` had `gate_2 = "Pending"` while at Step 10. Retroactively updated to `gate_2 = "Deferred — DRAFT mode (no finalized designs as of 2026-05-22)"` to match the new semantics. Backup saved to `_pipeline-state.json.bak-before-gate2-fix-20260522` in the feature folder.

### Why this matters

The doctor surfaced this as a state-corruption warning on `nestfully-ai`, but the state was actually internally consistent — it just used an ambiguous value. This patch makes the deferred-Gate-2 flow explicit in the data so it's distinguishable from unstarted-Gate-2 going forward.

---

## v2.8.0 — 2026-05-22

### Changed

- **Confluence publish scope tightened.** Three artifacts are no longer published as their own Confluence child pages:
  - **Step 4a: Product Review** — internal review artifact, kept local in `product-review/`.
  - **Step 4b: Technical Review** — internal review artifact, kept local in `technical-review/`.
  - **Full User Stories Breakdown** (with Gherkin AC) — too large for Confluence; the full doc lives at `user-stories/[feature]-user-stories.md` and is also attached to each Jira Epic.
- **Step 10: User Stories is now a lightweight Jira-index page** instead of the full breakdown. The page contains:
  - A brief explanation that the full breakdown is local + attached to each Jira Epic.
  - One section per Epic with title, phase, theme, story count (FE/BE split), and Jira Epic URL.
  - Story titles listed under each Epic (no Gherkin AC, no testing notes, no per-story detail).
  - Composed from `_pipeline-state.json` → `user_stories.epics[]`, not from the breakdown file directly. The breakdown file's mtime is still used for change detection (so editing the breakdown triggers a re-publish of the index).
- **Parent hub page gains a "Kept local — not published" subsection** listing the three excluded artifacts with their local paths, so stakeholders can see they exist without being able to click through.

### Why this matters

PMs in stakeholder meetings get the page that's actually useful — a navigable index of the work tracked in Jira — without Confluence being polluted with 50+ pages of Gherkin scenarios that stakeholders don't read. Internal review artifacts (Product/Technical Review) stay where they belong: in the local workspace and in the PRD's decision log when their findings ended up shaping the spec.

### Migration

- **Existing pre-v2.8.0 Confluence pages for the three excluded artifacts are NOT deleted automatically.** This preserves any stakeholder bookmarks. If you want to delete them after upgrading, do so manually in Confluence.
- **Legacy state entries** (`confluence_hub.artifacts.step_4a_product_review` etc.) from pre-v2.8.0 runs are ignored on subsequent publishes — no errors, just skipped.
- The first `/publish-to-confluence` run after upgrading will replace the previously-published full User Stories Breakdown page (at the same URL — `updateConfluencePage`) with the new lightweight Jira-index content. Page URL preserved; content reflects the new format.

### Not changed

- Per-file mtime change detection still works the same way (only changed pages re-publish).
- Parent hub page format, the per-Epic publishes for multi-epic features, the Figma embeds for Visual Diagram and Timeline pages, the migration path for pre-v2.3.0 single-PRD pages — all unchanged.
- The PRD attachment per-Epic in Jira is unchanged. Stakeholders who want the full Gherkin can open the Jira Epic and download the attached PRD + breakdown.

---

## v2.7.0 — 2026-05-22

### Added

- **`/pipeline-doctor`** — a new diagnostic command that scans the skill and feature workspaces for drift, inconsistencies, and stalls. Four check categories:
  - **(A) Skill self-consistency** — every step in `pipeline-configs.yaml` has matching prose in `SKILL.md` and `subprompts/build-product.md` (A3 — the check that catches the bug Judy hit where Step 10.5 was in the config but missing from the orchestrator); every gate's `quality_checks[]` entries are defined in the top-level `quality_checks:` section; every step's `instruction:` file path actually resolves; every step block has an explicit "Next:" handoff (A5).
  - **(B) Feature-state consistency** — `_pipeline-state.json` parses and has required keys; each artifact entry with a `path` actually exists on disk; gate states are coherent with `current_step` (e.g., past Gate 2 means Gate 1 + 2 must be Approved); DRAFT stories list in state matches what's actually marked DRAFT in the breakdown; Confluence hub artifacts have sensible mtimes; timeline.applied_edits[] is consistent with timeline.computed.
  - **(C) Slash command coverage** — every `subprompts/*.md` has a registered `~/.claude/commands/[name].md`; every `~/.claude/commands/*.md` that references this skill points at a real file; the orchestrator (`/build-product`) itself is registered.
  - **(D) Stale features** — pipelines started >30 days ago without completion, state files unchanged for 14+ days, orphaned feature folders with no `_pipeline-state.json`.
- **Per-finding fix approval.** After the scan, the doctor walks through CRITICAL/WARNING findings one at a time, proposes a concrete fix per finding, and asks the PM to approve, skip, or see file context. Fixes are never applied silently.
- **Timestamped reports.** Every run writes a full markdown report to `~/Desktop/Resources/PDLC Workflow Docs/_pipeline-doctor-report-[YYYY-MM-DD-HHMM].md` and appends a one-line entry to `_pipeline-doctor-history.md` at the workspace root.

### Why this matters

The Step 10.5 / Step 12 orchestrator-drift bug Judy hit (fixed in the prior commit) was exactly the class of failure the doctor would have caught: `pipeline-configs.yaml` listed Step 10.5 (added in v2.2.0) but `subprompts/build-product.md` never got the matching prose block, so the orchestrator silently didn't know it existed. As the skill keeps growing — more steps, more commands, more state schema additions — this kind of drift becomes inevitable. Doctor is the safety net.

### Not changed

- Doctor is **read-only by default**. It doesn't modify any file until the PM approves a specific fix.
- No external network calls. Doctor only reads local files; it doesn't query Jira, Confluence, Figma, or any MCP. (A potential future `/pipeline-doctor --remote` could verify external resources, but not in v1.)
- Idempotent. Two runs in a row produce the same findings (no state changes between runs except the timestamped report file).

---

## v2.6.0 — 2026-05-21

### Added

- **💾 Save to skill button in the HTML Gantt** (Chrome / Edge / Chromium-based browsers). Uses the File System Access API (`window.showSaveFilePicker`, `FileSystemFileHandle`) to write the plan JSON directly into the feature's `timeline/` folder — no Downloads round-trip. First click pops a native save dialog; the file handle is persisted to IndexedDB so subsequent clicks save silently to the same file. Permission re-requests gracefully if revoked. Safari and Firefox don't support the API; in those browsers the button auto-relabels to "💾 Save (download)" and falls back to the existing download flow.
- **Auto-discovery on `/timeline apply`.** The command now accepts zero arguments and scans for plan JSON files in this order:
  1. **Feature's own `timeline/` folder** (where Save-to-skill writes by default): `[feature]/timeline/*-plan-*.json` sorted by mtime descending.
  2. **Downloads folder** (where Export Plan writes): `~/Downloads/[feature]-plan-*.json` sorted by mtime descending, filtered by feature-name prefix.
  - If exactly one candidate, confirms with a one-line prompt. If multiple, lists them with timestamps. If none, asks for an inline paste or explicit path.

### Why this matters

The round-trip from browser to skill was three steps (Export Plan → find the file → paste path to chat). With both changes, the typical flow becomes:

- **Chrome/Edge users:** edit → click 💾 Save → type `/timeline apply`. Three steps with no path-typing or file-hunting.
- **Safari/Firefox users:** edit → click ⬇ Export plan → type `/timeline apply`. Auto-discovery picks up the latest plan from `~/Downloads/` automatically.

### Changed

- `ai-framework/06b-timeline.md` Step 6 (HTML JS) extended with the FSA spec — feature detection, first-click vs. subsequent-click flows, IndexedDB handle persistence, permission re-request, error handling, and the Safari/Firefox download fallback.
- `ai-framework/06b-timeline.md` Apply Mode Step A-1 extended with the three input paths (explicit path, inline paste, auto-discovery) and the multi-candidate selection UX.
- `subprompts/timeline.md` workflow section updated with the three round-trip options (fastest / cross-browser / explicit) so PMs can pick whichever fits the moment.

### Not changed

- The HTML remains a single self-contained file with no CDN dependency. The FSA API code is inline.
- localStorage auto-save still works — Save-to-skill is additive, not a replacement.
- The `build-product-timeline-plan-v1` schema is unchanged. Plans saved by v2.5.0 Export Plan are accepted by v2.6.0 apply unchanged.

---

## v2.5.0 — 2026-05-21

### Added

- **Interactive editing in the HTML Gantt.** The `timeline/[feature]-timeline.html` file is now editable in the browser:
  - **Drag any bar** to shift its start date. Downstream bars (anything that originally started after this bar's original end) cascade by the same amount automatically.
  - **Drag the right edge** of a bar to resize its duration. Same cascade behavior.
  - **Hold Shift while dragging** to lock other bars — only the dragged bar moves; the rest stay in place (may now overlap, intentionally).
  - **Arrow-key edits**: focus a bar (Tab) and use `←` / `→` to shift by 1 working day, or Shift+arrow to resize. Alt suppresses the cascade.
  - **Auto-saved to localStorage** on every edit, keyed by feature name. Reload preserves edits.
  - **Toolbar buttons**: `↺ Reset to original` (discards edits, restores skill-computed baseline; confirms first) and `⬇ Export plan` (downloads a JSON plan).
  - Still **vanilla JS, no CDN, opens offline.** The HTML remains a single self-contained file.
- **`/timeline apply [path]` — round-trip edits back into the skill.** A new mode of the existing `/timeline` command that promotes browser edits into the official skill state:
  - Reads the exported JSON (path or pasted content) and validates against `schema: "build-product-timeline-plan-v1"`.
  - Computes new calendar dates from the edited working-day offsets, skipping weekends and any off-time ranges from `timeline.parameters.off_time`.
  - Shows a per-epic diff (old date → new date, span delta) and an overall feature-end shift, with a target-gap warning if the new end exceeds the target launch date.
  - On approval: updates `timeline.computed` and `timeline.parameters` (per-epic durations), logs the apply event to a new `timeline.applied_edits[]` history, re-renders the markdown sidecar with the applied dates as the new baseline, and re-renders the HTML so its `data-start` / `data-span` reflect the edits (localStorage cleared, edits become ground truth for future runs).
  - **Idempotent** — applying the same JSON twice is a detected no-op.
  - **Schema-gated** — refuses any JSON that isn't a build-product-timeline plan.
- **`timeline.applied_edits[]` history** added to `_pipeline-state.json` so subsequent `/change-mode`, `/timeline`, and `/publish-to-confluence` runs can see when the plan was edited and what shifted.

### Changed

- `ai-framework/06b-timeline.md` Step 6 (HTML generation) substantially expanded with the interactive spec — data model, drag/resize/cascade rules, accessibility (keyboard editing, ARIA), localStorage persistence, JSON export shape.
- `ai-framework/06b-timeline.md` adds a new Apply Mode section (Steps A-1 through A-8) covering the round-trip flow.
- `subprompts/timeline.md` adds an "Editing the timeline interactively" section walking the PM through the drag/cascade/export/apply flow.
- `_pipeline-state.json` schema in SKILL.md gains a documented `timeline.applied_edits[]` array and a fully-described `timeline.parameters` / `timeline.computed` block (these were previously written but not formally schema-documented).
- After `/timeline apply`, if Confluence has been published for the feature, the skill reminds the PM to run `/publish-to-confluence` to refresh the Timeline child page (does not auto-republish — PM decides when to share).

### Why this matters

The Gantt stops being a read-only artifact and becomes an actual planning tool. PMs can negotiate dates with stakeholders live in a meeting, see ripple effects immediately ("if Epic 2 slips by a week, where does that put launch?"), and then promote the agreed plan back into the skill's state in one paste — without leaving the artifact, without re-running the pipeline, without re-typing dates into a spreadsheet.

### Not changed

- User stories breakdown, Jira tickets, PRD, system design, design catalog, codebase review, research, reviews — none of these are touched by Apply Mode. Scope changes still require `/change-mode`. The Timeline governs "when", not "what".
- The Figma FigJam timeline is regenerated only by a normal `/timeline` run, not by `/timeline apply`. Edits round-trip into local files + state; refresh Figma manually if you want it synced.

---

## v2.4.0 — 2026-05-21

### Added

- **Multi-epic support in the User Stories Breakdown.** Step 10 now proposes an Epic grouping for the PM at a new sub-step (Step 1.5 in `ai-framework/06-user-stories.md`):
  - **Default heuristic:** one Epic per PRD phase, with sub-epics for clear functional clusters within a phase.
  - **PM acceptance / tuning:** PM can merge, split, rename, or move stories between epics before the breakdown is composed.
  - **Persisted to state:** the accepted grouping lives in `_pipeline-state.json` → `user_stories.epics[]`, each entry with `epic_id`, `title`, `phase`, `theme`, `story_ids`, and (after export) `jira_key`.
  - **Step 11a Jira Export creates one Jira Epic per group** instead of a single Epic per feature. Each Epic's description is scoped to its own stories (not the entire PRD). The PRD + User Stories Breakdown files are attached to every Epic so each is self-contained. Existing-Epic detection runs per group so re-runs reuse the right Epics.
  - **Single-Epic fallback** preserved for pre-v2.4.0 breakdowns without `user_stories.epics[]`.
- **DRAFT mode for stories without finalized designs.** Step 10 begins with a new design-availability check (Step 0.5):
  - If no finalized designs exist, the PM picks: **wait** (pause the pipeline), **write anyway in DRAFT mode**, or **cancel**.
  - In DRAFT mode, design-dependent stories get `Status: ⚠ DRAFT — needs design` at the top, sized with `*` suffix (e.g., `M*`), and a `Known design gaps` block listing the deferred items.
  - Gate 3 quality checks exempt DRAFT stories from UX state coverage requirements and ≥2-scenario requirements (counted as known gaps in a Gate 3 info block, not failures).
  - Each DRAFT story is recorded in `_pipeline-state.json` → `user_stories.draft_stories[]` with `us_id`, `epic_id`, and `reason`.
  - Step 11a Jira Export adds a `draft` label and a `⚠ DRAFT — needs design refresh` note in the Description for every DRAFT story.
  - **MIXED mode** is automatic when only some phases have designs — full mode for phases with designs, DRAFT mode for phases without.
- **New `/change-mode` trigger: "Designs arrived"** (seventh trigger type). When finalized designs land for a feature that has DRAFT stories:
  - Reads `user_stories.draft_stories[]` and the new design catalog.
  - Walks the PM through each DRAFT story with the design content in context, refreshing AC (full Gherkin including UX state coverage), sizing (removes `*` suffix), and Testing Notes.
  - Updates the corresponding Jira ticket via `updateJiraIssue` — refreshed AC, refreshed sizing label, **`draft` label removed**, DRAFT note removed from Description.
  - Skips the broader artifact-propagation order (PRD, system design, etc.) unless the PM says the designs imply scope changes.
  - When the last DRAFT is cleared, `user_stories.mode` flips back to `full` and `user_stories.draft_resolved_at` is set.
- **Step 9 (Update PRD from Designs) now branches on design availability.** If the design catalog exists, runs as before (sync PRD + Gate 2 design-approval). If not, skips the PRD-from-designs sync; Gate 2 becomes a quick "proceed to user stories without finalized designs?" confirmation; `user_stories.mode` is pre-set to `DRAFT`.

### Changed

- `_pipeline-state.json` schema: `user_stories` is now an object (was previously `{ "path": ..., "size_bytes": ... }`). New fields: `mode`, `epics[]`, `draft_stories[]`, `draft_resolved_at`. The `path` and `size_bytes` fields are preserved for backward compatibility.
- `ai-framework/pipeline-configs.yaml`: Gate 3 quality_checks now includes `every_story_has_epic`. Existing checks (`every_story_has_two_scenarios`, `every_story_has_edge_case`, `ux_state_coverage_per_fe_story`) updated to note DRAFT exemptions.
- `ai-framework/05-change-propagation.md`: seventh trigger type added, with focused propagation for the DRAFT-story refresh case.

### Why this matters

Two real-world frictions removed in one release. (1) PMs no longer have to wait on design to write tickets — they can lock in the BE work, behavioral FE work, and ticket structure early, and refresh design-dependent details later in one batched `/change-mode` run. (2) Larger features no longer pile every story under one Epic — they get a natural Epic grouping that matches how engineering and stakeholders actually consume work.

### Not changed

- Gates 1–3 banner formats, parallel Dual Review block, parallel Step 11 Export block.
- Other intake parameters (feature name, Jira project, tech stack, product type, permission model, backend/API surface, Jira ticket conventions).
- Confluence hub, transcript export, pipeline timing (all from v2.3.0) are unaffected.

---

## v2.3.0 — 2026-05-21

### Added
- **Pipeline timing instrumentation + `/pipeline-timing` report.** Every pipeline step now records `started_at` and `completed_at` to `_pipeline-state.json` → `step_timings[step_id]`; gate steps additionally record `presented_at` and `approved_at` (so we know how long the PM took to approve each gate). The new `/pipeline-timing` standalone command — and the timing block automatically included in the Step 12 transcript and the Confluence parent hub — reports:
  - **Wall-clock total** = `pipeline_completed_at - pipeline_started_at` (includes all gate waits, breaks, overnight pauses).
  - **Active-work total** = wall-clock minus the sum of all gate-wait times (what the model was actually working on).
  - **Per-step breakdown** with each step's duration and notes on which parts were gate wait.
  - **Per-gate table** showing how long each of Gate 1 / 2 / 3 sat waiting on PM approval.
  - If a step is missing instrumented timestamps (older runs or interrupted instrumentation), the report falls back to parsing the session JSONL for that step and marks the inferred row with `~`. Both data sources can be mixed in a single report.
- **State schema additions:** top-level `pipeline_started_at`, `pipeline_completed_at`, `step_timings` dict, and `timing_report` cache (last generated totals so downstream consumers don't have to re-run the report).
- **Step 12: Export Conversation Transcript** — new pipeline step that runs automatically at the end of every `/build-product` run, and is also callable standalone via `/export-transcript`. Reads the current Claude Code session's JSONL file (`~/.claude/projects/-Users-judydarvin/[session-uuid].jsonl`), filters to messages within the feature's pipeline window (using `_pipeline-state.json` → `pipeline_started_at` as the lower bound), and writes two markdown files to `[feature]/transcript/`:
  - `[feature]-transcript-clean.md` — user messages + assistant text only, the readable back-and-forth.
  - `[feature]-transcript-full.md` — same conversation plus tool calls and tool results (truncated to first 10 + last 5 lines per result, with collapsible `<details>` blocks). System reminders and permission-mode events included for forensic debugging.
  - The model's `thinking` blocks are excluded from both files (private reasoning, never part of the visible conversation).
  - Prior exports are timestamp-suffix-renamed rather than overwritten, so every run is preserved.
- **Confluence Publish now publishes the entire feature workspace, not just the PRD.** Step 11c (and the standalone command, now also callable as `/publish-to-confluence`) creates a parent **feature hub** page with one numbered child page per artifact:
  - `Step 1: Research`
  - `Step 2: Codebase Review`
  - `Step 3: PRD`
  - `Step 4a: Product Review` · `Step 4b: Technical Review`
  - `Step 6: System Design` (only if generated)
  - `Step 7: Visual Diagram` (Figma iframe embed when available)
  - `Step 8: Design Catalog — Phase [N]` (one page per phase file)
  - `Step 10: User Stories Breakdown`
  - `Step 10½: Timeline` (Figma embed + link to the local HTML Gantt)
  - The parent hub holds owner, pipeline status, Jira Epic + Drive links, decision log summary, open questions, and a table of all child pages with their status.
- **Per-file mtime change detection.** Each artifact's source-file modification time is compared to the last published mtime in `_pipeline-state.json` → `confluence_hub.artifacts.[key].source_mtime`. Only artifacts whose source actually changed are re-published; unchanged artifacts are skipped (existing page version preserved). The parent hub is always updated to keep status fresh.
- **New `/publish-to-confluence` slash command** registered at `~/.claude/commands/publish-to-confluence.md`. The older `/prd-to-confluence` slash command and its underlying `subprompts/prd-to-confluence.md` file were **removed** in this release — the command publishes nine artifacts, not just the PRD, so the old name was actively misleading. The subprompt file was renamed to `subprompts/publish-to-confluence.md`.
- **Legacy migration.** Features with a pre-v2.3.0 single-PRD Confluence page (recorded as `export_urls.confluence_page`) are offered a one-time migration on the next publish run: the legacy page is reparented under the new hub as `Step 3: PRD`, preserving its URL so existing bookmarks keep working.

### Changed
- `subprompts/read-feedback.md` now defaults to scanning **every** child page in `confluence_hub.artifacts` when the PM chooses "scan everything", and groups comments by source page (so PRD comments edit the PRD file, design comments edit the design catalog, etc.). Legacy single-PRD pages still work via a Path C fallback.
- `subprompts/share-for-review.md` defaults to sharing the parent hub URL ("reviewers see the full picture and can comment on any page") and lets the PM narrow to one specific child page when needed.
- The Confluence publish subprompt was substantially rewritten and renamed from `subprompts/prd-to-confluence.md` → `subprompts/publish-to-confluence.md` to match the new behavior (hub + children, not just PRD).
- `_pipeline-state.json` schema: added `confluence_hub` (space, parent ID/URL, last_published_at, per-artifact records with page_id / page_url / source_mtime / last_published_at). Added `export_urls.confluence_hub` and `export_urls.figma_timeline_url`. `export_urls.confluence_page` is now marked DEPRECATED but still set, pointing at the `Step 3: PRD` child page for backward-compat with anything still reading that field.

### Why this matters
A stakeholder opening a feature in Confluence now sees the entire product record — research, reviews, designs, user stories, the Gantt — not just the PRD. They can comment on any step (e.g., a tech lead can comment on the codebase review directly), and `/read-feedback` will pull those comments back to the correct source file. PMs no longer have to manually export research / reviews / designs to a separate Confluence page; one publish step does it all.

### Not changed
- Jira ticket creation, Drive sync, Figma diagram + timeline generation, all gates, all intake parameters.
- Existing Confluence page URLs from prior pipeline runs are preserved across the migration to the hub model.

---

## v2.2.0 — 2026-05-21

### Added
- **`/timeline` — new pipeline step and standalone command.** Produces a Gantt-style timeline at the Epic + Phase level after Gate 3 (User Stories Breakdown approval) and before Step 11 (Export). Two outputs per run:
  - **Figma FigJam timeline** via the Figma MCP `generate_diagram` call. URL stored in `_pipeline-state.json` → `export_urls.figma_timeline_url`. If Figma MCP is unavailable, the step skips this output cleanly and notes the skip in the sidecar.
  - **Interactive HTML Gantt** — self-contained HTML file (no external dependencies, opens offline) at `timeline/[feature-name]-timeline.html`. Per-bar hover details, today marker, optional target-launch marker, print stylesheet.
- **Hybrid duration estimation.** The skill computes proposed durations from the user-stories breakdown's `Size` column × default velocity (8 person-days per dev per sprint), then the PM tunes any epic or phase before the visuals are rendered. Recompute loop until the PM accepts.
- **Honest target-gap math.** If the PM supplies a target launch date and the computed end exceeds it, the step surfaces the gap and offers scope cut / team increase / slip — it does not silently shrink durations to fit.
- **Step 10.5 wired into `pipeline-configs.yaml`** (id: `timeline`, mode: auto, no gate). The orchestrator runs it automatically inside `/build-product`; the same prompt is callable standalone via `/timeline`.
- New files: `ai-framework/06b-timeline.md` (core instructions), `subprompts/timeline.md` (standalone wrapper). New artifact subfolder: `timeline/`.

### Changed
- Pipeline step count updated from 11 to 12 in README and `docs/index.html`.
- `_pipeline-state.json` schema: added `timeline.parameters`, `timeline.computed`, `timeline.outputs.html_path`, `timeline.outputs.markdown_path`, and `export_urls.figma_timeline_url`.

### Why this matters
Commit-quality roadmaps stop being a separate spreadsheet exercise. The same Sized stories that drive Jira ticket creation now drive a Gantt without re-entering anything, and the visuals (Figma FigJam for stakeholder review, HTML Gantt for offline / Confluence embed) match the rest of the pipeline's outputs.

### Not changed
- Gates 1–3, parallel Dual Review, parallel Step 11 Export, all intake parameters, and the user-stories breakdown format.

---

## v2.1.0 — 2026-05-20

### Added
- **Open-ended Jira ticket conventions probe at intake.** Intake question #3 used to ask only for labels. It now asks an open-ended question with concrete examples covering: labels, title format (verb-first, `[BE]`/`[FE]` prefix, Epic naming), BE/FE split rule, custom field defaults (e.g., Testable = Yes/No), fields to leave blank (e.g., Story Points), link conventions (Blocked by, Relates to). The PM answers in free text. (`CLAUDE.md`, `subprompts/build-product.md`)
- **Explicit Stage 0.5 — Intake** in `subprompts/build-product.md`. Walks through all 7 intake questions and persists them to `_pipeline-state.json` → `intake` before Step 1 begins. Also offers to reuse prior intake from another feature in the same workspace.
- **Conventions are applied automatically downstream.** `subprompts/prd-to-jira.md` and `ai-framework/06-user-stories.md` now read `intake.jira_ticket_conventions` and apply title format, BE/FE split rule, custom field defaults, fields-to-omit, and link conventions at the right step. The PM no longer has to specify these per-run or rely on a personal CLAUDE.md to capture them.

### Why this matters
A new team installing the skill no longer has to write their own home CLAUDE.md to encode their Jira conventions — the skill probes for them at intake and applies them automatically. Personal conventions stop being a quiet prerequisite.

### Not changed
- The 7 default intake questions otherwise (feature name, Jira project, tech stack, product type, permission model, backend/API surface).
- All other pipeline behavior, including Figma FigJam diagrams, Figma Make prompts, Gates 1–3, and Step 11 parallel export.

---

## v2.0.0 — 2026-05-20

### Breaking changes
- **Removed** Personal pipeline (Full / Medium / Light). The skill is now Work-only — planning and design for PMs, no implementation.
- **Removed** commands: `/design`, `/design-with-v0`, `/execute-plan`, `/validate`, `/update-prd-from-build`, `/generate-tests`, `/learn`, `/review-parallel`, `/validate-parallel`
- **Removed** files: `ai-framework/03-design.md`, `ai-framework/04-execute-plan.md`, `ai-framework/04b-update-prd-from-build.md`, `subprompts/design.md`, `subprompts/design-with-v0.md`, `subprompts/execute-plan.md`, `subprompts/validate.md`, `subprompts/update-prd-from-build.md`, `subprompts/review-and-fix.md`, `subprompts/learn.md`, `subprompts/generate-tests.md`

### Added
- **`/design-prompts`** (`subprompts/design-prompts.md`) — Tool-agnostic screen design prompt generator. The same structured prompt works in both v0 and Figma Make. Figma Make gets an optional "Component references" block for design system integration.
- **Figma MCP as primary diagram format** in `/visual-diagram` — calls `generate_diagram` via Figma MCP to create a shareable FigJam diagram. URL is stored in `_pipeline-state.json` → `export_urls.figma_diagram_url`. Mermaid is kept as a fallback only.

### Fixed
- `subprompts/prd-to-confluence.md` incorrectly stated "Confluence renders Mermaid natively." Corrected: Mermaid does not render in Confluence without a third-party plugin. Diagram section now uses the Figma iframe embed URL format (`figma.com/embed`) when `figma_diagram_url` is available; omits diagram embed otherwise.

### Changed
- `_pipeline-state.json` schema: added `export_urls.figma_diagram_url`; removed `pr_urls`, `validations`, `learnings` (Work pipeline never produces these).
- Work pipeline Step 8 asks "v0 or Figma Make?" — both use the same prompt structure.
- `ai-framework/pipeline-configs.yaml`: removed `personal_full`, `personal_medium`, `personal_light` pipeline entries.

---

## v1.2.2 — 2026-05-20

### Fixed
- Share-for-review was offered at Gates 1 and 2 before Confluence was published. Moved offer to Gate 3 only, where a Confluence page is guaranteed to exist.

---

## v1.2.1 — 2026-05-20

### Fixed
- `/share-for-review` attempted to share a Confluence URL before `Step 11c` (Confluence publish) had run. Corrected sequencing: publish to Confluence first, then share the link.

---

## v1.2.0 — 2026-05-20

### Added
- **`/share-for-review`** (`subprompts/share-for-review.md`) — Posts a Confluence artifact link to a Slack channel with tagged reviewers and a deadline. Detects Slack MCP at runtime; falls back to a formatted message for manual paste if Slack MCP is unavailable.
- **`/read-feedback`** (`subprompts/read-feedback.md`) — Pulls inline and footer comments from a Confluence page, synthesizes them into suggested PRD edits, applies PM-approved changes, and re-syncs PRD to Confluence.
- **`/lint-style`** (`subprompts/lint-style.md`) — Mechanical style-guide checker. Reads `ai-framework/style-preferences.md` and flags or auto-fixes violations in any generated document.
- **Optional review lenses** — Security, Accessibility, Data Privacy, and AI Safety reviewer personas added to `ai-framework/personas.md`. PM activates lenses at pipeline intake; they compose into the dual-review parallel block.
- **`ai-framework/pipeline-configs.yaml`** — Declarative pipeline config. Steps, gates, quality-check IDs, and conditions defined in YAML rather than hardcoded in prose. Orchestrator derives step logic from config.

---

## v1.1.0 — 2026-05-19

### Improved
- **True parallel review** — Dual review now uses the Agent tool (two isolated sub-agents with self-contained prompts). Previously, roles were switched within a single context window.
- **Resumable state** — Pipeline state moved from `_pipeline-state.md` to `_pipeline-state.json`. Schema includes `size_bytes` per artifact for session-resumption integrity verification.
- **Context checkpoints** — `_context-checkpoint.md` written after every gate. Sessions resume by reading the checkpoint rather than re-loading all artifacts.
- **Conditional gates** — `_open-conditions.md` enables "approved with conditions" advancement. Conditions are verified at the next gate's quality check before advancing.
- **Knowledge base seeding** — Research step (Stage 0) now queries `_knowledge-base.md` before running web search, pre-populating the PRD's Decision Log and Open Questions from past learnings.
- **Auto lint-style** — PRD creation calls `/lint-style` before saving.
- **Jira deduplication** — Manifest check + JQL query before bulk creation prevent duplicate tickets on re-runs. Pre-creation manifest enables safe retry if bulk creation is interrupted.
- **`05-parallel-rules.md`** — Upgraded with explicit Agent tool call syntax, isolated prompt requirements per agent, and optional review lens composition instructions.
- **Gate 3 quality checks** — 8 automated checks added (all PRD stories present, FE/BE pairs complete, UX state coverage, HIGH risks in testing notes, etc.).

---

## v1.0.0 — 2026-05-19

Initial public release.

### Included
- **Work pipeline** — Research → Codebase Review → PRD → Dual Review → Apply Fixes → Gate 1 → System Design → Visual Diagram → Design → Update PRD from Designs → Gate 2 → User Stories Breakdown → Gate 3 → Export
- **Personal pipelines** — Full, Medium, Light (removed in v2.0.0)
- Jira export with Epic creation, custom fields (User Story ADF, Gherkin AC, Testable, FE/BE labels, sequence + size labels), and FE/BE linking
- Optional Google Drive sync and Confluence publishing at Step 11
- `/change-mode` — safe change propagation after any gate, with diff-by-diff approval
- `/reopen-gate-1/2/3` — unwind approved gates without losing downstream work
- 15+ standalone commands
- Knowledge base (`_knowledge-base.md`) for cross-feature learning
- Resumable pipeline state with per-step file output
