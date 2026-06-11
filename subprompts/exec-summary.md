# Executive Summary

Synthesize a feature's full pipeline artifacts (PRD, system design, user-stories breakdown, timeline, decision log) into a single exec-readable document. Use when:

- A stakeholder or executive asks for the "what is this and why" without wading through a 170 KB PRD.
- You're about to ask leadership to greenlight engineering spend on a feature.
- You want a sharable artifact for cross-functional alignment (Legal, Finance, Marketing) that doesn't require deep domain context.
- The PRD changed materially and the previously-generated exec summary is stale.

Standalone command. Does **not** run automatically inside `/build-product` — the PM invokes it when an exec doc is needed.

---

## Inputs

The skill collects inputs at Step 0:

- **Required:** PRD path (asked if not in conversation context). If no PRD exists, refuse to proceed and direct PM to `/create-prd` first.
- **Recommended** (skill reads if present, gracefully skips if not):
  - System design — pulls architecture / moving-pieces / vendor list.
  - User-stories breakdown — pulls scope count, phase structure, wave assignments, deferred items.
  - Timeline — pulls computed end date, gap-vs-target, external dependency milestones.
  - Decision log — pulls "open decisions needed" section.
  - Codebase review — pulls cross-cutting risks for the risks section.
- **Asked at Step 0:**
  - Feature name (auto-derived from working folder if running inside one).
  - Executive sponsor name (for the masthead).
  - PM/owner name (auto-filled from `_pipeline-state.json` → `intake.pm_name` if present; asked otherwise).

If a recommended input is missing, the skill notes the absence in a bracketed `[NOTE: X not available, section reduced]` annotation in the affected section rather than failing.

---

## Output

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/executive-summary/
├── [feature-name]-executive-summary.md   ← source markdown
├── [feature-name]-executive-summary.pdf  ← shareable PDF
└── [feature-name]-executive-summary.docx ← Word-editable copy
```

All three files live in the feature's `executive-summary/` subfolder (not the feature root) — this keeps the executive summary grouped as its own deliverable and parallels how `prd/`, `timeline/`, `user-stories/`, etc. are organized. Create the folder if it doesn't exist.

**Idempotent.** Re-running the skill on the same feature overwrites all three files. The skill does **not** maintain inline version history in the markdown — per the workspace's artifact conventions, change history lives in `changelog/[feature]-changelog.md`. The exec summary's top line carries the current PRD version it was generated against, nothing more.

**Length target: ~20 KB markdown / ~5–7 PDF pages.** Long enough to cover all sections substantively, short enough that a busy exec reads it cover-to-cover.

---

## Document structure (9 sections, exact order)

The output must follow this skeleton. Section order is fixed; section length adapts to what the inputs support.

### Masthead (~3 lines)

```
# [Feature Name] — Executive Summary

**Target launch:** [month year] · **Status:** PRD v[X.Y] [gate-status] · **Owner:** [PM name] · **Sponsor:** [exec sponsor]
**Change history:** see [changelog/[feature]-changelog.md](changelog/[feature]-changelog.md)
```

No "Document version" line. No "Last updated" line. No inline changelog block.

### 1. The vision in one paragraph

Single paragraph, 4–6 sentences. State **what** the feature is, **who** it's for, and the **north star** (a vivid concrete agent/user scene). Pulled from PRD §1 Executive Summary, but compressed. No jargon. No acronyms without expansion on first use.

### 2. What [v1] does for users — N use cases

A single table with three columns: Category / Examples / Tools-or-systems-serving. One row per major use-case category, not per individual story. Count of use cases shown in the section heading. Pulled from PRD §2.

### 3. The moving pieces — who builds what

Three subsections, each a bulleted list (2–6 bullets each):

- **External vendors building for us** — third-party integrations and what each owns. Pulled from system design + PRD §5 (external services).
- **Internal systems we're building or modifying** — orchestrator, ingestion platform, mobile app, etc. Pulled from system design.
- **Cross-cutting work we own** — observability, security, AI safety, legal-copy slots, accessibility, etc. Pulled from PRD §3-§5.

Each bullet is one sentence: **what the piece is** + **who owns it** (internal team or vendor name). No deep dives.

### 4. Teams & roles — who needs to be aligned

A table or stacked list. For each team/role: name, primary work scope, key dependencies. Pulled from `_pipeline-state.json` → `team_composition` if present, supplemented from system design and timeline external dependencies. Include external vendors as their own row(s) (e.g., "Vendor contract manager — owns integration delivery, blocks downstream FE work").

### 5. Critical timeline alignment

Three short paragraphs:

1. **The plan** — start date, computed end, target launch, gap (working days ahead/behind). Pulled from `_pipeline-state.json` → `timeline.computed`.
2. **What has to happen by when** — top 3–5 external milestones with owners and dates. Pulled from `timeline.external_dependencies`.
3. **What slips if things slip** — one or two specific cascades (e.g., "if X contract slips into Phase 1, use cases L8–L11 defer to v1.x; v1 still ships").

### 6. Open decisions needed

A bulleted list of unresolved open questions with: **ID, one-sentence statement, who decides, by when**. Pulled from `_pipeline-state.json` → `open_questions`. Include both PRD open questions (O-series) and timeline open questions (OT-series). Filter out resolved entries.

If everything is resolved, write: `All open questions resolved as of [date]. Pipeline ready to advance.`

### 7. Risks & how we're managing them

A table with three columns: Risk / Likelihood-impact / Mitigation. 4–8 rows. Pulled from:
- PRD §[risks section if present]
- Technical review HIGH/MEDIUM findings
- AI safety review HIGH/MEDIUM findings (if applicable)
- Timeline risk callouts

Do not invent risks. If a risk source artifact doesn't exist, note its absence.

### 8. What we're asking executives to greenlight

A short numbered list (3–6 items) of the specific decisions, approvals, or budget items the exec audience needs to confirm. Pulled from PRD §[ask section if present] or synthesized from open questions + scope decisions. Each item must be actionable, not aspirational.

Examples of good asks:
- "Approve REA contract scope at $X for 12 base + 2 conditional MCP tools, signed by [date]."
- "Confirm October 2026 launch target given +8 working-day buffer in current plan."
- "Greenlight engineering investment of [N] dev-months across [M] sprints."

### 9. What's next

A short paragraph (2–4 sentences) on the immediate next 2–3 milestones with dates. Pulled from `_pipeline-state.json` → `current_step` + `next_step`. End with a pointer to the full PRD path for execs who want depth.

### Footer (~1 line)

```
---

*End of [Feature Name] — Executive Summary.*
```

No version tagline. No "Generated [date]" line. No changelog block.

---

## Tone rules

- **Plain English.** Spell out acronyms on first use (MCP = Model Context Protocol; FCM = Firebase Cloud Messaging; etc.). If an acronym is core to the product and unavoidable (e.g., MLS for a real-estate product), define it once in §1.
- **Active voice.** "Platform owns the ingestion pipeline" not "the ingestion pipeline is owned by Platform."
- **No engineering minutiae.** No code, no API contract examples, no schema fragments. If you find yourself writing `SELECT` or curly braces, you're in the wrong document.
- **Concrete numbers** where present in the source artifacts. "27 use cases," "79 working days," "12 base tools + 2 conditional," "+8 days ahead of target." Avoid vague qualifiers like "many," "several," "significantly."
- **Exec-readable density.** Aim for one substantive idea per sentence. Bullets over paragraphs where a list is the natural shape.
- **No hedging filler.** Remove "we believe," "we think," "potentially," "may," "could possibly" unless they're load-bearing. State what the PRD says, attribute uncertainty to specific open questions with IDs.
- Apply `ai-framework/style-preferences.md` over everything else.

---

## Generation steps

### Step 0 — Confirm inputs

- Resolve feature folder. If running inside one, auto-detect. Otherwise ask: "Which feature?"
- Read `_pipeline-state.json` if present. Extract: feature_name, current PRD path + version, system_design path, user_stories path, timeline (computed + external_dependencies), open_questions, team_composition, current_step, next_step, intake (pm_name, sponsor if present).
- If `_pipeline-state.json` is missing or incomplete, ask the PM directly for the missing items.
- Confirm to the PM: "I'll generate the exec summary for `[feature]` against PRD v[X.Y]. Output will overwrite the existing exec summary if one is present. Proceed?"

### Step 1 — Read source artifacts in parallel

Read these files in parallel:
- PRD (always)
- System design (if present)
- User-stories breakdown — sections needed: header, Build Sequence Map / Wave Summary, Appendix C (deferred items)
- Timeline markdown — sections needed: parameter snapshot, external dependency milestones, risk callouts
- Decision log — the sidecar `decisions/[feature]-decision-log.md` (the PRD's § 10 is only a pointer to it)
- Technical review + AI safety review (if present) — for risks section

For large artifacts (PRD often >150 KB), read strategically: §1, §2, §3 (roles & permissions), §5 (architecture overview), §11 (open questions), §[risks if present], §[ask if present]. Skip §4 (deep technical), §6 (deep UX), §7 (deep phase detail), §8 (deep NFR). The decision log is no longer inline in the PRD — read it from the sidecar above. Those deep PRD sections are PRD-only territory.

### Step 2 — Compose

Write the 9 sections in order. For each section, before writing:
- Identify the source artifact(s).
- If sources are missing, write the section with a `[NOTE: ...]` annotation rather than fabricating.
- Cap section length per the budget below.

**Section length budget** (approximate, to hit the 20 KB total):

| Section | Target lines |
|---|---|
| Masthead | 3 |
| 1. Vision | 6–10 |
| 2. Use cases | 15–25 (table) |
| 3. Moving pieces | 20–30 |
| 4. Teams & roles | 15–25 |
| 5. Timeline alignment | 10–15 |
| 6. Open decisions | 8–15 |
| 7. Risks | 10–15 (table) |
| 8. Ask | 6–12 |
| 9. What's next | 3–6 |
| Footer | 2 |

If composed output materially exceeds 25 KB, re-tighten the densest section before writing.

### Step 3 — Write markdown

Write to `[feature-folder]/executive-summary/[feature-name]-executive-summary.md`. Create the `executive-summary/` folder if it doesn't exist. Overwrite the markdown if it exists. Print the path before writing.

### Step 4 — Generate PDF + DOCX

Convert the markdown into `executive-summary/[feature-name]-executive-summary.pdf` and `executive-summary/[feature-name]-executive-summary.docx`, alongside the markdown. Overwrite if present.

**Preferred toolchain when available (no install required on macOS):**

1. **DOCX:** `pandoc input.md -o output.docx --from=gfm` — pandoc handles markdown → DOCX directly.
2. **PDF:** if a LaTeX engine (pdflatex/xelatex) is installed, use `pandoc input.md -o output.pdf --pdf-engine=xelatex`. If not, do this two-step instead:
   - `pandoc input.md -o /tmp/output.html --from=gfm --standalone --css=/tmp/style.css --embed-resources` to get a styled HTML
   - Chrome headless to print the HTML to PDF: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu --print-to-pdf=output.pdf --no-pdf-header-footer file:///tmp/output.html`

A reasonable CSS for engineering-style PDFs (Georgia serif body, Helvetica headings, light-slate code blocks, table borders, page-break-before on H1) can be inlined directly or sourced from the skill's local assets directory if present. Reuse or duplicate inline.

**Alternative paths:** the `anthropic-skills:pdf` and `anthropic-skills:docx` skills may be invoked if pandoc/Chrome aren't available.

**If no conversion path works:** write the markdown only and tell the PM "PDF + DOCX generation tooling not available in this environment — share the markdown directly or convert manually." Do not fail the whole run.

### Step 5 — Update pipeline state

If `_pipeline-state.json` exists for this feature, update `artifacts.exec_summary` block:

```json
"exec_summary": {
  "path": ".../executive-summary/[feature-name]-executive-summary.md",
  "pdf_path": ".../executive-summary/[feature-name]-executive-summary.pdf",
  "docx_path": ".../executive-summary/[feature-name]-executive-summary.docx",
  "generated_against_prd_version": "v[X.Y]",
  "generated_at": "[ISO timestamp]"
}
```

Do **not** add a change_history entry — the exec summary is regenerated freely and doesn't warrant a propagation record on its own. (When the PRD changes via `/change-mode`, the change_history entry written by that pass is the relevant audit trail.)

### Step 6 — Output

Print:

```
✓ Executive summary generated → [feature]/executive-summary/[feature]-executive-summary.md
  PDF: [feature]/executive-summary/[feature]-executive-summary.pdf
  DOCX: [feature]/executive-summary/[feature]-executive-summary.docx
  PRD version covered: v[X.Y]
  Length: [N] lines / ~[K] KB
```

---

## Re-runnability

The skill is idempotent — re-running on the same feature overwrites all three output files. It does not preserve old versions inline. If you want a snapshot from a previous date, retrieve it from git history or from the `changelog/` folder where prior runs are recorded.

---

## Failure modes

- **PRD missing.** Refuse to proceed. Direct PM to `/create-prd` first.
- **PRD present but pre-Gate-1 / unstable.** Run anyway; mark `Status: PRE-GATE-1 DRAFT — content may shift before exec review.` in the masthead. The skill does not block on gate state.
- **All recommended inputs missing (system design, user stories, timeline absent).** Run from PRD-only — the moving-pieces section becomes thinner, the timeline section becomes "TBD, see PRD §[phase section]," but the doc still ships. Flag at the top: `[PRD-only synthesis — downstream artifacts not yet produced]`.
- **PDF/DOCX tooling missing.** Write markdown only, notify PM. Do not block.

---

## What this command does NOT do

- It does not modify the PRD or any other source artifact. Read-only against source files.
- It does not validate the PRD or surface inconsistencies (that's `/validate-prd`).
- It does not check for drift between PRD and downstream artifacts (that's `/sync-artifacts`).
- It does not generate slides, decks, or non-document formats. PDF + DOCX is the surface.
- It does not push to Confluence, Drive, or Slack. Those are separate steps (`/publish-to-confluence`, `/drive-sync`, `/share-for-review`) — though the PM can run those against the exec summary file the same way.

---

## Related commands

- `/create-prd` — produces the source PRD this skill synthesizes from.
- `/change-mode` — propagates a change across the full artifact set; re-run `/exec-summary` after a `/change-mode` pass to refresh the exec doc.
- `/publish-to-confluence` — can publish the exec summary as a standalone Confluence page or as part of the feature hub.
- `/share-for-review` — posts a Confluence link to Slack with reviewers + deadline.
