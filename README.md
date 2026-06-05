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
| Visual diagram in Figma FigJam (temporary Mermaid fallback only if the Figma MCP isn't connected) | Figma + your computer |
| Screen design prompts for v0 or Figma Make | Your computer |
| Optional: Real Figma frames generated from the prompts via Figma MCP, wired to your team's design system (one page per category, mid-fi but on-brand) | Figma |
| User Stories Breakdown with acceptance criteria (Gherkin or plain English — your choice), FE/BE pairs, sizing | Your computer |
| Gantt timeline (Figma FigJam + interactive HTML) at Epic + Phase level | Figma + your computer |
| Jira Epic + Stories with labels, links, and custom fields | Your team's Jira |
| Optional: Confluence feature hub with a numbered child page per artifact — Research, Codebase Review, PRD, System Design, Visual Diagram, Design Catalog per phase, Timeline, plus a lightweight User Stories page that links to each Jira Epic. Product/Technical Reviews and the full Gherkin breakdown stay local. | Your team's Confluence |
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
| 7 | **Visual Diagram** — Figma FigJam via Figma MCP (the deliverable); temporary, clearly-labeled Mermaid fallback only if the MCP isn't connected | AUTO | `diagrams/` |
| 8 | **Design Prompts** — Structured v0 or Figma Make prompts per screen, per state | AUTO | `design/` |
| 9 | **Update PRD from Designs → Gate 2** — Surgical sync, your approval | **GATE** | `prd/` (updated) |
| 10 | **User Stories Breakdown → Gate 3** — Exhaustive acceptance criteria with a **format fit-check** (Gherkin vs. plain-English criteria — the skill shows you one real story both ways and you pick), FE/BE pairs, multi-epic grouping, **wave sequencing** (topological sort on dependencies — cycles are CRITICAL findings), optional DRAFT mode for stories without finalized designs | **GATE** | `user-stories/` |
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
| **Confluence** | Atlassian MCP | Step 11c (publishes feature hub + child page per stakeholder-relevant artifact, with per-file mtime change detection. v2.8.0+: Product/Technical Reviews and the full Gherkin breakdown are kept local; the Step 10 page is a lightweight Jira Epic index instead), `/publish-to-confluence`, `/read-feedback` |
| **Google Drive** | Drive MCP | Step 11b, `/drive-sync` |
| **Figma** | Figma MCP | Step 7 — FigJam diagram generation. Also `/push-to-figma` (generate one Figma frame per screen from the design prompts file, wired to the team's design system) and `/pull-from-figma` (refresh the catalog + PRD + stories from the iterated Figma file after the designer has refined the pushed frames). |
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
| `/visual-diagram` | Figma FigJam diagram from a PRD (Figma is the deliverable; temporary Mermaid fallback only if the MCP isn't connected, flagged to regenerate as Figma) |
| `/user-stories` | User Stories Breakdown from an approved PRD. Multi-epic grouping (skill proposes, PM tunes). Wave sequencing via topological sort on dependencies. DRAFT mode for missing designs — refresh later via `/change-mode` → "Designs arrived" |
| `/timeline` | Gantt timeline (Figma FigJam + editable HTML) at Epic + Phase level. Drag-to-shift / drag-edge-to-resize with auto-cascade in the browser. 💾 Save to skill writes plan JSON directly to disk (Chrome/Edge). `/timeline apply` (no args) auto-discovers the latest plan and round-trips it into the skill state. |
| `/prd-to-jira` | Create Jira tickets from a breakdown or PRD |
| `/exec-summary` | Synthesize PRD + system design + user stories + timeline + decision log into a single ~20 KB / ~5–7 PDF page executive summary (markdown + PDF + DOCX). 9-section structure: vision, use cases, moving pieces, teams, timeline, open decisions, risks, the ask, what's next. Standalone — not auto-invoked by `/build-product`. Idempotent on re-run. |
| `/create-slidedeck` | Turn a feature's artifacts into a presentation-ready slide deck. **Always runs a full interview** (deck type, audience, goal, slide count, depth, tone/branding, sources, speaker notes, render surfaces) and confirms a one-line-per-slide outline first. Presets: `exec` / `kickoff` / `demo` (+`custom`). Writes a **slide-spec markdown** (source of truth), always emits a **paste-into-Claude/Figma-Make deck-prompt**, and optionally renders **Figma Slides** (Figma MCP) and/or self-contained **HTML + PDF** (same toolchain as `/exec-summary`). Pipeline-aware; works standalone on pasted content. Idempotent per deck-type. Output → `slides/`. |
| `/infosec-doc` | Fill out Bright's **InfoSec Questionnaire** (the Ops/DevOps security-review `.xlsx`) for a feature. Derives answers across all 7 tabs from the PRD, system design, technical review, codebase review, diagrams, and `_pipeline-state.json`, then **batch-interviews** the PM for what no artifact carries (severity, escalation/emergency contacts, DR region, vendor SLA, approved-AI-list + opt-out, legal sign-off). **Never fabricates** a security answer — each cell is sourced, PM-confirmed, or `⚠ NEEDS INPUT`; standard defaults (TLS/AES) are flagged for confirmation. Writes the real workbook via `openpyxl` (formatting + severity dropdown preserved) to `infosec/[feature]-infosec-questionnaire.xlsx`. Standalone — not auto-invoked. Idempotent. |
| `/drive-sync` | Sync artifacts to Google Drive |
| `/publish-to-confluence` | Publish the whole feature workspace as a Confluence hub + one numbered child page per artifact. Per-file mtime tracking — only re-publishes pages whose source actually changed. **v2.14.0+** — Step 3.5 auto-pushes Mermaid diagrams + timelines to Figma via `/visual-diagram` and `/timeline` when the Figma MCP is connected and no URL is in state, then embeds the resulting iframes. **v2.15.0+** — pre-flight drift + comment check on every `update` page: fetches Confluence version + inline + footer comments before overwriting, surfaces `🚨 DRIFT` (someone edited the page outside the skill), `⚠ N inline comments at risk of orphaning`, and per-page resolution actions (proceed / skip / pull-comments to sidecar / show-drift / show-comments). Drift never auto-resolves — explicit PM choice required. |
| `/export-transcript` | Write the full PM↔model conversation for a feature's pipeline run to two markdown files (clean reading version + full forensic version). |
| `/pipeline-timing` | Wall-clock + active-work timing report for a feature's pipeline run. Reads instrumented timestamps; JSONL fallback. |
| `/share-for-review` | Post a Confluence link to Slack with reviewers and a deadline |
| `/read-feedback` | Pull Confluence comments, synthesize into PRD edits, re-sync |

### Design

| Command | What it does |
|---------|--------------|
| `/design-prompts` | Screen design prompts for v0 or Figma Make |
| `/push-to-figma` | **v2.12.0+** — Generate real, editable Figma frames programmatically from the design prompts file via the Figma MCP. Wires every frame to the team's design system color variables (and components where they fit); when brand tokens change, every frame updates. One Figma page per category (A, B, C, …) with sub-frames laid out horizontally. Frame dimensions picked from intake's product type (mobile 390×844 or web 1440×900). Output catalog at `design/[feature]-figma-catalog.md` with every frame's direct node URL. Companion to `/design-prompts`; typical flow is `/design-prompts` → PM review → `/push-to-figma`. No v0 equivalent — v0 has no programmatic-push API. |
| `/pull-from-figma` | **v2.13.0+** — Reverse direction of `/push-to-figma`. Pulls the post-iteration state of a Figma file back into the feature workspace after the designer has refined frames. Refreshes the design catalog with real screenshots + URLs, surfaces designer changes (renamed / deleted / added frames, token swaps), then optionally diffs against the PRD and user stories and offers to apply updates. Standalone (not in the auto-run pipeline) because designers iterate asynchronously. Screenshots are saved to a dated `figma-pulls/[YYYY-MM-DD]/` subfolder so old snapshots are preserved for audit. Read-only on Figma; writes only to the local workspace (and Jira via MCP if connected). Closes the loop: `/design-prompts` → `/push-to-figma` → designer iterates → `/pull-from-figma` → PRD + stories synced. |
| `/update-prd-from-designs` | Sync PRD with finalized design catalog |
| `/compare-figma-prd` | Figma vs PRD and Jira gap analysis after designer delivers |

### Utilities

| Command | What it does |
|---------|--------------|
| `/lint-style` | Check any document against `style-preferences.md` |
| `/pipeline-doctor` | Scan the skill and feature workspaces for **structural** drift — missing steps, broken slash commands, stale features, state-file inconsistencies. Read-only with per-finding fix approval; writes a timestamped report. |
| `/validate-prd` | **Semantic** content validation for a PRD. Six checks (consistency, hallucinated data, completeness, VOC traceability, NFR measurability, scope coherence). Per-finding fix approval + timestamped report. |
| `/validate-user-stories` | **Semantic** content validation for the user-stories breakdown. Seven checks (PRD traceability, AC duplication / contradiction, FE/BE pair coherence, AC specificity, sizing sanity, DRAFT consistency, wave / dependency sanity). Most expensive command in the skill — best run on-demand before Gate 3 or `/prd-to-jira`. |
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

**v2.21.0** — **No more front-matter plumbing junk at the top of docs.** Kills the `Status: v1.6 — readability reformat… / Predecessor: / Refactor authority: / Refactor lineage: / Last updated:` block that accreted at the top of the user-stories doc over repeated `/change-mode` runs. The front matter is now a strict ≤ 8-line allowlist (what this is, what it was generated from, current counts/mode, a one-line changelog pointer); the version/refactor narrative is banned everywhere — it lives in the centralized changelog. v2.18.0 banned change history under `## ` headings; this closes the front-matter-disguise loophole, adds a `/change-mode` "don't grow the front matter" guard, and a `/validate-user-stories` Check 8c to catch it. See [CHANGELOG.md](./CHANGELOG.md) for full version history.

**v2.20.0** — **Figma is the single surface for diagrams.** The old rule that kept both a Figma link *and* an always-on Mermaid block in every diagram doc is reversed: Figma/FigJam is now the deliverable, and Mermaid is only a temporary, clearly-labeled fallback when the Figma MCP isn't connected (flagged in state to regenerate as Figma on a re-run). Reason — Mermaid doesn't render reliably across the tools the team opens, and the product team works in Figma. Timeline (FigJam + HTML), designs (v0 / Figma Make), and `/system-design`'s inline ASCII sketches were already Figma/non-Mermaid and are unchanged. See [CHANGELOG.md](./CHANGELOG.md) for full version history.

**v2.19.0** — **Acceptance criteria stop assuming Gherkin.** A new Step 2.7 fit-checks the AC format for each feature — Gherkin suits behavioral/multi-path stories, plain-English criteria read clearer for simple/config/spike work — and shows the PM one real story written **both ways, side by side**, before they pick (Gherkin / plain English / mixed). The choice flows through authoring, validation, Gate 3, and Jira export. The Jira export now also explicitly fills all three story fields — Description ← the plain-English description, Acceptance Criteria ← the AC in its chosen format (verbatim, never converted), User Story ← the "As a… I want… so that…" narrative. See [CHANGELOG.md](./CHANGELOG.md) for full version history.

**v2.18.0** — **Process-tracking content separated from reader docs.** Change/refactor history is banned inline anywhere in an artifact (by any heading — `## Change log`, `## Refactor history`, etc.) and consolidated into one feature-level `changelog/[feature]-changelog.md` with a section per artifact. The **decision log moves out of the PRD** into a `decisions/[feature]-decision-log.md` sidecar — the PRD's § 10 is now a one-line pointer, with every producer/consumer step rewired to the sidecar. Closes two leaks (User Stories Appendix B refactor history; `/pull-from-figma` inline change log) and adds `/validate-user-stories` Check 8b to catch inline change history anywhere in a breakdown. See [CHANGELOG.md](./CHANGELOG.md) for full version history.

**v2.17.0** — **Intake Q3 becomes a suggested-defaults checklist + Jira conventions persist across features.** The Jira ticket-conventions question no longer asks from a blank slate — it presents proposed conventions the PM confirms or edits: every-ticket labels, per-layer `backend`/`frontend` labels, title format with `[BE]`/`[FE]` layer prefix (explicitly asking how BE vs. FE should be marked in titles), BE/FE split, Testable Yes/No on every ticket, Story Points left blank, and BE↔FE "Relates to" / "Blocked by" link conventions. Q2 now also captures the **board** for teams using shared boards with Components as swim lanes. Answers save once to a **durable conventions profile** (`_jira-conventions.json`) and are confirm-reused ("reuse / edit / start fresh") on every future run instead of re-asked. See [CHANGELOG.md](./CHANGELOG.md) for full version history.

**v2.15.0** — **`/publish-to-confluence` adds pre-flight drift + comment check.** Every page resolving to `update` now runs three lightweight MCP calls before the publish plan is presented: `getConfluencePage` (drift), `getConfluencePageInlineComments` (orphan risk), `getConfluencePageFooterComments` (safe — context only). The publish plan surfaces `🚨 DRIFT (v[N] > last-published v[M])` when someone edited the page in Confluence outside the skill, and `⚠ N inline + M footer` with author + anchor-text excerpts when comments are at risk of orphaning. Per-page resolution actions: **proceed** (overwrites; drift + inline anchors may go), **skip** (leave this page alone this run), **pull-comments** (write a structured comment dump to `confluence-feedback/[YYYY-MM-DD]/[step-N]-comments.md` and skip the publish for that page — non-destructive path), **show-drift** (unified diff of current Confluence body vs about-to-publish body), **show-comments** (full comment thread bodies). Drift never auto-resolves — explicit PM choice required. New state field `last_published_version` per artifact; legacy state files migrate gracefully via `last_published_version = 0`. See [CHANGELOG.md](./CHANGELOG.md) for full version history.

## License

MIT
