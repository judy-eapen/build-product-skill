# Create Slide Deck

Turn a feature's pipeline artifacts (PRD, exec summary, system design, user stories, timeline, design catalog, diagrams) into a presentation-ready slide deck. Use when:

- A stakeholder needs the feature as **slides**, not a 170 KB PRD or a 20 KB exec summary.
- You're prepping a leadership greenlight, a team kickoff, or a stakeholder demo and want a deck seeded from the real artifacts — not hand-assembled from scratch.
- The PRD changed materially and a previously generated deck is stale.

Standalone command. Does **not** run automatically inside `/build-product` — the PM invokes it when a deck is needed.

Read `ai-framework/rules.md`, `ai-framework/error-handling.md`, and `ai-framework/style-preferences.md` before executing.

---

## Core design — one spec, many surfaces

The command **always** produces a **slide-spec markdown** (one block per slide) as the durable source of truth. Everything else renders *from* that spec. This mirrors how `/visual-diagram` keeps Figma as the deliverable but always has a definitive source, and how `/exec-summary` keeps a markdown master that PDF/DOCX render from.

From the spec, the PM picks one or more **render surfaces**:

1. **Claude deck-prompt** (always generated) — a self-contained prompt the PM pastes into **Claude (claude.ai)** to generate an interactive slide-deck **Artifact**, or into **Figma Make / Gemini / Canva**. This is the "hand the prompt to Claude design" path. It is tool-agnostic and never depends on any MCP.
2. **Figma Slides** (via the Figma MCP) — push a real, editable deck into Figma. Honors the workspace's Figma-first rule. If the Figma MCP is unavailable, this surface is skipped and the deck-prompt + HTML surfaces cover it (do **not** fabricate a Figma link).
3. **HTML + PDF deck** (offline) — render a styled, self-contained deck via the same pandoc + Chrome-headless toolchain `/exec-summary` uses. No MCP dependency; produces a shareable file.

The deck-prompt is always written. Figma Slides and HTML+PDF are produced only if the PM selects them at the interview.

---

## Inputs

### Required
- A **feature** (pipeline-aware) **or** source content (standalone). Resolve at Step 0.

### Recommended (read if present, skip gracefully if not)
The command reads whichever of these exist; the chosen preset pre-selects the relevant ones:

| Artifact | Path under `[feature]/` | Feeds |
|---|---|---|
| Pipeline state | `_pipeline-state.json` | feature name, PM/sponsor, current step, open questions, timeline computed |
| Exec summary | `executive-summary/[feature]-executive-summary.md` | vision, use cases, asks, risks — the densest source for an exec deck |
| PRD | `prd/[feature]-prd.md` | problem, scope, personas, phases |
| System design | `technical-review/[feature]-system-design.md` | moving pieces, architecture, vendors |
| User stories | `user-stories/[feature]-user-stories.md` | scope count, epics, waves, deferred items |
| Timeline | `timeline/[feature]-timeline.md` | computed end, milestones, gap-vs-target |
| Design catalog | `design/[feature]-phase-[N]-designs.md` | screens, output URLs, states (demo deck) |
| Diagrams | `diagrams/[feature]-feature-diagram.md` | the Figma diagram link to embed on a slide |
| Decision log | `decisions/[feature]-decision-log.md` | open decisions slide |

If a source the deck wants is missing, annotate the affected slide `[NOTE: X not available — slide reduced]` rather than fabricating content.

---

## Output

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/slides/
├── [feature-name]-[deck-type]-slides.md           ← slide spec (source of truth)
├── [feature-name]-[deck-type]-deck-prompt.md       ← paste-into-Claude/Figma-Make prompt
├── [feature-name]-[deck-type]-slides.html          ← (if HTML+PDF surface chosen)
└── [feature-name]-[deck-type]-slides.pdf           ← (if HTML+PDF surface chosen)
```

`[deck-type]` is one of `exec`, `kickoff`, `demo`, or `custom`. All files live in the feature's `slides/` subfolder — parallels `executive-summary/`, `timeline/`, etc. Create the folder if it doesn't exist.

**Idempotent per deck-type.** Re-running with the same deck-type overwrites those files. Different deck-types coexist (a feature can have both an `exec` and a `kickoff` deck). The command keeps **no inline version history** — change history lives in `changelog/[feature]-changelog.md` per workspace convention. The spec's front matter carries only the PRD version it was generated against.

---

## Deck presets

Presets seed defaults (slide count range, section skeleton, which artifacts to pull). The PM can always override every value in the interview. **Custom** starts from a blank skeleton.

### Preset: `exec` — Exec / leadership greenlight
- **Range:** 5–10 slides. **Primary source:** exec summary (falls back to PRD if absent).
- **Skeleton:** Title → The ask in one line → Vision → What it does (use cases) → Moving pieces / who builds → Timeline & gap → Risks & mitigations → The decision we need → Next steps.
- **Goal default:** "Get [decision] approved."

### Preset: `kickoff` — Team kickoff / alignment
- **Range:** 10–15 slides. **Primary sources:** PRD + system design + user stories.
- **Skeleton:** Title → Why now → What we're building → Scope (in/out) → Personas & jobs → Architecture / moving pieces → Who builds what → Phases & waves → Dependencies & blockers → Open questions → How we work / next 2 weeks.
- **Goal default:** "Get the team aligned on scope, ownership, and sequence."

### Preset: `demo` — Stakeholder demo / feature overview
- **Range:** 8–12 slides. **Primary sources:** PRD + design catalog + diagrams.
- **Skeleton:** Title → The problem → Who it's for → The solution in one line → Walkthrough (one slide per key screen/flow, pulled from design catalog) → The value → What's live vs. coming → Q&A.
- **Goal default:** "Show the feature and the value it delivers."

> The `status` (sprint/status update) archetype is intentionally **not** a built-in preset (per the PM's selection). A status deck can still be built via `custom` — point it at the timeline + pipeline state.

---

## Generation steps

### Step 0 — Resolve source

- **Pipeline-aware (default):** If running inside or pointed at a feature, auto-detect the feature folder and read `_pipeline-state.json`. Confirm: "Building a deck for `[feature]` against PRD v[X.Y]."
- **Standalone fallback:** If no feature/pipeline artifacts are found, ask: "No pipeline feature detected. Point me at source content — a feature name, a markdown file path (PRD, exec summary, notes), or paste the content directly." Build the deck from whatever is provided; traceability and artifact-driven slides degrade gracefully (annotate `[NOTE: built from provided content only]`).

Do not proceed until a source is resolved.

### Step 1 — Full interview (every run)

Per the PM's standing preference, **always run the full interview** — do not skip dimensions even when a preset could default them. Present the preset's proposed value as the default for each, so the PM confirms fast but every dimension is explicit. Batch these into one `AskUserQuestion` set where the tool allows; otherwise ask in tight sequence.

1. **Deck type** — `exec` / `kickoff` / `demo` / `custom`. (Drives the skeleton + default sources.)
2. **Audience** — who is in the room? (e.g., "CEO + CFO," "the AC pod eng team," "external MLS partner"). Shapes tone and acronym handling.
3. **Primary goal** — the single decision or takeaway you want when the last slide lands. One sentence.
4. **Slide count** — preset proposes a range; PM picks an exact target. Hard cap influences density.
5. **Depth per slide** — `headline-only` (big statement, ≤1 supporting line) / `bullets` (3–5 bullets) / `dense` (bullets + a table or figure). Affects how much each artifact section is compressed.
6. **Tone & branding** — neutral / BrightMLS-branded (colors, logo) / minimal. If branded, ask for any brand colors or a logo path; otherwise use a clean default theme.
7. **Source artifacts** — multi-select; the preset pre-checks its primary sources. PM adds/removes (e.g., add the timeline to an exec deck, drop system design from a demo).
8. **Speaker notes** — yes/no. If yes, every slide gets a `Notes:` block (what to say, not what's on the slide).
9. **Render surface(s)** — multi-select: **Claude deck-prompt** (always on, can't be unchecked) / **Figma Slides** (requires Figma MCP) / **HTML + PDF**. If Figma Slides is picked but the MCP is unavailable, say so and proceed with the others.

Capture every answer; echo the final config back as a compact summary before building.

### Step 2 — Read source artifacts in parallel

Read the selected artifacts in parallel. For large files (PRD often >150 KB), read strategically — pull only the sections each slide needs (see the per-preset skeletons). The exec summary, when present, is the highest-yield source for an `exec` deck; prefer it over re-deriving from the PRD.

### Step 3 — Build the outline and confirm

Produce a **one-line-per-slide outline**: slide number, title, one-line purpose, and the source artifact/section each slide draws from. Respect the chosen slide count exactly. Present it to the PM:

> "Here's the [N]-slide outline for the [deck-type] deck. Anything to add, cut, reorder, or merge before I write the full slides?"

Apply edits. Do not write full slide content until the outline is confirmed. This is the one mandatory pause — it's cheap to reshuffle an outline, expensive to rewrite a full deck.

### Step 4 — Write the slide spec (source of truth)

Write `slides/[feature]-[deck-type]-slides.md`. Print the path before writing. Use this block format per slide:

```markdown
---
deck: [Feature Name] — [Deck Type] Deck
audience: [audience]
goal: [primary goal]
generated_against_prd: v[X.Y]
slides: [N]
theme: [neutral | brightmls | minimal]
---

## Slide 1 — [Title]
**Layout:** title | section-header | bullets | two-column | quote | image | table | closing
**Headline:** [the one line that dominates the slide, if any]
**Body:**
- [bullet / element 1]
- [bullet / element 2]
**Visual:** [chart/table/screenshot/diagram to include, or a Figma diagram link to embed; "none" if text-only]
**Source:** [artifact §section this slide traces to]
**Notes:** [speaker notes — only if speaker notes = yes]

## Slide 2 — [Title]
...
```

Rules for the spec:
- **One idea per slide.** If a section needs more, split it across slides — do not overstuff.
- **Respect depth.** `headline-only` slides have a Headline and at most one Body line. `dense` slides may carry a table or figure reference under Visual.
- **Concrete numbers** from the artifacts ("27 use cases," "+8 working days ahead," "12 base + 2 conditional tools"). No vague qualifiers.
- **Plain English, active voice.** Spell out acronyms on first use (MCP = Model Context Protocol, etc.). Apply `style-preferences.md` over everything.
- **Trace every content slide** to a source. Slides with no traceable source (title, closing, transitions) are fine but should be obviously structural.

### Step 5 — Write the Claude deck-prompt (always)

Write `slides/[feature]-[deck-type]-deck-prompt.md` — a single, self-contained prompt the PM can paste into **Claude (claude.ai)** to generate an interactive slide-deck Artifact, or into **Figma Make / Gemini / Canva**. It must stand alone (the destination tool has none of this context). Structure:

```
Build a [N]-slide presentation deck titled "[Deck title]".

Audience: [audience]. Goal: [primary goal]. Tone: [tone/branding].
Format: [16:9 slides]. Depth: [headline-only | bullets | dense]. [Include speaker notes / no speaker notes].
[If branded:] Brand: use colors [hex list] and the [product] logo. [If minimal/neutral:] Use a clean, modern, high-contrast theme.

Render this as an interactive, presentable slide deck (one slide per section below). Each slide:

Slide 1 — [Title]
[headline + bullets + visual instruction, verbatim from the spec]

Slide 2 — [Title]
...

Requirements:
- Keep one idea per slide; do not add slides beyond the [N] listed.
- Use the exact numbers and facts given; do not invent data.
- [If speaker notes:] Add presenter notes under each slide describing what to say.
- Make it visually clean and consistent: consistent type scale, generous spacing, and a simple accent color.
```

Add a short header at the top of the file telling the PM exactly how to use it:

```markdown
# Deck prompt — paste into Claude / Figma Make / Gemini / Canva

**To generate this deck:** copy everything below the line into Claude (claude.ai) and it will produce an interactive slide-deck artifact you can present and export. The same prompt works in Figma Make, Gemini, or Canva.

---
```

### Step 6 — Render selected surfaces

**Figma Slides (if selected and Figma MCP available):**
- Use the Figma MCP to create a Slides file from the spec (follow the Figma plugin's `/figma-generate-design` skill; the Figma server's MCP instructions cover Figma Slides). One Figma slide per spec block, mapping `Layout` to slide templates and binding to the team's design-system color variables where a branded theme is requested.
- Record the resulting URL in the spec's front matter as `figma_slides_url:` and print `[Open in Figma](URL)`.
- If the MCP is unavailable or errors: do **not** fabricate a link. Tell the PM "Figma MCP unavailable — skipped Figma Slides; the deck-prompt and/or HTML deck cover this surface," and continue.

**HTML + PDF (if selected):**
- Render the spec to a self-contained slide deck using the `/exec-summary` toolchain. Preferred: a Marp- or reveal.js-style HTML where each `## Slide` block becomes one slide (page-break per slide), then Chrome-headless print to PDF:
  - `pandoc` or a small HTML template to build `/tmp/deck.html` with slide-per-page CSS (`section { page-break-after: always; }`, 16:9 aspect, large type).
  - `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu --print-to-pdf=output.pdf --no-pdf-header-footer --no-margins file:///tmp/deck.html`
- Reuse the engineering-PDF CSS in the skill assets, adapted for slides (larger headings, centered titles, one section per page).
- If no conversion path works: write the HTML only (or just the spec) and tell the PM "HTML/PDF tooling unavailable — present from the spec or the deck-prompt." Do not fail the run.

**Claude deck-prompt:** already written in Step 5 — no rendering needed.

### Step 7 — Update pipeline state

If `_pipeline-state.json` exists, update an `artifacts.slide_decks` block (keyed by deck-type so multiple decks coexist):

```json
"slide_decks": {
  "[deck-type]": {
    "spec_path": ".../slides/[feature]-[deck-type]-slides.md",
    "deck_prompt_path": ".../slides/[feature]-[deck-type]-deck-prompt.md",
    "html_path": ".../slides/[feature]-[deck-type]-slides.html",
    "pdf_path": ".../slides/[feature]-[deck-type]-slides.pdf",
    "figma_slides_url": null,
    "slides": [N],
    "audience": "[audience]",
    "generated_against_prd_version": "v[X.Y]",
    "generated_at": "[ISO timestamp]"
  }
}
```

Set paths/URLs to `null` for surfaces that weren't produced. Do **not** add a change_history entry — decks are regenerated freely (same rationale as `/exec-summary`).

### Step 8 — Output

Print:

```
✓ [Deck-type] deck generated → [feature]/slides/[feature]-[deck-type]-slides.md ([N] slides)
  Deck prompt (paste into Claude/Figma Make): [feature]/slides/[feature]-[deck-type]-deck-prompt.md
  HTML:  [feature]/slides/[feature]-[deck-type]-slides.html        (or: not generated)
  PDF:   [feature]/slides/[feature]-[deck-type]-slides.pdf         (or: not generated)
  Figma: [Open in Figma](URL)                                       (or: skipped — MCP unavailable)
  PRD version covered: v[X.Y]
```

Then a one-line next step: "Paste the deck prompt into Claude for an interactive artifact, open the Figma deck to edit, or present the PDF as-is."

---

## Rules

- **The spec is the source of truth.** Every other surface renders from it. Never let Figma/HTML drift from the spec without regenerating the spec.
- **No fabricated data.** Pull numbers and claims from the artifacts. If a slide's source is missing, annotate `[NOTE: ...]` — do not invent.
- **Respect the slide count exactly.** If the content won't fit, tell the PM and propose either a higher count or which slides to merge — don't silently overflow.
- **One mandatory pause only** — outline confirmation (Step 3). Everything else runs straight through.
- **No fake Figma links.** If the Figma MCP is unavailable, skip that surface honestly (mirrors the `/visual-diagram` Mermaid-fallback discipline: never present an absent surface as present).
- **Read-only on source artifacts.** This command never modifies the PRD or any pipeline artifact. It writes only into `slides/` and the `slide_decks` block of `_pipeline-state.json`.
- **Do not implement, validate, or check drift.** That's `/validate-prd`, `/validate-user-stories`, `/sync-artifacts`.

---

## Failure modes

- **No feature and no provided content.** Refuse to proceed; ask for a feature, a file path, or pasted content.
- **PRD/exec summary both missing but other artifacts present.** Build from what exists; flag `[built from partial artifacts]` on the title slide notes.
- **Slide count too small for the goal.** Warn ("a greenlight deck under 5 slides usually drops either risks or the ask") and let the PM decide.
- **Figma MCP unavailable.** Skip Figma Slides, keep deck-prompt + HTML. Never fail the run for this.
- **PDF/HTML tooling missing.** Write the spec + deck-prompt only; notify the PM.

---

## Related commands

- `/exec-summary` — the highest-yield source for an `exec` deck; run it first if it doesn't exist.
- `/visual-diagram` — produces the Figma diagram a `kickoff`/`demo` deck can embed on an architecture slide.
- `/pull-from-figma` — refresh the design catalog the `demo` deck walks through.
- `/change-mode` — after a change propagates, re-run `/create-slidedeck [deck-type]` to refresh the deck.
- `/share-for-review` — post the rendered deck (PDF or Figma link) to Slack with reviewers + a deadline.
