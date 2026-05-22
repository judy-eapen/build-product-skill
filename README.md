# /build-product

A Claude Code skill for product managers. It takes a feature idea through research, a PRD, dual AI review, system design, visual diagrams, screen design prompts, an exhaustive user-stories breakdown, a Gantt timeline, and finally to a complete set of Jira tickets — in one continuous Work pipeline with three human approval gates.

> **New to Claude Code or this skill?** Read [GETTING-STARTED.md](./GETTING-STARTED.md) for the full setup walkthrough. It assumes no prior experience with terminals, Git, or MCP integrations.

📄 **[Interactive pipeline explainer →](https://judy-eapen.github.io/build-product-skill/)**

---

## Who this is for

Product managers at companies that use Jira and want to spend less time on the mechanical parts of feature documentation: writing PRDs from scratch, hand-creating tickets, translating decisions into Gherkin acceptance criteria, and keeping designers and engineers in sync as scope evolves.

You do not need to know how to code. You do need:

- A Claude account on the Pro or Team plan
- Claude Code installed on your computer
- Access to your team's Jira

Full setup instructions are in [GETTING-STARTED.md](./GETTING-STARTED.md).

---

## What you get for every feature

| Artifact | Where it lives |
|----------|----------------|
| Research document | Your computer (`~/Desktop/Resources/PDLC Workflow Docs/[feature]/`) |
| 11-section PRD | Your computer |
| Product Review + Technical Review (parallel AI critiques) | Your computer |
| System Design doc (if architecture is non-trivial) | Your computer |
| Visual diagram (Figma FigJam if connected, Mermaid otherwise) | Figma + your computer |
| Screen design prompts for v0 or Figma Make | Your computer |
| User Stories Breakdown with Gherkin AC, FE/BE pairs, sizing | Your computer |
| Gantt timeline (Figma FigJam + interactive HTML) at Epic + Phase level | Figma + your computer |
| Jira Epic + Stories with labels, links, and custom fields | Your team's Jira |
| Optional: Confluence feature hub with a numbered child page per artifact (Research, Codebase Review, PRD, Product/Technical Review, System Design, Visual Diagram, Design Catalog per phase, User Stories, Timeline) | Your team's Confluence |
| Optional: Google Drive folder mirroring everything | Your team's Drive |
| Conversation transcript — clean reading version + full forensic version of every message you sent and reply you got during the pipeline run | Your computer |
| Pipeline timing report — wall-clock total, active-work total, per-step breakdown, per-gate wait time | Your computer |

You approve at three gates (PRD, Designs, User Stories) before anything moves to external systems.

---

## The 13-step pipeline at a glance

| # | Step | Mode | Output |
|---|------|------|--------|
| 1 | **Research Idea** — Interactive discovery, knowledge-base check, web scan | AUTO | `research/` |
| 2 | **Codebase Review** — Reads folder structure, samples files, rates risks | AUTO | `codebase-review/` |
| 3 | **Create PRD** — 11-section PRD, scoping-aware, auto-linted | AUTO | `prd/` |
| 4 | **Dual Review** — Two parallel agents: Product Reviewer + Technical Reviewer | PARALLEL | `product-review/` + `technical-review/` |
| 5 | **Apply Fixes → Gate 1** — Quality checks, conflict cards, your approval | **GATE** | `prd/` (updated) |
| 6 | **System Design** — Architecture, data model, build order (optional) | AUTO | `technical-review/` |
| 7 | **Visual Diagram** — Figma FigJam via Figma MCP (falls back to Mermaid) | AUTO | `diagrams/` |
| 8 | **Design Prompts** — Structured v0 or Figma Make prompts per screen, per state | AUTO | `design/` |
| 9 | **Update PRD from Designs → Gate 2** — Surgical sync, your approval | **GATE** | `prd/` (updated) |
| 10 | **User Stories Breakdown → Gate 3** — Exhaustive Gherkin AC, FE/BE pairs, multi-epic grouping, optional DRAFT mode for stories without finalized designs | **GATE** | `user-stories/` |
| 10.5 | **Timeline (Gantt)** — Figma FigJam + interactive HTML at Epic + Phase level; hybrid estimation (skill proposes, PM tunes); HTML is editable in-browser (drag bars, auto-cascade); **💾 Save to skill** writes plan JSON directly to disk in Chrome/Edge; `/timeline apply` auto-discovers the latest plan | AUTO | `timeline/` |
| 11 | **Export** — Jira tickets always; Google Drive + Confluence (hub + child page per artifact) optional, parallel | PARALLEL | Jira + Drive + Confluence |
| 12 | **Export Transcript** — Reads the live session JSONL and writes the full PM↔model conversation to two markdown files (clean + forensic) | AUTO | `transcript/` |

---

## Installation summary

For experienced users. New users should read [GETTING-STARTED.md](./GETTING-STARTED.md).

```bash
# 1. Clone the skill
git clone https://github.com/judy-eapen/build-product-skill.git ~/.claude/skills/build-product

# 2. Register slash commands (one-time)
mkdir -p ~/.claude/commands && for f in ~/.claude/skills/build-product/subprompts/*.md; do n=$(basename "$f" .md); printf 'Read and follow `~/.claude/skills/build-product/subprompts/%s.md`.\n' "$n" > ~/.claude/commands/"$n.md"; done

# 3. Open Claude Code, type `/build-product`
```

---

## Update to the latest version

To pull the latest skill changes (and pick up any new slash commands added since your last install), paste this one-liner in your terminal:

```bash
cd ~/.claude/skills/build-product && git pull && mkdir -p ~/.claude/commands && for f in subprompts/*.md; do n=$(basename "$f" .md); printf 'Read and follow `~/.claude/skills/build-product/subprompts/%s.md`.\n' "$n" > ~/.claude/commands/"$n.md"; done
```

It does two things in sequence:

1. Pulls the latest commits from GitHub into your local skill folder.
2. Re-registers all slash commands so any new commands added in this version become available.

Restart Claude Code after running it.

To see what changed in this update, check [CHANGELOG.md](./CHANGELOG.md).

---

## Optional integrations

| Integration | MCP | Used for |
|-------------|-----|----------|
| **Jira** | Atlassian MCP | Step 11a — ticket creation (required for the export step) |
| **Confluence** | Atlassian MCP | Step 11c (publishes feature hub + child page per artifact, per-file mtime change detection), `/publish-to-confluence`, `/read-feedback` |
| **Google Drive** | Drive MCP | Step 11b, `/drive-sync` |
| **Figma** | Figma MCP | Step 7 — FigJam diagram generation |
| **Slack** | Slack MCP | `/share-for-review` — post review links with reviewers tagged |

Any integration the skill cannot reach is skipped cleanly. It never blocks the rest of the pipeline.

---

## All commands

### Pipeline

| Command | What it does |
|---------|--------------|
| `/build-product` | Full Work pipeline — research through Jira export |
| `/change-mode` | Propagate a scope change after any gate across all artifacts. Seven trigger types including "Designs arrived" (refreshes DRAFT stories + Jira tickets) |
| `/reopen-gate-1` | Unwind Gate 1 approval; re-run steps before it |
| `/reopen-gate-2` | Unwind Gate 2 approval; re-run steps before it |
| `/reopen-gate-3` | Unwind Gate 3 approval; re-run steps before it |

### Standalone steps

| Command | What it does |
|---------|--------------|
| `/research-idea` | Research stage only |
| `/codebase-review` | Codebase review only |
| `/create-prd` | Generate a PRD from research or a brief |
| `/review-prd` | Product Reviewer pass on an existing PRD |
| `/cto-review` | Technical Reviewer pass on an existing PRD |
| `/system-design` | System design doc from a PRD |
| `/visual-diagram` | Figma FigJam diagram from a PRD (Mermaid fallback) |
| `/user-stories` | User Stories Breakdown from an approved PRD. Multi-epic grouping (skill proposes, PM tunes). DRAFT mode for missing designs — refresh later via `/change-mode` → "Designs arrived" |
| `/timeline` | Gantt timeline (Figma FigJam + editable HTML) at Epic + Phase level. Drag-to-shift / drag-edge-to-resize with auto-cascade in the browser. 💾 Save to skill writes plan JSON directly to disk (Chrome/Edge). `/timeline apply` (no args) auto-discovers the latest plan and round-trips it into the skill state. |
| `/prd-to-jira` | Create Jira tickets from a breakdown or PRD |
| `/drive-sync` | Sync artifacts to Google Drive |
| `/publish-to-confluence` | Publish the whole feature workspace as a Confluence hub + one numbered child page per artifact. Per-file mtime tracking — only re-publishes pages whose source actually changed. |
| `/export-transcript` | Write the full PM↔model conversation for a feature's pipeline run to two markdown files (clean reading version + full forensic version). |
| `/pipeline-timing` | Wall-clock + active-work timing report for a feature's pipeline run. Reads instrumented timestamps; JSONL fallback. |
| `/share-for-review` | Post a Confluence link to Slack with reviewers and a deadline |
| `/read-feedback` | Pull Confluence comments, synthesize into PRD edits, re-sync |

### Design

| Command | What it does |
|---------|--------------|
| `/design-prompts` | Screen design prompts for v0 or Figma Make |
| `/update-prd-from-designs` | Sync PRD with finalized design catalog |
| `/compare-figma-prd` | Figma vs PRD and Jira gap analysis after designer delivers |

### Utilities

| Command | What it does |
|---------|--------------|
| `/lint-style` | Check any document against `style-preferences.md` |
| `/team-status` | Portfolio dashboard: all features, phases, owners, blockers |
| `/feature-kickoff` | Role-specific briefing for an engineer or designer picking up a feature |
| `/project-status` | Pipeline state and next step for a single feature |
| `/prioritize` | RICE, MoSCoW, or Value-vs-Effort prioritization |
| `/meeting-notes` | Parse raw meeting notes into decisions, actions, and next steps |
| `/learn-codebase` | Plain-language walkthrough of any codebase |

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
    ├── timeline/
    ├── transcript/
    ├── timing/
    ├── jira-export/
    ├── changelog/
    ├── stakeholders/
    ├── _pipeline-state.json    ← resumable session state
    ├── _context-checkpoint.md  ← compact context for session resumption
    └── _open-conditions.md     ← conditions from "approved with conditions" gates
```

---

## Customize

Common starting points if you want to adjust skill behavior:

- `ai-framework/style-preferences.md` — your writing style rules
- `ai-framework/01-research-idea.md` — the 15-dimension Interactive Discovery checklist
- `ai-framework/02-create-prd.md` — PRD sections and scoping logic
- `subprompts/prd-to-jira.md` — how tickets are composed and which Jira custom fields are filled
- `CLAUDE.md` — intake parameters every PM answers at pipeline start

As of v2.1.0, Jira conventions (labels, title format, BE/FE split, custom field defaults, link conventions) are captured at intake rather than encoded in a personal `CLAUDE.md`. See [CHANGELOG.md](./CHANGELOG.md) for details.

---

## Architecture notes

- **Resumable.** `_pipeline-state.json` is written at the end of every step and read first in every new session. Gate approvals, artifact paths, and export URLs are all persisted.
- **True parallelism.** Dual Review (Step 4) and Export (Step 11) use the Claude Agent tool to run isolated sub-agents simultaneously, not role-switching in a single context.
- **Forcing functions.** 15-dimension research checklist; conflict cards that must be resolved at Gate 1; pre-flight Jira field validation before bulk creation.
- **Safe change propagation.** `/change-mode` computes blast radius and walks through diff-by-diff approval. `/reopen-gate-N` unwinds without losing downstream artifacts.
- **Cross-feature memory.** `_knowledge-base.md` accumulates lessons. Future PRDs surface relevant past decisions before web search runs.

---

## Version

**v2.6.0** — Two refinements to the v2.5.0 editable Gantt that cut the round-trip down to two steps. (1) New **💾 Save to skill** button uses the File System Access API to write the plan JSON directly into the feature's `timeline/` folder (Chrome/Edge); Safari/Firefox fall back to download. (2) `/timeline apply` now accepts zero arguments and auto-discovers the latest plan JSON in the feature's `timeline/` folder or `~/Downloads/`. Typical flow: edit in browser → 💾 Save → `/timeline apply` in chat. See [CHANGELOG.md](./CHANGELOG.md) for full version history.

## License

MIT
