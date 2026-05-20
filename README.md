# /build-product

**A Claude skill that takes a PM from a feature idea → a Jira Epic with all linked tickets.** Research, codebase grounding, PRD, dual AI review, design prompts for v0 or Figma Make, exhaustive user stories with Gherkin acceptance criteria, and Jira ticket creation — in one continuous Work pipeline with three human approval gates.

> **v2.1.0 — Open-ended Jira conventions at intake.** The intake step now probes for every per-ticket convention your team uses (labels, title format, BE/FE split, custom-field defaults, link conventions) and applies them automatically when generating user stories and creating Jira tickets. No need to encode conventions in a personal CLAUDE.md.
>
> **v2.0.0** — Work pipeline only. The Personal pipeline (Full/Medium/Light) was removed in 2.0.

📄 **[View the interactive pipeline explainer →](https://judy-eapen.github.io/build-product-skill/)**

---

## What it does

Type `/build-product` in Claude Code. An AI orchestrator runs an 11-step Work pipeline:

| # | Step | Mode | Output |
|---|------|------|--------|
| 1 | **Research Idea** — Interactive discovery, knowledge-base check, web scan | AUTO | `research/` |
| 2 | **Codebase Review** — Reads folder structure, samples files, rates risks | AUTO | `codebase-review/` |
| 3 | **Create PRD** — 11-section PRD, scoping-aware, auto-linted | AUTO | `prd/` |
| 4 | **Dual Review** — Two parallel agents: Product Reviewer + Technical Reviewer | PARALLEL | `product-review/` + `technical-review/` |
| 5 | **Apply Fixes → Gate 1** — ~10 auto quality checks, conflict cards, your approval | **GATE** | `prd/` (updated) |
| 6 | **System Design** — Architecture, data model, build order (optional) | AUTO | `technical-review/` |
| 7 | **Visual Diagram** — Figma FigJam via Figma MCP (falls back to Mermaid) | AUTO | `diagrams/` |
| 8 | **Design Prompts** — Structured v0 / Figma Make prompts per screen, per state | AUTO | `design/` |
| 9 | **Update PRD from Designs → Gate 2** — Surgical sync, your approval | **GATE** | `prd/` (updated) |
| 10 | **User Stories Breakdown → Gate 3** — Exhaustive Gherkin AC, FE/BE pairs, 8 quality checks | **GATE** | `user-stories/` |
| 11 | **Export** — Jira tickets always; Google Drive + Confluence optional, parallel | PARALLEL | Jira + Drive + Confluence |

---

## Installation

### Prerequisites

- A Claude account (Pro or Team).
- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed.
- `git` installed (`git --version` to check; on Mac, Xcode Command Line Tools includes it).

### Install

```bash
git clone https://github.com/judy-eapen/build-product-skill.git ~/.claude/skills/build-product
```

**Register the slash commands** (one-time; creates wrapper files in `~/.claude/commands/`):

```bash
mkdir -p ~/.claude/commands && for f in ~/.claude/skills/build-product/subprompts/*.md; do n=$(basename "$f" .md); printf 'Read and follow `~/.claude/skills/build-product/subprompts/%s.md`.\n' "$n" > ~/.claude/commands/"$n.md"; done
```

Open Claude Code, type `/`, and `/build-product` plus all standalone commands appear in autocomplete.

> **Why the second step?** Claude Code discovers slash commands from `~/.claude/commands/`, not from inside a skill folder. The one-liner creates a thin wrapper for each subprompt so they all surface as top-level commands.

### Update

```bash
cd ~/.claude/skills/build-product && git pull
```

---

## Quickstart

1. Open Claude Code in any folder.
2. Type `/build-product`.
3. Answer a few setup questions (feature name, Jira project, product type, tech stack).
4. The pipeline runs. You approve at 3 gates.
5. Artifacts land at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/`.

---

## All commands

### Pipeline
| Command | What it does |
|---------|-------------|
| `/build-product` | Full Work pipeline — research → Jira export |
| `/change-mode` | Propagate a scope change after any gate across all artifacts |
| `/reopen-gate-1` | Unwind Gate 1 approval; re-run steps before it |
| `/reopen-gate-2` | Unwind Gate 2 approval; re-run steps before it |
| `/reopen-gate-3` | Unwind Gate 3 approval; re-run steps before it |

### Standalone steps
| Command | What it does |
|---------|-------------|
| `/research-idea` | Research stage only |
| `/codebase-review` | Codebase review only |
| `/create-prd` | Generate a PRD from research or a brief |
| `/review-prd` | Product Reviewer pass on an existing PRD |
| `/cto-review` | Technical Reviewer pass on an existing PRD |
| `/system-design` | System design doc from a PRD |
| `/visual-diagram` | Figma FigJam diagram from a PRD (Mermaid fallback) |
| `/user-stories` | User Stories Breakdown from an approved PRD |
| `/prd-to-jira` | Create Jira tickets from a breakdown or PRD |
| `/drive-sync` | Sync artifacts to Google Drive |
| `/prd-to-confluence` | Publish PRD as a Confluence page |
| `/share-for-review` | Post a Confluence link to Slack with reviewers + deadline |
| `/read-feedback` | Pull Confluence comments, synthesize into PRD edits, re-sync |

### Design
| Command | What it does |
|---------|-------------|
| `/design-prompts` | Screen design prompts for v0 or Figma Make |
| `/update-prd-from-designs` | Sync PRD with finalized design catalog |
| `/compare-figma-prd` | Figma vs PRD & Jira gap analysis after designer delivers |

### Utilities
| Command | What it does |
|---------|-------------|
| `/lint-style` | Check any document against `style-preferences.md` |
| `/team-status` | Portfolio dashboard: all features, phases, owners, blockers |
| `/feature-kickoff` | Role-specific briefing for an engineer or designer picking up a feature |
| `/project-status` | Pipeline state and next step for a single feature |
| `/prioritize` | RICE / MoSCoW / Value-vs-Effort prioritization |
| `/meeting-notes` | Parse raw meeting notes into decisions, actions, next steps |
| `/learn-codebase` | Plain-language walkthrough of any codebase |

---

## Optional integrations

| Integration | MCP | Used for |
|-------------|-----|---------|
| **Jira** | Atlassian MCP | Step 11a — ticket creation (required for export) |
| **Confluence** | Atlassian MCP | Step 11c + `/prd-to-confluence` + `/read-feedback` |
| **Google Drive** | Drive MCP | Step 11b + `/drive-sync` |
| **Figma** | Figma MCP | Step 7 — FigJam diagram generation |
| **Slack** | Slack MCP | `/share-for-review` — post review links with @mentions |

Any integration the skill can't reach is skipped cleanly — it never blocks the rest of the pipeline.

---

## Output structure

```
~/Desktop/Resources/PDLC Workflow Docs/
├── _knowledge-base.md          ← cross-feature learnings (append-only)
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
    ├── stakeholders/
    ├── _pipeline-state.json    ← resumable session state
    ├── _context-checkpoint.md  ← compact context for session resumption
    └── _open-conditions.md     ← conditions from "approved with conditions" gates
```

---

## Customize

Common starting points:

- `ai-framework/style-preferences.md` — your writing style rules.
- `ai-framework/01-research-idea.md` — the 15-dimension Interactive Discovery checklist.
- `ai-framework/02-create-prd.md` — PRD sections and scoping logic.
- `subprompts/prd-to-jira.md` — how tickets are composed and which Jira custom fields are filled.
- `CLAUDE.md` — intake parameters every PM answers at pipeline start.

---

## Architecture notes

- **Resumable** — `_pipeline-state.json` is written at the end of every step and read first in every new session. Gate approvals, artifact paths, and export URLs are all persisted.
- **True parallelism** — Dual review (Step 4) and Export (Step 11) use the Claude Agent tool to run isolated sub-agents simultaneously, not role-switching in a single context.
- **Forcing functions** — 15-dimension research checklist, conflict cards that must be resolved at Gate 1, pre-flight Jira field validation before bulk creation.
- **Safe change propagation** — `/change-mode` computes blast radius and walks you through diff-by-diff approval. `/reopen-gate-N` unwinds without losing downstream artifacts.
- **Cross-feature memory** — `_knowledge-base.md` accumulates lessons. Future PRDs surface relevant past decisions before web search runs.

---

## Version

**v2.1.0** — See [CHANGELOG.md](./CHANGELOG.md) for full version history.

## License

MIT
