# /build-product

**A Claude skill that takes a product manager from an idea for a feature → a Jira Epic with all the linked tickets.** Research, codebase grounding, PRD, dual review, designs, exhaustive user stories with Gherkin acceptance criteria, and ticket creation in Jira (plus optional Google Drive sync and Confluence publishing) — in one continuous workflow with three human approval gates.

## What it does

When you type `/build-product` in Claude Code, an AI orchestrator runs an 11-step pipeline:

1. **Research Idea** — Interactive Discovery, 15-dimension internal checklist, no fixed question count.
2. **Codebase Review** — point at folder paths on disk; the skill reads structure, samples files, rates risks.
3. **Create PRD** — 11-section PRD, scoping-aware, self-checking against decisions.
4. **Dual Review (parallel)** — two true agents at the same instant: Product Reviewer + Technical Reviewer.
5. **Apply Fixes → Gate 1** — ~10 automated quality checks + conflict-card resolution + your approval.
6. **System Design** (optional, if architecturally non-trivial).
7. **Visual Diagram** — user journey, system architecture, or wireflow (Mermaid).
8. **Design** — in-repo screens or v0 prompts, with required state coverage per screen.
9. **Update PRD from Designs → Gate 2** — surgical sync, your approval.
10. **User Stories Breakdown → Gate 3** — exhaustive Gherkin AC, FE/BE pairing, testing notes, sizing, 8 quality checks.
11. **Export (parallel)** — Jira tickets always, plus optional Google Drive sync + Confluence publishing.

Plus `/change-mode` for propagating changes after a gate, `/reopen-gate-1/2/3` for unwinding bad approvals, and 11 standalone commands so you can use individual steps without running the whole pipeline.

## Installation

### Prerequisites

- A Claude account (Pro or Team).
- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed.
- `git` installed. Check with `git --version`. On Mac, git ships with Xcode Command Line Tools — if you don't have it, just run any `git` command and macOS will prompt to install. Or `brew install git` if you use Homebrew.

### Install the skill

Clone this repo into your local Claude skills directory:

```bash
git clone https://github.com/judy-eapen/build-product-skill.git ~/.claude/skills/build-product
```

That's it. Open Claude Code in any folder, type `/`, and `/build-product` will appear in the autocomplete.

### Update to the latest version

```bash
cd ~/.claude/skills/build-product
git pull
```

## Quickstart

1. Open Claude Code in any folder (works from anywhere).
2. Type `/build-product`.
3. Answer a few setup questions (feature name, Jira project, product type).
4. The pipeline walks you through 11 steps. You approve at 3 gates along the way.
5. Find every artifact at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/`.

## Where outputs land

```
~/Desktop/Resources/PDLC Workflow Docs/
└── [feature-name]/
    ├── research/
    ├── codebase-review/
    ├── prd/
    ├── product-review/
    ├── technical-review/
    ├── diagrams/
    ├── design/
    ├── user-stories/
    ├── jira-export/
    ├── changelog/
    └── _pipeline-state.md
```

## Customize

The skill is yours to edit once cloned. Common starting points:

- `ai-framework/style-preferences.md` — your writing style.
- `ai-framework/01-research-idea.md` — the 15-dimension Interactive Discovery checklist.
- `ai-framework/02-create-prd.md` — PRD sections and scoping logic.
- `subprompts/prd-to-jira.md` — how tickets get composed and which Jira custom fields are filled.
- `CLAUDE.md` — intake parameters every PM is asked at the start.

Pull updates from this repo with `git pull` (your local edits stay unless they conflict).

## Optional integrations

- **Jira** — always required for Step 11. Connect Atlassian MCP in Claude Code.
- **Google Drive** — optional. Install a Google Drive MCP and authenticate. Enables `/drive-sync` and the Step 11b parallel agent.
- **Confluence** — optional. Uses the same Atlassian MCP. Enables `/prd-to-confluence` and the Step 11c parallel agent.

The skill skips any optional integration cleanly if its MCP isn't connected — it never blocks the rest of the pipeline.

## Architecture highlights

- **Resumable**: a `_pipeline-state.md` file is written at the end of every step and read first on every new conversation. Sessions can pause for hours or days and resume in a fresh chat window.
- **Cross-step traceability**: each step's output is a structured input to specific downstream steps. The codebase review handoff seeds the PRD. The PRD decision log is referenced everywhere. The breakdown feeds the Jira export. The Jira manifest enables `/change-mode`.
- **Forcing functions, not soft instructions**: 15-dimension checklist forces completeness. Conflict cards block Gate 1 approval. Pre-flight validates Jira custom field IDs before bulk creation.
- **Memory across features**: `_knowledge-base.md` captures lessons from every shipped feature. Future PRDs surface relevant past learnings.
- **Recovery without restart**: `/change-mode` blast-radius reports + diff-by-diff approval. Reopen gates without losing downstream work.

## License

MIT (see LICENSE file if included). Feel free to fork, modify, and adapt.

## Version

v1.0 — initial public release.
