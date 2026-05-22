# Changelog

All notable changes are documented here. Follows [Semantic Versioning](https://semver.org): MAJOR.MINOR.PATCH.

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
