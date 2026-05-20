# Changelog

All notable changes are documented here. Follows [Semantic Versioning](https://semver.org): MAJOR.MINOR.PATCH.

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
