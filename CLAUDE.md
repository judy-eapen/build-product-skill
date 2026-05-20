# Product Workspace — Claude Code

All product work in this workspace follows the pipeline defined in `ai-framework/`. Read `ai-framework/rules.md`, `ai-framework/error-handling.md`, and `ai-framework/style-preferences.md` before starting any product work.

---

## Intake Parameters

At the start of every `/build-product` run the skill asks each PM the following questions. Answers are used to configure the pipeline for the current feature. No values are hardcoded.

| # | Question | Notes |
|---|----------|-------|
| 1 | **Feature name** | Used to derive the output folder name. Spaces become hyphens, all lowercase. |
| 2 | **Jira project name** | The team's standard Jira project where tickets will be created. The skill does not assume a project. |
| 3 | **Team Jira label conventions** | Any labels your team applies to tickets (for example role prefixes, pod tags, area tags). Defaults below can be overridden here. |
| 4 | **Tech stack** | Either name your stack, say "I don't know", or describe the product type and the skill will propose defaults appropriate to that type. |
| 5 | **Product type** | Web app, mobile app, backend service, internal tool, AI feature, or other. Used to decide which PRD sections apply and which defaults to propose. |
| 6 | **Permission model?** | Yes / No / Not yet decided. Drives whether the Roles & Permissions section of the PRD is required. |
| 7 | **Backend or API surface?** | Yes / No / Not yet decided. Drives whether the API Contracts section of the PRD is required. |

The skill will not proceed until these are answered.

---

## Pipeline Selection

| Situation | Command | Pipeline |
|-----------|---------|----------|
| New product or greenfield app | `/build-product` | Personal → Full |
| New feature in existing app | `/build-product` | Personal → Medium |
| Bug fix, small UI change | `/build-product` | Personal → Light |
| Work project — planning + Jira only | `/build-product` | Work |

**Fast mode** (default): Chains all auto-steps. Pauses only at 3 approval gates.
**Gated mode**: Pauses after every step. Use when reviewing each output matters.

---

## Available Commands

### Pipeline Orchestration
- `/build-product` — Full pipeline (research → ship). Handles Personal and Work projects.
- `/change-mode` — Propagate a change after Gate 1 across every artifact for an existing feature.
- `/reopen-gate-1`, `/reopen-gate-2`, `/reopen-gate-3` — Re-open an approved gate.
- `/review-parallel` — Dual product + CTO review in parallel.
- `/validate-parallel` — Three-way parallel validation: AC compliance + design match + NFR.

### Standalone Pipeline Steps
Each command runs one stage of the pipeline independently. Each asks for the inputs it needs (PRD, design catalog, etc.) if you call it outside the full `/build-product` run.

- `/research-idea` — Run only the research stage.
- `/codebase-review` — Run only the codebase review (paste code, describe architecture, or confirm greenfield).
- `/create-prd` — Generate a PRD from research output or a brief.
- `/review-prd` — Product Reviewer pass on an existing PRD.
- `/cto-review` — Technical Reviewer pass on an existing PRD (also reads codebase review if available).
- `/system-design` — Generate a system-design doc from a PRD.
- `/visual-diagram` — Generate a Mermaid diagram from a PRD or feature description.
- `/user-stories` — Generate the User Stories Breakdown (Gherkin AC + FE/BE pairing) from an approved PRD.
- `/prd-to-jira` — Create Jira tickets from a user-stories breakdown or PRD.
- `/drive-sync` — Sync a feature's pipeline artifacts to Google Drive (requires Google Drive MCP installed).
- `/prd-to-confluence` — Publish a PRD as a Confluence page (requires Atlassian MCP connected).
- `/share-for-review` — Post a Confluence artifact link to Slack with tagged reviewers and a deadline. Works with Slack MCP when connected; outputs formatted message for manual paste otherwise.
- `/read-feedback` — Pull reviewer comments from a Confluence page, synthesize into suggested PRD edits, and apply approved changes. Re-syncs PRD to Confluence after edits.

### Design
- `/design` — In-repo UI design: AI builds screens directly in code.
- `/design-with-v0` — v0-based design: AI generates prompts, you paste into v0.
- `/update-prd-from-designs` — Sync PRD with finalized design catalog.
- `/compare-figma-prd` — Figma vs PRD & Jira gap analysis (Work pipeline, after designer delivers).

### Execution & Validation
- `/execute-plan` — Implement current phase (solo or team mode).
- `/validate` — Smoke-test against PRD acceptance criteria and designs.
- `/update-prd-from-build` — Sync PRD with what was actually implemented.
- `/generate-tests` — Scaffold test stubs from a user-stories breakdown (Gherkin AC → test framework). Run after breakdown is approved, before execution.
- `/lint-style` — Check any generated document against `ai-framework/style-preferences.md` and flag or fix style violations.

### Status & Utilities
- `/team-status` — Portfolio dashboard: all products, phases, owners, blockers.
- `/feature-kickoff` — Structured briefing for a team member picking up a PRD.
- `/project-status` — Current pipeline state and next step for a single product.
- `/prioritize` — Feature/initiative prioritization (RICE, MoSCoW, Value vs Effort).
- `/meeting-notes` — Parse raw notes into decisions, action items, next steps.
- `/learn-codebase` — Plain-language walkthrough of any app in this workspace.
- `/learn` — Post-ship reflection and learning report.


---

## PRD — Single Source of Truth

The PRD must be updated at every stage transition:

| Trigger | Action |
|---------|--------|
| After reviews | Apply all product + CTO review fixes |
| After design finalized | Run `/update-prd-from-designs` |
| After phase validated | Run `/update-prd-from-build` |
| Scope change | Update stories, phases, AC before proceeding |
| New decision made | Add row to decision log |

Never start execution with a PRD that does not reflect the approved design.

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
