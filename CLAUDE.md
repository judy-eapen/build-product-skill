# Product Workspace — Claude Code

All product work in this workspace follows the pipeline defined in `ai-framework/`. Read `ai-framework/rules.md`, `ai-framework/error-handling.md`, and `ai-framework/style-preferences.md` before starting any product work.

---

## Intake Parameters

At the start of every `/build-product` run the skill asks each PM the following questions. Answers are used to configure the pipeline for the current feature. No values are hardcoded.

| # | Question | Notes |
|---|----------|-------|
| 1 | **Feature name** | Used to derive the output folder name. Spaces become hyphens, all lowercase. |
| 2 | **Jira project name** | The team's standard Jira project where tickets will be created. The skill does not assume a project. |
| 3 | **Jira ticket conventions** | **Open-ended probe with examples.** The skill asks: "What conventions does your team apply to every Jira ticket? Examples to think about: (a) labels you always apply (e.g., pod tags, area tags, team names); (b) title format (e.g., verb-first, `[BE]` / `[FE]` prefix, "Feature - Sub-feature" for Epics); (c) how BE and FE work is split (separate tickets vs. combined); (d) default values for custom fields (e.g., Testable: Yes/No on every ticket); (e) fields you intentionally leave blank (e.g., Story Points); (f) link conventions (e.g., always link BE↔FE pairs as 'Relates to'; use 'Blocked by' with a note). Share whatever's standard for your team in your own words — bullets, prose, or "we don't have specific conventions yet" are all fine." Answer is captured free-text to `_pipeline-state.json` → `intake.jira_ticket_conventions` and applied at Step 10 (User Stories Breakdown) and Step 11a (Jira Export). |
| 4 | **Tech stack** | Either name your stack, say "I don't know", or describe the product type and the skill will propose defaults appropriate to that type. |
| 5 | **Product type** | Web app, mobile app, backend service, internal tool, AI feature, or other. Used to decide which PRD sections apply and which defaults to propose. |
| 6 | **Permission model?** | Yes / No / Not yet decided. Drives whether the Roles & Permissions section of the PRD is required. |
| 7 | **Backend or API surface?** | Yes / No / Not yet decided. Drives whether the API Contracts section of the PRD is required. |

The skill will not proceed until these are answered.

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
- `/visual-diagram` — Generate a Figma FigJam diagram from a PRD (falls back to Mermaid if Figma MCP is unavailable).
- `/user-stories` — Generate the User Stories Breakdown (Gherkin AC + FE/BE pairing) from an approved PRD. Proposes a multi-Epic grouping (one Epic per phase by default, sub-epics for functional clusters) which the PM accepts or adjusts. Supports DRAFT mode for stories whose design details aren't finalized yet — DRAFT stories get `Status: DRAFT — needs design` at the top, no UX state coverage required, sized with `*` suffix. Refresh later via `/change-mode` → "Designs arrived".
- `/timeline` — Generate a Gantt timeline (Figma FigJam + editable HTML) at the Epic + Phase level from an approved user-stories breakdown. Hybrid estimation — skill proposes from sizing × velocity, PM tunes. **HTML is editable in the browser (v2.5.0+):** drag bars to shift/resize with auto-cascade (Shift to lock). Click Export Plan → run `/timeline apply [json-path]` to round-trip edits back into `_pipeline-state.json` and re-render the markdown sidecar + HTML with the applied dates as the new baseline.
- `/export-transcript` — Write the full back-and-forth between the PM and the model for a feature's pipeline run to two markdown files (clean reading version + full forensic version with tool calls). Auto-runs as Step 12 at the end of `/build-product`; also callable standalone any time.
- `/pipeline-timing` — Generate a timing report for a feature's pipeline run with both wall-clock time and active-work time (gate waits excluded). Reads instrumented timestamps from `step_timings` in state; falls back to JSONL parsing for un-instrumented runs.
- `/prd-to-jira` — Create Jira tickets from a user-stories breakdown or PRD.
- `/drive-sync` — Sync a feature's pipeline artifacts to Google Drive (requires Google Drive MCP installed).
- `/publish-to-confluence` — Publish the whole feature workspace to Confluence as a parent hub + one numbered child page per artifact (Research, Codebase Review, PRD, Product Review, Technical Review, System Design, Visual Diagram, Design Catalog per phase, User Stories, Timeline). Per-file mtime change detection — only re-publishes pages whose source actually changed. Requires Atlassian MCP connected.
- `/share-for-review` — Post a Confluence artifact link to Slack with tagged reviewers and a deadline. Works with Slack MCP when connected; outputs formatted message for manual paste otherwise.
- `/read-feedback` — Pull reviewer comments from a Confluence page, synthesize into suggested PRD edits, and apply approved changes. Re-syncs PRD to Confluence after edits.

### Design
- `/design-prompts` — Generate screen design prompts for v0 or Figma Make from an approved PRD.
- `/update-prd-from-designs` — Sync PRD with finalized design catalog.
- `/compare-figma-prd` — Figma vs PRD & Jira gap analysis after designer delivers.

### Execution & Validation
- `/lint-style` — Check any generated document against `ai-framework/style-preferences.md` and flag or fix style violations.

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
