v2.21.0 — 2026-05-29

Kills the front-matter plumbing junk — the `Status: v1.6 — readability reformat…` / `Predecessor:` / `Refactor authority:` / `Refactor lineage:` / `Last updated:` block that accretes at the top of docs over repeated `/change-mode` runs. v2.18.0 banned change history under `## ` headings; this closes the front-matter-disguise loophole.

**Changed: front matter is a strict allowlist (`ai-framework/06-user-stories.md` Step 5).** The user-stories doc front matter is now ≤ 8 enumerated lines (Source PRD + version, Generated date, Phases, Mode, Stories count + DRAFT, Waves, the one-line change-history pointer, optional compact Jira pointer). Everything else is forbidden — explicitly a `Status:` line carrying a version/change narrative, plus `Predecessor:`, `Refactor authority:`, `Refactor lineage:`, `Version history:`, `Last updated:`. (Bare `Status: Draft/Final`, per-story `Status: ⚠ DRAFT` markers, and gate-reopen `STATUS: DRAFT` banners are live-state and stay.)

**Changed: cross-artifact rule + change-mode guard.** `ai-framework/style-preferences.md` § Artifact Conventions now names the front-matter version/refactor fields as banned in every artifact (not just under `## ` headings). `ai-framework/05-change-propagation.md` adds an explicit "do not grow the front matter" rule — change-mode updates allowlisted values in place and never appends a new narrative line. The workspace `~/CLAUDE.md` User-Stories Layout reminder is updated to match (and its stale "Refactor history → Appendix B" line is corrected — Appendix B is a one-line pointer since v2.18.0).

**New: `/validate-user-stories` Check 8c** flags front-matter junk fields (Status-with-version-narrative, Predecessor, Refactor authority/lineage, Version history, Last updated) and over-long front matter, recommending a cut back to the allowlist.

---

v2.20.0 — 2026-05-29

Figma / FigJam is now the single deliverable surface for diagrams. The old "two visible surfaces" rule (Figma link **and** an always-on Mermaid block) is reversed — Mermaid is demoted to a temporary fallback used only when the Figma MCP is unavailable.

**Changed: Diagram Rendering policy reversed (`~/CLAUDE.md`).** Diagram docs no longer carry an always-on Mermaid block. Figma is the canonical surface (`[Open in Figma](URL)`); Mermaid appears only as a clearly-labeled temporary fallback when the Figma MCP isn't connected, and is flagged for regeneration. Rationale flipped: Mermaid doesn't render reliably across the tools the team opens (Confluence without a plugin, many viewers), and the product team works in Figma.

**Changed: `/visual-diagram` fallback framing (`ai-framework/03c-visual-diagram.md`).** When the Figma MCP is unavailable, the step tells the PM plainly, offers to let them connect Figma first, and — if they proceed — produces a Mermaid block banner-labeled `⚠ Temporary fallback`, sets `export_urls.figma_diagram_url = null` and `visual_diagram.needs_figma_regen = true` so a later run regenerates it in Figma. When Figma succeeds, the doc shows only the Figma link — no Mermaid block alongside it.

**Unchanged (already Figma/no-Mermaid):** the Timeline step (FigJam + interactive HTML Gantt), designs (v0 / Figma Make / push-to-figma), and `/system-design`'s lightweight inline ASCII sketches (which render everywhere and were never Mermaid).

---

v2.19.0 — 2026-05-29

Acceptance criteria are no longer assumed to be Gherkin — the user-stories step fit-checks the format and lets the PM choose, and the Jira export carries the chosen format plus all three story fields correctly.

**New: Step 2.7 — acceptance-criteria format fit-check (`ai-framework/06-user-stories.md`).** Before writing any AC, the skill assesses whether Gherkin actually reads clearest for this feature's stories (it fits behavioral/multi-path work; it adds ceremony to simple config/content/migration/spike stories). It tells the PM its read, shows **one real story written both ways — Gherkin vs. plain-English criteria — side by side**, and asks the PM to pick Gherkin / plain English / mixed. The choice is stored in `_pipeline-state.json` → `user_stories.ac_format` (+ `ac_format_overrides` for mixed) and carries through Step 3 authoring, validation, Gate 3, and Jira export. Honors the workspace rule "Acceptance Criteria — Gherkin or plain bullet points, whichever is clearest."

**Changed: AC is format-aware everywhere.** Story blueprints (full + DRAFT), the coverage rules, UX-state coverage, the Step 4 self-checks, Gate 3 quality checks (`pipeline-configs.yaml`, `subprompts/build-product.md`), and `/validate-user-stories` (Checks 2/4/5) now treat a "scenario/criterion" as either a Gherkin block or a plain-English bullet. New Gate-3 check `ac_format_matches_decision` flags silent format reversion.

**Changed: Jira export carries all three fields + the chosen format (`subprompts/prd-to-jira.md`).** Explicit rule that every Story populates three distinct custom fields — **Description** ← thorough plain-English Description, **Acceptance Criteria** ← AC verbatim in the breakdown's chosen format (never converted between Gherkin and plain English), **User Story** ← the "As a… I want… so that…" narrative. The AC ADF mapping now has both a Gherkin (paragraph-per-line) and a plain-English (`bulletList`) form. Fixes the prior contradiction where the Description field was told to carry only a "short" pointer.

---

v2.18.0 — 2026-05-29

Process-tracking content is fully separated from reader-facing artifacts: change/refactor history is banned inline (everywhere, by any heading) and consolidated into one per-artifact-sectioned changelog, and the decision log moves out of the PRD into a `decisions/` sidecar.

**Changed: No inline change history, enforced by name.** `ai-framework/style-preferences.md` § Artifact Conventions now forbids not just `**v0.X (date):**` blocks but any section titled `## Change log` / `## Changelog` / `## Revision history` / `## Refactor history` / `## Refactor summary` — by any wording — and any "Document version" / "Last updated" stamp line. Two leaks that survived the v2.11.0 rule are closed: the User Stories Breakdown's **Appendix B "Refactor history / changelog"** is now a one-line pointer (`06-user-stories.md`), and `/pull-from-figma` no longer adds a `## Change log` to the stories file. `/validate-user-stories` Check 8 gains sub-check **8b** that flags inline change history anywhere in the breakdown, not just above the meat.

**Changed: Centralized changelog is grouped by artifact.** `changelog/[feature]-changelog.md` is now a single feature-level file with one `## ` section per artifact (PRD, System Design, User Stories Breakdown, Design Catalog, Timeline, …), each append-only and dated. `/change-mode` Step 6 writes one dated line under each affected artifact's section, so each artifact's history reads top-to-bottom in its own section.

**Changed: Decision log moves out of the PRD.** The running log of locked decisions now lives in a sidecar `decisions/[feature]-decision-log.md` (its own stakeholder-readable doc). The PRD's § 10 is reduced to a one-line pointer. Every producer (`/create-prd` pre-population, apply-fixes, `/update-prd-from-designs`, `/read-feedback`, conflict resolution, `/change-mode`) appends to the sidecar; every consumer (`/validate-prd`, `/exec-summary`, gate checks, self-checks, `/publish-to-confluence`) reads from it. New `decisions/` folder added to the output tree and Drive-sync mirror.

---

v2.17.0 — 2026-05-29

Intake Q3 (Jira ticket conventions) is now a **suggested-defaults checklist** instead of a blank prompt, and Jira conventions persist across features via a **durable conventions profile** that's confirm-reused on every run.

**Changed: Q3 presents proposed conventions to confirm/edit, not a blank ask.** Stage 0.5 now surfaces a scannable checklist with a concrete suggestion next to each item — every-ticket labels, per-layer labels (`backend`/`frontend`), title format with `[BE]`/`[FE]` layer prefix (explicitly asking how BE vs. FE should be marked in the title), BE/FE split, Testable Yes/No on every ticket, Story Points left blank, and BE↔FE "Relates to" / "Blocked by" link conventions. The PM confirms all, edits any, or says "no conventions yet" (suggestions become the working defaults). Closes the failure mode where a blank open-ended question left PMs inventing conventions from scratch each run.

**New: Durable conventions profile at `_jira-conventions.json`.** Jira project, board, and ticket conventions (Q2 + Q3) are saved once to `~/Desktop/Resources/PDLC Workflow Docs/_jira-conventions.json` and confirm-reused on every future run — intake shows the saved values and asks "reuse / edit / start fresh" instead of re-asking. The profile is rewritten with the latest agreed values after each run. Replaces the prior best-effort scavenge of another feature's `_pipeline-state.json`. Q2 now also captures the **board** (for teams using a shared board with Components as swim lanes), persisted to `intake.board`.

---

v2.16.0 — 2026-05-26

User-stories breakdown layout is now **meat-first with appendix-style metadata**, and a count-parity rule prevents drift between prose count claims and actual blueprint headers.

**New: Meat-first layout spec (`ai-framework/06-user-stories.md` Step 5).** Per-story blueprints (the content stakeholders actually open the doc to read) now appear near the top of the document, behind only the front matter, At-a-glance, How-to-read, Epic outlines (with "What this delivers"), and a small Wave overview table. Operational metadata — ID Stability Policy, refactor history, format conventions, full per-story dependency table, vendor sprint detail, PRD cross-reference — moves to Appendices A through F at the end. Closes the failure mode where the nestfully-ai v1.2 breakdown accreted 460 lines of front-matter through repeated `/change-mode` runs, pushing the first per-story blueprint to line 463 and making the doc unreadable for sponsors and designers.

**New: Source-of-truth count rule.** The number of unique `### US-` blueprint headers is the single computed count. Every prose claim ("Total v1 stories: N", "All N stories written", per-epic "N stories" openers) reads from that computed value. `/user-stories`, `/change-mode`, and `/validate-user-stories` all recompute and reconcile claim sites in the same pass — hand-narrated counts are not allowed.

**New: Validation Checks 8 and 9 in `/validate-user-stories`.** Check 8 flags any policy / refactor / Pass-N / full-dependency-table section appearing above the first `### US-` blueprint header. Check 9 extracts every numeric story-count claim and compares to the actual blueprint count (per-doc and per-epic). Drifted claims are surfaced for fix with the corrected number proposed; layout findings propose moving the section to its named appendix.

**New: `/sync-artifacts` internal-consistency hop.** Agent C now also runs count parity and meat-first layout checks against the breakdown itself, not just PRD-vs-breakdown drift. HIGH internal-consistency findings route the user to `/validate-user-stories` for the per-line walkthrough.

**New: Workspace-level CLAUDE.md reminder** under "User-Stories Document Layout" — the same pattern as the existing Diagram Rendering and ID Stability Policy reminders. The feature-level spec in `ai-framework/06-user-stories.md` Step 5 remains load-bearing; the workspace reminder keeps the rule visible across every regeneration step.

Closes the readability failure mode raised on nestfully-ai v1.2: when the per-story blueprints live behind a wall of maintenance content, the doc stops serving its primary audience.

---

v2.15.0 — 2026-05-25

`/publish-to-confluence` gains a pre-flight drift + comment check that runs before any `update` API call, so the PM sees Confluence-side edits and at-risk inline comments before deciding to overwrite a page.

**New: Pre-flight (Step 2 expanded).** For every page resolving to `update`, the skill fetches `getConfluencePage` + `getConfluencePageInlineComments` + `getConfluencePageFooterComments` in parallel and surfaces three signals in the publish plan: (a) **drift** — current Confluence version vs `last_published_version` in state, flagged `🚨 DRIFT` when someone edited outside the skill; (b) **inline comments** — count + author + anchor-text excerpt for each, flagged `⚠ N inline` because `updateConfluencePage` may orphan them; (c) **footer comments** — count only, marked safe (they survive overwrites). The PM picks per-page: proceed / skip / **pull-comments** (writes a sidecar at `confluence-feedback/[YYYY-MM-DD]/[step-N]-comments.md` and skips the publish for that page) / show-drift / show-comments. Drift never auto-resolves — explicit PM choice required.

**New state field: `last_published_version`** captured from every successful create/update API response and persisted per artifact. Legacy state files (pre-v2.15.0) without the field are treated as `last_published_version = 0`, so the first post-upgrade run never triggers a false drift alarm.

Closes the silent-overwrite failure mode that v2.14.0's auto-Figma-push did not address: PMs can now see and act on Confluence-side edits + inline comments before the skill overwrites the page body.

---

v2.14.0 — 2026-05-24

`/publish-to-confluence` now auto-pushes Mermaid diagrams + timelines to Figma before publishing, so Step 7 (Visual Diagram) and Step 10½ (Timeline) embed as Figma iframes instead of falling back to raw Mermaid code blocks that Confluence can't render without a third-party plugin.

**New: Step 3.5 — Figma auto-push pre-composition.** When the Figma MCP is connected and `_pipeline-state.json` → `export_urls.figma_diagram_url` / `figma_timeline_url` is missing for a Mermaid-bearing artifact being created or updated this run, `/publish-to-confluence` invokes the existing `/visual-diagram` and `/timeline` skills to push to Figma, captures the returned URLs, persists them to state, then composes the Confluence pages with the iframe-embed branch. Opt-out via "skip figma push" at the Step 2 confirmation prompt; falls back gracefully to the Mermaid-source note on push failure or when the Figma MCP is unavailable. Existing Figma URLs in state are treated as authoritative — no silent re-push when the local source changes.

Closes the rendering gap that was sending stakeholders to raw Mermaid code blocks in Confluence, and removes the need for PMs to manually chain `/visual-diagram` + `/timeline` + `/publish-to-confluence`.

---

v2.13.0 — 2026-05-23

/pull-from-figma command added — reverse direction of /push-to-figma. Pulls the post-iteration state of a Figma file (screenshots, metadata, variable bindings) back into the feature workspace after the designer has refined frames.

**New: `/pull-from-figma` standalone command.** Reads the current state of the Figma file `/push-to-figma` seeded (or any file URL the PM provides), refreshes the design catalog with real screenshots + URLs, then optionally diffs against the PRD and user stories and offers to apply updates. Surfaces designer changes since the last push: renamed / deleted / added frames, plus token swaps where the designer bound a different variable. Standalone — not part of the auto-run pipeline, because designers iterate asynchronously (days or weeks after the push). Read-only on Figma; writes only to the local workspace (and Jira via MCP if connected). Screenshots are written to a dated `figma-pulls/[YYYY-MM-DD]/` subfolder so old snapshots are preserved for audit.

Closes the round-trip loop: `/design-prompts` → `/push-to-figma` → designer iterates → `/pull-from-figma` → PRD + user stories synced.

---

v2.12.0 — 2026-05-23

/push-to-figma command added — MCP-driven Figma frame generation from design prompts.

**New: `/push-to-figma` standalone command.** Generates real, editable Figma frames programmatically from a feature's design prompts file via the Figma MCP. Each frame is wired to the team's design system color variables (and components where they fit) — when brand tokens change, every frame updates. Output is the standard design catalog at `design/[feature]-figma-catalog.md` with one row per frame and its direct node URL. Companion to `/design-prompts`: the typical flow is `/design-prompts` → review → `/push-to-figma`. No v0 equivalent (v0 has no programmatic-push API).

Validated on the nestfully-ai feature — 29 mobile frames across 13 categories, generated against the 🐦 Nestfully Mobile library.

---

v2.11.0 — 2026-05-22

/exec-summary skill added + artifact-convention rule (no inline changelogs).

**New: `/exec-summary` standalone command.** Synthesizes a feature's PRD, system design, user-stories breakdown, timeline, and decision log into a single ~20 KB / ~5–7 PDF page executive summary structured for exec readability: vision, use cases, moving pieces, teams, timeline, open decisions, risks, the ask, what's next. Outputs markdown + PDF + DOCX side-by-side in the feature's root folder. Idempotent — re-running overwrites the same three files.

Standalone only — not auto-invoked by `/build-product`. PMs run it whenever an exec or stakeholder needs the "what is this and why" without wading through a 170 KB PRD. PRD is required; other inputs are opportunistic — missing artifacts produce a thinner section with a bracketed note rather than failing.

**New rule: no inline version-history blocks in generated artifacts.** Added to `ai-framework/style-preferences.md` (new "Artifact Conventions" section) and reinforced in `05-change-propagation.md` (Step 4 Writing). Going forward, artifacts get version in the title line + a single changelog pointer line; no `**v0.X (date):** ...` bullets, no "## Change log" sections inside the file. Full history lives in `changelog/[feature]-changelog.md` + per-run summary files + `_pipeline-state.json` → `change_history[]`. `/change-mode` and the generative commands all respect this.

See CHANGELOG.md for full version history.
