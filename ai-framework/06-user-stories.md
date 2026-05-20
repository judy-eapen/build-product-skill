# User Stories Breakdown

Runs after Gate 2 (design approval) and before Gate 3 (breakdown approval) when called from the orchestrator, OR can be called standalone via `/user-stories` when a PM has an approved PRD and wants the Gherkin breakdown without running the full pipeline.

Produces a standalone, human-readable user-stories document with exhaustive Gherkin AC, FE/BE pairing, and testing notes. This document becomes the source of truth for Jira ticket creation (Step 11 or standalone `/prd-to-jira`).

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Input Check (gracefully handle standalone calls)

Before doing anything else, determine which inputs you have available.

**Required input:**
- A PRD with user stories.

**Optional inputs (each improves the breakdown — warn about limitations if missing):**
- Design catalog — informs UX state coverage per FE story (empty / loading / error / populated).
- Codebase review — feeds HIGH-risk traceability into Testing Notes per story.
- Visual diagram — feeds screen-to-story mapping.

### If running inside the orchestrator (called from `/build-product`)

All inputs are already in conversation context at:
- PRD: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`
- Design catalog: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`
- Codebase review: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/codebase-review/[feature-name]-codebase-review.md`
- Visual diagram: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`

Skip Step 0 and proceed to Step 1.

### If running standalone (called via `/user-stories`)

Ask the PM:

> "To produce the user stories breakdown I need a PRD at minimum, plus some optional inputs that improve the output.
>
> **Required:**
> - Paste the PRD or give me the file path.
>
> **Optional (which do you have?):**
> - Design catalog — if yes, paste it or give me the path. (Without it, UX state coverage in the breakdown will be inferred from the PRD only — may miss design-driven detail.)
> - Codebase review — if yes, paste or give path. (Without it, the breakdown won't link HIGH risks to stories in the Testing Notes.)
> - Visual diagram — if yes, paste or give path. (Without it, screen-to-story mapping will be inferred from the PRD only.)
>
> Tell me what you have and I'll work with that."

Wait for the PM's response. Also ask for the feature name (used for output file path).

For each optional input the PM doesn't have, explicitly note the limitation in the breakdown output. Example header note: "⚠ This breakdown was generated without a design catalog. UX state coverage was inferred from the PRD only."

If the PM does not provide a PRD, stop. State: "I need at least a PRD to produce a user stories breakdown. Please paste one or run /create-prd first."

---

## Step 1 — Read inputs and extract stories

Read the PRD's Phased Plan (Section 7). Extract every user story across every phase.

For each PRD user story, decide whether it is:
- **FE only** — UI work, no backend changes.
- **BE only** — backend / API / data work, no UI changes.
- **Both** — UI work plus backend work. In this case, **split into a linked FE/BE pair** with the same root number (e.g., the PRD story "user sees their saved searches" becomes US-1.1 FE + US-1.2 BE).

If the PRD already labeled stories FE / BE, respect that labeling.

Pull the design catalog to confirm which screens map to which stories. If a FE story has no design counterpart, flag it as "missing design" in the breakdown.

Pull the codebase review to identify which HIGH risks touch which stories. Carry that into the Testing Notes section per story.

---

## Step 2 — Build the Sequence Map

Produce a single table covering every story:

| US-ID | Title | Type | Phase | Depends On | Related To | Size |
|---|---|---|---|---|---|---|
| US-1.1 | View saved searches | FE | 1 | US-1.2 | US-1.2 (BE pair) | M |
| US-1.2 | Saved searches endpoint | BE | 1 | — | US-1.1 (FE pair) | S |
| ... | ... | ... | ... | ... | ... | ... |

Rules for the table:
- **Depends On**: hard prerequisites. List the US-IDs of stories that must ship before this one can start. Use `—` for none.
- **Related To**: soft links, especially the FE/BE counterpart for the same feature. List the US-IDs.
- **Size**: S (less than a day), M (one to three days), L (more than three days or significant unknowns). If a story would be larger than L, mark `L+` and propose a split in the per-story section.

Below the table, produce a one-paragraph **build-order summary** a PM could read aloud in sprint planning:

> "Start with US-1.2 BE and US-1.1 FE in parallel (linked pair). Once US-1.2 is deployed, US-2.1 and US-2.2 can begin. Final block is US-3.1 + US-3.2 which depend on the auth changes in Phase 2."

---

## Step 3 — Per-story sections

For each US-ID in the Sequence Map, write a section with this structure:

```markdown
### US-[X.Y] — [Title]

**Type:** FE | BE | both
**Phase:** [N]
**Linked pair:** US-[A.B] ([type]) — [one-line note on relationship], or "None"
**Depends on:** [US-IDs or "None"]
**Size:** S | M | L | L+ (proposed split: ...)

**User Story**

As a [role], I want [goal] so that [benefit].

**Acceptance Criteria (Gherkin)**

\`\`\`
Scenario: [Happy-path scenario name]
  Given [...]
  When [...]
  Then [...]

Scenario: [Negative case — invalid input]
  Given [...]
  When [...]
  Then [...]
  And [...]

Scenario: [Edge case — empty state]
  Given [...]
  When [...]
  Then [...]

Scenario: [Edge case — boundary / max / min]
  Given [...]
  When [...]
  Then [...]

Scenario: [Error case — upstream dependency fails]
  Given [...]
  When [...]
  Then [...]
\`\`\`

**Testing Notes (high level)**
- **Test-coverage areas:** [list, e.g. UI rendering, data flow, error handling, accessibility, performance]
- **Cross-boundary verification:** how to verify this works end-to-end with the [linked pair / dependency]. Example: "Hit the BE endpoint US-1.2 directly with a saved-search payload; confirm the FE in US-1.1 renders it correctly."
- **Edge cases to cover in QA:** [list]
- **Data conditions to test:** [list, e.g. empty, single item, max payload size, special characters, unicode]
- **Performance threshold:** [pull from PRD NFR section if applicable]
- **HIGH risks from codebase review affecting this story:** [list any, with mitigation note]
- **UX state coverage (FE stories only):** empty / loading / error / populated — confirm at least one scenario covers each
```

### Gherkin coverage rules

Each story must cover these AC categories exhaustively (no minimums other than these — write as many scenarios as needed):

- **Happy path** — at least 1 scenario. The most common successful flow.
- **Negative cases** — invalid input, missing required fields, validation failures, unauthorized access (where applicable). At least 1 scenario.
- **Edge cases** — empty state / first use, max boundary, min boundary, concurrent actions, offline or degraded service, timezone or locale if applicable. At least 1 scenario.
- **Error cases** — upstream timeouts, partial failures, retries, fallbacks. At least 1 scenario (especially for BE stories).

Use proper Gherkin syntax: each scenario starts with `Scenario:`, followed by `Given` / `When` / `Then` / `And` lines. Indent two spaces.

For BE stories: scenarios should describe API behavior (request → response), not UI.
For FE stories: scenarios should describe user-visible behavior (action → screen state).

### UX state coverage for FE stories

For every FE story, confirm at least one scenario covers each of:
- **Empty state** — what the user sees when there's no data yet.
- **Loading state** — what the user sees while waiting.
- **Error state** — what the user sees when something goes wrong.
- **Populated state** — what the user sees in the normal "data present" case.

If any state is missing, add a scenario for it.

---

## Step 4 — Self-check (pre-Gate-3)

Before presenting the breakdown to the PM, run the 8 quality checks:

1. **Every PRD user story appears in the breakdown** (no drops). Cross-check against the PRD's Phased Plan.
2. **Every story has a unique US-ID.** No duplicates.
3. **Every story has at least 2 Gherkin scenarios.** Single-scenario stories are insufficient.
4. **Every story has at least one edge-case OR error-state scenario.** Not just happy path.
5. **Every linked FE/BE pair has both sides present.** If you list US-1.1 (FE) → linked to US-1.2 (BE), US-1.2 must exist as its own story.
6. **No story is sized L+ without a proposed split.** If you marked L+, the per-story section must include a "Proposed split into US-X.Y + US-X.Z" note.
7. **HIGH risks from the codebase review appear in at least one story's testing notes.** Traceability check. Read the codebase review file and confirm.
8. **UX state coverage per FE story.** Empty / loading / error / populated — all four states have at least one scenario between them.

If any check fails, flag as a WARNING (severity: WARNING) for Gate 3. Do not silently auto-fix. Surface to the PM at Gate 3 so they can decide to fix-first or approve-anyway.

If zero flags, the quality check passes silently.

---

## Step 5 — Write the file

Write the complete document to:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md
```

Structure:
1. Front matter (Source PRD, Generated date, Phases covered).
2. Build Sequence Map table.
3. Build-order summary paragraph.
4. Per-story sections in US-ID order.
5. Appendix: list of PRD user stories cross-referenced to breakdown US-IDs (for the no-drops check).

---

## Step 6 — Present Gate 3

After writing, present the breakdown to the PM at Gate 3 (defined in SKILL.md). The Gate 3 quality check block lists any flags from Step 4. The PM can:

- **Approve** — proceeds to Step 11 (Jira Export).
- **Fix specific issues** — name them; the prompt revises the breakdown; re-presents Gate 3.
- **Reject and rework** — flag concerns; pipeline pauses.

Until Gate 3 is approved, do not proceed to Jira Export.

---

## Rules

- **Source of truth is the PRD.** Do not add user stories the PRD doesn't have. Do not invent acceptance criteria not implied by the PRD's AC.
- **Split into FE/BE pairs only when warranted.** A pure-UI story (e.g., a static info screen) doesn't need a BE pair.
- **Verbatim language from Voice of Customer (Step 1 of pipeline) should appear** in user-story narratives and AC where possible. Don't paraphrase user-facing language unless necessary.
- **Update `_pipeline-state.md`** at the end of this step.
- **Self-check (Error Type 3 in error-handling.md):** does this breakdown contradict any decision recorded in the PRD's decision log? If yes, flag to the PM before writing.
