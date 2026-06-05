# Product Workspace — Claude Code

All product work in this workspace follows the pipeline defined in `ai-framework/`. Read `ai-framework/rules.md`, `ai-framework/error-handling.md`, and `ai-framework/style-preferences.md` before starting any product work.

---

## Intake Parameters

At the start of every `/build-product` run the skill asks each PM the following questions. Answers are used to configure the pipeline for the current feature. No values are hardcoded.

| # | Question | Notes |
|---|----------|-------|
| 1 | **Feature name** | Used to derive the output folder name. Spaces become hyphens, all lowercase. |
| 2 | **Jira project + board** | The team's standard Jira project (and board, if they use a shared board with Components for swim lanes) where tickets will be created. The skill does not assume a project. |
| 3 | **Jira ticket conventions** | **Suggested-defaults checklist, NOT a blank prompt.** The skill presents proposed conventions the PM confirms, edits, or removes: (a) every-ticket labels (*suggestion:* a pod/area tag); (b) per-layer labels (*suggestion:* `backend` on BE, `frontend` on FE); (c) title format (*suggestion:* verb-first with `[BE]`/`[FE]` layer prefix, `"Feature - Sub-feature"` for Epics — explicitly asks how the PM wants BE vs. FE marked in the title); (d) BE/FE split (*suggestion:* separate tickets); (e) Testable field (*suggestion:* Yes/No on every ticket); (f) fields left blank (*suggestion:* Story Points); (g) link conventions (*suggestion:* BE↔FE as "Relates to", "Blocked by" with a note). PM confirms all / edits any / says "no conventions yet" (suggestions become defaults). Final agreed set captured verbatim to `_pipeline-state.json` → `intake.jira_ticket_conventions`, applied at Step 10 (User Stories Breakdown) and Step 11a (Jira Export). |
| 4 | **Tech stack** | Either name your stack, say "I don't know", or describe the product type and the skill will propose defaults appropriate to that type. |
| 5 | **Product type** | Web app, mobile app, backend service, internal tool, AI feature, or other. Used to decide which PRD sections apply and which defaults to propose. |
| 6 | **Permission model?** | Yes / No / Not yet decided. Drives whether the Roles & Permissions section of the PRD is required. |
| 7 | **Backend or API surface?** | Yes / No / Not yet decided. Drives whether the API Contracts section of the PRD is required. |

The skill will not proceed until these are answered.

**Durable conventions profile.** The Jira project, board, and ticket conventions (Q2 + Q3) are saved once to `~/Desktop/Resources/PDLC Workflow Docs/_jira-conventions.json` and **confirm-reused** on every future run — on intake the skill shows the saved values and asks "reuse these? (yes / edit / start fresh)" instead of re-asking from scratch. The profile is rewritten with the latest agreed values after each run. See `subprompts/build-product.md` → Stage 0.5 for the flow.

---

## Pipeline Selection

| Situation | Command | Pipeline |
|-----------|---------|----------|
| Any feature — planning + Jira export | `/build-product` | Work |

**Fast mode** (default): Chains all auto-steps. Pauses only at 3 approval gates.
**Gated mode**: Pauses after every step. Use when reviewing each output matters.

---

## Available Commands

### Pipeline Orchestration
- `/build-product` — Work pipeline (research → PRD → design → Jira export). Planning only, no implementation.
- `/change-mode` — Propagate a change after Gate 1 across every artifact for an existing feature. Seven trigger types; "Designs arrived" (v2.4.0+) is the trigger for refreshing DRAFT stories once finalized designs are available — refreshes AC, sizing, UX state coverage, and Jira tickets in place.
- `/reopen-gate-1`, `/reopen-gate-2`, `/reopen-gate-3` — Re-open an approved gate.

### Standalone Pipeline Steps
Each command runs one stage of the pipeline independently. Each asks for the inputs it needs (PRD, design catalog, etc.) if you call it outside the full `/build-product` run.

- `/research-idea` — Run only the research stage.
- `/codebase-review` — Run only the codebase review (paste code, describe architecture, or confirm greenfield).
- `/create-prd` — Generate a PRD from research output or a brief.
- `/review-prd` — Product Reviewer pass on an existing PRD.
- `/cto-review` — Technical Reviewer pass on an existing PRD (also reads codebase review if available).
- `/system-design` — Generate a system-design doc from a PRD.
- `/visual-diagram` — Generate a Figma FigJam diagram from a PRD (Figma is the deliverable). If the Figma MCP is unavailable, it produces a clearly-labeled **temporary Mermaid fallback** flagged for regeneration as Figma once the MCP is connected — never a co-equal Mermaid surface.
- `/user-stories` — Generate the User Stories Breakdown (acceptance criteria + FE/BE pairing) from an approved PRD. **Step 2.7 fit-checks the AC format** — Gherkin vs. plain-English criteria — and shows the PM a side-by-side sample of one real story both ways before committing (Gherkin / plain / mixed; stored as `user_stories.ac_format`). Proposes a multi-Epic grouping (one Epic per phase by default, sub-epics for functional clusters) which the PM accepts or adjusts. **Computes a Wave assignment** for every story via topological sort on dependencies (v2.10.0+) — waves group stories that can ship in parallel; cycles in the dependency graph are CRITICAL Gate 3 findings. Supports DRAFT mode for stories whose design details aren't finalized yet — DRAFT stories get `Status: DRAFT — needs design` at the top, no UX state coverage required, sized with `*` suffix. Refresh later via `/change-mode` → "Designs arrived".
- `/timeline` — Generate a Gantt timeline (Figma FigJam + editable HTML) at the Epic + Phase level from an approved user-stories breakdown. Hybrid estimation — skill proposes from sizing × velocity, PM tunes. **HTML is editable in the browser:** drag bars to shift/resize with auto-cascade (Shift to lock). **💾 Save to skill** (v2.6.0+, Chrome/Edge) writes plan JSON directly to the feature's `timeline/` folder via the File System Access API. Then run `/timeline apply` (no args) — auto-discovers the latest plan and round-trips edits back into `_pipeline-state.json` and re-renders the markdown sidecar + HTML.
- `/export-transcript` — Write the full back-and-forth between the PM and the model for a feature's pipeline run to two markdown files (clean reading version + full forensic version with tool calls). Auto-runs as Step 12 at the end of `/build-product`; also callable standalone any time.
- `/pipeline-timing` — Generate a timing report for a feature's pipeline run with both wall-clock time and active-work time (gate waits excluded). Reads instrumented timestamps from `step_timings` in state; falls back to JSONL parsing for un-instrumented runs.
- `/prd-to-jira` — Create Jira tickets from a user-stories breakdown or PRD.
- `/exec-summary` — Synthesize the feature's PRD, system design, user-stories breakdown, timeline, and decision log into a 9-section, ~20 KB executive summary (markdown + PDF + DOCX). Standalone — not auto-invoked by `/build-product`. Idempotent: overwrites the same three output files on re-run. Use when an exec or stakeholder needs the "what is this and why" without reading the full PRD.
- `/infosec-doc` — Fill out Bright's InfoSec Questionnaire (the Ops/DevOps security-review `.xlsx`) for a feature. Derives answers across all 7 tabs from the PRD, system design, technical review, codebase review, diagrams, and `_pipeline-state.json`; batch-interviews the PM for the operational facts no artifact carries (severity — gates DR; escalation/emergency contacts; DR region; vendor SLA; approved-AI-list + data-sharing opt-out; legal sign-off). **Never fabricates a security answer** — every cell is sourced from an artifact, PM-confirmed, or written `⚠ NEEDS INPUT`; Bright-standard defaults (TLS/AES) are filled but flagged for confirmation. Writes the real workbook via `openpyxl` (copying the read-only golden template each run, preserving formatting + the severity dropdown) to `[feature]/infosec/[feature]-infosec-questionnaire.xlsx`. Standalone — not auto-invoked by `/build-product`. Idempotent.
- `/create-slidedeck` — Turn a feature's pipeline artifacts into a presentation-ready slide deck. Standalone, pipeline-aware (works on pasted/standalone content too). **Always runs a full interview** (deck type, audience, goal, slide count, depth, tone/branding, source artifacts, speaker notes, render surfaces) and confirms a one-line-per-slide outline before writing. Three presets: `exec` (leadership greenlight), `kickoff` (team alignment), `demo` (stakeholder overview); `custom` for anything else. Produces a **slide-spec markdown as the source of truth**, always emits a **paste-into-Claude/Figma-Make deck-prompt**, and optionally renders **Figma Slides** (via Figma MCP) and/or a self-contained **HTML + PDF** deck (same pandoc/Chrome toolchain as `/exec-summary`). Idempotent per deck-type; multiple deck-types coexist. Output → `[feature]/slides/`.
- `/drive-sync` — Sync a feature's pipeline artifacts to Google Drive (requires Google Drive MCP installed).
- `/publish-to-confluence` — Publish the feature workspace to Confluence as a parent hub + one numbered child page per stakeholder-relevant artifact: **Research, Codebase Review, PRD, System Design, Visual Diagram, Design Catalog (per phase), User Stories (lightweight Jira Epic index — not full Gherkin), Timeline.** Product/Technical Reviews and the full User Stories Breakdown stay local — not published. Per-file mtime change detection — only re-publishes pages whose source actually changed. Requires Atlassian MCP connected.
- `/share-for-review` — Post a Confluence artifact link to Slack with tagged reviewers and a deadline. Works with Slack MCP when connected; outputs formatted message for manual paste otherwise.
- `/read-feedback` — Pull reviewer comments from a Confluence page, synthesize into suggested PRD edits, and apply approved changes. Re-syncs PRD to Confluence after edits.

### Design
- `/design-prompts` — Generate screen design prompts for v0 or Figma Make from an approved PRD.
- `/push-to-figma` — Generate real Figma frames programmatically from the design prompts file via the Figma MCP. Wires each frame to your team's design system color variables (and components where they fit). Output is editable Figma frames bound to the source library — not static images. Companion to `/design-prompts`; typical flow is `/design-prompts` → review → `/push-to-figma`. Requires Figma MCP connected. Frame dimensions are mobile (390×844) or desktop (1440×900) based on intake's product type. No v0 equivalent — v0 has no programmatic-push API.
- `/pull-from-figma` — Pull the post-iteration state of a Figma file back into the feature workspace after the designer has refined the pushed frames. Refreshes the design catalog with real screenshots + URLs, then optionally diffs against the PRD and user stories and offers to apply updates. Standalone (not in the auto-run pipeline) because designers iterate asynchronously. Requires Figma MCP connected. Read-only on Figma; writes only to the local workspace (and Jira via MCP if connected).
- `/update-prd-from-designs` — Sync PRD with finalized design catalog.
- `/compare-figma-prd` — Figma vs PRD & Jira gap analysis after designer delivers.

### Execution & Validation
- `/lint-style` — Check any generated document against `ai-framework/style-preferences.md` and flag or fix style violations.
- `/pipeline-doctor` — Scan the skill and feature workspaces for **structural** drift and inconsistencies. Four check categories: (A) skill self-consistency — every step in `pipeline-configs.yaml` has matching prose in `SKILL.md` and `subprompts/build-product.md`, every quality_check is defined, every instruction file exists; (B) feature-state consistency — `_pipeline-state.json` schema, artifact-path existence on disk, gate-state coherence, DRAFT/epic/Confluence cross-checks; (C) slash command coverage — every subprompt has a registered command, no broken pointers; (D) stale features — pipelines older than 30 days without completion. Read-only by default; per-finding fix approval. Writes a timestamped report to the workspace root.
- `/validate-prd` — **Semantic** content validation for a PRD (not structural — see `/pipeline-doctor` for that). Six checks: (1) internal consistency between sections, (2) hallucinated data / unsourced claims, (3) completeness (TBD/TODO/empty sections), (4) VOC traceability against the research output, (5) NFR measurability with concrete thresholds, (6) scope coherence between in-scope/out-of-scope/success-metrics. Inline summary + per-finding walkthrough + timestamped report to `[feature]/validation/`.
- `/validate-user-stories` — **Semantic** content validation for a user-stories breakdown. Seven checks: (1) story↔PRD traceability, (2) AC duplication / contradiction across stories, (3) FE/BE pair coherence (endpoint contracts, naming, permissions), (4) AC specificity (catches vague AC in either format), (5) sizing sanity (scenario/criterion count + surface count vs size label), (6) DRAFT consistency between state and the breakdown file, (7) wave / dependency sanity (acyclic, phase-ordered). Same UX as validate-prd. **Most expensive command in the skill** — runtime ~1–3 minutes on a 500KB+ breakdown.

### Status & Utilities
- `/team-status` — Portfolio dashboard: all products, phases, owners, blockers.
- `/feature-kickoff` — Structured briefing for a team member picking up a PRD.
- `/project-status` — Current pipeline state and next step for a single product.
- `/prioritize` — Feature/initiative prioritization (RICE, MoSCoW, Value vs Effort).
- `/meeting-notes` — Parse raw notes into decisions, action items, next steps.
- `/learn-codebase` — Plain-language walkthrough of any app in this workspace.


---

## PRD — Single Source of Truth

The PRD must be updated at every stage transition:

| Trigger | Action |
|---------|--------|
| After reviews | Apply all product + CTO review fixes |
| After design finalized | Run `/update-prd-from-designs` |
| Scope change | Update stories, phases, AC before proceeding |
| New decision made | Add row to decision log |

Never start the User Stories Breakdown with a PRD that does not reflect the approved design.

---

## Default conventions, adjust to your team

These defaults can be overridden at intake. They are not enforced; they exist as a starting point. The skill will ask which Jira project and label conventions your team uses before applying any of these.

- Every feature has its own subdirectory under the output root.
- PRD Executive Summary includes an **Owner** field (person responsible).
- **Jira project:** Use your team's standard Jira project. The skill will ask which project to use at intake.
- BE tickets: `[BE]` prefix + labels: `backend`.
- FE tickets: `[FE]` prefix + labels: `frontend`.
- Pod or area label (for example a team tag): add at intake.
- Link BE and FE tickets for the same feature using the **Relates to** field.
- Story Points: do not fill in.
- Testable field: always set to Yes or No.

---

## Output Folder Structure

Outputs land at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/`. See `SKILL.md` (OUTPUT CONVENTIONS) for the full subfolder layout and per-step file paths.

---

## Writing Style

Writing style preferences live in a separate file at `ai-framework/style-preferences.md`. Edit that file to match your preferences. Leave it empty to use no style rules. The skill reads from that file and applies whatever rules are present.

---

## Release Checklist

When making significant changes to this skill (new commands, pipeline changes, bug fixes), update these four files before committing:

| File | What to update |
|------|---------------|
| `CHANGELOG.md` | Add a new version entry at the top following the existing format. Use semantic versioning: MAJOR for breaking changes, MINOR for new features, PATCH for bug fixes. |
| `VERSION.md` | Update to the new version number and date. |
| `README.md` | Update the pipeline table, commands table, or any section that reflects what changed. Update the version badge at the top. |
| `docs/index.html` | Update the version comment on line 6, the `.version-badge` text, the stats if counts changed, step cards if pipeline steps changed, the commands section if commands changed, and add a new entry to the changelog section. |

The HTML at `docs/index.html` is served via GitHub Pages at `https://judy-eapen.github.io/build-product-skill/`. It must always reflect the current version — update it in the same commit as the skill changes.
